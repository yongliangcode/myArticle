---
title: MQ2# RocketMQ Consumer命令
categories: RocketMQ
tags: RocketMQ
abbrlink: e790de00
date: 2020-12-17 11:10:01
---



# 创建订阅组

```
bin/mqadmin updateSubGroup -n localhost:9876 -c DefaultCluster -g zto-tst-consumer
```



# 删除订阅组

```
bin/mqadmin deleteSubGroup -n 192.168.1.x:9876 -c AdpMqCluster -g CODCANCELSIGN

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=128m; support was removed in 8.0

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=128m; support was removed in 8.0

delete subscription group [CODCANCELSIGN] from broker [192.168.1.x:10911] in cluster [AdpMqCluster] success.

delete subscription group [CODCANCELSIGN] from broker [192.168.1.x:10911] in cluster [AdpMqCluster] success.

delete subscription group [CODCANCELSIGN] from broker [192.168.1.x:10919] in cluster [AdpMqCluster] success.

delete subscription group [CODCANCELSIGN] from broker [192.168.1.x:10915] in cluster [AdpMqCluster] success.

delete subscription group [CODCANCELSIGN] from broker [192.168.1.x:10915] in cluster [AdpMqCluster] success.

delete subscription group [CODCANCELSIGN] from broker [192.168.1.x:10919] in cluster [AdpMqCluster] success.

delete topic [%RETRY%CODCANCELSIGN] from cluster [xMqCluster] success.

delete topic [%RETRY%CODCANCELSIGN] from NameServer success.

delete topic [%DLQ%CODCANCELSIGN] from cluster [xMqCluster] success.

delete topic [%DLQ%CODCANCELSIGN] from NameServer success.
```



<!--more-->



# 查看消费组情况及IP地址

```
bin/mqadmin consumerStatus -g TraceToCaiNiao -n 192.168.1.x:9876

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=128m; support was removed in 8.0

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=128m; support was removed in 8.0

001 192.168.12.122@25063 V3_5_8 1539756615406/192.168.x.x@25063

002 192.168.13.96@3351 V3_5_8 1539756615406/192.168.x.x@3351

003 192.168.x.x@13385 V3_5_8 1539756615406/192.168.x.x@13385

004 192.168.x.x@6691 V3_5_8 1539756615406/192.168.x.x@6691
```



# 查看消费组情况

```
bin/mqadmin consumerProgress -n 192.168.1.x:9876 -g SortComplementConsumer

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=128m; support was removed in 8.0

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=128m; support was removed in 8.0

Topic Broker Name QID Broker Offset Consumer Offset Diff LastTime

%RETRY%SortComplementConsumer broker-a 0 1 1 0 2018-10-22 12:43:13

%RETRY%SortComplementConsumer broker-b 0 2 2 0 2018-10-22 12:43:13

%RETRY%SortComplementConsumer broker-c 0 2 2 0 2018-10-22 12:43:28

%RETRY%SortComplementConsumer broker-d 0 2 2 0 2018-10-22 12:44:33

SCANRECORD broker-a 0 39717505 39717502 3 2018-10-22 13:53:21

SCANRECORD broker-a 1 39721504 39721502 2 2018-10-22 13:53:21

SCANRECORD broker-a 2 39733704 39733703 1 2018-10-22 13:53:21
```





