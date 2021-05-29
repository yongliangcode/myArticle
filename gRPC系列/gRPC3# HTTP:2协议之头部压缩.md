---
title: gRPC3# HTTP/2协议之头部压缩
categories: gRPC
tags: gRPC
date: 2020-12-03 11:55:01
---



# 前言

为了报文传输更小、更快，在HTTP/2中Header头是经过压缩的，使用的压缩算法为HPACK。本文先通过Wireshark抓包截图直观感受下头部压缩效果，进而分析下这种压缩算法是如何工作的。 



<!--more-->



# **压缩效果对比**



**压缩前效果**

以Header中的user-agent为例，在压缩前的大小为63个字节。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225095546.png)



**压缩后效果**

Header中的user-agent在压缩后，大小为1个字节。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225095620.png)



小结：Header中user-agent从压缩前的63个字节到压缩后的1个字节，HTTP/2是如何做到的呢？



# HPACK算法

HTTP/2头部通过HPACK算法进行压缩，这种算法通过服务端和客户端个字维护索引表来实现。索引表又分为静态表和动态表。



**伪头字段** 

Header传输以二进制桢的方式进行，为了与HTTP1中Header区分，这些以冒号开头的字段被称为“伪头字段”。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225095715.png)



**静态表**

静态表中定义了61个Header字段与Index，可以通过传输Index进而获取Header的字段与值，极大减少了报文大小。静态表中的字段和值固定，而且是只读的。

静态表部分值

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225095753.png)



详见：https://tools.ietf.org/html/rfc7541#appendix-A



**动态表**

动态表接在静态表之后，结构与静态表相同，可随时更新。下图中索引号62、63即为动态表字段。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210225095828.png)



# 总结

回到本文前的压缩效果对比，客户端通过传输索引号，服务端根据索引号在动态表中获取Header的key与value。user-agent索引号占1个字节。另外，索引表中不存在的使用huffman编码，再更新到动态表中。