---
title: MQ36# RocketMQ Topic创建
categories: RocketMQ
tags: RocketMQ
date: 2020-12-20 22:59:01
---



# Topic创建的方式

Topic的创建分为自动创建和通过命令行创建两种。通过broker配置参数autoCreateTopicEnable设置。 通常可以在非生产环境开启自动创建，生产环境待审批后再进行创建。

\* 命令行创建

```
sh bin/mqadmin updateTopic -c DefaultCluster -n localhost:9876 -t threezto-test -r 12 -w 12
```



# 客户端发起Topic创建请求

\* 客户端工作：向集群中各个broker主节点通知topic配置变更

\* 参数设定：通过参数指定读队列数量、写队列数量、权限、当指定-c时，在该集群的所有broker都会创建

\* 调用链

```
UpdateTopicSubCommand.execute()->defaultMQAdminExtImpl.createAndUpdateTopicConfig->MQClientAPIImpl.createTopic
```

```
//-c指定，集群中每个broker主节点都会创建

Set<String> masterSet =

CommandUtil.fetchMasterAddrByClusterName(defaultMQAdminExt, clusterName);

for (String addr : masterSet) {

defaultMQAdminExt.createAndUpdateTopicConfig(addr, topicConfig);

System.out.printf("create topic to %s success.%n", addr);

}
```



<!--more-->



# Broker处理Topic创建

\* Broker处理请求

1.更改本地topic配置缓存topicConfigTable

2.将缓存topicConfigTable配置信息写入磁盘

3.向NameServer上报变更信息

4.主从同步变更信息

\* 调用链

```
AdminBrokerProcessor.processRequest()->RequestCode.UPDATE_AND_CREATE_TOPIC->updateAndCreateTopic
```

```
private RemotingCommand updateAndCreateTopic(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException {

//省略代码

//更改broker缓存topicConfigTable信息并落盘存储

this.brokerController.getTopicConfigManager().updateTopicConfig(topicConfig);

//向nameSrv上报变更信息，主从同步

this.brokerController.registerBrokerAll(false, true);

return null;

}
```



\* 更改本地topic配置缓存topicConfigTable以及持久化磁盘

```
public void updateTopicConfig(final TopicConfig topicConfig) {

//更新Topic缓存配置

TopicConfig old = this.topicConfigTable.put(topicConfig.getTopicName(), topicConfig);

if (old != null) {

LOG.info("update topic config, old:[{}] new:[{}]", old, topicConfig);

} else {

LOG.info("create new topic [{}]", topicConfig);

}

//递增版本

this.dataVersion.nextVersion();

//存储本地磁盘

this.persist();

}

/**

\* 持久化磁盘

*/

public synchronized void persist() {

String jsonString = this.encode(true);

if (jsonString != null) {

String fileName = this.configFilePath();

try {

MixAll.string2File(jsonString, fileName);

} catch (IOException e) {

PLOG.error("persist file Exception, " + fileName, e);

}

}

}
```



\* Topic配置存储的磁盘格式config/topics.json

```
"topic_online_test":{

"order":false,

"perm":6,

"readQueueNums":4,

"topicFilterType":"SINGLE_TAG",

"topicName":"topic_online_test",

"topicSysFlag":0,

"writeQueueNums":4

}
```



\* 向NameServer上报变更信息，主从同步.

NameServer收到请求处理见：[RocketMQ NameServer源码梳理笔记](https://www.jianshu.com/p/db5ed97fd19d)

```
public synchronized void registerBrokerAll(final boolean checkOrderConfig, boolean oneway) {

TopicConfigSerializeWrapper topicConfigWrapper = this.getTopicConfigManager().buildTopicConfigSerializeWrapper();

//Broker只有读或者写权限时, Topic设置为与broker相同的权限

if (!PermName.isWriteable(this.getBrokerConfig().getBrokerPermission())

|| !PermName.isReadable(this.getBrokerConfig().getBrokerPermission())) {

ConcurrentHashMap<String, TopicConfig> topicConfigTable = new ConcurrentHashMap<String, TopicConfig>();

for (TopicConfig topicConfig : topicConfigWrapper.getTopicConfigTable().values()) {

TopicConfig tmp =

new TopicConfig(topicConfig.getTopicName(), topicConfig.getReadQueueNums(), topicConfig.getWriteQueueNums(),

this.brokerConfig.getBrokerPermission());

topicConfigTable.put(topicConfig.getTopicName(), tmp);

}

topicConfigWrapper.setTopicConfigTable(topicConfigTable);

}

//向所有NameServer节点注册Topic信息

RegisterBrokerResult registerBrokerResult = this.brokerOuterAPI.registerBrokerAll(

this.brokerConfig.getBrokerClusterName(),

this.getBrokerAddr(),

this.brokerConfig.getBrokerName(),

this.brokerConfig.getBrokerId(),

this.getHAServerAddr(),

topicConfigWrapper,

this.filterServerManager.buildNewFilterServerList(),

oneway,

this.brokerConfig.getRegisterBrokerTimeoutMills());

if (registerBrokerResult != null) {

if (this.updateMasterHAServerAddrPeriodically && registerBrokerResult.getHaServerAddr() != null) {

this.messageStore.updateHaMasterAddress(registerBrokerResult.getHaServerAddr());

}

//Slave同步Master信息 每分钟定时同步

this.slaveSynchronize.setMasterAddr(registerBrokerResult.getMasterAddr());

if (checkOrderConfig) {

this.getTopicConfigManager().updateOrderTopicConfig(registerBrokerResult.getKvTable());

}

}

}
```

