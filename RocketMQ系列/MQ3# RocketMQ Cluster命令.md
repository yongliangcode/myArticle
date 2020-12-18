---
title: MQ3# RocketMQ Cluster命令
categories: RocketMQ
tags: RocketMQ
date: 2020-12-17 11:12:01
---



# 查看集群信息

```
bin/mqadmin clusterList -n localhost:9876

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=128m; support was removed in 8.0

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=128m; support was removed in 8.0

Cluster Name Broker Name BID Addr Version InTPS OutTPS

AdpMqCluster broker-a 0 192.168.1.x:10911 V3_4_6 2819.22 1077.39

AdpMqCluster broker-b 0 192.168.1.x:10911 V3_4_6 2839.42 1070.89

AdpMqCluster broker-c 0 192.168.1.x:10919 V3_4_6 2800.12 1058.39

AdpMqCluster broker-d 0 192.168.1.x:10915 V3_4_6 2897.04 1126.04

AdpMqCluster broker-e 0 192.168.1.x:10919 V3_4_6 2987.20 1181.88

AdpMqCluster broker-f 0 192.168.1.x:10915 V3_4_6 3010.40 1187.08
```



<!--more-->



# 创建Topic

```
bin/mqadmin updateTopic -n localhost:9876 -c DefaultCluster -t zto-example
```



# 删除Topic

```
bin/mqadmin deleteTopic -n localhost:9876 -c DefaultCluster -t zto-example
```