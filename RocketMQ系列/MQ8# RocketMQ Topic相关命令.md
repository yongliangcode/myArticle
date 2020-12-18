---
title: MQ8# RocketMQ Topic相关命令
categories: RocketMQ
tags: RocketMQ
date: 2020-12-17 15:14:01
---



# 分配MQ

```
bin/mqadmin allocateMQ -n localhost:9876 -t tst-topic -i ipList

ipList 以逗号分隔
```



# 删除topic

```
bin/mqadmin deleteTopic -n localhost:9876 -t zto-example -c DefultCluster
```



# 获取topic的cluster

```
bin/mqadmin topicClusterList -n 192.168.1.x:9876 -t SCANRECORD

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=128m; support was removed in 8.0

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=128m; support was removed in 8.0

ZmsClusterB
```



# 查看Topic列表信息

```
bin/mqadmin topicList -n localhost:9876
```



# 查看Topic路由信息

```
bin/mqadmin topicRoute -n 192.168.1.x:9876 -t SCANRECORD

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=128m; support was removed in 8.0

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=128m; support was removed in 8.0

{

"brokerDatas":[

{

"brokerAddrs":{0:"192.168.1.x:10911",1:"192.168.1.x:10920"

},

"brokerName":"broker-d"

},

{

"brokerAddrs":{0:"192.168.1.x:10911",1:"192.168.1.x:10920"

},

"brokerName":"broker-b"

},

{

"brokerAddrs":{0:"192.168.1.x:10911",1:"192.168.1.x:10915"

},

"brokerName":"broker-c"

},

{

"brokerAddrs":{0:"192.168.1.x:10911",1:"192.168.1.x:10915"

},

"brokerName":"broker-a"

}

],

"filterServerTable":{},

"queueDatas":[

{

"brokerName":"broker-b",

"perm":6,

"readQueueNums":64,

"topicSynFlag":0,

"writeQueueNums":64

},

{

"brokerName":"broker-a",

"perm":6,

"readQueueNums":64,

"topicSynFlag":0,

"writeQueueNums":64

},

{

"brokerName":"broker-c",

"perm":6,

"readQueueNums":64,

"topicSynFlag":0,

"writeQueueNums":64

},

{

"brokerName":"broker-d",

"perm":6,

"readQueueNums":64,

"topicSynFlag":0,

"writeQueueNums":64

}

]

}
```



# 查看topic状态

```
bin/mqadmin topicStatus -n 192.168.1.174:9876 -t SCANRECORD

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=128m; support was removed in 8.0

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=128m; support was removed in 8.0

Broker Name QID Min Offset Max Offset Last Updated

broker-a 0 34757063 39718709 2018-10-22 13:54:32,526

broker-a 1 34760194 39722786 2018-10-22 13:54:32,149

broker-a 2 34765030 39734966 2018-10-22 13:54:32,537

broker-a 3 34775398 39758820 2018-10-22 13:54:32,507

broker-a 4 34804472 39800334 2018-10-22 13:54:32,511

broker-a 5 34835232 39854584 2018-10-22 13:54:32,528

broker-a 6 34863554 39910095 2018-10-22 13:54:32,528
```



# 更新orderConf

```
sh bin/mqadmin updateOrderConf -t SCANRECORD -m put -n 192.168.1.x:9876

-m option type [eg. put|get|delete
```



# 更改Topic权限

```
bin/mqadmin updateTopicPerm -t SCANRECORD -p put -n 192.168.1.x:9876

-p : set topic's permission(2|4|6), intro[2:W; 4:R; 6:RW]

bin/mqadmin updateTopicPerm -c ZmsClusterB -t %DLQ%starunion-freight -p 6 -n 192.168.1.x:9876

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=128m; support was removed in 8.0

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=128m; support was removed in 8.0

update topic perm from 2 to 6 in 192.168.1.x:10911 success.

update topic perm from 2 to 6 in 192.168.1.x:10911 success.

update topic perm from 2 to 6 in 192.168.1.x:10911 success.

update topic perm from 2 to 6 in 192.168.1.x:10911 success.
```





<!--more-->



# 创建/修改Topic

```
sh bin/mqadmin updateTopic -c DefaultCluster -n localhost:9876 -t threezto-test -r 12 -w 12

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=128m; support was removed in 8.0

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=128m; support was removed in 8.0

create topic to x.x.x.40:10911 success.

TopicConfig [topicName=threezto-test, readQueueNums=12, writeQueueNums=12, perm=RW-, topicFilterType=SINGLE_TAG, topicSysFlag=0, order=false]

-b brokerAddr create topic to which broker

-c clusterName create topic to which cluster

-t topic topic name

-r readQueueNums set read queue nums

-w writeQueueNums set write queue nums

-p perm set topic's permission(2|4|6), intro[2:W 4:R; 6:RW]

-o order set topic's order(true|false

-u unit is unit topic (true|false

-s hasUnitSub has unit sub (true|false
```

