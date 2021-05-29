---
title: gRPC1# HTTP/2协议之连接前言
categories: gRPC
tags: gRPC
date: 2020-12-01 11:55:01
---



# 前言

HTTP/2在传输数据之前，先建立连接，`建立HTTP/2连接的标记为Client发送连接前言Magic`。HTTP/2属于应用层，位于TPC/IP及安全传输层协议TLS之上。在建立HTTP/2连接的过程中，会先后经历TCP握手、TLS握手、HTTP/2连接前言。下图网络分层图示：



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225093219.png)





<!--more-->



**TCP握手回顾**

回顾下TCP的三次握手，三次握手后TPC连接建立，具体步骤如下：
第一步：Client发送[SYN]报文到Server。Client进入SYN_SENT状态，等待Server响应。[SYN]报文序号Seq=x《备注：截图中Seq=0》
第二步：Server收到后发送[SYN,ACK]报文给Client，ACK为x+1(备注：截图中ACK=1); [SYN,ACK]报文序号为y(备注：截图中Seq=0),Server进入SYN_RECV状态
第三步：Client收到后，发送[ACK]报文到Server，包序号Seq=x+1，ACK=y+1。Server收到后Client/Server进入ESTABLISHED状态。



**TPC握手报文**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225093303.png)



**TPC握手交互图**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225093336.png)



# TLS握手简化回顾

TLS传输层安全协议，主要回顾简化的交互过程：

**第一步**

Client向Server发送ClientHello，包括支持的协议版本、Client随机数、支持的加密算法等

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225093415.png)



**第二步** 

Server向Client发送ServerHello，包括确认协议版本、Server随机数、确认加密算法、Server证书

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225093515.png)



**第三步**

Server向Client发送证书，客户端校验证书有效性

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225093556.png)



**第四步**

Client通知Server用协商的密钥进行通信

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225093627.png)

**第五步** 

传输加密数据

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225093654.png)



# 建立HTTP/2连接前言

在TLS之后，Client会向Server发送Magic标记着HTTP/2连接的建立，具体Magic为：PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n 详见下图：



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225093730.png)