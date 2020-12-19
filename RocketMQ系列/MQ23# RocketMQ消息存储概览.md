---
title: MQ23# RocketMQ消息存储概览
categories: RocketMQ
tags: RocketMQ
date: 2020-12-20 20:39:01
---



先梳理消息存储主干流程。本分切分为两部分，第一部分消息存储流程概览，主要为校验流程；第二部分CommitLog存储概览，即消息存储流程。



# 消息存储流程概览

调用链

```java
@1 SendMessageProcessor#sendMessage

//消息存储

PutMessageResult putMessageResult = this.brokerController.getMessageStore().putMessage(msgInner);

@2 DefaultMessageStore#putMessage
```



流程图

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219132807.png)



```
备注：PageCache是否繁忙，内存锁定时间为1秒，在集群流量负载很高时可能出现system busy，broker buys等异常信息。
```

源代码

```java
public PutMessageResult putMessage(MessageExtBrokerInner msg) {

//如果消息存储服务已关闭，则消息写入被拒绝

if (this.shutdown) {

log.warn("message store has shutdown, so putMessage is forbidden");

return new PutMessageResult(PutMessageStatus.SERVICE_NOT_AVAILABLE, null);

}

//Slave不处理消息存储

if (BrokerRole.SLAVE == this.messageStoreConfig.getBrokerRole()) {

long value = this.printTimes.getAndIncrement();

if ((value % 50000) == 0) {

log.warn("message store is slave mode, so putMessage is forbidden ");

}

return new PutMessageResult(PutMessageStatus.SERVICE_NOT_AVAILABLE, null);

}

//如果消息存储服务不可写，则消息写入会被拒绝

//出现该错误可能磁盘已满

if (!this.runningFlags.isWriteable()) {

long value = this.printTimes.getAndIncrement();

if ((value % 50000) == 0) {

log.warn("message store is not writeable, so putMessage is forbidden " + this.runningFlags.getFlagBits());

}

return new PutMessageResult(PutMessageStatus.SERVICE_NOT_AVAILABLE, null);

} else {

this.printTimes.set(0);

}

//Topic长度的限制不能超过127个字节

if (msg.getTopic().length() > Byte.MAX_VALUE) {

log.warn("putMessage message topic length too long " + msg.getTopic().length());

return new PutMessageResult(PutMessageStatus.MESSAGE_ILLEGAL, null);

}

//消息属性长度检查不能超过32K

if (msg.getPropertiesString() != null && msg.getPropertiesString().length() > Short.MAX_VALUE) {

log.warn("putMessage message properties length too long " + msg.getPropertiesString().length());

return new PutMessageResult(PutMessageStatus.PROPERTIES_SIZE_EXCEEDED, null);

}

//判断PageCache是否繁忙：阀值[osPageCacheBusyTimeOutMills = 1000 ] 比较时间为当前时间与Commit Lock时间之差

//如果返回true，意味着此时有消息在写入CommitLog，且那条消息的写入耗时较长（超过1s），则本条消息不再写入

//返回内存页写入繁忙

if (this.isOSPageCacheBusy()) {

return new PutMessageResult(PutMessageStatus.OS_PAGECACHE_BUSY, null);

}

long beginTime = this.getSystemClock().now();

//将消息写入CommitLog

PutMessageResult result = this.commitLog.putMessage(msg);

//消息写入时间过长，发出警告

long eclipseTime = this.getSystemClock().now() - beginTime;

if (eclipseTime > 500) {

log.warn("putMessage not in lock eclipse time(ms)={}, bodyLength={}", eclipseTime, msg.getBody().length);

}

//对消息的存储耗时进行分级记录，并记录当前所有消息存储时的最大耗时

this.storeStatsService.setPutMessageEntireTimeMax(eclipseTime);

//记录存粗失败次数

if (null == result || !result.isOk()) {

this.storeStatsService.getPutMessageFailedTimes().incrementAndGet();

}

return result;

}
```



<!--more-->



# CommitLog存储流程

调用链

```java
@1 DefaultMessageStore#putMessage

//将消息写入CommitLog

PutMessageResult result = this.commitLog.putMessage(msg);

@2 CommitLog#putMessage
```



流程图

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219132948.png)



```
备注：此时写入消息并没有写入磁盘，而是写入了writeBuffer或者mappedByteBuffer（PageCache或堆外内存）
```

源代码

```
public PutMessageResult putMessage(final MessageExtBrokerInner msg) {

// Set the storage time

//设置消息存储时间（存储到Broker的时间）

msg.setStoreTimestamp(System.currentTimeMillis());

// Set the message body BODY CRC (consider the most appropriate setting

// on the client)

//Message Body的循环冗余校验码，防止消息体内容被篡改

msg.setBodyCRC(UtilAll.crc32(msg.getBody()));

// Back to Results

AppendMessageResult result = null;

//统计存储耗时相关的Metric

StoreStatsService storeStatsService = this.defaultMessageStore.getStoreStatsService();

String topic = msg.getTopic();

int queueId = msg.getQueueId();

//获取消息类型

final int tranType = MessageSysFlag.getTransactionValue(msg.getSysFlag());

//不处理事务消息

//重试(延时)消息发到SCHEDULE_TOPIC中

if (tranType == MessageSysFlag.TRANSACTION_NOT_TYPE//

|| tranType == MessageSysFlag.TRANSACTION_COMMIT_TYPE) {

// Delay Delivery

//延时投递时间级别

if (msg.getDelayTimeLevel() > 0) {

if (msg.getDelayTimeLevel() > this.defaultMessageStore.getScheduleMessageService().getMaxDelayLevel()) {

msg.setDelayTimeLevel(this.defaultMessageStore.getScheduleMessageService().getMaxDelayLevel());

}

//将Topic更改为 SCHEDULE_TOPIC_XXXX

topic = ScheduleMessageService.SCHEDULE_TOPIC;

//根据延时级别获取延时消息新队列ID（queueId等于延时级别-1）

queueId = ScheduleMessageService.delayLevel2QueueId(msg.getDelayTimeLevel());

// Backup real topic, queueId

//将消息中原topic和queueId存入到消息属性中

MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_TOPIC, msg.getTopic());

MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_QUEUE_ID, String.valueOf(msg.getQueueId()));

msg.setPropertiesString(MessageDecoder.messageProperties2String(msg.getProperties()));

msg.setTopic(topic);

msg.setQueueId(queueId);

}

}

long eclipseTimeInLock = 0;

MappedFile unlockMappedFile = null;

//获取最新的日志文件CommitLog 内存映射文件 零拷贝

//mappedFileQueue 管理这些连续的CommitLog文件

//MappedFile 和 MappedFileQueue高性能的磁盘接口

//mappedFileQueue可以理解为commitLog文件夹，而MappedFile对应文件夹下的文件

MappedFile mappedFile = this.mappedFileQueue.getLastMappedFile();

//加锁，默认使用自旋锁。

//依赖于messageStoreConfig#useReentrantLockWhenPutMessage配置

//putMessage会有多个工作线程并行处理，所以需要加锁。串行写入commitLog

putMessageLock.lock(); //spin or ReentrantLock ,depending on store config

try {

long beginLockTimestamp = this.defaultMessageStore.getSystemClock().now();

this.beginTimeInLock = beginLockTimestamp;

// Here settings are stored timestamp, in order to ensure an orderly

// global

//再次设置时间戳全局有序

msg.setStoreTimestamp(beginLockTimestamp);

//文件已满或者没有映射文件重新创建一个文件

if (null == mappedFile || mappedFile.isFull()) {

mappedFile = this.mappedFileQueue.getLastMappedFile(0); // Mark: NewFile may be cause noise

}

//创建映射文件失败（可能磁盘已满）

if (null == mappedFile) {

log.error("create maped file1 error, topic: " + msg.getTopic() + " clientAddr: " + msg.getBornHostString());

//消息写入完成后，先将beginTimeInLock设置为0，然后释放锁

//该值用来计算消息写入耗时。写入新消息前，会根据该值来检查操作系统内存页写入是否繁忙

//如果上一条消息在1s内没有成功写入，则本次消息不再写入

beginTimeInLock = 0;

return new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, null);

}

//向映射文件中写入消息

//注意：只是将消息写入映射文件中的writeBuffer/mappedByteBuffer，没有刷盘

result = mappedFile.appendMessage(msg, this.appendMessageCallback);

switch (result.getStatus()) {

case PUT_OK: //消息成功写入

break;

//文件已经到结尾了，重新建一个新的mappedFile.

case END_OF_FILE: //当前CommitLog可用空间不足

unlockMappedFile = mappedFile;

// Create a new file, re-write the message

//创建新的CommitLog，并重新写入消息

mappedFile = this.mappedFileQueue.getLastMappedFile(0);

if (null == mappedFile) {

// XXX: warn and notify me

log.error("create maped file2 error, topic: " + msg.getTopic() + " clientAddr: " + msg.getBornHostString());

beginTimeInLock = 0;

return new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, result);

}

result = mappedFile.appendMessage(msg, this.appendMessageCallback);

break;

case MESSAGE_SIZE_EXCEEDED: //消息长度超过了最大阀值

case PROPERTIES_SIZE_EXCEEDED: //消息属性超过了最大阀值

beginTimeInLock = 0;

return new PutMessageResult(PutMessageStatus.MESSAGE_ILLEGAL, result);

case UNKNOWN_ERROR: //未知错误

beginTimeInLock = 0;

return new PutMessageResult(PutMessageStatus.UNKNOWN_ERROR, result);

default:

beginTimeInLock = 0;

return new PutMessageResult(PutMessageStatus.UNKNOWN_ERROR, result);

}

eclipseTimeInLock = this.defaultMessageStore.getSystemClock().now() - beginLockTimestamp;

beginTimeInLock = 0;

} finally {

putMessageLock.unlock(); //释放锁

}

if (eclipseTimeInLock > 500) {

log.warn("[NOTIFYME]putMessage in lock cost time(ms)={}, bodyLength={} AppendMessageResult={}", eclipseTimeInLock, msg.getBody().length, result);

}

if (null != unlockMappedFile && this.defaultMessageStore.getMessageStoreConfig().isWarmMapedFileEnable()) {

this.defaultMessageStore.unlockMappedFile(unlockMappedFile);

}

PutMessageResult putMessageResult = new PutMessageResult(PutMessageStatus.PUT_OK, result);

// Statistics Metrics指标

storeStatsService.getSinglePutMessageTopicTimesTotal(msg.getTopic()).incrementAndGet();

storeStatsService.getSinglePutMessageTopicSizeTotal(topic).addAndGet(result.getWroteBytes());

//同步刷盘或者异步刷盘

handleDiskFlush(result, putMessageResult, msg);

//主从同步

handleHA(result, putMessageResult, msg);

return putMessageResult;

}
```

