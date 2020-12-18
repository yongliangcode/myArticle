---
title: MQ6# RocketMQ NameSrv命令
categories: RocketMQ
tags: RocketMQ
date: 2020-12-17 13:15:01
---

# 获取namesrv配置

```
[baseuser@HZPL00xxxx rocketmq]$ sh bin/mqadmin getNamesrvConfig -n 192.168.1.x:9876

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=128m; support was removed in 8.0

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=128m; support was removed in 8.0

============192.168.1.x:9876============

serverChannelMaxIdleTimeSeconds = 120

listenPort = 9876

serverCallbackExecutorThreads = 0

serverAsyncSemaphoreValue = 64

serverSocketSndBufSize = 4096

rocketmqHome = /home/baseuser/rocketmq

clusterTest = false

serverSelectorThreads = 3

useEpollNativeSelector = false

orderMessageEnable = false

serverPooledByteBufAllocatorEnable = true

kvConfigPath = /home/baseuser/namesrv/kvConfig.json

serverWorkerThreads = 8

serverSocketRcvBufSize = 4096

productEnvName = center

serverOnewaySemaphoreValue = 256

configStorePath = /home/baseuser/namesrv/namesrv.properties
```

# 删除nameSrv配置

```
sh bin/mqadmin deleteKvConfig -n 192.168.1.x:9876 -s namesapce -k key

Delete KV config.

-s set the namespace

-k key set the key name
```



<!--more-->



# 更新了nameSrv配置

```
sh bin/mqadmin updateKvConfig -n 192.168.1.x:9876 -s namesapce -k key

update KV config.

-s set the namespace

-k key set the key name

-v value set the key value
```



# 更新nameSrv配置

```
sh bin/mqadmin updateNamesrvConfig -n 192.168.1.x:9876 -s namesapce -k key

update KV config.

-k key set the key name

-v value set the key value
```



# 取消Broker写权限

```
sh bin/mqadmin wipeWritePerm -n 192.168.1.x:9876 -b brokerAddr

Wipe write perm of broker in all name server

-b brokerName
```

