---
title: MQ11# RocketMQ集群生产环境节点下线操作
categories: RocketMQ
tags: RocketMQ
date: 2020-12-19 18:14:01
---



# 现状描述

集群其中一台物理机未知原因导致单用户无法登陆机器，该物理机需要重启修改密码或者重装系统。该台为master节点，运行正常。

配置策略为：异步刷盘 & 主从异步复制



如果直接下线该master，由于主从异步复制，可能导致部分消息来不及复制到slave造成消息丢失。所以该方案不可行。

另一种方案选择：关闭该broker的写入权限，待该broker不再有写入和消费时，再下线该节点。



# 关闭broker写权限

2表示只写权限，4表示只读权限，6表示读写权限

```
bin/mqadmin updateBrokerConfig -b 192.168.x.x:10911 -n 192.168.x.x:9876 -k brokerPermission -v 4

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=128m; support was removed in 8.0

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=128m; support was removed in 8.0

update broker config success, 192.168.x.x:10911
```



# 观察节点流量

```
bin/mqadmin clusterList -n 192.168.x.x:9876

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=128m; support was removed in 8.0

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=128m; support was removed in 8.0

\#Cluster Name #Broker Name #BID #Addr #Version #InTPS(LOAD) #OutTPS(LOAD) #PCWait(ms) #Hour #SPACE

ZmsClusterA broker-a 0 192.168.x.x:10911 V4_1_0_SNAPSHOT 2492.95(0,0ms) 2269.27(1,0ms) 0 137.57 0.1861

ZmsClusterA broker-a 1 192.168.x.x:10911 V4_1_0_SNAPSHOT 2485.45(0,0ms) 0.00(0,0ms) 0 125.26 0.3055

ZmsClusterA broker-b 0 192.168.x.x:10911 V4_1_0_SNAPSHOT 26.47(0,0ms) 26.08(0,0ms) 0 137.24 0.1610

ZmsClusterA broker-b 1 192.168.x.x:10915 V4_1_0_SNAPSHOT 20.47(0,0ms) 0.00(0,0ms) 0 125.22 0.3055

ZmsClusterA broker-c 0 192.168.x.x:10911 V4_1_0_SNAPSHOT 2061.09(0,0ms) 1967.30(0,0ms) 0 125.28 0.2031

ZmsClusterA broker-c 1 192.168.x.x:10911 V4_1_0_SNAPSHOT 2048.20(0,0ms) 0.00(0,0ms) 0 137.51 0.2789

ZmsClusterA broker-d 0 192.168.x.x:10911 V4_1_0_SNAPSHOT 2017.40(0,0ms) 1788.32(0,0ms) 0 125.22 0.1261

ZmsClusterA broker-d 1 192.168.x.x:10915 V4_1_0_SNAPSHOT 2026.50(0,0ms) 0.00(0,0ms) 0 137.61 0.2789
```



观察InTPS和OutTPS，理想情况都为零时，并不再变化时，则该节点可下线了。

然而，在实际过程中并没有出现为零的情况，InTPS和OutTPS总是有值，有时个位数字有时是两位数字，大部分时间在20多的值。此刻要分析下broker目前的消费状态。



<!--more-->



# 观察broker消费状态

```
sh bin/mqadmin brokerConsumeStats -b 192.168.x.x:10911 -n 192.168.x.x:9876 >> brokerConsumeStats.tmp
```



查看brokerConsumeStats.tmp，主要查看#LastTime和#Diff。 发现%RETRY%重试类队列#Diff有很微小（1或者3）的数据，而其他topic均为0. LastTime时间最新也是发生在%RETRY%队列中。此时可以让该节点下线操作。



```
#Topic #Group #Broker Name #QID #Broker Offset #Consumer Offset #Diff #LastTime

SV_Multi_Message ZTO_SV_EmchatWebConsumerGroup broker-b 0 2171742 2171742 0 2019-04-24 23:38:09

SV_Multi_Message ZTO_SV_EmchatWebConsumerGroup broker-b 1 2171756 2171756 0 2019-04-24 23:38:50

SV_Multi_Message ZTO_SV_EmchatWebConsumerGroup broker-b 2 2171740 2171740 0 2019-04-24 23:42:58

SV_Multi_Message ZTO_SV_EmchatWebConsumerGroup broker-b 3 2171759 2171759 0 2019-04-24 23:40:44

SV_Multi_Message ZTO_SV_EmchatWebConsumerGroup broker-b 4 2171743 2171743 0 2019-04-24 23:32:48

SV_Multi_Message ZTO_SV_EmchatWebConsumerGroup broker-b 5 2171740 2171740 0 2019-04-24 23:35:58

SV_Multi_Message ZTO_SV_EmchatWebConsumerGroup broker-b 6 2171758 2171758 0 2019-04-24 23:36:23

SV_Multi_Message ZTO_SV_EmchatWebConsumerGroup broker-b 7 2171740 2171740 0 2019-04-24 23:37:50

%RETRY%ZTO_SV_EmchatWebConsumerG ZTO_SV_EmchatWebConsumerGroup broker-b 0 61876 61876 0 2019-04-24 10:09:04

%RETRY%SVC_TRACK_CONSUMER SVC_TRACK_CONSUMER broker-b 0 497968 497968 0 2019-04-19 12:51:24

SVC_TRACK_TOPIC SVC_TRACK_CONSUMER broker-b 0 191710 191710 0 2019-04-24 23:44:22

SVC_TRACK_TOPIC SVC_TRACK_CONSUMER broker-b 1 191706 191706 0 2019-04-24 23:44:25

SVC_TRACK_TOPIC SVC_TRACK_CONSUMER broker-b 2 191697 191697 0 2019-04-24 23:44:44

SVC_TRACK_TOPIC SVC_TRACK_CONSUMER broker-b 3 191695 191695 0 2019-04-24 23:44:47

SVC_TRACK_TOPIC SVC_TRACK_CONSUMER broker-b 4 191688 191688 0 2019-04-24 23:44:47

SVC_TRACK_TOPIC SVC_TRACK_CONSUMER broker-b 5 191683 191683 0 2019-04-24 23:44:48

SVC_TRACK_TOPIC SVC_TRACK_CONSUMER broker-b 6 191676 191676 0 2019-04-24 23:44:49

SVC_TRACK_TOPIC SVC_TRACK_CONSUMER broker-b 7 191672 191672 0 2019-04-24 23:44:49
```



# borker读写权限恢复

```
bin/mqadmin updateBrokerConfig -b 192.168.x.x:10911 -n 192.168.x.x:9876 -k brokerPermission -v 6
```



# 观察各节点流量是否正常

```
bin/mqadmin clusterList -n 192.168.x.x:9876

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=128m; support was removed in 8.0

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=128m; support was removed in 8.0

\#Cluster Name #Broker Name #BID #Addr #Version #InTPS(LOAD) #OutTPS(LOAD) #PCWait(ms) #Hour #SPACE

ZmsClusterA broker-a 0 192.168.x.x:10911 V4_1_0_SNAPSHOT 2492.95(0,0ms) 2269.27(1,0ms) 0 137.57 0.1861

ZmsClusterA broker-a 1 192.168.x.x:10911 V4_1_0_SNAPSHOT 2485.45(0,0ms) 0.00(0,0ms) 0 125.26 0.3055

ZmsClusterA broker-b 0 192.168.x.x:10911 V4_1_0_SNAPSHOT 2299.47(0,0ms) 2226.08(0,0ms) 0 137.24 0.1610

ZmsClusterA broker-b 1 192.168.x.x:10915 V4_1_0_SNAPSHOT 2280.47(0,0ms) 0.00(0,0ms) 0 125.22 0.3055

ZmsClusterA broker-c 0 192.168.x.x:10911 V4_1_0_SNAPSHOT 2061.09(0,0ms) 1967.30(0,0ms) 0 125.28 0.2031

ZmsClusterA broker-c 1 192.168.x.x:10911 V4_1_0_SNAPSHOT 2048.20(0,0ms) 0.00(0,0ms) 0 137.51 0.2789

ZmsClusterA broker-d 0 192.168.x.x:10911 V4_1_0_SNAPSHOT 2017.40(0,0ms) 1788.32(0,0ms) 0 125.22 0.1261

ZmsClusterA broker-d 1 192.168.x.x:10915 V4_1_0_SNAPSHOT 2026.50(0,0ms) 0.00(0,0ms) 0 137.61 0.2789
```

