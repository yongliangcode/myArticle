---
title: MQ40# RocketMQ一次消费性能问题排查
categories: RocketMQ
tags: RocketMQ
date: 2020-12-20 23:41:01
---



# 需求描述

在容器推广中，为了测试容器的性能，需要消息SDK与ECS上在发送和消费的性能对比；在对比消费性能时，发现容器中的消费性能居然是ECS的2倍。容器并发消费的20个线程TPS在3万左右，ECS中20个消费线程TPS在1.5万左右。

问题：配置均采用8C16G，容器中的性能几乎是ECS的两倍，这不科学，事出反常必有妖。



# 问题分析

## tcpdump网络情况

tcpdump显示在消费的机器存在频繁的域名解析过程；10.x.x.185向DNS服务器100.x.x.136.domain和10.x.x.138.domain请求解析。而10.x.x.185这台机器又是消息发送者的机器IP，测试的发送和消费分别部署在两台机器上。

问题：消费时为何会有消息发送方的IP呢？而且该IP还不断进行域名解析。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219150223.png)



<!--more-->



## 查看消费线程堆栈

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219150247.png)



# 消费代码定位

在消费时有通过MessageExt.bornHost.getBornHostNameString获取消费这信息；问题由此引起。

```
public class MessageExt extends Message {

private static final long serialVersionUID = 5720810158625748049L;

private int queueId;

private int storeSize;

private long queueOffset;

private int sysFlag;

private long bornTimestamp;

private SocketAddress bornHost;

private long storeTimestamp;

private SocketAddress storeHost;

private String msgId;

private long commitLogOffset;

private int bodyCRC;

private int reconsumeTimes;

private long preparedTransactionOffset;

}
```



调用GetBornHostNameString获取HostName时会根据IP反查DNS服务器；

```
InetSocketAddress inetSocketAddress = (InetSocketAddress)this.bornHost;

return inetSocketAddress.getAddress().getHostName();

```



# 后记

将getBornHostNameString注释或者直接返回IP，ECS的消费性能基本稳定在3万左右。