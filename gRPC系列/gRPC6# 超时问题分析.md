---
title: gRPC5# Reactor线程模型
categories: gRPC
tags: gRPC
date: 2020-12-08 11:55:01
---



链路日志显示服务AppHelloPlatformCommonService调用服务AppHelloSuperMallService的方法selectGoodsInGroupForMarketPage发生了5秒超时。链路ID=39278a03c505689c330f5911c54fdd9d

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20211104104903.png)





客户端AppHelloPlatformCommonService日志错误超时。



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20211104105127.png)





服务端AppHelloSuperMallService日志显示有收到请求39278a03c505689c330f5911c54fdd9d的日志。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20211104105350.png)



