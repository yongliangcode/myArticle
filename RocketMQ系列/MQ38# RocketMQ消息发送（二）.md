---
title: MQ38# RocketMQ消息发送（二）
categories: RocketMQ
tags: RocketMQ
date: 2020-12-20 23:29:01
---



# 发送接口分类

\* 按照发送方式分类

1. 同步发送：等待返回结果

2. 异步发送：异步回调发送结果

3. 一次发送：无结果返回

\* 按一次发送消息数量分类

1. 单条消息发送

2. 批量消息发送

\* 按照是否指定MessageQueue分类

1. 随机选择发送

2. 指定特定MessageQueue

3. 自定义MessageQueue选择器

<!--more-->

**详细接口**

|接口 |描述 |

| --- | --- |

| send(final Message msg) |同步单条消息发送 |

| send(final Message msg, final long timeout) |同步单条消息发送（超时设置） |

| send(final Message msg, final SendCallback sendCallback) | 异步单条消息发送 |

| send(final Message msg, final SendCallback sendCallback, final long timeout) |异步单条消息发送（超时） |

|sendOneway(final Message msg) | 一次单条消息发送 |

| send(final Message msg, final MessageQueue mq) | 同步单条发送指定Queue |

| send(final Message msg, final MessageQueue mq, final long timeout) | 同步单条发送指定 Queue（超时设置） |

| send(final Message msg, final MessageQueue mq, final SendCallback sendCallback) |异步单条发送指定 Queue |

| send(final Message msg, final MessageQueue mq, final SendCallback sendCallback, long timeout) |异步单条发送指定 Queue（超时设置） |

| sendOneway(final Message msg, final MessageQueue mq) |一次单条发送指定 Queue |

| send(final Message msg, final MessageQueueSelector selector, final Object arg) |同步单条发送自定义实现Queue选择器 |

| send(final Message msg, final MessageQueueSelector selector, final Object arg,final long timeout) |同步单条发送自定义实现Queue选择器（超时设置） |

| send(final Message msg, final MessageQueueSelector selector, final Object arg, final SendCallback sendCallback) |异步单条发送自定义实现Queue选择器 |

| send(final Message msg, final MessageQueueSelector selector, final Object arg,final SendCallback sendCallback, final long timeout) |异步单条发送自定义实现Queue选择器（超时设置） |

|sendOneway(final Message msg, final MessageQueueSelector selector, final Object arg) |一次单条发送自定义实现Queue选择器 |

|send(final Collection<Message> msgs) |批量同步发送 |

|send(final Collection<Message> msgs, final long timeout) |批量同步发送（超时设置） |

|send(final Collection<Message> msgs, final MessageQueue mq) |批量同步指定Queue发送 |

|send(final Collection<Message> msgs, final MessageQueue mq, final long timeout)|批量同步指定Queue发送（超时设置）|



# 随机发送与自定义MessageQueue选择器

\* 随机发送：消息发往topic的哪个Queue是不确定的

\* 自定义MessageQueue发送：按照指定的算法路由到特定的MessageQueue，最常见需求，相同的key路由到相同的队列，实现发送分区有序

**随机发送**

\* 通过自增数取模消息队列数选择队列

```java
public MessageQueue selectOneMessageQueue(final TopicPublishInfo tpInfo, final String lastBrokerName) {

//开启高可用(开启故障延迟机制)

if (this.sendLatencyFaultEnable) {

try {

//自增序号

int index = tpInfo.getSendWhichQueue().getAndIncrement();

for (int i = 0; i < tpInfo.getMessageQueueList().size(); i++) {

//取模消息队列数

int pos = Math.abs(index++) % tpInfo.getMessageQueueList().size();

if (pos < 0)

pos = 0;

MessageQueue mq = tpInfo.getMessageQueueList().get(pos);

//判断队列是否为可用的

if (latencyFaultTolerance.isAvailable(mq.getBrokerName())) {

//正常情况下lastBrokerName==null;

//在消息重试（上次发送失败重新发送时）上次选择broker可用，优先选择

if (null == lastBrokerName || mq.getBrokerName().equals(lastBrokerName))

return mq;

}

}

final String notBestBroker = latencyFaultTolerance.pickOneAtLeast();

int writeQueueNums = tpInfo.getQueueIdByBroker(notBestBroker);

if (writeQueueNums > 0) {

final MessageQueue mq = tpInfo.selectOneMessageQueue();

if (notBestBroker != null) {

mq.setBrokerName(notBestBroker);

mq.setQueueId(tpInfo.getSendWhichQueue().getAndIncrement() % writeQueueNums);

}

return mq;

} else {

latencyFaultTolerance.remove(notBestBroker);

}

} catch (Exception e) {

log.error("Error occurred when selecting message queue", e);

}

return tpInfo.selectOneMessageQueue();

}

//不开启高可用

return tpInfo.selectOneMessageQueue(lastBrokerName);

}

public MessageQueue selectOneMessageQueue(final String lastBrokerName) {

//轮询消息队列的过程

if (lastBrokerName == null) {

return selectOneMessageQueue();

} else {

int index = this.sendWhichQueue.getAndIncrement();

for (int i = 0; i < this.messageQueueList.size(); i++) {

int pos = Math.abs(index++) % this.messageQueueList.size();

if (pos < 0)

pos = 0;

MessageQueue mq = this.messageQueueList.get(pos);

//规避上次发送失败的broker

if (!mq.getBrokerName().equals(lastBrokerName)) {

return mq;

}

}

return selectOneMessageQueue();

}

}
```



**自定义Queue选择器**

\* 分区有序：根据key进行路由选择，相同的key会路由到相同MessageQueue

```
private static MessageQueueSelector hashSelector = new MessageQueueSelector() {

@Override

public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {

int id = msg.getKeys().hashCode() % mqs.size();

if (id < 0) {

return mqs.get(-id);

} else {

return mqs.get(id);

}

}

};
```



<!--more-->



# 发送Broker容错处理

**两种Broker规避时长**

\* 正常发送规避时长为发送前后时间差值(endTimestamp-beginTimestampPrev)

\* 异常发送规避时长为30秒.

为何是30秒呢? NameSrv每10秒中清理下线broker，在启动时每30秒清理broker本地缓存表

\* 开启故障延迟需要设置producer.setSendLatencyFaultEnable(true)，默认为false不开启

**正常发送入参isolation为false**

```
beginTimestampPrev = System.currentTimeMillis();

sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout);

endTimestamp = System.currentTimeMillis();

this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);
```



**异常发送时入参isolation为true**

```
} catch (RemotingException e) {

endTimestamp = System.currentTimeMillis();

this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true);

log.warn(String.format("sendKernelImpl exception, resend at once, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mq), e);

log.warn(msg.toString());

exception = e;

continue;

} catch (MQClientException e) {

endTimestamp = System.currentTimeMillis();

this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true);

log.warn(String.format("sendKernelImpl exception, resend at once, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mq), e);

log.warn(msg.toString());

exception = e;

continue;

} catch (MQBrokerException e) {
```



**容错接口方法**

```
//代码位置：MQFaultStrategy->updateFaultItem

/**

\* @param brokerName

\* @param currentLatency 本次消息延迟时间（发送产生异常时的时间戳-开始发送消息时的时间戳）

\* @param isolation 是否隔离，true 使用默认30s来计算Broker规避时长；如果false则使用本次消息发送延迟时间来计算Broker规避时长

*/

public void updateFaultItem(final String brokerName, final long currentLatency, boolean isolation) {

if (this.sendLatencyFaultEnable) {

long duration = computeNotAvailableDuration(isolation ? 30000 : currentLatency);

this.latencyFaultTolerance.updateFaultItem(brokerName, currentLatency, duration);

}

}

public void updateFaultItem(final String name, final long currentLatency, final long notAvailableDuration) {

FaultItem old = this.faultItemTable.get(name);

if (null == old) {

final FaultItem faultItem = new FaultItem(name);

faultItem.setCurrentLatency(currentLatency);

//startTimeStamp = 当前系统时间+需要规避的时间

faultItem.setStartTimestamp(System.currentTimeMillis() + notAvailableDuration);

old = this.faultItemTable.putIfAbsent(name, faultItem);

if (old != null) {

old.setCurrentLatency(currentLatency);

old.setStartTimestamp(System.currentTimeMillis() + notAvailableDuration);

}

} else {

old.setCurrentLatency(currentLatency);

old.setStartTimestamp(System.currentTimeMillis() + notAvailableDuration);

}

}
```

**发送时的容错判断**

```
//代码位置：MQFaultStrategy->selectOneMessageQueue

//判断队列是否为可用的

if (latencyFaultTolerance.isAvailable(mq.getBrokerName())) {

//正常情况下lastBrokerName==null;

//在消息重试（上次发送失败重新发送时）上次选择broker可用，优先选择

if (null == lastBrokerName || mq.getBrokerName().equals(lastBrokerName))

return mq;

}

public boolean isAvailable(final String name) {

final FaultItem faultItem = this.faultItemTable.get(name);

if (faultItem != null) {

return faultItem.isAvailable();

}

return true;

}

public boolean isAvailable() {

//判断broker是否可用，startTimestamp在设置时：= 当前系统时间+需要规避的时间

//所以此处判断当前时间与startTimestamp的大小即可

return (System.currentTimeMillis() - startTimestamp) >= 0;

}
```



# 发送失败时的重试次数

\* 同步发送和异步发送在发送失败时，会进行消息重试。一次发送没有消息重试。

\* 重试次数由retryTimesWhenSendFailed和retryTimesWhenSendAsyncFailed参数决定，默认2. 总共重试3次。超过次数依然失败返回异常错误

**同步发送重试次数代码块**

```
//代码位置：DefaultMQProducerImpl->sendDefaultImpl()

//同步发送默认3(1+2)次 其他1次

//异步发送通过retryTimesWhenSendAsyncFailed来控制，在发送结果返回后再处理

int timesTotal = communicationMode == CommunicationMode.SYNC ? 1 + this.defaultMQProducer.getRetryTimesWhenSendFailed() : 1;

int times = 0;

String[] brokersSent = new String[timesTotal];

for (; times < timesTotal; times++) {

String lastBrokerName = null == mq ? null : mq.getBrokerName();

//选一个MessageQueue进行发送

MessageQueue tmpmq = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName);

if (tmpmq != null) {

mq = tmpmq;

brokersSent[times] = mq.getBrokerName();

....

}

```



**异步发送重试次数代码块**

\* 由if (needRetry && tmp <= timesTotal) 判断是否达到重试的阀值

```
//代码位置：MQClientAPIImpl.java

private void sendMessageAsync(//

final String addr, //

final String brokerName, //

final Message msg, //

final long timeoutMillis, //

final RemotingCommand request, //

final SendCallback sendCallback, //

final TopicPublishInfo topicPublishInfo, //

final MQClientInstance instance, //

final int retryTimesWhenSendFailed, //

final AtomicInteger times, //

final SendMessageContext context, //

final DefaultMQProducerImpl producer //

) throws InterruptedException, RemotingException {

this.remotingClient.invokeAsync(addr, request, timeoutMillis, new InvokeCallback() {

@Override

public void operationComplete(ResponseFuture responseFuture) {

RemotingCommand response = responseFuture.getResponseCommand();

if (null == sendCallback && response != null) {

try {

SendResult sendResult = MQClientAPIImpl.this.processSendResponse(brokerName, msg, response);

if (context != null && sendResult != null) {

context.setSendResult(sendResult);

context.getProducer().executeSendMessageHookAfter(context);

}

} catch (Throwable e) {

//

}

producer.updateFaultItem(brokerName, System.currentTimeMillis() - responseFuture.getBeginTimestamp(), false);

return;

}

if (response != null) {

try {

SendResult sendResult = MQClientAPIImpl.this.processSendResponse(brokerName, msg, response);

assert sendResult != null;

if (context != null) {

context.setSendResult(sendResult);

context.getProducer().executeSendMessageHookAfter(context);

}

try {

sendCallback.onSuccess(sendResult);

} catch (Throwable e) {

}

producer.updateFaultItem(brokerName, System.currentTimeMillis() - responseFuture.getBeginTimestamp(), false);

} catch (Exception e) {

producer.updateFaultItem(brokerName, System.currentTimeMillis() - responseFuture.getBeginTimestamp(), true);

onExceptionImpl(brokerName, msg, 0L, request, sendCallback, topicPublishInfo, instance,

retryTimesWhenSendFailed, times, e, context, false, producer);

}

} else {

producer.updateFaultItem(brokerName, System.currentTimeMillis() - responseFuture.getBeginTimestamp(), true);

if (!responseFuture.isSendRequestOK()) {

MQClientException ex = new MQClientException("send request failed", responseFuture.getCause());

onExceptionImpl(brokerName, msg, 0L, request, sendCallback, topicPublishInfo, instance,

retryTimesWhenSendFailed, times, ex, context, true, producer);

} else if (responseFuture.isTimeout()) {

MQClientException ex = new MQClientException("wait response timeout " + responseFuture.getTimeoutMillis() + "ms",

responseFuture.getCause());

onExceptionImpl(brokerName, msg, 0L, request, sendCallback, topicPublishInfo, instance,

retryTimesWhenSendFailed, times, ex, context, true, producer);

} else {

MQClientException ex = new MQClientException("unknow reseaon", responseFuture.getCause());

onExceptionImpl(brokerName, msg, 0L, request, sendCallback, topicPublishInfo, instance,

retryTimesWhenSendFailed, times, ex, context, true, producer);

}

}

}

});

}

private void onExceptionImpl(final String brokerName, //

final Message msg, //

final long timeoutMillis, //

final RemotingCommand request, //

final SendCallback sendCallback, //

final TopicPublishInfo topicPublishInfo, //

final MQClientInstance instance, //

final int timesTotal, //

final AtomicInteger curTimes, //

final Exception e, //

final SendMessageContext context, //

final boolean needRetry, //

final DefaultMQProducerImpl producer // 12

) {

int tmp = curTimes.incrementAndGet();

if (needRetry && tmp <= timesTotal) {

String retryBrokerName = brokerName;//by default, it will send to the same broker

if (topicPublishInfo != null) { //select one message queue accordingly, in order to determine which broker to send

MessageQueue mqChosen = producer.selectOneMessageQueue(topicPublishInfo, brokerName);

retryBrokerName = mqChosen.getBrokerName();

}

String addr = instance.findBrokerAddressInPublish(retryBrokerName);

log.info("async send msg by retry {} times. topic={}, brokerAddr={}, brokerName={}", tmp, msg.getTopic(), addr,

retryBrokerName);

try {

request.setOpaque(RemotingCommand.createNewRequestId());

sendMessageAsync(addr, retryBrokerName, msg, timeoutMillis, request, sendCallback, topicPublishInfo, instance,

timesTotal, curTimes, context, producer);

} catch (InterruptedException e1) {

onExceptionImpl(retryBrokerName, msg, timeoutMillis, request, sendCallback, topicPublishInfo, instance, timesTotal, curTimes, e1,

context, false, producer);

} catch (RemotingConnectException e1) {

producer.updateFaultItem(brokerName, 3000, true);

onExceptionImpl(retryBrokerName, msg, timeoutMillis, request, sendCallback, topicPublishInfo, instance, timesTotal, curTimes, e1,

context, true, producer);

} catch (RemotingTooMuchRequestException e1) {

onExceptionImpl(retryBrokerName, msg, timeoutMillis, request, sendCallback, topicPublishInfo, instance, timesTotal, curTimes, e1,

context, false, producer);

} catch (RemotingException e1) {

producer.updateFaultItem(brokerName, 3000, true);

onExceptionImpl(retryBrokerName, msg, timeoutMillis, request, sendCallback, topicPublishInfo, instance, timesTotal, curTimes, e1,

context, true, producer);

}

} else {

if (context != null) {

context.setException(e);

context.getProducer().executeSendMessageHookAfter(context);

}

try {

sendCallback.onException(e);

} catch (Exception ignored) {

}

}

}
```

