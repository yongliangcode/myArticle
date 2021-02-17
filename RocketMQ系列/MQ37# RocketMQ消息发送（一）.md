---
title: MQ37# RocketMQ消息发送（一）
categories: RocketMQ
tags: RocketMQ
date: 2020-12-20 23:19:01
---



# 消息发送代码

\* 需要设置produerGroup

\* 需要设置NameServer地址

```
DefaultMQProducer producer = new DefaultMQProducer("melon-tst");

producer.setNamesrvAddr("localhost:9876");

producer.setVipChannelEnabled(false);

producer.start();

for(int i=0;i<100;i++){

Message msg = new Message("topic_online_test",("Hello RocketMQ" + i).getBytes(RemotingHelper.DEFAULT_CHARSET));

//msg.setDelayTimeLevel(10);

SendResult sendResult = producer.send(msg);

System.out.printf("%s%n",sendResult);

}

producer.shutdown();
```

<!--more-->

# 方法启动所做的事情

\* 将instanceName从默认值DEFAULT修改为PID

1. MQClientInstance封装了网络通信的管道，存储于factoryTable（ConcurrentHashMap）

2. factoryTable为MQClientManager的成员变量，MQClientManager是单例模式

3. key为clientId对应一个MQClientInstance，被客户端共享使用

4. clientId的组成ClientIP@InstanceName，在同一个客户端连接多个集群时需要修改ClientIP或者InstanceName以确保clientId唯一

\* 注册producer到producerTable（ConcurrentHashMap）, key为producerGroup名称，不同的produer需要设置不同的producerGroup名称

\* 客户端工厂实例启动

\* 设置默认主题TBW102的路由信息

\* 向各个broker发送心跳包

```
//defaultMQProducerImpl.start()

public void start(final boolean startFactory) throws MQClientException {

//注意：此处的serviceState默认为CREATE_JUST 是DefaultMQProducer的成员变量

switch (this.serviceState) {

case CREATE_JUST:

this.serviceState = ServiceState.START_FAILED;

//合法性校验

this.checkConfig();

//将实例的名称改成PID 避免一台机器上启动多个实例造成clientId重名

if (!this.defaultMQProducer.getProducerGroup().equals(MixAll.CLIENT_INNER_PRODUCER_GROUP)) {

this.defaultMQProducer.changeInstanceNameToPID();

}

//获取MQClientInstance,作为生产者与NameServer,Broker沟通的通道

this.mQClientFactory = MQClientManager.getInstance().getAndCreateMQClientInstance(this.defaultMQProducer, rpcHook);

//将当前生产者加入到MQClientInstance管理中，方便后续调用网络请求、进行心跳检测

boolean registerOK = mQClientFactory.registerProducer(this.defaultMQProducer.getProducerGroup(), this);

//一个ProductGroup只允许注册一次

if (!registerOK) {

this.serviceState = ServiceState.CREATE_JUST;

throw new MQClientException("The producer group[" + this.defaultMQProducer.getProducerGroup()

\+ "] has been created before, specify another name please." + FAQUrl.suggestTodo(FAQUrl.GROUP_NAME_DUPLICATE_URL),

null);

}

//设置默认主题TBW102的路由信息

this.topicPublishInfoTable.put(this.defaultMQProducer.getCreateTopicKey(), new TopicPublishInfo());

if (startFactory) {

mQClientFactory.start();

}

log.info("the producer [{}] start OK. sendMessageWithVIPChannel={}", this.defaultMQProducer.getProducerGroup(),

this.defaultMQProducer.isSendMessageWithVIPChannel());

this.serviceState = ServiceState.RUNNING;

break;

case RUNNING:

case START_FAILED:

case SHUTDOWN_ALREADY:

throw new MQClientException("The producer service state not OK, maybe started once, "//

\+ this.serviceState//

\+ FAQUrl.suggestTodo(FAQUrl.CLIENT_SERVICE_NOT_OK),

null);

default:

break;

}

this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();

}

/**

\* 同一个JVM中的不同消费者和不同生产者在启动时获取的MQClientInstance实例都是同一个

\* MQClientInstance封装了RocketMQ网络处理的API，是消息生产者（Producer）、消息消费者(Consumer)与NameServer、Broker打交道的网络通道

\* @param clientConfig

\* @param rpcHook

\* @return

*/

public MQClientInstance getAndCreateMQClientInstance(final ClientConfig clientConfig, RPCHook rpcHook) {

//clientId由ip地址@实例名称构成

//如果ProducerGroup为CLIENT_INNER_PRODUCER，实例名称为被更改为PID进程ID

String clientId = clientConfig.buildMQClientId();

MQClientInstance instance = this.factoryTable.get(clientId);

if (null == instance) {

instance =

new MQClientInstance(clientConfig.cloneClientConfig(),

this.factoryIndexGenerator.getAndIncrement(), clientId, rpcHook);

MQClientInstance prev = this.factoryTable.putIfAbsent(clientId, instance);

if (prev != null) {

instance = prev;

log.warn("Returned Previous MQClientInstance for clientId:[{}]", clientId);

} else {

log.info("Created new MQClientInstance for clientId:[{}]", clientId);

}

}

return instance;

}

public String buildMQClientId() {

StringBuilder sb = new StringBuilder();

sb.append(this.getClientIP());

sb.append("@");

sb.append(this.getInstanceName());

if (!UtilAll.isBlank(this.unitName)) {

sb.append("@");

sb.append(this.unitName);

}

return sb.toString();

}
```



<!--more-->



# 客户端工厂实例启动

\* 开启消息通道（Netty客户端启动）

\* 启动系列定时任务

1. 每30秒定时从NameServer获取Topic的路由信息

2. 每30秒定时清理下线的broker以及向broker发送心跳

3. 持久化消息消费进度，默认5秒保存一次（本地存储和Broker存储）

\* 开启拉去消息的线程pullMessageService

\* 队列消费负载实现

\* 发送消息服务启动

```
//MQClientInstance mQClientFactory.start()

public void start(final boolean startFactory) throws MQClientException {

//注意：此处的serviceState默认为CREATE_JUST 是DefaultMQProducer的成员变量

switch (this.serviceState) {

case CREATE_JUST:

this.serviceState = ServiceState.START_FAILED;

//合法性校验

this.checkConfig();

//将实例的名称改成PID 避免一台机器上启动多个实例造成clientId重名

if (!this.defaultMQProducer.getProducerGroup().equals(MixAll.CLIENT_INNER_PRODUCER_GROUP)) {

this.defaultMQProducer.changeInstanceNameToPID();

}

//获取MQClientInstance,作为生产者与NameServer,Broker沟通的通道

this.mQClientFactory = MQClientManager.getInstance().getAndCreateMQClientInstance(this.defaultMQProducer, rpcHook);

//将当前生产者加入到MQClientInstance管理中，方便后续调用网络请求、进行心跳检测

boolean registerOK = mQClientFactory.registerProducer(this.defaultMQProducer.getProducerGroup(), this);

//一个ProductGroup只允许注册一次

if (!registerOK) {

this.serviceState = ServiceState.CREATE_JUST;

throw new MQClientException("The producer group[" + this.defaultMQProducer.getProducerGroup()

\+ "] has been created before, specify another name please." + FAQUrl.suggestTodo(FAQUrl.GROUP_NAME_DUPLICATE_URL),

null);

}

//设置默认主题的路由信息

this.topicPublishInfoTable.put(this.defaultMQProducer.getCreateTopicKey(), new TopicPublishInfo());

if (startFactory) {

mQClientFactory.start();

}

log.info("the producer [{}] start OK. sendMessageWithVIPChannel={}", this.defaultMQProducer.getProducerGroup(),

this.defaultMQProducer.isSendMessageWithVIPChannel());

this.serviceState = ServiceState.RUNNING;

break;

case RUNNING:

case START_FAILED:

case SHUTDOWN_ALREADY:

throw new MQClientException("The producer service state not OK, maybe started once, "//

\+ this.serviceState//

\+ FAQUrl.suggestTodo(FAQUrl.CLIENT_SERVICE_NOT_OK),

null);

default:

break;

}

//向各个broker发送心跳包

this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();

}
```



# 消息发送

\* 获取Topic路由信息

1. 先从缓存topicPublishInfoTable中获取

2. 没有再从NameServer中请求获取

3. 依然没有则使用默认topic（TBW102）的路由信息

\* 选择一个MessageQueue进行发送

\* 组装requestHeader发送消息

1. 设置客户端MsgId

2. 超过4K消息压缩设置压缩消息标记

3. 设置事务消息标记

4. 判断发送前钩子执行

5. 消息发送完钩子执行



```java
private SendResult sendDefaultImpl(//

Message msg, //待发送的消息

final CommunicationMode communicationMode, //

final SendCallback sendCallback, //

final long timeout//

) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {

//确保初始化完成

this.makeSureStateOK();

//消息校验

Validators.checkMessage(msg, this.defaultMQProducer);

final long invokeID = random.nextLong();

long beginTimestampFirst = System.currentTimeMillis();

long beginTimestampPrev = beginTimestampFirst;

long endTimestamp = beginTimestampFirst;

//获取Topic的路由信息,1.本地缓存 2.NameServer 3.TBW102 默认Topic的路由信息

TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());

if (topicPublishInfo != null && topicPublishInfo.ok()) {

MessageQueue mq = null;

Exception exception = null;

SendResult sendResult = null;

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

try {

beginTimestampPrev = System.currentTimeMillis();

sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout);

endTimestamp = System.currentTimeMillis();

this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);

switch (communicationMode) {

case ASYNC:

return null;

case ONEWAY:

return null;

case SYNC:

if (sendResult.getSendStatus() != SendStatus.SEND_OK) {

if (this.defaultMQProducer.isRetryAnotherBrokerWhenNotStoreOK()) {

continue;

}

}

return sendResult;

default:

break;

}

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

endTimestamp = System.currentTimeMillis();

this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true);

log.warn(String.format("sendKernelImpl exception, resend at once, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mq), e);

log.warn(msg.toString());

exception = e;

switch (e.getResponseCode()) {

case ResponseCode.TOPIC_NOT_EXIST:

case ResponseCode.SERVICE_NOT_AVAILABLE:

case ResponseCode.SYSTEM_ERROR:

case ResponseCode.NO_PERMISSION:

case ResponseCode.NO_BUYER_ID:

case ResponseCode.NOT_IN_CURRENT_UNIT:

continue;

default:

if (sendResult != null) {

return sendResult;

}

throw e;

}

} catch (InterruptedException e) {

endTimestamp = System.currentTimeMillis();

this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);

log.warn(String.format("sendKernelImpl exception, throw exception, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mq), e);

log.warn(msg.toString());

log.warn("sendKernelImpl exception", e);

log.warn(msg.toString());

throw e;

}

} else {

break;

}

}

if (sendResult != null) {

return sendResult;

}

String info = String.format("Send [%d] times, still failed, cost [%d]ms, Topic: %s, BrokersSent: %s",

times,

System.currentTimeMillis() - beginTimestampFirst,

msg.getTopic(),

Arrays.toString(brokersSent));

info += FAQUrl.suggestTodo(FAQUrl.SEND_MSG_FAILED);

MQClientException mqClientException = new MQClientException(info, exception);

if (exception instanceof MQBrokerException) {

mqClientException.setResponseCode(((MQBrokerException) exception).getResponseCode());

} else if (exception instanceof RemotingConnectException) {

mqClientException.setResponseCode(ClientErrorCode.CONNECT_BROKER_EXCEPTION);

} else if (exception instanceof RemotingTimeoutException) {

mqClientException.setResponseCode(ClientErrorCode.ACCESS_BROKER_TIMEOUT);

} else if (exception instanceof MQClientException) {

mqClientException.setResponseCode(ClientErrorCode.BROKER_NOT_EXIST_EXCEPTION);

}

throw mqClientException;

}

//没有设置NameServer地址错误

List<String> nsList = this.getmQClientFactory().getMQClientAPIImpl().getNameServerAddressList();

if (null == nsList || nsList.isEmpty()) {

throw new MQClientException(

"No name server address, please set it." + FAQUrl.suggestTodo(FAQUrl.NAME_SERVER_ADDR_NOT_EXIST_URL), null).setResponseCode(ClientErrorCode.NO_NAME_SERVER_EXCEPTION);

}

//发送抛错没有找到Topic路由信息

throw new MQClientException("No route info of this topic, " + msg.getTopic() + FAQUrl.suggestTodo(FAQUrl.NO_TOPIC_ROUTE_INFO),

null).setResponseCode(ClientErrorCode.NOT_FOUND_TOPIC_EXCEPTION);

}

private SendResult sendKernelImpl(final Message msg, //待发送的消息

final MessageQueue mq, //将消息发送到该队列上

final CommunicationMode communicationMode, //消息发送模式，SYNC、ASYNC、ONEWAY

final SendCallback sendCallback, //异步消息回调函数

final TopicPublishInfo topicPublishInfo, //主题路由信息

final long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {

String brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());

if (null == brokerAddr) {

tryToFindTopicPublishInfo(mq.getTopic());

brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());

}

SendMessageContext context = null;

if (brokerAddr != null) {

brokerAddr = MixAll.brokerVIPChannel(this.defaultMQProducer.isSendMessageWithVIPChannel(), brokerAddr);

byte[] prevBody = msg.getBody();

try {

//for MessageBatch,ID has been set in the generating process

//设置客户端MsgId

if (!(msg instanceof MessageBatch)) {

MessageClientIDSetter.setUniqID(msg);

}

int sysFlag = 0;

//超过4K消息压缩 压缩消息标记

if (this.tryToCompressMessage(msg)) {

sysFlag |= MessageSysFlag.COMPRESSED_FLAG;

}

//事务消息标记

final String tranMsg = msg.getProperty(MessageConst.PROPERTY_TRANSACTION_PREPARED);

if (tranMsg != null && Boolean.parseBoolean(tranMsg)) {

sysFlag |= MessageSysFlag.TRANSACTION_PREPARED_TYPE;

}

//限制发送钩子

if (hasCheckForbiddenHook()) {

CheckForbiddenContext checkForbiddenContext = new CheckForbiddenContext();

checkForbiddenContext.setNameSrvAddr(this.defaultMQProducer.getNamesrvAddr());

checkForbiddenContext.setGroup(this.defaultMQProducer.getProducerGroup());

checkForbiddenContext.setCommunicationMode(communicationMode);

checkForbiddenContext.setBrokerAddr(brokerAddr);

checkForbiddenContext.setMessage(msg);

checkForbiddenContext.setMq(mq);

checkForbiddenContext.setUnitMode(this.isUnitMode());

this.executeCheckForbiddenHook(checkForbiddenContext);

}

//发送前钩子

if (this.hasSendMessageHook()) {

context = new SendMessageContext();

context.setProducer(this);

context.setProducerGroup(this.defaultMQProducer.getProducerGroup());

context.setCommunicationMode(communicationMode);

context.setBornHost(this.defaultMQProducer.getClientIP());

context.setBrokerAddr(brokerAddr);

context.setMessage(msg);

context.setMq(mq);

String isTrans = msg.getProperty(MessageConst.PROPERTY_TRANSACTION_PREPARED);

if (isTrans != null && isTrans.equals("true")) {

context.setMsgType(MessageType.Trans_Msg_Half);

}

if (msg.getProperty("__STARTDELIVERTIME") != null || msg.getProperty(MessageConst.PROPERTY_DELAY_TIME_LEVEL) != null) {

context.setMsgType(MessageType.Delay_Msg);

}

this.executeSendMessageHookBefore(context);

}

//组装RequestHeader

SendMessageRequestHeader requestHeader = new SendMessageRequestHeader();

requestHeader.setProducerGroup(this.defaultMQProducer.getProducerGroup());//生产者组

requestHeader.setTopic(msg.getTopic());//主题名称

requestHeader.setDefaultTopic(this.defaultMQProducer.getCreateTopicKey());//默认创建主题key

requestHeader.setDefaultTopicQueueNums(this.defaultMQProducer.getDefaultTopicQueueNums());//队列数量

requestHeader.setQueueId(mq.getQueueId());//队列ID

requestHeader.setSysFlag(sysFlag);//消息系统标记 标志压缩，事务消息

requestHeader.setBornTimestamp(System.currentTimeMillis());//消息发送时间

requestHeader.setFlag(msg.getFlag());//消息标记，RocketMQ中不做处理

requestHeader.setProperties(MessageDecoder.messageProperties2String(msg.getProperties()));//消息扩展属性

requestHeader.setReconsumeTimes(0);//重试第一次为0

requestHeader.setUnitMode(this.isUnitMode());//问题？：默认false不清楚做何使用

requestHeader.setBatch(msg instanceof MessageBatch);//是否批量消息

//重试队列设置requestHeader

if (requestHeader.getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {

//获取已消费的次数

String reconsumeTimes = MessageAccessor.getReconsumeTime(msg);

if (reconsumeTimes != null) {

requestHeader.setReconsumeTimes(Integer.valueOf(reconsumeTimes));

MessageAccessor.clearProperty(msg, MessageConst.PROPERTY_RECONSUME_TIME);

}

//最大消费次数

String maxReconsumeTimes = MessageAccessor.getMaxReconsumeTimes(msg);

if (maxReconsumeTimes != null) {

requestHeader.setMaxReconsumeTimes(Integer.valueOf(maxReconsumeTimes));

MessageAccessor.clearProperty(msg, MessageConst.PROPERTY_MAX_RECONSUME_TIMES);

}

}

//调用通道发送消息

SendResult sendResult = null;

switch (communicationMode) {

case ASYNC:

sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(//

brokerAddr, // 1

mq.getBrokerName(), // 2

msg, // 3

requestHeader, // 4

timeout, // 5

communicationMode, // 6

sendCallback, // 7

topicPublishInfo, // 8

this.mQClientFactory, // 9

this.defaultMQProducer.getRetryTimesWhenSendAsyncFailed(), // 10

context, //

this);

break;

case ONEWAY:

case SYNC:

sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(

brokerAddr,

mq.getBrokerName(),

msg,

requestHeader,

timeout,

communicationMode,

context,

this);

break;

default:

assert false;

break;

}

//消息发送完毕钩子

if (this.hasSendMessageHook()) {

context.setSendResult(sendResult);

this.executeSendMessageHookAfter(context);

}

return sendResult;

} catch (RemotingException e) {

if (this.hasSendMessageHook()) {

context.setException(e);

this.executeSendMessageHookAfter(context);

}

throw e;

} catch (MQBrokerException e) {

if (this.hasSendMessageHook()) {

context.setException(e);

this.executeSendMessageHookAfter(context);

}

throw e;

} catch (InterruptedException e) {

if (this.hasSendMessageHook()) {

context.setException(e);

this.executeSendMessageHookAfter(context);

}

throw e;

} finally {

msg.setBody(prevBody);

}

}

throw new MQClientException("The broker[" + mq.getBrokerName() + "] not exist", null);

}
```

