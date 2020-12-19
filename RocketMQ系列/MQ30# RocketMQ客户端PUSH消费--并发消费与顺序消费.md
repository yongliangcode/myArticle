---
title: MQ30# RocketMQ客户端PUSH消费--并发消费与顺序消费
categories: RocketMQ
tags: RocketMQ
date: 2020-12-20 20:52:01
---



# 消息拉取与处理

**消息拉取**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219140825.png)

```
小结：PullMessageService处理拉取消息请求。通过组织RequetHeader需要包含从哪里开始拉取（ConsumerGroup、Topic，Queue，queueOffset）等信息，向Broker发起请求，取回消息后对消息进行处理。当该Queue的消息数量超过1000，或者最小与最大偏移量之间的差距超过默认2000也会触发限流，即：延迟50毫秒放入请求队列。也可以通过挂起消费线程来延迟（1秒）消息拉取，从而达到消费限流作用。
```



**消息处理**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219140851.png)

```
小结：PullMessageService处理消息部分流程：将消息提交给了processQueue红黑树缓存；同时将消息提交给consumeMessageService来处理具体的消息内容。
```



<!--more-->



# 并发消费流程



**ConsumeMessageConcurrentlyService职责**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219140921.png)

```
小结：ConsumeMessageService并发消费（ConsumeMessageConcurrentlyService）主要工作交给Listener（客户端传入）进行处理，并对处理结果进行统计和处理；对于失败消息，广播消费会丢弃，集群消费会发回Broker重新消费；清理ProcessQueue并更新缓存（offsetTable）消费进度。
```



# 顺序消费流程

**ConsumeMessageOrderlyService职责**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219140952.png)

```
小结：顺序消费流程跟并发消费最大的区别在于，对要处理的队列加锁，确保同一队列，同一时间，只允许一个消费线程处理。
```

```
疑问：

1.为什么顺序消费时需要对Broker发请求对要处理的队列加锁？

2.对Broker端队列加锁流程是怎么样的？

3.既然加锁了需要解锁吗？

4.会存在Broker加锁过期了客户端还在处理该队列的情况吗？
```



**Broker端队列加锁流程**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219141026.png)

```
小结：顺序消费时对Broker端队列加锁防止该队列在特定时间内（一次默认60秒）被分配给其他clientId处理；Broker端加锁了不需要解锁，一次加锁失效时长为60秒；不存在Broker加锁过期了客户端还在处理该队列的情况，Broker加锁时长为60秒，而客户端加锁时长为30秒，当客户端加锁时长失效时会重新请求Broker加锁并更新时间戳，从而可以持续延长加锁时间。
```



# 交互示意图

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219141054.png)

# 源码清单

```
1.PullMessageService.java

2.ConsumeMessageConcurrentlyService.java

3.ConsumeMessageOrderlyService.java

4.RebalanceLockManager.java

```



