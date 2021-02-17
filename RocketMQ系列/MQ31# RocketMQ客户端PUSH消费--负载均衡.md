---
title: MQ31# RocketMQ客户端PUSH消费--负载均衡
categories: RocketMQ
tags: RocketMQ
date: 2020-12-20 20:52:01
---



# 问题思考

1.主题队列是如何分配的？

2.什么时候会进行负载均衡？

3.负载均衡后是否会导致消息重复消费？

<!--more-->

# 调用链条

**初始化链条**

```
@1 DefaultMQPushConsumerImpl#start

this.mQClientFactory

= MQClientManager.getInstance().getAndCreateMQClientInstance

@2 MQClientManager#getAndCreateMQClientInstance

instance = new MQClientInstance

@3 MQClientInstance#MQClientInstance

this.rebalanceService = new RebalanceService
```



**启动链条**

```
@1 DefaultMQPushConsumerImpl#start

mQClientFactory.start()

@2 MQClientInstance#start

this.rebalanceService.start
```

```
小结：从初始化链和调用链可以看出RebalanceService为线程类，随着消费启动时而启动，消费不退出则一直运行着。
```



# 负载均衡流程

**负载均衡链条**



```
@1 RebalanceService#run

mqClientFactory.doRebalance()

@2 MQClientInstance#doRebalance

impl.doRebalance()

@3 DefaultMQPushConsumerImpl#doRebalance

this.rebalanceImpl.doRebalance

@4 RebalanceImpl#doRebalance

rebalanceByTopic
```



**负载均衡流程**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219141402.png)

```
小结：在负载均衡时，会循环该消费组订阅的所有Topic都会执行负载均衡。
```



**更新缓存processQueue流程**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219141423.png)

```
小结：
1. 更新缓存时如果消费组订阅的队列不在新分配的队列集合中或者队列拉取时间超时失效，则将快照ProcessQueue设置为丢弃。
2. 消费拉取时判断ProcessQueue为丢弃，则不再对该队列拉取。
3. 顺序消费时如果获取消费锁成功，表明此队列空闲没有被消费，此时向Broker发起解锁请求，解锁成功后将该队列从缓存（processQueueTable）移除。
4. 顺序消费时获取锁失败，表明正在消费则不从processQueueTable移除，由于ProcessQueue设置为丢弃，在顺序消费下次拉取时会退出该队列的拉取请求。
```



<!--more-->



**向Broker发送心跳流程**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219141446.png)

**队列分配算法**

负载均衡流程图中对clientId和分区队列的分配提交给分区算法执行，那该算法是如何运作的呢？

接口AllocateMessageQueueStrategy队列分配策略提供五种分配算法实现：

\* 1.平均分配策略AllocateMessageQueueAveragely

\* 2.环形分配策略AllocateMessageQueueAveragelyByCircle

\* 3.机房分配策略AllocateMessageQueueByMachineRoom

\* 4.一致性Hash分配策略AllocateMessageQueueConsistentHash

\* 5.配置文件分配策略AllocateMessageQueueByConfig



除此之外可以自定义分配算法，实现接口接口即可，默认使用平均分配算法，也是最常用的，下面以该算法看看如何工作的。

```
public List<MessageQueue> allocate

(String consumerGroup, String currentCID, List<MessageQueue> mqAll,

List<String> cidAll) {

List<MessageQueue> result = new ArrayList<MessageQueue>();

int index = cidAll.indexOf(currentCID);

int mod = mqAll.size() % cidAll.size();

int averageSize = mqAll.size() <= cidAll.size() ?

1 : (mod > 0 && index < mod ?

mqAll.size() / cidAll.size() + 1 : mqAll.size() / cidAll.size());

int startIndex = (mod > 0 && index < mod) ? index * averageSize : index * averageSize + mod;

int range = Math.min(averageSize, mqAll.size() - startIndex);

for (int i = 0; i < range; i++) {

result.add(mqAll.get((startIndex + i) % mqAll.size()));

}

return result;

}
```



代码不是很好阅读，看下面验证结果即可。



**平均分配算法验证**

\* 只有一个clientId时分配情况

会把1个Broker的16个分区全部分配给该客户端，每隔20秒触发一次负载均衡。

currentCID=2.0.1.138@consumer01分到的队列为0～15

```
----------2019-08-04 22:10:15-----------

currentCID=2.0.1.138@consumer01

index=0

mod=0

averageSize=16

startIndex=0

range=16

result=[MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=0], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=1], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=2], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=3], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=4], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=5], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=6], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=7], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=8], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=9], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=10], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=11], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=12], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=13], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=14], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=15]]

----------2019-08-04 22:10:35-----------

currentCID=2.0.1.138@consumer01

index=0

mod=0

averageSize=16

startIndex=0

range=16

result=[MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=0], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=1], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=2], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=3], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=4], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=5], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=6], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=7], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=8], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=9], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=10], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=11], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=12], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=13], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=14], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=15]]
```



\* 新加入第二个client时

此时有两个clinetId分别为2.0.1.138@consumer01和2.0.1.138@consumer02，1个 Broker16个分区的分配情况。

currentCID=2.0.1.138@consumer01分到的分区为0～7

currentCID=2.0.1.138@consumer02分到的分区为8～16

```
----------2019-08-04 22:12:25-----------

currentCID=2.0.1.138@consumer01

index=0

mod=0

averageSize=8

startIndex=0

range=8

result=[MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=0], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=1], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=2], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=3], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=4], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=5], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=6], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=7]]

----------2019-08-04 22:12:45-----------

currentCID=2.0.1.138@consumer02

index=1

mod=0

averageSize=8

startIndex=8

range=8

result=[MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=8], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=9], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=10], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=11], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=12], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=13], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=14], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=15]]
```



\* 新加入第三个client时

此时有三个客户端2.0.1.138@consumer01、2.0.1.138@consumer02、2.0.1.138@consumer03，1个Broker的16个队列的分配情况。

currentCID=2.0.1.138@consumer01分到的队列0～5

currentCID=2.0.1.138@consumer02分到的队列6～10

currentCID=2.0.1.138@consumer03分到的队列11～15

```
----------2019-08-04 22:13:58-----------

currentCID=2.0.1.138@consumer01

index=0

mod=1

averageSize=6

startIndex=0

range=6

result=[MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=0], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=1], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=2], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=3], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=4], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=5]]

----------2019-08-04 22:14:18-----------

currentCID=2.0.1.138@consumer02

index=1

mod=1

averageSize=5

startIndex=6

range=5

result=[MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=6], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=7], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=8], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=9], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=10]]

----------2019-08-04 22:14:39-----------

currentCID=2.0.1.138@consumer03

index=2

mod=1

averageSize=5

startIndex=11

range=5

result=[MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=11], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=12], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=13], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=14], MessageQueue [topic=topic_test, brokerName=liangyongdeMacBook-Pro.local, queueId=15]]
```



# 四、总结

1.主题队列是如何分配的？

备注：见队列分配算法，通常使用平均分配算法。



2.什么时候会进行负载均衡？

备注：负载均衡线程每隔20秒执行一次，当有新客户端退出或者加入或者新的Broker加入或掉线都会触发重新负载均衡。



3.负载均衡后是否会导致消息重复消费？



备注：

并发消费可能导致消息被重复消费，看以下代码。如果负载均衡前已分配的队列不在负载均衡后的新队列集合中，会丢弃该队列即：processQueue.isDropped()。而该队列可能已经被消费完了，在处理结果时被丢弃了，消费进度没有更新。别的消费客户端重新拉取该队列时造成重复消费。

顺序消费不会导致消息被重复消费。

```
//并发消费对结果的处理

ConsumeMessageConcurrentlyService#ConsumeRequest

if (!processQueue.isDropped()) { ConsumeMessageConcurrentlyService.this.processConsumeResult(status, context, this);

} else { //被丢弃，消费进度不会更新

log.warn("processQueue is dropped without process consume result. messageQueue={}, msgs={}", messageQueue, msgs);

}

```



