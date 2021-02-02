---
title: Netty6# Netty之事件轮询与处理
categories: Netty
tags: Netty
date: 2020-12-20 11:55:01
---



## 前言

在前面的文章第三篇《Netty组件之Channel注册》分析了channel是如何注册到Selector上的。第五篇《Netty之客户端连接调用》，分析了建立连接的过程。本文将梳理如下内容：

1.就绪事件如何轮询的？bossGroup和workGroup都轮询什么感兴趣的事件

2.bossGroup的职责是什么？又是如何将客户端新建连接Channel传递到workGroup的？

3.workGroup的职责是什么？如何回调到我们自己加入的childHandler中的？



##  一、准备工作

#### 1.示例代码

本文将以这段示例为入口进行分析，示例中设置了bossGroup、workerGroup以及childHandler为HttpUploadServerInitializer。

```java
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
EventLoopGroup workerGroup = new NioEventLoopGroup();

ServerBootstrap b = new ServerBootstrap();
b.group(bossGroup, workerGroup);
b.channel(NioServerSocketChannel.class);
b.handler(new LoggingHandler(LogLevel.INFO));
b.childHandler(new HttpUploadServerInitializer(sslCtx));

Channel ch = b.bind(PORT).sync().channel();
```



#### 2.原理文章

在阅读本文前，可以先阅读下之前的两篇文章。[系统五种I/O模型](https://mp.weixin.qq.com/s/A4x5xCqHOV4HXdU3R4frnw) 和 [Reactor线程模型](https://mp.weixin.qq.com/s/WZB4ChAsail2dPD8zuCutA)。



## 二、就绪事件轮询

接着第三篇《Netty组件之Channel注册》channel注册到Selector，返回selectionKey。其中包含isReadable、isWritable、isConnectable、isAcceptable等通道的就绪状态。一起看下Netty是如何轮询这些就绪事件的。

#### 1.注册selectionKey集合

将Channel注册到Selector时再往后看下implRegister()方法以及实现AbstractPollSelectorImpl。

```
protected abstract void implRegister(SelectionKeyImpl ski)
```

```java
protected void implRegister(SelectionKeyImpl ski) {
    synchronized (closeLock) {
        if (closed)
            throw new ClosedSelectorException();

        // Check to see if the array is large enough
        if (channelArray.length == totalChannels) {
            // Make a larger array
            int newSize = pollWrapper.totalChannels * 2;
            SelectionKeyImpl temp[] = new SelectionKeyImpl[newSize];
            // Copy over
            for (int i=channelOffset; i<totalChannels; i++)
                temp[i] = channelArray[i];
            channelArray = temp;
            // Grow the NativeObject poll array
            pollWrapper.grow(newSize);
        }
        channelArray[totalChannels] = ski;
        ski.setIndex(totalChannels);
        pollWrapper.addEntry(ski.channel);
        totalChannels++;
        keys.add(ski); // 注解@1
    }
}
```

```java
protected SelectorImpl(SelectorProvider sp) {
    super(sp);
    keys = new HashSet<SelectionKey>();
    selectedKeys = new HashSet<SelectionKey>();
    if (Util.atBugLevel("1.4")) {
        publicKeys = keys;
        publicSelectedKeys = selectedKeys;
    } else {
        publicKeys = Collections.unmodifiableSet(keys); // 注解@2
        publicSelectedKeys = Util.ungrowableSet(selectedKeys); // 注解@3
    }
}
```

注解@1：将注册channel返回的selectionKey放入到了HashSet<SelectionKey> keys中。

注解@2：Set<SelectionKey> publicKeys 也就是selectionKey的集合。



小结：SelectorImpl类中有这么一个结合publicKeys存储了selectionKey，而就绪事件的轮询需要依靠轮询selectionKey。



#### 2.就绪selectionKey集合

当执行select()方法时，以AbstractPollSelectorImpl为例，会执行到updateSelectedKeys()方法。

```java
/**
 * Copy the information in the pollfd structs into the opss
 * of the corresponding Channels. Add the ready keys to the
 * ready queue.
 */
protected int updateSelectedKeys() {
    int numKeysUpdated = 0;
    // Skip zeroth entry; it is for interrupts only
    for (int i=channelOffset; i<totalChannels; i++) {
        // 获取通道就绪操作类型（可读、可写、错误等）
        int rOps = pollWrapper.getReventOps(i);
        if (rOps != 0) {
            SelectionKeyImpl sk = channelArray[i];
            pollWrapper.putReventOps(i, 0);
            if (selectedKeys.contains(sk)) {
                // 将ReventOps就绪的操作类型转换到SelectionKeyImpl
                if (sk.channel.translateAndSetReadyOps(rOps, sk)) {
                    numKeysUpdated++;
                }
            } else {
                sk.channel.translateAndSetReadyOps(rOps, sk);
                if ((sk.nioReadyOps() & sk.nioInterestOps()) != 0) {
                    selectedKeys.add(sk); // 注解@4
                    numKeysUpdated++;
                }
            }
        }
    }
    return numKeysUpdated;
}
```

注解@3：由SelectorImpl方法可以看出publicSelectedKeys即为selectedKeys。

注解@4：将就绪的key放入了selectedKeys集合中。



小结：SelectorImpl中的publicSelectedKeys存放了就绪selectedKey。



#### 3.轮询就绪selectionKey集合

```java
private static void doBind0(
        final ChannelFuture regFuture, final Channel channel,
        final SocketAddress localAddress, final ChannelPromise promise) {
    channel.eventLoop().execute(new Runnable() { // 注解@5
        @Override
        public void run() {
            if (regFuture.isSuccess()) {
                channel.bind(localAddress, promise).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
            } else {
                promise.setFailure(regFuture.cause());
            }
        }
    });
}
```

注解@5： 顺着示例b.bind(PORT)进入到doBind0()方法。channel.eventLoop().execute(...)，此处的eventLoop()即NioEventLoop。也就是后续的轮询事件在该NioEventLoop线程中进行。SingleThreadEventExecutor是NioEventLoop的父类，实际执行到SingleThreadEventExecutor的execute方法。

```java
@Override
public void execute(Runnable task) {
    ObjectUtil.checkNotNull(task, "task");
    execute(task, !(task instanceof LazyRunnable) && wakesUpForTask(task));
}
```

```java
private void execute(Runnable task, boolean immediate) {
    boolean inEventLoop = inEventLoop();
    addTask(task); // 注解@6
    if (!inEventLoop) {
        startThread(); // 注解@7
        // ...
    }

    // ...
}
```

```java
private void doStartThread() {
    // ...
    executor.execute(new Runnable() {
        @Override
        public void run() {
            //...
            SingleThreadEventExecutor.this.run(); // 注解@8
            //...   
    });
}
```

注解@6：将Runnable放入Queue中。

注解@7：此处启动了一个线程默认为ThreadPerTaskExecutor。

注解@8：具体执行逻辑由NioEventLoop#run()来执行。

```java
protected void run() {
    int selectCnt = 0;
    for (;;) {  // 注解@9
     // ... 
      try {
     		  // ...
      		 processSelectedKeys(); // @10
      } finally {
      	 // Ensure we always run tasks.
         ranTasks = runAllTasks(); // @11
      }
      // ...   
    }
}
```

注解@9：一个死循环在不断轮询就绪事件

注解@10：处理就绪事件

注解@11：处理放入Queue中的Runnable任务

```java
private void processSelectedKeys() {
    if (selectedKeys != null) {
        processSelectedKeysOptimized();
    } else {
        processSelectedKeysPlain(selector.selectedKeys()); // 注解@12
    }
}
```

注解@12：selector.selectedKeys()即为SelectorImpl的publicSelectedKeys，即获取了就绪事件集合。

小结：就绪事件的轮询SingleThreadEventExecutor#run方法负责，不断轮询就绪事件集合publicSelectedKeys，来判断是否有就绪事件。



<!--more-->



## 三、事件处理

```java
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
    // ...
    try {
        int readyOps = k.readyOps();
        // We first need to call finishConnect() before try to trigger a read(...) or write(...) as otherwise
        // the NIO JDK channel implementation may throw a NotYetConnectedException.
        if ((readyOps & SelectionKey.OP_CONNECT) != 0) { // 注解@13
            // remove OP_CONNECT as otherwise Selector.select(..) will always return without blocking
            // See https://github.com/netty/netty/issues/924
            int ops = k.interestOps();
            ops &= ~SelectionKey.OP_CONNECT;
            k.interestOps(ops);

            unsafe.finishConnect();
        }

        // Process OP_WRITE first as we may be able to write some queued buffers and so free memory.
        if ((readyOps & SelectionKey.OP_WRITE) != 0) { // 注解@14
          // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
          ch.unsafe().forceFlush();
        }

        // Also check for readOps of 0 to workaround possible JDK bug which may otherwise lead
        // to a spin loop
        if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) { // 注解@15
            unsafe.read();
        }
    } catch (CancelledKeyException ignored) {
        unsafe.close(unsafe.voidPromise());
    }
}
```

注解@13：处理连接事件

注解@14：处理可写事件

注解@15：处理可读事件和处理客户端连接事件



小结：在以上的事件处理中，bossGroup主要处理客户端连接OP_ACCEPT事件。当有新的客户端的连接时触发unsafe.read()执行。具体为NioMessageUnsafe#read()方法。

```
public void read() {
    // ...
    try {
        try {
            do {
                int localRead = doReadMessages(readBuf); // 注解@16
                // ...
                allocHandle.incMessagesRead(localRead);
            } while (allocHandle.continueReading());
        } catch (Throwable t) {
            exception = t;
        }
			  int size = readBuf.size();
        for (int i = 0; i < size; i ++) {
            readPending = false;
            pipeline.fireChannelRead(readBuf.get(i)); // 注解@17
        }
      	// ...
    } finally {
        // ...
    }
}
```

注解@16：见下面doReadMessages逻辑。

注解@17：pipeline.fireChannelRead触发了pipeline链表向下执行。

```java
protected int doReadMessages(List<Object> buf) throws Exception {
    SocketChannel ch = SocketUtils.accept(javaChannel()); // 注解@18
    try {
        if (ch != null) {
            buf.add(new NioSocketChannel(this, ch)); // 注解@19
            return 1;
        }
    } catch (Throwable t) {
        // ...
    }
    return 0;
}
```

```java
protected AbstractNioByteChannel(Channel parent, SelectableChannel ch) {
    super(parent, ch, SelectionKey.OP_READ);
}
```

注解@18：获取新建立的连接通道SocketChannel。

注解@19：将连接通道SocketChannel转换为NioSocketChannel，建立时注册的OP_READ读事件。并将新建立的连接通道放入了List中。

再回到注解@17，pipeline.fireChannelRead触发了链表向下传递。具体传递到那个Handler中需要翻前面初始化代码。

```java
void init(Channel channel) {
    setChannelOptions(channel, newOptionsArray(), logger); 
    setAttributes(channel, attrs0().entrySet().toArray(EMPTY_ATTRIBUTE_ARRAY));

    ChannelPipeline p = channel.pipeline();

    final EventLoopGroup currentChildGroup = childGroup;
    final ChannelHandler currentChildHandler = childHandler;
    final Entry<ChannelOption<?>, Object>[] currentChildOptions;
    synchronized (childOptions) {
        currentChildOptions = childOptions.entrySet().toArray(EMPTY_OPTION_ARRAY);
    }
    final Entry<AttributeKey<?>, Object>[] currentChildAttrs = childAttrs.entrySet().toArray(EMPTY_ATTRIBUTE_ARRAY);

    p.addLast(new ChannelInitializer<Channel>() { 
        @Override
        public void initChannel(final Channel ch) { // 注解@20
            final ChannelPipeline pipeline = ch.pipeline();
            ChannelHandler handler = config.handler();
            if (handler != null) {
                pipeline.addLast(handler);
            }

            ch.eventLoop().execute(new Runnable() {
                @Override
                public void run() {
                    pipeline.addLast(new ServerBootstrapAcceptor( // 注解21
                            ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                }
            });
        }
    });
}
```

注解@20：在Server初始化时回调

注解@21：Server初始化后在pipeline加入了ServerBootstrapAcceptor，同时注意参数currentChildGroup和currentChildHandler，分别对应示例中传入的childGroup和childHandler。



接着看下ServerBootstrapAcceptor的channelRead方法。

```java
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    final Channel child = (Channel) msg; // 注解@22

    child.pipeline().addLast(childHandler); // 注解@23

    setChannelOptions(child, childOptions, logger);
    setAttributes(child, childAttrs);

    try {
        childGroup.register(child).addListener(new ChannelFutureListener() { // 注解@24
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                if (!future.isSuccess()) {
                    forceClose(child, future.cause());
                }
            }
        });
    } catch (Throwable t) {
        forceClose(child, t);
    }
}
```

注解@22：注意此处有“注解@17”步回调回来，传入的msg为新建立连接的NioSocketChannel，再强调下该通道注册了可读事件。

注解@23：把childHandler加入到新建立连接Channel的pipeline，也就是示例中的HttpUploadServerInitializer。HttpUploadServerInitializer的初始化（channel注册后回调）又加入了HttpRequestDecoder、HttpResponseEncoder、HttpContentCompressor、HttpUploadServerHandler到该channel的pipeline。

```java
public void initChannel(SocketChannel ch) {
    ChannelPipeline pipeline = ch.pipeline();

    if (sslCtx != null) {
        pipeline.addLast(sslCtx.newHandler(ch.alloc()));
    }

    pipeline.addLast(new HttpRequestDecoder());
    pipeline.addLast(new HttpResponseEncoder());

    // Remove the following line if you don't want automatic content compression.
    pipeline.addLast(new HttpContentCompressor());

    pipeline.addLast(new HttpUploadServerHandler());
}
```



注解@24：将childGroup绑定到该NioSocketChannel，并将该NioSocketChannel注册到Selector。



小结：1.当轮询到有新的客户端连接建立时，将该新建的通道NioSocketChannel通过pipeline转发给ServerBootstrapAcceptor，这个过程由线程池bossGroup分配的线程负责。

2.当ServerBootstrapAcceptor收到新建立的通道NioSocketChannel时与workGroup分配的线程绑定，并将用户添加的childHandler加入到该channel的pipeline，注册OP_READ读事件到Selector。该过程由线程池workGroup分配的线程负责。

3.当有数据可读时由线程池workGroup分配的线程处理，并一路通过pipeline回调到我们自己加入的ChannelHandler 见注解@23.

例如：示例中加入的HttpUploadServerHandler，父类为SimpleChannelInboundHandler。当监听到有可读时间时，通过pipeline一路回调到SimpleChannelInboundHandler的channelRead()，进而调用到HttpUploadServerHandler的channelRead0()方法，从而处理我们的逻辑。





















