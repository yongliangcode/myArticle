---

title: Nacos9# 服务端响应连接和注册源码分析（二）
categories: Nacos
tags: Nacos
date: 2021-07-03 11:55:01
---



# 引言

在《Nacos4# 服务端响应连接和注册源码分析（一）》在服务注册后发布了三个事件ClientEvent.ClientChangedEvent、ClientOperationEvent.ClientRegisterServiceEvent、MetadataEvent.InstanceMetadataEvent。这三个事件后来都干了点啥还没撸。

Nacos的CP协议使用Distro，中间穿插了几篇关于该协议的主要逻辑，本文接着撸服务端响应。



# 内容提要



### ClientRegisterServiceEvent事件

* 当注册请求到服务端时，服务端会给订阅该服务的Clients发送推送请求，通知实例变了

* 当注册请求到服务端时，服务端发布了客户端注册事件ClientRegisterServiceEvent
* ClientRegisterServiceEvent事件被ClientServiceIndexesManager订阅后发布服务变更事件ServiceChangedEvent
* ServiceChangedEvent被NamingSubscriberServiceV2Impl订阅，创建PushDelayTask被PushExecuteTask执行，负责向订阅该服务的订阅者发起推送serviceInfo请求
* 推送的请求被NamingPushRequestHandler处理并发布InstancesChangeEvent，最终回调到我们的代码逻辑AbstractEventListener



### ClientChangedEvent 事件

* 当注册请求到服务端时，该节点会向集群中其他节点增量同步新增的Client信息

* 当注册请求到服务端时，发布ClientChangedEvent事件
* 该事件被DistroClientDataProcessor订阅发起与其他节点的增量同步



### InstanceMetadataEvent事件

* 当注册请求到服务端时，发布ClientChangedEvent事件，属性expired为false
* NamingMetadataManager订阅了该事件主要判断元数据是否过期

<!--more-->

# ClientRegisterServiceEvent事件



代码翻到ClientServiceIndexesManager#subscribeTypes()，这里订阅ClientRegisterServiceEvent时间

```java
@Override
public List<Class<? extends Event>> subscribeTypes() {
    List<Class<? extends Event>> result = new LinkedList<>();
    result.add(ClientOperationEvent.ClientRegisterServiceEvent.class);
    result.add(ClientOperationEvent.ClientDeregisterServiceEvent.class);
    result.add(ClientOperationEvent.ClientSubscribeServiceEvent.class);
    result.add(ClientOperationEvent.ClientUnsubscribeServiceEvent.class);
    result.add(ClientEvent.ClientDisconnectEvent.class);
    return result;
}
```

当接收到事件会调用到onEvent()方法。

```java
@Override
public void onEvent(Event event) {
    if (event instanceof ClientEvent.ClientDisconnectEvent) {
        handleClientDisconnect((ClientEvent.ClientDisconnectEvent) event);
    } else if (event instanceof ClientOperationEvent) {
        handleClientOperation((ClientOperationEvent) event);
    }
}

private void handleClientOperation(ClientOperationEvent event) {
  Service service = event.getService();
  String clientId = event.getClientId();
  if (event instanceof ClientOperationEvent.ClientRegisterServiceEvent) {
    addPublisherIndexes(service, clientId); 
  } else if (event instanceof ClientOperationEvent.ClientDeregisterServiceEvent) {
    removePublisherIndexes(service, clientId);
  } else if (event instanceof ClientOperationEvent.ClientSubscribeServiceEvent) {
    addSubscriberIndexes(service, clientId);
  } else if (event instanceof ClientOperationEvent.ClientUnsubscribeServiceEvent) {
    removeSubscriberIndexes(service, clientId);
  }
}

 private void addPublisherIndexes(Service service, String clientId) {
   publisherIndexes.computeIfAbsent(service, (key) -> new ConcurrentHashSet<>());
   // 注解@1
   publisherIndexes.get(service).add(clientId);
   // 注解@2
   NotifyCenter.publishEvent(new ServiceEvent.ServiceChangedEvent(service, true));
 }
```

**注解@1** 一个服务通常有多个ClientId，clientId缓存在ConcurrentHashSet，通过ConcurrentHashMap关联。

**注解@2** 又发布了一个ServiceChangedEvent事件



谁订阅了服务变更事件ServiceChangedEvent？接着跟



代码翻到NamingSubscriberServiceV2Impl#subscribeTypes()，该类订阅了ServiceChangedEvent事件。

```java
@Override
public List<Class<? extends Event>> subscribeTypes() {
    List<Class<? extends Event>> result = new LinkedList<>();
    result.add(ServiceEvent.ServiceChangedEvent.class);
    result.add(ServiceEvent.ServiceSubscribedEvent.class);
    return result;
}
```



同样收到ServiceChangedEvent事件后回调到onEvent()方法。

```java
@Override
public void onEvent(Event event) {
    if (!upgradeJudgement.isUseGrpcFeatures()) {
        return;
    }
    if (event instanceof ServiceEvent.ServiceChangedEvent) {
        // If service changed, push to all subscribers.
        ServiceEvent.ServiceChangedEvent serviceChangedEvent = (ServiceEvent.ServiceChangedEvent) event;
        Service service = serviceChangedEvent.getService();
      	// 注解@3
        delayTaskEngine.addTask(service, new PushDelayTask(service, 500L));
    } else if (event instanceof ServiceEvent.ServiceSubscribedEvent) {
        // If service is subscribed by one client, only push this client.
        ServiceEvent.ServiceSubscribedEvent subscribedEvent = (ServiceEvent.ServiceSubscribedEvent) event;
        Service service = subscribedEvent.getService();
        delayTaskEngine.addTask(service, new PushDelayTask(service, 500L, subscribedEvent.getClientId()));
    }
}
```

**注解@3**  向delayTaskEngine引擎添加PushDelayTask，接着看该引擎的工作过程。

```java
public NacosDelayTaskExecuteEngine(String name, int initCapacity, Logger logger, long processInterval) {
    super(logger);
    tasks = new ConcurrentHashMap<>(initCapacity);
    processingExecutor = ExecutorFactory.newSingleScheduledExecutorService(new NameThreadFactory(name));
  	// 注解@4
    processingExecutor
            .scheduleWithFixedDelay(new ProcessRunnable(), processInterval, processInterval, TimeUnit.MILLISECONDS);
}
```

**注解@4** NacosDelayTaskExecuteEngine构造函数时启动了一个定时任务来运行，默认间隔为100ms。

```java
private static class PushDelayTaskProcessor implements NacosTaskProcessor {
    
    private final PushDelayTaskExecuteEngine executeEngine;
    
    public PushDelayTaskProcessor(PushDelayTaskExecuteEngine executeEngine) {
        this.executeEngine = executeEngine;
    }
    
    @Override
    public boolean process(NacosTask task) {
        PushDelayTask pushDelayTask = (PushDelayTask) task;
        Service service = pushDelayTask.getService();
      	// 注解@5
        NamingExecuteTaskDispatcher.getInstance()
                .dispatchAndExecuteTask(service, new PushExecuteTask(service, executeEngine, pushDelayTask));
        return true;
    }
}
```

**注解@5** 具体到PushDelayTask是由PushExecuteTask运行，下面看其run方法。

```java
public void run() {
    try {
    		// 注解@6
        PushDataWrapper wrapper = generatePushData();
        // 注解@7
        for (String each : getTargetClientIds()) {
            Client client = delayTaskEngine.getClientManager().getClient(each);
            if (null == client) {
                // means this client has disconnect
                continue;
            }
            // 注解@8
            Subscriber subscriber = delayTaskEngine.getClientManager().getClient(each).getSubscriber(service);
            // 注解@9
            delayTaskEngine.getPushExecutor().doPushWithCallback(each, subscriber, wrapper,
                    new NamingPushCallback(each, subscriber, wrapper.getOriginalData()));
        }
    } catch (Exception e) {
        Loggers.PUSH.error("Push task for service" + service.getGroupedServiceName() + " execute failed ", e);
        delayTaskEngine.addTask(service, new PushDelayTask(service, 1000L));
    }
}
```

**注解@6** 组织推送数据，主要为service信息以及注册host等。

**注解@7** 获取需要通知的客户端集合ClientIds

**注解@8** 获取服务的订阅者Subscriber

**注解@9** 根据clientId从connections集合中获取连接，将变更推送给客户端

**客户端如何接受的呢？** 

NamingGrpcClientProxy#start()

```java
private void start(ServerListFactory serverListFactory, ServiceInfoHolder serviceInfoHolder) throws NacosException {
    rpcClient.serverListFactory(serverListFactory);
    // gRPC Client启动
    rpcClient.start();
    // 注解@10
    rpcClient.registerServerRequestHandler(new NamingPushRequestHandler(serviceInfoHolder));
    // 注册连接事件Listener，当连接建立和断开时处理事件
    rpcClient.registerConnectionListener(namingGrpcConnectionEventListener);
}
```

**注解@10** 在客户端构建gRPC时，注册registerServerRequestHandler用于处理从Nacos Push到Client的请求，添加到了serverRequestHandlers集合。



GrpcClient#connectToServer()

```java
@Override
public Connection connectToServer(ServerInfo serverInfo) {
    try {
       
        RequestGrpc.RequestFutureStub newChannelStubTemp = createNewChannelStub(serverInfo.getServerIp(),
                serverInfo.getServerPort() + rpcPortOffset());
        if (newChannelStubTemp != null) {
            Response response = serverCheck(newChannelStubTemp);
            BiRequestStreamGrpc.BiRequestStreamStub biRequestStreamStub = BiRequestStreamGrpc
                    .newStub(newChannelStubTemp.getChannel());
            GrpcConnection grpcConn = new GrpcConnection(serverInfo, grpcExecutor);
            grpcConn.setConnectionId(((ServerCheckResponse) response).getConnectionId());
            //create stream request and bind connection event to this connection.
            // 注解@11
            StreamObserver<Payload> payloadStreamObserver = bindRequestStream(biRequestStreamStub, grpcConn);

            // ...
            return grpcConn;
        }
        return null;
    } catch (Exception e) {
       // ...
    }
    return null;
}
```

**注解@11** 在连接server时绑定相关事件

```java
private StreamObserver<Payload> bindRequestStream(final BiRequestStreamGrpc.BiRequestStreamStub streamStub,
        final GrpcConnection grpcConn) {

    return streamStub.requestBiStream(new StreamObserver<Payload>() {

        @Override
        public void onNext(Payload payload) {

            LoggerUtils.printIfDebugEnabled(LOGGER, "[{}]Stream server request receive, original info: {}",
                    grpcConn.getConnectionId(), payload.toString());
            try {
                Object parseBody = GrpcUtils.parse(payload);
                final Request request = (Request) parseBody;
                if (request != null) {

                    try {
                        // 注解@12
                        Response response = handleServerRequest(request);
                        if (response != null) {
                            response.setRequestId(request.getRequestId());
                            sendResponse(response);
                        } else {
                          	
                        }

                    } catch (Exception e) {
                        
                    }
                }

            } catch (Exception e) {
							
            }
        }

    });
}
```

**注解@12**  接受server push处理，本事件具体回调到NamingPushRequestHandler#requestReply

```java
@Override
public Response requestReply(Request request) {
    if (request instanceof NotifySubscriberRequest) {
        NotifySubscriberRequest notifyResponse = (NotifySubscriberRequest) request;
        serviceInfoHolder.processServiceInfo(notifyResponse.getServiceInfo());
        return new NotifySubscriberResponse();
    }
    return null;
}
```

下面这部分代码在以前的文章已经分析过了，当serviceInfo变更时发布InstancesChangeEvent，InstancesChangeNotifier订阅了该事件，进而回调到我们示例代码的AbstractEventListener实现中。

```java
public ServiceInfo processServiceInfo(ServiceInfo serviceInfo) {
    String serviceKey = serviceInfo.getKey();
    if (serviceKey == null) {
        return null;
    }
    ServiceInfo oldService = serviceInfoMap.get(serviceInfo.getKey());
    if (isEmptyOrErrorPush(serviceInfo)) {
        //empty or error push, just ignore
        return oldService;
    }
    // 缓存服务信息
    serviceInfoMap.put(serviceInfo.getKey(), serviceInfo);
    // 判断注册的实例信息是否已变更
    boolean changed = isChangedServiceInfo(oldService, serviceInfo);
    if (StringUtils.isBlank(serviceInfo.getJsonFromServer())) {
        serviceInfo.setJsonFromServer(JacksonUtils.toJson(serviceInfo));
    }
    // 通过prometheus-simpleclient监控服务缓存Map的大小
    MetricsMonitor.getServiceInfoMapSizeMonitor().set(serviceInfoMap.size());
    // 服务实例已变更
    if (changed) {
        NAMING_LOGGER.info("current ips:(" + serviceInfo.ipCount() + ") service: " + serviceInfo.getKey() + " -> "
                + JacksonUtils.toJson(serviceInfo.getHosts()));
        // 添加实例变更事件，会被推动到订阅者执行
        NotifyCenter.publishEvent(new InstancesChangeEvent(serviceInfo.getName(), serviceInfo.getGroupName(),
                serviceInfo.getClusters(), serviceInfo.getHosts()));
        // 记录Service本地文件
        DiskCache.write(serviceInfo, cacheDir);
    }
    return serviceInfo;
}
```



**小结：** 当注册请求到服务端时，服务端发布了客户端注册事件（ClientRegisterServiceEvent）；ClientRegisterServiceEvent事件被ClientServiceIndexesManager订阅后发布服务变更事件（ServiceChangedEvent）；ServiceChangedEvent被NamingSubscriberServiceV2Impl订阅并创建PushDelayTask并被PushExecuteTask执行，负责向订阅该服务的订阅者发起推送serviceInfo请求；推送的请求被NamingPushRequestHandler处理并发布InstancesChangeEvent，最终回调到我们的代码逻辑AbstractEventListener。



# ClientChangedEvent 事件

还是看ClientChangedEvent事件的订阅者，代码翻到DistroClientDataProcessor#subscribeTypes()。

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

当收到该事件时会执行到onEvent方法()也就是增量同步，增量同步逻辑在《Nacos7# Distro协议增量同步》已梳理。

```
@Override
public void onEvent(Event event) {
    if (EnvUtil.getStandaloneMode()) {
        return;
    }
    if (!upgradeJudgement.isUseGrpcFeatures()) {
        return;
    }
    if (event instanceof ClientEvent.ClientVerifyFailedEvent) {
        syncToVerifyFailedServer((ClientEvent.ClientVerifyFailedEvent) event);
    } else {
        // 同步给其他节点信息
        syncToAllServer((ClientEvent) event);
    }
}
```

**小结：**  当注册请求到服务端时，发布ClientChangedEvent事件；该事件被DistroClientDataProcessor订阅发起与其他节点的增量同步。



# InstanceMetadataEvent事件

翻到NamingMetadataManager#subscribeTypes()，订阅了该事件。

```java
@Override
public List<Class<? extends Event>> subscribeTypes() {
    List<Class<? extends Event>> result = new LinkedList<>();
    // 订阅实例变更事件
    result.add(MetadataEvent.InstanceMetadataEvent.class);
    result.add(MetadataEvent.ServiceMetadataEvent.class);
    result.add(ClientEvent.ClientDisconnectEvent.class);
    return result;
}
```

还是onEvent()处理事件

```java
public void onEvent(Event event) {
    // 处理实例元数据变更事件
    if (event instanceof MetadataEvent.InstanceMetadataEvent) {
        handleInstanceMetadataEvent((MetadataEvent.InstanceMetadataEvent) event);
    } else if (event instanceof MetadataEvent.ServiceMetadataEvent) {
        handleServiceMetadataEvent((MetadataEvent.ServiceMetadataEvent) event);
    } else {
        handleClientDisconnectEvent((ClientEvent.ClientDisconnectEvent) event);
    }
}

private void handleInstanceMetadataEvent(MetadataEvent.InstanceMetadataEvent event) {
        Service service = event.getService();
        // 格式 ip:port:cluster
        String metadataId = event.getMetadataId();
        if (containInstanceMetadata(service, metadataId)) {
            updateExpiredInfo(event.isExpired(),
                    ExpiredMetadataInfo.newExpiredInstanceMetadata(event.getService(), event.getMetadataId()));
        }
 }

private void updateExpiredInfo(boolean expired, ExpiredMetadataInfo expiredMetadataInfo) {
        if (expired) {
            expiredMetadataInfos.add(expiredMetadataInfo);
        } else {  // false
            expiredMetadataInfos.remove(expiredMetadataInfo);
        }
 }
```

**小结：** 当注册请求到服务端时，发布ClientChangedEvent事件，expired为false；NamingMetadataManager订阅了该事件主要判断元数据是否过期。





