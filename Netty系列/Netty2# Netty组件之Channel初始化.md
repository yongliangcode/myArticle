---
title: Netty2# Netty组件之Channel初始化
categories: Netty
tags: Netty
date: 2020-12-15 11:55:01
---



# 前言

继上文分析Channel实例化流程后，本文通过分析Channel的初始化流程。旨在从整体上厘清DefaultChannelPipeline、ChannelHandlerContext、ChannelHandler的逻辑关系。



# 一、DefaultChannelPipeline实例化



DefaultChannelPipeline随着Channel的创建而创建，即只要创建了Channel就会同时创建与其对应的ChannelPipeline。下面代码是Channel实例化时调用，上篇文章文末的代码。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202184535.png)



**ChannelHandlerContext类图结构** 



ChannelHandlerContext直观从命名上看出为ChannelHandler上下文，每次构造DefaultChannelHandlerContext都会传入与之对应的ChannelHandler.

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202184642.png)



**ChannelHandlerContext类图结构**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202184711.png)



**ChannelPipeline类图结构**



从下面类图结构可以看出，ChannelPipeline提供了很多操作链表的方法，addFirst/addLast/addBefore/addLast/remove/replace等，入参为ChannelHandler。ChannelPipeline的各种fire操作均通过HandlerContext进行处理。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202184932.png)



**链表关系图示**



先从下面代码看下运营时的链表结构，截图如下。

**示例代码**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202185015.png)

**内存结构**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/1612263172761.jpg)

画个示意图来说明ChannelPipeline、ChannelHandlerContext、ChannelHandler的关系。

**关系图示**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202185341.png)



# 二、Channel初始化



切换到Channel初始化过程，在客户端引导类Bootstrap调用b.connect()或者服务端引导类ServerBootstrap调用bind()时，会调用到抽象引导类AbstractBootstrap的initAndRegister()。下面红色部分即channel初始化入口。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202185411.png)



**客户端初始化**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202185431.png)

@1 客户端初始化代码中，将ChannelHandler通过DefaultChannelHandlerContext加入ChannelPipeline链表中

@2 setChannelOptions以NioChannelOption为例，客户端最后调用到SocketChannelImpl#setOption(); 可以对以下属性进行设置。

\* StandardSocketOptions.SO_RCVBUF // 接受缓存区大小

\* StandardSocketOptions.SO_SNDBUF // 发送缓存区大小

\* StandardSocketOptions.SO_LINGER // 设置延迟关闭的时间

\* StandardSocketOptions.IP_TOS // 设置数据包优先级

\* StandardSocketOptions.IP_MULTICAST_TTL // 设置多播组数据的TTL值

\* ...

**服务端初始化**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202185459.png)

@1 setChannelOptions同样以NioChannelOption为例，服务端会调用到ServerSocketChannelImpl#setOption()，参数含义见客户端端初始化@1

@2 ChannelInitializer实现了ChannelHandler加入到了ChannelPipeline的链表中，其中的逻辑在另文分析EventLoopGroup时梳理。