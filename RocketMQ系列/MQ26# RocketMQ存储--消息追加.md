---
title: MQ26# RocketMQ存储--消息追加
categories: RocketMQ
tags: RocketMQ
date: 2020-12-20 20:45:01
---



# 问题

1.消息追加到何处了呢？

2.消息格式是怎么样的？



<!--more-->



# 调用链

```
@1 CommitLog#putMessage

result = mappedFile.appendMessage(msg, this.appendMessageCallback);

@2 MaapedFile#appendMessagesInner

result = cb.doAppend

@3 CommitLog#DefaultAppendMessageCallback#doAppend
```



# 流程图

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219134505.png)

```
小结：在消息写入Buffer的过程中有3个坐标
```


**1. wrotePosition**

\* commitLog内存（ByteBuffer）写入位点，标记消息写到哪了，下次从该位置开始写。在消息写完后递增，递增大小为消息的长度

**2. wroteOffset**

\* 物理偏移量，标记在commitlog物理文件中消息的位置

\* 物理偏移量=文件名称（fileFromOffset）+ 内存相对位置byteBuffer.position(wrotePosition)

**3. queueOffset**

\* topicQueue逻辑偏移量，标记消息在topic的分区中的消息的位置，在消息写入后递增，递增长度为1

```
疑问？
写入Buffer有两类，堆外内存Buffer(writeBuffer)和mmap映射Buffer(mappedByteBuffer)。mappedByteBuffer为commitLog日志文件的直接映射，而堆外内存writeBuffer是怎么落盘的呢？
```

此处先记录疑问，分析刷盘时回头再看。

\###### 四、消息格式

在追加单条消息时，第4步组织消息，格式如下表格：

| 序号 | 内容 | 所占空间 |

| --- | --- | --- |

| 1 | msgLen消息长度 | 4个字节 |

| 2 | MAGIC_CODE 魔数 | 4个字节 |

| 3 | BodyCRC 校验码 | 4个字节 |

| 4 | QueueId 消息所在的分区|4个字节 |

| 5 | 消息Flag | 4个字节 |

| 6 | queueOffset 分区偏移量| 8个字节 |

| 7 | fileFromOffset + byteBuffer.position() 物理偏移量 | 8个字节 |

| 8 | SysFlag 系统标记压缩等| 4个字节 |

| 9 | BornTimestamp 发送时间| 8个字节 |

| 10 |BornHost 发送的机器IP| 8个字节 |

| 11 | StoreTimestamp 存储时间| 8个字节 |

| 12 | StoreHost 存储的broker|8个字节 |

| 13 | ReconsumeTimes 消费重试次数|4个字节 |

| 14 | PreparedTransactionOffset 事物消息偏移量| 8字节 |

| 15 | bodyLength 消息体长度 | |

| 16 | body 消息体内容 | |

| 17 | topicLength 主题长度| |

| 18 | topicData主题内容 | |

| 19 | propertiesLength 属性长度| |

| 20 | propertiesData 属性内容 | |

```
**小结：1到14项是每条消息都有的，所占空间为84个字节**
```



<!--more-->



# 总结

**1.消息追加到何处了呢？**

注：消息追加内存Buffer中，分两类。堆外内存Buffer(writeBuffer)和mmap映射Buffer(mappedByteBuffer)

**2.消息格式是怎么样的？**

消息格式顺序见第四部分。



# 附录源代码

**消息追加源代码**

```
public AppendMessageResult appendMessagesInner(final MessageExt messageExt, final AppendMessageCallback cb) {

assert messageExt != null;

assert cb != null;

//当前MappedFile的写入位置

int currentPos = this.wrotePosition.get();

//文件还有剩余空间（小于1G继续写入）

if (currentPos < this.fileSize) {

//仅当transientStorePoolEnable 为true，刷盘策略为异步刷盘（FlushDiskType为ASYNC_FLUSH）,并且broker为主节点时，才启用堆外分配内存。此时：writeBuffer不为null

//Buffer与同步和异步刷盘相关

//writeBuffer/mappedByteBuffer的position始终为0，而limit则始终等于capacity

//slice创建一个新的buffer, 是根据position和limit来生成byteBuffer

ByteBuffer byteBuffer = writeBuffer != null ? writeBuffer.slice() : this.mappedByteBuffer.slice();

byteBuffer.position(currentPos); //设置写的起始位置

AppendMessageResult result = null;

//处理单个消息

if (messageExt instanceof MessageExtBrokerInner) {

result = cb.doAppend(this.getFileFromOffset(), byteBuffer, this.fileSize - currentPos, (MessageExtBrokerInner) messageExt);

//处理批量消息

} else if (messageExt instanceof MessageExtBatch) {

result = cb.doAppend(this.getFileFromOffset(), byteBuffer, this.fileSize - currentPos, (MessageExtBatch)messageExt);

} else {

return new AppendMessageResult(AppendMessageStatus.UNKNOWN_ERROR);

}

this.wrotePosition.addAndGet(result.getWroteBytes());//修改写的位置

this.storeTimestamp = result.getStoreTimestamp();

return result;

}

//写满会报错，正常不会进入该代码，调用该方法前有判断

log.error("MappedFile.appendMessage return null, wrotePosition: {} fileSize: {}", currentPos, this.fileSize);

return new AppendMessageResult(AppendMessageStatus.UNKNOWN_ERROR);

}
```



**消息格式源代码**

```java
public AppendMessageResult doAppend(final long fileFromOffset, final ByteBuffer byteBuffer, final int maxBlank,

final MessageExtBrokerInner msgInner) {

// STORETIMESTAMP + STOREHOSTADDRESS + OFFSET <br>

//fileFromOffset(起始位置): 一个commitLog文件（对应一个MappedFile）对应的偏移量（文件名就代表这个偏移量）

//byteBuffer.position()(相对位置):当前MappedFile(对应一个commitLog)的写位置

//wroteOffset：绝对位置

// PHY OFFSET

long wroteOffset = fileFromOffset + byteBuffer.position();

this.resetByteBuffer(hostHolder, 8);

//根据broker存储的地址和消息的物理绝对位置生成唯一的MessageId

String msgId = MessageDecoder.createMessageId(this.msgIdMemory, msgInner.getStoreHostBytes(hostHolder), wroteOffset);

// Record ConsumeQueue information

//消息队列（ConsumeQueue）逻辑偏移量

keyBuilder.setLength(0);

keyBuilder.append(msgInner.getTopic());

keyBuilder.append('-');

keyBuilder.append(msgInner.getQueueId());

String key = keyBuilder.toString();

Long queueOffset = CommitLog.this.topicQueueTable.get(key);

if (null == queueOffset) {

queueOffset = 0L;

CommitLog.this.topicQueueTable.put(key, queueOffset);

}

// Transaction messages that require special handling

final int tranType = MessageSysFlag.getTransactionValue(msgInner.getSysFlag());

switch (tranType) {

// Prepared and Rollback message is not consumed, will not enter the

// consumer queuec

case MessageSysFlag.TRANSACTION_PREPARED_TYPE:

case MessageSysFlag.TRANSACTION_ROLLBACK_TYPE:

queueOffset = 0L;

break;

case MessageSysFlag.TRANSACTION_NOT_TYPE:

case MessageSysFlag.TRANSACTION_COMMIT_TYPE:

default:

break;

}

final byte[] propertiesData =

msgInner.getPropertiesString() == null ? null : msgInner.getPropertiesString().getBytes(MessageDecoder.CHARSET_UTF8);

final int propertiesLength = propertiesData == null ? 0 : propertiesData.length;

if (propertiesLength > Short.MAX_VALUE) {

log.warn("putMessage message properties length too long. length={}", propertiesData.length);

return new AppendMessageResult(AppendMessageStatus.PROPERTIES_SIZE_EXCEEDED);

}

final byte[] topicData = msgInner.getTopic().getBytes(MessageDecoder.CHARSET_UTF8);

final int topicLength = topicData.length;

final int bodyLength = msgInner.getBody() == null ? 0 : msgInner.getBody().length;

//计算message大小

final int msgLen = calMsgLength(bodyLength, topicLength, propertiesLength);

// Exceeds the maximum message

if (msgLen > this.maxMessageSize) {

CommitLog.log.warn("message size exceeded, msg total size: " + msgLen + ", msg body size: " + bodyLength

\+ ", maxMessageSize: " + this.maxMessageSize);

return new AppendMessageResult(AppendMessageStatus.MESSAGE_SIZE_EXCEEDED);

}

//确定当前这个Commietlog文件是否有足够的可用空间存储

//maxBlank:当前这个Commitlog文件（对应的MappedFile）的剩余空间

//一个Message不能跨越两个Commitlog

//每个CommitLog文件都要确保预留8个字节来表示这个CommitLog文件结尾

// Determines whether there is sufficient free space

if ((msgLen + END_FILE_MIN_BLANK_LENGTH) > maxBlank) {

this.resetByteBuffer(this.msgStoreItemMemory, maxBlank);

// 1 TOTALSIZE

this.msgStoreItemMemory.putInt(maxBlank);

// 2 MAGICCODE

//表示一个CommitLog文件结尾魔数，当读到这个魔数表示文件已结束

this.msgStoreItemMemory.putInt(CommitLog.BLANK_MAGIC_CODE);

// 3 The remaining space may be any value

//

// Here the length of the specially set maxBlank

final long beginTimeMills = CommitLog.this.defaultMessageStore.now();

//将消息内容存储在ByteBuffer中，此处存储在MappedFile对应的内存映射Buffer中，并没有写入到磁盘

byteBuffer.put(this.msgStoreItemMemory.array(), 0, maxBlank);

return new AppendMessageResult(AppendMessageStatus.END_OF_FILE, wroteOffset, maxBlank, msgId, msgInner.getStoreTimestamp(),

queueOffset, CommitLog.this.defaultMessageStore.now() - beginTimeMills);

}

// Initialization of storage space

this.resetByteBuffer(msgStoreItemMemory, msgLen);

// 1 TOTALSIZE 该消息条目长度，4个字节

this.msgStoreItemMemory.putInt(msgLen);

// 2 MAGICCODE 魔数，4字节。固定值0xdaa320a7

this.msgStoreItemMemory.putInt(CommitLog.MESSAGE_MAGIC_CODE);

// 3 BODYCRC 消息体CRC校验码，4个字节

this.msgStoreItemMemory.putInt(msgInner.getBodyCRC());

// 4 QUEUEID 消息消费队列ID 4个字节

this.msgStoreItemMemory.putInt(msgInner.getQueueId());

// 5 FLAG 消息FLAG，RocketMQ不做处理，供应用程序使用，默认4个字节

this.msgStoreItemMemory.putInt(msgInner.getFlag());

// 6 QUEUEOFFSET 消息在消息消费队列的偏移量，8个字节

this.msgStoreItemMemory.putLong(queueOffset);

// 7 PHYSICALOFFSET 消息在CommitLog文件中的偏移量，8字节

this.msgStoreItemMemory.putLong(fileFromOffset + byteBuffer.position());

// 8 SYSFLAG 消息系统Flag，例如是否压缩、是否事务消息 4字节

this.msgStoreItemMemory.putInt(msgInner.getSysFlag());

// 9 BORNTIMESTAMP 消息生产者调用消息发送API的时间戳，8个字节

this.msgStoreItemMemory.putLong(msgInner.getBornTimestamp());

// 10 BORNHOST 消息发送者IP、端口号、8字节

this.resetByteBuffer(hostHolder, 8);

this.msgStoreItemMemory.put(msgInner.getBornHostBytes(hostHolder));

// 11 STORETIMESTAMP 消息存储的时间戳，8字节

this.msgStoreItemMemory.putLong(msgInner.getStoreTimestamp());

// 12 STOREHOSTADDRESS Broker服务器IP+端口号，8字节

this.resetByteBuffer(hostHolder, 8);

this.msgStoreItemMemory.put(msgInner.getStoreHostBytes(hostHolder));

//this.msgBatchMemory.put(msgInner.getStoreHostBytes());

// 13 RECONSUMETIMES 消息重试次数，4字节

this.msgStoreItemMemory.putInt(msgInner.getReconsumeTimes());

// 14 Prepared Transaction Offset 事务消息物理偏移量，8字节

this.msgStoreItemMemory.putLong(msgInner.getPreparedTransactionOffset());

// 15 BODY 消息体内容，长度为bodyLenth中存储的值

this.msgStoreItemMemory.putInt(bodyLength);

if (bodyLength > 0)

this.msgStoreItemMemory.put(msgInner.getBody());

// 16 TOPIC 主题，长度为TopicLength中存储的值

this.msgStoreItemMemory.put((byte) topicLength);

this.msgStoreItemMemory.put(topicData);

// 17 PROPERTIES 消息属性长度，属性长度不能超过32767（short的最大值）

this.msgStoreItemMemory.putShort((short) propertiesLength);

if (propertiesLength > 0)

this.msgStoreItemMemory.put(propertiesData);

final long beginTimeMills = CommitLog.this.defaultMessageStore.now();

// Write messages to the queue buffer

byteBuffer.put(this.msgStoreItemMemory.array(), 0, msgLen);

AppendMessageResult result = new AppendMessageResult(AppendMessageStatus.PUT_OK, wroteOffset, msgLen, msgId,

msgInner.getStoreTimestamp(), queueOffset, CommitLog.this.defaultMessageStore.now() - beginTimeMills);

switch (tranType) {

case MessageSysFlag.TRANSACTION_PREPARED_TYPE:

case MessageSysFlag.TRANSACTION_ROLLBACK_TYPE:

break;

case MessageSysFlag.TRANSACTION_NOT_TYPE:

case MessageSysFlag.TRANSACTION_COMMIT_TYPE:

// The next update ConsumeQueue information

CommitLog.this.topicQueueTable.put(key, ++queueOffset);

break;

default:

break;

}

return result;

}
```

