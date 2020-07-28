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

Time To Live(TTL) 特性

RabbitMQ可以针对Queue设置x-expires 或者 针对Message设置 x-message-ttl，来控制消息的生存时间，如果超时(两者同时设置以最先到期的时间为准)，则消息变为dead letter(死信)
RabbitMQ针对队列中的消息过期时间有两种方法可以设置。
A: 通过队列属性设置，队列中所有消息都有相同的过期时间。
B: 对消息进行单独设置，每条消息TTL可以不同。

**如果同时使用，则消息的过期时间以两者之间TTL较小的那个数值为准。**消息在队列的生存时间一旦超过设置的TTL值，就成为dead letter

```
/**
     * 队列，消息死信后，投递到deadExchangeName
     */
    @Bean("fanoutTimeQueue")
    Queue fanoutTimeQueue(){
        // 将普通队列绑定到死信队列交换机上
        Map<String, Object> args = new HashMap<>(2);
        args.put(DEAD_LETTER_QUEUE_KEY, deadExchangeName);
        args.put(DEAD_LETTER_ROUTING_KEY, deadRoutingKey);
        Queue queue = new Queue("fanoutTimeQueue", true, false, false, args);
        return queue;
    }
    /**
     * 交换机
     */
    @Bean("fanoutTimeExchange")
    FanoutExchange fanoutTimeExchange() {
        return new FanoutExchange("fanoutTimeExchange");
    }
 
    /**
     * 绑定交换机fanoutTimeExchange与队列fanoutTimeQueue
     * @param fanoutTimeQueue 创建的队列
     * @param fanoutTimeExchange 创建的交换机
     * @return
     */
    @Bean
    Binding bindingTimeExchange(Queue fanoutTimeQueue, FanoutExchange fanoutTimeExchange{
        return BindingBuilder.bind(fanoutTimeQueue).to(fanoutTimeExchange);
    }
```



发送消息

```java
@RequestMapping(value = "/sendTimeMessage")
    @ResponseBody
    public String sendTimeMessage() {
        JSONObject jsonObject = new JSONObject();
        jsonObject.put("email", "xx@163.com");
        jsonObject.put("timestamp", 0);
        String jsonString = jsonObject.toJSONString();
        System.out.println("jsonString:" + jsonString);
 
        CorrelationData cData = new CorrelationData("email-" + new Date().getTime());
 
        rabbitTemplate.setConfirmCallback((correlationData,isack,cause) -> {
            System.out.println("本次消息的唯一标识是:" + correlationData);
            System.out.println("是否存在消息拒绝接收？" + isack);
            if(isack == false){
                System.out.println("消息拒绝接收的原因是:" + cause);
            }else{
                System.out.println("消息发送成功");
            }
        });
 
        rabbitTemplate.setReturnCallback( (message,i,s,s1,s2) -> {
            System.out.println("err code :" + i);
            System.out.println("错误消息的描述 :" + s);
            System.out.println("错误的交换机是 :" + s1);
            System.out.println("错误的路右键是 :" + s2);
        });
 
 
        // 声明消息处理器 这个对消息进行处理 可以设置一些参数 对消息进行一些定制化处理 我们这里 来设置消息的编码 以及消息的过期时间
        // 因为在.net 以及其他版本过期时间不一致 这里的时间毫秒值 为字符串
        MessagePostProcessor messagePostProcessor = message -> {
            MessageProperties messageProperties = message.getMessageProperties();
            // 设置编码
            messageProperties.setContentEncoding("utf-8");
            // 设置过期时间10*1000毫秒
            messageProperties.setExpiration("10000");
            return message;
        };
        // fanoutTimeExchange 发送消息 10*1000毫秒后过期 形成死信,具体的时间可以根据自己的业务指定
        rabbitTemplate.convertAndSend("fanoutTimeExchange", "", jsonString, messagePostProcessor, cData);
 
        SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        System.out.println(simpleDateFormat.format(new Date()));
        return "success";
    }
```



总结上面的方式：经过测试，我们可以发现，当我们先增加一条过期时间大(10000)的A消息进入，之后再增加一个过期时间小的(1000)消息B,并没有出现想象中的B消息先被消费，A消息后被消费，而是出现了当10000过去的时候，AB消息同时被消费，也就是B消息的消费被阻塞了。

为什么会出现这样的现象呢？
我们知道利用TTL DLX特性实现的方式，实际上在第一个延时队列C里面设置了dlx，生产者生产了一条带ttl的消息放入了延时队列C中，等到延时时间到了，延时队列C中的消息变成了死信，根据延时队列C中设置的dlx的exchange的转发规则，转发到了实际消费队列D中，当该队列中的监听器监听到消息时就会正式开始消费。那么实际上延时队列中的消息也是放入队列中的，队列满足先进先出，而延时大的消息A还没出队，所以B消息也不能顺利出队。

https://www.jianshu.com/p/0e0dfae18970

死信队列 https://www.jianshu.com/p/be0361e222ac

