---
title: Netty7# Netty之事件传递
categories: Netty
tags: Netty
date: 2020-12-26 11:55:01
---



# 前言

前面的文章中写了Channel实例化、Channel初始化、Channel注册、异步通知机制、客户端发起连接、事件的轮询和处理机制。Netty作为client/server高效通信框架，事件在ChannelPipeline是如何传递的，本文就聊聊这事。



<!--more-->



# 事件传递过程

ChannelPipeline随着Channel的创建而创建，在 [Netty2# Netty组件之Channel初始化 ](https://mp.weixin.qq.com/s/tvy0j0Mo9H82SK4jztqu0w) 文章中梳理了ChannelPipeline、ChannelHandlerContext、ChannelHandler的关系如下图。

ChannelPipeline大管道维护了一个ChannelHandlerContext链表，头部为HeadContext，尾部为TailContext。事件传播会沿着链表逐级向下传递。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201226112641.png)



### Inbound&Outbound标识

当ChannelHandlerContext创建时，它是Inbound还是Outbound，是哪个方向的就确定了。下面分析是这种身份是如何确定的。

```java
AbstractChannelHandlerContext(DefaultChannelPipeline pipeline, EventExecutor executor,
                              String name, Class<? extends ChannelHandler> handlerClass) {
    this.name = ObjectUtil.checkNotNull(name, "name");
    this.pipeline = pipeline;
    this.executor = executor;
    this.executionMask = mask(handlerClass); // 注解@1
    ordered = executor == null || executor instanceof OrderedEventExecutor;
}
```

**注解@1** 父类的构造函数中有一个mask(handlerClass)方法，这个方法确定了ChannelHandlerContext的身份。如下代码ChannelInboundHandler.class.isAssignableFrom(handlerType)即如果是ChannelInboundHandler类型mask |= MASK_ALL_INBOUND；反之如果是ChannelOutboundHandler类型，mask |= MASK_ALL_OUTBOUND；**通过赋予mask不同的值来区分是哪个方向的Handler**。isSkippable中的逻辑判断主要对加注解@Skip的方法不再进行事件回调。

```java
private static int mask0(Class<? extends ChannelHandler> handlerType) {
    int mask = MASK_EXCEPTION_CAUGHT;
    try {
        if (ChannelInboundHandler.class.isAssignableFrom(handlerType)) {
            mask |= MASK_ALL_INBOUND; // 标识为INBOUND
            if (isSkippable(handlerType, "channelRegistered", ChannelHandlerContext.class)){
                mask &= ~MASK_CHANNEL_REGISTERED;
            }
            // ...
        }
        if (ChannelOutboundHandler.class.isAssignableFrom(handlerType)) {
            mask |= MASK_ALL_OUTBOUND; // 标识为OUTBOUND
            if (isSkippable(handlerType, "bind", ChannelHandlerContext.class,
                    SocketAddress.class, ChannelPromise.class)) {
                mask &= ~MASK_BIND;
            }
            // ...
        }
        // ...
    } catch (Exception e) {
       // ...
    }
    return mask;
}
```



### Outbound传递过程

**示例入口** 

```java
ChannelFuture future = b.connect(HOST, PORT).sync(); 
future.channel().writeAndFlush("Hi");
```

以writeAndFlush方法跟踪下Outbound事件在ChannelPipeline的传递过程。

```java
@Override
public final ChannelFuture writeAndFlush(Object msg) {
    return tail.writeAndFlush(msg); // 注解@2
}
```

```javascript
private void write(Object msg, boolean flush, ChannelPromise promise) {
    // ...
  	final AbstractChannelHandlerContext next = findContextOutbound(flush ?
            (MASK_WRITE | MASK_FLUSH) : MASK_WRITE); // 注解@3
    final Object m = pipeline.touch(msg, next);
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        if (flush) {
            next.invokeWriteAndFlush(m, promise);
        } else {
            next.invokeWrite(m, promise);
        }
    } else {
        final WriteTask task = WriteTask.newInstance(next, m, promise, flush);
        if (!safeExecute(executor, task, promise, m, !flush)) {
            //...
            task.cancel();
        }
    }
}
```

```java
private AbstractChannelHandlerContext findContextOutbound(int mask) {
    AbstractChannelHandlerContext ctx = this;
    EventExecutor currentExecutor = executor();
    do {
        ctx = ctx.prev;
    } while (skipContext(ctx, currentExecutor, mask, MASK_ONLY_OUTBOUND)); // 注解@4
    return ctx;
}
```

```java
private static boolean skipContext(
        AbstractChannelHandlerContext ctx, EventExecutor currentExecutor, int mask, int onlyMask) {
    return (ctx.executionMask & (onlyMask | mask)) == 0 ||
            (ctx.executor() == currentExecutor && (ctx.executionMask & mask) == 0); // 注解@5
}
```

**注解@2** 从链表尾部TailContext开始执行。

**注解@3** 我们示例writeAndFlush所以findContextOutbound的mask为(MASK_WRITE | MASK_FLUSH)

**注解@4** 循环链表查找，注意skipContext是判断的跳过逻辑。我们查找Outbound的ChannelHandlerContext，遇到Inbound的都会跳过。

**注解@5** 判断是否跳过，这段逻辑是位操作，不好阅读。下面示例抽取各个入参的值测试下：如果下一个ChannelHandlerContext为inBound，则skipContext返回true，从而在查找outBound的do/while循环中跳过。

```java
int executionMask =  MASK_EXCEPTION_CAUGHT |= MASK_ALL_INBOUND; // inBound 身份标识

int mask = MASK_WRITE | MASK_FLUSH; // writeAndFlush 标识

int onlyMask = MASK_ONLY_OUTBOUND; // outBound操作集合

System.out.println((executionMask & (onlyMask | mask))==0);

输出为：true
```



**小结：** outBound事件在ChannelPipeline中传递时，只会选择身份为outBound的ChannelHandlerContext执行。



### Inbound传递过程

以上一篇文章 [Netty6# Netty之事件轮询与处理](https://mp.weixin.qq.com/s/rcVDX0tNMhRGchjYHddARw) 当有新的客户端的连接时触发unsafe.read()执行。具体为NioMessageUnsafe#read()方法。具体为上文中**注解@17**。

```java
// 入口
pipeline.fireChannelRead(readBuf.get(i));

// 实现
@Override
public final ChannelPipeline fireChannelRead(Object msg) {
  AbstractChannelHandlerContext.invokeChannelRead(head, msg);
  return this;
}

private void invokeChannelRead(Object msg) {
  if (invokeHandler()) {
    try {
      ((ChannelInboundHandler) handler()).channelRead(this, msg);
    } catch (Throwable t) {
      notifyHandlerException(t);
    }
  } else {
    fireChannelRead(msg); // 注解@6
  }
}
```

**注解@6** 下面的方法findContextInbound(MASK_CHANNEL_READ)，只查找Inbound的ChannelHandlerContext。具体逻辑与Outbound传递过程相似，不再重复。

```java
@Override
public ChannelHandlerContext fireChannelRead(final Object msg) {
    invokeChannelRead(findContextInbound(MASK_CHANNEL_READ), msg);
    return this;
}
```



**小结：** inBound事件在ChannelPipeline中传递时，只会选择身份为inBound的ChannelHandlerContext执行。



# ChannelPipeline重要API

ChannelPipeline的默认实现为DefaultChannelPipeline，以下API源码梳理均来自该实现类。

| 接口                                                         | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| ChannelPipeline addFirst(ChannelHandler... handlers)         | 在ChannelPipeline中，将ChannelHandlerContext添加HeadContext的后面。 |
| ChannelPipeline addLast(ChannelHandler... handlers);         | 在ChannelPipeline中，将ChannelHandlerContext添加到TailContext的前面。 |
| ChannelPipeline addBefore(String baseName, String name, ChannelHandler handler); | 在ChannelPipeline中，将ChannelHandlerContext添加到某ChannelHandlerContext的前面。 |
| ChannelPipeline addAfter(String baseName, String name, ChannelHandler handler); | 在ChannelPipeline中，将ChannelHandlerContext添加到指定的ChannelHandlerContext的后面。 |
| ChannelPipeline remove(ChannelHandler handler);              | 在ChannelPipeline中，将ChannelHandlerContext从链表中移除。   |
| ChannelPipeline replace(ChannelHandler oldHandler, String newName, ChannelHandler newHandler); | 在ChannelPipeline中，将新newCtx在链表中替换就得oldCtx。      |



### addFirst

说明：在ChannelPipeline中，将ChannelHandlerContext添加HeadContext的后面。

```java
@Override
public final ChannelPipeline addFirst(EventExecutorGroup group, String name, ChannelHandler handler) {
        final AbstractChannelHandlerContext newCtx;
        synchronized (this) {
            checkMultiplicity(handler); // 注解@7
            name = filterName(name, handler); // 注解@8

            newCtx = newContext(group, name, handler); 

            addFirst0(newCtx); // 注解@9
						
          	// ...
        }
        callHandlerAdded0(newCtx); // 注解@10
        return this;
    }
```

**注解@7：**checkMultiplicity()方法校验ChannelHandler是否重复。如果ChannelHandler中有注解@Sharable标识，则允许同一个ChannelHandler添加到不同的ChannelPipeline中。**未加@Sharable注解的ChannelHandler只允许添加到一个ChannelPipeline**。

```java
private static void checkMultiplicity(ChannelHandler handler) {
    if (handler instanceof ChannelHandlerAdapter) {
        ChannelHandlerAdapter h = (ChannelHandlerAdapter) handler;
        if (!h.isSharable() && h.added) { // 判断@Sharable注解和该Handler是否已经被添加
            throw new ChannelPipelineException(
                    h.getClass().getName() +
                    " is not a @Sharable handler, so can't be added or removed multiple times.");
        }
        h.added = true;
    }
}
```

@Sharable使用示例

```java
@Sharable
public class StringEncoder extends MessageToMessageEncoder<CharSequence> {
}
```

**注解@8：** ChannelHandler名字判断，没有设置Handler名字则自动生成一个；设置了Handler名字，不能与该ChannelPipeline其他ChannelHandler名字重复。

```java
private String filterName(String name, ChannelHandler handler) {
    if (name == null) {
        return generateName(handler); 
    }
    checkDuplicateName(name);
    return name;
}
```

**注解@9：**创建DefaultChannelHandlerContext，并加入将其加入到链表中。

```java
private void addFirst0(AbstractChannelHandlerContext newCtx) {
	AbstractChannelHandlerContext nextCtx = head.next;
	newCtx.prev = head;
	newCtx.next = nextCtx;
	head.next = newCtx;
	nextCtx.prev = newCtx;
}
```

**经过上面链表的顺序调整，addFirst将ChannelHandlerContext添加到了HeadContext的后面。**



**注解@10** ：当ChannelHandler添加ChannelPipeline后，回调该Handler的handlerAdded方法，也是通知机制的常用做法。

### addLast

说明：在ChannelPipeline中，将ChannelHandlerContext添加到TailContext的前面。

```java
@Override
public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
    final AbstractChannelHandlerContext newCtx;
    synchronized (this) {
        checkMultiplicity(handler);

        newCtx = newContext(group, filterName(name, handler), handler);

        addLast0(newCtx); // 注解@11

        // ...
    }
    callHandlerAdded0(newCtx);
    return this;
}
```

**注解@11**  **其他逻辑同addFirst，addLast0将HandlerContext添加到TailContext的前一个位置。**

```java
private void addLast0(AbstractChannelHandlerContext newCtx) {
    AbstractChannelHandlerContext prev = tail.prev;
    newCtx.prev = prev;
    newCtx.next = tail;
    prev.next = newCtx;
    tail.prev = newCtx;
}
```



**addBefore**

说明：在ChannelPipeline中，将ChannelHandlerContext添加到某ChannelHandlerContext的前面。

```java
public final ChannelPipeline addBefore(
        EventExecutorGroup group, String baseName, String name, ChannelHandler handler) {
    final AbstractChannelHandlerContext newCtx;
    final AbstractChannelHandlerContext ctx;
    synchronized (this) {
        checkMultiplicity(handler);
        name = filterName(name, handler);
        ctx = getContextOrDie(baseName); // 注解@12

        newCtx = newContext(group, name, handler); 

        addBefore0(ctx, newCtx); // 注解@13

        // ...
    }
    callHandlerAdded0(newCtx);
    return this;
}
```

**注解@12** 根据传入的baseName在ChannelPipleline查找对应的HandlerContext。

```java
private AbstractChannelHandlerContext getContextOrDie(String name) {
    AbstractChannelHandlerContext ctx = (AbstractChannelHandlerContext) context(name);
    if (ctx == null) {
        throw new NoSuchElementException(name);
    } else {
        return ctx;
    }
}
```

**注解@13** 添加到链表中，添加到baseName的前面。

```java
private static void addBefore0(AbstractChannelHandlerContext ctx, AbstractChannelHandlerContext newCtx) {
    newCtx.prev = ctx.prev;
    newCtx.next = ctx;
    ctx.prev.next = newCtx;
    ctx.prev = newCtx;
}
```



**addAfter**

说明：在ChannelPipeline中，将ChannelHandlerContext添加到指定的ChannelHandlerContext的后面。

```java
public final ChannelPipeline addAfter(
        EventExecutorGroup group, String baseName, String name, ChannelHandler handler) {
    final AbstractChannelHandlerContext newCtx;
    final AbstractChannelHandlerContext ctx;

    synchronized (this) {
        checkMultiplicity(handler);
        name = filterName(name, handler);
        ctx = getContextOrDie(baseName);

        newCtx = newContext(group, name, handler);

        addAfter0(ctx, newCtx); // 注解@14

        
    }
    callHandlerAdded0(newCtx);
    return this;
}
```

**注解@14：**调整链表顺序，调整方式同上。

```java
private static void addAfter0(AbstractChannelHandlerContext ctx, AbstractChannelHandlerContext newCtx) {
    newCtx.prev = ctx;
    newCtx.next = ctx.next;
    ctx.next.prev = newCtx;
    ctx.next = newCtx;
}
```



### remove

说明：在ChannelPipeline中，将ChannelHandlerContext从链表中移除。

```java
private AbstractChannelHandlerContext remove(final AbstractChannelHandlerContext ctx) {
    assert ctx != head && ctx != tail; // 注解@15
    synchronized (this) {
        atomicRemoveFromHandlerList(ctx); // 注解@16
    }
    // ... 
    callHandlerRemoved0(ctx);
    return ctx;
}
```

**注解@15** 被移除的ChannelHandlerContext不能是HeadContext和TailContext。

**注解@16** 通过调整前后ChannelHandlerContext的指针指向实现移除操作。

```java
private synchronized void atomicRemoveFromHandlerList(AbstractChannelHandlerContext ctx) {
    AbstractChannelHandlerContext prev = ctx.prev;
    AbstractChannelHandlerContext next = ctx.next;
    prev.next = next;
    next.prev = prev;
}
```



### replace

说明：在ChannelPipeline中，将新newCtx在链表中替换就得oldCtx。

```JAVA
private ChannelHandler replace(
        final AbstractChannelHandlerContext ctx, String newName, ChannelHandler newHandler) {
    assert ctx != head && ctx != tail; // 注解@17

    final AbstractChannelHandlerContext newCtx;
    synchronized (this) {
        checkMultiplicity(newHandler);
        if (newName == null) {
            newName = generateName(newHandler);
        } else {
            boolean sameName = ctx.name().equals(newName);
            if (!sameName) {
                checkDuplicateName(newName);
            }
        }
        newCtx = newContext(ctx.executor, newName, newHandler);
        replace0(ctx, newCtx); // 注解@18
        // ...
    }
    //...
    return ctx.handler();
}
```

**注解@17** 被替换的ChannelHandlerContext不能是HeadContext和TailContext。

**注解@18** 下面替换逻辑中，首先将oldCtx的前后指针暂存；newCtx前后指针指向刚才的暂存；把暂存的pre的next指向newCtx，暂存的next的prev指向newCtx，此时newCtx已经替换到了链表中；将oldCtx的prev和next都指向了newCtx，目的为了已经进入了oldCtx的数据正确流转无论是inbound还是outbound数据。

```java
private static void replace0(AbstractChannelHandlerContext oldCtx, AbstractChannelHandlerContext newCtx) {
    AbstractChannelHandlerContext prev = oldCtx.prev;
    AbstractChannelHandlerContext next = oldCtx.next;
    newCtx.prev = prev;
    newCtx.next = next;

    // Finish the replacement of oldCtx with newCtx in the linked list.
    // Note that this doesn't mean events will be sent to the new handler immediately
    // because we are currently at the event handler thread and no more than one handler methods can be invoked
    // at the same time (we ensured that in replace().)
    prev.next = newCtx;
    next.prev = newCtx;

    // update the reference to the replacement so forward of buffered content will work correctly
    oldCtx.prev = newCtx;
    oldCtx.next = newCtx;
}
```



**小结：** ChannelPipeline提供了方便的API对链表中的ChannelHandlerContext进行插入、删除、添加、替换操作。

