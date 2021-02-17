---
title: MQ16# RocketMQ一次延迟消息故障排查
categories: RocketMQ
tags: RocketMQ
date: 2020-12-20 20:16:01
---



# 问题描述

RocketMQ社区版本支持18个延迟级别，每个级别在设定的时间都被会消费者准确消费到。为此也专门测试过消费的间隔是不是准确，测试结果显示很准确。

然而，如此准确的特性居然出问题了，接到业务同学报告线上某个集群延迟消息消费不到，开发环境、测试环境都没问题。各个环境的版本都是统一的RocketMQ 4.5.2。诡异！

<!--more-->

# 临时方案

将该业务的topic和consumer转移到线上其他集群，延迟消息消费正常。



# 问题定位



## 搜查日志

发现storeerror.log一直刷下面的日志，观察发现offset远远超过了存储的大小！

```
2020-03-18 16:38:06 WARN ScheduleMessageTimerThread - Offset not matched. Request offset: 509548460, firstOffset: 0, lastOffset: 6000000, mappedFileSize: 6000000, mappedFiles count: 1

2020-03-18 16:38:06 WARN ScheduleMessageTimerThread - Offset not matched. Request offset: 665682360, firstOffset: 0, lastOffset: 6000000, mappedFileSize: 6000000, mappedFiles count: 1

2020-03-18 16:38:07 WARN ScheduleMessageTimerThread - Offset not matched. Request offset: 509548460, firstOffset: 0, lastOffset: 6000000, mappedFileSize: 6000000, mappedFiles count: 1

2020-03-18 16:38:07 WARN ScheduleMessageTimerThread - Offset not matched. Request offset: 665682360, firstOffset: 0, lastOffset: 6000000, mappedFileSize: 6000000, mappedFiles count: 1

2020-03-18 16:38:07 WARN ScheduleMessageTimerThread - Offset not matched. Request offset: 509548460, firstOffset: 0, lastOffset: 6000000, mappedFileSize: 6000000, mappedFiles count: 1

2020-03-18 16:38:07 WARN ScheduleMessageTimerThread - Offset not matched. Request offset: 665682360, firstOffset: 0, lastOffset: 6000000, mappedFileSize: 6000000, mappedFiles count: 1

2020-03-18 16:38:07 WARN ScheduleMessageTimerThread - Offset not matched. Request offset: 509548460, firstOffset: 0, lastOffset: 6000000, mappedFileSize: 6000000, mappedFiles count: 1
```



## 消息查询

通过查询发送延迟的消息，发现该消息存储成功；已经存储在了“SCHEDULE_TOPIC_XXXX”中了。

```
bin/mqadmin queryMsgById -n x.x.x.x:9876 -i 0A6F160300002A9F000000BACA44720C

RocketMQLog:WARN No appenders could be found for logger (io.netty.util.internal.PlatformDependent0).

RocketMQLog:WARN Please initialize the logger system properly.

OffsetID: 0A6F160300002A9F000000BACA44720C

OffsetID: 0A6F160300002A9F000000BACA44720C

Topic: SCHEDULE_TOPIC_XXXX

Tags: [null]

Keys: [rocketmq.key.-1351724170]

Queue ID: 3

Queue Offset: 142

CommitLog Offset: 802257400332

Reconsume Times: 0

Born Timestamp: 2020-03-18 15:27:37,512

Store Timestamp: 2020-03-18 15:27:37,513

Born Host: 10.x.x.x:56650

Store Host: 10.x.x.x:10911

System Flag: 0

Properties: {REAL_TOPIC=melon_online_test, KEYS=rocketmq.key.-1351724170, UNIQ_KEY=0A6F15AB6305070DEA4E5ADD60280023, WAIT=true, DELAY=4, REAL_QID=1}

Message Body Path: /tmp/rocketmq/msgbodys/0A6F15AB6305070DEA4E5ADD60280023

org.apache.rocketmq.client.exception.MQClientException: CODE: 17 DESC: No topic route info in name server for the topic: SCHEDULE_TOPIC_XXXX

See http://rocketmq.apache.org/docs/faq/ for further details.

at org.apache.rocketmq.client.impl.MQClientAPIImpl.getTopicRouteInfoFromNameServer(MQClientAPIImpl.java:1351)

at org.apache.rocketmq.client.impl.MQClientAPIImpl.getTopicRouteInfoFromNameServer(MQClientAPIImpl.java:1321)

at org.apache.rocketmq.tools.admin.DefaultMQAdminExtImpl.examineTopicRouteInfo(DefaultMQAdminExtImpl.java:305)

at org.apache.rocketmq.tools.admin.DefaultMQAdminExtImpl.queryTopicConsumeByWho(DefaultMQAdminExtImpl.java:619)

at org.apache.rocketmq.tools.admin.DefaultMQAdminExtImpl.messageTrackDetail(DefaultMQAdminExtImpl.java:776)

at org.apache.rocketmq.tools.admin.DefaultMQAdminExt.messageTrackDetail(DefaultMQAdminExt.java:435)

at org.apache.rocketmq.tools.command.message.QueryMsgByIdSubCommand.printMsg(QueryMsgByIdSubCommand.java:145)

at org.apache.rocketmq.tools.command.message.QueryMsgByIdSubCommand.queryById(QueryMsgByIdSubCommand.java:49)

at org.apache.rocketmq.tools.command.message.QueryMsgByIdSubCommand.execute(QueryMsgByIdSubCommand.java:252)

at org.apache.rocketmq.tools.command.MQAdminStartup.main0(MQAdminStartup.java:138)

at org.apache.rocketmq.tools.command.MQAdminStartup.main(MQAdminStartup.java:89)
```



## 原理回顾

延迟消息存储时被替换为“SCHEDULE_TOPIC_XXXX”主题，broker会为每个等级的建立定时任务进行调度，将各个等级到时间的消息替换为原来的主题，从而消费者可以消费到该消息。

以上日志加上消息查询结果已存储在“SCHEDULE_TOPIC_XXXX”，可以断定调度环节出问题了，由于offset远远超过最大offset而报错；没能将目标主题成功替换。



## 报错源码

```
public MappedFile findMappedFileByOffset(final long offset, final boolean returnFirstOnNotFound) {

try {

MappedFile firstMappedFile = this.getFirstMappedFile();

MappedFile lastMappedFile = this.getLastMappedFile();

if (firstMappedFile != null && lastMappedFile != null) {

// 这里报错了，每次用offset=509548460来查，可是总大小才6000000

// 每次来查报错返回null

if (offset < firstMappedFile.getFileFromOffset() || offset >= lastMappedFile.getFileFromOffset() + this.mappedFileSize) {

LOG_ERROR.warn("Offset not matched. Request offset: {}, firstOffset: {}, lastOffset: {}, mappedFileSize: {}, mappedFiles count: {}",

offset,

firstMappedFile.getFileFromOffset(),

lastMappedFile.getFileFromOffset() + this.mappedFileSize,

this.mappedFileSize,

this.mappedFiles.size());

} else {

// ...

}

} catch (Exception e) {

log.error("findMappedFileByOffset Exception", e);

}

return null;

}
```



调用位置：ScheduleMessageService#executeOnTimeup

```
// 根据offset捞取消息内容

MessageExt msgExt = ScheduleMessageService.this.defaultMessageStore.lookMessageByOffset(
offsetPy, sizePy);

// 处理结束后处理位点自增

nextOffset = offset + (i / ConsumeQueue.CQ_STORE_UNIT_SIZE);
ScheduleMessageService.this.updateOffset(this.delayLevel, nextOffset);
```



其中offsetPy来自ConsumeQueue；另外delayOffset.json存储延迟消息的处理进度。



# 解决方式

将"delayOffset.json"和"consumequeue/SCHEDULE_TOPIC_XXXX"移到其他目录，相当于删除；逐台重启broker节点。重启结束后，经过验证，延迟消息功能正常发送和消费。



# 结束了吗？

看到这里你可能还有疑问，offset如何能增长到665682360这么大的？而且offset完全由broker自身去调度后自增的，并非由客户端上报的。记得前年RocketMQ也出现过类似问题，那时候RocketMQ版本是4.1，在测试环境延迟消息失效。此问题很罕见，一年未必碰到一会，后面会提交给社区讨论下。