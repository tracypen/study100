## Spring中的循环依赖

> 面试必备

### 循环依赖

Spring`循环依赖`就是循环引用，比如两个bean A和B 两者之间相互引用对方，或者说两者相互持有对方的引用。最终形成一个`环`。

### 场景分析

spring中bean的注入方式有两种：`Setter`和`Constractor`注入(还有一种，接口注入基本不怎么用了),另外，bean的作用于也有多种（5中）singleton、property、reuest、session、application。Spring也不能对以上每种情况都可以进行循环依赖的处理。具体来说，值对于单例的setter注入bean可以处理。

#### 1.原型(Prototype)

不支持，直接抛出`BeanCurrentlyInCreationException`，原因是 `Spring` 容器对原型作用域的 `bean` 是不进行缓存，因此无法提前暴露一个创建中的 `bean`，所以也是无法解决这种情况的循环依赖。

#### 2.构造器(Constractor)

同样不支持，也会抛出`BeanCurrentlyInCreationException`，原因也很简单

### 3.单例setter

> 首先确定的是单例作用下的setter注入的循环依赖spring是可以已解决的

##### 单例的特点

1. 提前实例化 `bean`，将有问题的配置问题提前暴露
2. 将 `bean` 实例放入单例缓存 `singletonFactories` 中，当需要再次使用时，直接从缓存中取，加快了运行效率。

那么spring是如何解决的呢

### 源码解析

在类的加载中，核心方法 `org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean`，在这一步中有对循环依赖的校验和处理。

跟进去方法能够发现，如果 `bean` 是单例，并且允许循环依赖，那么可以通过提前暴露一个单例工厂方法，从而使其他 `bean` 能引用到，最终解决循环依赖的问题。

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {

		// Instantiate the bean.
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		final Object bean = instanceWrapper.getWrappedInstance();
		Class<?> beanType = instanceWrapper.getWrappedClass();
		if (beanType != NullBean.class) {
			mbd.resolvedTargetType = beanType;
		}

		// Allow post-processors to modify the merged bean definition.
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				try {
					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Post-processing of merged bean definition failed", ex);
				}
				mbd.postProcessed = true;
			}
		}

		// Eagerly cache singletons to be able to resolve circular references
    	// 是否需要提前进行初始化，用于之后解决循环依赖
		// even when triggered by lifecycle interfaces like BeanFactoryAware.
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isTraceEnabled()) {
				logger.trace("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
            // 解决循环依赖 第二个参数是回调接口，实现的功能是将切面动态织入 bean
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}

		// Initialize the bean instance. 实例化bean
		Object exposedObject = bean;
		try {
            // 填充属性
			populateBean(beanName, mbd, instanceWrapper);
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
		catch (Throwable ex) {
			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
				throw (BeanCreationException) ex;
			}
			else {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
			}
		}

		if (earlySingletonExposure) {
			Object earlySingletonReference = getSingleton(beanName, false);
			if (earlySingletonReference != null) {
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				}
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
					for (String dependentBean : dependentBeans) {
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
							actualDependentBeans.add(dependentBean);
						}
					}
					if (!actualDependentBeans.isEmpty()) {
						throw new BeanCurrentlyInCreationException(beanName,
								"Bean with name '" + beanName + "' has been injected into other beans [" +
								StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
								"] in its raw version as part of a circular reference, but has eventually been " +
								"wrapped. This means that said other beans do not use the final version of the " +
								"bean. This is often the result of over-eager type matching - consider using " +
								"'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
					}
				}
			}
		}

		// Register bean as disposable.
		try {
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
		}

		return exposedObject;
	}
```

---

- `addSingletonFactory`方法

```
protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
   Assert.notNull(singletonFactory, "Singleton factory must not be null");
   synchronized (this.singletonObjects) {
   		// 判断 singletonObjects 不存在 beanName
      if (!this.singletonObjects.containsKey(beanName)) {
      	//放入 beanName -> beanFactory，到时在 getSingleton() 
      	//获取单例时，可直接获取创建对应 bean 的工厂，解决循环依赖
         this.singletonFactories.put(beanName, singletonFactory);
         //从提前声明的缓存中移除，之前在 getSingleton() 放入的
         this.earlySingletonObjects.remove(beanName);
         //往注册缓存中添加 beanName
         this.registeredSingletons.add(beanName);
      }
   }
}
```



另外，Spring内部维护了三个**Map**，也常被成为缓存

在`DefaultSingletonBeanRegistry`类中：

```java
	/** Cache of singleton objects: bean name to bean instance. 单例池*/
	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

	/** Cache of singleton factories: bean name to ObjectFactory. 映射创建Bean的原始工厂*/
	private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

	/** Cache of early singleton objects: bean name to bean instance. 映射Bean的早期引用*/
	private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);
```

- singletonObjects

  用于保存 `BeanName` 和创建 `bean` 实例之间的关系，`bean-name` --> `instanct`

- singletonFactories

  用于保存 `BeanName` 和创建 `bean` 的 **工厂** 之间的关系，`bean-name` --> `objectFactory`

- earlySingletonObjects

  也是保存 `beanName` 和创建 `bean` 实例之间的关系，与 `singletonObjects` 的**不同之处在于**，当一个单例 `bean` 被放入到这里之后，那么其他 `bean` 在创建过程中，就能通过 `getBean` 方法获取到，**目的是用来检测循环引用**

后两个Map其实是“**垫脚石**”级别的，只是创建Bean的时候，用来借助了一下，创建完成就清掉了。



### 总结

假设有两个bean CircleA` 和 `CircleB 并循环依赖，他们初始化的过程如下：

![](https://upload-images.jianshu.io/upload_images/8387919-364b91d308bcded1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



1. 创建 `beanA`，先记录对应的 `beanName` 然后将 `beanA` 的**创建工厂 beanFactoryA** 放入缓存中
2. 对 `beanA` 的属性填充方法 `populateBean`，检查到依赖 `beanB`，缓存中没有 `beanB` 的实例或者单例缓存，于是要去实例化 `beanB`。
3. 开始实例化 `beanB`，经历创建 `beanA` 的过程，到了属性填充方法，检查到依赖了 `beanA`。
4. 调用 `getBean(A)` 方法，在这个函数中，不是真正去实例化 `beanA`，而是先去检测缓存中是否有已经创建好的对应的 `bean`，或者已经创建好的 `beanFactory`
5. 检测到 `beanFactoryA` 已经创建好了，而是直接调用 `ObjectFactory` 去创建 `beanA`

### 结合关键代码梳理流程

#### 创建原始 bean

```
BeanWrapper instanceWrapper = createBeanInstance(beanName, mbd, args);
// 原始 bean
final Object bean = instanceWrapper.getWrappedInstance();
复制代码
```

在这一步中，创建的是原始 `bean`，因为还没到最后一步属性解析，所以这个类里面没有属性值，可以将它想象成 `new ClassA`，同时没有构造函数等赋值的操作，这个原始 `bean` 信息将会在下一步使用到。

------

#### addSingleFactory

```
// 注释 5.2 解决循环依赖 第二个参数是回调接口，实现的功能是将切面动态织入 bean
addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
复制代码
```

前面也提到过这个方法，它会将需要提前曝光的单例加入到缓存中，将单例的 `beanName` 和 `beanFactory` 加入到缓存，在之后需要用到的时候，直接从缓存中取出来。

------

#### populateBean 填充属性

刚才第一步时也说过了，一开始创建的只是初始 `bean`，没有属性值，所以在这一步会解析类的属性。在属性解析时，会判断属性的类型，如果判断到是 `RuntimeBeanReference` 类型，将会解析引用。

就像我们写的例子，`CircleA` 引用了 `CircleB`，在加载 `CircleA`时，发现 `CircleB` 依赖，于是乎就要去加载 `CircleB`。

我们来看下代码中的具体流程吧：

```
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
    ...
    if (pvs != null) {
		// 将属性应用到 bean 中，使用深拷贝，将子类的属性一并拷贝
		applyPropertyValues(beanName, mbd, bw, pvs);
	}
}

protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) {
    ...
    String propertyName = pv.getName();
	Object originalValue = pv.getValue();
	// 注释 5.5 解析参数，如果是引用对象，将会进行提前加载
	Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);
	...
}

public Object resolveValueIfNecessary(Object argName, @Nullable Object value) {
	// 我们必须检查每个值，看看它是否需要一个运行时引用，然后来解析另一个 bean
	if (value instanceof RuntimeBeanReference) {
		// 注释 5.6 在这一步中，如果判断是引用类型，需要解析引用，加载另一个 bean
		RuntimeBeanReference ref = (RuntimeBeanReference) value;
		return resolveReference(argName, ref);
	}
	...
}
复制代码
```

跟踪到这里，加载引用的流程比较清晰了，发现是引用类的话，最终会委派 `org.springframework.beans.factory.support.BeanDefinitionValueResolver#resolveReference` 进行引用处理，核心的两行代码如下：

```
// 注释 5.7 在这里加载引用的 bean
bean = this.beanFactory.getBean(refName);
this.beanFactory.registerDependentBean(refName, this.beanName);
复制代码
```

在这一步进行 `CircleB` 的加载，但是我们写的例子中，`CircleB` 依赖了 `CircleA`，那它是如何处理的呢，所以这时，我们刚才将 `CircleA` 放入到缓存中的信息就起到了作用。

------

#### getSingleton

还记得之前在类加载时学到的只是么，单例模式每次加载都是取同一个对象，如果在缓存中有，可以直接取出来，在缓存中没有的话才进行加载，所以再来熟悉一下取单例的方法：

```
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
	Object singletonObject = this.singletonObjects.get(beanName);
	// 检查缓存中是否存在实例
	if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
		// 记住，公共变量都需要加锁操作，避免多线程并发修改
		synchronized (this.singletonObjects) {
			// 如果此 bean 正在加载则不处理
			singletonObject = this.earlySingletonObjects.get(beanName);
			if (singletonObject == null && allowEarlyReference) {
				// 当某些方法需要提前初始化，调用 addSingletonFactory 方法将对应的
				// objectFactory 初始化策略存储在 earlySingletonObjects，并且从 singletonFactories 移除
				ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
				if (singletonFactory != null) {
					singletonObject = singletonFactory.getObject();
					this.earlySingletonObjects.put(beanName, singletonObject);
					this.singletonFactories.remove(beanName);
				}
			}
		}
	}
	return singletonObject;
}
复制代码
```

虽然 `CircleB` 引用了 `CircleA`，但在之前的方法 `addSingletonFactory` 时，`CircleA` 的 `beanFactory` 就提前暴露。

所以 `CircleB` 在获取单例 `getSingleton()` 时，能够拿到 `CircleA` 的信息，所以 `CircleB` 顺利加载完成，同时将自己的信息加入到缓存和注册表中，接着返回去继续加载 `CircleA`，由于它的依赖已经加载到缓存中，所以 `CircleA` 也能够顺利完成加载，最终整个加载操作完成~

说白了就是先只是简单构造对象，然后将beanfactory中的bean再进行二次处理，设置对应属性





#### 参考文档

- https://juejin.im/post/5e927e27f265da47c8012ed9