---
title: No.172# Redis集群模式通信成本影响因素
categories: 分布式缓存
tags: 分布式缓存
date: 2022-08-28 11:55:01
---



# 引言



Redis集群模式节点之间通过Gossip协议彼此之间传递节点状态、槽信息等，没有依赖元数据组件来维护。简化了架构的同时，通信成本也限制了集群规模。



本文除了走查了通信开销影响因素，还梳理了节点扩缩容和请求路由的原理。主要内容有：



一、通信开销影响因素

二、扩缩容与槽位迁移

三、请求路由与重定向



# 一、通信开销影响因素



### 节点数量



每秒从本地实例列表选择5个节点，在这5个节点中选择最久没有通信的实例，向该实例发送PING消息。



即：定时发送PING消息的节点数量=5。



避免一些实例节点一直选不到，会有一个定时任务扫描兜底措施。



集群内部每秒10次的固定频率扫描本地缓存节点列表，也就是每100ms一次。



如果节点：PONG更新时间node.pong_received>（cluster-node-timeout/2）立即向该节点发送PING消息，假设该数量为N。



即：兜底发送的节点数量=10 * N。



通过调大cluster_node_timeout可以减少通信的节点数量，例如：从15秒调整到30秒。



但是，cluster_node_timeout过大会影响故障发现的时间和新节点发现的时间。



### 消息大小



一次通信包含消息头和消息体。



消息头：PING消息头相对固定，主要占用的发送节点负责的槽位（myslots[CLUSTER_SLOTS/8]）占用2KB。



消息体：会携带一定数量的其他节点信息，默认包含集群总节点数的1/10，最少包含集群的3个节点，最多包含集群总节点数-2。



消息体clusterMsgDataGossip各个字段字节大小，共计104个字节。

| 属性                           | 大小     |
| ------------------------------ | -------- |
| char nodename[CLUSTER_NAMELEN] | 40字节   |
| uint32_t ping_sent             | 4字节    |
| uint32_t pong_received         | 4字节    |
| char ip[NET_IP_STR_LEN]        | 46字节   |
| uint16_t port                  | 2字节    |
| uint16_t cport                 | 2字节    |
| uint16_t flags                 | 2字节    |
| uint16_t pport                 | 保留字段 |
| uint16_t notused1              | 4字节    |
| 合计                           | 104字节  |



200个节点的redis集群，一次通信成本：2KB的消息头+2KB的消息体（20*104）= 4KB，一来一回8KB。



携带消息体的大小与集群规模相关，规模越大消息体越大，通信成本越高。



达到一定程度后整体集群性能会下降，Redis Cluster官方建议最大规模1000个实例，实际中通常不会超过500个实例。





# 二、扩缩容与槽位迁移



节点扩缩容本质上是槽在节点之间的迁移。



节点扩容后，需要将原有节点上的槽迁移到新节点。



如下图所示，当集群中加入节点4时，将节点1的Slo01，节点2的Slot04，节点3的Slot07迁移给节点4以实现数据均衡。

![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/%E8%8A%82%E7%82%B9%E6%89%A9%E5%AE%B9.png)

节点缩容前，需要将待下线节点上的槽先迁移走。



如下图所示，当集群中节点4下线，需要先将其拥有的槽位Slot01、Slot04、Slot07迁移走。





![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/%E8%8A%82%E7%82%B9%E7%BC%A9%E5%AE%B9.png)



槽位迁移命令有：ADDSLOTS、DELSLOTS、FLUSHSLOTS、SETSLOT。







# 三、请求路由与重定向



数据存储在槽里，槽分布在实例上，处理客户端请求也是找对应槽的过程。



### 请求重定向

请求路由过程如下：

* @1 客户端发送请求命令到集群任意节点

* @2 计算key对应的槽，计算公式：slot=CRC16（key）&16383

* @3 槽在本节点，执行命令，每个实例维护自身负责的槽也维护其他实例负责的槽位
* @4 槽不在本节点，回复MOVE到其他节信息点
* @5 向目标节点发起请求



![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/%E8%AF%B7%E6%B1%82%E8%B7%AF%E7%94%B1%E8%BF%87%E7%A8%8B.png)

为了减少MOVE重定向的开销，例如Jedis在客户端实现时缓存了槽与节点的关系，减少通信的开销。



然而也增加了客户端的复杂性，客户端会为集群中每个节点独立的连接池，集群规模大时占用更多的本地缓存。



### ASK重定向



如果访问的槽正在做迁移，一部分数据在源节点，而另一部分已经迁移到目标节点，这个流程是如何的？



ASK重定向流程：

* @1 发送请求命令
* @2 计算key对应的槽
* @3 槽在本节点，数据也在，执行命令
* @4 访问的数据正在迁移，回复ASK信息含请求数据的目标节点
* @5 向目标节点发起ASKING请求、执行命令获取数据



![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/ASK%E9%87%8D%E5%AE%9A%E5%90%91%E6%B5%81%E7%A8%8B.png)



ASK的重定向是临时性的，客户端（Jedis）收到回复不更新客户端槽与节点映射，而MOVE的重定向会更新本地槽映射关系。









