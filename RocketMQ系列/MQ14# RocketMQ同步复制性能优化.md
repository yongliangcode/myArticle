---
title: MQ14# RocketMQ同步复制性能优化
categories: RocketMQ
tags: RocketMQ
date: 2020-12-20 20:14:01
---



# 问题描述

早些时候写过性能测试和性能优化文章，主要基于异步刷盘/异步复制；由于业务需要需要搭建异步刷盘/同步复制集群；同时对性能进行压测。

```
压测结果显示集群几乎无法使用，TPS居然是个位数，客户端也在报错。
```

## 压测日志

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201218231410.png)



## 客户端日志

```
2019-09-19 19:22:38,038 ERROR RocketmqClient - [BENCHMARK_PRODUCER] Send Exception

org.apache.rocketmq.client.exception.MQBrokerException: CODE: 2 DESC: [TIMEOUT_CLEAN_QUEUE]broker busy, start flow control for a while, period in queue: 209ms, size of queue: 9

For more information, please visit the url, http://rocketmq.apache.org/docs/faq/

at org.apache.rocketmq.client.impl.MQClientAPIImpl.processSendResponse(MQClientAPIImpl.java:671)

at org.apache.rocketmq.client.impl.MQClientAPIImpl.sendMessageSync(MQClientAPIImpl.java:467)

at org.apache.rocketmq.client.impl.MQClientAPIImpl.sendMessage(MQClientAPIImpl.java:449)

at
```



# 解决发送失败情况

经排查，将transientStorePoolEnable关闭(默认为false)；压测显示最高TPS有1.9万。

```
brokerRole=SYNC_MASTER

\#transientStorePoolEnable=true
```

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201218231653.png)



# 解决发送TPS过低情况

最高TPS只有1.9万，依然过低，与预期相差甚远，我们预期压测应该可以到7到8万这样可以满足业务发展需要。再次检查broker端参数配置，没有发现有参数导致性能如此过低。

```
回顾性能调优的几个方面：系统调优、集群调优、JVM调优。
```

系统调优与集群调优都已经做过了，唯一没有优化的JVM调优，堆内存设置默认的8G。

JAVA_OPT="${JAVA_OPT} -server -Xms8g -Xmx8g -Xmn4g"

将JVM堆内存提高4倍后，压测效果明显提升，基本可以达到预期7万多的TPS。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201218231833.png)



<!--more-->



# 原因分析

1.为什么在异步刷盘/同步复制时开启堆外内存池transientStorePoolEnable后，集群压测几乎无法进行？

2.为什么在异步刷盘/同步复制时调大JVM堆内存后，性能明显提升呢？提升了的倍数几乎是堆内存增大的倍数。

## 刷盘流程回顾



[RocketMQ存储--消息追加【源码笔记】](http://mp.weixin.qq.com/s?__biz=MzAxMzY2MDYzMA==&mid=2247483827&idx=1&sn=d7204473069dbad47b0cc7e05ad83acc&chksm=9b9e790aace9f01c84d0e9d686309875064a22704d843cdb941fbebe46728dfb338dc4cfa4bd&scene=21#wechat_redirect)

[RocketMQ存储--同步刷盘和异步刷盘【源码笔记】](http://mp.weixin.qq.com/s?__biz=MzAxMzY2MDYzMA==&mid=2247483855&idx=1&sn=710f8b2eb72ec28ff10eb7dbaf44dc7f&chksm=9b9e7976ace9f0608b5da4d4cf9676580aa599e7ce239b6aba9736cc6c60ff8efafb7910f7b1&scene=21#wechat_redirect)



**异步刷盘未开启堆外缓存示意图**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201218232152.png)

**异步刷盘开启堆外缓存示意图**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201218232214.png)

```
小结：异步刷盘未开启transientStorePoolEnable时，消息追加到mappedByteBuffer中，异步线程刷调用mappedByteBuffer.force落盘；异步刷盘开启transientStorePoolEnable时，消息写入wrtieBuffer中，异步线程将消息提交到fileChannel，然后异步线程调用fileChannel.force落盘。
```



## 主从复制回顾

[RocketMQ存储--主从同步【源码笔记】](http://mp.weixin.qq.com/s?__biz=MzAxMzY2MDYzMA==&mid=2247483701&idx=1&sn=9760e50c69285aad2d868a13d810394e&chksm=9b9e798cace9f09aa5d31b5c4b972e627b3fe1afcb892d40fbf57c561b18175bd12119e8c08c&scene=21#wechat_redirect)

HAConnection#WriteSocketService负责向Slave发送数据

```
//查找待拉取偏移量之后所有的可读消息

SelectMappedBufferResult selectResult = HAConnection.this.haService.getDefaultMessageStore().getCommitLogData(this.nextTransferFromWhere);

// ...

SelectMappedBufferResult result = mappedFile.selectMappedBuffer(pos);

// ...

ByteBuffer byteBuffer = this.mappedByteBuffer.slice();

byteBuffer.position(pos);

int size = readPosition - pos; //计算距离最大可读位置的大小

ByteBuffer byteBufferNew = byteBuffer.slice();

byteBufferNew.limit(size);

return new SelectMappedBufferResult()
```

```
小结：主从复制使用mappedByteBuffer向Slave同步数据。
```

## 流程模拟

### 开启堆外内存池流程

```
@Test

public void test01(){

// 堆外内存池transientStorePoolEnable开启后，消息追加操作

try {

File file = new File("/Users/yongliang/logs/temp.log");

FileChannel fileChannel = new RandomAccessFile(file, "rw").getChannel();

String data = "beautiful girl!";

// mmap 文件映射操作

MappedByteBuffer mappedByteBuffer = fileChannel.map(FileChannel.MapMode.READ_WRITE, 0, data.length());

// 堆外内存transientStorePoolEnable开启

ByteBuffer byteBuffer = ByteBuffer.allocateDirect(data.length());

// ----------------消息追加开始-----------------------

// 注意此时使用堆内内存分配

ByteBuffer msgStoreItemMemory = ByteBuffer.allocate(data.length());

msgStoreItemMemory.put(data.getBytes());

// 开启transientStorePoolEnable消息写入了ByteBuffer

byteBuffer.put(msgStoreItemMemory.array(),0,data.length());

// ----------------消息追加结束-----------------------

// ----------------消息提交开始-----------------------

byteBuffer.position(0);

byteBuffer.limit(data.length());

fileChannel.write(byteBuffer);

// ----------------消息提交结束-----------------------

// --------主从复制从mappedByteBuffer获取消息开始----------

mappedByteBuffer.position(0);

mappedByteBuffer.limit(data.length());

Charset charset = Charset.forName("UTF-8");

CharsetDecoder decoder = charset.newDecoder();

CharBuffer charBuffer = decoder.decode(mappedByteBuffer.asReadOnlyBuffer());

System.out.println(charBuffer.toString());

// --------主从复制从mappedByteBuffer获取消息结束----------

} catch (Exception e) {

e.printStackTrace();

}

}
```

```
小结：模拟开启堆外内存池transientStorePoolEnable的消息追加及主从复制流程。
```

### 未开启堆外内存池流程

```
@Test

public void test02(){

try {

File file = new File("/Users/yongliang/logs/temp1.log");

FileChannel fileChannel = new RandomAccessFile(file, "rw").getChannel();

String data = "beautiful girl!";

// mmap 文件映射操作

MappedByteBuffer mappedByteBuffer = fileChannel.map(FileChannel.MapMode.READ_WRITE, 0, data.length());

// ----------------消息追加开始-----------------------

// 注意消息组装使用堆内内存分配

ByteBuffer msgStoreItemMemory = ByteBuffer.allocate(data.length());

msgStoreItemMemory.put(data.getBytes());

mappedByteBuffer.put(msgStoreItemMemory.array(),0,data.length());

// ----------------消息追加结束-----------------------

// --------主从复制从mappedByteBuffer获取消息开始----------

mappedByteBuffer.position(0);

mappedByteBuffer.limit(data.length());

Charset charset = Charset.forName("UTF-8");

CharsetDecoder decoder = charset.newDecoder();

CharBuffer charBuffer = decoder.decode(mappedByteBuffer.asReadOnlyBuffer());

System.out.println(charBuffer.toString());

// --------主从复制从mappedByteBuffer获取消息结束----------

} catch (Exception e) {

e.printStackTrace();

}

}
```



```
小结：模拟未开启堆外内存池transientStorePoolEnable的消息追加及主从复制流程。
```

# 原因总结

1.为什么在异步刷盘/同步复制时开启堆外内存transientStorePoolEnable后，集群压测几乎无法进行？

解释：

1>主从同步复制使用mappedByteBuffer；

2>开启堆外内存池transientStorePoolEnable后数据先落到WriteBuffer，再通过异步提交线程提交到FileChannel，再通过mmap将数据映射到mappedByteBuffer；

3>未开启堆外内存池transientStorePoolEnable数据直接写入到mappedByteBuffe；

由于开启堆外内存数据映射到mappedByteBuffer比直接写入mappedByteBuffer多了很多步骤，再加上发送队列处理事件默认只有200毫秒（waitTimeMillsInSendQueue=200），造成集群不能正常压测的原因。

2.为什么在异步刷盘/同步复制时调大JVM堆内存后，性能明显提升呢？提升了的倍数几乎是对内存增大的倍数。

解释：

从模拟流程中可以看出，在组装消息时使用堆内存，提高堆内存显著提高写入Tps的原因所在。

```
// 注意消息组装使用堆内内存分配

ByteBuffer msgStoreItemMemory = ByteBuffer.allocate(data.length());
```

