---
title: Netty3# Netty组件之Channel注册
categories: Netty
tags: Netty
date: 2020-12-16 11:55:01
---



# 前言



本文将分析EventLoopGroup初始化、EventLoop的选择策略以及Channel是如何通过EventLoop注册到Selector上的。



<!--more-->



# 一、EventLoopGroup类图概览



在客户端示例代码中的中实例化了NioEventLoopGroup，接下来分析下该实例化过程。

```java
EventLoopGroup workerGroup = new NioEventLoopGroup();

Bootstrap b = new Bootstrap();

b.group(workerGroup);
```



从以下类图结构io.netty.util.concurrent.AbstractEventExecutorGroup分支主要负责多线程任务的处理；io.netty.channel.EventLoopGroup分支主要负责Channel相关的注册。MultithreadEventExecutorGroup与MultithreadEventLoopGroup分别继承和实现了上面AbstractEventExecutorGroup和EventLoopGroup，将其负责的功能进行融合。



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202180638.png)



# 二、构造函数解读

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202180703.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202180713.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202180724.png)

**构造函数**

nThreads：eventLoopThreads线程数量，默认值0时取CPU核数的2倍，可以通过参数io.netty.eventLoopThreads指定

Executor：默认ThreadPerTaskExecutor

SelectorProvider默认SelectorProvider.provider()，用于开启Selector和Channel

SelectStrategyFactory：SelectStrategy工厂类，默认DefaultSelectStrategyFactory

EventExecutorChooserFactory：EventExecutor选择器，默认为DefaultEventExecutorChooserFactory



# 三、初始化EventExecutor数组



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202180746.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202180802.png)



**代码解读**

EventExecutor[] children：数组大小为nThreads，默认为CPU核数乘以2。

EventExecutor继承了EventExecutorGroup本质上为线程框架类Executor

children[i]：数据元素为EventLoop，本示例中为NioEventLoop。

**NioEventLoop类图**

NioEventLoop继承了SingleThreadEventLoop，SingleThreadEventLoop同时继承和实现了EventExecutor和EventLoop。即：NioEventLoop拥有了线程类框架处理多线程任务的能力和处理Channel能力。



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202180841.png)



```
备注：本文中EventExecutor数组children的元素为NioEventLoop，NioEventLoop同时拥有线程框架能力和Channel注册等处理能力。
```



#  四、EventExecutor选择器



第三部分对EventExecutor[] children进行初始化分析，然在使用时如何选择其中一个元素呢？

在初始化过程中有以下一行代码，用于初始化EventExecutorChooser。

```java
chooser = chooserFactory.newChooser(children);
```



**EventExecutorChooser类图结构**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202180923.png)

**选择策略**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202180943.png)

@1 如果数组长度是2的幂次方，选择PowerOfTwoEventExecutorChooser，在选取EventExecutor时使用executors[idx.getAndIncrement() & executors.length - 1]

@2 如果数组长度不是2的幂次方，选择GenericEventExecutorChooser，executors[Math.abs(idx.getAndIncrement() % executors.length)]。



# 五、Channel注册

**Channel注册入口**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202181021.png)

**选择EventLoop**

本文为NioEventLoop

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202181034.png)

**绑定Channel到EventExecutor**

通过DefaultChannelPromise绑定Channel到EventExecutor（NioEventLoop）。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202181106.png)

**将Channel注册到Selector**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202181120.png)