---
title: MQ41# RocketMQ特性--事物消息
categories: RocketMQ
tags: RocketMQ
date: 2020-12-20 23:43:01
---



# 问题思考

从官方给的例子入手，代码如下:

示例类：org.apache.rocketmq.example.transaction.TransactionProducer.java

```
public static void main(String[] args) throws MQClientException, InterruptedException {

//@1 定义TransactionListener

TransactionListener transactionListener = new TransactionListenerImpl();

//@2 使用事务发送Producer

TransactionMQProducer producer = new TransactionMQProducer("please_rename_unique_group_name");

//@3 定义线程池

ExecutorService executorService = new ThreadPoolExecutor(2, 5, 100, TimeUnit.SECONDS, new ArrayBlockingQueue<Runnable>(2000), new ThreadFactory() {

@Override

public Thread newThread(Runnable r) {

Thread thread = new Thread(r);

thread.setName("client-transaction-msg-check-thread");

return thread;

}

});

//设置线程池

producer.setExecutorService(executorService);

//设置监听器

producer.setTransactionListener(transactionListener);

producer.setNamesrvAddr("127.0.0.1:9876");

//@4 发送者启动

producer.start();

String[] tags = new String[] {"TagA", "TagB", "TagC", "TagD", "TagE"};

for (int i = 0; i < 10; i++) {

try {

Message msg =

new Message("TopicTest1234", tags[i % tags.length], "KEY" + i,

("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET));

//@5 消息发送

SendResult sendResult = producer.sendMessageInTransaction(msg, null);

System.out.printf("%s%n", sendResult);

Thread.sleep(10);

} catch (MQClientException | UnsupportedEncodingException e) {

e.printStackTrace();

}

}

for (int i = 0; i < 100000; i++) {

Thread.sleep(1000);

}

//发送者关闭

producer.shutdown();

}
```

<!--more-->

从上面客户端例子中思考一些问题：

```
1. @1定义TransactionListener做什么用？
2. @2定义的TransactionMQProducer与普通Produer区别在哪里？
3. @3定义线程池executorService又是干啥的？
4. @4事务发送者启动发送流程是怎么样？
5. 发送事务消息如何和Broker进行交互的？
```



# 事务消息客户端发送流程

## 事务发送与普通启动差异

```
@1 producer.start();

@2 TransactionMQProducer#start

this.defaultMQProducerImpl.initTransactionEnv();

super.start();

@3 DefaultMQProducerImpl#initTransactionEnv()

this.checkExecutor = producer.getExecutorService();
```



```
小结：事务发送时比普通发送启动多了initTransactionEnv操作，即：给ExecutorService checkExecutor赋值。
```



## 事务消息发送调用链

```
@1 SendResult sendResult = producer.sendMessageInTransaction

@2 TransactionMQProducer#sendMessageInTransaction

@3 DefaultMQProducerImpl#sendMessageInTransaction
```



## 事务消息发送分析

方法：DefaultMQProducerImpl#sendMessageInTransaction

```
//@1

TransactionListener transactionListener = getCheckListener();

if (null == localTransactionExecuter && null == transactionListener) {

throw new MQClientException("tranExecutor is null", null);

}

Validators.checkMessage(msg, this.defaultMQProducer);

SendResult sendResult = null;

//@2 表示消息的prepare消息

MessageAccessor.putProperty(msg, MessageConst.PROPERTY_TRANSACTION_PREPARED, "true");

//@3 生产者组，用于回查本地事务事，从生产者组中选择随机选择一个生产者即可

MessageAccessor.putProperty(msg, MessageConst.PROPERTY_PRODUCER_GROUP, this.defaultMQProducer.getProducerGroup());

try {

//@4 消息发送

sendResult = this.send(msg);

} catch (Exception e) {

throw new MQClientException("send message Exception", e);

}
```



方法：DefaultMQProducerImpl#sendKernelImpl

```
//事务消息发送，设置PREPARED标记

final String tranMsg = msg.getProperty(MessageConst.PROPERTY_TRANSACTION_PREPARED);

if (tranMsg != null && Boolean.parseBoolean(tranMsg)) {

sysFlag |= MessageSysFlag.TRANSACTION_PREPARED_TYPE;

}

//@5 请求header中设置事务标记

requestHeader.setSysFlag(sysFlag);

//@6 发送消息请求的RequestCode

request = RemotingCommand.createRequestCommand(RequestCode.SEND_MESSAGE, requestHeader);
```



```
小结：
@1获取TransactionListener即示例代码传入的Listener
@2在消息属性中加入PROPERTY_TRANSACTION_PREPARED = "TRAN_MSG"即事务半消息
@3设置ProducerGroup Broker在事务回查时调用
@4事务消息发送采用同步发送，发送流程与普通消息发送一致
@5请求header中设置事务标记SEND_MESSAGE = 10
```



## 事务消息发送结果分析

方法：DefaultMQProducerImpl#sendMessageInTransaction

```
LocalTransactionState localTransactionState = LocalTransactionState.UNKNOW;

switch (sendResult.getSendStatus()) {

//@1

case SEND_OK: {

...

//@2 执行本地事务

localTransactionState = transactionListener.executeLocalTransaction(msg, arg);

...

}

break;

case FLUSH_DISK_TIMEOUT:

case FLUSH_SLAVE_TIMEOUT:

case SLAVE_NOT_AVAILABLE:

localTransactionState = LocalTransactionState.ROLLBACK_MESSAGE;

break;

default:

break;

}
```



```
小结：
@1 发送半消息（Prepared）消息成功，设置transactionId。
@2 发送半消息成功后，通过transactionListener回调客户端查询本地事务执行情况，并返回事务执行状态。
LocalTransactionState有COMMIT_MESSAGE、ROLLBACK_MESSAGE、UNKNOW三种状态。
```



## 结束事务分析

方法：DefaultMQProducerImpl#sendMessageInTransaction

```
try {

//@1 结束事务，根据返回的事务状态执行提交、回滚、暂时不处理

this.endTransaction(sendResult, localTransactionState, localException);

} catch (Exception e) {

log.warn("local transaction execute " + localTransactionState + ", but end broker transaction failed", e);

}

TransactionSendResult transactionSendResult = new TransactionSendResult();

...

return transactionSendResult;
```



方法：DefaultMQProducerImpl#endTransaction

```
...

switch (localTransactionState) {

//@2 设置事务提交标记Header

case COMMIT_MESSAGE: requestHeader.setCommitOrRollback(MessageSysFlag.TRANSACTION_COMMIT_TYPE);

break;

//@3 设置事务回滚标记Header

case ROLLBACK_MESSAGE:

requestHeader.setCommitOrRollback(MessageSysFlag.TRANSACTION_ROLLBACK_TYPE);

break;

//@4 设置事务未知标记Header

case UNKNOW:

requestHeader.setCommitOrRollback(MessageSysFlag.TRANSACTION_NOT_TYPE);

break;

default:

break;

}

//@5 通过一次发送方式向Broker提交事务 this.mQClientFactory.getMQClientAPIImpl().endTransactionOneway(brokerAddr, requestHeader, remark,

this.defaultMQProducer.getSendMsgTimeout());

}
```



```
小结：
@1 根据本地事务执行返回的状态localTransactionState，调用结束事务方法
@2 requestHeader设置事务提交标记0x2 << 2=8
@3 requestHeader设置事务回滚标记0x3 << 2=12
@4 requestHeader设置未知标记0
@5 通过一次发送方式向Broker提交事务 RequestCode为END_TRANSACTION = 37
```



# 事务消息服务端存储流程

## 事务消息存储调用链

```
@1 SendMessageProcessor#processRequest

response = this.sendMessage(ctx, request, mqtraceContext, requestHeader)

@2 SendMessageProcessor#sendMessage
```



## 事务半消息存储代码分析（一）

方法：SendMessageProcessor#sendMessage

```
//@1 可以通过配置来是否接受事务消息存储

if (traFlag != null && Boolean.parseBoolean(traFlag)) {

if (this.brokerController.getBrokerConfig().isRejectTransactionMessage()){

response.setCode(ResponseCode.NO_PERMISSION);

response.setRemark(

"the broker[" + this.brokerController.getBrokerConfig().getBrokerIP1()

\+ "] sending transaction message is forbidden");

return response;

}

//@2 prepare消息存储

putMessageResult = this.brokerController.getTransactionalMessageService().prepareMessage(msgInner);

} else {

putMessageResult = this.brokerController.getMessageStore().putMessage(msgInner);

}

```

```
小结：
@1 可以通过Broker配置属性rejectTransactionMessage来决定是否接受事务消息请求，默认为false即接受。
@2 半消息存储
```



<!--more-->



## 事务半消息存储代码分析（二）

方法：TransactionalMessageBridge#putHalfMessage

```
public PutMessageResult putHalfMessage(MessageExtBrokerInner messageInner){

return store.putMessage(parseHalfMessageInner(messageInner));

}
```



方法：TransactionalMessageBridge#parseHalfMessageInner

```
//@1 备份原主题

MessageAccessor.putProperty(msgInner, MessageConst.PROPERTY_REAL_TOPIC, msgInner.getTopic());

//@2 备份原queueID

MessageAccessor.putProperty(msgInner, MessageConst.PROPERTY_REAL_QUEUE_ID,

String.valueOf(msgInner.getQueueId()));

//@3 重置sysFlag

msgInner.setSysFlag(

MessageSysFlag.resetTransactionValue(msgInner.getSysFlag(), MessageSysFlag.TRANSACTION_NOT_TYPE));

//@3 主题变更 RMQ_SYS_TRANS_HALF_TOPIC

msgInner.setTopic(TransactionalMessageUtil.buildHalfTopic());

//@4 消息队列变更为0

msgInner.setQueueId(0);

msgInner.setPropertiesString(MessageDecoder.messageProperties2String(msgInner.getProperties()));
```



```
小结：半消息在存储前将存储的主题设置为RMQ_SYS_TRANS_HALF_TOPIC，将原来的Topic备份到属性中，同时也备份了原来的QueueId。这也是为什么半消息不会被消费者消费的原因。
```

# 事务消息服务端响应结束事务请求

## 处理未知类型请求

方法：EndTransactionProcessor#processRequest

```
case MessageSysFlag.TRANSACTION_NOT_TYPE: {

LOGGER.warn("The producer[{}] end transaction in sending message, and it's pending status."

\+ "RequestHeader: {} Remark: {}",

RemotingHelper.parseChannelRemoteAddr(ctx.channel()),

requestHeader.toString(),

request.getRemark());

//@1

return null;

}
```



```
小结：结束事务在处理未知类型TRANSACTION_NOT_TYPE时，只打印告警日志不做处理。
```



## 处理事务提交请求

半消息查找。方法：EndTransactionProcessor#processRequest

```
if (MessageSysFlag.TRANSACTION_COMMIT_TYPE

== requestHeader.getCommitOrRollback()){

//@1 将prepare消息找出来

result = this.brokerController.getTransactionalMessageService()

.commitMessage(requestHeader);

...

//@2

MessageExtBrokerInner msgInner = endMessageTransaction(result.getPrepareMessage());

}
```



半消息还原。方法：EndTransactionProcessor#endMessageTransaction

```
MessageExtBrokerInner msgInner = new MessageExtBrokerInner();

//@3 置换为原来的Topic

msgInner.setTopic(msgExt.getUserProperty(MessageConst.PROPERTY_REAL_TOPIC));

//@4 置换为原来的QueueId

msgInner.setQueueId(Integer.parseInt(msgExt.getUserProperty(MessageConst.PROPERTY_REAL_QUEUE_ID)));

...

//清除属性设置

MessageAccessor.clearProperty(msgInner, MessageConst.PROPERTY_REAL_TOPIC);

MessageAccessor.clearProperty(msgInner, MessageConst.PROPERTY_REAL_QUEUE_ID);

return msgInner;
```



```
小结：
@1 根据偏移量将半消息查找出来
@2 将存储在RMQ_SYS_TRANS_HALF_TOPIC还原
@3 置换为原来的Topic
@4 置换为原来的QueueId
```



还原后消息存储。方法：EndTransactionProcessor#processRequest

```
//@1 新组装的消息存储（提交）

RemotingCommand sendResult = sendFinalMessage(msgInner);

if (sendResult.getCode() == ResponseCode.SUCCESS) {

//@2 删除prepare消息 是将消息存储于RMQ_SYS_TRANS_OP_HALF_TOPIC中

this.brokerController.getTransactionalMessageService().deletePrepareMessage(result.getPrepareMessage());

}
```



```
小结：
@1 将还原后的消息存储
@2 删除半消息消息
```

半消息删除。方法：TransactionalMessageServiceImpl#putOpMessage

```
public boolean putOpMessage(MessageExt messageExt, String opType) {

MessageQueue messageQueue = new MessageQueue(messageExt.getTopic(),

this.brokerController.getBrokerConfig().getBrokerName(), messageExt.getQueueId());

if (TransactionalMessageUtil.REMOVETAG.equals(opType)) {

return addRemoveTagInTransactionOp(messageExt, messageQueue);

}

return true;

}
```

```
private boolean addRemoveTagInTransactionOp(MessageExt messageExt, MessageQueue messageQueue) {

//@1 主题变更为RMQ_SYS_TRANS_OP_HALF_TOPIC

Message message = new Message(TransactionalMessageUtil.buildOpTopic(), TransactionalMessageUtil.REMOVETAG, String.valueOf(messageExt.getQueueOffset()).getBytes(TransactionalMessageUtil.charset));

//@2 存储消息

writeOp(message, messageQueue);

return true;

}
```



```
小结：半消息的删除是将Topic从RMQ_SYS_TRANS_HALF_TOPIC变更为RMQ_SYS_TRANS_OP_HALF_TOPIC存储到日志文件，依靠文件删除机制删除。
```

## 处理事务回滚请求

方法：EndTransactionProcessor#processRequest

```
//@1 查找半消息

result = this.brokerController.getTransactionalMessageService().rollbackMessage(requestHeader);

if (result.getResponseCode() == ResponseCode.SUCCESS) {

if (res.getCode() == ResponseCode.SUCCESS) {

//@2 删除prepare消息 是将消息存储于RMQ_SYS_TRANS_OP_HALF_TOPIC中 this.brokerController.getTransactionalMessageService().deletePrepareMessage(result.getPrepareMessage());

}

return res;

}

}
```



```
小结：处理事务回滚请求，将半消息查找出来，将其删除即：将Topic从RMQ_SYS_TRANS_HALF_TOPIC变更为RMQ_SYS_TRANS_OP_HALF_TOPIC并存储，依靠文件删除机制删除。
```



# 事务消息服务端状态回查

## 事务回查线程类调用链

线程类初始化：TransactionalMessageCheckService

```
@1 main(String[] args)

start(createBrokerController(args));

@2 createBrokerController

@3 initialize()

@4 initialTransaction()

this.transactionalMessageCheckService =

new TransactionalMessageCheckService(this);
```



```
小结：在Broker初始化启动时，TransactionalMessageCheckService线程类也随着启动初始化。
```



## 事务回查逻辑

方法：TransactionalMessageCheckService#run

```
//@1 时间间隔为60秒

long checkInterval = brokerController.getBrokerConfig().getTransactionCheckInterval();

while (!this.isStopped()) {

this.waitForRunning(checkInterval);

}
```



方法：TransactionalMessageCheckService#onWaitEnd

```
//@2 transactionTimeOut默认6秒

long timeout = brokerController.getBrokerConfig().getTransactionTimeOut();

//@3 最大核查次数为15次

int checkMax = brokerController.getBrokerConfig().getTransactionCheckMax();

long begin = System.currentTimeMillis();

this.brokerController.getTransactionalMessageService().check(timeout, checkMax, this.brokerController.getTransactionalMessageCheckListener());
```



```
小结：事务回查每隔60秒执行一次，一次执行超时时间为6秒，最大回查次数为15次。
```



回查逻辑（一）。方法：TransactionalMessageServiceImpl#check

```
//@1

Set<MessageQueue> msgQueues = transactionalMessageBridge.fetchMessageQueues(topic);

for (MessageQueue messageQueue : msgQueues) {

//获取对应的RMQ_SYS_TRANS_OP_HALF_TOPIC中的队列

MessageQueue opQueue = getOpQueue(messageQueue);

//半消息消费队列中偏移量

long halfOffset = transactionalMessageBridge.fetchConsumeOffset(messageQueue);

//OP已删除消费队列中的偏移量

long opOffset = transactionalMessageBridge.fetchConsumeOffset(opQueue);

}

//@2

PullResult pullResult = fillOpRemoveMap(removeMap, opQueue, opOffset, halfOffset, doneOpOffset);
```



方法：TransactionalMessageServiceImpl#fillOpRemoveMap

```
//@3

PullResult pullResult = pullOpMsg(opQueue, pullOffsetOfOp, 32);

List<MessageExt> opMsg = pullResult.getMsgFoundList();

for (MessageExt opMessageExt : opMsg) {

Long queueOffset = getLong(new String(opMessageExt.getBody(), TransactionalMessageUtil.charset));

if (TransactionalMessageUtil.REMOVETAG.equals(opMessageExt.getTags())) {

//已经处理过的消息即commit和rollback

if (queueOffset < miniOffset) {

doneOpOffset.add(opMessageExt.getQueueOffset());

} else {

//已经处理删除过了，但是半消息还没有更新

removeMap.put(queueOffset, opMessageExt.getQueueOffset());

}

} else {

log.error("Found a illegal tag in opMessageExt= {} ", opMessageExt);

}

}

```



```
小结：
@1 从半消息队列中查找消息队列
@2 opQueue队列中的消息均为已经删除的半消息，需要检查下是否已经删除了，当时半消息队列还没有更新。
@3 miniOffset为半消息消费队列中的最大偏移量；queueOffset为删除消费队列的消息偏移量；通过比较两者来确定是否已经删除了，而半消息状态还没有更新，并将这类消息存储在removeMap中。
```

回查逻辑（二）。方法：TransactionalMessageServiceImpl#check

```
while (true) {

//查找消息

GetResult getResult = getHalfMsg(messageQueue, i);

//@1 已经处理过了，半消息滞后了，偏移量继续递增往下走

if (removeMap.containsKey(i)) {

log.info("Half offset {} has been committed/rolled back", i);

removeMap.remove(i);

}

//@2 needSkip 超过存储时间（默认3天） needDiscard 超过回查次数，默认15次

if (needDiscard(msgExt, transactionCheckMax) || needSkip(msgExt)) {

listener.resolveDiscardMsg(msgExt);

newOffset = i + 1;

i++;

continue;

}

//@3 消息存储时间大于开始时间暂不处理

if (msgExt.getStoreTimestamp() >= startTime) {

break;

}

//@4 存储的时间小于需要回查的时间 跳过

if (valueOfCurrentMinusBorn < checkImmunityTime) {

if (checkPrepareQueueOffset(removeMap, doneOpOffset, msgExt)) {

newOffset = i + 1;

i++;

continue;

}

}

//接着往下处理

newOffset = i + 1;

i++;

}
```



```
小结
@1 removeMap（即已删除队列有而半消息队列未更新的消息）有则不在处理跳过该消息。
@2 超过存储时间或者回查次数超过15次不再处理
@3 消息存储时间大于核查程序开始时间暂不处理
@4 如果定义了回查的时间间隔需要判断是否到时间了
```

回查逻辑（三）。方法：TransactionalMessageServiceImpl#check

```
if (isNeedCheck) {

//@1

if (!putBackHalfMsgQueue(msgExt, i)) {

continue;

}

//@2

listener.resolveHalfMsg(msgExt);

}} else {

}

//@3

//保存prepare消息队列的回查消费进度

if (newOffset != halfOffset) {

transactionalMessageBridge.updateConsumeOffset(messageQueue, newOffset);

}

long newOpOffset = calculateOpOffset(doneOpOffset, opOffset);

//保存OP消费进度

if (newOpOffset != opOffset) {

transactionalMessageBridge.updateConsumeOffset(opQueue, newOpOffset);

}
```



```
小结：
@1 将半消息重新存储在RMQ_SYS_TRANS_HALF_TOPIC中，由于本次回查尚未知道结果，所以进行存储。
@2 发到客户端进行回查，回查的RequestCode为CHECK_TRANSACTION_STATE = 39，根据ProductGroup随机获取客户端通道Channel进行回查。
@3 保存半消息和已处理消息的消费进度。
```



## 客户端响应事务回查

方法：ClientRemotingProcessor#checkTransactionState

```
producer.checkTransactionState(addr, messageExt, requestHeader);
```



方法：MQProducerInner#checkTransactionState

```
//@1

localTransactionState = transactionListener.checkLocalTransaction(message);

//@2

DefaultMQProducerImpl.this.mQClientFactory.getMQClientAPIImpl().endTransactionOneway(brokerAddr, thisHeader, remark,

3000);
```



```
小结：
@1 执行本地事务回查并返回事务回查状态
@2 将事务回查状态提交到Broker
```



# 事务消息交互示意图

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219151320.png)