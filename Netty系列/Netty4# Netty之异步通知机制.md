---
title: Netty4# Netty之异步通知机制
categories: Netty
tags: Netty
date: 2020-12-17 11:55:01
---



# 前言



前面的文章分析了Channel实例化、初始化、注册机制，本文分析下异步结果的通知，也就是回调，同时梳理下Future、Promise、ChannelFuture、ChannelPromise的关系。



<!--more-->



#  一、异步通知代码走查



在Channel注册到Selector后，会返回ChannelFuture。如果注册未完成，会通过增加Listener来进行异步通知注册结果，接下来看下是如何回调的。

**代码块**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202175023.png)

```
备注：上面代码块中在注册完Channel后返回ChannelFuture，在ChannelFuture注册了ChannelFutureListener，通过异步通知的方式获取注册结果。
```

**代码块**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202175039.png)

```
备注：构建DefaultChannelPromise绑定了EventLoop和Channel，上面注册的ChannelFutureListener实际注册到了DefaultChannelPromise。
```

**代码块**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202175059.png)

```
备注：通过ChannelPromise标记Channel注册成功。
```

**代码块**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202175115.png)

```
备注：在DefaultPromise中通过cas设置Channel注册结果，并回调加在其身上的Listener。
```

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202175138.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202175150.png)

```
备注：将注册的所有Listener，通过回调GenericFutureListener的operationComplete方法，完成结果的通知。
```



# 二、异步通知流程图

下面以channel注册为例，勾勒异步回调流程图。Future/Promise作为结果载体与执行Listener的执行主体。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202175347.png)







# 三、Future/Promise关系图谱

以下类图中展现了Future/Promise的类图结构，Netty中Future继承Java中的Future并`增加了基于Listener的异步通知机制`。

Promise允许在标志某个操作结果后再回调Listener（比如：在注册成功后调用Promise#trySuccess将成功结果在Promise中标记，并回调Listener）。

ChannelFuture与特定的Channel绑定，ChannelPromise继承ChannelFuture与Promise即拥有绑定特定Channel与标记操作结果回调Listener的能力。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202175418.png)





