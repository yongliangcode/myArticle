---
title: Nacos4# 服务端响应连接和注册源码分析（一）
categories: Nacos
tags: Nacos
date: 2021-06-13 11:55:01

---



# 引言

上篇文章分析了Nacos服务端启动的逻辑，本文分析启动时加载了哪些Handler，以及处理连接请求和注册请求的逻辑。



# 内容提要



### 加载RequestHandler

* 容器启动时加载了13个RequestHandler由他们实际处理各种请求逻辑后面章节再细撸
* 当然也加载了InstanceRequest->InstanceRequestHandler



### 服务端响应客户端连接

* 处理客户端的连接请求由GrpcBiStreamRequestAcceptor#requestBiStream负责
* 构建和缓存了GrpcConnection对象和Client对象



### 服务端处理注册请求

*  处理注册请求的逻辑由InstanceRequestHandler执行
*  建立起了client与service、instance的关系
* 发布了三个事件ClientEvent.ClientChangedEvent、ClientOperationEvent.ClientRegisterServiceEvent、MetadataEvent.InstanceMetadataEvent



# 加载RequestHandler

com.alibaba.nacos.core.remote#RequestHandlerRegistry实现了ApplicationListener，在spring启动后回调onApplicationEvent。

```java
@Override
public void onApplicationEvent(ContextRefreshedEvent event) {
  // 注解@1
  Map<String, RequestHandler> beansOfType = event.getApplicationContext().getBeansOfType(RequestHandler.class);
  Collection<RequestHandler> values = beansOfType.values();
  for (RequestHandler requestHandler : values) {

    Class<?> clazz = requestHandler.getClass();
    boolean skip = false;
    while (!clazz.getSuperclass().equals(RequestHandler.class)) {
      if (clazz.getSuperclass().equals(Object.class)) {
        skip = true;
        break;
      }
      clazz = clazz.getSuperclass();
    }
    if (skip) { // 注解@2 
      continue;
    }

    try {
      // 注解@3
      Method method = clazz.getMethod("handle", Request.class, RequestMeta.class);
      if (method.isAnnotationPresent(TpsControl.class) && TpsControlConfig.isTpsControlEnabled()) {
        TpsControl tpsControl = method.getAnnotation(TpsControl.class);
        String pointName = tpsControl.pointName();
        TpsMonitorPoint tpsMonitorPoint = new TpsMonitorPoint(pointName);
        tpsMonitorManager.registerTpsControlPoint(tpsMonitorPoint);
      }
    } catch (Exception e) {
      //ignore.
    }

    // 注解@4
    Class tClass = (Class) ((ParameterizedType) clazz.getGenericSuperclass()).getActualTypeArguments()[0];
    registryHandlers.putIfAbsent(tClass.getSimpleName(), requestHandler);
  }
}
```

**注解@1：** 获取RequestHandler的众多实现类

**注解@2：** 跳过超类RequestHandler

**注解@3：** 获取需要带有「TpsControl」注解的方法，表示需要对流量Tps控制

**注解@4：** 将Handler装载到缓存 例如：InstanceRequest->InstanceRequestHandler

**小结：** 在spring启动时转载了众多的RequestHandler，这些Handler是真正的逻辑所在。下面列表时加载的13个RequestHandler

| Key                            | Value                                 | 备注 |
| ------------------------------ | ------------------------------------- | ---- |
| ConfigBatchListenRequest       | ConfigChangeBatchListenRequestHandler |      |
| ConfigChangeClusterSyncRequest | ConfigChangeClusterSyncRequestHandler |      |
| ConfigPublishRequest           | ConfigPublishRequestHandler           |      |
| ConfigQueryRequest             | ConfigQueryRequestHandler             |      |
| ConfigRemoveRequest            | ConfigRemoveRequestHandler            |      |
| HealthCheckRequest             | HealthCheckRequestHandler             |      |
| ServerLoaderInfoRequest        | ServerLoaderInfoRequestHandler        |      |
| ServerReloadRequest            | ServerReloaderRequestHandler          |      |
| DistroDataRequest              | DistroDataRequestHandler              |      |
| InstanceRequest                | InstanceRequestHandler                |      |
| ServiceListRequest             | ServiceListRequestHandler             |      |
| ConfigPublishRequest           | ConfigPublishRequestHandler           |      |
| SubscribeServiceRequest        | SubscribeServiceRequestHandler        |      |



# 服务端响应客户端连接

### client连接server

在前面《Nacos2# 服务注册与发现客户端示例与源码解析（二）》中client会与server建立连接具体方法为connectToServer()。

在分下《Nacos3# 服务注册与发现服务端启动源码解析》时protobuf文件装载了两类grpc请求类型，其中一类为双向流调用方式BiRequestStream#biRequestStream。

客户端发起connectToServer()时正式双向流调用类型：

```java
 public Connection connectToServer(ServerInfo serverInfo) {
 		// ... 
    BiRequestStreamGrpc.BiRequestStreamStub biRequestStreamStub = BiRequestStreamGrpc
                        .newStub(newChannelStubTemp.getChannel());
    
 }
private StreamObserver<Payload> bindRequestStream(final BiRequestStreamGrpc.BiRequestStreamStub streamStub,
            final GrpcConnection grpcConn) {
        
        return streamStub.requestBiStream(new StreamObserver<Payload>() {
          
        }
}
```

那在nacos server端如何响应的呢？



### Nacos server响应连接

在分析《Nacos3# 服务注册与发现服务端启动源码解析》服务端grpc启动构建暴露的服务定义信息时添加了requestBiStream

```java
 private void addServices(MutableHandlerRegistry handlerRegistry, ServerInterceptor... serverInterceptor) {
   // ...
 		// 服务接口处理类
   final ServerCallHandler<Payload, Payload> biStreamHandler = ServerCalls.asyncBidiStreamingCall(
     (responseObserver) -> grpcBiStreamRequestAcceptor.requestBiStream(responseObserver));
   // ...
    final ServerServiceDefinition serviceDefOfBiStream = ServerServiceDefinition
                .builder(REQUEST_BI_STREAM_SERVICE_NAME).addMethod(biStreamMethod, biStreamHandler).build();    
 }
```

接下来看其处理连接请求逻辑。坐标GrpcBiStreamRequestAcceptor#requestBiStream

```java
@Override
public StreamObserver<Payload> requestBiStream(StreamObserver<Payload> responseObserver) {

  StreamObserver<Payload> streamObserver = new StreamObserver<Payload>() {
		// 连接标识 eg. connectionId=1623495108814_127.0.0.1_54314
    final String connectionId = CONTEXT_KEY_CONN_ID.get();
		// eg. localPort=9848
    final Integer localPort = CONTEXT_KEY_CONN_LOCAL_PORT.get();
		// eg. remotePort=54314
    final int remotePort = CONTEXT_KEY_CONN_REMOTE_PORT.get();
		// eg. remoteIp=127.0.0.1
    String remoteIp = CONTEXT_KEY_CONN_REMOTE_IP.get();
		// eg. clientIp=192.168.0.110
    String clientIp = "";

    @Override
    public void onNext(Payload payload) {
			// 解析客户端IP
      clientIp = payload.getMetadata().getClientIp();
      traceDetailIfNecessary(payload);

      Object parseObj = null;
      try {
        parseObj = GrpcUtils.parse(payload); // 将二进制转换为对象
      } catch (Throwable throwable) {
        
      }
    
      if (parseObj instanceof ConnectionSetupRequest) {
        ConnectionSetupRequest setUpRequest = (ConnectionSetupRequest) parseObj;
        Map<String, String> labels = setUpRequest.getLabels();
        String appName = "-";
        if (labels != null && labels.containsKey(Constants.APPNAME)) {
          appName = labels.get(Constants.APPNAME);
        }

       ConnectionMeta metaInfo = new ConnectionMeta(connectionId, payload.getMetadata().getClientIp(),
           remoteIp, remotePort, localPort,ConnectionType.GRPC.getType(),setUpRequest.getClientVersion(), appName, setUpRequest.getLabels());
        metaInfo.setTenant(setUpRequest.getTenant());
        // 注解@5
        Connection connection = new GrpcConnection(metaInfo, responseObserver, CONTEXT_KEY_CHANNEL.get());
        connection.setAbilities(setUpRequest.getAbilities());
        boolean rejectSdkOnStarting = metaInfo.isSdkSource() && !ApplicationUtils.isStarted();
				// 注解@6
        if (rejectSdkOnStarting || !connectionManager.register(connectionId, connection)) {
          try {
           
            connection.request(new ConnectResetRequest(), 3000L);
            connection.close();
          } catch (Exception e) {
            //Do nothing.
            if (connectionManager.traced(clientIp)) {
             
            }
          }
          return;
        }

      } else if (parseObj != null && parseObj instanceof Response) {
        Response response = (Response) parseObj;
        if (connectionManager.traced(clientIp)) {
         
        }
        RpcAckCallbackSynchronizer.ackNotify(connectionId, response);
        connectionManager.refreshActiveTime(connectionId);
      } else {
      
        return;
      }

    }

 
  return streamObserver;
}
```

**注解@5** 构建连接对象

**注解@6**  注册连接信息，一个是缓存连接信息到Map connections中；另一个是触发client连接通知

```java
public synchronized boolean register(String connectionId, Connection connection) {

  if (connection.isConnected()) {
    
    connections.put(connectionId, connection);
    connectionForClientIp.get(connection.getMetaInfo().clientIp).getAndIncrement();

    clientConnectionEventListenerRegistry.notifyClientConnected(connection);
   
    return true;

  }
  return false;

}
```

client连接通知处理逻辑：坐标ConnectionBasedClientManager#clientConnected。主要构建Client对象并缓存，clientId=connectId

```java
public void clientConnected(Connection connect) {
  if (!RemoteConstants.LABEL_MODULE_NAMING.equals(connect.getMetaInfo().getLabel(RemoteConstants.LABEL_MODULE)))	{
    return;
  }
  String type = connect.getMetaInfo().getConnectType();
  ClientFactory clientFactory = ClientFactoryHolder.getInstance().findClientFactory(type);
  // 构建Client对象
  clientConnected(clientFactory.newClient(connect.getMetaInfo().getConnectionId()));
}
```

```java
 public boolean clientConnected(Client client) {
     Loggers.SRV_LOG.info("Client connection {} connect", client.getClientId());
     if (!clients.containsKey(client.getClientId())) {
       // 缓存Client信息
       clients.putIfAbsent(client.getClientId(), (ConnectionBasedClient) client);
     }
     return true;
 }
```

**小结：** Nacos Server响应客户端连接主要干了两件事：@1 构建和缓存GrpcConnection对象，包含ConnectionMeta、channel和StreamObserver；@2 构建和缓存Client对象包含；clientId即connectionId。

**ConnectionMeta属性**

| 属性           | 含义                                   |
| -------------- | -------------------------------------- |
| connectType    | ConnectionType，例如：GRPC             |
| clientIp       | 客户端IP，例如：clientIp=192.168.0.110 |
| remoteIp       | 远程节点IP，例如：remoteIp=127.0.0.1   |
| remotePort     | 远程节点端口，例如：remotePort=54314   |
| localPort      | 本地节点端口，例如：localPort=9848     |
| version        | client 版本                            |
| connectionId   | 连接标识                               |
| createTime     | 连接创建时间                           |
| lastActiveTime | 最新保活时间                           |
| appName        | app名称默认为“-”                       |
| tenant         | 租户                                   |

**ConnectionBasedClient属性**

| 属性            | 含义                                                         |
| --------------- | ------------------------------------------------------------ |
| connectionId    | 连接标识                                                     |
| isNative        | true表示client直接连接到该server；false表示该连接是被从其他server同步而来 |
| lastRenewTime   | 当isNative为false时，上次从源server校验的时间戳              |
| lastUpdatedTime | 最新保鲜时间                                                 |





# 服务端处理注册请求

在上面文章《服务注册与发现服务端启动源码解析》有提到由RequestAcceptor.request会处理

```java
// 服务接口处理类
final ServerCallHandler<Payload, Payload> payloadHandler = ServerCalls
  .asyncUnaryCall((request, responseObserver) -> {
    grpcCommonRequestAcceptor.request(request, responseObserver);
  });
```

下面看下request处理逻辑，忽略一些校验

```java
@Override
public void request(Payload grpcRequest, StreamObserver<Payload> responseObserver) {
  // 查看是否在监控列表
  traceIfNecessary(grpcRequest, true);
  String type = grpcRequest.getMetadata().getType();

  // server启动校验、server校验
  // ...
  // 注解@7
  RequestHandler requestHandler = requestHandlerRegistry.getByRequestType(type);
  // 没有Handler校验
  // ...
  
  String connectionId = CONTEXT_KEY_CONN_ID.get();
  boolean requestValid = connectionManager.checkValid(connectionId);
  // 校验connId是否有效无效返回
  // ...
  Object parseObj = null;
  try {
    parseObj = GrpcUtils.parse(grpcRequest); // 注解@8
  } catch (Exception e) {
    // ...
    return;
  }
  // 非null校验
  if (parseObj == null) {
    // ...
  }
  // 非request校验
  if (!(parseObj instanceof Request)) {
    // ...
    return;
  }
  Request request = (Request) parseObj;
  try {
    Connection connection = connectionManager.getConnection(CONTEXT_KEY_CONN_ID.get());
    RequestMeta requestMeta = new RequestMeta();
    // 注解@9
    requestMeta.setClientIp(connection.getMetaInfo().getClientIp());
    requestMeta.setConnectionId(CONTEXT_KEY_CONN_ID.get());
    requestMeta.setClientVersion(connection.getMetaInfo().getVersion());
    requestMeta.setLabels(connection.getMetaInfo().getLabels());
    // 注解@10
    connectionManager.refreshActiveTime(requestMeta.getConnectionId());
    // 注解@11
    Response response = requestHandler.handleRequest(request, requestMeta);
    Payload payloadResponse = GrpcUtils.convert(response);
    traceIfNecessary(payloadResponse, false);
    responseObserver.onNext(payloadResponse);
    responseObserver.onCompleted();
  } catch (Throwable e) {
    // ...
    return;
  }

}
```

**注解@7**  根据请求类型获取处理的RequestHandler，处理注册的为InstanceRequest->InstanceRequestHandler

**注解@8** 解析成Request对象

**注解@9** 装载clientIP、connId、version、其他map信息

**注解@10** 刷新连接保鲜时间

**注解@11** 执行RequestHandler逻辑，具体为registerInstance

```java
@Override
@Secured(action = ActionTypes.WRITE, parser = NamingResourceParser.class)
public InstanceResponse handle(InstanceRequest request, RequestMeta meta) throws NacosException {
  // 构建注册service
  Service service = Service
    .newService(request.getNamespace(), request.getGroupName(), request.getServiceName(), true);
  switch (request.getType()) {
      // 处理注册请求
    case NamingRemoteConstants.REGISTER_INSTANCE:
      return registerInstance(service, request, meta);
      // 处理注销请求
    case NamingRemoteConstants.DE_REGISTER_INSTANCE:
      return deregisterInstance(service, request, meta);
    default:
      throw new NacosException(NacosException.INVALID_PARAM,
                               String.format("Unsupported request type %s", request.getType()));
  }
}
```

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210607141823.png)

接着分析注册干了点啥？坐标：EphemeralClientOperationServiceImpl#registerInstance

```java
@Override
public void registerInstance(Service service, Instance instance, String clientId) {
  // 注解@12
  Service singleton = ServiceManager.getInstance().getSingleton(service);
  // 注解@13
  Client client = clientManager.getClient(clientId);
  InstancePublishInfo instanceInfo = getPublishInfo(instance);
  // 注解@14
  client.addServiceInstance(singleton, instanceInfo);
  client.setLastUpdatedTime();
  // 注解@15
  NotifyCenter.publishEvent(new ClientOperationEvent.ClientRegisterServiceEvent(singleton, clientId));
  // 注解@16
  NotifyCenter
    .publishEvent(new MetadataEvent.InstanceMetadataEvent(singleton, instanceInfo.getMetadataId(), false));
}
```

**注解@12** singletonRepository缓存Singleton（service），namespaceSingletonMaps建立namespace和Singleton关系

**注解@13** 从缓存中获取client信息，也就是上一小节建立连接被缓存的client

**注解@14** 建立client与service、instance关系并发布事件ClientEvent.ClientChangedEvent

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

**注解@15** 发布ClientOperationEvent.ClientRegisterServiceEvent事件

**注解@16** 发布MetadataEvent.InstanceMetadataEvent事件



**小结：** 处理注册请求的逻辑由InstanceRequestHandler执行，建立起了client与service、instance的关系。并通过NotifyCenter发布了三个事件：@1 ClientEvent.ClientChangedEvent；@2 ClientOperationEvent.ClientRegisterServiceEvent事件；@3 MetadataEvent.InstanceMetadataEvent事件。



那谁订阅了这些事件呢？他们又做了啥？下回接着撸。






