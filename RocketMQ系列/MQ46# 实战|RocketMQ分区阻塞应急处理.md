---
title: MQ46# 实战|RocketMQ分区阻塞应急处理
categories: RocketMQ
tags: RocketMQ
date: 2021-08-15 20:55:01
---



# 现象反馈









# 现象反馈



```
org.apache.rocketmq.client.exception.MQBrokerException: CODE: 13  DESC: the message is illegal, maybe msg body or properties length not matched. msg body length limit 128k, msg properties length limit 32k.
For more information, please visit the url, http://rocketmq.apache.org/docs/faq/
        at org.apache.rocketmq.client.impl.MQClientAPIImpl.processSendResponse(MQClientAPIImpl.java:711) ~[rocketmq-client-4.7.0.jar:4.7.0]
        at org.apache.rocketmq.client.impl.MQClientAPIImpl.sendMessageSync(MQClientAPIImpl.java:505) ~[rocketmq-client-4.7.0.jar:4.7.0]
        at org.apache.rocketmq.client.impl.MQClientAPIImpl.sendMessage(MQClientAPIImpl.java:487) ~[rocketmq-client-4.7.0.jar:4.7.0]
        at org.apache.rocketmq.client.impl.MQClientAPIImpl.sendMessage(MQClientAPIImpl.java:431) ~[rocketmq-client-4.7.0.jar:4.7.0]
    
```



```java
handlePutMessageResult{

	switch (putMessageResult.getPutMessageStatus()) {
	
		case PUT_OK:
      sendOK = true;
      response.setCode(ResponseCode.SUCCESS);
      break;
    // ...
      case MESSAGE_ILLEGAL:
            case PROPERTIES_SIZE_EXCEEDED:
                response.setCode(ResponseCode.MESSAGE_ILLEGAL);
                response.setRemark(
                    "the message is illegal, maybe msg body or properties length not matched. msg body length limit 128k, msg properties length limit 32k.");
                break;
	}
}
```



```java
private PutMessageStatus checkMessage(MessageExtBrokerInner msg) {
  if (msg.getTopic().length() > Byte.MAX_VALUE) {
    log.warn("putMessage message topic length too long " + msg.getTopic().length());
    return PutMessageStatus.MESSAGE_ILLEGAL;
  }

  if (msg.getPropertiesString() != null && msg.getPropertiesString().length() > Short.MAX_VALUE) {
    log.warn("putMessage message properties length too long " + msg.getPropertiesString().length());
    return PutMessageStatus.MESSAGE_ILLEGAL;
  }
  return PutMessageStatus.PUT_OK;
}
```



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20211110210858.png)





![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20211110210957.png)



```
bin/mqadmin  queryMsgByOffset -n x.x.x.x:9876 -t delay_notify_level_04_topic -b latency_mq_a2 -i 1 -o 62565613
```



```
RocketMQLog:WARN No appenders could be found for logger (io.netty.util.internal.PlatformDependent0).
RocketMQLog:WARN Please initialize the logger system properly.
OffsetID:            0A6F1AEE00002A9F0000070BD4CE3505
Topic:               delay_notify_level_04_topic
Tags:                [delayStrategy]
Keys:                [2559003397109317634]
Queue ID:            1
Queue Offset:        62565613
CommitLog Offset:    7747396318469
Reconsume Times:     0
Born Timestamp:      2021-11-09 00:00:01,721
Store Timestamp:     2021-11-09 00:00:01,725
Born Host:           x.x.x.x:40046
Store Host:          x.x.x.x:10911
System Flag:         0
Properties:          {uber-trace-id=30863ccffc4785f65fcd844b53882621%3Aba4f3c364e11d4c2%3A75c3cab020852919%3A0, uberctx-us_app=AppRcpOperatingService, clientAppId=AppRcpOperatingService, reqId=359a420f5e0644c4a24ce653fc1003b7, MIN_OFFSET=62046858, MAX_OFFSET=62576938, KEYS=2559003397109317634, uberctx-us=asyncSend%3Arcp_alert_delay_topic, rpcId=1.1.1.3.1.1.1.1.1.1.1.1.1.1.1.1.3.1.1.1.1.1.1.1.1.1.1.1.1.1.1.1.1.1.1.1.3.1.1.1.1.1.1.1.1.1.1.1.1.3.1.1.1.1.1.1.1.1.1.1.1.1.1.1.1.1.1.1.1.1.3.1.1.1.1.1.1.1.1.1.1.1.1.1.1.1.1.3.1.1.1.1.1.1.1.1.1.1.1.1.1.1.1.1.1.1.1.1.1.
//.......
11.1.3.1.1.3.7.41.9.5.1.1.11.1.27.1.1.5.3.4.2.2.2.20.9.1.7.1.1.15.3.1.7.1.1.1.1.1.5.7.1.1.1.1.1.1.1.3.1.3.1.3.1.1.3.1.1.1.3.1.1.1.1.3.1.1.1.1.3.5, UNIQ_KEY=0A484BC80001764C12B62932E6B9247C, WAIT=true, TAGS=delayStrategy, uberctx-us_type=hms}
Message Body Path:   /tmp/rocketmq/msgbodys/0A484BC80001764C12B62932E6B9247C

```



消费体内容存储在/tmp/rocketmq/msgbodys/0A484BC80001764C12B62932E6B9247C，内容小于1KB。





