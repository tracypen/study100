### RabbitMQ实现消息投递可靠性

> rabbitmq的消息可靠性投递提现在两个方面，分别是生产者端和消费者端的可靠性控制

#### 1.生产者端

> 生产端可靠性一般通过confirm消息确认和Return消息机制

##### 1.1  confirm

>  当生产者发送消息后，消息到达broker后就会进行confim回调，在回到中根据投递标签（Tag）进行消息的唯一确定。根据ack结果分为两种

- true 标识消息正常投递，被broker接受
- false 消息为正常投递 （可能因为内存、磁盘等原因导致）

##### 1.2 Return

> 当消息未找到exchange或routingkey不正确消息最终路由错误，这两种情况都会导致消息不可达，最终执行return回调  需要开启 `spring.rabbitmq.template.mandatory=true`

#### 2.消费者端

> 消费端的ack是控制消息是否从broker进行正常消费，可以进行三种确认操作

- ack
- nack
- reject

其中`basicReject`、 `basicNack`的区别参考：https://blog.csdn.net/fly_leopard/article/details/102821776

*注意*：要设置 关闭自动ack模式 改为手动`MANUAL`

### 一、生产端可靠性解决方案

> 消息落库，对消息状态进行打标

> 实现本地消息表，对消息的状态进行标记，更改，定期抓取非正常状态的消息进行重新投递或补偿

![](https://upload-images.jianshu.io/upload_images/8387919-f0e5b9744324ee8f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 1.业务数据和消息数据同时写入数据库，此时消息状态为0标识投递中

```java
public static final String ORDER_SENDING = "0";
	
public static final String ORDER_SEND_SUCCESS = "1";

public static final String ORDER_SEND_FAILURE = "2";
```



- 2.上一步确保成功后，生产端发送消息到broker
- 3.broker通过confirm机制，回调confirm方法

```java
	final ConfirmCallback confirmCallback = new RabbitTemplate.ConfirmCallback() {
		@Override
		public void confirm(CorrelationData correlationData, boolean ack, String cause) {
			System.err.println("correlationData: " + correlationData);
			String messageId = correlationData.getId();
			if(ack){
				//如果confirm返回成功 则进行更新
				brokerMessageLogMapper.changeBrokerMessageLogStatus(messageId, Constants.ORDER_SEND_SUCCESS, new Date());
			} else {
				//失败则进行具体的后续操作:重试 或者补偿等手段
				System.err.println("异常处理...");
			}
		}
	};

```

其中`brokerMessageLogMapper.changeBrokerMessageLogStatus`就是更改消息状态为投递成功

```xml
  <update id="changeBrokerMessageLogStatus" >
    update broker_message_log bml
    set bml.status = #{status,jdbcType=VARCHAR},
      	bml.update_time = #{updateTime, jdbcType=TIMESTAMP}
    where bml.message_id = #{messageId,jdbcType=VARCHAR}  
  </update>
```



- 4.上一步如果成功，更改消息状态为1 代表消息投递成功，如果失败可以进行重试
- 5.通过定时任务抓取消息状态为0的消息，并且发送时间至少为5分钟以前的（防止新消息发送中导致误判）消息，进行重新发送

```java
@Scheduled(initialDelay = 3000, fixedDelay = 10000)
	public void reSend(){
		System.err.println("---------------定时任务开始---------------");
		//pull status = 0 and timeout message 
		List<BrokerMessageLog> list = brokerMessageLogMapper.query4StatusAndTimeoutMessage();
		list.forEach(messageLog -> {
			if(messageLog.getTryCount() >= 3){
				//update fail message 
				brokerMessageLogMapper.changeBrokerMessageLogStatus(messageLog.getMessageId(), Constants.ORDER_SEND_FAILURE, new Date());
			} else {
				// resend 
				brokerMessageLogMapper.update4ReSend(messageLog.getMessageId(),  new Date());
				Order reSendOrder = FastJsonConvertUtil.convertJSONToObject(messageLog.getMessage(), Order.class);
				try {
					rabbitOrderSender.sendOrder(reSendOrder);
				} catch (Exception e) {
					e.printStackTrace();
					System.err.println("-----------异常处理-----------");
				}
			}			
		});
		
	}
```



- 6. 每次重新发送的时候，更改消息表中的重试次数+1

```xml
brokerMessageLogMapper.update4ReSend()

<update id="update4ReSend" >
    update broker_message_log bml
    set bml.try_count = bml.try_count + 1,
      bml.update_time = #{updateTime, jdbcType=TIMESTAMP}
    where bml.message_id = #{messageId,jdbcType=VARCHAR}
  </update>
```

- 7. 判断重试次数大于上限时比如3次 更改消息状态为2 标识消息投递失败（这个一般就是机器或程序不可抗因素，需要人工补偿了）

```
brokerMessageLogMapper.changeBrokerMessageLogStatus()方法就是将消息状态改为投递失败，最终进行补偿
```

  另外可以利用return机制进行不可达消息的追踪，具体规则根据业务而定

二、消费端可靠性解决方案

- 由于业务异常，可以进行日志记录，然后进行补偿
- 由于服务器宕机等严重问题，那么就需要手动ack保证消费成功（一般都是手动ack）
- 可以根据业务设置消息是否重回队列

```java
	@RabbitHandler
	public void onOrderMessage(@Payload com.bfxy.springboot.entity.Order order, Channel channel, 
			@Headers Map<String, Object> headers) throws Exception {
		Long deliveryTag = (Long)headers.get(AmqpHeaders.DELIVERY_TAG);
		//手工ACK
		channel.basicAck(deliveryTag, false);
	}
```

