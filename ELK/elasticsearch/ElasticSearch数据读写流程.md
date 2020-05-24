### ElasticSearch数据处理流程----面试必问

##### 文档路由的过程

es通过如下公式进行文档的路由

`shard = hash(routing) % number_of_primary_shards`

- hash算法保证将文档均匀的分散到分片中
- routing默认是文档id，也可以自行制定
- number_of_primary_shards 主分片数

##### ElasticSearch写入数据

1. 客户端选择一个Node发送请求过去，这个node就成为coordinating node(协调节点)

2. coordinating node对 document进行路由，将请求转发到对应的node(有primary shard)
   
3. 实际的node上的primary shard处理请求，然后将数据同步到replica node

4. coordinating node 发现 pimary node 和 replica node都处理完毕后，就返回响应结果给客户端

   ![](https://upload-images.jianshu.io/upload_images/8387919-a252bc4550ca924e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### ElasticSearch读取数据

根据文档id进行路由到对应的节点机器上（相当于请求转发）

- 客户端发送请求到**任意**一个 node，成为 `coordinate node`。
- `coordinate node` 对 `doc id` 进行路由，将请求转发到对应的 node，实际集群中可能该数据对应的shard有多个（主、备），根据负载轮询策略在对应的shard上进行数据的读取。
- 接收请求的 node 返回 document 给 `coordinate node`。
- `coordinate node` 返回 document 给客户端。

##### 文档批量创建的流程

1. 客户端选择一个Node发送bulk请求，此节点为coordinate node
2. coordinate node通过routing算法找到所有对应的shard，然后同时将对用的请求转发至主shard
3. 其余过程与单个写入基本一直

##### match query流程

![](https://upload-images.jianshu.io/upload_images/8387919-e4b0b0fd12fa972d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 索引数据过程

> 两个阶段 query和fetch

- 客户端连接到 任一节点（`coordinate node`）。

- 协调节点将搜索请求转发到**所有**的 shard 对应的 `primary shard` 或 `replica shard`，都可以。

- query phase：每个 shard 将自己的搜索结果（其实就是一些 `doc id`）返回给协调节点，由协调节点进行数据的合并、排序、分页等操作，产出最终结果。

- fetch phase：接着由协调节点根据 `doc id` 去各个节点上**拉取实际**的 `document` 数据，最终返回给客户端。

##### 删除更新文档流程

​	segment一旦生成就不能更改，Lucene专门维护了一个`.del` 文件，内部内陆着所有已经删除的文档，如果是删除操作，就将里面将某个 doc 标识为 `deleted` 状态，那么搜索的时候根据 `.del` 文件就知道这个 doc 是否被删除了。将已经删除的文档过滤掉。

如果是更新操作，就是将原来的 doc 标识为 `deleted` 状态，然后新写入一条数据。

##### 文档refresh流程

segment写入磁盘的过程十分耗时，可以借助文件系统缓存的特性，先将segment在缓存中创建，并开放查询来确保近实时性，该过程称为refresh。在refresh之前文档先会存储在一个buffer中，refresh时会将文档中的buffer清空，并生成segment，es默认每一秒进行一次refresh，因此文档的实时性为1秒，这也是es近实时性的原因。

![](https://upload-images.jianshu.io/upload_images/8387919-a66f2d47af271fc0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 文档translog流程

如果内存中的segment还没还得及写入磁盘中，服务器宕机了，name其中的文档就无法恢复了，为了解决这个问题，es引入了reanslog机制，当写入文档到buffer时，同时将该操作写入translog，而translog文件会即时写入磁盘中（fsync），es重启后会通过translog文件进行数据恢复

![](https://upload-images.jianshu.io/upload_images/8387919-27dd2536d27703a9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 文档flush流程

flush负责将内存中的segment写入磁盘，并将index buffer清空，更新commit point并写入磁盘，同时删除旧的translog文件

![](https://upload-images.jianshu.io/upload_images/8387919-d9caffeea8e70466.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### Segment merging流程

由于每次进行一次refresh，这样下来 `segment file` 会越来越多，此时会定期执行 merge。每次 merge 的时候，会将多个 `segment file` 合并成一个，同时这里会将标识为 `deleted` 的 doc 给**物理删除掉**，然后将新的 `segment file` 写入磁盘，这里会写一个 `commit point`，标识所有新的 `segment file`，然后打开 `segment file` 供搜索使用，同时删除旧的 `segment file`。

#### 思考：

1. 为什么es删除更新的时候不直接删除，而是通过.del文件进行删除

   因为es的倒排索引是不可变的，一旦生成倒排索引就不会再变，这也是es高性能的原因之一
   
2. 倒排索引不可变的优缺点

   **优点：**

   - 不用考虑并发写文件的问题，杜绝锁机制带来的性能消耗
   - 由于文件不可更改，可以充分利用文件系统缓存，只需要载入缓存一次，只要内存足够，就会命中缓存，性能高
   - 有利于对文件进行压缩存储，节省磁盘和内存空间

   **缺点：**

   - 写入新文档时，必须重新构建索引然后替换老的索引文件，新文档才能被检索，导致文档实时性差。

**声明：**文中内容为自学慕课网笔记，部分图片为课程中资料截图，仅供学习记录使用。