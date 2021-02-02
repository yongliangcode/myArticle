---
title: Netty1# Netty组件之Channel实例化
categories: Netty
tags: Netty
date: 2020-12-11 11:55:01
---



# 一、Channel概述

Channel提供了I/O的基本操作。从以下子接口中可以看出Netty对不同的底层协议提供了对应的channel来处理，例如：TCP/IP、UDP/IP、SCTP/IP、HTTP2等。



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202185721.png)



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202185853.png)



# 二、实例化流程



从客户端引导类示例中查看Channel初始化过程。示例中使用NioSocketChannel作为通信通道，在java中通信中会建立socket连接

```java
EventLoopGroup workerGroup = new NioEventLoopGroup();

Http2ClientInitializer initializer = new Http2ClientInitializer(sslCtx, Integer.MAX_VALUE);

Bootstrap b = new Bootstrap();

b.group(workerGroup);

b.channel(NioSocketChannel.class);

b.option(ChannelOption.SO_KEEPALIVE, true);

b.remoteAddress(HOST, PORT);

b.handler(initializer);

Channel channel = b.connect().syncUninterruptibly().channel();

System.out.println("Connected to [" + HOST + ':' + PORT + ']');
```



Channel通过ChannelFactory创建，下面看一下ChannelFactory类图。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202185957.png)

ReflectiveChannelFactory提供了newChannel()方法通过反射实例化。

示例中通过b.channel(NioSocketChannel.class)将NioSocketChannel.class赋值给ReflectiveChannelFactory的成员变量Constructor<? extends T> constructor，Channel在connect的时候实例化，下面为实例化调用链路。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202190011.png)

# 三、实例化过程



**客户端实例化过程**

了解了Channel初始化调用链，再来看下以NioSocketChannel为例初始化做了哪些事情。下面是NioSocketChannel的四个构造重载方法。

```java
public NioSocketChannel() {

this(DEFAULT_SELECTOR_PROVIDER); // @1

}

public NioSocketChannel(SelectorProvider provider) {

this(newSocket(provider)); // @2

}

public NioSocketChannel(SocketChannel socket) {

this(null, socket);

}

public NioSocketChannel(Channel parent, SocketChannel socket) {

super(parent, socket);

config = new NioSocketChannelConfig(this, socket.socket());

}

protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {

super(parent);

this.ch = ch;

this.readInterestOp = readInterestOp;

ch.configureBlocking(false); // @3

}

}
```



**代码解读**

@1 默认使用SelectorProvider.provider()

@2 使用Provider创建SocketChannel。provider.openSocketChannel()->new SocketChannelImpl(this)。

@3 设置NioChannel非阻塞模式

```
小结：客户端NioSocketChannel实例化过程中已经回到所熟悉的java nio。创建了通道SocketChannel，并设置为非阻塞。
```



**服务端实例化过程**

Channel服务端的实例化流程与客户端是相同的，下面以NioServerSocketChannel为例走查实例化过程。服务端引导初始化示例代码如下。

```java
EventLoopGroup group = new NioEventLoopGroup();

ServerBootstrap b = new ServerBootstrap();

b.option(ChannelOption.SO_BACKLOG, 1024);

b.group(group)

.channel(NioServerSocketChannel.class)

.handler(new LoggingHandler(LogLevel.INFO))

.childHandler(new Http2ServerInitializer(sslCtx));

Channel ch = b.bind(PORT).sync().channel();
```



NioServerSocketChannel的构造方法。

```java
public NioServerSocketChannel() {

this(newSocket(DEFAULT_SELECTOR_PROVIDER)); // @1

}

public NioServerSocketChannel(SelectorProvider provider) {

this(newSocket(provider)); // @2

}

public NioServerSocketChannel(ServerSocketChannel channel) {

super(null, channel, SelectionKey.OP_ACCEPT);

config = new NioServerSocketChannelConfig(this, javaChannel().socket());

}

protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {

super(parent);

this.ch = ch;

this.readInterestOp = readInterestOp;

ch.configureBlocking(false); // @3

}
```



**代码解读**

@1 使用默认Provider类SelectorProvider

@2 开启服务端通道ServerSocketChannel。provider.openServerSocketChannel()->new ServerSocketChannelImpl(this)。

@3 将ServerSocketChanne设置为非阻塞。

```
小结：服务端NioServerSocketChannel的实例化过程同样回到熟悉的Java NIO，创建非阻塞ServerSocketChanne通道。
```



**实例化其他事项** 

在实例化的过程中，会调父类的构造方法super(parent)。

```java
protected AbstractChannel(Channel parent) {

this.parent = parent;

id = newId(); // @1

unsafe = newUnsafe(); // @2

pipeline = newChannelPipeline(); // @3

}
```



**@1 ChannelId初始化**

ChannelId是Channel的唯一标识，下面看下DefaultChannelId的生成规则。

```java
private DefaultChannelId() {

data = new byte[MACHINE_ID.length + PROCESS_ID_LEN + SEQUENCE_LEN + TIMESTAMP_LEN + RANDOM_LEN];

int i = 0;

// machineId

System.arraycopy(MACHINE_ID, 0, data, i, MACHINE_ID.length);

i += MACHINE_ID.length;

// processId

i = writeInt(i, PROCESS_ID);

// sequence

i = writeInt(i, nextSequence.getAndIncrement());

// timestamp (kind of)

i = writeLong(i, Long.reverse(System.nanoTime()) ^ System.currentTimeMillis());

// random

int random = PlatformDependent.threadLocalRandom().nextInt();

i = writeInt(i, random);

assert i == data.length;

hashCode = Arrays.hashCode(data);

}
```



```
小结：默认的ChannelId由machineId、processId、sequence、timestamp、random构成。
machineId：可以由参数io.netty.machineId自定义，默认为8位随机byte构成
processId：可以由参数io.netty.processId自定义，默认为4位进程ID
sequence：原子自增序号AtomicInteger，每创建一个Chanenl会进行自增
timestamp：8位的timestamp
random：4位的随机整数
```

**@2 unsafe初始化**

unsafe即I/O的核心操作，byte的读写都靠它来处理。服务端NioServerSocketChannel初始化使用NioMessageUnsafe。客户端NioSocketChannel初始化使用NioSocketChannelUnsafe。以NIO为例看下Unsafe的类图结构。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202190316.png)

**@3 ChannelPipeline初始化**

默认使用DefaultChannelPipeline，从构造方法可以看出为链表结构，详细分析另文分析。

```java
protected DefaultChannelPipeline(Channel channel) {

this.channel = ObjectUtil.checkNotNull(channel, "channel");

succeededFuture = new SucceededChannelFuture(channel, null);

voidPromise = new VoidChannelPromise(channel, true);

tail = new TailContext(this);

head = new HeadContext(this);

head.next = tail;

tail.prev = head;

}
```

