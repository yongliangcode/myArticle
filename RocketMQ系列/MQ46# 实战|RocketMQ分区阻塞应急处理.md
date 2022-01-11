---
title: MQ46# 实战|RocketMQ分区阻塞应急处理
categories: RocketMQ
tags: RocketMQ
date: 2021-11-21 20:55:01
---



# 现象反馈



同事发现一个主题的某个分区卡主不再消费，如下图所示，通常这种情况是客户端消费线程阻塞造成的。而这次确不是，而且该现象还是头一次遇到，邪乎。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20211121212303.png)



<!--more-->

# 问题定位



**1.日志分析** 

在消费客户端发现了如下错误，显示着该消息不合法，超过了RocketMQ消息大小限制。

```
org.apache.rocketmq.client.exception.MQBrokerException: CODE: 13  DESC: the message is illegal, maybe msg body or properties length not matched. msg body length limit 128k, msg properties length limit 32k.
For more information, please visit the url, http://rocketmq.apache.org/docs/faq/
        at org.apache.rocketmq.client.impl.MQClientAPIImpl.processSendResponse(MQClientAPIImpl.java:711) ~[rocketmq-client-4.7.0.jar:4.7.0]
        at org.apache.rocketmq.client.impl.MQClientAPIImpl.sendMessageSync(MQClientAPIImpl.java:505) ~[rocketmq-client-4.7.0.jar:4.7.0]
        at org.apache.rocketmq.client.impl.MQClientAPIImpl.sendMessage(MQClientAPIImpl.java:487) ~[rocketmq-client-4.7.0.jar:4.7.0]
        at org.apache.rocketmq.client.impl.MQClientAPIImpl.sendMessage(MQClientAPIImpl.java:431) ~[rocketmq-client-4.7.0.jar:4.7.0]
    
```



**2.源码跟踪** 

跟踪RocketMQ报错的地方如下：

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

跟下去看到是消息属性过大造成的。

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



**3.消息确认** 

既然消息不合法，那就捞出来看看它长啥样，通过检索命令查看：

```
bin/mqadmin  queryMsgByOffset -n x.x.x.x:9876 -t delay_notify_level_04_topic -b latency_mq_a2 -i 1 -o 62565613
```

**消息内容** 

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
11.1.3.1.1.3.7.41.9.5.1.1.11.1.27.1.1.5.3.4.2.2.2.20.9.1.7.1.1.15.3.1.7.1.1.1.1.1.5.7.1.1.1.1.1.1.1.3.1.3.1.3.1.1.3.1.1.1.3.1.1.1.1.3.1.1.1.1.3.5, UNIQ_KEY=0A484BC80001764C12B62932E6B9247C, WAIT=true, TAGS=delayStrategy}
Message Body Path:   /tmp/rocketmq/msgbodys/0A484BC80001764C12B62932E6B9247C

```



备注：消费体内容存储在/tmp/rocketmq/msgbodys/0A484BC80001764C12B62932E6B9247C，内容小于1KB，发现其Properties部分有个rpcId通过测算其长度长达33KB，问题就出在这里，RocketMQ最长为32KB。



**4.阻塞根因** 

* 使用场景为消费服务从某主题消费消息进行业务处理后再发送到其他主题
* 消费服务在获取到这条大Properties消息后使用自定义封装的消息SDK发送
* 自定义的SDK会在Properties增加一些属性，由于超过32KB导致该消息一直发不成功，从而造成阻塞



# 解决方案

解决思路有两个：

* 一个是消费服务让拿到消息后不要再发送了，把位点提交了就好了
* 另外一个是通过在broker中手动修改消费组的消费位点consumerOffset.json，把这条大消息跳过去

第一种需要修改SDK代码，测试升级周期较长，为快速解决该问题选择了第二种解决方式。

将consumerOffset.json中的62565631修改为62565632，从而跳过问题消息。需要注意的是修改时需要把消费者下线并关闭broker，否则修改不成功，该位点会从消费服务的缓存中上报。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20211110210858.png)

还有就是追踪该巨无霸rpcId产生的不合理使用场景追踪，从而彻底规避该问题的产生。





