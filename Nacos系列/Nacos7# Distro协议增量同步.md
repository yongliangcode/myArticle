---
title: Nacos7# Distro协议增量同步
categories: Nacos
tags: Nacos
date: 2021-07-03 11:55:01
---

# 引言



本文接着撸Distro协议，上文中分析了在Nacos server启动时会进行全量数据同步和数据校验，具体数据即客户端注册节点信息含命名空间、分组名称、服务名称、节点Instance信息等。什么时候会触发增量同步？增量同步都干了些啥，下文接着撸撸增量数据同步。



# 内容提要



### 增量数据同步

* 在Nacos节点启动时通过事件驱动模式订阅了ClientChangedEvent、ClientDisconnectEvent和ClientVerifyFailedEvent事件
* 当节点收到ClientChangedEvent事件时，会向集群中其他节点发送更新Client信息请求，其他节点收到后更新缓存
* 当节点收到ClientVerifyFailedEvent事件时，向该Event指定的目标节点发起新增该Event指定的Client信息请求，目标节点收到后更新到自己缓存中
* 当节点收到ClientDisconnectEvent事件时，会向集群中其他节点发送删除Client信息请求，其他节点收到后将该Client缓存删除



### 增量事件触发

* 当有服务注册或者注销时会触发ClientEvent.ClientChangedEvent事件，即客户端调用naming.registerInstance或者naming.deregisterInstance
* 定时任务每隔3秒钟定时检查缓存中的所有连接，如果超过保鲜期20秒则再次发起连接请求，连接未成功则注销关闭该连接并发布ClientEvent.ClientDisconnectEvent事件
* Nacos集群之间通过每5秒发送心跳校验数据请求（具体为本节点负责Client信息），其他节点接受到校验请求，如果缓存中存在该client表示校验成功，同时更新保鲜时间；否则校验失败，回调返回失败Response，请求节点收到失败的Response后会发布ClientVerifyFailedEvent事件



<!--more-->



# 增量数据同步



将代码翻到DistroClientDataProcessor类中，该类继承了SmartSubscriber，遵循Subscriber/Notify模式，即事件驱动模式。该模式前面文章中分析过，当有订阅的事件时会进行回调通知。



### 订阅的事件

DistroClientDataProcessor订阅了ClientChangedEvent、ClientDisconnectEvent和ClientVerifyFailedEvent事件。

```java
@Override
public List<Class<? extends Event>> subscribeTypes() {
  List<Class<? extends Event>> result = new LinkedList<>();
  result.add(ClientEvent.ClientChangedEvent.class);
  result.add(ClientEvent.ClientDisconnectEvent.class);
  result.add(ClientEvent.ClientVerifyFailedEvent.class);
  return result;
}
```

当有上述三个事件产生时，DefaultPublisher回调onEvent方法。

```java
public void onEvent(Event event) {
    if (EnvUtil.getStandaloneMode()) {
        return;
    }
    if (!upgradeJudgement.isUseGrpcFeatures()) {
        return;
    }
    if (event instanceof ClientEvent.ClientVerifyFailedEvent) {
      	// 注解@1
        syncToVerifyFailedServer((ClientEvent.ClientVerifyFailedEvent) event);
    } else {
        // 注解@2
        syncToAllServer((ClientEvent) event);
    }
}
```

**注解@1** 将ClientVerifyFailedEvent同步给校验失败的节点，操作类型为ADD

**注解@2** 将同步给集群中的其他节

```java
private void syncToAllServer(ClientEvent event) {
    Client client = event.getClient();
    // Only ephemeral data sync by Distro, persist client should sync by raft.
    if (null == client || !client.isEphemeral() || !clientManager.isResponsibleClient(client)) {
        return;
    }
    if (event instanceof ClientEvent.ClientDisconnectEvent) {
      	// 注解@3
        DistroKey distroKey = new DistroKey(client.getClientId(), TYPE);
        distroProtocol.sync(distroKey, DataOperation.DELETE);
    } else if (event instanceof ClientEvent.ClientChangedEvent) {
      	// 注解@4
        DistroKey distroKey = new DistroKey(client.getClientId(), TYPE);
        distroProtocol.sync(distroKey, DataOperation.CHANGE);
    }
}
```

**注解@3** 当客户端断开连接事件ClientDisconnectEvent时，向其他节点同步DELETE操作

**注解@4** 当客户端变更事件ClientChangedEvent时，向其他节点同步CHANGE操作

接着看下不同操作类型的处理

```java
@Override
public boolean process(NacosTask task) {
    if (!(task instanceof DistroDelayTask)) {
        return true;
    }
    DistroDelayTask distroDelayTask = (DistroDelayTask) task;
    DistroKey distroKey = distroDelayTask.getDistroKey();
    switch (distroDelayTask.getAction()) {
        case DELETE: // 删除操作
            DistroSyncDeleteTask syncDeleteTask = new DistroSyncDeleteTask(distroKey, distroComponentHolder);
            distroTaskEngineHolder.getExecuteWorkersManager().addTask(distroKey, syncDeleteTask);
            return true;
        case CHANGE:
        case ADD: // 更新和新增操作
            DistroSyncChangeTask syncChangeTask = new DistroSyncChangeTask(distroKey, distroComponentHolder);
            distroTaskEngineHolder.getExecuteWorkersManager().addTask(distroKey, syncChangeTask);
            return true;
        default:
            return false;
    }
}
```

向指定的集群节点同步更新数据

```java
@Override
public boolean syncData(DistroData data, String targetServer) {
    if (isNoExistTarget(targetServer)) {
        return true;
    }
    // 构造请求数据并设置数据类型
    DistroDataRequest request = new DistroDataRequest(data, data.getType());
    // 查找目标节点缓存数据
    Member member = memberManager.find(targetServer);
    // 节点状态检查需UP状态，即：可通信状态
    if (checkTargetServerStatusUnhealthy(member)) {
        Loggers.DISTRO.warn("[DISTRO] Cancel distro sync caused by target server {} unhealthy", targetServer);
        return false;
    }
    try {
        // 向目标节点发送数据
        Response response = clusterRpcClientProxy.sendRequest(member, request);
        return checkResponse(response);
    } catch (NacosException e) {
        Loggers.DISTRO.error("[DISTRO-FAILED] Sync distro data failed! ", e);
    }
    return false;
}
```

异步更新操作

```java
@Override
public void syncData(DistroData data, String targetServer, DistroCallback callback) {
    if (isNoExistTarget(targetServer)) {
        callback.onSuccess();
    }
    DistroDataRequest request = new DistroDataRequest(data, data.getType());
    Member member = memberManager.find(targetServer);
    try {
        // 异步更新操作
        clusterRpcClientProxy.asyncRequest(member, request, new DistroRpcCallbackWrapper(callback, member));
    } catch (NacosException nacosException) {
        callback.onFailed(nacosException);
    }
}
```



**节点收到这些操作请求如何处理呢？** 



代码翻到DistroDataRequestHandler#handle()，集群中节点收到请求后处理逻辑在这里：

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
        Loggers.DISTRO.error("[DISTRO-FAILED] distro handle with exception", e);
        DistroDataResponse result = new DistroDataResponse();
        result.setErrorCode(ResponseCode.FAIL.getCode());
        result.setMessage("handle distro request with exception");
        return result;
    }
}
```

可以看出ADD、CHANGE和DELETE均由handleSyncData处理。

```java
private DistroDataResponse handleSyncData(DistroData distroData) {
    DistroDataResponse result = new DistroDataResponse();
    if (!distroProtocol.onReceive(distroData)) {
        result.setErrorCode(ResponseCode.FAIL.getCode());
        result.setMessage("[DISTRO-FAILED] distro data handle failed");
    }
    return result;
}
```

```java
@Override
public boolean processData(DistroData distroData) {
    switch (distroData.getType()) {
        case ADD:
        case CHANGE:
            ClientSyncData clientSyncData = ApplicationUtils.getBean(Serializer.class)
                    .deserialize(distroData.getContent(), ClientSyncData.class);
            handlerClientSyncData(clientSyncData); // 注解@5
            return true;
        case DELETE:
            String deleteClientId = distroData.getDistroKey().getResourceKey();
            Loggers.DISTRO.info("[Client-Delete] Received distro client sync data {}", deleteClientId);
            clientManager.clientDisconnected(deleteClientId); // 注解@6
            return true;
        default:
            return false;
    }
}
```

**注解@5** 将同步过来的Client信息进行缓存

```java
private void handlerClientSyncData(ClientSyncData clientSyncData) {
    Loggers.DISTRO.info("[Client-Add] Received distro client sync data {}", clientSyncData.getClientId());
    clientManager.syncClientConnected(clientSyncData.getClientId(), clientSyncData.getAttributes());
    Client client = clientManager.getClient(clientSyncData.getClientId());
  	// 注解@5.1
    upgradeClient(client, clientSyncData);
}
```

需要的是从其他节点通过过来的Client信息，ConnectionBasedClient属性isNative为false表示该连接时从其他节点同步过来的；true表示该连接客户端直接连接的。

```java
public boolean syncClientConnected(String clientId, ClientSyncAttributes attributes) {
    String type = attributes.getClientAttribute(ClientConstants.CONNECTION_TYPE);
    ClientFactory clientFactory = ClientFactoryHolder.getInstance().findClientFactory(type);
    return clientConnected(clientFactory.newSyncedClient(clientId, attributes));
}

@Override
public ConnectionBasedClient newSyncedClient(String clientId, ClientSyncAttributes attributes) {
  return new ConnectionBasedClient(clientId, false); // false表示从其他节点同步过来的client
}

@Override
public boolean clientConnected(Client client) {
  Loggers.SRV_LOG.info("Client connection {} connect", client.getClientId());
  if (!clients.containsKey(client.getClientId())) {
    clients.putIfAbsent(client.getClientId(), (ConnectionBasedClient) client); // 缓存client
  }
  return true;
}
```

**注解@5.1**  更新Client的Service以及Instance信息。

```java
private void upgradeClient(Client client, ClientSyncData clientSyncData) {

    List<String> namespaces = clientSyncData.getNamespaces();
    List<String> groupNames = clientSyncData.getGroupNames();
    List<String> serviceNames = clientSyncData.getServiceNames();
    List<InstancePublishInfo> instances = clientSyncData.getInstancePublishInfos();
    Set<Service> syncedService = new HashSet<>();
    for (int i = 0; i < namespaces.size(); i++) {
        Service service = Service.newService(namespaces.get(i), groupNames.get(i), serviceNames.get(i));
        Service singleton = ServiceManager.getInstance().getSingleton(service);
        syncedService.add(singleton);
        InstancePublishInfo instancePublishInfo = instances.get(i);
        if (!instancePublishInfo.equals(client.getInstancePublishInfo(singleton))) {
            client.addServiceInstance(singleton, instancePublishInfo);
            NotifyCenter.publishEvent(
                    new ClientOperationEvent.ClientRegisterServiceEvent(singleton, client.getClientId()));
        }
    }
    for (Service each : client.getAllPublishedService()) {
        if (!syncedService.contains(each)) {
            client.removeServiceInstance(each);
            NotifyCenter.publishEvent(
                    new ClientOperationEvent.ClientDeregisterServiceEvent(each, client.getClientId()));
        }
    }
}
```

**注解@6** 响应删除操作，从clients缓存中移除。

```java
@Override
public boolean clientDisconnected(String clientId) {
    Loggers.SRV_LOG.info("Client connection {} disconnect, remove instances and subscribers", clientId);
    ConnectionBasedClient client = clients.remove(clientId);
    if (null == client) {
        return true;
    }
    client.release();
    NotifyCenter.publishEvent(new ClientEvent.ClientDisconnectEvent(client));
    return true;
}
```



**小结：** 增量同步的逻辑如下：当本节点DistroClientDataProcessor收到ClientChangedEvent、ClientDisconnectEvent和ClientVerifyFailedEvent事件时，会向Nacos集群的其他节点同步Client信息；集群中其他节点收到同步信息后更新或者删除本地缓存的Client信息；通过增量同步的Client信息isNative为false表示不是由客户端直连的。



# 增量事件触发

在Nacos server启动时从运行时内存信息可以看出，总共缓存了17个事件类型。当然也包括ClientChangedEvent、ClientDisconnectEvent和ClientVerifyFailedEvent。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210629174459.png)



### ClientChangedEvent事件触发



当处理服务注册和注销事件时会触发ClientChangeEvent事件，详见InstanceRequestHandler#handle处理逻辑。

```java
public InstanceResponse handle(InstanceRequest request, RequestMeta meta) throws NacosException {
    Service service = Service
            .newService(request.getNamespace(), request.getGroupName(), request.getServiceName(), true);
    switch (request.getType()) {
        // 注解@7
        case NamingRemoteConstants.REGISTER_INSTANCE:
            return registerInstance(service, request, meta);
        // 注解@8
        case NamingRemoteConstants.DE_REGISTER_INSTANCE:
            return deregisterInstance(service, request, meta);
        default:
            throw new NacosException(NacosException.INVALID_PARAM,
                    String.format("Unsupported request type %s", request.getType()));
    }
}
```

**注解@7** 处理注册请求，会调用到addServiceInstance方法，该方法中发布了ClientEvent.ClientChangedEvent事件。

```java
public boolean addServiceInstance(Service service, InstancePublishInfo instancePublishInfo) {
    if (null == publishers.put(service, instancePublishInfo)) {
        MetricsMonitor.incrementInstanceCount();
    }
    NotifyCenter.publishEvent(new ClientEvent.ClientChangedEvent(this));
    Loggers.SRV_LOG.info("Client change for service {}, {}", service, getClientId());
    return true;
}
```

**注解@8** 处理注销请求，会调用到removeServiceInstance方法，该方法中发布了ClientEvent.ClientChangedEvent事件

```java
public InstancePublishInfo removeServiceInstance(Service service) {
        InstancePublishInfo result = publishers.remove(service);
        if (null != result) {
            MetricsMonitor.decrementInstanceCount();
            NotifyCenter.publishEvent(new ClientEvent.ClientChangedEvent(this));
        }
        Loggers.SRV_LOG.info("Client remove for service {}, {}", service, getClientId());
        return result;
}
```



**小结：** 当有服务注册或者注销时会触发ClientEvent.ClientChangedEvent事件。



### ClientDisconnectEvent事件触发

下面一段代码通过检测连接是否超过保鲜期，超过保鲜期的会被注销关闭，翻到代码ConnectionManager#start()。

```java
@PostConstruct
public void start() {
    // 定时任务每3秒执行一次
    RpcScheduledExecutor.COMMON_SERVER_EXECUTOR.scheduleWithFixedDelay(new Runnable() {
        @Override
        public void run() {
            try {
                // 获取缓存连接
                int totalCount = connections.size();
                Loggers.REMOTE_DIGEST.info("Connection check task start");
                MetricsMonitor.getLongConnectionMonitor().set(totalCount);
                // 所有连接集合
                Set<Map.Entry<String, Connection>> entries = connections.entrySet();
                // 获取通过SDK连接的数量
                int currentSdkClientCount = currentSdkClientCount();
                boolean isLoaderClient = loadClient >= 0;
                int currentMaxClient = isLoaderClient ? loadClient : connectionLimitRule.countLimit;

                int expelCount = currentMaxClient < 0 ? 0 : Math.max(currentSdkClientCount - currentMaxClient, 0);

                List<String> expelClient = new LinkedList<>();

                Map<String, AtomicInteger> expelForIp = new HashMap<>(16);

                // 1. calculate expel count  of ip.
                // 加载Connection ConnectionLimitRule
                // 默认路径为 ${usr.home}/nacos/data/loader/limitRule
                for (Map.Entry<String, Connection> entry : entries) {

                    Connection client = entry.getValue();
                    String appName = client.getMetaInfo().getAppName();
                    String clientIp = client.getMetaInfo().getClientIp();
                    if (client.getMetaInfo().isSdkSource() && !expelForIp.containsKey(clientIp)) {
                        //get limit for current ip.
                        // 默认无limit限制
                        int countLimitOfIp = connectionLimitRule.getCountLimitOfIp(clientIp);
                        // 默认无limit限制
                        if (countLimitOfIp < 0) {
                            int countLimitOfApp = connectionLimitRule.getCountLimitOfApp(appName);
                            countLimitOfIp = countLimitOfApp < 0 ? countLimitOfIp : countLimitOfApp;
                        }
                        if (countLimitOfIp < 0) { // 默认无限制
                            countLimitOfIp = connectionLimitRule.getCountLimitPerClientIpDefault();
                        }

                        if (countLimitOfIp >= 0 && connectionForClientIp.containsKey(clientIp)) {
                            AtomicInteger currentCountIp = connectionForClientIp.get(clientIp);
                            if (currentCountIp != null && currentCountIp.get() > countLimitOfIp) {
                                expelForIp.put(clientIp, new AtomicInteger(currentCountIp.get() - countLimitOfIp));
                            }
                        }
                    }
                }

                if (expelForIp.size() > 0) { // 默认等于0
                    Loggers.REMOTE_DIGEST.info("Over limit ip expel info,", expelForIp);
                }

                Set<String> outDatedConnections = new HashSet<>();
                long now = System.currentTimeMillis();
                // 2.get expel connection for ip limit.
                //
                for (Map.Entry<String, Connection> entry : entries) {
                    Connection client = entry.getValue();
                    String clientIp = client.getMetaInfo().getClientIp();
                    AtomicInteger integer = expelForIp.get(clientIp);
                    if (integer != null && integer.intValue() > 0) {
                        integer.decrementAndGet();
                        expelClient.add(client.getMetaInfo().getConnectionId());
                        expelCount--;
                    } else if (now - client.getMetaInfo().getLastActiveTime() >= KEEP_ALIVE_TIME) { // 保鲜时间超过20秒放入outDatedConnections集合
                        outDatedConnections.add(client.getMetaInfo().getConnectionId());
                    }

                }

                // 3. if total count is still over limit.
                // expelCount 默认为0
                if (expelCount > 0) {
                    for (Map.Entry<String, Connection> entry : entries) {
                        Connection client = entry.getValue();
                        if (!expelForIp.containsKey(client.getMetaInfo().clientIp) && client.getMetaInfo()
                                .isSdkSource() && expelCount > 0) {
                            expelClient.add(client.getMetaInfo().getConnectionId());
                            expelCount--;
                            outDatedConnections.remove(client.getMetaInfo().getConnectionId());
                        }
                    }
                }

                String serverIp = null;
                String serverPort = null;
                if (StringUtils.isNotBlank(redirectAddress) && redirectAddress.contains(Constants.COLON)) {
                    String[] split = redirectAddress.split(Constants.COLON);
                    serverIp = split[0];
                    serverPort = split[1];
                }

                for (String expelledClientId : expelClient) { // 默认空集合
                    try {
                        Connection connection = getConnection(expelledClientId);
                        if (connection != null) {
                            ConnectResetRequest connectResetRequest = new ConnectResetRequest();
                            connectResetRequest.setServerIp(serverIp);
                            connectResetRequest.setServerPort(serverPort);
                            connection.asyncRequest(connectResetRequest, null);
                        }

                    } catch (ConnectionAlreadyClosedException e) {
                        unregister(expelledClientId);
                    } catch (Exception e) {
                        Loggers.REMOTE_DIGEST.error("Error occurs when expel connection :", expelledClientId, e);
                    }
                }

                //4.client active detection.
                Loggers.REMOTE_DIGEST.info("Out dated connection ,size={}", outDatedConnections.size());
                // 超过保鲜期的链接集合
                if (CollectionUtils.isNotEmpty(outDatedConnections)) {
                    Set<String> successConnections = new HashSet<>();
                    final CountDownLatch latch = new CountDownLatch(outDatedConnections.size());
                    for (String outDateConnectionId : outDatedConnections) {
                        try {
                            Connection connection = getConnection(outDateConnectionId);
                            if (connection != null) {
                                ClientDetectionRequest clientDetectionRequest = new ClientDetectionRequest();
                                // 超过保鲜时间的连接，重新异步发起连接
                                connection.asyncRequest(clientDetectionRequest, new RequestCallBack() {
                                    @Override
                                    public Executor getExecutor() {
                                        return null;
                                    }

                                    @Override
                                    public long getTimeout() {
                                        return 1000L;
                                    }

                                    @Override
                                    public void onResponse(Response response) {
                                        latch.countDown();
                                        if (response != null && response.isSuccess()) {
                                            // 刷新激活时间
                                            connection.freshActiveTime();
                                            successConnections.add(outDateConnectionId);
                                        }
                                    }

                                    @Override
                                    public void onException(Throwable e) {
                                        latch.countDown();
                                    }
                                });
							
                            } else {
                                latch.countDown();
                            }

                        } catch (ConnectionAlreadyClosedException e) {
                            latch.countDown();
                        } catch (Exception e) {
                           	// ... 
                            latch.countDown();
                        }
                    }

                    latch.await(3000L, TimeUnit.MILLISECONDS);
                    Loggers.REMOTE_DIGEST
                            .info("Out dated connection check successCount={}", successConnections.size());

                    // 无效连接集合
                    for (String outDateConnectionId : outDatedConnections) {
                        if (!successConnections.contains(outDateConnectionId)) {
                            Loggers.REMOTE_DIGEST
                                    .info("[{}]Unregister Out dated connection....", outDateConnectionId);
                            // 注销关闭connection
                            unregister(outDateConnectionId);
                        }
                    }
                }

                if (isLoaderClient) {  // 重置
                    loadClient = -1;
                    redirectAddress = null;
                }

            } catch (Throwable e) {
               
            }
        }
    }, 1000L, 3000L, TimeUnit.MILLISECONDS);

}
```



```java
public synchronized void unregister(String connectionId) {
    Connection remove = this.connections.remove(connectionId);
    if (remove != null) {
        String clientIp = remove.getMetaInfo().clientIp;
        AtomicInteger atomicInteger = connectionForClientIp.get(clientIp);
        if (atomicInteger != null) {
            int count = atomicInteger.decrementAndGet();
            if (count <= 0) {
                connectionForClientIp.remove(clientIp);
            }
        }
        remove.close();
        Loggers.REMOTE_DIGEST.info("[{}]Connection unregistered successfully. ", connectionId);
        clientConnectionEventListenerRegistry.notifyClientDisConnected(remove); // 异步
    }
}

public void notifyClientDisConnected(final Connection connection) {
        
        for (ClientConnectionEventListener clientConnectionEventListener : clientConnectionEventListeners) {
            try {
                clientConnectionEventListener.clientDisConnected(connection);
            } catch (Throwable throwable) {
                Loggers.REMOTE.info("[NotifyClientDisConnected] failed for listener {}",
                        clientConnectionEventListener.getName(), throwable);
            }
        }
        
 }

@Override
public boolean clientDisconnected(String clientId) {
  Loggers.SRV_LOG.info("Client connection {} disconnect, remove instances and subscribers", clientId);
  ConnectionBasedClient client = clients.remove(clientId);
  if (null == client) {
    return true;
  }
  client.release();
  NotifyCenter.publishEvent(new ClientEvent.ClientDisconnectEvent(client)); // 发布ClientDisconnectEvent事件
  return true;
}
```



**小结：** 连接可以配置限制规则具体在${usr.home}/nacos/data/loader/limitRule文件配置，默认无限制；通过定时任务每隔3秒钟定时检查缓存中的所有连接包括通过来源sdk的连接和集群的连接；如果连接超过保鲜期20秒，并再次发起连接请求，未能连接成功则注销关闭该连接；注销关闭时发布ClientEvent.ClientDisconnectEvent事件。



### ClientVerifyFailedEvent事件触发



上一篇文章中梳理了Nacos集群中，每个节点会对集群中其他节点每隔5秒发送校验数据，也就是心跳。当校验的结果会进行回调（gRPC为例），我们翻着看看这部分。

```java
public void syncVerifyData(DistroData verifyData, String targetServer, DistroCallback callback) {
    if (isNoExistTarget(targetServer)) {
        callback.onSuccess();
    }
    DistroDataRequest request = new DistroDataRequest(verifyData, DataOperation.VERIFY);
    Member member = memberManager.find(targetServer);
    try {
        DistroVerifyCallbackWrapper wrapper = new DistroVerifyCallbackWrapper(targetServer,
                verifyData.getDistroKey().getResourceKey(), callback, member);
        clusterRpcClientProxy.asyncRequest(member, request, wrapper); // 向其他节点发送本节点负责的cleintId信息
    } catch (NacosException nacosException) {
        callback.onFailed(nacosException);
    }
}
```

重点看下DistroVerifyCallbackWrapper部分，校验失败发布ClientVerifyFailedEvent事件。

```java
@Override
public void onResponse(Response response) {
    if (checkResponse(response)) {
        NamingTpsMonitor.distroVerifySuccess(member.getAddress(), member.getIp());
        distroCallback.onSuccess();
    } else {
        Loggers.DISTRO.info("Target {} verify client {} failed, sync new client", targetServer, clientId);
      	// 校验失败发布ClientVerifyFailedEvent事件
        NotifyCenter.publishEvent(new ClientEvent.ClientVerifyFailedEvent(clientId, targetServer));
        NamingTpsMonitor.distroVerifyFail(member.getAddress(), member.getIp());
        distroCallback.onFailed(null);
    }
}
```

最后看下ClientVerifyFailedEvent这个类，关注下成员变量包含了clientId和targetServer。当收到ClientVerifyFailedEvent时用于向targetServer目标节点添加客户端clientId信息。

```java
public static class ClientVerifyFailedEvent extends ClientEvent {

    private static final long serialVersionUID = 2023951686223780851L;

    private final String clientId;
    
    private final String targetServer;
    
    public ClientVerifyFailedEvent(String clientId, String targetServer) {
        super(null);
        this.clientId = clientId;
        this.targetServer = targetServer;
    }
    
    public String getClientId() {
        return clientId;
    }
    
    public String getTargetServer() {
        return targetServer;
    }
}
```



**小结：** Nacos集群之间通过每5秒发送心跳校验数据请求（具体为本节点负责Client信息），其他节点接受到校验请求，如果缓存中存在该client表示校验成功，同时更新保鲜时间；否则校验失败，回调返回失败Response，请求节点收到失败的Response后会发布ClientVerifyFailedEvent事件。



















