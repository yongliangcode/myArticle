---
title: MQ22# RocketMQ内存传输及4.7消费线程参数设置
categories: RocketMQ
tags: RocketMQ
date: 2020-12-20 20:19:01
---



# 前言

RocketMQ配置中有一个设置项为transferMsgByHeap，即是否通过堆内存传输数据。在文章“RocketMQ存储--同步刷盘和异步刷盘”中对其进行过梳理。那transferMsgByHeap是开启好呢？还是关闭好！第二个问题是可以设置消费的线程数，由于无界队列所以只需要设置最小线程数consumeThreadMin即可，那在rocket-client4.7版本中还能这么用吗？



# transferMsgByHeap误解

transferMsgByHeap设置为false时，通过堆外内存传输数据，相比堆内存传输减少了数据拷贝、零字节拷贝、效率更高，所以关闭transferMsgByHeap应该成为我们的优先选择，但是实践来看，你或许会改变想法，下面是transferMsgByHeap=false，客户端大量超时错误时的日志截图。

## Broker日志截图

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219081609.png)



## CPU日志截图

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219081632.png)





## 系统日志截图

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219081704.png)



## 源码报错截图

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219082807.png)

```
小结：你看到这里会发现，在关闭transferMsgByHeap时，可能造成堆外内存分配不够，触发系统内存回收和落盘操作。此时CPU会有一个陡坡，具体客户端表现为发送大量超时。解决方式开启transferMsgByHeap即可，让运行更加平稳。
```



<!--more-->



# 消费的最小线程数

我们在使用rocketmq消费时，有两个参数consumeThreadMin和consumeThreadMax。在以往的版本中，我们只需要设置consumeThreadMin即可，例如consumeThreadMin=64。在rocket-client4.7版本中，如果设置consumeThreadMin=64会导致消费失败，下面看下原因。

## 错误提示

```
org.apache.rocketmq.client.exception.MQClientException: consumeThreadMin (64) is larger than consumeThreadMax (20)
```



## 源码原因

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219082853.png)



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219082913.png)

小结：在rocketmq-client新版本中，增加了consumeThreadMax的判断。当consumeThreadMin大于20时需要同时设置consumeThreadMax，所以单独设置consumeThreadMin=64会抛出错误导致消费失败。