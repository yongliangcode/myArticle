---
title: MQ27# RocketMQ存储--同步刷盘和异步刷盘
categories: RocketMQ
tags: RocketMQ
date: 2020-12-20 20:47:01
---



# 问题思考

1.同步刷盘是怎么工作的？

2.异步刷盘是怎么工作的？

3.上篇文章的疑问，写入堆外内存的消息如何落盘的？



<!--more-->



# Broker启动刷盘有关调用链



**调用链**



```
//初始化链条

@1 BrokerStartup#main

start(createBrokerController(args));

@2 BrokerStartup#createBrokerController

final BrokerController controller = new BrokerController(...)

boolean initResult = controller.initialize();

@3 BrokerController#initialize

this.messageStore = new DefaultMessageStore(...);

@4 DefaultMessageStore#DefaultMessageStore()

this.commitLog = new CommitLog(this);

@5 CommitLog#CommitLog()

if (FlushDiskType.SYNC_FLUSH == defaultMessageStore.getMessageStoreConfig()

.getFlushDiskType()) {

this.flushCommitLogService = new GroupCommitService();

} else {

this.flushCommitLogService = new FlushRealTimeService();

}

this.commitLogService = new CommitRealTimeService();

//启动链条

@6 BrokerStartup#start

controller.start();

@7 BrokerController#start()

this.messageStore.start();

@8 DefaultMessageStore#start()

this.commitLog.start();

@9 CommitLog#start()

this.flushCommitLogService.start();

if (defaultMessageStore.getMessageStoreConfig()

.isTransientStorePoolEnable()) {

this.commitLogService.start();

}
```



<u>小结：由调用链可以看出，初始化并启动了以下线程类</u>

\* 同步刷盘 GroupCommitService

\* 异步刷盘 FlushRealTimeService

\* 如果开启堆外内存并且为异步刷盘 CommitRealTimeService



**线程类关系图**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219134925.png)



# 线程类工作流程

既然线程类在Broker启动时就启动了，他们在做啥呢？



**堆外内存线程类**CommitRealTimeService工作流程

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219134952.png)

```
小结：
1.CommitRealTimeService主要工作是将写入堆外内存(writeBuffer)的消息，写入到fileChannel中，fileChannel为commitLog文件通道
2.committedPosition用于记录将writeBuffer数据写入到fileChannel中的内存位点（相对偏移量offset）
3.committedWhere用于记录写入fileChannel中的物理偏移量（文件名称+相对偏移量offset）
```



**同步刷盘线程类GroupCommitService工作流程**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219135027.png)

**注1：**

\* 执行onWaitEnd时交换读写容器，该线程类提供两个容器来装GroupCommitRequest

\* requestsWrite和requestsRead，每次执行提交(刷盘)前都会进行容器交换

\* 好处：读写请求容器分离，避免潜在的锁竞争

```
private void swapRequests() {

List<GroupCommitRequest> tmp = this.requestsWrite;

this.requestsWrite = this.requestsRead;

this.requestsRead = tmp;

}
```





**注2:**

\* flushedPosition 标记已经刷盘内存的位点。即刷盘相对偏移量，刷盘到什么位置了，下次从此处刷盘即可

\* flushedWhere 标记已经刷盘的物理偏移量，根据此位置可精确查找到文件中消息的存储位置

flushedWhere = 当前刷盘文件名称（该日志文件的起始物理偏移量） + flushedPosition



**注3**

\* 流程图中标记红色部分，将刷盘结果通知给等待线程

```
小结：同步刷盘线程类GroupCommitService主要工作
1.将请求从读容器中取出并通过mappedByteBuffer.force()将数据落盘。
```



**异步刷盘线程类FlushRealTimeService工作流程**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219135125.png)

```
小结：FlushRealTimeService主要工作
1.不开启堆外外内存刷盘方式为mappedByteBuffer.force()
2.开启堆外内存刷盘方式为fileChannel.force
```

<u>**疑问：同步刷盘线程类GroupCommitService每执行一次都会交换读写容器，那刷盘请求什么时候放到写容器（requestsWrite）呢？**</u>



# 消息追加与线程类的交互

分析完线程类后，把镜头切换到消息追加，看看消息进来后是如何跟线程类交互的？

**调用链**

```
@1 CommitLog#putMessage

//同步刷盘或者异步刷盘

handleDiskFlush(result, putMessageResult, msg);

@2 CommitLog#handleDiskFlush
```



**同步刷盘主要代码**

同步刷盘时构造刷盘请求，将请求提交给线程类GroupCommitService，service.putRequest(request)，并获取刷盘结果。

```
if (FlushDiskType.SYNC_FLUSH == this.defaultMessageStore.getMessageStoreConfig().getFlushDiskType()) {

final GroupCommitService service = (GroupCommitService) this.flushCommitLogService;

GroupCommitRequest request = new GroupCommitRequest(result.getWroteOffset() + result.getWroteBytes());

service.putRequest(request);

//等待MappedFile刷盘成功状态通过countDownLatch来控制

boolean flushOK = request.waitForFlush(this.defaultMessageStore.getMessageStoreConfig().getSyncFlushTimeout());

}

}
```



**异步刷盘主要代码**

未开启堆外内存唤醒FlushRealTimeServicee，开启堆外内存唤醒CommitRealTimeService。

```
if (!this.defaultMessageStore.getMessageStoreConfig()

.isTransientStorePoolEnable()) {

flushCommitLogService.wakeup();

} else {

commitLogService.wakeup();

}
```



<!--more-->



# 刷盘方式示意图

**同步刷盘示意图**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219135252.png)



**异步刷盘未开启堆外缓存示意图**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219135312.png)





**异步刷盘开启堆外缓存示意图**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219135332.png)



# 文章总结

1.同异步刷盘通过Broker属性flushDiskType来设置，默认为ASYNC_FLUSH，同步刷盘配置为SYNC_FLUSH

2.同步刷盘是怎么工作的？

注：见GroupCommitService工作流程及与消息追加交互

3.异步刷盘是怎么工作的？

注：见FlushRealTimeService和CommitRealTimeService工作流程及与消息追加交互

4.上篇文章的疑问，写入堆外内存的消息如何落盘的？

注：见异步刷盘开启堆外缓存示意图



# 主要源码类清单

\* CommitLog.java

\* CommitLog#putMessage

\* CommitLog#GroupCommitService

\* CommitLog#FlushRealTimeService

\* CommitLog#CommitRealTimeService