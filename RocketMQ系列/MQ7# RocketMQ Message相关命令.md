---
title: MQ7# RocketMQ Message相关命令
categories: RocketMQ
tags: RocketMQ
date: 2020-12-17 14:14:01
---

# 发送测试消息

```
bin/mqadmin checkMsgSendRT -n 192.168.x.x:9876 -t topic_online_test -s 1024

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=128m; support was removed in 8.0

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=128m; support was removed in 8.0Broker Name           QID #Send Result      RTbroker-a             0   true          314broker-a             1   true          0broker-a             2   true          0

broker-a             3   true          0

broker-b             0   true          2

broker-b             1   true          1

broker-b             2   true          1

broker-b             3   true          0

broker-a             0   true          1

broker-a             1   true          0

broker-a             2   true          0

broker-a             3   true          1

broker-b             0   true          4

-t topic topic name

-a amount message amout | default 100

-s size message size | default 128 Byte
```

<!--more-->

# print message by queue

```
bin/mqadmin printMsgByQueue -n 192.168.1.x:9876 -b 192.168.1.x -i 0
```





# 打印topic中的信息

```
bin/mqadmin printMsg -n 192.168.1.x:9876 -t SCANRECORD

"t", "topic", true, "topic name"

"c", "charsetName ", true, "CharsetName(eg: UTF-8,GBK)"

"s", "subExpression ", true, "Subscribe Expression(eg: TagA || TagB)"

"b", "beginTimestamp ", true,  Begin timestamp[currentTimeMillis|yyyy-MM-dd#HH:mm:ss:SSS]

"e", "endTimestamp ", true, End timestamp[currentTimeMillis|yyyy-MM-dd#HH:mm:ss:SSS]

"d", "printBody ", true,"print body"

MSGID: C0A81F8166832F2C9B1953831616FB45 MessageExt [queueId=20, storeSize=686, queueOffset=35052080, sysFlag=0, bornTimestamp=1539724299798, bornHost=/192.168.x.x:29781, storeTimestamp=1539724299179, storeHost=/192.168.x.x:10911, msgId=C0A801B400002A9F000005F308F9B3A5, commitLogOffset=6541385773989, bodyCRC=1167936923, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message [topic=SCANRECORD, flag=0, properties={MIN_OFFSET=35046369, MAX_OFFSET=40298522, KEYS=1dd04932-b7d9-43ee-8b31-39d16dd043d4, UNIQ_KEY=C0A81F8166832F2C9B1953831616FB45, WAIT=true, TAGS=2}, body=484]] BODY: {"billCode":"640010206680","blReturnBillId":0,"blUntreadPieceId":0,"clazz":"2","dataFrom":"2","dispatchId":0,"ownerBagNo":"3783585805","pdaCode":"S50117101164","piece":1,"preOrNexStaId":14761,"preOrNextStation":"\u6F6E\u6C55\u4E2D\u5FC3","prepProvinceId":440000,"registerDate":1539724298000,"scanDate":1539724241000,"scanMan":"\u9648\u4E9A\u5973","scanManCode":"020019.0865","scanProvinceId":440000,"scanSite":"\u5E7F\u5DDE\u5E7F\u56ED","scanSiteId":1051037,"scanType":"\u53D1\u4EF6"}
```



# 通过messageId查询消息

```
bin/mqadmin queryMsgById -n 192.168.x.x:9876 -i C0A801B400002A9F000005F308F9B3A5

"i", "msgId", true, "Message Id"

"consumerGroup", true, "consumer group name"

"clientId", true, "The consumer's client id"

"sendMessage", true, "resend message"

"unitName", true, "unit name"
```

```
bin/mqadmin queryMsgById -n 192.168.x.x:9876 -i C0A801B400002A9F000005F308F9B3A5

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=128m; support was removed in 8.0

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=128m; support was removed in 8.0

OffsetID:      C0A801B400002A9F000005F308F9B3A5

OffsetID:      C0A801B400002A9F000005F308F9B3A5

Topic:        SCANRECORD

Tags:        [2]

Keys:        [1dd04932-b7d9-43ee-8b31-39d16dd043d4]

Queue ID:      20

Queue Offset:    35052080

CommitLog Offset:  6541385773989

Reconsume Times:   0

Born Timestamp:   2018-10-17 05:11:39,798

Store Timestamp:   2018-10-17 05:11:39,179

Born Host:      192.168.31.129:29781

Store Host:     192.168.1.180:10911

System Flag:     0

Properties:     {KEYS=1dd04932-b7d9-43ee-8b31-39d16dd043d4, UNIQ_KEY=C0A81F8166832F2C9B1953831616FB45, WAIT=true, TAGS=2}

Message Body Path:  /tmp/rocketmq/msgbodys/C0A81F8166832F2C9B1953831616FB45

MessageTrack [consumerGroup=ZtoSignGroup, trackType=CONSUMED_BUT_FILTERED, exceptionDesc=null]MessageTrack [consumerGroup=newOpenPartnerDeadlineJob, trackType=CONSUMED_BUT_FILTERED, exceptionDesc=null]MessageTrack [consumerGroup=smartidivision-scanrecord-dis, trackType=CONSUMED_BUT_FILTERED, exceptionDesc=null]MessageTrack [consumerGroup=PanguRecordGroup, trackType=CONSUMED, exceptionDesc=null]MessageTrack [consumerGroup=code-send-consumer, trackType=CONSUMED_BUT_FILTERED, exceptionDesc=null]MessageTrack [consumerGroup=shenZhouOneSiteKpiConsumer, trackType=CONSUMED, exceptionDesc=null]MessageTrack [consumerGroup=SortComplementConsumer, trackType=CONSUMED_BUT_FILTERED, exceptionDesc=null]MessageTrack [consumerGroup=tf_wonder_waybill_center_scanrecord, trackType=CONSUMED_BUT_FILTERED, exceptionDesc=null]MessageTrack [consumerGroup=dpmComScanRecordConsumer, trackType=UNKNOWN, exceptionDesc=org.apache.rocketmq.client.exception.MQClientException: CODE: 17 DESC: No topic route info in name server for the topic: %RETRY%dpmComScanRecordConsumer

See [http://rocketmq.apache.org/docs/faq/](http://rocketmq.apache.org/docs/faq/) for further details., org.apache.rocketmq.client.impl.MQClientAPIImpl.getTopicRouteInfoFromNameServer(MQClientAPIImpl.java:1212)]MessageTrack
```



# 根据key查询存储消息

```
bin/mqadmin queryMsgByKey -n 192.168.x.x:9876 -t SCANRECORD -k 1dd04932-b7d9-43ee-8b31-39d16dd043d4

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=128m; support was removed in 8.0

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=128m; support was removed in 8.0

Message ID                    QID                 Offset

C0A81F8166832F2C9B1953831616FB45           20                 35052080
```



# 根据offset查询储存消息

```
bin/mqadmin queryMsgByOffset -n 192.168.x.x:9876 -t SCANRECORD -b broker-a -i 20 -o 35052080

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=128m; support was removed in 8.0

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=128m; support was removed in 8.0

OffsetID:      C0A801B400002A9F000005F308F9B3A5

OffsetID:      C0A801B400002A9F000005F308F9B3A5

Topic:        SCANRECORD

Tags:        [2]

Keys:        [1dd04932-b7d9-43ee-8b31-39d16dd043d4]

Queue ID:      20

Queue Offset:    35052080

CommitLog Offset:  6541385773989

Reconsume Times:   0

Born Timestamp:   2018-10-17 05:11:39,798

Store Timestamp:   2018-10-17 05:11:39,179

Born Host:      192.168.31.129:29781

Store Host:     192.168.1.180:10911

System Flag:     0

Properties:     {MIN_OFFSET=35046369, MAX_OFFSET=40342481, KEYS=1dd04932-b7d9-43ee-8b31-39d16dd043d4, UNIQ_KEY=C0A81F8166832F2C9B1953831616FB45, WAIT=true, TAGS=2}

Message Body Path:  /tmp/rocketmq/msgbodys/C0A81F8166832F2C9B1953831616FB45

MessageTrack [consumerGroup=ZtoSignGroup, trackType=CONSUMED_BUT_FILTERED, exceptionDesc=null]MessageTrack [consumerGroup=newOpenPartnerDeadlineJob, trackType=CONSUMED_BUT_FILTERED, exceptionDesc=null]MessageTrack [consumerGroup=smartidivision-scanrecord-dis, trackType=CONSUMED_BUT_FILTERED, exceptionDesc=null]MessageTrack [consumerGroup=PanguRecordGroup, trackType=CONSUMED, exceptionDesc=null]MessageTrack [consumerGroup=code-send-consumer, trackType=CONSUMED_BUT_FILTERED, exceptionDesc=null]MessageTrack [consumerGroup=shenZhouOneSiteKpiConsumer, trackType=CONSUMED, exceptionDesc=null]MessageTrack [consumerGroup=SortComplementConsumer, trackType=CONSUMED_BUT_FILTERED, exceptionDesc=null]MessageTrack [consumerGroup=tf_wonder_waybill_center_scanrecord, trackType=CONSUMED_BUT_FILTERED, exceptionDesc=null]MessageTrack [consumerGroup=dpmComScanRecordConsumer, trackType=UNKNOWN, exceptionDesc=org.apache.rocketmq.client.exception.MQClientException: CODE: 17 DESC: No topic route info in name server for the topic: %RETRY%dpmComScanRecordConsumer

See [http://rocketmq.apache.org/docs/faq/](http://rocketmq.apache.org/docs/faq/) for further details., org.apache.rocketmq.client.impl.MQClientAPIImpl.getTopicRouteInfoFromNameServer(MQClientAPIImpl.java:1212)]MessageTrack [

"t", "topic", true, "topic name"

"b", "brokerName", true, "Broker Name"

"i", "queueId", true, "Queue Id"

"o", "offset", true, "Queue Offset"
```



# 通过UniqueKey查询消息内容

```
bin/mqadmin queryMsgByUniqueKey -n 192.168.x.x:9876 -t SCANRECORD -i C0A801B400002A9F000005F308F9B3A5

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=128m; support was removed in 8.0

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=128m; support was removed in 8.0

Topic:        SCANRECORD

Tags:        [2]

Keys:        [1dd04932-b7d9-43ee-8b31-39d16dd043d4]

Queue ID:      20

Queue Offset:    35052080

CommitLog Offset:  6541385773989

Reconsume Times:   0

Born Timestamp:   2018-10-17 05:11:39,798

Store Timestamp:   2018-10-17 05:11:39,179

Born Host:      192.168.31.129:29781

Store Host:     192.168.1.180:10911

System Flag:     0

Properties:     {KEYS=1dd04932-b7d9-43ee-8b31-39d16dd043d4, UNIQ_KEY=C0A81F8166832F2C9B1953831616FB45, WAIT=true, TAGS=2}

Message Body Path:  /tmp/rocketmq/msgbodys/C0A81F8166832F2C9B1953831616FB45

MessageTrack [consumerGroup=ZtoSignGroup, trackType=CONSUMED_BUT_FILTERED, exceptionDesc=null]MessageTrack [consumerGroup=newOpenPartnerDeadlineJob, trackType=CONSUMED_BUT_FILTERED, exceptionDesc=null]MessageTrack [consumerGroup=smartidivision-scanrecord-dis, trackType=CONSUMED_BUT_FILTERED, exceptionDesc=null]MessageTrack [consumerGroup=PanguRecordGroup, trackType=CONSUMED, exceptionDesc=null]MessageTrack [consumerGroup=code-send-consumer, trackType=CONSUMED_BUT_FILTERED, exceptionDesc=null]MessageTrack [consumerGroup=shenZhouOneSiteKpiConsumer, trackType=CONSUMED, exceptionDesc=null]MessageTrack [consumerGroup=SortComplementConsumer, trackType=CONSUMED_BUT_FILTERED, exceptionDesc=null]MessageTrack [consumerGroup=tf_wonder_waybill_center_scanrecord, trackType=CONSUMED_BUT_FILTERED, exceptionDesc=null]MessageTrack [consumerGroup=dpmComScanRecordConsumer, trackType=UNKNOWN, exceptionDesc=org.apache.rocketmq.client.exception.MQClientException: CODE: 17 DESC: No topic route info in name server for the topic: %RETRY%dpmComScanRecordConsumer

See [http://rocketmq.apache.org/docs/faq/](http://rocketmq.apache.org/docs/faq/) for further details., org.apache.rocketmq.client.impl.MQClientAPIImpl.getTopicRouteInfoFromNameServer(MQClientAPIImpl.java:1212)]MessageTrack 

"i", "msgId", true, "Message Id"

"g", "consumerGroup", true, "consumer group name"

"d", "clientId", true, "The consumer's client id"

"t", "topic", true, "The topic of msg"
```

