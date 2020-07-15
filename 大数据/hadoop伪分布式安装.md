![](https://upload-images.jianshu.io/upload_images/8387919-adf9976617e57ac7.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 一、安装jdk
1. 解压
```
tar -zxvf jdk-8u161-linux-x64.tar.gz -C /opt/app/
```
2.配置环境变量
```
vim /etc/profile
//在文件最后追加：
export JAVA_HOME=/opt/app/jdk1.8.0_161
export CLASSPATH=.:$JAVA_HOME/jre/lib/rt.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH=$PATH:$JAVA_HOME/bin
//使修改生效 
source /etc/profile
//验证
[root@localhost.haopeng jdk1.8.0_161]# java -version
java version "1.8.0_161"
Java(TM) SE Runtime Environment (build 1.8.0_161-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.161-b12, mixed mode)
```
### 二、安装ssh
```
yum install -y ssh
ssh-keygen -t rsa
cp ~/.ssh/id_rsa.pub   ~/.ssh/authorized_keys
```
### 三、安装hadoop
1.下载并解压
- 下载地址 [http://archive.cloudera.com/cdh5/cdh/5/](http://archive.cloudera.com/cdh5/cdh/5/)

```
tar -zxvf hadoop-2.6.0-cdh5.7.0.tar.gz -C /opt/app/
```
### 四、配置hadoop
配置文件位置 hadoop_home/etc/hadoop/
```
cd /opt/app/hadoop-2.6.0-cdh5.7.0/
```
- 1.hadoop-env.sh
```
vim etc/hadoop/hadoop-env.sh
export JAVA_HOME=/opt/app/jdk1.8.0_161
```
- 2.core-site.xml
`vim core-site.xml`
```
<property>
         <name>fs.defaultFS</name>
         <value>hdfs://hadoop01:8020</value>  //9000是1.x的默认端口 8020是hadoop2默认端口
 </property>
 <!-- 临时文件，一定要设置，否则每次重启后数据就没了 -->
 <property>
         <name>hadoop.tmp.dir</name>
         <value>/opt/app/hadoop-2.6.0-cdh5.7.0/tmp</value>
     </property>
```
继续执行 `mkdir -p /opt/app/hadoop-2.6.0-cdh5.7.0/tmp` 创建目录
- 3.hdfs-site.xml 
`vim hdfs-site.xml `
```
<!-- 只有一个节点  所以副本数为1 -->
<configuration>
    <property>
            <name>dfs.replication</name>
            <value>1</value>
    </property>
</configuration>
```
- 4.slaves 
> 表示集群中的slave节点，由于是伪分布式，所以改为本机
> `vim slaves`
> 修改为`localhost`或`hadoop01`
- 5.格式化`namenode`
```
bin/hdfs namenode -format  //首次格式化
```
- 6.mapred-site.xml
`vim mapred-site.xml`
```
 <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
</property>
```
- 7.yarn-site.xml
`vim yarn-site.xml`
```
<property>
		<name>yarn.resourcemanager.address</name>
		<value>hadoop01:8080</value>
	</property>
	<property>
	         <name>yarn.resoucemanager.resouce-tracker.address</name>
	         <value>hadoop01:8082</value>
	</property>
	<property>
		<name>yarn.nodemanager.aux-services</name>
		<value>mapreduce_shuffle</value>
	</property>
	<property>
		<name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>
		<value>org.apache.hadoop.mapred.ShuffleHandler</value>
	</property>
```
- 8.启动`hadoop`
```
./sbin/start-all.sh 
```
- 9.验证
```
[root@hadoop01 hadoop-2.6.0-cdh5.7.0]# jps
3249 ResourceManager
3649 Jps
2931 DataNode
3334 NodeManager
3112 SecondaryNameNode
2847 NameNode
```
- [http://192.168.2.154:50070/](http://192.168.2.154:50070/)

![](https://upload-images.jianshu.io/upload_images/8387919-d674d00e29b7b215.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- [http://192.168.2.154:8088/](http://192.168.2.154:8088/)

![](https://upload-images.jianshu.io/upload_images/8387919-4cf99e7c6f6776e4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 日志文件
```
tail -f /opt/app/hadoop-2.6.0-cdh5.7.0/logs/hadoop-root-namenode-hadoop01.log 
```
*注意：*启动日志写的是`.out`日志文件，实际上是.log文件