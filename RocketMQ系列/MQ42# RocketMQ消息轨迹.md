---
title: MQ42# RocketMQ消息轨迹
categories: RocketMQ
tags: RocketMQ
date: 2020-12-20 23:45:01
---



# Broker配置

首先看下broker.conf配置的两个属性

| 属性 |默认值 |

| --- | --- |

|traceTopicEnable |false |

| msgTraceTopicName | RMQ_SYS_TRACE_TOPIC |

在一个集群中可以配置一台机器专门负责消息轨迹的收集工作，该台机器上配置traceTopicEnable = true,

borker启动的时候自动创建默认轨迹topic

TopicConfigManager.java构造方法，BrokerController在启动的时候，会初始化TopicConfigManager实现trace topic的创建工作

```java
{

if (this.brokerController.getBrokerConfig().isTraceTopicEnable()) {

String topic = this.brokerController.getBrokerConfig().getMsgTraceTopicName();

TopicConfig topicConfig = new TopicConfig(topic);

this.systemTopicList.add(topic);

topicConfig.setReadQueueNums(1);

topicConfig.setWriteQueueNums(1);

this.topicConfigTable.put(topicConfig.getTopicName(), topicConfig);

}

}
```

<!--more-->

# **客户端发送实现**

客户端发送

DefaultMQProducer producer = new DefaultMQProducer("ProducerGroupName",true);

```
public DefaultMQProducer(final String producerGroup, RPCHook rpcHook, boolean enableMsgTrace,final String customizedTraceTopic) {

this.producerGroup = producerGroup;

defaultMQProducerImpl = new DefaultMQProducerImpl(this, rpcHook);

//if client open the message trace feature

if (enableMsgTrace) {

try {

AsyncTraceDispatcher dispatcher = new AsyncTraceDispatcher(customizedTraceTopic, rpcHook);

dispatcher.setHostProducer(this.getDefaultMQProducerImpl());

traceDispatcher = dispatcher;

//为消息轨迹注册hook,在消息发送前执行

this.getDefaultMQProducerImpl().registerSendMessageHook(

new SendMessageTraceHookImpl(traceDispatcher));

} catch (Throwable e) {

log.error("system mqtrace hook init failed ,maybe can't send msg trace data");

}

}

}
```

SendMessageTraceHookImpl 实现了SendMessageHook接口，在消息发送前后会被调用

AsyncTraceDispatcher 主要负责消息的发送工作；内部队列，由线程池批量（100条）发送

# **Hook调用**

发送前hook调用

```
//如果有hook在消息发送前执行，消息轨迹通过这种方式记录

if (this.hasSendMessageHook()) {

context = new SendMessageContext();

context.setProducer(this); //发送对象

context.setProducerGroup(this.defaultMQProducer.getProducerGroup()); //生产组

context.setCommunicationMode(communicationMode); //发送模式

context.setBornHost(this.defaultMQProducer.getClientIP()); //客户端IP

context.setBrokerAddr(brokerAddr); //发往broker的地址

context.setMessage(msg); //消息内容

context.setMq(mq); //消息 Queue

String isTrans = msg.getProperty(MessageConst.PROPERTY_TRANSACTION_PREPARED);

if (isTrans != null && isTrans.equals("true")) {

context.setMsgType(MessageType.Trans_Msg_Half);

}

if (msg.getProperty("__STARTDELIVERTIME") != null || msg.getProperty(MessageConst.PROPERTY_DELAY_TIME_LEVEL) != null) {

context.setMsgType(MessageType.Delay_Msg);

}

this.executeSendMessageHookBefore(context); //执行自定义个hook业务

}
```



发送后hook调用

```
//消息发送后执行的hook，消息轨迹会调用

if (this.hasSendMessageHook()) {

context.setSendResult(sendResult);

this.executeSendMessageHookAfter(context);

}
```



# **发送轨迹**

Producer启动时注册钩子，该钩子持有负责消息发送的AsyncTraceDispatcher实例，消息发送后进而发送消息轨迹

**发送轨迹的消息格式**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219151653.png)



# 客户端消费轨迹实现

消费轨迹：与消息发送的轨迹实现思路相同

DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("CID_JODIE_1",true);

**注册消费钩子**

ConsumeMessageTraceHookImpl实现了ConsumeMessageHook，在消费的前后会进行回调

```
public DefaultMQPushConsumer(final String consumerGroup, RPCHook rpcHook,

AllocateMessageQueueStrategy allocateMessageQueueStrategy, boolean enableMsgTrace, final String customizedTraceTopic) {

this.consumerGroup = consumerGroup;

this.allocateMessageQueueStrategy = allocateMessageQueueStrategy;

defaultMQPushConsumerImpl = new DefaultMQPushConsumerImpl(this, rpcHook);

if (enableMsgTrace) {

try {

AsyncTraceDispatcher dispatcher = new AsyncTraceDispatcher(customizedTraceTopic, rpcHook);

dispatcher.setHostConsumer(this.getDefaultMQPushConsumerImpl());

traceDispatcher = dispatcher;

//注册消费hook

this.getDefaultMQPushConsumerImpl().registerConsumeMessageHook(

new ConsumeMessageTraceHookImpl(traceDispatcher));

} catch (Throwable e) {

log.error("system mqtrace hook init failed ,maybe can't send msg trace data");

}

}

}
```

ConsumeMessageConcurrentlyService.ConsumeRequest.run消费前执行

```
//消费前执行hook 消费轨迹会执行

ConsumeMessageContext consumeMessageContext = null;

if (ConsumeMessageConcurrentlyService.this.defaultMQPushConsumerImpl.hasHook()) {

consumeMessageContext = new ConsumeMessageContext();

consumeMessageContext.setConsumerGroup(defaultMQPushConsumer.getConsumerGroup());

consumeMessageContext.setProps(new HashMap<String, String>());

consumeMessageContext.setMq(messageQueue);

consumeMessageContext.setMsgList(msgs);

consumeMessageContext.setSuccess(false); //消费状态

ConsumeMessageConcurrentlyService.this.defaultMQPushConsumerImpl.executeHookBefore(consumeMessageContext);

}
```



**消费后执行**

```
if (ConsumeMessageConcurrentlyService.this.defaultMQPushConsumerImpl.hasHook()) {

consumeMessageContext.setStatus(status.toString());

consumeMessageContext.setSuccess(ConsumeConcurrentlyStatus.CONSUME_SUCCESS == status);

ConsumeMessageConcurrentlyService.this.defaultMQPushConsumerImpl.executeHookAfter(consumeMessageContext);

}

```



**消费轨迹格式**

分为两部分，一部分为消费前，一部分为消费后

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219151820.png)



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219151831.png)