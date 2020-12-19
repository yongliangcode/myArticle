---
title: MQ20# RocketMQ集群实现平滑扩缩容
categories: RocketMQ
tags: RocketMQ
date: 2020-12-20 20:18:01
---



# 运维需求

在RocketMQ集群的实践中，对集群扩容、缩容、节点下线等运维做到平滑、业务无感知、数据无丢失，这个对于集群运维的同学来说非常重要。

比如前些日子出现的问题，由于线上集群频繁出现CPU毛刺甚至直接挂掉并伴随着集群抖动，对内核参数的调整只能减缓毛刺却不能消除抖动。集群抖动业务使用会伴随着发送延迟告警，始终是个必须处理的隐患。最终决定更换操作系统即更换内核。

```
集群信息: RocketMQ版本4.5.2

主从信息：4主4从 broker-a, broker-b, broker-c, broker-d,

broker-a(slave), broker-b(slave), broker-c(slave), broker-d(slave)
```

操作系统: centos6

```
Linux version 2.6.32-754.18.2.el6.x86_64 (mockbuild@x86-01.bsys.centos.org) (gcc version 4.4.7 20120313 (Red Hat 4.4.7-23) (GCC) ) #1 SMP Wed Aug 14 16:26:59 UTC 2019
```

目标系统：centos7

```
Linux version 3.10.0-1062.4.1.el7.x86_64 (mockbuild@kbuilder.bsys.centos.org) (gcc version 4.8.5 20150623 (Red Hat 4.8.5-39) (GCC) ) #1 SMP Fri Oct 18 17:15:30 UTC 2019
```



集群的部署如下下图所示：

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219080526.png)

```
其需要简单概括如下，就是要将上述集群的操作系统内核从 Centos6 升级到 Centos7,但业务不能停。
```



# 平滑扩容

## 下线从节点

从节点正常关闭rocketmq，从节点下线对业务不会造成影响，如果配置，slaveReadEnable=true，从节点的通常在消息回溯延迟超过内存消息的40%时使用。

```
sh bin/mqshutdown broker

or

kill pid
```



从节点下线之后，集群的部署情况如下图所示：

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219080558.png)

直接停掉从节点，并不影响当前业务的使用，因为写，读都可以通过主节点，对业务无影响。

## 重组主从模式

新申请4台机器与原从节点重新组合成主从关系。

具体操作为将新申请的4台机器的内核全部升级为Centos7内核，并部署为 broker-a1,broker-b1,broker-c1,broker-d1，这里如果使用 broker-e,f,f,h 命名，会造成流量切斜，导致消费不均衡，原因在文末会给出，大家不妨思考一下。

然后将上一步下掉的从节点，其内核全部升级为 centos7的内核。并将这些节点设置为新增4个主节点的从，其部署结构如下图所示：

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219080619.png)

此时集群中变成8主，其中新增集群有从节点，即装有centor7内核的新机器构成了4主4从，接下来就只需要将内核为centos6的主节点的数据消费完成，并下线即可。



<!--more-->



# 平滑缩容

接下来主要是将装有centos6内核的旧机器从集群中移除，具体操作如下。

## 关闭broker写权限

逐台关闭 broker-a, broker-b, broker-c, broker-d 的写入权限，其中4表示只读权限，6表示读写权限。

```
bin/mqadmin updateBrokerConfig -b x.x.x.x:10911 -n x.x.x.x:9876 -k brokerPermission -v 4

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=128m; support was removed in 8.0

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=128m; support was removed in 8.0

update broker config success, x.x.x.x:10911
```



## 验证broker流量情况

```
bin/mqadmin clusterList -n x.x.x.x:9876
```

等待broker出入流量均归零

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219080712.png)



验证broker积压情况

```
bin/mqadmin brokerConsumeStats -b x.x.x.x:10911 -n x.x.x.x:9876
```



观察最后一行Diff Total等于0时表示该节点已经没有积压，即全部消费完毕。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219080746.png)



## 节点下线

当节点流量和积压都为0时，节点可以下线了。如果broker一直有流量或者积压一直存在呢？通常线上集群的存储时间为2～3天；可以在过了存储时间后再安排下线。

```
sh bin/mqshutdown broker

or

kill pid
```



# 问题答疑

上文中有提到过扩容新增节点时命名时不要使用 broker-e, broker-f, broker-g, broker-h；而采用broker-a1, broker-b1, broker-c1, broker-d1。

按照默认平均分配消费算法，如果采用第一种命名，当关闭broker-a, broker-b, broker-c, broker-d的写入权限时，数据会全部集中在broker-e, broker-f, broker-g, broker-h节点，假如线上部署了四台消费机器，会有两台机器分到broker-a, broker-b, broker-c, broker-d的分区，而另外两台机器分到broker-e, broker-f, broker-g, broker-h的分区。

而broker-a, broker-b, broker-c, broker-d节点的写权限被关闭后，会造成其中两台节点无数据，数据全部分配到另外的消费机器上。

因为只是关闭了broker-a,b,c,d的写权限，读权限未关闭，但如果使用broker-a,a1，b,b1这种命名方式，就能平衡其流量，不至于连续出现大部分队列上无数据的情况，使消费者负载趋于均衡。

在不影响业务的情况下，把集群内所有的节点全部重新更新内核就是这么溜，欢迎留言与作者互动，共同交流。