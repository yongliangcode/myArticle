---
title: MQ35# RocketMQ NameServe
categories: RocketMQ
tags: RocketMQ
date: 2020-12-20 22:58:01
---



# NameServer启动

\* 从生产环境实践来看，NameServer启动使用默认配置即可，运行良好。

```
启动命令：nohup sh bin/mqnamesrv &
```

\* NamesrvStartup.java 启动入口类，NameServer 启动默认端口9876

```
nettyServerConfig.setListenPort(9876)
```

\* 每10秒钟扫描一次，移除失效的broker，同时移除缓存元数据



<!--more-->

```
//初始化NameServer

boolean initResult = controller.initialize();

public boolean initialize() {

//加载KV配置

this.kvConfigManager.load();

this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);

this.remotingExecutor =

Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), new ThreadFactoryImpl("RemotingExecutorThread_"));

this.registerProcessor();

//每10秒钟扫描一次，移除失效的broker

this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

@Override

public void run() {

NamesrvController.this.routeInfoManager.scanNotActiveBroker();

}

}, 5, 10, TimeUnit.SECONDS);

//每隔10秒钟打印一次KV配置信息

this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

@Override

public void run() {

NamesrvController.this.kvConfigManager.printAllPeriodically();

}

}, 1, 10, TimeUnit.MINUTES);

return true;

}
```



\* 失效时间为2分钟，即：Broker在2分钟内未上报心跳会被移除

```
//失效时间为2分钟，即：Broker在2分钟内未上报心跳会被移除

public void scanNotActiveBroker() {

Iterator<Entry<String, BrokerLiveInfo>> it = this.brokerLiveTable.entrySet().iterator();

while (it.hasNext()) {

Entry<String, BrokerLiveInfo> next = it.next();

long last = next.getValue().getLastUpdateTimestamp();

if ((last + BROKER_CHANNEL_EXPIRED_TIME) < System.currentTimeMillis()) {

RemotingUtil.closeChannel(next.getValue().getChannel());

it.remove();

log.warn("The broker channel expired, {} {}ms", next.getKey(), BROKER_CHANNEL_EXPIRED_TIME);

this.onChannelDestroy(next.getKey(), next.getValue().getChannel());

}

}

}

private final static long BROKER_CHANNEL_EXPIRED_TIME = 1000 * 60 * 2;
```



# DefaultRequestProcessor

\* 用于响应客户端、Broker的请求。主要向NameServer发送心跳包、获取Cluster、Broker、Topic元数据信息。

\* 调用链：在NameServer启动时注册，NamesrvController.initialize()->registerProcessor()。

```
public RemotingCommand processRequest(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException {

if (log.isDebugEnabled()) {

log.debug("receive request, {} {} {}",

request.getCode(),

RemotingHelper.parseChannelRemoteAddr(ctx.channel()),

request);

}

switch (request.getCode()) {

case RequestCode.PUT_KV_CONFIG://增加NameServer配置信息；由DefaultMQAdminExt使用

return this.putKVConfig(ctx, request);

case RequestCode.GET_KV_CONFIG://根据NameSpace和key获取NameServer配置信息；由DefaultMQAdminExt使用

return this.getKVConfig(ctx, request);

case RequestCode.DELETE_KV_CONFIG: //据NameSapce和Key删除NameServerr配置信息

return this.deleteKVConfig(ctx, request);

case RequestCode.REGISTER_BROKER: //注册Broker信息；由BrokerOuterAPI.registerBroker使用，在BrokerController启动时调用

Version brokerVersion = MQVersion.value2Version(request.getVersion());

if (brokerVersion.ordinal() >= MQVersion.Version.V3_0_11.ordinal()) {

return this.registerBrokerWithFilterServer(ctx, request);

} else {

return this.registerBroker(ctx, request);

}

case RequestCode.UNREGISTER_BROKER://移除注销broker信息；由BrokerOuterAPI.unregisterBroker使用，在BrokerController.shutdown时调用

return this.unregisterBroker(ctx, request);

case RequestCode.GET_ROUTEINTO_BY_TOPIC: //获取Topic路由信息 TopicRouteData

return this.getRouteInfoByTopic(ctx, request);

case RequestCode.GET_BROKER_CLUSTER_INFO://获取Cluster及Broker信息

return this.getBrokerClusterInfo(ctx, request);

case RequestCode.WIPE_WRITE_PERM_OF_BROKER: //去除该broker上所有topic的写权限

return this.wipeWritePermOfBroker(ctx, request);

case RequestCode.GET_ALL_TOPIC_LIST_FROM_NAMESERVER: //获取所有的Topic列表

return getAllTopicListFromNameserver(ctx, request);

case RequestCode.DELETE_TOPIC_IN_NAMESRV: //从nameServer中删除topic

return deleteTopicInNamesrv(ctx, request);

case RequestCode.GET_KVLIST_BY_NAMESPACE: //获取配置信息 configTable

return this.getKVListByNamespace(ctx, request);

case RequestCode.GET_TOPICS_BY_CLUSTER: //获取该集群下的所有topic list

return this.getTopicsByCluster(ctx, request);

case RequestCode.GET_SYSTEM_TOPIC_LIST_FROM_NS: // 此处意思为：系统会将集群名称、broker名称作为默认topic创建。现在获取这类topic

return this.getSystemTopicListFromNs(ctx, request);

case RequestCode.GET_UNIT_TOPIC_LIST: //暂无使用

return this.getUnitTopicList(ctx, request);

case RequestCode.GET_HAS_UNIT_SUB_TOPIC_LIST: //暂无使用

return this.getHasUnitSubTopicList(ctx, request);

case RequestCode.GET_HAS_UNIT_SUB_UNUNIT_TOPIC_LIST://暂无使用

return this.getHasUnitSubUnUnitTopicList(ctx, request);

case RequestCode.UPDATE_NAMESRV_CONFIG: //更新properties请求

return this.updateConfig(ctx, request);

case RequestCode.GET_NAMESRV_CONFIG: //获取properties内容

return this.getConfig(ctx, request);

default:

break;

}

return null;

}
```

<!--more-->



# **注册broker信息**

\* Broker每隔30秒向所有的NameServer上报Topic注册信息

\* Broker调用链：BrokerController.start()->this.registerBrokerAll()->this.brokerOuterAPI.registerBrokerAll()

```
//每隔30秒向所有的NameServer上报Topic注册信息

this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

@Override

public void run() {

try {

BrokerController.this.registerBrokerAll(true, false);

} catch (Throwable e) {

log.error("registerBrokerAll Exception", e);

}

}

}, 1000 * 10, 1000 * 30, TimeUnit.MILLISECONDS);
```



\* 服务端处理主要包括：注册集群信息clusterAddrTable、注册broker信息brokerAddrTable、注册topic信息topicQueueTable、

broker心跳包brokerLiveTable

\* NameServer处理链：DefaultRequestProcessor->processRequest->RequestCode.REGISTER_BROKER->this.registerBroker->

RouteInfoManager.registerBroker()

```
public RegisterBrokerResult registerBroker(

final String clusterName,

final String brokerAddr,

final String brokerName,

final long brokerId,

final String haServerAddr,

final TopicConfigSerializeWrapper topicConfigWrapper,

final List<String> filterServerList,

final Channel channel) {

RegisterBrokerResult result = new RegisterBrokerResult();

try {

try {

//加写锁，防止并发修改

this.lock.writeLock().lockInterruptibly();

//注册集群信息

Set<String> brokerNames = this.clusterAddrTable.get(clusterName);

if (null == brokerNames) {

brokerNames = new HashSet<String>();

this.clusterAddrTable.put(clusterName, brokerNames);

}

brokerNames.add(brokerName);

boolean registerFirst = false;

//注册broker信息

BrokerData brokerData = this.brokerAddrTable.get(brokerName);

if (null == brokerData) {

registerFirst = true;

brokerData = new BrokerData(clusterName, brokerName, new HashMap<Long, String>());

this.brokerAddrTable.put(brokerName, brokerData);

}

String oldAddr = brokerData.getBrokerAddrs().put(brokerId, brokerAddr);

registerFirst = registerFirst || (null == oldAddr);

//Topic配置变化了；Master Broker第一次注册或者Topic dataVersion不相同时更新路由信息

//有Topic新增时dataVersion会递增

if (null != topicConfigWrapper //

&& MixAll.MASTER_ID == brokerId) {

if (this.isBrokerTopicConfigChanged(brokerAddr, topicConfigWrapper.getDataVersion())//

|| registerFirst) {

ConcurrentMap<String, TopicConfig> tcTable =

topicConfigWrapper.getTopicConfigTable();

if (tcTable != null) {

for (Map.Entry<String, TopicConfig> entry : tcTable.entrySet()) {

this.createAndUpdateQueueData(brokerName, entry.getValue()); //更新topicQueueTable

}

}

}

}

//更新broker心跳信息

BrokerLiveInfo prevBrokerLiveInfo = this.brokerLiveTable.put(brokerAddr,

new BrokerLiveInfo(

System.currentTimeMillis(),

topicConfigWrapper.getDataVersion(),

channel,

haServerAddr));

//新broker注册时会有日志输出

if (null == prevBrokerLiveInfo) {

log.info("new broker registerd, {} HAServer: {}", brokerAddr, haServerAddr);

}

//更新filterServer信息

if (filterServerList != null) {

if (filterServerList.isEmpty()) {

this.filterServerTable.remove(brokerAddr);

} else {

this.filterServerTable.put(brokerAddr, filterServerList);

}

}

//Slave设置MasterAddr和HaServerAddr

if (MixAll.MASTER_ID != brokerId) {

String masterAddr = brokerData.getBrokerAddrs().get(MixAll.MASTER_ID);

if (masterAddr != null) {

BrokerLiveInfo brokerLiveInfo = this.brokerLiveTable.get(masterAddr);

if (brokerLiveInfo != null) {

result.setHaServerAddr(brokerLiveInfo.getHaServerAddr());

result.setMasterAddr(masterAddr);

}

}

}

} finally {

this.lock.writeLock().unlock();

}

} catch (Exception e) {

log.error("registerBroker Exception", e);

}

return result;

}


```





# NameServer缓存元数据结构

\* 注册集群信息clusterAddrTable

```
// Broker集群信息; key为集群名称，value为所有的broker名称集合

private final HashMap<String/* clusterName */, Set<String/* brokerName */>> clusterAddrTable;
```

\* 注册broker信息brokerAddrTable

```
/**

\* Broker信息；key为brokerName，value为BrokerData

*/

private final HashMap<String/* brokerName */, BrokerData> brokerAddrTable;

public class BrokerData implements Comparable<BrokerData> {

/**

\* Cluster名称

*/

private String cluster;

/**

\* broker名称

*/

private String brokerName;

/**

\* 0->ip:port

*/

private HashMap<Long/* brokerId */, String/* broker address */> brokerAddrs;

}
```



\* 注册topic信息topicQueueTable

```
//消息队列路由信息；key为topic，value为QueueData

private final HashMap<String/* topic */, List<QueueData>> topicQueueTable;

public class QueueData implements Comparable<QueueData> {

// Broker名称

private String brokerName;

//读队列个数，默认4个

private int readQueueNums;

//写队列个数，默认4个

private int writeQueueNums;

// 队列权限

private int perm;

//配置的,同步复制还是异步复制标记,对应TopicConfig.topicSysFlag

private int topicSynFlag;

}
```



\* Broker心跳包brokerLiveTable

```
//Broker状态信息，NameServer每次收到心跳包会替换该信息，每隔30秒更新一次

//brokerAddr: ip:port->{}

private final HashMap<String/* brokerAddr */, BrokerLiveInfo> brokerLiveTable;

class BrokerLiveInfo {

// 存储上次收到心跳包的时间,每隔30秒更新一次

private long lastUpdateTimestamp;

private DataVersion dataVersion;

private Channel channel;

private String haServerAddr;

}
```

