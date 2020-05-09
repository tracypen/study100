> head插件是日常使用ElasticSearch中常用的一款插件，可以很方便的对索引数据进行管理和查询

##### 1.安装head插件服务（nodejs服务）

- git clone `git://github.com/mobz/elasticsearch-head.git`
- `cd elasticsearch-head`
- `npm install`
- `npm run start`
- 设置elasticsearch允许跨域访问
    ```
    在elasticsearch.yml后追加
    http.cors.enabled: true
    http.cors.allow-origin: "*"
    ```


- open http://localhost:9100/
##### 2.通过docker方式运行
- for Elasticsearch 5.x:
`docker run -p 9100:9100  mobz/elasticsearch-head:5`

- for Elasticsearch 2.x: `docker run -p 9100:9100 mobz/elasticsearch-head:2`
- for Elasticsearch 1.x: `docker run -p 9100:9100 mobz/elasticsearch-head:1`
- open http://localhost:9100/

##### 3.安装chrome插件

> 可以看到以上两种方式都是通过服务的方式运行的，缺点就是占用目标机器的资源，而且当有多个es数据源的话，往往需要单独搭建，因此chrome插件方式也是比较好的一种选择

- 打开chrome 设置--->更多工具--->拓展程序 打开chrome插件市场

- 搜索 head 

![](https://upload-images.jianshu.io/upload_images/8387919-7e1203f421f01e69.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

直接进行安装即可

##### 2.配置

首先需要配置elasticsearch允许跨域

```properties
http.cors.enabled: true
http.cors.allow-origin: "*"
```



##### 3.使用

- 打开上一步安装的head插件

- 输入连接es的地址，默认http://localhost:9200/

![](https://upload-images.jianshu.io/upload_images/8387919-cbdd2fc5b94528fd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](https://upload-images.jianshu.io/upload_images/8387919-31d427fd072d811a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 参考

- [head插件 GitHub地址](https://github.com/mobz/elasticsearch-head.git)

