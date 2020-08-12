#### 一、经典实用场景

- 1.Spring中事务的隔离级别

  ```java
  public abstract class TransactionSynchronizationManager {
      private static final Log logger = LogFactory.getLog(TransactionSynchronizationManager.class);
      private static final ThreadLocal<Map<Object, Object>> resources = new NamedThreadLocal("Transactional resources");
      private static final ThreadLocal<Set<TransactionSynchronization>> synchronizations = new NamedThreadLocal("Transaction synchronizations");
      private static final ThreadLocal<String> currentTransactionName = new NamedThreadLocal("Current transaction name");
      private static final ThreadLocal<Boolean> currentTransactionReadOnly = new NamedThreadLocal("Current transaction read-only status");
      private static final ThreadLocal<Integer> currentTransactionIsolationLevel = new NamedThreadLocal("Current transaction isolation level");
      private static final ThreadLocal<Boolean> actualTransactionActive = new NamedThreadLocal("Actual transaction active");
      }
  ```

- 2.状态（回话）信息共享

  比如在项目中标记微信用户的openId，这么做是因为，在之前的老系统中添加微信小程序的功能，为了尽可能最小的影响之前系统的认证体系，又能满足现有小程序功能中的认证和会话标记。

  ```java
  /**
   * 微信用户信息（openId）上下文
   * @author haopeng
   * @date 2020-06-09 15:19
   */
  public class MaUserContextHolder {
      private static ThreadLocal<String > threadLocal = new ThreadLocal<>();
  
      public static void setOpenId(String  openId){
          threadLocal.set(openId);
      }
  
  
      public static String getOpenId() {
          return threadLocal.get();
      }
  
  
      public static void remove() {
          threadLocal.remove();
      }
  
  }
  ```

- 3.解决线程安全问题

  典型应用就是`SimpleDateFormat`

  ```java
   //SimpleDataFormat的parse()方法，内部有一个Calendar对象，调用SimpleDataFormat的parse()方法会先调用Calendar.clear（），
      // 然后调用Calendar.add()，如果一个线程先调用了add()然后另一个线程又调用了clear()，这时候parse()方法解析的时间就不对了。
      private static ThreadLocal dateFormat = ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));
  ```

#### 二、示例代码

分析一下代码执行结果

```java

/**
 * @author haopeng
 * @date 2020-07-31 11:05
 */
public class TheadLocalTest {
    //1.声明ThreadLocal变量
    private static ThreadLocal<String> localName = ThreadLocal.withInitial(() -> "张三");

    public static void main(String[] args) throws InterruptedException {
        System.out.println("主线程第一次获取到的值为： " + localName.get());
        localName.set("李四");
        //创建子线程
        Thread thread = new Thread(() -> {
            System.out.println("内部线程获取到的的值为：" + localName.get());
        });
        thread.start();
        thread.join();
        System.out.println("主线程第二次获取到的值为： " + localName.get());
        localName.remove();

        //如果需要线程数据共享可以使用InheritableThreadLocal
        ThreadLocal<String> local = new InheritableThreadLocal<>();
        local.set("测试线程数据共享");
    }
}

运行结果：
    主线程第一次获取到的值为： 张三
    内部线程获取到的的值为：张三
    主线程第二次获取到的值为： 李四
    
```

- join()是为了保证等待子线程执行完毕
- 通过运行结果可以发现，变量`张三`在线程之间是相互隔离的

#### 三、原理分析

```java
 /**
     * Sets the current thread's copy of this thread-local variable
     * to the specified value.  Most subclasses will have no need to
     * override this method, relying solely on the {@link #initialValue}
     * method to set the values of thread-locals.
     *
     * @param value the value to be stored in the current thread's copy of
     *        this thread-local.
     */
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```

> 通过分析以上`set`方法的源码可以看出是通过当前线程获取一个`ThreadLocalMap`,并最终将设置的值以键值对形式存储在此`Map`中。

继续查看` getMap(t)`源码

```java
 ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```

> 这里可以看到`ThreadLocal.ThreadLocalMap`是线程类`Thread`中的一个静态变量，这也就是其数据隔离的本质原因了-----> 每一个线程对象内部维护一个`ThreadLocalMap`，隔离的数据就是跟线程绑定的，而不同线程之间是不能共享的

查看`ThreadLocalMap`关键源码

```java
static class ThreadLocalMap {
    //1 Entry对象
      static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
    // 2.Entry数组
    private Entry[] table;
    
    //3. set方法
         private void set(ThreadLocal<?> key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();

                if (k == key) {
                    e.value = value;
                    return;
                }

                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
}
```

通过源码可以看出：

- 1.`ThreadLocalMap`并没有实现`Map`接口，所以它与`HashMap`不大一样,并且其内部并没有链表的概念

- 2.相同的地方是`ThreadLocalMap`内部也是一个Entry

- 3.`Entry`对象实现了`WeakReference`其内部结构如下：

  ![](https://upload-images.jianshu.io/upload_images/8387919-8a46278d0a751a92.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  
- 4.当插入数据时，计算下标`int i = key.threadLocalHashCode & (len-1)`，当该数组对应下标位置没有元素时，直接初始化一个Entry放到这里。当出现Hash冲突时，继续判断key是否相等，也就是判断这个当前的`ThreadLocal`是否相等，如果相等，进行更新，否则继续找下一个位置，知道找到一个为空的位置

  

#### 四、内存泄漏

1. 首先要明白，由于是线程绑定的实现数据隔离，自然醒到数据时存在占空间中的，其实不完全是，只能说每一个线程的`ThreadLocalMap`的引用是存在栈空间中的，实际上`ThreadLocalMap`也是被创建出来的，所以存在堆空间中的

2. 由于存在堆空间中的，所以需要垃圾回收。`Entry`中的Key实现了弱引用，这就导致了，当这`ThreadLocal`对象使用之后，没有外部强引用时，下次gc时会被回收,而此时时间的value值没有被回收，这就是内存泄漏的原因

   ```java
   static class Entry extends WeakReference<ThreadLocal<?>> {
               /** The value associated with this ThreadLocal. */
               Object value;
   
               Entry(ThreadLocal<?> k, Object v) {
                   super(k);
                   value = v;
               }
           }
   ```

   3. 解决内存泄漏的办法就是使用完手动remove

> 最后，如果想使用线程之间共享的一个全局变量呢 ，可以使用`InheritableThreadLocal`他是`ThreadLocal`的子类

```java
public class InheritableThreadLocal<T> extends ThreadLocal<T> {
    /**
     * Computes the child's initial value for this inheritable thread-local
     * variable as a function of the parent's value at the time the child
     * thread is created.  This method is called from within the parent
     * thread before the child is started.
     * <p>
     * This method merely returns its input argument, and should be overridden
     * if a different behavior is desired.
     *
     * @param parentValue the parent thread's value
     * @return the child thread's initial value
     */
    protected T childValue(T parentValue) {
        return parentValue;
    }

    /**
     * Get the map associated with a ThreadLocal.
     *
     * @param t the current thread
     */
    ThreadLocalMap getMap(Thread t) {
       return t.inheritableThreadLocals;
    }

    /**
     * Create the map associated with a ThreadLocal.
     *
     * @param t the current thread
     * @param firstValue value for the initial entry of the table.
     */
    void createMap(Thread t, T firstValue) {
        t.inheritableThreadLocals = new ThreadLocalMap(this, firstValue);
    }
}
```

使用如下：

```java
ThreadLocal<String> local = new InheritableThreadLocal<>();
local.set("测试线程数据共享");
```

