---
title: Kafka3# 集群平滑扩容及Leader均衡
categories: Kafka
tags: Kafka
date: 2019-05-19 11:55:01
---



# 问题描述

当集群中新增加节点时，需要对已有的topic的副本进行迁移，以平衡流量。以公司集群扩增两个节点broker 4和broker 5为例说明操作过程。

问题：怎么做才能做到平滑呢？即尽量做到客户端应用无感知。

为了解决平滑问题，分为三步完成

1.副本均衡设置

​	对Topic的副本平均分配到各个broker上

2.偏好副本设置

​	将偏好副本平均分配到各个broker上, 为Leader均衡做准备

3.Leader均衡

​	执行Leader平衡



<!--more-->



**要点备注** 

第一步操作要点

为了不影响客户端使用，保持原有集群Leader副本现状不变，将其他副本平均分配到各个broker上。

副本均衡设置后，需要等待与观察，最终让这些设置的副本进入ISR列表后（新加入的副本跟上了Leader副本数据进度）再执行第二步。



第二步操作要点

在第二步偏好副本设置时，将偏好副本均匀的分布broker上，每个broker上的偏好副本数量=分区总数/broker数量



第三步操作要点

这一步开始实际leader均衡操作



# 副本均衡设置

**查看当前Topic的副本分配情况**

bin/kafka-topics.sh --describe --zookeeper 192.168.x.x:2181,192.168.x.x:2181,192.168.x.x:2181 --topic zto_sign_disfee



```
Topic:zto_sign_disfee   PartitionCount:12       ReplicationFactor:3     Configs:
Topic: zto_sign_disfee  Partition: 0    Leader: 2       Replicas: 2,0,3 Isr: 3,0,2
Topic: zto_sign_disfee  Partition: 1    Leader: 0       Replicas: 0,1,3 Isr: 3,0,1
Topic: zto_sign_disfee  Partition: 2    Leader: 1       Replicas: 1,2,3 Isr: 3,1,2
Topic: zto_sign_disfee  Partition: 3    Leader: 2       Replicas: 2,1,3 Isr: 3,1,2
Topic: zto_sign_disfee  Partition: 4    Leader: 0       Replicas: 0,2,3 Isr: 3,0,2
Topic: zto_sign_disfee  Partition: 5    Leader: 1       Replicas: 1,0,2 Isr: 0,1,2
Topic: zto_sign_disfee  Partition: 6    Leader: 2       Replicas: 2,0,1 Isr: 0,1,2
Topic: zto_sign_disfee  Partition: 7    Leader: 0       Replicas: 0,1,3 Isr: 3,0,1
Topic: zto_sign_disfee  Partition: 8    Leader: 1       Replicas: 1,2,0 Isr: 0,1,2
Topic: zto_sign_disfee  Partition: 9    Leader: 2       Replicas: 2,1,3 Isr: 3,1,2
Topic: zto_sign_disfee  Partition: 10   Leader: 0       Replicas: 0,2,3 Isr: 3,0,2
Topic: zto_sign_disfee  Partition: 11   Leader: 1       Replicas: 1,0,3 Isr: 3,0,1
```



**准备执行计划的Topic**

```
echo '{"version":1,"topics":[{"topic":"zto_sign_disfee"}]}' > plan02/zto_sign_disfee.json
```



**生成执行计划**

bin/kafka-reassign-partitions.sh --zookeeper 192.168.x.x:2181,192.168.x.x:2181,192.168.x.x:2181 --topics-to-move-json-file plan02/zto_sign_disfee.json --broker-list "0,1,2,3,4,5" --generate

```
Current partition replica assignment
{"version":1,"partitions":[{"topic":"zto_sign_disfee","partition":11,"replicas":[1,0,3]},{"topic":"zto_sign_disfee","partition":7,"replicas":[0,1,3]},{"topic":"zto_sign_disfee","partition":3,"replicas":[2,1,3]},{"topic":"zto_sign_disfee","partition":1,"replicas":[0,1,3]},{"topic":"zto_sign_disfee","partition":2,"replicas":[1,2,3]},{"topic":"zto_sign_disfee","partition":5,"replicas":[1,0,2]},{"topic":"zto_sign_disfee","partition":4,"replicas":[0,2,3]},{"topic":"zto_sign_disfee","partition":0,"replicas":[2,0,3]},{"topic":"zto_sign_disfee","partition":9,"replicas":[2,1,3]},{"topic":"zto_sign_disfee","partition":8,"replicas":[1,2,0]},{"topic":"zto_sign_disfee","partition":6,"replicas":[2,0,1]},{"topic":"zto_sign_disfee","partition":10,"replicas":[0,2,3]}]}
Proposed partition reassignment configuration
{"version":1,"partitions":[{"topic":"zto_sign_disfee","partition":11,"replicas":[4,3,5]},{"topic":"zto_sign_disfee","partition":7,"replicas":[0,5,1]},{"topic":"zto_sign_disfee","partition":1,"replicas":[0,4,5]},{"topic":"zto_sign_disfee","partition":3,"replicas":[2,0,1]},{"topic":"zto_sign_disfee","partition":4,"replicas":[3,1,2]},{"topic":"zto_sign_disfee","partition":2,"replicas":[1,5,0]},{"topic":"zto_sign_disfee","partition":5,"replicas":[4,2,3]},{"topic":"zto_sign_disfee","partition":0,"replicas":[5,3,4]},{"topic":"zto_sign_disfee","partition":9,"replicas":[2,1,3]},{"topic":"zto_sign_disfee","partition":8,"replicas":[1,0,2]},{"topic":"zto_sign_disfee","partition":6,"replicas":[5,4,0]},{"topic":"zto_sign_disfee","partition":10,"replicas":[3,2,4]}]}
```



**重新分配副本**

zto_sign_disfee-reassign.json

```
{
    "version": 1,
    "partitions": [{
        "topic": "zto_sign_disfee",
        "partition": 11,
        "replicas": [1,
        4,
        3]
    },
    {
        "topic": "zto_sign_disfee",
        "partition": 7,
        "replicas": [0,
        4,
        3]
    },
    {
        "topic": "zto_sign_disfee",
        "partition": 3,
        "replicas": [2,
        4,
        3]
    },
    {
        "topic": "zto_sign_disfee",
        "partition": 1,
        "replicas": [0,
        4,
        3]
    },
    {
        "topic": "zto_sign_disfee",
        "partition": 2,
        "replicas": [1,
        5,
        3]
    },
    {
        "topic": "zto_sign_disfee",
        "partition": 5,
        "replicas": [1,
        4,
        2]
    },
    {
        "topic": "zto_sign_disfee",
        "partition": 4,
        "replicas": [0,
        5,
        3]
    },
    {
        "topic": "zto_sign_disfee",
        "partition": 0,
        "replicas": [2,
        4,
        5]
    },
    {
        "topic": "zto_sign_disfee",
        "partition": 9,
        "replicas": [2,
        1,
        5]
    },
    {
        "topic": "zto_sign_disfee",
        "partition": 8,
        "replicas": [1,
        5,
        0]
    },
    {
        "topic": "zto_sign_disfee",
        "partition": 6,
        "replicas": [2,
        0,
        1]
    },
    {
        "topic": "zto_sign_disfee",
        "partition": 10,
        "replicas": [0,
        2,
        5]
    }]
}

```



**执行计划**

bin/kafka-reassign-partitions.sh --zookeeper 192.168.x.x:2181,192.168.x.x:2181,192.168.x.x:2181 --reassignment-json-file plan02/zto_sign_disfee-reassign.json --execute

```
Current partition replica assignment
{"version":1,"partitions":[{"topic":"zto_sign_disfee","partition":11,"replicas":[1,0,3]},{"topic":"zto_sign_disfee","partition":7,"replicas":[0,1,3]},{"topic":"zto_sign_disfee","partition":3,"replicas":[2,1,3]},{"topic":"zto_sign_disfee","partition":1,"replicas":[0,1,3]},{"topic":"zto_sign_disfee","partition":2,"replicas":[1,2,3]},{"topic":"zto_sign_disfee","partition":5,"replicas":[1,0,2]},{"topic":"zto_sign_disfee","partition":4,"replicas":[0,2,3]},{"topic":"zto_sign_disfee","partition":0,"replicas":[2,0,3]},{"topic":"zto_sign_disfee","partition":9,"replicas":[2,1,3]},{"topic":"zto_sign_disfee","partition":8,"replicas":[1,2,0]},{"topic":"zto_sign_disfee","partition":6,"replicas":[2,0,1]},{"topic":"zto_sign_disfee","partition":10,"replicas":[0,2,3]}]}
Save this to use as the --reassignment-json-file option during rollback
Successfully started reassignment of partitions.
```



**验证** 

bin/kafka-reassign-partitions.sh --zookeeper 192.168.x.x:2181,192.168.x.x:2181,192.168.x.x:2181 --reassignment-json-file plan02/zto_sign_disfee-reassign.json --verify



```
Status of partition reassignment:
Reassignment of partition [zto_sign_disfee,9] completed successfully
Reassignment of partition [zto_sign_disfee,10] completed successfully
Reassignment of partition [zto_sign_disfee,6] completed successfully
Reassignment of partition [zto_sign_disfee,7] completed successfully
Reassignment of partition [zto_sign_disfee,5] completed successfully
Reassignment of partition [zto_sign_disfee,3] completed successfully
Reassignment of partition [zto_sign_disfee,11] completed successfully
Reassignment of partition [zto_sign_disfee,1] completed successfully
Reassignment of partition [zto_sign_disfee,8] completed successfully
Reassignment of partition [zto_sign_disfee,2] completed successfully
Reassignment of partition [zto_sign_disfee,0] completed successfully
Reassignment of partition [zto_sign_disfee,4] completed successfully
```



**查看topic副本情况**

bin/kafka-topics.sh --describe --zookeeper 192.168.x.x:2181,192.168.x.x:2181,192.168.x.x:2181 --topic zto_sign_disfee

```
Topic:zto_sign_disfee   PartitionCount:12       ReplicationFactor:3     Configs:
Topic: zto_sign_disfee  Partition: 0    Leader: 2       Replicas: 2,4,5 Isr: 5,2,4
Topic: zto_sign_disfee  Partition: 1    Leader: 0       Replicas: 0,4,3 Isr: 3,0,4
Topic: zto_sign_disfee  Partition: 2    Leader: 1       Replicas: 1,5,3 Isr: 3,1,5
Topic: zto_sign_disfee  Partition: 3    Leader: 2       Replicas: 2,4,3 Isr: 3,2,4
Topic: zto_sign_disfee  Partition: 4    Leader: 0       Replicas: 0,5,3 Isr: 3,0,5
Topic: zto_sign_disfee  Partition: 5    Leader: 1       Replicas: 1,4,2 Isr: 1,2,4
Topic: zto_sign_disfee  Partition: 6    Leader: 2       Replicas: 2,0,1 Isr: 0,1,2
Topic: zto_sign_disfee  Partition: 7    Leader: 0       Replicas: 0,4,3 Isr: 3,0,4
Topic: zto_sign_disfee  Partition: 8    Leader: 1       Replicas: 1,5,0 Isr: 0,1,5
Topic: zto_sign_disfee  Partition: 9    Leader: 2       Replicas: 2,1,5 Isr: 1,2,5
Topic: zto_sign_disfee  Partition: 10   Leader: 0       Replicas: 0,2,5 Isr: 0,2,5
Topic: zto_sign_disfee  Partition: 11   Leader: 1       Replicas: 1,4,3 Isr: 3,1,4
```



# 偏好副本设置

**准备计划文件**

zto_sign_disfee-perf-reassign.json

```
{"version":1,"partitions":[{"topic":"zto_sign_disfee","partition":11,"replicas":[3,4,1]},{"topic":"zto_sign_disfee","partition":7,"replicas":[4,0,3]},{"topic":"zto_sign_disfee","partition":3,"replicas":[3,4,2]},{"topic":"zto_sign_disfee","partition":1,"replicas":[4,0,3]},{"topic":"zto_sign_disfee","partition":2,"replicas":[5,1,3]},{"topic":"zto_sign_disfee","partition":5,"replicas":[1,4,2]},{"topic":"zto_sign_disfee","partition":4,"replicas":[0,5,3]},{"topic":"zto_sign_disfee","partition":0,"replicas":[2,4,5]},{"topic":"zto_sign_disfee","partition":9,"replicas":[2,1,5]},{"topic":"zto_sign_disfee","partition":8,"replicas":[5,1,0]},{"topic":"zto_sign_disfee","partition":6,"replicas":[1,0,2]},{"topic":"zto_sign_disfee","partition":10,"replicas":[0,2,5]}]}
```



**执行与验证** 

bin/kafka-reassign-partitions.sh --zookeeper 192.168.x.x:2181,192.168.x.x:2181,192.168.x.x:2181 --reassignment-json-file plan02/zto_sign_disfee-perf-reassign.json --execute

bin/kafka-reassign-partitions.sh --zookeeper 192.168.x.x:2181,192.168.x.x:2181,192.168.x.x:2181 --reassignment-json-file plan02/zto_sign_disfee-perf-reassign.json --verify



# Leader均衡

**准备计划文件** 

zto_sign_disfee-leader-reassign.json

```
{"partitions":[{"topic":"zto_sign_disfee","partition":0},{"topic":"zto_sign_disfee","partition":1},{"topic":"zto_sign_disfee","partition":2},{"topic":"zto_sign_disfee","partition":3},{"topic":"zto_sign_disfee","partition":4},{"topic":"zto_sign_disfee","partition":5},{"topic":"zto_sign_disfee","partition":6},{"topic":"zto_sign_disfee","partition":7},{"topic":"zto_sign_disfee","partition":8},{"topic":"zto_sign_disfee","partition":9},{"topic":"zto_sign_disfee","partition":10},{"topic":"zto_sign_disfee","partition":11}]}
```



**执行计划**

bin/kafka-preferred-replica-election.sh --zookeeper 192.168.x.x:2181,192.168.x.x:2181,192.168.x.x:2181 --path-to-json-file plan02/zto_sign_disfee-leader-reassign.json



