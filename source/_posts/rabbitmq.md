title: RabbitMq相关
tags:
  - java
  - middle-ware
categories: 谈技论术
date: 2018/06/21 13:25
---

### 基本组成
+ 图示
```flow
cf=>start: ConnectionFactory
conn=>operation: Connection
c=>operation: Channel
broker=>operation: Broker
io=>inputoutput: consist of: exchange, routingkey, queue
bd=>subroutine: bindings
cf->conn->c->broker(right)->io(bottom)->bd
```
>
其中：
channel，信道，虚拟连接概念，通过connection获取，java api中该对象用于创建及操作其他对象，包括定义Queue、定义Exchange、绑定Queue与Exchange、发布msg、msg的ack/nack等处理

+ 常用exchange
 - direct(routingkey = queueName),
 - fanout(分发/广播，绑定在此exchange上的队列，均可收到message),
 - topic 正则匹配，`routingkey.match(queueName)`, eg: `nova.*,  nova.wind.#`

direct(routingkey = queueName),
fanout(分发/广播，绑定在此exchange上的队列 均可收到message),
topic 正则匹配(routingkey.match(queueName), eg: nova.*,  nova.wind.#)

---
### RabbitMqConfig
+ 指定queue、exchange、routingkey，可自动在RabbitMq中创建相应的queue等，并实现绑定

```
@Bean("directQueue")
   public Queue directQueue(){
   return new Queue(rabbitCustomizedProperties.getDirectQueue(), true, false, false);
}

@Bean
public DirectExchange directExchange(){
   return new DirectExchange(rabbitCustomizedProperties.getDirectExchange(), true, false);
}
@Bean
public Binding bind(Queue directQueue, DirectExchange directExchange){
   return BindingBuilder.bind(directQueue).to(directExchange)
         .with(rabbitCustomizedProperties.getDirectRoutingkey());
```
+ RabbitMq消息序列化：默认是SimpleMessageConverter(jdk)，以下template中指定为Jackson2JsonConverter实现json序列化
  
```
@Bean
public MessageConverter jackson2JsonConverter(){
   return new Jackson2JsonMessageConverter();
}
@Bean
public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory, MessageConverter jackson2JsonConverter){
   RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory);
   rabbitTemplate.setMessageConverter(jackson2JsonConverter);
   return rabbitTemplate;
}
```
+ consumer
 - 注解: `@RabbitListener(queues= "queueName"),   @RabbitHandler`
 - 实现接口 `MessageListener`， 实现`onMessage(Message message)`方法
   or 实现接口 `ChannelAwareMessageListener`， 实现`onMessage (Message message, Channel channel)`方法，此接口可以根据channel实现手动ack/nack

---

### 延迟队列 delayed queue
+ 基本设置
 - 队列上设置TTL
 ![延迟队列](https://gitee.com/bearwind/image_host/raw/master/2018-06/delay_queue.png)
 - message设置TTL
   ``setExpiration(300000)``
 - 指定死信交换器 DLX(DeadLetterExchange)，当消息变为死信(DL)时，自动publish到一个DLX里进行相关处理，internal设置为no， 否则消息无法消费，仅用于exchange间的绑定。
 ![死信交换器](https://gitee.com/bearwind/image_host/raw/master/2018-06/delay_ex.png)
+ 应用场景简述
  淘宝下订单，producer发消息至 delayedQueue(A)， 队列过期时间30m，30m后过期转发至isPaidQueue(B)
    1. 客户已支付，consumer消费queue B消息，判断该订单状态为paid，不做处理；
    2. 客户未支付，consumer消费 queue B消息，判断该订单状态为unpaid，则将该订单取消，库存恢复。

    ```flow
    repo=>start: 库存
    order=>operation: 下单
    prod=>operation: producer
    QA=>subroutine: delayedQueue A (30m 过期)
    QB=>subroutine: isPaidQueue B
    paidOrNot=>condition: is paid or not?
    io=>inputoutput: 未支付，入库存
    e=>end
    repo->order->prod()->QA->QB(right)->paidOrNot
    paidOrNot(yes)->e
    paidOrNot(no)->io(right)->order
    ```
+ process
   ![死信-process](https://gitee.com/bearwind/image_host/raw/master/2018-06/delay-ex-process.png)
+ code

```
//delay queue
@Bean
public DirectExchange delayExchange(){
   return new DirectExchange(rcps.getDelayExchange(),true, false);
}
@Bean
public Queue delayQueue(){
   Map<String, Object> args = new HashMap<>();
   args.put("x-dead-letter-exchange", rcps.getDeadLetterExchange());
   args.put("x-dead-letter-routing-key", rcps.getDelayRoutingkey());
   //message过期时间
   args.put("x-message-ttl", rcps.getTimeToLive());
   //args.put("x-expires",1000); 队列过期时间
   return new Queue(rcps.getDelayQueue(), true, false, false, args);
}
@Bean
public Binding bindDelay(DirectExchange delayExchange, Queue delayQueue){

   return BindingBuilder.bind(delayQueue).to(delayExchange).with(rcps.getDelayRoutingkey());
}

@Bean
public DirectExchange dlx(){
   return new DirectExchange(rcps.getDeadLetterExchange(),true, false);
}
@Bean
public Queue dlQueue(){
   return new Queue(rcps.getDeadLetterQueue(), true);
}

@Bean
public Binding bindDl(DirectExchange dlx, Queue dlQueue){
   //with delay routingkey 需与延迟队列key保持一致 否则无法接收到死信 或不指定 默认一致
   return BindingBuilder.bind(dlQueue).to(dlx).with(rcps.getDelayRoutingkey());
```
