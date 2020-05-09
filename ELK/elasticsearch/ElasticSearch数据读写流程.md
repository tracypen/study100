### ElasticSearch数据读写流程

##### ElasticSearch写入数据

1. 客户端选择一个Node发送请求过去，这个node就成为coordinating node(协调节点)

2. coordinating node对 document进行路由，将请求转发到对应的node(有primary shard)
  3.实际的node上的primary shard处理请求，然后将数据同步到replica node

3. coordinating node 发现 pimary node 和 replica node都处理完毕后，就返回响应结果给客户端

   ![](https://upload-images.jianshu.io/upload_images/8387919-a252bc4550ca924e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### ElasticSearch读取数据

根据文档id进行路由到对应的节点机器上（相当于请求转发）

- 客户端发送请求到**任意**一个 node，成为 `coordinate node`。
- `coordinate node` 对 `doc id` 进行路由，将请求转发到对应的 node，实际集群中可能该数据对应的shard有多个（主、备），根据负载轮询策略在对应的shard上进行数据的读取。
- 接收请求的 node 返回 document 给 `coordinate node`。
- `coordinate node` 返回 document 给客户端。

##### 索引数据过程

> 核心就是两个阶段 query和fetch

- 客户端连接到 任一节点（`coordinate node`）。

- 协调节点将搜索请求转发到**所有**的 shard 对应的 `primary shard` 或 `replica shard`，都可以。

- query phase：每个 shard 将自己的搜索结果（其实就是一些 `doc id`）返回给协调节点，由协调节点进行数据的合并、排序、分页等操作，产出最终结果。

- fetch phase：接着由协调节点根据 `doc id` 去各个节点上**拉取实际**的 `document` 数据，最终返回给客户端。

##### 删除更新索引

>  如果是删除操作，commit 的时候会生成一个 `.del` 文件，里面将某个 doc 标识为 `deleted` 状态，那么搜索的时候根据 `.del` 文件就知道这个 doc 是否被删除了。

如果是更新操作，就是将原来的 doc 标识为 `deleted` 状态，然后新写入一条数据。

buffer 每次 refresh 一次，就会产生一个 `segment file`，所以默认情况下是 1 秒钟一个 `segment file`，这样下来 `segment file` 会越来越多，此时会定期执行 merge。每次 merge 的时候，会将多个 `segment file` 合并成一个，同时这里会将标识为 `deleted` 的 doc 给**物理删除掉**，然后将新的 `segment file` 写入磁盘，这里会写一个 `commit point`，标识所有新的 `segment file`，然后打开 `segment file` 供搜索使用，同时删除旧的 `segment file`。