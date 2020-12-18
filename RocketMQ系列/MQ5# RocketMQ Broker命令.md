---
title: MQ5# RocketMQ Broker命令
categories: RocketMQ
tags: RocketMQ
date: 2020-12-17 11:14:01
---

# 查看broker状态信息

```
bin/mqadmin brokerStatus -b 192.168.1.x:10911 -n 192.168.1.x:9876

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=128m; support was removed in 8.0

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=128m; support was removed in 8.0

bootTimestamp : 1535597348792

brokerVersion : 232

brokerVersionDesc : V4_1_0_SNAPSHOT

commitLogDirCapacity : Total : 8.7 TiB, Free : 7.7 TiB.

commitLogDiskRatio : 0.11147464487710743

commitLogMaxOffset : 6652182757965

commitLogMinOffset : 5676873023488

consumeQueueDiskRatio : 0.11147464487710743

dispatchBehindBytes : 0

dispatchMaxBuffer : 0

earliestMessageTimeStamp : 1539290831130

getFoundTps : 24229.27707229277 24670.716261707163 24323.07766631239

getMessageEntireTimeMax : 4895

getMissTps : 23384.56154384562 24047.195280471955 23735.196223133637

getTotalTps : 47613.83861613839 48717.91154217912 48058.27388944603

getTransferedTps : 28046.395360463954 28519.931340199313 28905.699949690646

msgGetTotalTodayMorning : 30682264110

msgGetTotalTodayNow : 31750490006

msgGetTotalYesterdayMorning : 29219504313

msgPutTotalTodayMorning : 7329767893

msgPutTotalTodayNow : 7523090457

msgPutTotalYesterdayMorning : 7031035625

pageCacheLockTimeMills : 0

pullThreadPoolQueueCapacity : 100000

pullThreadPoolQueueHeadWaitTimeMills: 0

pullThreadPoolQueueSize : 0

putMessageAverageSize : 807.2943103360182

putMessageDistributeTime : [<=0ms]:229497 [0~10ms]:1275 [10~50ms]:4 [50~100ms]:0 [100~200ms]:0 [200~500ms]:0 [500ms~1s]:0 [1~2s]:0 [2~3s]:0 [3~4s]:0 [4~5s]:0 [5~10s]:0 [10s~]:0

putMessageEntireTimeMax : 10885

putMessageSizeTotal : 6073348121272

putMessageTimesTotal : 7523090456

putTps : 4424.957504249575 4466.0700596607 4577.724617932119

remainHowManyDataToCommit : 6.9 KiB

remainHowManyDataToFlush : 0 B

remainTransientStoreBufferNumbs : 3

runtime : [ 48 days, 7 hours, 52 minutes, 34 seconds ]

scheduleMessageOffset_1 : 5169358,5169358

scheduleMessageOffset_10 : 8161108,8171531

scheduleMessageOffset_11 : 7864781,7877885

scheduleMessageOffset_12 : 7584977,7601110

scheduleMessageOffset_13 : 7278660,7297334

scheduleMessageOffset_14 : 7123700,7146082

scheduleMessageOffset_15 : 7013493,7063463

scheduleMessageOffset_16 : 6864847,6950753

scheduleMessageOffset_17 : 6549491,6803414

scheduleMessageOffset_18 : 12887811,12894744

scheduleMessageOffset_3 : 13667462,13667752

scheduleMessageOffset_4 : 9780224,9781009

scheduleMessageOffset_5 : 63142105,63144981

scheduleMessageOffset_6 : 9185173,9188251

scheduleMessageOffset_7 : 9002046,9006727

scheduleMessageOffset_8 : 8834782,8841531

scheduleMessageOffset_9 : 1112865066,1112879874

sendThreadPoolQueueCapacity : 10000

sendThreadPoolQueueHeadWaitTimeMills: 0

sendThreadPoolQueueSize : 0

startAcceptSendRequestTimeStamp : 0

You have mail in /var/spool/mail/root
```



```
bin/mqadmin brokerConsumeStats -b 192.168.1.x:10911 -n 192.168.1.x:9876

zeus-package-mismatch-topic zeus-package-mismatch-consumer broker-a 0 698533 698532 1 2018-10-17 18:35:58

zeus-package-mismatch-topic zeus-package-mismatch-consumer broker-a 1 698521 698520 1 2018-10-17 18:36:01

zeus-package-mismatch-topic zeus-package-mismatch-consumer broker-a 2 698514 698513 1 2018-10-17 18:36:01

zeus-package-mismatch-topic zeus-package-mismatch-consumer broker-a 3 698495 698494 1 2018-10-17 18:36:01

```





# 清除废弃的ConsumeQueue

```
bin/mqadmin cleanExpiredCQ -b 192.168.1.x:10911 -n 192.168.1.x:9876
```



# 清除废弃的Topic

```
bin/mqadmin cleanUnusedTopic -b 192.168.1.x:10911 -n 192.168.1.x:9876
```



# 获取broker配置信息

```
bin/mqadmin getBrokerConfig -b 192.168.1.x:10911 -n 192.168.1.x:9876

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=128m; support was removed in 8.0

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=128m; support was removed in 8.0

============192.168.1.x:10911============

autoCreateSubscriptionGroup = false

brokerName = broker-a

haListenPort = 10912

clientManagerThreadPoolQueueCapacity = 1000000

flushCommitLogThoroughInterval = 10000

flushCommitLogLeastPages = 4

clientCallbackExecutorThreads = 48

notifyConsumerIdsChangedEnable = true

cleanResourceInterval = 10000

channelNotActiveInterval = 60000

diskMaxUsedSpaceRatio = 88

debugLockEnable = false

messageDelayLevel = 1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h

clusterTopicEnable = true

messageIndexEnable = true

serverPooledByteBufAllocatorEnable = true

shortPollingTimeMills = 1000

commercialEnable = true

redeleteHangedFileInterval = 120000

flushConsumerOffsetInterval = 5000

flushCommitLogTimed = false

maxMessageSize = 65536

brokerId = 0

syncFlushTimeout = 5000

flushConsumeQueueThoroughInterval = 60000

clientChannelMaxIdleTimeSeconds = 120

flushDelayOffsetInterval = 10000

serverSocketRcvBufSize = 131072

flushDiskType = ASYNC_FLUSH

maxTransferBytesOnMessageInMemory = 262144

clientManageThreadPoolNums = 32

serverChannelMaxIdleTimeSeconds = 120

serverCallbackExecutorThreads = 0

transientStorePoolSize = 5

maxTransferBytesOnMessageInDisk = 65536

pullMessageThreadPoolNums = 128

clientCloseSocketIfTimeout = false

fetchNamesrvAddrByAddressServer = false

sendThreadPoolQueueCapacity = 10000

diskFallRecorded = true

transientStorePoolEnable = true

offsetCheckInSlave = false

disableConsumeIfConsumerReadSlowly = false

commitCommitLogThoroughInterval = 200

consumerManagerThreadPoolQueueCapacity = 1000000

flushIntervalConsumeQueue = 1000

clientOnewaySemaphoreValue = 65535

warmMapedFileEnable = true

slaveReadEnable = false

transferMsgByHeap = true

consumerFallbehindThreshold = 17179869184

serverAsyncSemaphoreValue = 64

startAcceptSendRequestTimeStamp = 0

flushConsumerOffsetHistoryInterval = 60000

brokerIP2 = 192.168.1.x

maxTransferCountOnMessageInDisk = 8

brokerIP1 = 192.168.1.x

deleteCommitLogFilesInterval = 100

adminBrokerThreadPoolNums = 16

storePathCommitLog = /data/rocketmq/store/commitlog

filterServerNums = 0

deleteConsumeQueueFilesInterval = 100

checkCRCOnRecover = true

serverOnewaySemaphoreValue = 256

defaultQueryMaxNum = 32

clientWorkerThreads = 4

clientSocketRcvBufSize = 131072

maxDelayTime = 40

connectTimeoutMillis = 3000

clientPooledByteBufAllocatorEnable = false

commercialTimerCount = 1

serverSocketSndBufSize = 131072

regionId = DefaultRegion

duplicationEnable = false

cleanFileForciblyEnable = true

fastFailIfNoBufferInStorePool = false

rejectTransactionMessage = false

serverSelectorThreads = 3

consumerManageThreadPoolNums = 32

haSendHeartbeatInterval = 5000

mapedFileSizeConsumeQueue = 50000000

storeCheckpoint = /data/rocketmq/store/checkpoint

commitCommitLogLeastPages = 4

longPollingEnable = true

flushConsumeQueueLeastPages = 2

storePathRootDir = /data/rocketmq/store

defaultTopicQueueNums = 4

highSpeedMode = false

commercialBaseCount = 1

checkTransactionMessageEnable = false

accessMessageInMemoryMaxRatio = 40

autoCreateTopicEnable = false

commitIntervalCommitLog = 200

brokerTopicEnable = true

namesrvAddr = 192.168.1.x:9876;192.168.1.x:9876

clientAsyncSemaphoreValue = 65535

maxMsgsNumBatch = 64

storePathConsumeQueue = /data/rocketmq/store/consumequeue

fileReservedTime = 120

deleteWhen = 04

waitTimeMillsInSendQueue = 200

commercialTransCount = 1

osPageCacheBusyTimeOutMills = 1000

abortFile = /data/rocketmq/store/abort

maxIndexNum = 20000000

registerBrokerTimeoutMills = 6000

messageIndexSafe = false

putMsgIndexHightWater = 600000

listenPort = 10911

serverWorkerThreads = 8

clientSocketSndBufSize = 131072

traceOn = true

maxHashSlotNum = 5000000

brokerRole = ASYNC_MASTER

storePathIndex = /data/rocketmq/store/index

rocketmqHome = /home/baseuser/rocketmq

useReentrantLockWhenPutMessage = false

haHousekeepingInterval = 20000

brokerPermission = 6

maxTransferCountOnMessageInMemory = 1000

useEpollNativeSelector = false

haSlaveFallbehindMax = 268435456

haTransferBatchSize = 32768

messageStorePlugIn =

pullThreadPoolQueueCapacity = 100000

brokerClusterName = ZmsClusterB

destroyMapedFileIntervalForcibly = 120000

mapedFileSizeCommitLog = 1073741824

commercialBigCount = 1

flushLeastPagesWhenWarmMapedFile = 4096

sendMessageThreadPoolNums = 1

flushIntervalCommitLog = 500
```

<!--more-->



# 发送消息到broker

```
bin/mqadmin sendMsgStatus -b 192.168.1.x:10911 -n 192.168.1.x:9876
```



# 更新broker信息

```
bin/mqadmin updateBrokerConfig -b 192.168.1.x:10911 -n 192.168.1.x:9876 -k key -v value
```