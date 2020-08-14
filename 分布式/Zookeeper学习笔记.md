## 简介

> `Zookeeper`是一个分布式的、开源的程序协调服务，是 hadoop 项目下的一个子项目。他提供的主要功 能包括：配置管理、名字服务、分布式锁、集群管理等。从名字来看zookeeper 就是动物园管理员，他是用来管 hadoop（大象）、Hive(蜜蜂)、pig(小猪)的管理员， Apache Hbase 和 Apache Solr 的分布式集群都用到了 zookeeper。另外zookeeper是分布式的基础，学习好zookeeper是学习分布式系统的基础，比如ZAB协议等。

## 基本概念

ZooKeepr 提供基于类似于文件系统的目录节点树方式的数据存储，这是一个共享的**内存中的树型结构**。有几个概念需要关注一下。

1. **Session**会话 客户端启动会与服务端建立一个 TCP 长连接，通过这个连接可以发送请求并接受响应，以及接受服务端的 Watcher 事件通知

2. **Znode** 数据节点 ，会保存自己的数据内容和属性信息，分为持久和临时节点，节点有 SEQUENTIAL 属性，Znode有一下几种类型

   - (1)**PERSISTENT** **持久化节点**: 所谓持久节点，是指在节点创建后，就一直存在，直到 有删除操作来主动清除这个节点。否则不会因为创建该节点的客户端会话失效而消失。

   - (2)**PERSISTENT_SEQUENTIAL** **持久顺序节点**：这类节点的基本特性和上面的节点类 型是一致的。额外的特性是，在 ZK 中，每个父节点会为他的第一级子节点维护一份时序， 会记录每个子节点创建的先后顺序。基于这个特性，在创建子节点的时候，可以设置这个属 性，那么在创建节点过程中，ZK 会自动为给定节点名加上一个数字后缀，作为新的节点名。 这个数字后缀的范围是整型的最大值。 在创建节点的时候只需要传入节点 “/test_”，这样 之后，zookeeper 自动会给”test_”后面补充数字。

   - (3)**EPHEMERAL 临时节点**：和持久节点不同的是，临时节点的生命周期和客户端会 话绑定。也就是说，如果客户端会话失效，那么这个节点就会自动被清除掉。注意，这里提 到的是会话失效，而非连接断开。另外，在临时节点下面不能创建子节点。 这里还要注意一件事，就是当你客户端会话失效后，所产生的节点也不是一下子就消失 了，也要过一段时间，大概是 10 秒以内，可以试一下，本机操作生成节点，在服务器端用 命令来查看当前的节点数目，你会发现客户端已经 stop，但是产生的节点还在。

   - (4) **EPHEMERAL_SEQUENTIAL 临时自动编号节点**：此节点是属于临时节点，不过带 有顺序，客户端会话结束节点就消失。

4. **Watcher 事件监听器** ,类似一个观察者设计模式，客户端可以监听节点或者节点对应的数据,该机制是 ZooKeeper 实现分布式协调服务的重要特性。其实zk就是znode+watch，通过4中节点和watch机制就可以玩出很多花样来。

5. Zookeeper 集群中的角色

   - **leader**领导者 负责进行投票的发起和决议
   - **follower**跟随者 用于接收客户端请求，并相应结果，在选主过程中参与投票
   - **observer**观察者 将写请求转发给leader （**思考一下为什么会有个观察者节点，有什么作用呢？**答案在最后）
   
5. **ACL**

   ZooKeeper 采用 ACL（AccessControlLists）策略来进行权限控制，类似于 UNIX 文件系统的权限控制。

   - create：创建子节点的权限
   - read 获取节点数据和子节点的权限
   - write 更新节点数据的权限
   - delete 删除子节点的权限
   - admin 设置节点ACL的权限

   其中尤其需要注意的是，CREATE 和 DELETE 这两种权限都是针对子节点的权限控制。
## stat结构体

- czxid 创建节点的事务id 全局唯一有次序的时间戳，如zxid1小于zxid2，那么zxid1的事务一定发生在zxid2之前
- ctime-znode 被创建的毫秒数
- mzxid 节点最后更新的事务id
- mtime-znode 最后修改的毫秒数
- pZxid-znode 最后更新的子节点zxid
- cversion-znode 子节点变化号，znode修改次数
- dataversion-znode数据变化号
- aclVersion 访问控制列表的变化号（acl访问控制，可以理解为权限）
- ephemeralOwner  如果是临时节点 表示临时节点所属客户端的sessionId

## Zookeeper使用场景

![](https://upload-images.jianshu.io/upload_images/8387919-c6c4659eacb4c50b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 众所周知，zookeeper的架构特备简单，一个unix文件系统+一个监听器就可以实现有很多功能。

- 数据发布/订阅
- 负载均衡
- 命名服务
- 分布式协调/通知
- 集群管理
- Master 选举
- 分布式锁
- 分布式队列

#### 数据发布订阅

数据发布/订阅的一个常见的场景是配置中心，发布者把数据发布到 ZooKeeper 的一个或一系列的节点上，供订阅者进行数据订阅，达到**动态获取数据**的目的。

配置信息一般有几个特点:

1. 数据量小的KV
2. 数据内容在运行时会发生动态变化
3. 集群机器共享，配置一致

![](https://upload-images.jianshu.io/upload_images/8387919-aa9041d0c5b6a53d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

ZooKeeper 采用的是**推拉结合**的方式。

1. 推: 服务端会推给注册了监控节点的客户端 Wathcer 事件通知
2. 拉: 客户端获得通知后，然后主动到服务端拉取最新的数据

实现的思路可以如下。

```bash
mysql.driverClassName=com.mysql.jdbc.Driver
dbJDBCUrl=jdbc:mysql://127.0.0.1/mcgrady
username=root
password=root
```

1. 把配置信息写到一个 Znode 上，例如 `/conf`
2. 客户端启动初始化阶段读取服务端节点的数据，并且注册一个数据变更的 Watcher
3. 配置变更只需要对 Znode 数据进行 set 操作，数据变更的通知会发送到客户端，客户端重新获取新数据，完成配置动态修改

#### 分布式协调

这个其实是 zookeeper 很经典的一个用法，简单来说，就好比，你 A 系统发送个请求到 mq，然后 B 系统消息消费之后处理了。那 A 系统如何知道 B 系统的处理结果？用 zookeeper 就可以实现分布式系统之间的协调工作。A 系统发送请求之后可以在 zookeeper 上**对某个节点的值注册个监听器**，一旦 B 系统处理完了就修改 zookeeper 那个节点的值，A 系统立马就可以收到通知，完美解决。

![](https://upload-images.jianshu.io/upload_images/8387919-2027c8b9e3688a62.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 负载均衡

负载均衡是一种手段，用来把对某种资源的访问分摊给不同的设备，从而**减轻单点**的压力。

实现的思路:

1. 首先建立 Servers 节点，并建立监听器监视 Servers 子节点的状态（用于在服务器增添时及时同步当前集群中服务器列表）
2. 在每个服务器启动时，在 Servers 节点下建立**临时子节点** Worker Server，并在对应的字节点下存入服务器的相关信息，包括服务的地址，IP，端口等等
3. 可以**自定义一个负载均衡算法**，在每个请求过来时从 ZooKeeper 服务器中获取当前集群服务器列表，根据算法选出其中一个服务器来处理请求

![](https://upload-images.jianshu.io/upload_images/8387919-d56aff803fd2ce54.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 分布式锁

举个栗子。对某一个数据连续发出两个修改操作，两台机器同时收到了请求，但是只能一台机器先执行完另外一个机器再执行。那么此时就可以使用 zookeeper 分布式锁，一个机器接收到了请求之后先获取 zookeeper 上的一把分布式锁，就是可以去创建一个 znode，接着执行操作；然后另外一个机器也**尝试去创建**那个 znode，结果发现自己创建不了，因为被别人创建了，那只能等着，等第一个机器执行完了自己再执行。

![](https://upload-images.jianshu.io/upload_images/8387919-8e5a139e56c1fdc3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 元数据/配置信息管理

zookeeper 可以用作很多系统的配置信息的管理，比如 kafka、storm 等等很多分布式系统都会选用 zookeeper 来做一些元数据、配置信息的管理，包括 dubbo 注册中心不也支持 zookeeper 么？

![](https://upload-images.jianshu.io/upload_images/8387919-5858a5b1b4950925.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### HA高可用性

这个应该是很常见的，比如 hadoop、hdfs、yarn 等很多大数据系统，都选择基于 zookeeper 来开发 HA 高可用机制，就是一个**重要进程一般会做主备**两个，主进程挂了立马通过 zookeeper 感知到切换到备用进程。

![](https://upload-images.jianshu.io/upload_images/8387919-aec97bb8e275409c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 安装zookeeper

#### 单机模式

1. 下载zookeeper 下载地址 `http://mirrors.hust.edu.cn/apache/zookeeper/`

2. 解压

   ```properties
   tar -zxvf  apache-zookeeper-3.6.0.tar.gz -C /usr/software/
   ```

3. 配置

```properties
cd conf 进入到zookeeper的配置文件目录
cp zoo_sample.cfg zoo.cfg 复制一份名为zoo.cfg的配置文件
需要注意的几个配置
tickTime=2000 #心跳间隔
initLimit=5 #集群中的follower服务器(F)与leader服务器(L)之间初始连接时能容忍的最多心跳数（tickTime的数量）
syncLimit=2 #集群中的follower服务器与leader服务器之间请求和应答之间能容忍的最多心跳数（tickTime的数量）
dataDir=/usr/software/zookeeper-3.6.0/data  # 数据存储路径
dataLogDir=/usr/software/zookeeper-3.6.0/logs #日志路径
clientPort=2181 #端口
```

4.  常用命令

```properties
启动命令：./bin/zkServer.sh start

停止命令：./bin/zkServer.sh stop　　

重启命令：./bin/zkServer.sh restart

状态查看命令：./bin/zkServer.sh status
```

5. 启动客户端

```
./bin/zkCli.sh
```

#### 集群模式

1. 三台机器上分别部署一个zookeeper实例
2. 分别在3台机器的配置文件

```properties
tickTime=2000
initLimit=5
syncLimit=2
dataDir=/usr/software/zookeeper-3.6.0/data
dataLogDir=/usr/software/zookeeper-3.6.0/logs
clientPort=2181

server.1=127.0.0.1:2888:3888
server.2=127.0.0.2:2888:3888
server.3=127.0.0.3:2888:3888
```

**注意：**server.A=B:C:D中的A是一个数字,表示这个是第几号服务器,B是这个服务器的IP地址，C第一个端口用来集群成员的信息交换,表示这个服务器与集群中的leader服务器交换信息的端口，D是在leader挂掉时专门用来进行选举leader所用的端口。

3. 创建ServerID标识

除了修改zoo.cfg配置文件外,zookeeper集群模式下还要配置一个myid文件,这个文件需要放在dataDir目录下。这个文件里面有一个数据就是A的值（该A就是zoo.cfg文件中server.A=B:C:D中的A）,在zoo.cfg文件中配置的dataDir路径中创建myid文件。

```properties
三台机器分别执行： 
echo "1" > /usr/software/zookeeper-3.6.0/myid 
echo "2" > /usr/software/zookeeper-3.6.0/myid 
echo "3" > /usr/software/zookeeper-3.6.0/myid 
```

4. 依次启动每个节点

```properties
./bin/zkServer.sh start
```

5. 查看集群状态

```properties
分别 执行
./bin/zkServer.sh status

可以看到lwader：
ZooKeeper JMX enabled by default
Using config: /usr/software/zookeeper-3.6.0/bin/../conf/zoo.cfg
Mode: leader

follower：
ZooKeeper JMX enabled by default
Using config: /usr/software/zookeeper-3.6.0/bin/../conf/zoo.cfg
Mode: follower

```

6. 客户端常用命令

```properties
ls 查看命令
get 获取节点数据和更新信息
create 创建节点
create -e 创建临时节点
create -s 创建顺序节点 自动累加
set path data [version] 修改节点
delete path [version] 删除节点
监听节点数据的变化
get path [watch]
监听子节点增减的变化
ls path [watch]
watch监听有不同的类型，有监听状态的stat ，内容的get，目录结构的ls。
另外命令使用一次，只监听一次，监听到了就打印内容，打印结束就退出了监听。
```



##  ZAB 协议 & Paxos 算法

#### Ⅰ.ZAB 协议 & Paxos 算法

Paxos 算法可以说是 ZooKeeper 的灵魂了。但是，ZooKeeper 并没有完全采用 Paxos 算法 ，而是使用 ZAB 协议作为其保证数据一致性的核心算法。

另外，在 ZooKeeper 的官方文档中也指出，ZAB 协议并不像 Paxos 算法那样，是一种通用的分布式一致性算法，它是一种特别为 ZooKeeper 设计的崩溃可恢复的原子消息广播算法。

#### Ⅱ.ZAB 协议介绍

ZAB（ZooKeeper Atomic Broadcast 原子广播）协议是为分布式协调服务 ZooKeeper 专门设计的一种支持崩溃恢复的原子广播协议。

在 ZooKeeper 中，主要依赖 ZAB 协议来实现分布式数据一致性，基于该协议，ZooKeeper 实现了一种主备模式的系统架构来保持集群中各个副本之间的数据一致性。

#### Ⅲ.ZAB 协议两种基本的模式

ZAB 协议包括两种基本的模式，分别是崩溃恢复和消息广播。

当整个服务框架在启动过程中，或是当 Leader 服务器出现网络中断、崩溃退出与重启等异常情况时，ZAB 协议就会进入恢复模式并选举产生新的 Leader 服务器。

当选举产生了新的 Leader 服务器，同时集群中已经有过半的机器与该 Leader 服务器完成了状态同步之后，ZAB 协议就会退出恢复模式。

其中，所谓的状态同步是指数据同步，用来保证集群中存在过半的机器能够和 Leader 服务器的数据状态保持一致。

当集群中已经有过半的 Follower 服务器完成了和 Leader 服务器的状态同步，那么整个服务框架就可以进人消息广播模式了。

当一台同样遵守 ZAB 协议的服务器启动后加入到集群中时，如果此时集群中已经存在一个 Leader 服务器在负责进行消息广播。

那么新加入的服务器就会自觉地进人数据恢复模式：找到 Leader 所在的服务器，并与其进行数据同步，然后一起参与到消息广播流程中去。

ZooKeeper 设计成只允许唯一的一个 Leader 服务器来进行事务请求的处理。

Leader 服务器在接收到客户端的事务请求后，会生成对应的事务提案并发起一轮广播协议。

而如果集群中的其他机器接收到客户端的事务请求，那么这些非 Leader 服务器会首先将这个事务请求转发给 Leader 服务器。

zookeeper进行ack时采用过半机制,不用等待所有的follower都ack

zookeeper如果写入数据时因为网络或其他原因没有提交到从节点，那么从节点在下次启动时会自动从leader同步

普通的从节点虽然会提高读请求的并发能力，但是对于写请求由于投票机制会变慢

### 思考

一下是一些zk的常见面试题

1. Zookeeper工作原理

> Zookeeper 的核心是原子广播，这个机制保证了各个Server之间的同步。实现这个机制的协议叫做Zab协议。Zab协议有两种模式，它们分别是恢复模式（选主）和广播模式（同步）。当服务启动或者在领导者崩溃后，Zab就进入了恢复模式，当领导者被选举出来，且大多数Server完成了和 leader的状态同步以后，恢复模式就结束了。状态同步保证了leader和Server具有相同的系统状态。此时系统的不可用的，所以ZooKeeper采用的是CP原则

2. zookeeper中的观察者节点有什么作用？

> 首先观察者节点个floower节点最大的区别就是观察着节点不参与选举，试想下问什么会这样设计，其实也很好理解，再zk集群中，添加一个节点，就会提升对应的并发能力，但是集群在选举master的时候是不是也对应多了一个节点进行投票选举，这就导致选举投票机制变慢，所以观察者节点就是单纯的提高读请求并发能力，对于写请求基本没有影响(zk写的时候会将数据包发给观察者节点)。这一点和rabbitMQ中的内存节点磁盘节点，ElasticSearch中的data节点非data节点设计思路如出一辙。

3. zookeeper是如何满足cap理论的

> zk中使用的cap中的cp 即满足想一致性

4. zookeeper集群中为什么选择奇数个数节点

> 因为zk的过半机制，比如分别有3和节点和4个节点的zk集群，他们都是最多挂掉一个就不可用了，所以。。。。

5. Zookeeper 下 Server工作状态

> 每个Server在工作过程中有三种状态：
>
> looking：当前Server不知道leader是谁，正在搜寻
>
> leading：当前Server即为选举出来的leader
>
> following：leader已经选举出来，当前Server与之同步

6. zookeeper是如何保证事务的顺序一致性的？

> 两段协议，首先会向其他的server发出事务执行请求，如果超过半数的机器都能执行并且能够成功，那么就会开始执行。

7. zookeeper实现分布式锁的原理

- 首先要有一个main()线程

- 在main线程中创建zookeeper客户端,这时就会创建两个线程,一个负责网络连接通信(connect),一个负责监听(listener)。

- 通过connect线程将注册的监听事件发送给zookeeper。

- 在zookeeper的注册监听器列表中将注册的监听事件添加到列表中。

- zookeeper监听到有数据或路径变化,就会将这个消息发送给listener线程。
- listener线程内部调用了process()方法

![](https://upload-images.jianshu.io/upload_images/8387919-ca9f906cd7e22be2.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)