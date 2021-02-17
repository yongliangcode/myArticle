---
title: MQ9# RocketMQ Offset 相关命令
categories: RocketMQ
tags: RocketMQ
date: 2020-12-17 17:14:01
---



# 克隆消费组的offset（同一个集群）

```
bin/mqadmin cloneGroupOffset -n 192.168.1.x:9876 -s SCANRECORD_GROUP -d my-tst-cloneoffset -t SCANRECORD

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=128m; support was removed in 8.0

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=128m; support was removed in 8.0

clone group offset success. srcGroup[SCANRECORD_GROUP], destGroup=[my-tst-cloneoffset], topic[SCANRECORD][baseuser@HZPL001180 rocketmq]$
```



<!--more-->



# 验证

```
bin/mqadmin statsAll -t SCANRECORD -n 192.168.1.x:9876

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=128m; support was removed in 8.0

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=128m; support was removed in 8.0

Topic Consumer Group Accumulation InTPS OutTPS InMsg24Hour OutMsg24Hour

SCANRECORD ZtoSignGroup 2290 3063.95 522.72 233588408 29106863

SCANRECORD newOpenPartnerDeadlineJob 2641 3063.95 522.72 233588408 29052057

SCANRECORD smartidivision-scanrecord-dis 3110 3063.95 399.03 233588408 30052791

SCANRECORD PanguRecordGroup 574 3063.95 2041.17 233588408 174462813

SCANRECORD code-send-consumer 881 3063.95 1162.63 233588408 60072932

SCANRECORD shenZhouOneSiteKpiConsumer 669 3063.95 1798.57 233588408 137543100

SCANRECORD SortComplementConsumer 2746 3063.95 522.72 233588408 11466488

SCANRECORD **my-tst-cloneoffset** 0 3063.95 0.00 233588408 0

SCANRECORD tf_wonder_waybill_center_scanrec 2516 3063.95 522.72 233588408 29056664

SCANRECORD dpmComScanRecordConsumer 0 3063.95 0.00 233588408 0

SCANRECORD zms_syncer_SCANRECORD 286 3063.95 3063.73 233588408 233588419
```





# 根据时间重置消费位点

```
bin/mqadmin resetOffsetByTime -n 192.168.1.x:9876 -g ReceiveOrderGroupNew -t SCANRECORD -s now

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=128m; support was removed in 8.0

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=128m; support was removed in 8.0

rollback consumer offset by specified group[ReceiveOrderGroupNew], topic[SCANRECORD], force[true], timestamp(string)[now], timestamp(long)[1540196941761]

brokerName queueId offset

broker-a 20 40375806

broker-b 55 40209057

broker-c 24 40151853

broker-b 22 40284014

broker-a 53 40321569

broker-d 59 39980239

bin/mqadmin resetOffsetByTime -n 192.168.1.x:9876 -g ReceiveOrderGroupNew -t SCANRECORD -s 1540196941761

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=128m; support was removed in 8.0

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=128m; support was removed in 8.0

rollback consumer offset by specified group[ReceiveOrderGroupNew], topic[SCANRECORD], force[true], timestamp(string)[1540196941761], timestamp(long)[1540196941761]

brokerName queueId offset

broker-a 20 40375806

broker-b 55 40209057

broker-c 24 40151853

broker-b 22 40284014

"g", "group", true, "set the consumer group"

"s", "timestamp", true, "set the timestamp[now|currentTimeMillis|yyyy-MM-dd#HH:mm:ss:SSS]"

"f", "force", true, "set the force rollback by timestamp switch[true|false]"

"c", "cplus", false, "reset c++ client offset"
```

