---
layout: post
title: springboot整合rabbitmq实现延迟队列
date: 2023-04-18
tags: [mq]
description: 5.1调休周日上班摸鱼
---


不了解rabbitmq的可以看看我上篇文章[rabbitmq入门](https://juejin.cn/post/7222431754476060730)。

> 欢迎关注个人公众号【好好学技术】交流学习

## 如何保证消息不丢失

rabbitmq消息投递路径

> 生产者->交换机->队列->消费者

总的来说分为三个阶段。

*   1.生产者保证消息投递可靠性。
*   2.mq内部消息不丢失。
*   3.消费者消费成功。

### 什么是消息投递可靠性

简单点说就是消息百分百发送到消息队列中。\
我们可以开启confirmCallback\
生产者投递消息后，mq会给生产者一个ack.根据ack,生产者就可以确认这条消息是否发送到mq.

开启confirmCallback\
修改配置文件

```yaml
#NONE：禁用发布确认模式，是默认值，CORRELATED：发布消息成功到交换器后会触发回调方法
spring:
  rabbitmq:
    publisher-confirm-type: correlated
```

测试代码

```java
@Test  
public void testConfirmCallback() throws InterruptedException {  
    rabbitTemplate.setConfirmCallback(new RabbitTemplate.ConfirmCallback() {  
    /**  
    *  
    * @param correlationData 配置  
    * @param ack 交换机是否收到消息，true是成功，false是失败  
    * @param cause 失败的原因  
    */  
    @Override  
    public void confirm(CorrelationData correlationData, boolean ack, String cause) {  
        System.out.println("confirm=====>");  
        System.out.println("confirm==== ack="+ack);  
        System.out.println("confirm==== cause="+cause);  
        //根据ACK状态做对应的消息更新操作 TODO  
    }  
    });  
    rabbitTemplate.convertAndSend(RabbitMQConfig.EXCHANGE_NAME,"ikun.mei", "鸡你太美");  
    Thread.sleep(10000);  
}
```

通过returnCallback保证消息从交换器发送到队列成功。
修改配置文件

```yaml

spring:
  rabbitmq:
    #开启returnCallback
    publisher-returns: true
    #交换机处理消息到路由失败，则会返回给生产者
    template:
      mandatory: true
```

测试代码

```java
@Test  
void testReturnCallback() {  
    //为true,则交换机处理消息到路由失败，则会返回给生产者 配置文件指定，则这里不需指定  
    rabbitTemplate.setMandatory(true);  
    //开启强制消息投递（mandatory为设置为true），但消息未被路由至任何一个queue，则回退一条消息  
    rabbitTemplate.setReturnsCallback(returned -> {  
        int code = returned.getReplyCode();  
        System.out.println("code="+code);  
        System.out.println("returned="+ returned);  
    });  
    rabbitTemplate.convertAndSend(RabbitMQConfig.EXCHANGE_NAME,"123456","测试returnCallback");  
}
```

消费者消费消息时需要通过ack手动确认消息已消费。

修改配置文件

```yaml
spring:
  rabbitmq:
    listener:  
      simple:  
        acknowledge-mode: manual
```

编写测试代码

```java
@RabbitHandler  
public void consumer(String body, Message message, Channel channel) throws IOException {  
    long msgTag = message.getMessageProperties().getDeliveryTag();  
    System.out.println("msgTag="+msgTag);  
    System.out.println("message="+ message);  
    System.out.println("body="+body);  

    //成功确认，使用此回执方法后，消息会被 rabbitmq broker 删除  
    channel.basicAck(msgTag,false);  
    // channel.basicNack(msgTag,false,true);  
  
}
```

deliveryTags是消息投递序号，每次消费消息或者消息重新投递后，deliveryTag都会增加

## ttl死信队列

### 什么是死信队列

没有被及时消费的消息存放的队列

### 消息有哪几种情况成为死信

*   消费者拒收消息 **（basic.reject/ basic.nack）** ，并且没有重新入队 **requeue=false**
*   消息在队列中未被消费，且超过队列或者消息本身的过期时间**TTL(time-to-live)**
*   队列的消息长度达到极限
*   结果：消息成为死信后，如果该队列绑定了死信交换机，则消息会被死信交换机重新路由到死信队列

死信队列经常用来做延迟队列消费。

## 延迟队列

生产者投递到mq中并不希望这条消息立马被消费，而是等待一段时间后再去消费。

## springboot整合rabbitmq实现订单超时自动关闭

```java
package com.fandf.test.rabbit;  
  
import org.springframework.amqp.core.*;  
import org.springframework.beans.factory.annotation.Qualifier;  
import org.springframework.context.annotation.Bean;  
import org.springframework.context.annotation.Configuration;  
  
import java.util.HashMap;  
import java.util.Map;  
  
/**  
* @author fandongfeng  
* @date 2023/4/15 15:38  
*/  
@Configuration  
public class RabbitMQConfig {  
  
    /**  
    * 订单交换机  
    */  
    public static final String ORDER_EXCHANGE = "order_exchange";  
    /**  
    * 订单队列  
    */  
    public static final String ORDER_QUEUE = "order_queue";  
    /**  
    * 订单路由key  
    */  
    public static final String ORDER_QUEUE_ROUTING_KEY = "order.#";  

    /**  
    * 死信交换机  
    */  
    public static final String ORDER_DEAD_LETTER_EXCHANGE = "order_dead_letter_exchange";  
    /**  
    * 死信队列 routingKey  
    */  
    public static final String ORDER_DEAD_LETTER_QUEUE_ROUTING_KEY = "order_dead_letter_queue_routing_key";  

    /**  
    * 死信队列  
    */  
    public static final String ORDER_DEAD_LETTER_QUEUE = "order_dead_letter_queue";  


    /**  
    * 创建死信交换机  
    */  
    @Bean("orderDeadLetterExchange")  
    public Exchange orderDeadLetterExchange() {  
        return new TopicExchange(ORDER_DEAD_LETTER_EXCHANGE, true, false);  
    }  

    /**  
    * 创建死信队列  
    */  
    @Bean("orderDeadLetterQueue")  
    public Queue orderDeadLetterQueue() {  
        return QueueBuilder.durable(ORDER_DEAD_LETTER_QUEUE).build();  
    }  

    /**  
    * 绑定死信交换机和死信队列  
    */  
    @Bean("orderDeadLetterBinding")  
    public Binding orderDeadLetterBinding(@Qualifier("orderDeadLetterQueue") Queue queue, @Qualifier("orderDeadLetterExchange")Exchange exchange) {  
        return BindingBuilder.bind(queue).to(exchange).with(ORDER_DEAD_LETTER_QUEUE_ROUTING_KEY).noargs();  
    }  


    /**  
    * 创建订单交换机  
    */  
    @Bean("orderExchange")  
    public Exchange orderExchange() {  
        return new TopicExchange(ORDER_EXCHANGE, true, false);  
    }  

    /**  
    * 创建订单队列  
    */  
    @Bean("orderQueue")  
    public Queue orderQueue() {  
        Map<String, Object> args = new HashMap<>(3);  
        //消息过期后，进入到死信交换机  
        args.put("x-dead-letter-exchange", ORDER_DEAD_LETTER_EXCHANGE);  

        //消息过期后，进入到死信交换机的路由key  
        args.put("x-dead-letter-routing-key", ORDER_DEAD_LETTER_QUEUE_ROUTING_KEY);  

        //过期时间，单位毫秒  
        args.put("x-message-ttl", 10000);  

        return QueueBuilder.durable(ORDER_QUEUE).withArguments(args).build();  
    }  

    /**  
    * 绑定订单交换机和队列  
    */  
    @Bean("orderBinding")  
    public Binding orderBinding(@Qualifier("orderQueue") Queue queue, @Qualifier("orderExchange")Exchange exchange) {  
        return BindingBuilder.bind(queue).to(exchange).with(ORDER_QUEUE_ROUTING_KEY).noargs();  
    }  
  
  
}
```

消费者

```java
package com.fandf.test.rabbit;  
  
import cn.hutool.core.date.DateUtil;  
import com.rabbitmq.client.Channel;  
import org.springframework.amqp.core.Message;  
import org.springframework.amqp.rabbit.annotation.RabbitHandler;  
import org.springframework.amqp.rabbit.annotation.RabbitListener;  
import org.springframework.stereotype.Component;  
  
import java.io.IOException;  
  
/**  
* @author fandongfeng  
* @date 2023/4/15 15:42  
*/  
@Component  
@RabbitListener(queues = RabbitMQConfig.ORDER_DEAD_LETTER_QUEUE)  
public class OrderMQListener {  
  
  
  
    @RabbitHandler  
    public void consumer(String body, Message message, Channel channel) throws IOException {  
        System.out.println("收到消息：" + DateUtil.now());  
        long msgTag = message.getMessageProperties().getDeliveryTag();  
        System.out.println("msgTag=" + msgTag);  
        System.out.println("message=" + message);  
        System.out.println("body=" + body);  
        channel.basicAck(msgTag, false);  
    }  
  
}
```

测试类

```java
@Test  
void testOrder() throws InterruptedException {  
//为true,则交换机处理消息到路由失败，则会返回给生产者 配置文件指定，则这里不需指定  
    rabbitTemplate.setMandatory(true);  
    //开启强制消息投递（mandatory为设置为true），但消息未被路由至任何一个queue，则回退一条消息  
    rabbitTemplate.setReturnsCallback(returned -> {  
    int code = returned.getReplyCode();  
    System.out.println("code=" + code);  
    System.out.println("returned=" + returned);  
    });  
    rabbitTemplate.convertAndSend(RabbitMQConfig.ORDER_EXCHANGE, "order", "测试订单延迟");  
    System.out.println("发送消息：" + DateUtil.now());  
    Thread.sleep(20000);  
}
```

程序输出

```java
发送消息：2023-04-16 15:14:34
收到消息：2023-04-16 15:14:44
msgTag=1
message=(Body:'测试订单延迟' MessageProperties [headers={spring_listener_return_correlation=03169cfc-5061-41fe-be47-c98e36d17eac, x-first-death-exchange=order_exchange, x-death=[{reason=expired, count=1, exchange=order_exchange, time=Mon Apr 16 15:14:44 CST 2023, routing-keys=[order], queue=order_queue}], x-first-death-reason=expired, x-first-death-queue=order_queue}, contentType=text/plain, contentEncoding=UTF-8, contentLength=0, receivedDeliveryMode=PERSISTENT, priority=0, redelivered=false, receivedExchange=order_dead_letter_exchange, receivedRoutingKey=order_dead_letter_queue_routing_key, deliveryTag=1, consumerTag=amq.ctag-Eh8GMgrsrAH1rvtGj7ykOQ, consumerQueue=order_dead_letter_queue])
body=测试订单延迟
```

