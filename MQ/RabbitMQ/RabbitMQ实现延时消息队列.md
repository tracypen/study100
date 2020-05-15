> 延时消息队列的应用场景颇为广泛，比如订单超时未支付的自动取消，超时未评价的自动好评等等。本文将介绍`RabbitMQ`中两种常用方式来实现延时消息队列

### 一、安装延时插件

下载插件  [地址](https://www.rabbitmq.com/community-plugins.html)

找到 `rabbitmq_delayed_message_exchange`延时插件，选择对应的版本进行下载

拷贝插件文件到`/usr/lib/rabbitmq/lib/rabbitmq_server-3.6.5/plugins`下
```
rabbitmq-plugins enable rabbitmq_delayed_message_exchange
```
![](https://upload-images.jianshu.io/upload_images/8387919-de48c7ca60994ea2.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)


![](https://upload-images.jianshu.io/upload_images/8387919-e9d99fd3d63be2a2.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)

![](https://upload-images.jianshu.io/upload_images/8387919-981035bfe80c5a39.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)

![](https://upload-images.jianshu.io/upload_images/8387919-de33afb3b99cdf14.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)

### 二、死信队列

**原理**：让指定消息投递后自动过期从而进入死信队列中，通过监听消费该死信队列从而完成延时消息。







https://www.jianshu.com/p/0e0dfae18970

死信队列 https://www.jianshu.com/p/be0361e222ac

