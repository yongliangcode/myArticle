---
title: Kafka5# 常用JMX监控指标整理
categories: Kafka
tags: Kafka
date: 2019-05-19 11:55:01
---



# 目录

```
系统相关指标
GC相关指标
JVM相关指标
Topic相关指标
Broker相关指标
```



# 系统相关指标

**系统信息收**

java.lang:type=OperatingSystem

```
{"freePhysicalMemorySize":"806023168","maxFileDescriptorCount":"4096","openFileDescriptorCount":"283","processCpuLoad":"0.0017562901839817224","systemCpuLoad":"0.014336627412954635","systemLoadAverage":"0.37"}
```



**Thread信息收集**

java.lang:type=Threading

```
{"peakThreadCount":"88","threadCount":"74"}
```



**获取mmaped和direct空间**

通过BufferPoolMXBean获取used、capacity、count



<!--more-->



# GC相关指标

**Young GC**

java.lang:type=GarbageCollector,name=G1 Young Generation

```
{"collectionCount":"534","collectionTime":"8258"}
```



**Old GC**

java.lang:type=GarbageCollector,name=G1 Old Generation

```
{"collectionCount":"0","collectionTime":"0"}
```



# JVM相关指标

通过MemoryMXBean获取JVM相关信息HeapMemoryUsage和NonHeapMemoryUsage；通过MemoryPoolMXBean获取其他JVM内存空间指标，例如：Metaspace、Codespace等



# Topic相关指标

**Topic消息入站速率（Byte）**

kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec,topic=" + topic

```
{"count":"0","fifteenMinuteRate":"0.0","fiveMinuteRate":"0.0","meanRate":"0.0","oneMinuteRate":"0.0"}
```



**Topic消息出站速率（Byte）**

kafka.server:type=BrokerTopicMetrics,name=BytesOutPerSec,topic=" + topic

```
{"count":"0","fifteenMinuteRate":"0.0","fiveMinuteRate":"0.0","meanRate":"0.0","oneMinuteRate":"0.0"}
```



**Topic请求被拒速率**

kafka.server:type=BrokerTopicMetrics,name=BytesRejectedPerSec,topic=" + topic

```
{"count":"0","fifteenMinuteRate":"0.0","fiveMinuteRate":"0.0","meanRate":"0.0","oneMinuteRate":"0.0"}
```



**Topic失败拉去请求速**

kafka.server:type=BrokerTopicMetrics,name=FailedFetchRequestsPerSec,topic=" + topic;

```
{"count":"0","fifteenMinuteRate":"0.0","fiveMinuteRate":"0.0","meanRate":"0.0","oneMinuteRate":"0.0"}
```



**Topic发送请求失败速**

kafka.server:type=BrokerTopicMetrics,name=FailedProduceRequestsPerSec,topic=" + topic

```
{"count":"0","fifteenMinuteRate":"0.0","fiveMinuteRate":"0.0","meanRate":"0.0","oneMinuteRate":"0.0"}
```



**Topic消息入站速率**

kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec,topic=" + topic

```
{"count":"0","fifteenMinuteRate":"0.0","fiveMinuteRate":"0.0","meanRate":"0.0","oneMinuteRate":"0.0"}
```



# Broker相关指标

**Log flush rate and time**

kafka.log:type=LogFlushStats,name=LogFlushRateAndTimeMs

```
{"50thPercentile":"1.074103","75thPercentile":"1.669793","95thPercentile":"6.846556","98thPercentile":"6.846556","999thPercentile":"6.846556","99thPercentile":"6.846556","count":"19","max":"6.846556","mean":"1.628646052631579","min":"0.512879","stdDev":"1.6007003364105892"}
```



**同步失效的副本数**

kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions

```
{"value":"0"}
```



**消息入站速率（消息数）**

kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec

```
{"count":"86845","fifteenMinuteRate":"0.6456600497006455","fiveMinuteRate":"0.6444164288097876","meanRate":"0.5314899330400695","oneMinuteRate":"0.6494649408329609"}
```



**消息入站速率（Byte）**

kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec

```
{"count":"57302357","fifteenMinuteRate":"379.11342092748146","fiveMinuteRate":"371.8482236385939","meanRate":"351.37122686037435","oneMinuteRate":"351.8348952308101"}
```



**消息出站速率（Byte）**

kafka.server:type=BrokerTopicMetrics,name=BytesOutPerSec

```
{"count":"246","fifteenMinuteRate":"4.508738367219028E-34","fiveMinuteRate":"1.4721921790135324E-98","meanRate":"0.0015031168286836175","oneMinuteRate":"2.964393875E-314"}
```



**请求被拒速率** 

kafka.server:type=BrokerTopicMetrics,name=BytesRejectedPerSec

```
{"count":"0","fifteenMinuteRate":"0.0","fiveMinuteRate":"0.0","meanRate":"0.0","oneMinuteRate":"0.0"}
```



**失败拉去请求速率**

kafka.server:type=BrokerTopicMetrics,name=FailedFetchRequestsPerSec

```
{"count":"0","fifteenMinuteRate":"0.0","fiveMinuteRate":"0.0","meanRate":"0.0","oneMinuteRate":"0.0"}
```



**发送请求失败速率**

kafka.server:type=BrokerTopicMetrics,name=FailedProduceRequestsPerSec

```
{"count":"0","fifteenMinuteRate":"0.0","fiveMinuteRate":"0.0","meanRate":"0.0","oneMinuteRate":"0.0"}
```



**Leader副本数**

kafka.server:type=ReplicaManager,name=LeaderCount

```
{"value":"92"}
```



**Partition数量**

kafka.server:type=ReplicaManager,name=PartitionCount

```
{"value":"135"}
```



**下线Partition数量**

kafka.controller:type=KafkaController,name=OfflinePartitionsCount

```
{"value":"0"}
```



**Broker网络处理线程**

kafka.server:type=KafkaRequestHandlerPool,name=RequestHandlerAvgIdlePercent

```
{"count":"164506926671008","fifteenMinuteRate":"0.9999327359820058","fiveMinuteRate":"1.0000290054537715","meanRate":"0.9998854371393514","oneMinuteRate":"1.0007836499581673"}
```



**Leader选举比率**

kafka.controller:type=ControllerStats,name=LeaderElectionRateAndTimeMs

```
{"count":"7","fifteenMinuteRate":"5.134993718576819E-82","fiveMinuteRate":"6.882658450509451E-240","meanRate":"4.2525243043608314E-5","oneMinuteRate":"2.964393875E-314"}
```



**Unclean Leader选举比率**

kafka.controller:type=ControllerStats,name=UncleanLeaderElectionsPerSec

```
{"count":"0","fifteenMinuteRate":"0.0","fiveMinuteRate":"0.0","meanRate":"0.0","oneMinuteRate":"0.0"}
```



**Controller存活**

kafka.controller:type=KafkaController,name=ActiveControllerCount

```
{"value":"1"}
```



**请求速率**

kafka.network:type=RequestMetrics,name=RequestsPerSec,request=Produce

```
{"count":"83233","fifteenMinuteRate":"0.6303485369828705","fiveMinuteRate":"0.6357199085092445","meanRate":"0.5046486472186744","oneMinuteRate":"0.6563203475530601"}
```



**Consumer拉取速率**

kafka.network:type=RequestMetrics,name=RequestsPerSec,request=FetchConsumer

```
{"count":"125796","fifteenMinuteRate":"1.14193044007404E-33","fiveMinuteRate":"7.699516480260211E-100","meanRate":"0.7623419964866819","oneMinuteRate":"2.964393875E-314"}
```



**Follower拉去速率**

kafka.network:type=RequestMetrics,name=RequestsPerSec,request=FetchFollower

```
{"count":"375108","fifteenMinuteRate":"2.302746562040189","fiveMinuteRate":"2.292459728166488","meanRate":"2.2721808581484693","oneMinuteRate":"2.2814260196672973"}
```



**Request total time**

kafka.network:type=RequestMetrics,name=TotalTimeMs,request=Produce

```
{"50thPercentile":"1.0","75thPercentile":"1.0","95thPercentile":"2.0","98thPercentile":"2.0","999thPercentile":"28.0","99thPercentile":"4.0","count":"83384","max":"48.0","mean":"1.2344934279957787","min":"0.0","stdDev":"1.1783192073287214"}
```



**Consumer fetch total time**

kafka.network:type=RequestMetrics,name=TotalTimeMs,request=FetchConsumer

```
{"50thPercentile":"500.0","75thPercentile":"501.0","95thPercentile":"501.0","98thPercentile":"501.0","999thPercentile":"501.971","99thPercentile":"501.0","count":"125796","max":"535.0","mean":"499.83123469744663","min":"0.0","stdDev":"17.138716708632025"}
```



**Follower fetch total time**

kafka.network:type=RequestMetrics,name=TotalTimeMs,request=FetchFollower

```
{"50thPercentile":"500.0","75thPercentile":"500.0","95thPercentile":"501.0","98thPercentile":"501.0","999thPercentile":"507.826","99thPercentile":"501.0","count":"375564","max":"532.0","mean":"437.79763502359117","min":"0.0","stdDev":"148.25999023472986"}
```



**Time the follower fetch request waits in the request queue**

kafka.network:type=RequestMetrics,name=RequestQueueTimeMs,request=FetchFollo

```
{"50thPercentile":"0.0","75thPercentile":"0.0","95thPercentile":"0.0","98thPercentile":"0.0","999thPercentile":"0.0","99thPercentile":"0.0","count":"376206","max":"28.0","mean":"0.0010260336092459982","min":"0.0","stdDev":"0.1282889653905258"}
```



**Time the Consumer fetch request waits in the request queue**

kafka.network:type=RequestMetrics,name=RequestQueueTimeMs,request=FetchConsumer

```
{"50thPercentile":"0.0","75thPercentile":"0.0","95thPercentile":"0.0","98thPercentile":"0.0","999thPercentile":"0.0","99thPercentile":"0.0","count":"125796","max":"24.0","mean":"0.0018124582657636174","min":"0.0","stdDev":"0.18122860552537737"}
```



**Time the Produce fetch request waits in the request queue**

kafka.network:type=RequestMetrics,name=RequestQueueTimeMs,request=Produce

```
{"50thPercentile":"0.0","75thPercentile":"0.0","95thPercentile":"0.0","98thPercentile":"0.0","999thPercentile":"0.0","99thPercentile":"0.0","count":"83704","max":"12.0","mean":"2.6283092803211315E-4","min":"0.0","stdDev":"0.042892540270754634"}
```



**Broker I/O工作处理线程空闲率**

kafka.network:type=SocketServer,name=NetworkProcessorAvgIdlePercent

```
{"value":"1.0015540075894207"}
```



**ISR变化速率**

kafka.server:type=ReplicaManager,name=IsrShrinksPerSec

```
{"count":"0","fifteenMinuteRate":"0.0","fiveMinuteRate":"0.0","meanRate":"0.0","oneMinuteRate":"0.0"}
```





