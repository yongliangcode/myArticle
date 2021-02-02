---
title: Netty5# Netty之客户端连接调用
categories: Netty
tags: Netty
date: 2020-12-18 11:55:01
---



# 前言

本文主要梳理Netty客户端如何发起连接请求的以及最终通过SocketChannel与服务端建立连接，顺便分析了在此过程中涉及到的地址解析过程。



# 一、获取地址解析器

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202173713.png)



```
备注：在Netty客户端发起连接前，先获取了AddressResolver，并进行了解析判断。
```



**获取AddressResolver过程**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202173747.png)

```
备注： 创建AddressResolver并将其放到缓存Map中，key为executor。
```

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202173813.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202173826.png)

```
备注：默认使用DefaultAddressResolverGroup中的DefaultNameResolver构建InetSocketAddressResolver。
小结：从上面获取地址解析过程中，AddressResolverGroup拥有一组AddressResolver存储于Map中，key为EventExecutor，而AddressResolver是通过NameResolver构建的。
```



<!--more-->



#  二、地址解析器图谱



**AddressResolverGroup类图**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202173916.png)



**AddressResolver类图**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202173950.png)



**NameResolver类图**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202174015.png)



**关系图示**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202174037.png)



# 三、地址解析过程



地址解析通过下面的方法来实现。分别看下isSupported、isResolved、doResolve的逻辑。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202174104.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202174118.png)

@1 isSupported主要判断传入socket地址是否属于InetSocketAddress，通过JDK中isInstance来实现。

@2 doIsResolved判断包含了isSupported和非空判断，入参非空并且属于InetSocketAddress则标记解析成功

@3 doResolve 根据host name解析成InetSocketAddress，通过InetAddress.getByName(hostname)实现。

```
小结：地址解析主要得到SocketAddress是合法有效的，如果为host name默认为通过InetAddress.getByName转换为InetAddress。
```



#  四、建立连接

在地址解析成功后，该建立连接了，接下来看下netty是如何发起的。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202174150.png)



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202174201.png)

```
备注：接着调用AbstractChannel的connect方法，即：DefaultChannelPipeline#connect。
```

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202174244.png)

```
备注：从链表的最后一个tail发起连接。
```

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202174256.png)

```
备注：从之前文章分析中，我们知道链表构成。会调用到AbstractChannelHandlerContext#connect方法。通过方法findContextOutbound查找链表中负责出站的HandlerContext调用其connect方法，结束后向下一个出站HandlerContext传递调用。
```

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202174326.png)

```
备注：出站HandlerContext查找过程。
```

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202174340.png)

```
备注：上图为运行时DefaultChannelPipeline链表中的Handler结构。尾部为TailContext，头部为HeadContext。
```

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202174416.png)

```
备注：继续上面的连接传递，最后会调用HeaderContext的connect方法。通过unsafe.connect向服务端发起连接调用。
```

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202174434.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210202174446.png)

```
备注：调用NioSocketChannel#doConnect方法，最后通过Java NIO的SocketChannel#connect发起连接请求。
```





















