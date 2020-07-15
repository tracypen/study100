![](https://upload-images.jianshu.io/upload_images/8387919-215804f8506eb33c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 本文主要记录Spring源码的阅读环境的搭建和验证
### 1.前置条件（我的环境）
- Intellij IDEA 2020.1.3
- jdk 1.8
- mavne3.6.1
- gradle 5.6
- git
- 操作系统Win10
### 2. 安装Gradle
-  下载Gradle并解压到指定目录
[官方网站](https://gradle.org/install/#manually)
-  配置环境变量
1.变量名：`GRADLE_HOME` Gradle的解压目录
2. 变量名：`GRADLE_USER_HOME`   自定义Gradle仓库目录或者Maven的仓库目录
3. 添加Path %GRADLE_HOME%\bin
4. 验证
` gradle -v`
![](https://upload-images.jianshu.io/upload_images/8387919-cf9d0a551bd42cca.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
-  配置Gradle仓库源
在Gradle安装目录下的 init.d 文件夹下，新建一个 init.gradle 文件，里面填写以下配置，使用阿里云仓库镜像地址
```
allprojects {
    repositories {
        maven { url 'file:///C:/MyProgram/development/maven/repository'}
        mavenLocal()
        maven { name "Alibaba" ; url "https://maven.aliyun.com/repository/public" }
        maven { name "Bstek" ; url "http://nexus.bsdn.org/content/groups/public/" }
        mavenCentral()
    }

    buildscript { 
        repositories { 
            maven { name "Alibaba" ; url 'https://maven.aliyun.com/repository/public' }
            maven { name "Bstek" ; url 'http://nexus.bsdn.org/content/groups/public/' }
            maven { name "M2" ; url 'https://plugins.gradle.org/m2/' }
        }
    }
}
```
repositories 中写的是获取 jar 包的顺序。先是本地的 Maven 仓库路径；接着的 mavenLocal() 是获取 Maven 本地仓库的路径，应该是和第一条一样，但是不冲突；第三条和第四条是从国内和国外的网络上仓库获取；最后的 mavenCentral() 是从Apache提供的中央仓库获取 jar 包。

- IDEA中配置Gradle
新版本的IDEA中设置Gradle界面有很大的变化，更加简洁。如下图
![](https://upload-images.jianshu.io/upload_images/8387919-117636527e34d246.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
1. 配置gradle的仓库地址
2. 配置idea使用我们本地安装的gradle（指定安装目录）
3. 配置jdk版本

### 拉取源码
[github 地址](https://github.com/spring-projects/spring-framework/tree/5.1.x)
[码云镜像地址](https://gitee.com/mirrors/Spring-Framework/tree/5.1.x)
github服务器地址比较慢可以使用码云的镜像地址
```
git pull https://github.com/spring-projects/spring-framework
```
拉取成功后切换到指定分支
```
git beanch 查看当前分支
git checkout 5.1.x 切换到指定分支
```
### 构建
- 前置命令
在`README中找到`[Build from Source](https://github.com/spring-projects/spring-framework/wiki/Build-from-Source) 打开后可以看到具体的构建步骤
由于我这里是最终通过IDEA进行构建与编码，所以选择下面的通过
 [IntelliJ IDEA](https://github.com/spring-projects/spring-framework/blob/master/import-into-idea.md) 构建
可以看到大致步骤如下：
```
Precompile spring-oxm with ./gradlew :spring-oxm:compileTestJava
Import into IntelliJ (File -> New -> Project from Existing Sources -> Navigate to directory -> Select build.gradle)
When prompted exclude the spring-aspects module (or after the import via File-> Project Structure -> Modules)
Code away
```
过程中可能出现如下错误
![](https://upload-images.jianshu.io/upload_images/8387919-54ac10c814209f3d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
通过错误提示进入到指定目录删除对用目录的缓存文件，让gradle重新下载jar包
![](https://upload-images.jianshu.io/upload_images/8387919-cbe39b456776a90e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### 导入到IDEA中
在IDEA中选择open或import进行导入，导入之后，idea会自动识别build.gradle文件并进行构建，等待一会，出现build success 代表构建成功
![](https://upload-images.jianshu.io/upload_images/8387919-097898a1fb40ec73.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
#### 排除aspect模块
> 这里说下为什么要排除，因为sapect自己有编译器，会和jvm冲突，所以先排除
> 右键spring-aspect模块 选择Load/Unload Modules 将sapect模块移除
> ![](https://upload-images.jianshu.io/upload_images/8387919-0ede9de27d9274d3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### 测试& Code away
1. idea中新建模块
![](https://upload-images.jianshu.io/upload_images/8387919-b10680f6c8ed833d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![](https://upload-images.jianshu.io/upload_images/8387919-6be24f307c5ac0f5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
2. 创建bean
```
/**
 * @author haopeng
 * @date 2020-07-14 22:08
 */
public class Person {

	/**
	 * name
	 */
	private String name;

	/**
	 * age
	 */
	private Integer age;

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}

	public Integer getAge() {
		return age;
	}

	public void setAge(Integer age) {
		this.age = age;
	}

	public String say() {
		return "name:" + name + "=== age" + age;
	}
}

```
3. 创建beans配置类
```
@Configuration
public class MyConfiguration {
	@Bean
	public Person person() {
		Person person = new Person();
		person.setName("Mcgrady");
		person.setAge(28);
		return person;
	}
}
```
4. 测试
```
/**
 * @author haopeng
 * @date 2020-07-14 22:06
 */
public class HellloSpring {
	public static void main(String[] args) {
		AnnotationConfigApplicationContext context =
				new AnnotationConfigApplicationContext(MyConfiguration.class);

		Person person = (Person)context.getBean("person");
		System.out.println(person.say());
	}
}
```
![](https://upload-images.jianshu.io/upload_images/8387919-9e96f51a21fd5101.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

成功的迈出第一步！！！加油。

> 总结下来：Spring的源码构建中很遇到好多环境问题的坑，一定要耐心慢慢去查看错误提示，查看官网文档，或百度/google查找别人的解决办法，最终构建成功

##### 参考： 
- [https://spring.io/projects/spring-framework](https://spring.io/projects/spring-framework)
- https://juejin.im/post/5d03821cf265da1bcf5dd908
- https://zhuanlan.zhihu.com/p/63145978



