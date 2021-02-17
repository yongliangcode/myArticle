---
title: MQ28# RocketMQ存储--主从同步
categories: RocketMQ
tags: RocketMQ
date: 2020-12-20 20:49:01
---



# 问题思考

1.消息存储在Master上了，如何同步到Slave上了呢？

2.同步复制和异步复制流程是怎么样的？



<!--more-->



# Broker启动HA调用链

**HA初始化调用链**

```
@1 BrokerStartup#main

start(createBrokerController(args));

@2 BrokerStartup#createBrokerController

boolean initResult = controller.initialize();

@3 BrokerController#initialize

this.messageStore = new DefaultMessageStore

@4 DefaultMessageStore#DefaultMessageStore()

this.haService = new HAService(this);

this.defaultMessageStore = defaultMessageStore;

this.acceptSocketService =

new AcceptSocketService(defaultMessageStore.getMessageStoreConfig()

.getHaListenPort());

this.groupTransferService = new GroupTransferService();

this.haClient = new HAClient();
```



**启动调用链**

```
@1 BrokerStartup#start

controller.start();

@2 BrokerController#start

this.messageStore.start();

@3 DefaultMessageStore#start

@4 this.haService.start();

this.acceptSocketService.beginAccept();

this.acceptSocketService.start();

this.groupTransferService.start();

this.haClient.start();
```



```
小结：从初始化和启动调用链中可以看到，在Broker启动时，初始化并启动了三个线程类，分别为AcceptSocketService、GroupTransferService、HAClient。
```

**问题：这三个线程类在干啥？**



<!--more-->



# 线程类职责

**AcceptSocketService职责**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219135638.png)

```
小结：AcceptSocketService职责初始化TCP通道，监听新的连接并创建HAConnection。
```

<u>问题：HAConnection在做什么？</u>



**HAConnection职责**

```
//构造方法

public HAConnection(final HAService haService, final SocketChannel socketChannel) throws IOException {

this.haService = haService;

this.socketChannel = socketChannel;

//获取客户端请求地址

this.clientAddr = this.socketChannel.socket().getRemoteSocketAddress().toString();

//将通道调整为非阻塞

this.socketChannel.configureBlocking(false);

//关闭连接前将数据发送完毕

this.socketChannel.socket().setSoLinger(false, -1);

//将Nagle算法关闭，客户端每发送一次数据无论大小，都会将其发送出去

this.socketChannel.socket().setTcpNoDelay(true);

//设置接受缓存区为64K

this.socketChannel.socket().setReceiveBufferSize(1024 * 64);

//设置发包缓存区为64K

this.socketChannel.socket().setSendBufferSize(1024 * 64);

//写数据线程类

this.writeSocketService = new WriteSocketService(this.socketChannel);

//读数据线程类

this.readSocketService = new ReadSocketService(this.socketChannel);

this.haService.getConnectionCount().incrementAndGet();

}

//启动

public void start() {

//启动读数据线程

this.readSocketService.start();

//启动写数据线程

this.writeSocketService.start();

}
```



<u>疑问：HAConnection除了对通道做了一些设置外，启动了两个线程服务类，分别为readSocketService和writeSocketService，他们职责是什么呢？</u>



**writeSocketService职责**

流程图

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219135721.png)

```
小结：writeSocketService主要职责，将数据不断写入socketChannel通道；写入数据的大小为nextTransferFromWhere与最大可读位置getReadPosition之间数据；每次写完传输指针自增this.nextTransferFromWhere += size；每隔5秒发送心跳包到socketChannel通道。
```

**readSocketService职责**

流程图

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219135746.png)

```
小结：readSocketService主要职责解析slave发来的请求位点，并更新push2SlaveMaxOffset为该请求位点；唤醒groupTransferService线程。
```



**GroupTransferService职责**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219135816.png)

```
小结：GroupTransferService职责判断主从同步是否完成，完成后唤醒消息发送线程
```



**HAClient职责**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219135843.png)

<u>小结：HAClient职责Slave封装实现类，负责与Master建立连接通道，并从通道中获取数据存储；</u>

<u>并向Master上报Slave存储的最大物理偏移量。</u>



# 主从同步示意图

**主从同步交互消息格式**

1.1 Slave上报物理偏移量reportOffset量格式

| 格式 | 说明 |

| --- | --- |

| 00000018516677754880 | 长度为8位的20位数字 |

1.2 Master写入Slave的信息由Header与Body构成

| 格式 | 说明 |

| --- | --- |

| 00000018516677754880+size | Header部分由8位物理偏移量+消息体大小构成

| 消息具体内容 | Slave请求的位点与Master可读位置之间的数据

**主从同步示意图**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219135921.png)



# 源代码清单

\* HAService.java

\* HAService#AcceptSocketService

\* HAService#GroupTransferService

\* HAService#HAClient

\* HAConnection.java

\* HAConnection#ReadSocketService

\* HAConnection#WriteSocketService