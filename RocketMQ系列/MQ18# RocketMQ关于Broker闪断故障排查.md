---
title: MQ18# RocketMQ关于Broker闪断故障排查
categories: RocketMQ
tags: RocketMQ
date: 2020-12-20 20:17:01
---



# 问题描述

在2020-03-16 18:00左右收到告警，业务出现发送RocketMQ失败，在约1分钟左右后自动恢复。RocketMQ运行向来稳定，为何也抖动了？

# Broker日志分析

## 查看GC日志

通过查看发时间问题时间附近GC日志并无发现异常。

```
2020-03-16T17:49:13.785+0800: 13484510.599: Total time for which application threads were stopped: 0.0072354 seconds, Stopping threads took: 0.0001536 seconds

2020-03-16T18:01:23.149+0800: 13485239.963: [GC pause (G1 Evacuation Pause) (young) 13485239.965: [G1Ergonomics (CSet Construction) start choosing CSet, _pending_cards: 7738, predicted base time: 5.74 ms, remaining time: 194.26 ms, target pause time: 200.00 ms]

13485239.965: [G1Ergonomics (CSet Construction) add young regions to CSet, eden: 255 regions, survivors: 1 regions, predicted young region time: 0.52 ms]

13485239.965: [G1Ergonomics (CSet Construction) finish choosing CSet, eden: 255 regions, survivors: 1 regions, old: 0 regions, predicted pause time: 6.26 ms, target pause time: 200.00 ms]

, 0.0090963 secs]

[Parallel Time: 2.3 ms, GC Workers: 23]

[GC Worker Start (ms): Min: 13485239965.1, Avg: 13485239965.4, Max: 13485239965.7, Diff: 0.6]

[Ext Root Scanning (ms): Min: 0.0, Avg: 0.3, Max: 0.6, Diff: 0.6, Sum: 8.0]

[Update RS (ms): Min: 0.1, Avg: 0.3, Max: 0.6, Diff: 0.5, Sum: 7.8]

[Processed Buffers: Min: 2, Avg: 5.7, Max: 11, Diff: 9, Sum: 131]

[Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.1, Diff: 0.1, Sum: 0.8]

[Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.3]

[Object Copy (ms): Min: 0.2, Avg: 0.5, Max: 0.7, Diff: 0.4, Sum: 11.7]

[Termination (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.3]

[Termination Attempts: Min: 1, Avg: 1.0, Max: 1, Diff: 0, Sum: 23]

[GC Worker Other (ms): Min: 0.0, Avg: 0.2, Max: 0.3, Diff: 0.3, Sum: 3.6]

[GC Worker Total (ms): Min: 1.0, Avg: 1.4, Max: 1.9, Diff: 0.8, Sum: 32.6]

[GC Worker End (ms): Min: 13485239966.7, Avg: 13485239966.9, Max: 13485239967.0, Diff: 0.3]

[Code Root Fixup: 0.0 ms]

[Code Root Purge: 0.0 ms]

[Clear CT: 0.9 ms]

[Other: 5.9 ms]

[Choose CSet: 0.0 ms]

[Ref Proc: 1.9 ms]

[Ref Enq: 0.0 ms]

[Redirty Cards: 1.0 ms]

[Humongous Register: 0.0 ms]

[Humongous Reclaim: 0.0 ms]

[Free CSet: 0.2 ms]

[Eden: 4080.0M(4080.0M)->0.0B(4080.0M) Survivors: 16.0M->16.0M Heap: 4176.5M(8192.0M)->96.5M(8192.0M)]

[Times: user=0.05 sys=0.00, real=0.01 secs]
```



## 查看Broker日志

由日志可以看出：slave与问题节点broker同步信息异常。

```
2020-03-16 17:55:15 ERROR BrokerControllerScheduledThread1 - SyncTopicConfig Exception, x.x.x.x:10911

org.apache.rocketmq.remoting.exception.RemotingTimeoutException: wait response on the channel <x.x.x.x:10909> timeout, 3000(ms)

at org.apache.rocketmq.remoting.netty.NettyRemotingAbstract.invokeSyncImpl(NettyRemotingAbstract.java:427) ~[rocketmq-remoting-4.5.2.jar:4.5.2]

at org.apache.rocketmq.remoting.netty.NettyRemotingClient.invokeSync(NettyRemotingClient.java:375) ~[rocketmq-remoting-4.5.2.jar:4.5.2]
```

```
备注：通过查看RocketMQ的集群和GC日志，只能说明但是网络不可用，造成主从同步问题；并未发现Broker自身出问题了。
```



<!--more-->



## 系统监控分析

## 网络监控

网络ping在问题时间段发送中断，网络没有流量。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219074012.png)



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219074030.png)



## 磁盘IO监控

在问题时间段磁盘IO有陡增

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219074123.png)



## CPU监控

CPU在问题时间段有飙高后回落

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219074154.png)



## 内存信息

内存信息没有采集到，由于当时网络中断了。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219074213.png)



## 集群流量

除了网络终端未采集到数据外，问题时间断之前只有6000左右的TPS；网络恢复后有个脉冲到11301，然而总体负载很低。

```
2020-03-16T17:54:05.037+08:00 bike_mq totalTps 6576

2020-03-16T17:54:15.037+08:00 bike_mq totalTps 6375

2020-03-16T17:54:25.037+08:00 bike_mq totalTps 6334

2020-03-16T17:54:35.037+08:00 bike_mq totalTps 6048

2020-03-16T17:54:45.037+08:00 bike_mq totalTps 6162

2020-03-16T17:54:55.037+08:00 bike_mq totalTps 6123

2020-03-16T17:56:50.208+08:00 bike_mq totalTps 5128

2020-03-16T17:57:19.177+08:00 bike_mq totalTps 6240

2020-03-16T17:57:24.44+08:00 bike_mq totalTps 11301

2020-03-16T17:57:30.072+08:00 bike_mq totalTps 9577

2020-03-16T17:57:31.334+08:00 bike_mq totalTps 9485

2020-03-16T17:57:32.564+08:00 bike_mq totalTps 9190

2020-03-16T17:57:33.79+08:00 bike_mq totalTps 7742

2020-03-16T17:57:35.043+08:00 bike_mq totalTps 7323

2020-03-16T17:57:36.269+08:00 bike_mq totalTps 7058

2020-03-16T17:57:37.502+08:00 bike_mq totalTps 6697
```



```
备注：通过监控看到在问题时间段网络、CPU、磁盘IO都出现了问题；到底是磁盘IO引起CPU飙高的？进而影响网络的；还是CPU先飙高引起网络中断和磁盘IO的。机器上只有一个RocketMQ进程，而且当时流量负载并不高；所以由应用进程导致CPU、网络、磁盘IO等问题是解释不通的。那会不会阿里云抖动呢？但是如果阿里云抖动为何只影响RocketMQ集群的3个节点呢？其他RocketMQ集群没有问题；业务机器也没有发现网络等问题。
```



# Linux系统日志分析

通过查看问题时间段日志，发现页分配失败“page allocation failure. order:0, mode:0x20”，也就Page不够了。

```
2020-03-16T17:56:07.505715+08:00 VECS0xxxx kernel: <IRQ> [<ffffffff81143c31>] ? __alloc_pages_nodemask+0x7e1/0x960

2020-03-16T17:56:07.505717+08:00 VECS0xxxx kernel: java: page allocation failure. order:0, mode:0x20

2020-03-16T17:56:07.505719+08:00 VECS0xxxx kernel: Pid: 12845, comm: java Not tainted 2.6.32-754.17.1.el6.x86_64 #1

2020-03-16T17:56:07.505721+08:00 VECS0xxxx kernel: Call Trace:

2020-03-16T17:56:07.505724+08:00 VECS0xxxx kernel: <IRQ> [<ffffffff81143c31>] ? __alloc_pages_nodemask+0x7e1/0x960

2020-03-16T17:56:07.505726+08:00 VECS0xxxx kernel: [<ffffffff8148e700>] ? dev_queue_xmit+0xd0/0x360

2020-03-16T17:56:07.505729+08:00 VECS0xxxx kernel: [<ffffffff814cb3e2>] ? ip_finish_output+0x192/0x380

2020-03-16T17:56:07.505732+08:00 VECS0xxxx kernel: [<ffffffff811862e2>] ?
```



# 解决方案

## 调整内核参数

系统版本：CentOS 6.10 内核版本：Linux version 2.6.32-754.17.1.el6.x86_64

在sysctl.conf修改参数vm.zone_reclaim_mode和vm.min_free_kbytes。

```
vim /etc/sysctl.conf

vm.zone_reclaim_mode = 1

vm.min_free_kbytes = 512000

sysctl -p /etc/sysctl.conf
```

```
备注：修改以上系统参数后，通过两天的观察，内核没有再抛“page allocation failure. order:0, mode:0x20”。
```

参考文章

https://access.redhat.com/solutions/90883

https://billtian.github.io/digoal.blog/2017/10/24/03.html



## 参数含义说明

**min_free_kbytes**

```
This is used to force the Linux VM to keep a minimum number

of kilobytes free. The VM uses this number to compute a

watermark[WMARK_MIN] value for each lowmem zone in the system.

Each lowmem zone gets a number of reserved free pages based

proportionally on its size.

Some minimal amount of memory is needed to satisfy PF_MEMALLOC

allocations; if you set this to lower than 1024KB, your system will

become subtly broken, and prone to deadlock under high loads.

Setting this too high will OOM your machine instantly.
```



**zone_reclaim_mode**

```
Zone_reclaim_mode allows someone to set more or less aggressive approaches to

reclaim memory when a zone runs out of memory. If it is set to zero then no

zone reclaim occurs. Allocations will be satisfied from other zones / nodes

in the system.

This is value ORed together of

1 = Zone reclaim on

2 = Zone reclaim writes dirty pages out

4 = Zone reclaim swaps pages

zone_reclaim_mode is disabled by default. For file servers or workloads

that benefit from having their data cached, zone_reclaim_mode should be

left disabled as the caching effect is likely to be more important than

data locality.

zone_reclaim may be enabled if it's known that the workload is partitioned

such that each partition fits within a NUMA node and that accessing remote

memory would cause a measurable performance reduction. The page allocator

will then reclaim easily reusable pages (those page cache pages that are

currently not used) before allocating off node pages.

Allowing zone reclaim to write out pages stops processes that are

writing large amounts of data from dirtying pages on other nodes. Zone

reclaim will write out dirty pages if a zone fills up and so effectively

throttle the process. This may decrease the performance of a single process

since it cannot use all of system memory to buffer the outgoing writes

anymore but it preserve the memory on other nodes so that the performance

of other processes running on other nodes will not be affected.

Allowing regular swap effectively restricts allocations to the local

node unless explicitly overridden by memory policies or cpuset

configurations.
```



```
备注：zone_reclaim_mode默认为0即不启用zone_reclaim模式，1为打开zone_reclaim模式从本地节点回收内存；min_free_kbytesy允许内核使用的最小内存。
```

# 原理分析



在系统空闲内存低于 watermark[low]时，开始启动内核线程kswapd进行内存回收，直到该zone的空闲内存数量达到watermark[high]后停止回收。如果上层申请内存的速度太快，导致空闲内存降至watermark[min]后，内核就会进行direct reclaim（直接回收），即直接在应用程序的进程上下文中进行回收，再用回收上来的空闲页满足内存申请，因此实际会阻塞应用程序，带来一定的响应延迟，而且可能会触发系统OOM。这是因为watermark[min]以下的内存属于系统的自留内存，用以满足特殊使用，所以不会给用户态的普通申请来用。



**min_free_kbytes大小的影响**

```
min_free_kbytes设的越大，watermark的线越高，同时三个线之间的buffer量也相应会增加。这意味着会较早的启动kswapd进行回收，且会回收上来较多的内存（直至watermark[high]才会停止），这会使得系统预留过多的空闲内存，从而在一定程度上降低了应用程序可使用的内存量。极端情况下设置min_free_kbytes接近内存大小时，留给应用程序的内存就会太少而可能会频繁地导致OOM的发生。

min_free_kbytes设的过小，则会导致系统预留内存过小。kswapd回收的过程中也会有少量的内存分配行为（会设上PF_MEMALLOC）标志，这个标志会允许kswapd使用预留内存；另外一种情况是被OOM选中杀死的进程在退出过程中，如果需要申请内存也可以使用预留部分。这两种情况下让他们使用预留内存可以避免系统进入deadlock状态。
```



**三个watermark的计算方法**

```
watermark[min] = min_free_kbytes换算为page单位即可，假设为min_free_pages。（因为是每个zone各有一套watermark参数，实际计算效果是根据各个zone大小所占内存总大小的比例，而算出来的per zone min_free_pages） watermark[low] = watermark[min] * 5 / 4 watermark[high] = watermark[min] * 3 / 2
```

```
备注：摘自“内存管理参数min_free_kbytes分析”，链接：https://www.dazhuanlan.com
```