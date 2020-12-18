---
title: MQ1# RocketMQ Connection命令
categories: RocketMQ
tags: RocketMQ
abbrlink: 6957c801
date: 2020-12-14 11:10:01
---



# 消费端连接信息



```
bin/mqadmin consumerConnection -g T_SCANRECORD_NEW_GROUP -n 192.168.1.x:9876



Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=128m; support was removed in 8.0

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=128m; support was removed in 8.0

001 127.0.0.1@60190 192.168.2.x:50104 JAVA V3_2_6_SNAPSHOT

002 127.0.0.1@7454 192.168.7.x:27692 JAVA V3_2_6_SNAPSHOT

003 127.0.0.1@31600 192.168.21.x:40243 JAVA V3_2_6_SNAPSHOT

004 127.0.0.1@21322 192.168.21.x:31416 JAVA V3_2_6_SNAPSHOT

005 127.0.0.1@39557 192.168.2.x:45959 JAVA V3_2_6_SNAPSHOT

006 127.0.0.1@24765 192.168.7.x:15166 JAVA V3_2_6_SNAPSHOT

007 127.0.0.1@26993 192.168.7.x:57864 JAVA V3_2_6_SNAPSHOT

008 127.0.0.1@652 192.168.21.x:59261 JAVA V3_2_6_SNAPSHOT

009 127.0.0.1@10388 192.168.21.x:1671 JAVA V3_2_6_SNAPSHOT

Below is subscription:001 Topic: %RETRY%T_SCANRECORD_NEW_GROUP SubExpression: *

002 Topic: T_SCANRECORD_NEW SubExpression: 1||2||3||4 || 5 || -5||7 || 45

ConsumeType: CONSUME_PASSIVELY

MessageModel: CLUSTERING

ConsumeFromWhere: CONSUME_FROM_FIRST_OFFSET
```



<!--more-->



# 生产连接信息

```
bin/mqadmin producerConnection -t T_SCANRECORD_NEW -g producetGroup -n 192.168.1.x:9876
```







