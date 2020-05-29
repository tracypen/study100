### YARN架构

>  1\*ResourceManager +   n \* 多个nodeManager

![](https://hadoop.apache.org/docs/stable/hadoop-yarn/hadoop-yarn-site/yarn_architecture.gif)

##### `ResourceManager`的职责：

> 一个集群active的RM只有一个，负责整个集群资源管理和调度

- 1）处理客户端的请求（启动/杀死）

- 2）启动/监控`ApplicationMster`(一个作业对应一个AM)
- 3）监控NM(通过心跳)
- 4）系统的资源分配和调度

##### NodeManager的职责：

> 整个集群中有N个`NodeManager`负责单个节点的资源管理和使用，以及Task的运行情况。

- 1）定期向RM汇报本节点的资源情况和Task的运行状态
- 2）接收并处理RM的container启停的各种命令
- 3）单个节点资源管理和任务管理

#####  ApplicationMaster的职责：

> 一个任务/应用对应一个AM，负责应用程序的管理	

- 1）数据切分
- 2）为应用程序向RM申请资源`container`，并分配给内部任务
- 3）与`NodeManager`通信，以启停task，task是运行在`container`中的
- 4）task的监控和容错

##### Container:

> 对任务运行情况的描述与封装，如cpu、memory、环境变量

### YARN总体执行流程

![](https://upload-images.jianshu.io/upload_images/8387919-93262455842a335c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1. 用户向YARN提交作业
2. RM为改作业分配第一个container（AM）
3. RM与一定的NM，要求NM在这个container上启动应用程序的AM
4. AM首先向RM注册（这样用户就可以通过RM查询到作业的运行情况了）然后AM降为各个人物申请资源，并监控运行情况
5. AM采用轮询的方式，通过RPC协议向RM申请和领取资源
6. AM申请带资源后便和响应的`NodeManager`通信，要求`NodeManager`启动任务
7. NM启动我们的作业对应的task

### 运行MapReduce作业

```
创建hello.txt文件
vim hello.txt 
写入如下内容
hello hp world java flink
scala python java c# spark
print hello world scala
hadoop java scala spark

在hdfs上创建/input/wc文件夹
hadoop fs -mkdir -p /input/wc
将hello.txt文件上传至 /input/wc目录下
hadoop fs -put hello.txt /input/wc/

运行MapReduce实例wc程序
hadoop jar ./share/hadoop/mapreduce/hadoop-mapreduce-examples-2.6.0-cdh5.7.0.jar wordcount /input/wc/hello.txt /output/wc/

运行结束后查看运行结果
fs -ls /output/wc/
会多出来一个这样的文件part-r-00000后面的编号应该是任务编号
查看统计结果
hadoop fs -cat /output/wc/hello.txt/part-r-00000

c#	1
flink	1
hadoop	1
hello	2
hp	1
java	3
print	1
python	1
scala	3
spark	2
world	2

```



#### 参考

[Yarn架构官网介绍](https://hadoop.apache.org/docs/stable/hadoop-yarn/hadoop-yarn-site/YARN.html)