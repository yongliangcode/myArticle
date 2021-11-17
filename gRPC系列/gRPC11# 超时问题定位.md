---
title: gRPC11# 超时问题定位
categories: gRPC
tags: gRPC
date: 2021-11-14 11:55:01
---





客户端AppTaxiNormalService调用服务Appxxx发生超时，长达50秒。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20211104110315.png)





客户端等待中取消请求，发生调用时间为：2021-11-02 22:11:59.148



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20211104110454.png)





该服务基本上在同一时间段发起向下游的服务均发生超时。



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20211104113927.png)





队列显示瞬间增加很多任务



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20211104114455.png)





磁盘IO和CPU都有上升



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20211104145733.png)





线程dump情况，通信线程调用到了SynchronizationContext，底层的work通信线程怎么调用到了获取节点的业务方法去了。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20211104161804.png)



代码中有使用SynchronizationContext

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20211104163317.png)



SynchronizationContext使用的queue是ConcurrentLinkedQueue队列，被单线程串行执行。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20211104172746.png)



再回到上面的线程栈，业务节点发现事件和gRPC底层通信共用了SynchronizationContext造成阻塞，和线程错乱执行。
