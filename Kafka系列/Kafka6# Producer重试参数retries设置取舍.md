---
title: Kafka6# Producer重试参数retries设置取舍
categories: Kafka
tags: Kafka
date: 2019-05-20 11:55:01
---



# retries参数说明



参数的设置通常是一种取舍，看下retries参数在版本0.11.3说明：

```
Setting a value greater than zero will cause the client to resend

any record whose send fails with a potentially transient error.

Note that this retry is no different than if the client resent the

record upon receiving the error.

Allowing retries without setting max.in.flight.requests.per.connection to 1 will potentially change

the ordering of records because if two batches are sent to a single

partition, and the first fails and is retried but the second succeeds,

then the records in the second batch may appear first.
```



备注：当发送失败时客户端会进行重试，重试的次数由retries指定，此参数默认设置为0。即：快速失败模式，当发送失败时由客户端来处理后续是否要进行继续发送。如果设置retries大于0而没有设置max.in.flight.requests.per.connection=1则意味着放弃发送消息的顺序性。



<!--more-->



# retries使用建议



使用retries的默认值交给使用方自己去控制，结果往往是不处理。所以通用设置建议设置如下：

```
retries = Integer.MAX_VALUE

max.in.flight.requests.per.connection = 1
```



备注：这样设置后，发送客户端会一直进行重试直到broker返回ack；同时只有一个连接向broker发送数据保证了数据的顺序性。在Leader选举、集群中一个broker挂掉时，发送端会一直重试直到Leader选举结束。避免由于客户端对异常未处理造成的数据丢失，例如：遇到类似“This server is not the leader for that topic-partition”会自动恢复。



# retries后续发展



该参数的设置已经在kafka 2.4版本中默认设置为Integer.MAX_VALUE；同时增加了delivery.timeout.ms的参数设置。

```
The default value for the producer's retries config was changed to

Integer.MAX_VALUE, as we introduced delivery.timeout.ms in KIP-91,

which sets an upper bound on the total time between sending a

record and receiving acknowledgement from the broker.

By default, the delivery timeout is set to 2 minutes.

KIP-91: https://cwiki.apache.org/confluence/display/KAFKA/KIP-91+Provide+Intuitive+User+Timeouts+in+The+Producer
```

