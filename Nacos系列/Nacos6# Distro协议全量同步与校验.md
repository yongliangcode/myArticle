---
title: Nacos6# Distro协议全量同步与校验
categories: Nacos
tags: Nacos
date: 2021-06-26 11:55:01
---



# 引言



本文接着撸Distro协议，上文中分析了寻址模式。有了地址就要建立连接，有了连接就能通信了。集群之间都交互啥数据？本文就扒一扒全量同步和节点之间数据校验。



# 内容提要



### 节点间建立RCP连接

* 订阅了MembersChangeEvent事件，集群节点有变更能够收到回调通知
* 与集群中其他节点建立grpc连接并缓存到Map其中key格式为「Cluster-IP:Port」



### 节点间校验数据通信

* 节点之间发送校验数据是在全量同步后进行的
* 发送校验的频率默认为5秒钟一次
* 校验数据包括clientId和version，其中version为保留字段当前为0
* 接受到校验数据后如果缓存中存在该client表示校验成功，同时更新保鲜时间，否则校验失败



### 全量数据同步

* 在节点启动时会从集群中其他节点中的一个节点同步快照数据并缓存在Map中
* 缓存的数据类型分类两类分别为HTTP和gRPC
* 具体数据即客户端注册节点信息含命名空间、分组名称、服务名称、节点Instance信息等
* 集群中每个节点都拥有所有的快照数据



<!--more-->



# 节点间建立RPC连接

节点之间要通信，需要建立连接。Nacos集群节点之间也不例外，下面看下Nacos是如何和集群之间建立连接的，以gRPC为例。

Nacos中ClusterRpcClientProxy封装了集群中节点之间的通道。

```java
@PostConstruct
public void init() {
  try {
    // 注解@1
    NotifyCenter.registerSubscriber(this);
    // 注解@2
    List<Member> members = serverMemberManager.allMembersWithoutSelf(); 
    // 注解@3
    refresh(members);
    Loggers.CLUSTER
      .warn("[ClusterRpcClientProxy] success to refresh cluster rpc client on start up,members ={} ",
            members);
  } catch (NacosException e) {
    Loggers.CLUSTER.warn("[ClusterRpcClientProxy] fail to refresh cluster rpc client,{} ", e.getMessage());
  }
}
```

**注解@1** 注册自己订阅MembersChangeEvent事件

**注解@2** 获取集群中的节点列表剔除自身节点

**注解@3** 与各个节点建立rpc通道

```java
private void refresh(List<Member> members) throws NacosException {
	  for (Member member : members) {

        if (MemberUtil.isSupportedLongCon(member)) {
            // 注解@3.1
            createRpcClientAndStart(member, ConnectionType.GRPC);
        }
    }
	  Set<Map.Entry<String, RpcClient>> allClientEntrys = RpcClientFactory.getAllClientEntries();
    Iterator<Map.Entry<String, RpcClient>> iterator = allClientEntrys.iterator();
    List<String> newMemberKeys = members.stream().filter(a -> MemberUtil.isSupportedLongCon(a))
            .map(a -> memberClientKey(a)).collect(Collectors.toList());
    // 注解@3.2
    while (iterator.hasNext()) {
        Map.Entry<String, RpcClient> next1 = iterator.next();
        if (next1.getKey().startsWith("Cluster-") && !newMemberKeys.contains(next1.getKey())) {
            Loggers.CLUSTER.info("member leave,destroy client of member - > : {}", next1.getKey());
            RpcClientFactory.getClient(next1.getKey()).shutdown();
            iterator.remove();
        }
    }

}
```

**注解@3.1** 为集群中每个节点member创建rcp client

**注解@3.2** 关闭旧的grpc连接

```java
private void createRpcClientAndStart(Member member, ConnectionType type) throws NacosException {
    Map<String, String> labels = new HashMap<String, String>(2);
    labels.put(RemoteConstants.LABEL_SOURCE, RemoteConstants.LABEL_SOURCE_CLUSTER);
    // 注解@3.1.1
    String memberClientKey = memberClientKey(member);
    RpcClient client = RpcClientFactory.createClusterClient(memberClientKey, type, labels);
    if (!client.getConnectionType().equals(type)) {
        Loggers.CLUSTER.info(",connection type changed,destroy client of member - > : {}", member);
        RpcClientFactory.destroyClient(memberClientKey);
        // 注解@3.1.2
        client = RpcClientFactory.createClusterClient(memberClientKey, type, labels);
    }

    if (client.isWaitInitiated()) {
        Loggers.CLUSTER.info("start a new rpc client to member - > : {}", member);

        // 注解@3.1.3
        client.serverListFactory(new ServerListFactory() {
            @Override
            public String genNextServer() {
                return member.getAddress(); // 返回连接集群其他节点地址
            }

            @Override
            public String getCurrentServer() {
                return member.getAddress();
            }

            @Override
            public List<String> getServerList() {
                return Lists.newArrayList(member.getAddress());
            }
        });
        // 注解@3.1.4
        client.start(); 
    }
}
```

**注解@3.1.1** memberClientKey由「Cluster-IP:Port」构成，例如：Cluster-1.2.3.4:2008

**注解@3.1.2** 创建grpc client并缓存在 clientMap，key为memberClientKey 此时client的状态为WAIT_INIT

**注解@3.1.3** 集群中固定的某一台节点

**注解@3.1.4**  grpc连接集群中的member节点设置client的状态RUNNING



**小结：** 在与Nacos集群其他节点建立连接的过程中做了两件事情：@1.订阅了MembersChangeEvent事件 @2.与集群中其他节点建立grpc连接并缓存到Map其中key格式为「Cluster-IP:Port」。



# 节点间校验数据通信



节点之间建立rpc通道必然是为了互相之间能通信，其中一个通信是节点之间发送校验数据。那为什么要发这些校验数据？这些数据都是些什么内容？下面咱就去扒一扒。



在DistroProtocol的构造函数中的最后一个行有一个startDistroTask()，主要分析startVerifyTask的逻辑。

```java
public DistroProtocol(ServerMemberManager memberManager, DistroComponentHolder distroComponentHolder,
        DistroTaskEngineHolder distroTaskEngineHolder, DistroConfig distroConfig) {
    this.memberManager = memberManager;
    this.distroComponentHolder = distroComponentHolder;
    this.distroTaskEngineHolder = distroTaskEngineHolder;
    this.distroConfig = distroConfig;
    startDistroTask();
}
```

```java
private void startDistroTask() {
    // 单机模式直接返回
    if (EnvUtil.getStandaloneMode()) {
        isInitialized = true;
        return;
    }
    startVerifyTask();
    startLoadTask();
}
```

```java
private void startVerifyTask() {
  	// 注解@4
    GlobalExecutor.schedulePartitionDataTimedSync(new DistroVerifyTimedTask(memberManager, distroComponentHolder,
            distroTaskEngineHolder.getExecuteWorkersManager()), distroConfig.getVerifyIntervalMillis());
}
```

**注解@4**  每隔5秒执行，也就是节点之间发送校验时间的默认频率是5秒。

​			   可以通过配置参数「nacos.core.protocol.distro.data.verify_interval_ms」自定义。

接着看DistroVerifyTimedTask的run方法。

```java
@Override
public void run() {
    try {
        // 注解@5
        List<Member> targetServer = serverMemberManager.allMembersWithoutSelf();
        if (Loggers.DISTRO.isDebugEnabled()) {
            Loggers.DISTRO.debug("server list is: {}", targetServer);
        }

        // 注解@6
        for (String each : distroComponentHolder.getDataStorageTypes()) {
            verifyForDataStorage(each, targetServer);
        }
    } catch (Exception e) {
        Loggers.DISTRO.error("[DISTRO-FAILED] verify task failed.", e);
    }
}
```

**注解@5** 拿到集群中其他节点

**注解@6** 在Nacos server启动时初始化时两种类型HTTP和gRPC，本文以gRPC为例进行分析。

```java
private void verifyForDataStorage(String type, List<Member> targetServer) {
    // 注解@7
    DistroDataStorage dataStorage = distroComponentHolder.findDataStorage(type);
    // 注解@8
    if (!dataStorage.isFinishInitial()) {  // 未完成全量数据同步退出
        Loggers.DISTRO.warn("data storage {} has not finished initial step, do not send verify data",
                dataStorage.getClass().getSimpleName());
        return;
    }

    //注解@9
    List<DistroData> verifyData = dataStorage.getVerifyData();
    if (null == verifyData || verifyData.isEmpty()) {
        return;
    }

    for (Member member : targetServer) {
        DistroTransportAgent agent = distroComponentHolder.findTransportAgent(type);
        if (null == agent) {
            continue;
        }
      	// 注解@10
        executeTaskExecuteEngine.addTask(member.getAddress() + type,
                new DistroVerifyExecuteTask(agent, verifyData, member.getAddress(), type));
    }
}
```

**注解@7** Nacos启动时缓存在dataStorageMap中两种类型处理器分别用于处理gRPC和HTTP通信方式。

「Nacos:Naming:v2:ClientData->DistroClientDataProcessor」和 「com.alibaba.nacos.naming.iplist.->DistroDataStorageImpl」

**注解@8** 当从其他节点同步了全部数据后，则完成了初始化finished initial，全量数据同步下小节分析。

**注解@9**  获取校验的数据，数据为由本节点负责的clientId列表。

```java
@Override
public List<DistroData> getVerifyData() {
    List<DistroData> result = new LinkedList<>(); // 一组DistroData
    for (String each : clientManager.allClientId()) {
        Client client = clientManager.getClient(each);
        if (null == client || !client.isEphemeral()) { // 无效client或者非临时节点
            continue;
        }
        // 注解@9.1
        if (clientManager.isResponsibleClient(client)) {
            // 注解@9.2
            DistroClientVerifyInfo verifyData = new DistroClientVerifyInfo(client.getClientId(), 0);
            DistroKey distroKey = new DistroKey(client.getClientId(), TYPE);
            DistroData data = new DistroData(distroKey,
                    ApplicationUtils.getBean(Serializer.class).serialize(verifyData)); // 序列化校验数据
            data.setType(DataOperation.VERIFY);
            result.add(data);
        }
    }
    return result;
}
```

**注解@9.1** 判断client是否为本几点负责的逻辑为ClientManagerDelegate#isResponsibleClient。即：属于ConnectionBasedClient并且

isNative为true表示该client是直连到该节点的。

```java
@Override
public boolean isResponsibleClient(Client client) {
    return (client instanceof ConnectionBasedClient) && ((ConnectionBasedClient) client).isNative();
}
```

**注解@9.2** 构造Verify Data 主要信息为clientId，还有一个版本信息作为保留字段，目前都是0。

**注解@10** 向集群其他节点发送校验数据DistroVerifyExecuteTask#run

```java
@Override
public void run() {
    for (DistroData each : verifyData) {
        try {
            if (transportAgent.supportCallbackTransport()) { // grpc支持回调
                doSyncVerifyDataWithCallback(each);
            } else { // http不支持回调使用同步
                doSyncVerifyData(each);
            }
        } catch (Exception e) {
          //...
        }
    }
}
```



```java
@Override
public void syncVerifyData(DistroData verifyData, String targetServer, DistroCallback callback) {
    if (isNoExistTarget(targetServer)) {
        callback.onSuccess();
    }
    DistroDataRequest request = new DistroDataRequest(verifyData, DataOperation.VERIFY);
    Member member = memberManager.find(targetServer);
    try {
        DistroVerifyCallbackWrapper wrapper = new DistroVerifyCallbackWrapper(targetServer,
                verifyData.getDistroKey().getResourceKey(), callback, member);
       	// 注解@11         
        clusterRpcClientProxy.asyncRequest(member, request, wrapper); 
    } catch (NacosException nacosException) {
        callback.onFailed(nacosException);
    }
}
```

**注解@11** 向其他节点发送本节点负责的clientId信息



**那集群其他节点接收到校验数据做什么处理呢?** 



翻到DistroDataRequestHandler#handle，此处包含了处理校验数据的逻辑。

```java
@Override
public DistroDataResponse handle(DistroDataRequest request, RequestMeta meta) throws NacosException {
    try {
        switch (request.getDataOperation()) {
            case VERIFY:
                return handleVerify(request.getDistroData(), meta);
            case SNAPSHOT:
                return handleSnapshot();
            case ADD:
            case CHANGE:
            case DELETE:
                return handleSyncData(request.getDistroData());
            case QUERY:
                return handleQueryData(request.getDistroData());
            default:
                return new DistroDataResponse();
        }
    } catch (Exception e) {
        // ...
    }
}
```



```java
private DistroDataResponse handleVerify(DistroData distroData, RequestMeta meta) {
    DistroDataResponse result = new DistroDataResponse();
  	// 注解@12
    if (!distroProtocol.onVerify(distroData, meta.getClientIp())) {
        result.setErrorInfo(ResponseCode.FAIL.getCode(), "[DISTRO-FAILED] distro data verify failed");
    }
    return result;
}
```

**注解@12** 数据校验，下面可以看到，如果缓存存在client则校验成功，刷新client保鲜时间，否则校验失败。

```java
@Override
public boolean verifyClient(String clientId) {
    ConnectionBasedClient client = clients.get(clientId);
    if (null != client) {
        client.setLastRenewTime();
        return true;
    }
    return false;
}
```



**小结：** 节点之间发送校验数据是在全量同步后进行的；发送校验的频率默认为5秒钟一次；校验数据包括clientId和version，其中version为保留字段当前为0；接受到校验数据后如果缓存中存在该client表示校验成功，同时更新保鲜时间，否则校验失败。



# 全量数据同步



上文中提到在发送校验数据之前需要先完成全量数据同步，先翻回DistroProtocol#startDistroTask()方法的startLoadTask()部分。

```java
private void startLoadTask() {
    DistroCallback loadCallback = new DistroCallback() {
        @Override
        public void onSuccess() {
            isInitialized = true;
        }
        @Override
        public void onFailed(Throwable throwable) {
            isInitialized = false;
        }
    };
    GlobalExecutor.submitLoadDataTask(
            new DistroLoadDataTask(memberManager, distroComponentHolder, distroConfig, loadCallback));
}
```

DistroLoadDataTask#run

```java
@Override
public void run() {
  try {
    load(); // 注解@13
    if (!checkCompleted()) { // 注解@14
      GlobalExecutor.submitLoadDataTask(this, distroConfig.getLoadDataRetryDelayMillis());
    } else {
      loadCallback.onSuccess();
      Loggers.DISTRO.info("[DISTRO-INIT] load snapshot data success");
    }
  } catch (Exception e) {
    loadCallback.onFailed(e);
    Loggers.DISTRO.error("[DISTRO-INIT] load snapshot data failed. ", e);
  }
}
```

**注解@13** 从集群中其他节点全量加载数据

**注解@14** 如果没有加载成功延迟30秒钟重新执行一次，可以通过参数「nacos.core.protocol.distro.data.load_retry_delay_ms」指定

```java
private void load() throws Exception {
    while (memberManager.allMembersWithoutSelf().isEmpty()) {
        Loggers.DISTRO.info("[DISTRO-INIT] waiting server list init...");
        TimeUnit.SECONDS.sleep(1);
    }
    while (distroComponentHolder.getDataStorageTypes().isEmpty()) {
        Loggers.DISTRO.info("[DISTRO-INIT] waiting distro data storage register...");
        TimeUnit.SECONDS.sleep(1);
    }
    for (String each : distroComponentHolder.getDataStorageTypes()) { // 注解@15
        if (!loadCompletedMap.containsKey(each) || !loadCompletedMap.get(each)) {
            loadCompletedMap.put(each, loadAllDataSnapshotFromRemote(each)); // 加载快照
        }
    }
}
```

**注解@15** 为不同的数据类型缓存快照，此处有gRPC和http两类数据类型。即： Nacos:Naming:v2:ClientData和com.alibaba.nacos.naming.iplist.

```java
private boolean loadAllDataSnapshotFromRemote(String resourceType) {
    DistroTransportAgent transportAgent = distroComponentHolder.findTransportAgent(resourceType);
    DistroDataProcessor dataProcessor = distroComponentHolder.findDataProcessor(resourceType);
    if (null == transportAgent || null == dataProcessor) {
        return false;
    }
    for (Member each : memberManager.allMembersWithoutSelf()) { // 注解@16
        try {
          	DistroData distroData = transportAgent.getDatumSnapshot(each.getAddress());
            boolean result = dataProcessor.processSnapshot(distroData);
            if (result) {
                distroComponentHolder.findDataStorage(resourceType).finishInitial(); // 设置为完成初始化
                return true;
            }
        } catch (Exception e) {
           
        }
    }
    return false;
}
```

**注解@16** 获取集群中除了本节点的其他节点，循环重试获取快照，直到有成功节点返回快照，成功后设置状态状态完成初始化「finishInitial」。

```java
@Override
public DistroData getDatumSnapshot(String targetServer) {
    Member member = memberManager.find(targetServer);
    if (checkTargetServerStatusUnhealthy(member)) {
        throw new DistroException(
                String.format("[DISTRO] Cancel get snapshot caused by target server %s unhealthy", targetServer));
    }
    DistroDataRequest request = new DistroDataRequest();
  	// 设置请求操作为SNAPSHOT
    request.setDataOperation(DataOperation.SNAPSHOT); 
    try {
      	// 发起请求快照数据
        Response response = clusterRpcClientProxy.sendRequest(member, request);
        if (checkResponse(response)) {
            return ((DistroDataResponse) response).getDistroData();
        } else {
            throw new DistroException(
                    String.format("[DISTRO-FAILED] Get snapshot request to %s failed, code: %d, message: %s",
                            targetServer, response.getErrorCode(), response.getMessage()));
        }
    } catch (NacosException e) {
        throw new DistroException("[DISTRO-FAILED] Get distro snapshot failed! ", e);
    }
}
```

**接下来看看其他节点收到快照请求如何响应的** 



还是翻到DistroDataRequestHandler#handle，具体由handleSnapshot()方法来处理。

```java
private DistroDataResponse handleSnapshot() {
    DistroDataResponse result = new DistroDataResponse();
    DistroData distroData = distroProtocol.onSnapshot(DistroClientDataProcessor.TYPE);
    result.setDistroData(distroData);
    return result;
}
```

```java
@Override
public DistroData getDatumSnapshot() {
    List<ClientSyncData> datum = new LinkedList<>();
    // 把本节点的所有client数据全部封装
    for (String each : clientManager.allClientId()) {
        Client client = clientManager.getClient(each);
        if (null == client || !client.isEphemeral()) {
            continue;
        }
        datum.add(client.generateSyncData());
    }
    ClientSyncDatumSnapshot snapshot = new ClientSyncDatumSnapshot();
    snapshot.setClientSyncDataList(datum);
    byte[] data = ApplicationUtils.getBean(Serializer.class).serialize(snapshot); // 序列化数据
    return new DistroData(new DistroKey(DataOperation.SNAPSHOT.name(), TYPE), data);
}
```

下面看下client数据信息，命名空间、分组名称、服务名称、节点Instance信息（IP、端口等等）。

```java
public ClientSyncData generateSyncData() {
    List<String> namespaces = new LinkedList<>();
    List<String> groupNames = new LinkedList<>();
    List<String> serviceNames = new LinkedList<>();
    List<InstancePublishInfo> instances = new LinkedList<>();
    for (Map.Entry<Service, InstancePublishInfo> entry : publishers.entrySet()) {
        namespaces.add(entry.getKey().getNamespace());
        groupNames.add(entry.getKey().getGroup());
        serviceNames.add(entry.getKey().getName());
        instances.add(entry.getValue());
    }
    return new ClientSyncData(getClientId(), namespaces, groupNames, serviceNames, instances);
}
```



**小结：**集群中每个节点都拥有所有的快照数据； 在节点启动时会从集群中其他节点中的一个节点同步快照数据并缓存在Map中；缓存的数据类型分类两类分别为HTTP和gRPC；具体数据即客户端注册节点信息含命名空间、分组名称、服务名称、节点Instance信息等。



