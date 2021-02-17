---
title: MQ29# RocketMQ客户端PUSH消费概览
categories: RocketMQ
tags: RocketMQ
date: 2020-12-20 20:51:01
---



# 问题描述

PUSH消费整体流程是怎么样的？



<!--more-->



# PUSH消费流程概览

**从客户端示例开始**

```
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("CID_JODIE_1");

consumer.subscribe("Jodie_topic_1023", "*"); consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);

//wrong time format 2017_0422_221800

consumer.setConsumeTimestamp("20170422221800");

consumer.registerMessageListener(new MessageListenerConcurrently() {

@Override

public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {

System.out.printf(Thread.currentThread().getName() + " Receive New Messages: " + msgs + "%n");

return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;

}

});

consumer.start();

System.out.printf("Consumer Started.%n");

```



**客户端PUSH消费流程概览**

**概览流程1**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219140443.png)

<!--more-->



**概览流程2**



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219140501.png)



<u>小结：PUSH消费的主要内容实例化DefaultMQPushConsumer，注册MessageQueue分配策略；</u>

<u>初始化订阅数据并存入缓存；注册消费监听用于回调处理消息；创建并启动MQClientInstance实例；向Broker发送心跳等工作。</u>



**参数校验哪些内容？**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219140542.png)

```
小结：参数校验中MessageListener只能为顺序消费或者并发消费两种模式；消费最小线程consumeThreadMin取值需要小于1000即最多1000个消费线程；由于为无界队consumeThreadMax设置无效。
```



**MQClientInstance初始化与启动**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219140622.png)

<u>小结：MQClientInstance初始化启动连带一系列线程类的启动。例如：PullMessageService、RebalanceService等以及通过Netty建立TCP通道。</u>



# 总结

本篇文章主要对PUSH消费启动有个整体的印象，在分析消息拉取/并发消费/顺序消费/负载均衡时再来看各个类的具体职责。