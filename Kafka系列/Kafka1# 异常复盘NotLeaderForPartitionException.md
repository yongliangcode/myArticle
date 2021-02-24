---
title: Kafka1# 异常复盘NotLeaderForPartitionException.md
categories: Kafka
tags: Kafka
date: 2019-05-15 11:55:01
---



#  情况分析

**客户端异常报警** 

晚上10点20分接到使用方电话，日志持续报以下异常，持续时间已有10多分钟。

```
ERROR 2019-05-15 23:05:23,221 [kafka-producer-network-thread | producer-1] A failure occurred sending a message to Kafka.
org.apache.kafka.common.errors.NotLeaderForPartitionException: This server is not the leader for that topic-partition.
WARN 2019-05-15 23:05:23,237 [main] Error sending event to listener kafkahandler, status: ABEND, event: Commit transaction
```



<!--more-->



**集群状况查看**

监控及Kafka Manager显示节点数量正常。问题集群有6个节点，3个副本。查看日志发现，都在大量输出选举的日志，日志中暂时没有发现明显的ERROR和FATAL日志。类似内容如下：

```
[2019-05-15 22:03:49,511] INFO Rolled new log segment for 'order-KSTREAM-AGGREGATE-STATE-STORE-0000000002-repartition-196' in 1 ms. (kafka.log.Log)
[2019-05-15 22:20:13,880] INFO Updated PartitionLeaderEpoch. New: {epoch:1, offset:5910507}, Current: {epoch:0, offset4932329} for Partition: zto_big_marks_mq-46. Cache now contains 1 entries. (kafka.server.epoch.LeaderEpochFileCache)
[2019-05-15 22:20:13,881] INFO Updated PartitionLeaderEpoch. New: {epoch:78, offset:1070147099}, Current: {epoch:77, offset1041132585} for Partition: zto_scan_rec-11. Cache now contains 1 entries. 
```



**GC日志查看**

通过查看各个节点GC日志发现，其中一台节点GC回收异常，堆内存回后，只能回收600来M的空间。应该可以初步确定为问题节点。

日志内容如下：

```
2019-05-15T23:51:14.885+0800: 21994518.023: [GC pause (G1 Evacuation Pause) (young), 0.0749305 secs]
   [Parallel Time: 70.4 ms, GC Workers: 33]
      [GC Worker Start (ms): Min: 21994518023.0, Avg: 21994518023.5, Max: 21994518023.9, Diff: 0.9]
      [Ext Root Scanning (ms): Min: 14.9, Avg: 15.7, Max: 17.1, Diff: 2.2, Sum: 518.0]
      [Update RS (ms): Min: 3.4, Avg: 5.0, Max: 7.8, Diff: 4.5, Sum: 166.5]
         [Processed Buffers: Min: 1, Avg: 1.7, Max: 7, Diff: 6, Sum: 57]
      [Scan RS (ms): Min: 0.0, Avg: 2.1, Max: 3.7, Diff: 3.7, Sum: 69.4]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Object Copy (ms): Min: 45.9, Avg: 46.6, Max: 47.2, Diff: 1.3, Sum: 1536.2]
      [Termination (ms): Min: 0.0, Avg: 0.1, Max: 0.1, Diff: 0.1, Sum: 3.3]
         [Termination Attempts: Min: 1, Avg: 38.3, Max: 52, Diff: 51, Sum: 1263]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.1, Max: 0.2, Diff: 0.2, Sum: 2.8]
      [GC Worker Total (ms): Min: 69.1, Avg: 69.6, Max: 70.1, Diff: 1.0, Sum: 2296.1]
      [GC Worker End (ms): Min: 21994518093.0, Avg: 21994518093.1, Max: 21994518093.2, Diff: 0.2]
   [Code Root Fixup: 0.1 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 1.0 ms]
   [Other: 3.4 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 1.5 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.9 ms]
      [Humongous Register: 0.0 ms]
      [Humongous Reclaim: 0.1 ms]
      [Free CSet: 0.5 ms]
   [Eden: 380.0M(380.0M)->0.0B(356.0M) Survivors: 28.0M->52.0M Heap: 5649.6M(8192.0M)->5466.5M(8192.0M)]
 [Times: user=2.31 sys=0.00, real=0.07 secs] 
2019-05-15T23:51:44.433+0800: 21994547.571: [GC pause (G1 Evacuation Pause) (mixed), 0.1036003 secs]
   [Parallel Time: 81.6 ms, GC Workers: 33]
      [GC Worker Start (ms): Min: 21994547572.2, Avg: 21994547573.2, Max: 21994547574.3, Diff: 2.1]
      [Ext Root Scanning (ms): Min: 20.8, Avg: 24.4, Max: 80.8, Diff: 60.1, Sum: 805.5]
      [Update RS (ms): Min: 0.0, Avg: 2.0, Max: 3.4, Diff: 3.4, Sum: 65.7]
         [Processed Buffers: Min: 0, Avg: 2.0, Max: 5, Diff: 5, Sum: 65]
      [Scan RS (ms): Min: 0.0, Avg: 10.1, Max: 10.9, Diff: 10.9, Sum: 334.2]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Object Copy (ms): Min: 0.0, Avg: 33.4, Max: 35.2, Diff: 35.2, Sum: 1102.0]
      [Termination (ms): Min: 0.0, Avg: 10.3, Max: 10.7, Diff: 10.7, Sum: 338.8]
         [Termination Attempts: Min: 1, Avg: 26.2, Max: 38, Diff: 37, Sum: 865]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.1, Max: 0.4, Diff: 0.3, Sum: 2.8]
      [GC Worker Total (ms): Min: 79.2, Avg: 80.3, Max: 81.2, Diff: 2.1, Sum: 2649.0]
      [GC Worker End (ms): Min: 21994547653.4, Avg: 21994547653.5, Max: 21994547653.5, Diff: 0.1]
   [Code Root Fixup: 0.1 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 1.7 ms]
   [Other: 20.3 ms]
      [Choose CSet: 0.2 ms]
      [Ref Proc: 16.5 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.8 ms]
      [Humongous Register: 0.2 ms]
      [Humongous Reclaim: 0.0 ms]
      [Free CSet: 1.9 ms]
   [Eden: 356.0M(356.0M)->0.0B(364.0M) Survivors: 52.0M->44.0M Heap: 5822.5M(8192.0M)->5240.1M(8192.0M)]
 [Times: user=2.69 sys=0.00, real=0.10 secs] 
```



# 故障恢复

**问题节点下线** 

确定为问题节点后，先将该节点下线。由于三个副本，所以对可用性不能造成影响。在该节点下线后，客户端报警解除，恢复正常。



**问题节点恢复**

问题节点为正常下线（kill -s TERM）, 当时在恢复过程中出现卡顿，其中一个分区卡顿长达20多分钟没有继续往下走。

尝试将该问题分区移除日志目录，重新启动，该topic又出现了其他问题的卡顿。索性将该topic的所有分区全部移除日志目录。但是依然有其他topic的分区卡顿不前，在反复近10次重启后，依然没有恢复该节点。最后使用空目录启动，节点恢复。



# 问题复盘

**有几个问题需要澄清**

```
1.配置了GC回收短信告警，而告警没有发出

2.什么原因导致频繁的Leader选举

3.GC为何不能正常回收

4.为什么正常关机而重新启动后不能正常恢复

故障后的第二天，需要对以上问题作出分析。
```



**查看问题节点server日志** 

发现Unable to reconnect to ZooKeeper service出现频率非常高，没隔1到2分钟就会打印一次。

```
[2019-05-15 22:20:14,397] INFO Opening socket connection to server 192.168.x.x/192.168.x.x2181. Will not attempt to authenticate using SASL (unknown error) (org.apache.zookeeper.ClientCnxn)
[2019-05-15 22:20:14,398] INFO Socket connection established to 192.168.x.x/192.168.x.x2181, initiating session (org.apache.zookeeper.ClientCnxn)
[2019-05-15 22:20:14,399] WARN Unable to reconnect to ZooKeeper service, session 0x167772767ad92fd has expired (org.apache.zookeeper.ClientCnxn)
[2019-05-15 22:20:14,399] INFO zookeeper state changed (Expired) (org.I0Itec.zkclient.ZkClient)
[2019-05-15 22:20:14,399] INFO Updated PartitionLeaderEpoch. New: {epoch:21, offset:248090181}, Current: {epoch:20, offset234983013} for Partition: ZTO_BALANCE_SITE_LIST-7. Cache now contains 1 entries. (kafka.server.epoch.LeaderEpochFileCache)
[2019-05-15 22:20:14,399] INFO Unable to reconnect to ZooKeeper service, session 0x167772767ad92fd has expired, closing socket connection (org.apache.zookeeper.ClientCnxn)
[2019-05-15 22:20:14,399] INFO Initiating client connection, connectString=192.168.x.x:2181,192.168.x.x:2181,192.168.x.x:2181 sessionTimeout=6000 watcher=org.I0Itec.zkclient.ZkClient@50a638b5 (org.apache.zookeeper.ZooKeeper)
```



**查看zk日志** 

```
2019-05-15 22:49:58,992 [myid:3] - INFO  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:ZooKeeperServer@942] - Client attempting to establish new session at /192.168.x.x:20482
2019-05-15 22:49:58,993 [myid:3] - INFO  [CommitProcessor:3:ZooKeeperServer@687] - Established session 0x367772767999305 with negotiated timeout 6000 for client /192.168.x.x:20482
2019-05-15 22:49:58,994 [myid:3] - INFO  [ProcessThread(sid:3 cport:-1)::PrepRequestProcessor@648] - Got user-level KeeperException when processing sessionid:0x367772767999305 type:create cxid:0x3 zxid:0x204ae6e7d txntype:-1 reqpath:n/a Error Path:/brokers Error:KeeperErrorCode = NodeExists for /brokers
2019-05-15 22:49:58,994 [myid:3] - INFO  [ProcessThread(sid:3 cport:-1)::PrepRequestProcessor@648] - Got user-level KeeperException when processing sessionid:0x367772767999305 type:create cxid:0x4 zxid:0x204ae6e7e txntype:-1 reqpath:n/a Error Path:/brokers/ids Error:KeeperErrorCode = NodeExists for /brokers/ids
2019-05-15 22:50:01,633 [myid:3] - INFO  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:NIOServerCnxnFactory@192] - Accepted socket connection from /127.0.0.1:14379
2019-05-15 22:50:01,636 [myid:3] - INFO  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:ZooKeeperServer@942] - Client attempting to establish new session at /127.0.0.1:14379
2019-05-15 22:50:01,637 [myid:3] - INFO  [CommitProcessor:3:ZooKeeperServer@687] - Established session 0x367772767999306 with negotiated timeout 30000 for client /127.0.0.1:14379
2019-05-15 22:50:01,958 [myid:3] - WARN  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:NIOServerCnxn@368] - caught end of stream exception
EndOfStreamException: Unable to read additional data from client sessionid 0x367772767999306, likely client has closed socket
	at org.apache.zookeeper.server.NIOServerCnxn.doIO(NIOServerCnxn.java:239)
	at org.apache.zookeeper.server.NIOServerCnxnFactory.run(NIOServerCnxnFactory.java:203)
	at java.lang.Thread.run(Thread.java:745)
```



**查看GC日志** 

```
2019-05-15T22:25:59.422+0800: 21989402.560: [Full GC (Allocation Failure)  8071M->4285M(8192M), 10.3987109 secs]
   [Eden: 0.0B(408.0M)->0.0B(408.0M) Survivors: 0.0B->0.0B Heap: 8071.2M(8192.0M)->4285.7M(8192.0M)], [Metaspace: 32321K->32321K(1077248K)]
 [Times: user=21.74 sys=0.00, real=10.39 secs] 
2019-05-15T22:26:09.824+0800: 21989412.962: [GC concurrent-mark-abort]
2019-05-15T22:26:09.915+0800: 21989413.053: [GC pause (G1 Humongous Allocation) (young) (initial-mark), 0.0274477 secs]
   [Parallel Time: 22.3 ms, GC Workers: 33]
      [GC Worker Start (ms): Min: 21989413054.9, Avg: 21989413055.4, Max: 21989413055.8, Diff: 0.9]
      [Ext Root Scanning (ms): Min: 13.3, Avg: 14.2, Max: 15.2, Diff: 1.9, Sum: 468.0]
      [Update RS (ms): Min: 0.7, Avg: 1.2, Max: 2.1, Diff: 1.3, Sum: 40.4]
         [Processed Buffers: Min: 1, Avg: 2.7, Max: 6, Diff: 5, Sum: 89]
      [Scan RS (ms): Min: 0.0, Avg: 0.3, Max: 0.6, Diff: 0.6, Sum: 8.6]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Object Copy (ms): Min: 4.6, Avg: 5.8, Max: 6.6, Diff: 2.0, Sum: 191.1]
      [Termination (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.4]
         [Termination Attempts: Min: 1, Avg: 1.4, Max: 3, Diff: 2, Sum: 46]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.1, Max: 0.2, Diff: 0.2, Sum: 2.9]
      [GC Worker Total (ms): Min: 21.1, Avg: 21.6, Max: 22.0, Diff: 0.9, Sum: 711.5]
      [GC Worker End (ms): Min: 21989413076.9, Avg: 21989413076.9, Max: 21989413077.0, Diff: 0.2]
   [Code Root Fixup: 0.2 ms]
   [Code Root Purge: 0.1 ms]
   [Clear CT: 0.5 ms]
   [Other: 4.3 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 1.2 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.9 ms]
      [Humongous Register: 0.0 ms]
      [Humongous Reclaim: 0.0 ms]
      [Free CSet: 0.3 ms]
   [Eden: 192.0M(408.0M)->0.0B(396.0M) Survivors: 0.0B->12.0M Heap: 4476.9M(8192.0M)->4295.3M(8192.0M)]
 [Times: user=0.73 sys=0.00, real=0.03 secs] 
 
 2019-05-15T22:26:49.312+0800: 21989452.450: [Full GC (Allocation Failure)  8056M->3891M(8192M), 10.2331784 secs]
   [Eden: 0.0B(408.0M)->0.0B(408.0M) Survivors: 0.0B->0.0B Heap: 8056.7M(8192.0M)->3891.8M(8192.0M)], [Metaspace: 32321K->32321K(1077248K)]
 [Times: user=21.25 sys=0.00, real=10.23 secs] 
2019-05-15T22:26:59.548+0800: 21989462.686: [GC concurrent-mark-abort]
2019-05-15T22:26:59.554+0800: 21989462.692: [GC pause (G1 Humongous Allocation) (young) (initial-mark), 0.0472311 secs]
   [Parallel Time: 41.6 ms, GC Workers: 33]
      [GC Worker Start (ms): Min: 21989462693.5, Avg: 21989462693.8, Max: 21989462694.2, Diff: 0.7]
      [Ext Root Scanning (ms): Min: 14.4, Avg: 15.6, Max: 16.7, Diff: 2.4, Sum: 515.6]
      [Update RS (ms): Min: 0.4, Avg: 1.2, Max: 5.1, Diff: 4.7, Sum: 38.4]
         [Processed Buffers: Min: 1, Avg: 2.5, Max: 8, Diff: 7, Sum: 82]
      [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Object Copy (ms): Min: 19.3, Avg: 23.8, Max: 25.1, Diff: 5.8, Sum: 783.9]
      [Termination (ms): Min: 0.0, Avg: 0.2, Max: 0.2, Diff: 0.2, Sum: 5.8]
         [Termination Attempts: Min: 1, Avg: 72.2, Max: 93, Diff: 92, Sum: 2384]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.0, Max: 0.1, Diff: 0.1, Sum: 1.6]
      [GC Worker Total (ms): Min: 40.4, Avg: 40.8, Max: 41.1, Diff: 0.7, Sum: 1345.4]
      [GC Worker End (ms): Min: 21989462734.6, Avg: 21989462734.6, Max: 21989462734.6, Diff: 0.1]
   [Code Root Fixup: 0.3 ms]
   [Code Root Purge: 0.1 ms]
   [Clear CT: 0.8 ms]
   [Other: 4.6 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 1.6 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 1.1 ms]
      [Humongous Register: 0.0 ms]
      [Humongous Reclaim: 0.0 ms]
      [Free CSet: 0.1 ms]
   [Eden: 40.0M(408.0M)->0.0B(388.0M) Survivors: 0.0B->20.0M Heap: 3933.8M(8192.0M)->3912.6M(8192.0M)]
```



备注：频繁Full GC, 间隔在1到2分钟，每次Full GC的时间长达10秒以上。查看server和zk日志，发现问题节点在频繁的连接zk，导致partition leader的频繁选举。



**监控查看** 

问题节点堆内存和老年代回收居高不下

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210224095739.png)



回收时间和回收次数明显增加

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210224095811.png)



网络空闲率显著高于其他节点

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210224095900.png)



问题节点在三天前堆内存和老年代使用开始陡增

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210224095949.png)



# 复盘总结

1. 由于堆内存不能正常回收，导致频繁(1到2分钟)的Full GC.

2. 每次Full GC长达10秒以上，导致连接zk的session过期（默认6秒）

3. Session的过期导致问题节点频繁重连zk集群，造成频繁Partition Leader选举



# 未尽事项

1. 由于未dump当时的堆照，分析堆内存异常原因比较困难，如果调查三天来有哪些业务使用造成的，也比较困难。

2. 恢复GC短信告警，发生时问题时告警短信异常。

3. 除了增加GC回收告警外，增加Full GC告警

