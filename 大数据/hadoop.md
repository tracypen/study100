一、安装jdk
二、安装ssh
ssh-keygen -t rsa
cp .ssh/id_rsa.pub ~/.ssh/authorized_keys
三、下载hadoop并解压
tar -zxvf ***.tar.gz -C /opt/app
http://archive.cloudera.com/cdh5/cdh/5/
四、配置hadoop
配置文件位置 hadoop_home/etc/hadoop/
1.  vim etc/hadoop/hadoop-env.sh
export JAVA_HOME=/opt/app/jdk1.8.0_161
2. vim etc/hadoop/core-site.xml
<configuration>
    <property>
            <name>fs.defaultFS</name>
            <value>hdfs://hadoop01:8020</value>  //9000是1.x的默认端口 8020是hadoop2默认端口
    </property>
    <!-- 临时文件，一定要设置，否则每次重启后数据就没了 -->
    <property>
            <name>hadoop.tmp.dir</name>
            <value>/opt/app/hadoop-2.6.0-cdh5.7.0/tmp</value>
        </property>
</configuration>


3. hdfs-site.xml 

    <!-- 只有一个节点  所以副本数为1 -->
    <configuration>
        <property>
                <name>dfs.replication</name>
                <value>1</value>
        </property>
</configuration>

4. slaves 
# 配置当前的所有datanode，伪分布式所以就是本机了
改为当前主机名活localhost

5.格式化namenode

bin/hdfs namenode -format  //首次格式化

4. mapred-site.xml

 <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
</property>



5. yarn-site.xml

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



hadoop/sbin/start-all.sh 

jps

http://192.168.0.110:50070








