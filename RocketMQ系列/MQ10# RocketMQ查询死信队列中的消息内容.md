---
title: MQ10# RocketMQ查询死信队列中的消息内容
categories: RocketMQ
tags: RocketMQ
date: 2020-12-18 17:14:01
---



# 说明

RocketMQ中当重试消息超过最大重试次数（默认16次），会被发送到%DLQ%开头的死信队列，默认死信队列为只写权限。在有些情况下，想看看死信队列里的内容。



# 更改死信队列权限

```
bin/mqadmin updateTopicPerm -c ClusterB -t %DLQ%online-tst -p 6 -n 192.168.1.x:9876

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=128m; support was removed in 8.0

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=128m; support was removed in 8.0

update topic perm from 2 to 6 in 192.168.1.x:10911 success.

update topic perm from 2 to 6 in 192.168.1.x:10911 success.

update topic perm from 2 to 6 in 192.168.1.x:10911 success.

update topic perm from 2 to 6 in 192.168.1.x:10911 success.

注：将死信队列只写权限更改为读写权限
```



# 查询死信队列状态

```
bin/mqadmin topicStatus -n 192.168.1.x:9876 -t %DLQ%online-tst

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=128m; support was removed in 8.0

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=128m; support was removed in 8.0

Broker Name QID Min Offset Max Offset Last Updated

broker-a 0 0 109 2018-12-10 18:03:08,732

broker-a 1 0 109 2018-12-10 18:03:08,740

broker-a 2 0 110 2018-12-10 18:03:08,750

broker-a 3 0 109 2018-12-10 18:03:08,728
```



<!--more-->



# 根据offset查询消息内容

```
bin/mqadmin queryMsgByOffset -n localhost:9876 -t %DLQ%online-tst -b broker-a -i 0 -o 108

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=128m; support was removed in 8.0

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=128m; support was removed in 8.0

OffsetID: 0A090F2800002A9F000000D70519DD35

OffsetID: 0A090F2800002A9F000000D70519DD35

Topic: %DLQ%online-tst

Tags: [null]

Keys: [null]

Queue ID: 0

Queue Offset: 108

CommitLog Offset: 923503549749

Reconsume Times: 0

Born Timestamp: 2018-12-10 17:59:24,731

Store Timestamp: 2018-12-10 18:03:08,732

Born Host: 10.10.128.183:51889

Store Host: 10.9.15.40:10911

System Flag: 0

Properties: {MIN_OFFSET=0, MAX_OFFSET=109, UNIQ_KEY=0A0A80B78DE818B4AAC22FA2493B01B2, WAIT=true}

Message Body Path: /tmp/rocketmq/msgbodys/0A0A80B78DE818B4AAC22FA2493B01B2

注：使用打印命令消息临时存储在/tmp/rocketmq/msgbodys
```

# 查看消息内容

```
cat /tmp/rocketmq/msgbodys/0A0A80B78DE818B4AAC22FA2490F01AE

Hello RocketMQ430
```

