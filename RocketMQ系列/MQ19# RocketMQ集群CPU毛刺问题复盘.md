---
title: MQ19# RocketMQ集群CPU毛刺问题复盘
categories: RocketMQ
tags: RocketMQ
date: 2020-12-20 20:17:01
---



# 问题描述

RocketMQ从节点、主节点频繁CPU飙高，很明显的毛刺，很多次从节点直接挂掉了。

详情截图如下：

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219075750.png)



<!--more-->



# 集群情况

```
RocketMQ版本使用4.5.2，4主4从模式

集群tps在8000左右

单节点配置32C/128G/1.7T

其中2从部署在阿里云ECS上，即一个集群6台ECS

内核版本：Linux version 2.6.32-754.18.2.el6.x86_64 (mockbuild@x86-01.bsys.centos.org) (gcc version 4.4.7 20120313 (Red Hat 4.4.7-23) (GCC) ) #1 SMP Wed Aug 14 16:26:59 UTC 2019
```



# 内核参数

```
vm.overcommit_memory=1

vm.drop_caches=1

vm.zone_reclaim_mode=0

vm.max_map_count=655360

vm.dirty_background_ratio=50

vm.dirty_ratio=50

vm.dirty_writeback_centisecs=360000

vm.page-cluster=3

vm.swappiness=1
```

```
备注：搭建时的内核参数主从一致的，内容如上，之前使用该参数配置在RocketMQ集群4.1版本中，未发生任何异常情况。
```

# 从节点JVM参数

```
/usr/java/jdk1.8.0_66/bin/java -server -Xms8g -Xmx8g -Xmn4g -XX:+UseG1GC -XX:G1HeapRegionSize=16m -XX:G1ReservePercent=25 -XX:InitiatingHeapOccupancyPercent=30 -XX:SoftRefLRUPolicyMSPerMB=0 -verbose:gc -Xloggc:/dev/shm/rmq_broker_gc_%p_%t.log -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCApplicationStoppedTime -XX:+PrintAdaptiveSizePolicy -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=30m -XX:-OmitStackTraceInFastThrow -XX:+AlwaysPreTouch -XX:MaxDirectMemorySize=15g -XX:-UseLargePages -XX:-UseBiasedLocking -Djava.ext.dirs=/usr/java/jdk1.8.0_66/jre/lib/ext:/workspace/rocketmq-all-4.5.2-bin-release/bin/../lib -cp .:/workspace/rocketmq-all-4.5.2-bin-release/bin/../conf:.:/usr/java/jdk1.8.0_66/lib/tools.jar:/usr/java/jdk1.8.0_66/lib/dt.jar org.apache.rocketmq.broker.BrokerStartup -c /workspace/rocketmq-all-4.5.2-bin-release/conf/2m-2s-async/broker-d-s.properties
```



```
备注：堆内存分配为8G
```

# 问题详情

## 主节点出现闪断

在3月16日出现了CPU sys飙高导致broker约有1分钟的不可用，具体系统日志错误为

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



截图如下：

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219080000.png)



解决方式调整了主节点的内核参数

```
vm.zone_reclaim_mode = 1

vm.min_free_kbytes = 512000
```



当前主节点的内核参数配置为

```
vm.overcommit_memory=1

vm.drop_caches=1

vm.zone_reclaim_mode=0

vm.max_map_count=655360

vm.dirty_background_ratio=25

vm.dirty_ratio=25

vm.dirty_writeback_centisecs=360000

vm.page-cluster=3

vm.swappiness=1

vm.zone_reclaim_mode = 1

vm.min_free_kbytes = 512000
```

```
备注：主节点详细过程参见下文：
```

RocketMQ关于Broker闪断故障排查【实战笔记】

https://mp.weixin.qq.com/s/8EE7_MqV-oZBYqtrdrlUIw



## 从节点挂掉情况

```
调整主节点时把从节点也统一调整了，从节点也调整成了zone_reclaim_mode和min_free_kbytes参数
```

```
vm.overcommit_memory=0

vm.drop_caches=1

vm.zone_reclaim_mode=0

vm.max_map_count=655360

vm.dirty_background_ratio=10

vm.dirty_ratio=10

vm.dirty_writeback_centisecs=360000

vm.page-cluster=3

vm.swappiness=1

vm.zone_reclaim_mode = 1

vm.min_free_kbytes = 512000
```



此后从节点频繁出现了CPU飙高导致节点挂掉的情况，都是CPU sys在飙高，系统日志如下

```
30 2020-03-27T10:35:28.769900+08:00 VECSxxxx kernel: INFO: task AliYunDunUpdate:29054 blocked for more than 120 seconds.

31 2020-03-27T10:35:28.769932+08:00 VECSxxxx kernel: Not tainted 2.6.32-754.17.1.el6.x86_64 #1

32 2020-03-27T10:35:28.771650+08:00 VECS0xxxx kernel: "echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message.

33 2020-03-27T10:35:28.774631+08:00 VECS0xxxx kernel: AliYunDunUpda D ffffffff815592fb 0 29054 1 0x10000080

34 2020-03-27T10:35:28.777500+08:00 VECS0xxxx kernel: ffff8803ef75baa0 0000000000000082 ffff8803ef75ba68 ffff8803ef75ba64

35 2020-03-27T10:35:28.780557+08:00 VECS0xxxx kernel: ffff8803ef75bae8 00000000000014a6 002d61b26118849c ffff880099b18c00

36 2020-03-27T10:35:28.782853+08:00 VECS0xxxx kernel: 00000003f95c0502 0000000000000a2a ffff880488c665f8 ffff8803ef75bfd8
```



为解决“blocked for more than 120 seconds”问题两次调整从节点dirty参数，从50%调整到25%后还是出现飙高情况，然后调整到10%，最终还是出现CPU飙高，截图如下：

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219080121.png)



系统message日志

```
1656 2020-04-01T17:19:52.462310+08:00 VECS0xxxx kernel: INFO: task netstat:30311 blocked for more than 120 seconds.

1657 2020-04-01T17:19:52.464266+08:00 VECS0xxx kernel: Not tainted 2.6.32-754.18.2.el6.x86_64 #1

1658 2020-04-01T17:19:52.466011+08:00 VECS0xxxx kernel: "echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this messa ge.

1659 2020-04-01T17:19:52.468971+08:00 VECS0xxxx kernel: netstat D ffffffff8160d9e0 0 30311 1 0x10000084

1660 2020-04-01T17:19:52.471907+08:00 VECS0xxx kernel: ffff8801b0443d10 0000000000000082 0000000000000000 ffff88011ffc6ab0

1661 2020-04-01T17:19:52.474211+08:00 VECS0xxxx kernel: ffff88000005b5c0 ffff88011ffc6ab0 0000275feaea8627 0000000000000002

1662 2020-04-01T17:19:52.477311+08:00 VECS0xxxx kernel: 00000001029004e5 000000000000030f ffff88011ffc7068 ffff8801b0443fd8

1663 2020-04-01T17:19:52.477323+08:00 VECS0xxxx kernel: Call Trace:

1664 2020-04-01T17:19:52.478328+08:00 VECS0xxxx kernel: [<ffffffff811c4b60>] ? mntput_no_expire+0x30/0x110

1665 2020-04-01T17:19:52.482136+08:00 VECS0xxxx kernel: [<ffffffff8155c6d5>] rwsem_down_failed_common+0x95/0x1d0

1666 2020-04-01T17:19:52.482149+08:00 VECS0xxxx kernel: [<ffffffff81244db5>] ? security_inode_permission+0x25/0x30
```



# 继续调整内核参数

问题首先发生在主节点，调整min_free_kbytes和zone_reclaim_mode参数后，主节点问题没有再发生。是否由于把从节点的也调整了，导致了从节点CPU sys飙高挂掉？

```
将min_free_kbytes调低来看下从节点的运行情况，将min_free_kbytes调低到128000
```

```
小结：但是很遗憾，不管内核参数如何调整。CPU毛刺现象只能缓解，却不能有效根除。
```

CPU spike on slave node: https://github.com/apache/rocketmq/issues/1910



<!--more-->



# 解决方案

为此特在社区提了个问题，也得到了RocketMQ创始人冯嘉和RocketMQ Committer杜恒两位大佬的关注和指点。杜恒大佬有提Linux内核2.6的操作系统有bug会导致类似情况。回想一下原东家的RocketMQ集群运行以来从未出现过如此诡异现象，虽让好友将内核版本发与我，内核版本为3.10。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219080227.png)



问题必须解决，这里是生产环境。所以对线上所有集群主从节点全部从centos6升级到centos7，内核版本也从从2.6升级到3.10。升级后通过半个月左右的观察，未出现CPU毛刺问题。

```
Linux version 3.10.0-1062.4.1.el7.x86_64 (mockbuild@kbuilder.bsys.centos.org) (gcc version 4.8.5 20150623 (Red Hat 4.8.5-39) (GCC) ) #1 SMP Fri Oct 18 17:15:30 UTC 2019
```



