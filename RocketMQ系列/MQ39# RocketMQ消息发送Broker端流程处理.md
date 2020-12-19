---
title: MQ39# RocketMQ消息发送Broker端流程处理
categories: RocketMQ
tags: RocketMQ
date: 2020-12-20 23:39:01
---



# Broker处理消息的入口类SendMessageProcessor

processRequest方法主要三件事情：

1.处理consumer发回broker的消息重试

2.处理批量发送

3.处理单条消息发送

```
@Override

public RemotingCommand processRequest(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException {

SendMessageContext mqtraceContext;

switch (request.getCode()) {

//处理消息重试

case RequestCode.CONSUMER_SEND_MSG_BACK:

return this.consumerSendMsgBack(ctx, request);

default:

SendMessageRequestHeader requestHeader = parseRequestHeader(request);

if (requestHeader == null) {

return null;

}

mqtraceContext = buildMsgContext(ctx, requestHeader);

this.executeSendMessageHookBefore(ctx, request, mqtraceContext);

RemotingCommand response;

if (requestHeader.isBatch()) {

//批量发送

response = this.sendBatchMessage(ctx, request, mqtraceContext, requestHeader);

} else {

//处理消息发送

response = this.sendMessage(ctx, request, mqtraceContext, requestHeader);

}

this.executeSendMessageHookAfter(response, mqtraceContext);

return response;

}

}
```



# 单条处理流程

批处理流程与单条处理基本一致

SendMessageProcessor.sendMessage主要流程：

1.broker可以在指定的时间开始服务通过startAcceptSendRequestTimeStamp设定

2.消息校验：

\* broker没有写入权限并且topic为顺序topic则拒绝服务

\* 检查Topic不能和系统保留Topic[TBW102]冲突

\* 若Topic未创建，Broker开启自动创建

\* queueId校验，不能大于队列最大值

3.判断是否超过消费次数（16次），决定是否写入死信队列

4.消息内容组织

\* 设置Message扩展字段

\* 设置Message在客户端生成的时间

\* 设置发送Message机器的地址

\* 设置存储Message的Broker地址

\* 设置消费重试消息的次数

5.消息存储（单独梳理）

```
private RemotingCommand sendMessage(final ChannelHandlerContext ctx, //

final RemotingCommand request, //

final SendMessageContext sendMessageContext, //

final SendMessageRequestHeader requestHeader) throws RemotingCommandException {

final RemotingCommand response = RemotingCommand.createResponseCommand(SendMessageResponseHeader.class);

final SendMessageResponseHeader responseHeader = (SendMessageResponseHeader) response.readCustomHeader();

//标识RPC的SeqNumber

response.setOpaque(request.getOpaque());

//埋点不清楚用处

response.addExtField(MessageConst.PROPERTY_MSG_REGION, this.brokerController.getBrokerConfig().getRegionId());

response.addExtField(MessageConst.PROPERTY_TRACE_SWITCH, String.valueOf(this.brokerController.getBrokerConfig().isTraceOn()));

log.debug("receive SendMessage request command, {}", request);

//Broker启动后在设定的时间处理请求，通过startAcceptSendRequestTimeStamp来设置

final long startTimstamp = this.brokerController.getBrokerConfig().getStartAcceptSendRequestTimeStamp();

if (this.brokerController.getMessageStore().now() < startTimstamp) {

response.setCode(ResponseCode.SYSTEM_ERROR);

response.setRemark(String.format("broker unable to service, until %s", UtilAll.timeMillisToHumanString2(startTimstamp)));

return response;

}

response.setCode(-1);

//创建Topic

super.msgCheck(ctx, requestHeader, response);

if (response.getCode() != -1) {

return response;

}

final byte[] body = request.getBody();

int queueIdInt = requestHeader.getQueueId();

TopicConfig topicConfig = this.brokerController.getTopicConfigManager().selectTopicConfig(requestHeader.getTopic());

if (queueIdInt < 0) {

queueIdInt = Math.abs(this.random.nextInt() % 99999999) % topicConfig.getWriteQueueNums();

}

MessageExtBrokerInner msgInner = new MessageExtBrokerInner();

msgInner.setTopic(requestHeader.getTopic());

msgInner.setQueueId(queueIdInt);

//判断是否超过消费次数（16次），决定是否写入死信队列

if (!handleRetryAndDLQ(requestHeader, response, request, msgInner, topicConfig)) {

return response;

}

msgInner.setBody(body);

msgInner.setFlag(requestHeader.getFlag());

MessageAccessor.setProperties(msgInner, MessageDecoder.string2messageProperties(requestHeader.getProperties()));

//Message扩展字段，比如：Unikey, Keys, Tag都在这里面

msgInner.setPropertiesString(requestHeader.getProperties());

//Message在客户端生成的时间

msgInner.setBornTimestamp(requestHeader.getBornTimestamp());

//发送Message机器的地址

msgInner.setBornHost(ctx.channel().remoteAddress());

//存储Message的Broker地址

msgInner.setStoreHost(this.getStoreHost());

//重试消息的次数

msgInner.setReconsumeTimes(requestHeader.getReconsumeTimes() == null ? 0 : requestHeader.getReconsumeTimes());

//判断broker是否拒绝事物消息[rejectTransactionMessage]默认false

if (this.brokerController.getBrokerConfig().isRejectTransactionMessage()) {

String traFlag = msgInner.getProperty(MessageConst.PROPERTY_TRANSACTION_PREPARED);

if (traFlag != null) {

response.setCode(ResponseCode.NO_PERMISSION);

response.setRemark(

"the broker[" + this.brokerController.getBrokerConfig().getBrokerIP1() + "] sending transaction message is forbidden");

return response;

}

}

//消息存储

PutMessageResult putMessageResult = this.brokerController.getMessageStore().putMessage(msgInner);

return handlePutMessageResult(putMessageResult, response, request, msgInner, responseHeader, sendMessageContext, ctx, queueIdInt);

}
```



<!--more-->



# 处理消费重试消息

SendMessageProcess.consumerSendMsgBack

处理流程：

1.如果broker没有写入权限则返回拒绝写入

2.如果重试队列不存在则创建（%RETRY%+consumergroup）

3.根据offset（来自requestHeader）从commitlog中查找该条重试消息

4.将该该消息中Property中RETRY_TOPIC为空，将原Topic设置到该属性中

5.超过消费重试次数或者delayLevel为负数，进入死信队列

6.新消息构建

\* 设置新的Topic没有超过重试次数为%RETRY%+consumergroup，超过重试次数%DLQ%+consumergroup

\* 设置延迟级别delayLevel，每次重试逐级递增，首次为3 + msgExt.getReconsumeTimes()

\* 设置消息体、tagcode、queueId、sysFlag、BornTimestamp、BornHost、StoreHost、ReconsumeTimes

\* 将原msgId存储到property中的ORIGIN_MESSAGE_ID属性

7.消息存储

```
private RemotingCommand consumerSendMsgBack(final ChannelHandlerContext ctx, final RemotingCommand request)

throws RemotingCommandException {

final RemotingCommand response = RemotingCommand.createResponseCommand(null);

final ConsumerSendMsgBackRequestHeader requestHeader =

(ConsumerSendMsgBackRequestHeader) request.decodeCommandCustomHeader(ConsumerSendMsgBackRequestHeader.class);

if (this.hasConsumeMessageHook() && !UtilAll.isBlank(requestHeader.getOriginMsgId())) {

ConsumeMessageContext context = new ConsumeMessageContext();

context.setConsumerGroup(requestHeader.getGroup());

context.setTopic(requestHeader.getOriginTopic());

context.setCommercialRcvStats(BrokerStatsManager.StatsType.SEND_BACK);

context.setCommercialRcvTimes(1);

context.setCommercialOwner(request.getExtFields().get(BrokerStatsManager.COMMERCIAL_OWNER));

this.executeConsumeMessageHookAfter(context);

}

//消费组配置信息

SubscriptionGroupConfig subscriptionGroupConfig =

this.brokerController.getSubscriptionGroupManager().findSubscriptionGroupConfig(requestHeader.getGroup());

if (null == subscriptionGroupConfig) {

response.setCode(ResponseCode.SUBSCRIPTION_GROUP_NOT_EXIST);

response.setRemark("subscription group not exist, " + requestHeader.getGroup() + " "

\+ FAQUrl.suggestTodo(FAQUrl.SUBSCRIPTION_GROUP_NOT_EXIST));

return response;

}

if (!PermName.isWriteable(this.brokerController.getBrokerConfig().getBrokerPermission())) {

response.setCode(ResponseCode.NO_PERMISSION);

response.setRemark("the broker[" + this.brokerController.getBrokerConfig().getBrokerIP1() + "] sending message is forbidden");

return response;

}

if (subscriptionGroupConfig.getRetryQueueNums() <= 0) { //重试队列数量需要大于等于1

response.setCode(ResponseCode.SUCCESS);

response.setRemark(null);

return response;

}

String newTopic = MixAll.getRetryTopic(requestHeader.getGroup()); //重试队列

int queueIdInt = Math.abs(this.random.nextInt() % 99999999) % subscriptionGroupConfig.getRetryQueueNums(); //随机队列

int topicSysFlag = 0;

if (requestHeader.isUnitMode()) {

topicSysFlag = TopicSysFlag.buildSysFlag(false, true);

}

//创建重试队列

TopicConfig topicConfig = this.brokerController.getTopicConfigManager().createTopicInSendMessageBackMethod(//

newTopic, //

subscriptionGroupConfig.getRetryQueueNums(), //

PermName.PERM_WRITE | PermName.PERM_READ, topicSysFlag);

if (null == topicConfig) {

response.setCode(ResponseCode.SYSTEM_ERROR);

response.setRemark("topic[" + newTopic + "] not exist");

return response;

}

if (!PermName.isWriteable(topicConfig.getPerm())) {

response.setCode(ResponseCode.NO_PERMISSION);

response.setRemark(String.format("the topic[%s] sending message is forbidden", newTopic));

return response;

}

//从commitLog中查找消息

MessageExt msgExt = this.brokerController.getMessageStore().lookMessageByOffset(requestHeader.getOffset());

if (null == msgExt) {

response.setCode(ResponseCode.SYSTEM_ERROR);

response.setRemark("look message by offset failed, " + requestHeader.getOffset());

return response;

}

final String retryTopic = msgExt.getProperty(MessageConst.PROPERTY_RETRY_TOPIC);

if (null == retryTopic) {

MessageAccessor.putProperty(msgExt, MessageConst.PROPERTY_RETRY_TOPIC, msgExt.getTopic());

}

msgExt.setWaitStoreMsgOK(false);

int delayLevel = requestHeader.getDelayLevel();

int maxReconsumeTimes = subscriptionGroupConfig.getRetryMaxTimes();

if (request.getVersion() >= MQVersion.Version.V3_4_9.ordinal()) {

maxReconsumeTimes = requestHeader.getMaxReconsumeTimes();

}

//超过重试次数或者delayLevel为负数，进入死信队列人工干预

if (msgExt.getReconsumeTimes() >= maxReconsumeTimes//

|| delayLevel < 0) {

newTopic = MixAll.getDLQTopic(requestHeader.getGroup());

queueIdInt = Math.abs(this.random.nextInt() % 99999999) % DLQ_NUMS_PER_GROUP;

topicConfig = this.brokerController.getTopicConfigManager().createTopicInSendMessageBackMethod(newTopic, //

DLQ_NUMS_PER_GROUP, //

PermName.PERM_WRITE, 0

);

if (null == topicConfig) {

response.setCode(ResponseCode.SYSTEM_ERROR);

response.setRemark("topic[" + newTopic + "] not exist");

return response;

}

} else {

if (0 == delayLevel) {

delayLevel = 3 + msgExt.getReconsumeTimes();

}

msgExt.setDelayTimeLevel(delayLevel);

}

//构建新消息，会有新的消息Id

MessageExtBrokerInner msgInner = new MessageExtBrokerInner();

msgInner.setTopic(newTopic);

msgInner.setBody(msgExt.getBody());

msgInner.setFlag(msgExt.getFlag());

MessageAccessor.setProperties(msgInner, msgExt.getProperties());

msgInner.setPropertiesString(MessageDecoder.messageProperties2String(msgExt.getProperties()));

msgInner.setTagsCode(MessageExtBrokerInner.tagsString2tagsCode(null, msgExt.getTags()));

msgInner.setQueueId(queueIdInt);

msgInner.setSysFlag(msgExt.getSysFlag());

msgInner.setBornTimestamp(msgExt.getBornTimestamp());

msgInner.setBornHost(msgExt.getBornHost());

msgInner.setStoreHost(this.getStoreHost());

msgInner.setReconsumeTimes(msgExt.getReconsumeTimes() + 1);

String originMsgId = MessageAccessor.getOriginMessageId(msgExt); //设置原来的messageId ORIGIN_MESSAGE_ID

MessageAccessor.setOriginMessageId(msgInner, UtilAll.isBlank(originMsgId) ? msgExt.getMsgId() : originMsgId);

//消息存储

PutMessageResult putMessageResult = this.brokerController.getMessageStore().putMessage(msgInner);

if (putMessageResult != null) {

switch (putMessageResult.getPutMessageStatus()) {

case PUT_OK:

String backTopic = msgExt.getTopic();

String correctTopic = msgExt.getProperty(MessageConst.PROPERTY_RETRY_TOPIC);

if (correctTopic != null) {

backTopic = correctTopic;

}

this.brokerController.getBrokerStatsManager().incSendBackNums(requestHeader.getGroup(), backTopic);

response.setCode(ResponseCode.SUCCESS);

response.setRemark(null);

return response;

default:

break;

}

response.setCode(ResponseCode.SYSTEM_ERROR);

response.setRemark(putMessageResult.getPutMessageStatus().name());

return response;

}

response.setCode(ResponseCode.SYSTEM_ERROR);

response.setRemark("putMessageResult is null");

return response;

}
```

