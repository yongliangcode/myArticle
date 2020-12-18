---
title: MQ15# RocketMQ生产环境配置
categories: RocketMQ
tags: RocketMQ
date: 2020-12-20 20:15:01
---



一份RocketMQ生产环境的配置文件，供参考，集群架构为异步刷盘异步复制。另外有补充的欢迎后台留言给我。



```
#请修改

brokerClusterName=XXXCluster

brokerName=broker-a

brokerId=0

listenPort=10911

#请修改

namesrvAddr=x.x.x.x:9876;x.x.x.x::9876

defaultTopicQueueNums=4

autoCreateTopicEnable=false

autoCreateSubscriptionGroup=false

deleteWhen=04

fileReservedTime=48

mapedFileSizeCommitLog=1073741824

mapedFileSizeConsumeQueue=50000000

destroyMapedFileIntervalForcibly=120000

redeleteHangedFileInterval=120000

diskMaxUsedSpaceRatio=88

#存储路径

storePathRootDir=/data/rocketmq/store

#commitLog存储路径

storePathCommitLog=/data/rocketmq/store/commitlog

#消费队列存储路径

storePathConsumeQueue=/data/rocketmq/store/consumequeue

# 消息索引存储路径

storePathIndex=/data/rocketmq/store/index

# checkpoint 文件存储路径

storeCheckpoint=/data/rocketmq/store/checkpoint

#abort 文件存储路径

abortFile=/data/rocketmq/store/abort

maxMessageSize=65536

flushCommitLogLeastPages=4

flushConsumeQueueLeastPages=2

flushCommitLogThoroughInterval=10000

flushConsumeQueueThoroughInterval=60000

brokerRole=SYNC_MASTER

flushDiskType=ASYNC_FLUSH

checkTransactionMessageEnable=false

maxTransferCountOnMessageInMemory=1000

transientStorePoolEnable=true

warmMapedFileEnable=true

pullMessageThreadPoolNums=128

slaveReadEnable=true

transferMsgByHeap=false

waitTimeMillsInSendQueue=1000
```

