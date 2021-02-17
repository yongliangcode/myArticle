---
title: MQ24# RocketMQ存储--日志文件创建与映射流程
categories: RocketMQ
tags: RocketMQ
date: 2020-12-20 20:40:01
---



# 问题描述

日志目录(可配置)/data/rocketmq/store/commitlog会有20位长度的日志文件。

1.日志文件什么时候创建的？

2.日志文件创建流程是什么？

3.日志文件和内存映射是怎么样的？

```
-rw-rw-r-- 1 baseuser baseuser 1073741824 Jun 27 22:50 00000117290188144640

-rw-rw-r-- 1 baseuser baseuser 1073741824 Jun 27 22:52 00000117291261886464

-rw-rw-r-- 1 baseuser baseuser 1073741824 Jun 27 22:54 00000117292335628288

-rw-rw-r-- 1 baseuser baseuser 1073741824 Jun 27 22:56 00000117293409370112

-rw-rw-r-- 1 baseuser baseuser 1073741824 Jun 27 22:57 00000117294483111936

-rw-rw-r-- 1 baseuser baseuser 1073741824 Jun 27 22:56 00000117295556853760
```



<!--more-->



# 日志映射相关类初始化

在Broker启动时实例化了两个类DefaultMessageStore和AllocateMappedFileService。

AllocateMappedFileService是线程类继承了Runnable接口，该线程类持有DefaultMessageStore的引用（即：可操作管理DefaultMessageStore），并启动该线程类。

调用链

```
//Broker启动时调用

@1 BrokerStartup#main#createBrokerController()

boolean initResult = controller.initialize();

@2 Controller#initialize()

this.messageStore = new DefaultMessageStore(this.messageStoreConfig, this.brokerStatsManager, this.messageArrivingListener, this.brokerConfig);

@3 DefaultMessageStore#DefaultMessageStore()

//将DefaultMessageStore自身引用传给AllocateMappedFileService

this.allocateMappedFileService = new AllocateMappedFileService(this);

this.allocateMappedFileService.start();//启动该线程类

@4 class AllocateMappedFileService extends ServiceThread

public AllocateMappedFileService(DefaultMessageStore messageStore) {

this.messageStore = messageStore;

}
```



**线程类一直运行在干啥**

既然在Broker启动时该线程类AllocateMappedFileService就启动了，那么在做什么呢？run方法为while循环，即：**只要服务不停止并且mmapOperation()返回true则一直运行**。

```
/异步处理，调用mmapOperation完成请求的处理

public void run() {

log.info(this.getServiceName() + " service started");

//while循环，只要服务部停止即调用 mmapOperation方法

while (!this.isStopped() && this.mmapOperation()) {

}

log.info(this.getServiceName() + " service end");

}
```

**mmapOperation方法流程图**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219133504.png)

```
小结：mmapOperation方法主要做了两件事：初始化MappedFile和预热MappedFile。只要服务不停止和线程不被中断，这个过程一直重复运行。
```



# 提交映射文件请求（AllocateRequest）

既然AllocateMappedFileService一直从容器（优先级队列和ConcurrentHashMap）中获取AllocateRequest。AllocateRequest是什么时候产生并放到容器中的呢？

RocketMQ消息存储概览【源码笔记】中写入commitLog流程，获取最新的日志文件。

**调用链**

```
@1 CommitLog#putMessage

//文件已满或者没有映射文件重新创建一个文件

if (null == mappedFile || mappedFile.isFull()) {

mappedFile = this.mappedFileQueue.getLastMappedFile(0);

// Mark: NewFile may be cause noise

}

@2 MappedFileQueue#getLastMappedFile
```



**流程图**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219133550.png)

```
小结：MappedFileQueue#getLastMappedFile会向线程类AllocateMappedFileServic提交两个映射文件创建请求：分别为nextFilePath和nextNextFilePath；如果线程类AllocateMappedFileServic为null，则直接new一个MappedFile，此时只会创建一个文件
```

**下面为AllocateMappedFileServic#putRequestAndReturnMappedFile提交两个映射文件请求流程**

**调用链**

```
@1 MappedFileQueue#getLastMappedFile

if (this.allocateMappedFileService != null) {

mappedFile = this.allocateMappedFileService.putRequestAndReturnMappedFile(nextFilePath,

nextNextFilePath, this.mappedFileSize);

}

@2 AllocateMappedFileService#putRequestAndReturnMappedFile

```



**流程图**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219133629.png)

```
小结：处理提交的映射文件请求指的是实例化两个AllocateRequest并把他们提交到requestTable（ConcurrentHashMap）和requestQueue（PriorityBlockingQueue）中，等待5秒，此段时间线程会从这两个容器中获取请求并创建MappedFile，并将结果返回。
```



<!--more-->



# MappedFile初始化

本段梳理下上文中mmapOperation方法流程图第5步初始化MappedFile的流程

**调用链**

```
@1 AllocateMappedFileService#mmapOperation

mappedFile = new MappedFile(req.getFilePath(), req.getFileSize());

@2 MappedFile#init(fileName, fileSize);
```

**流程图**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219133707.png)

```
小结：MappedFile主要干了两件事：1.创建日志文件。2.并将文件映射到内存中
```



# 6.总结

**1.日志文件什么时候创建的？**

备注：在写入消息时，需要获取最新的日志文件（MappedFile），如果文件不存在或者已经写满，此时需要创建MappedFile。具体在MappedFile#init方法this.file = new File(fileName)进行创建。

**2.日志文件创建流程是什么？**

备注：将两个映射创建请求（nextReq和nextNextReq）提交到requestTable（ConcurrentHashMap）和requestQueue(PriorityBlockingQueue)容器中;由线程不断检查并从容器中取出创建日志文件（MappedFile）。

**3.日志文件和内存映射是怎么样的？**

备注：具体的映射时在MappedFile#init方法中通过this.mappedByteBuffer = this.fileChannel.map(MapMode.READ_WRITE, 0, fileSize);将日志文件映射到内存中的。