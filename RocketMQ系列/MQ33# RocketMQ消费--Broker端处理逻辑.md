---
title: MQ33# RocketMQ消费--Broker端处理逻辑
categories: RocketMQ
tags: RocketMQ
date: 2020-12-20 20:56:01
---



# 问题思考

1.Broker是如何处理消费流程的？

2.消费进度是如何流转的？

说明：本文分析均为PUSH消费模式



# Broker处理消费流程

本部分将消费的切分成三块梳理：Broker消费处理流程概览、查找消息流程、以及消息查询结果处理流程。

## Broker消费处理流程概览

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219143006.png)

```
小结：在拉取消息时会进行Broker和主题读权限的判断，实战中若有必要可以封锁Broker的拉取权限从而禁止从该broker进行消费；或者封锁某主题的读权限禁止消费组从该主题消费消息。
```



## 查找消息流程

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219143030.png)



<u>小结：如果需要从磁盘拉取消息则一次默认最多拉取8条，一次消息的消息大小最大为64K。</u>

<u>如果从缓存中拉取默认最多32条，一次拉取的消息大小最大256K。使用tagcode会在查找消息前进行过滤，使用SQL92过滤再消息查找出来后进行过滤。</u>



<!--more-->



## 消息查询结果处理流程

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219143112.png)

```
小结：建议开启slaveReadEnable=true，当拉取的消息超过Broker内存40%时会从Slave节点消费，Master不必从磁盘重新读取数据；transferMsgByHeap默认为true即消息先拉取到堆空间再返回到客户端；如果设置为false则使用Netty#FileRegion，可用零字节拷贝不必再拷贝到堆内存提高性能。
```



# 消费进度流转

## 客户端上报消费进度

```
//@1 顺序消费/并发消费流程相同

//ConsumeMessageOrderlyService#processConsumeResult

//ConsumeMessageConcurrentlyService#processConsumeResult

if (offset >= 0 && !consumeRequest.getProcessQueue().isDropped()) {

//更新消费进度偏移量

this.defaultMQPushConsumerImpl.getOffsetStore().updateOffset(consumeRequest.getMessageQueue(), offset, true);

}

@2 RemoteBrokerOffsetStore#updateOffset

AtomicLong offsetOld = this.offsetTable.get(mq);

MixAll.compareAndIncreaseOnly(offsetOld, offset);

@3 offsetTable存储结构：key为MessageQueue value为消费的偏移量进度

ConcurrentMap<MessageQueue, AtomicLong> offsetTable =

new ConcurrentHashMap<MessageQueue, AtomicLong>()

@4 定时同步消费进度

//持久化消息消费进度，默认5秒保存一次

this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

@Override

public void run() {

try {

MQClientInstance.this.persistAllConsumerOffset();

} catch (Exception e) {

log.error("ScheduledTask persistAllConsumerOffset exception", e);

}

}

}, 1000 * 10, this.clientConfig.getPersistConsumerOffsetInterval(), TimeUnit.MILLISECONDS);

@5 RemoteBrokerOffsetStore#persistAll

for (Map.Entry<MessageQueue, AtomicLong> entry : this.offsetTable.entrySet())

this.updateConsumeOffsetToBroker(mq, offset.get());
```



```
小结：PUSH消费中消费进度存储在offsetTable中，定时任务每5秒钟上报Broker一次。
```



## Broker端处理消费进度

**处理客户端定时上报消费进度**

```
//@1 ConsumerManageProcessor#processRequest#updateConsumerOffset

this.brokerController.getConsumerOffsetManager().commitOffset

//@2 ConsumerOffsetManager#commitOffset

String key = topic + TOPIC_GROUP_SEPARATOR + group;

this.commitOffset(clientHost, key, queueId, offset);

Long storeOffset = map.put(queueId, offset);

//@3 消费进度缓存结构

//key=topic@group

//value=ConcurrentMap<Integer/* queueId*/, Long/*offset*/>>

offsetTable = new ConcurrentHashMap<String, ConcurrentMap<Integer, Long>>(512);

//@4 5秒钟一次存储消费进度

this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

@Override

public void run() {

try {

BrokerController.this.consumerOffsetManager.persist();

} catch (Throwable e) {

log.error("schedule persist consumerOffset error.", e);

}

}

}, 1000 * 10, this.brokerConfig.getFlushConsumerOffsetInterval(), TimeUnit.MILLISECONDS);

//@5 consumerOffset.json文件格式

"zeus-package-mismatch-topic@autosort-packagelog":{0:9055300,1:9055157,2:9055304,3:9055232}
```



```
小结：Broker接到客户端消费进度上报后更新缓存offsetTable，每隔5秒中定时任务将offsetTable消费进度存储在磁盘文件consumerOffset.json中。
```

**消息拉取后实时更新消费进度**

```
//@1 PullMessageProcessor#processRequest

if (storeOffsetEnable) {

//更新消费进度

this.brokerController.getConsumerOffsetManager().commitOffset(RemotingHelper.parseChannelRemoteAddr(channel),

requestHeader.getConsumerGroup(), requestHeader.getTopic(), requestHeader.getQueueId(), requestHeader.getCommitOffset());

}
```



```
小结：PUSH消费客户端拉取消息后会实时更新消费的进度。
```



## 消费进度流转示意图

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219143256.png)