---
title: MQ32# RocketMQ客户端消费--ProcessQueue处理队列
categories: RocketMQ
tags: RocketMQ
date: 2020-12-20 20:55:01
---



# 问题思考

在消费消息时处处能看到处理队列ProcessQueue的身影，既然随处可见也一定很重要，那有必要分析下为何重要了。

```
1.ProcessQueue提供哪些方法？

2.这些方法的作用是什么？

3.哪里调用了这些方法？
```

<!--more-->

# ProcessQueue方法梳理

## isLockExpired方法

```
public boolean isLockExpired() {

	return (System.currentTimeMillis() - this.lastLockTimestamp) > REBALANCE_LOCK_MAX_LIVE_TIME;

}
```



**lastLockTimestamp(最新加锁时间戳)**

lastLockTimestamp属性调用

```
@1 RebalanceImpl#lock

//请求broker对该消费队列进行加锁

Set<MessageQueue> lockedMq = this.mQClientFactory.getMQClientAPIImpl().lockBatchMQ

...

processQueue.setLocked(true);//加锁

//设置加锁时间戳

processQueue.setLastLockTimestamp(System.currentTimeMillis());

}

@2 RebalanceImpl#lockAll

//向broker发送该clientId所对应的messageQueue锁定请求

Set<MessageQueue> lockOKMQSet =

this.mQClientFactory.getMQClientAPIImpl().lockBatchMQ

...

processQueue.setLocked(true);

processQueue.setLastLockTimestamp(System.currentTimeMillis());


```



**isLockExpired方法调用**

```
@1 ConsumeMessageOrderlyService#ConsumeRequest

//集群顺序消费时需要判断processQueue加锁是否过期

if (MessageModel.BROADCASTING.equals(ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.messageModel())

|| (this.processQueue.isLocked() && !this.processQueue.isLockExpired()))

```



```
小结：@1&@2中可以看出lastLockTimestamp在顺序消费时向Broker请求对队列加锁成功后设置的时间戳；REBALANCE_LOCK_MAX_LIVE_TIME由参数rocketmq.client.rebalance.lockMaxLiveTime设置默认为30秒；lastLockTimestamp的含义为加锁的有效时间为30秒，超过该时间则失效；顺序消费在判断过期时延迟拉取。
```



## isPullExpired方法

**isPullExpired方法代码**

```
public boolean isPullExpired() {

return (System.currentTimeMillis() - this.lastPullTimestamp) > PULL_MAX_IDLE_TIME;

}
```



<!--more-->



**lastPullTimestamp属性调用**

```
@1 DefaultMQPushConsumerImpl#pullMessage

//每次消息拉取后更新最后一次拉取时间戳

pullRequest.getProcessQueue()

.setLastPullTimestamp(System.currentTimeMillis());
```

```
@1 RebalanceImpl#updateProcessQueueTableInRebalance

//在负载均衡时如果拉取时间失效会将ProceeQueue丢弃

else if (pq.isPullExpired()) {

case CONSUME_ACTIVELY:

break;

case CONSUME_PASSIVELY:

pq.setDropped(true);

}
```

```
小结：lastPullTimestamp每次拉取消息都会更新时间戳；PULL_MAX_IDLE_TIME由rocketmq.client.pull.pullMaxIdleTime设置默认为120秒；方法在负载均衡更新ProcessQueueTable时调用如果拉取失效ProcessQueue将被丢弃。
```



## putMessage方法

putMessage方法代码

```
/**

\* 消息丢入红黑树

\* @param msgs

\* @return

*/

public boolean putMessage(final List<MessageExt> msgs) {

boolean dispatchToConsume = false;

try {

this.lockTreeMap.writeLock().lockInterruptibly();

try {

int validMsgCnt = 0;

for (MessageExt msg : msgs) {

//消息存入红黑树key为queueOffset,value为消息

MessageExt old = msgTreeMap.put(msg.getQueueOffset(), msg);

if (null == old) {

validMsgCnt++;//递增消息数量

//记录最大queueOffset

this.queueOffsetMax = msg.getQueueOffset();

}

}

//统计消息数量

msgCount.addAndGet(validMsgCnt);

//消息集合有数据可继续消费

if (!msgTreeMap.isEmpty() && !this.consuming) {

dispatchToConsume = true;

this.consuming = true;

}

if (!msgs.isEmpty()) {

//取出这批消息集合中的最后一条消息

MessageExt messageExt = msgs.get(msgs.size() - 1);

//获取分区最大的消息offet

String property = messageExt.getProperty(MessageConst.PROPERTY_MAX_OFFSET);

if (property != null) {

//计算还有多少消息未被消费

long accTotal = Long.parseLong(property) - messageExt.getQueueOffset();

if (accTotal > 0) {

this.msgAccCnt = accTotal;

}

}

}

} finally {

this.lockTreeMap.writeLock().unlock();

}

} catch (InterruptedException e) {

log.error("putMessage exception", e);

}

return dispatchToConsume;

}
```



putMessage方法调用

```
@1 DefaultMQPushConsumerImpl#pullMessage

boolean dispathToConsume = processQueue.putMessage(pullResult.getMsgFoundList())

@2 ConsumeMessageOrderlyService#submitConsumeRequest

if (dispathToConsume) { //dispathToConsume顺序消费返回结果为true

ConsumeRequest consumeRequest =

new ConsumeRequest(processQueue, messageQueue);

this.consumeExecutor.submit(consumeRequest);

}

@3 PullConsumerImpl#registerPullTaskCallback

pq.putMessage(pullResult.getMsgFoundList());
```



```
小结：在拉取消息后处理消息时，提交给ProcessQueue#putMessage中红黑树msgTreeMap存储一份数据，并统计消息的数量以及还有多少消息未被拉取；返回结果dispatchToConsume如果true表明消费集合有数据，顺序消息会据此构造消费请求继续处理。
```



## getMaxSpan方法

getMaxSpan方法代码

```
/**

\* 获取当前消息最大间隔

\* 消息队列第一条消息与最后一条消息的偏移量差值

\* @return

*/

public long getMaxSpan() {

try {

this.lockTreeMap.readLock().lockInterruptibly();

try {

if (!this.msgTreeMap.isEmpty()) {

return this.msgTreeMap.lastKey() - this.msgTreeMap.firstKey();

}

} finally {

this.lockTreeMap.readLock().unlock();

}

} catch (InterruptedException e) {

log.error("getMaxSpan exception", e);

}

return 0;

}
```



**getMaxSpan方法调用**

```
@1 DefaultMQPushConsumerImpl#pullMessage

f (!this.consumeOrderly) {//非顺序消费

//并发处理的消息跨度不能超过2000，超过2000，延迟50秒进行拉去

if (processQueue.getMaxSpan() > this.defaultMQPushConsumer.getConsumeConcurrentlyMaxSpan()) {

this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_FLOW_CONTROL);

if ((flowControlTimes2++ % 1000) == 0) { //每出发1000次打印流控日志

log.warn(

"the queue's messages, span too long, so do flow control, minOffset={}, maxOffset={}, maxSpan={}, pullRequest={}, flowControlTimes={}",

processQueue.getMsgTreeMap().firstKey(), processQueue.getMsgTreeMap().lastKey(), processQueue.getMaxSpan(),

pullRequest, flowControlTimes2);

}

return;

}

}
```



```
小结：getMaxSpan方法计算已拉取消息队列第一条消息与最后一条消息的偏移量差值，即拉取消息的数量跨度，在并发消息时调用，如果该跨度大于2000则延迟50秒再拉取数据。
```



## removeMessage方法

removeMessage方法代码

```
/**

\* 将消息从红黑树msgTreeMap中移除

\* @param msgs

\* @return

*/

public long removeMessage(final List<MessageExt> msgs) {

long result = -1;

final long now = System.currentTimeMillis();

try {

this.lockTreeMap.writeLock().lockInterruptibly();

this.lastConsumeTimestamp = now;

try {

if (!msgTreeMap.isEmpty()) {

result = this.queueOffsetMax + 1;

int removedCnt = 0;

for (MessageExt msg : msgs) {

//移除消息

MessageExt prev = msgTreeMap.remove(msg.getQueueOffset());

if (prev != null) {

removedCnt--;

}

}

msgCount.addAndGet(removedCnt);//消息总数扣除

if (!msgTreeMap.isEmpty()) {

result = msgTreeMap.firstKey(); //第一条消息的偏移量

}

}

} finally {

this.lockTreeMap.writeLock().unlock();

}

} catch (Throwable t) {

log.error("removeMessage exception", t);

}

return result;

}
```



removeMessage方法调用

```
@1 ConsumeMessageConcurrentlyService#processConsumeResult

//消费完毕后从ProcessQueue中清除这批消息

//offset为清除这批偏移量后processQueue.msgTreeMap中最小的偏移量

long offset = consumeRequest.getProcessQueue().removeMessage(consumeRequest.getMsgs());
```



\```

```
小结：removeMessage方法在并发消费后进行调用，消息处理完了将ProcessQueue中红黑树这批消息移除。
```

## cleanExpiredMsg方法

cleanExpiredMsg方法代码

```
/**

\* @param pushConsumer

*/

public void cleanExpiredMsg(DefaultMQPushConsumer pushConsumer) {

if (pushConsumer.getDefaultMQPushConsumerImpl().isConsumeOrderly()) {

return;

}

//每次调用最多清理16条

int loop = msgTreeMap.size() < 16 ? msgTreeMap.size() : 16;

for (int i = 0; i < loop; i++) {

MessageExt msg = null;

try {

this.lockTreeMap.readLock().lockInterruptibly();

try {

//默认超过15分钟未消费的消息将延迟3个延迟级别再消费

if (!msgTreeMap.isEmpty() && System.currentTimeMillis() - Long.parseLong(MessageAccessor.getConsumeStartTimeStamp(msgTreeMap.firstEntry().getValue())) > pushConsumer.getConsumeTimeout() * 60 * 1000) {

msg = msgTreeMap.firstEntry().getValue();

} else {

break;

}

} finally {

this.lockTreeMap.readLock().unlock();

}

} catch (InterruptedException e) {

log.error("getExpiredMsg exception", e);

}

try {

//返回broker，延迟3个级别消费

pushConsumer.sendMessageBack(msg, 3);

log.info("send expire msg back. topic={}, msgId={}, storeHost={}, queueId={}, queueOffset={}", msg.getTopic(), msg.getMsgId(), msg.getStoreHost(), msg.getQueueId(), msg.getQueueOffset());

try {

this.lockTreeMap.writeLock().lockInterruptibly();

try {

if (!msgTreeMap.isEmpty() && msg.getQueueOffset() == msgTreeMap.firstKey()) {

try {

//将消息移出

msgTreeMap.remove(msgTreeMap.firstKey());

} catch (Exception e) {

log.error("send expired msg exception", e);

}

}

} finally {

this.lockTreeMap.writeLock().unlock();

}

} catch (InterruptedException e) {

log.error("getExpiredMsg exception", e);

}

} catch (Exception e) {

log.error("send expired msg exception", e);

}

}

}
```



**cleanExpiredMsg方法调用**

```
@1 ConsumeMessageConcurrentlyService#start()

//定时任务15分钟运行一次

public void start() {

this.cleanExpireMsgExecutors.scheduleAtFixedRate(new Runnable() {

@Override

public void run() {

cleanExpireMsg();

}

}, this.defaultMQPushConsumer.getConsumeTimeout(), this.defaultMQPushConsumer.getConsumeTimeout(), TimeUnit.MINUTES);

}

private void cleanExpireMsg() {

Iterator<Map.Entry<MessageQueue, ProcessQueue>> it = this.defaultMQPushConsumerImpl.getRebalanceImpl().getProcessQueueTable().entrySet().iterator();

while (it.hasNext()) {

Map.Entry<MessageQueue, ProcessQueue> next = it.next();

ProcessQueue pq = next.getValue();

//清理过期消息

pq.cleanExpiredMsg(this.defaultMQPushConsumer);

}

}
```



```
小结：cleanExpiredMsg方法在并发消费中调用，每个15分钟执行一次；该方法会对超过15分钟未被消费的数据进行清理，每次最多清理16条。
```



## takeMessags方法

takeMessags方法代码

```
/**

\* 从ProcessQueue中取出batchSize条消息

\* @param batchSize

\* @return

*/

public List<MessageExt> takeMessags(final int batchSize) {

List<MessageExt> result = new ArrayList<MessageExt>(batchSize);

final long now = System.currentTimeMillis();

try {

this.lockTreeMap.writeLock().lockInterruptibly();

this.lastConsumeTimestamp = now;

try {

if (!this.msgTreeMap.isEmpty()) {

for (int i = 0; i < batchSize; i++) {

//消息从红黑树中取出

Map.Entry<Long, MessageExt> entry = this.msgTreeMap.pollFirstEntry();

if (entry != null) {

result.add(entry.getValue());

//取出的消息在msgTreeMapTemp中存储一份

msgTreeMapTemp.put(entry.getKey(), entry.getValue());

} else {

break;

}

}

}

if (result.isEmpty()) {

consuming = false;

}

} finally {

this.lockTreeMap.writeLock().unlock();

}

} catch (InterruptedException e) {

log.error("take Messages exception", e);

}

return result;

}
```



takeMessags方法调用

```
@1 ConsumeMessageOrderlyService#ConsumeRequest

//取出消息

List<MessageExt> msgs = this.processQueue.takeMessags(consumeBatchSize);

if (!msgs.isEmpty()) {

//客户端消费消息

status = messageListener.consumeMessage(Collections.unmodifiableList(msgs), context);

}
```



```
小结：顺序消费时通过ProcessQueue#takeMessags获取特定数量的消息（默认1条）并传给客户端Listener进行处理。
```



## rollback方法

rollback方法代码

```
/**

\* 将msgTreeMapTmp中所有消息重新放入到msgTreeMap并清除msgTreeMapTmp

*/

public void rollback() {

try {

this.lockTreeMap.writeLock().lockInterruptibly();

try {

this.msgTreeMap.putAll(this.msgTreeMapTemp);

this.msgTreeMapTemp.clear();

} finally {

this.lockTreeMap.writeLock().unlock();

}

} catch (InterruptedException e) {

log.error("rollback exception", e);

}

}
```



rollback方法调用

```
@1 ConsumeMessageOrderlyService#processConsumeResult

case ROLLBACK:

consumeRequest.getProcessQueue().rollback();
```



```
小结：在顺序消费客户端处理消息后，如果消息处理结果的状态为ROLLBACK，此时调用ProcessQueue#rollback方法；将msgTreeMapTmp中的消息重新写回红黑树msgTreeMap中；ROLLBACK此状态在顺序消费时已不建议使用。
```



## commit方法

commit方法代码

```
/**

\* 将msgTreeMapTmp中的消息清除，表示成功处理了该批消息

\* @return

*/

public long commit() {

try {

this.lockTreeMap.writeLock().lockInterruptibly();

try {

Long offset = this.msgTreeMapTemp.lastKey();

msgCount.addAndGet(this.msgTreeMapTemp.size() * (-1));

this.msgTreeMapTemp.clear();

if (offset != null) {

return offset + 1; //返回下一条消息offset

}

} finally {

this.lockTreeMap.writeLock().unlock();

}

} catch (InterruptedException e) {

log.error("commit exception", e);

}

return -1;

}
```



commit方法调用

```
@1 ConsumeMessageOrderlyService#processConsumeResult

case SUCCESS:

//清空msgTreeMapTemp

commitOffset = consumeRequest.getProcessQueue().commit();
```



```
小结：在顺序消费客户端处理消息状态为成功时，内存中消费偏移量提交即ProcessQueue#commit清空msgTreeMapTemp临时红黑树中的数据。
```



## makeMessageToCosumeAgain方法

makeMessageToCosumeAgain方法代码

```
/**

\* 重新消费该批消息

\* @param msgs

*/

public void makeMessageToCosumeAgain(List<MessageExt> msgs) {

try {

this.lockTreeMap.writeLock().lockInterruptibly();

try {

for (MessageExt msg : msgs) {

//将消息从msgTreeMapTemp移除

this.msgTreeMapTemp.remove(msg.getQueueOffset());

//将该批消息重新放入msgTreeMap

this.msgTreeMap.put(msg.getQueueOffset(), msg);

}

} finally {

this.lockTreeMap.writeLock().unlock();

}

} catch (InterruptedException e) {

log.error("makeMessageToCosumeAgain exception", e);

}
```



makeMessageToCosumeAgain方法调用

```
case SUSPEND_CURRENT_QUEUE_A_MOMENT:

//重新消费该批信息

onsumeRequest.getProcessQueue().makeMessageToCosumeAgain(msgs);
```



```
小结：akeMessageToCosumeAgain在顺序消费客户端返回消息状态为SUSPEND_CURRENT_QUEUE_A_MOMENT时调用；将消息从msgTreeMapTemp移除，并将该批消息重新放入msgTreeMap。
```



# 总结

ProcessQueue作为MessageQueue在消费端的镜像，从负载均衡、消息拉取、消费状态处理、offset提交，控制着整个消费的脉搏，尤其在顺序消费中参与更多。