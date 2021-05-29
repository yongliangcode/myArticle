---
title: gRPC5# Reactor线程模型
categories: gRPC
tags: gRPC
date: 2020-12-08 11:55:01
---



# 前言

Reactor模型是基于事件驱动的线程模型，可以分为Reactor单线程模型、Reactor多线程模型、主从Reactor多线程模型，通常基于在I/O多路复用实现。不同的角色职责有：Dispatcher负责事件分发、Acceptor负责处理客户端连接、Handler处理非连接事件（例如：读写事件）。



<!--more-->



# Reactor单线程模型



**原理图示**

在Reactor单线程模型中，操作在同一个Reactor线程中完成。根据事件的不同类型，由Dispatcher将事件转发到不同的角色中处理。连接事件转发到Acceptor处理、读写事件转发到不同的Handler处理。



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225101450.png)



**实现图示**

NIO实现中，可以将Accept事件注册到select选择器中，轮询是否有“接受就绪”事件。如果为“连接就绪”分发给Acceptor角色处理；“写就绪”事件分发给负责写的Handler角色处理；“读就绪”事件分发给负责读的Handler角色处理。这是事情都在一个线程中处理。



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225101514.png)



# Reactor多线程模型

**原理图示**

在Reactor多线程模型中。根据事件的不同类型，由Dispatcher将事件转发到不同的角色中处理。连接事件转发到Acceptor单线程处理、读写事件转发到不同的Handler由线程池处理。



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225101546.png)



**实现图示**

NIO实现中，可以将Accept事件注册到select选择器中，轮询是否有“接受就绪”事件。如果为“连接就绪”分发给Acceptor角色处理，此处处理“连接就绪”为一个线程；“写就绪”事件分发给负责写的Handler角色由线程池处理；“读就绪”事件分发给负责读的Handler角色由线程池处理。



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225101604.png)



# **主从Reactor多线程模型**

**原理图示**

Reactor多线程模型，由Acceptor接受客户端连接请求后，创建SocketChannel注册到Main-Reactor线程池中某个线程的Select中；具体处理读写事件还是使用线程池处理（Sub-Reactor线程池）。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225101641.png)



**实现图示**

将Accept事件注册到select选择器中，轮询是否有“接受就绪”事件；“连接就绪”分发给Acceptor角色处理，创建新的SocketChannel转发给Main-Reactor线程池中的某个线程处理；在指定的Main-Reactor某个线程中，将SocketChannel注册读写事件；当“写就绪/读就绪”事件分别由线程池（Sub-Reactor线程池）处理。



![image-20210225101705355](/Users/yongliang/Library/Application Support/typora-user-images/image-20210225101705355.png)









