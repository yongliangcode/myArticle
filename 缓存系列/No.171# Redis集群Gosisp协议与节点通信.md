---
title: No.171# Redis集群Gosisp协议与节点通信
categories: 分布式缓存
tags: 分布式缓存
date: 2022-08-28 11:55:01
---





# 一、数据分片与分配算法



为了应对流量并发瓶颈，以及方便数据迁移与扩容，数据分片方式是常用的解决方式。



Kafka的分区（partition）、RocketMQ的队列（Queue）、Elasticsearch的主分片/副本（shard）、数据库的分库分表等，均采用数据分片思想应对高并发流量。



Redis的集群模式也不例外，采用虚拟槽slot实现数据分片。



Redis的槽位范围0~16383，共16384个槽位。



Redis Cluster中每个节点负责一部分槽数量，分配算法：slot=CRC16（key）&16383。



槽位分配与选择示意图如下：

![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/%E6%A7%BD%E5%88%86%E9%85%8D%E5%9B%BE%E7%A4%BA.png)



# 二、Gosisp协议类型与格式



### 1、Gosisp协议类型

节点通信使用Gosisp协议，消息类型有：ping消息、pong消息、meet消息、fail消息。

![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20220828134754.png)

* MEET消息：当新节点加入时握手使用。

* PING消息：节点之间周期性地发送ping消息、交换状态。

* PONG消息：收到meet、ping消息的响应、并封装自身状态消息。

* FAIL消息：当节点下线时，像集群广播一个fail消息，其他节点收到会更新该节点的状态。



通信端口=节点端口+10000



每个节点周期性的选择几个节点发送ping消息



### 2、消息头格式 

消息头的结构在clusterMsg中，具体属性如下：

| 字段                                   | 说明                                                         | 简述                               |
| -------------------------------------- | ------------------------------------------------------------ | ---------------------------------- |
| char sig[4]                            | Signature "RCmb" (Redis Cluster message bus).                | 信号签名                           |
| uint32_t totlen                        | Total length of this message                                 | 消息长度                           |
| uint16_t ver                           | Protocol version, currently set to 1                         | 协议版本                           |
| uint16_t port                          | TCP base port number                                         | 端口信息                           |
| uint16_t type                          | Message type                                                 | 消息类型，ping、meet、pong等       |
| uint16_t count                         | Only used for some kind of messages                          | 消息体包含的节点数量               |
| uint64_t currentEpoch                  | The epoch accordingly to the sending node                    | 发送节点的纪元（epoch）配置        |
| uint64_t configEpoch                   | The config epoch if it's a master, <br />or the last epoch advertised by its master if it is a slave | 主从节点中，主节点的纪元配置       |
| uint64_t offset                        | Master replication offset if node is a master <br />or processed replication offset if node is a slave | 复制偏移量                         |
| char sender[CLUSTER_NAMELEN]           | Name of the sender node                                      | 发送节点的nodeId信息               |
| unsigned char myslots[CLUSTER_SLOTS/8] | myslots info                                                 | 发送节点负责的槽位信息             |
| char slaveof[CLUSTER_NAMELEN]          |                                                              | 从节点的nodeId信息                 |
| char myip[NET_IP_STR_LEN]              | Sender IP, if not all zeroed                                 | 发送者IP                           |
| uint16_t extensions                    | Number of extensions sent along with this packet             | 扩展信息                           |
| char notused1[30]                      | 30 bytes reserved for future usage                           | 保留30个字节扩展供未来使用         |
| uint16_t pport                         | Sender TCP plaintext port, if base port is TLS               | 如果基础端口为TLS，TCP的明文端口   |
| uint16_t cport                         | Sender TCP cluster bus port                                  | 发送者TCP集群总线端口              |
| uint16_t flags                         | Sender node flags                                            | 发送节点标识，区分主从以及是否下线 |
| unsigned char state                    | Cluster state from the POV of the sender                     | 发送者角度的集群状态               |
| unsigned char mflags[3]                | Message flags: CLUSTERMSG_FLAG[012]_...                      | 消息标识                           |
| union clusterMsgData data              |                                                              | 消息体正文                         |



### 3、消息体格式 

消息体clusterMsgData结构如下：

```
union clusterMsgData {
    /* PING, MEET and PONG */
    struct {
        /* Array of N clusterMsgDataGossip structures */
        clusterMsgDataGossip gossip[1];
        /* Extension data that can optionally be sent for ping/meet/pong
         * messages. We can't explicitly define them here though, since
         * the gossip array isn't the real length of the gossip data. */
    } ping;

    /* FAIL */
    struct {
        clusterMsgDataFail about;
    } fail;

    /* PUBLISH */
    struct {
        clusterMsgDataPublish msg;
    } publish;

    /* UPDATE */
    struct {
        clusterMsgDataUpdate nodecfg;
    } update;

    /* MODULE */
    struct {
        clusterMsgModule msg;
    } module;
};
```



备注：clusterMsgDataGossip：PING, MEET and PONG采用的消息结构体，详细如下。

```
typedef struct {
    char nodename[CLUSTER_NAMELEN];
    uint32_t ping_sent;
    uint32_t pong_received;
    char ip[NET_IP_STR_LEN];  /* IP address last time it was seen */
    uint16_t port;              /* base port last time it was seen */
    uint16_t cport;             /* cluster port last time it was seen */
    uint16_t flags;             /* node->flags copy */
    uint16_t pport;             /* plaintext-port, when base port is TLS */
    uint16_t notused1;
} clusterMsgDataGossip;
```

* nodename：节点NodeId
* ping_sent：最后一次向该节点发送ping消息时间
* pong_received：最后一次接受该节点pong消息时间
* ip/port/cport/flags/pport：IP端口以及节点标识





# 三、节点选择与通信流程



### 1、节点通信流程

两个节点之间发送MEET/PING消息，回复PONG消息的流程如下。

![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/%E8%8A%82%E7%82%B9%E9%80%9A%E4%BF%A1%E6%B5%81%E7%A8%8B.png)





### 2、通信节点选择

Gosisp协议PING/PONG通信时，具体选择哪个节点发起通信？



每秒从本地实例列表选择5个节点，在这5个节点中选择最久没有通信的实例，向该实例发送PING消息。



避免一些实例节点一直选不到，会有一个定时任务扫描兜底措施。



集群内部每秒10次的固定频率扫描本地缓存节点列表，也就是每100ms一次。



如果节点：PONG更新时间>（cluster-node-timeout/2）立即向该节点发送PING消息。



cluster-node-timeout是判定实例故障的心跳超时时间，默认15秒。



![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/%E9%80%9A%E4%BF%A1%E8%8A%82%E7%82%B9%E9%80%89%E6%8B%A9.png)











