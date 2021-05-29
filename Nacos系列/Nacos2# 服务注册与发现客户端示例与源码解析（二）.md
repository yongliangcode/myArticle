---
title: Nacos2# 服务注册与发现客户端示例与源码解析（二）
categories: Nacos
tags: Nacos
date: 2021-05-23 11:55:01
---



# 引言

上一篇客户端初始化没有撸完，这篇继续。Nacos从2.0以后增加了对grpc的支持，代码中HTTP的代理初始化还有保留，我们注册发现通常为临时节点，这部分已由gRPC接管。可以对比下新旧逻辑的实现差异。



# 内容提要



### HTTP代理初始化

**HTTP心跳检测器** 

* HTTP心跳检测只适用于注册的节点持久节点，临时节点会使用grpc代理（HTTP的心跳检测默认废弃由grpc替代）
* 在初始化时客户端注册代理NamingClientProxy时，初始化了一个HTTP心跳器用于向Nacos Server发起心跳
* 在注册节点时通过向心跳执行器添加心跳任务addBeatInfo触发
* 心跳执行器通过每隔五秒中向Nacos Server发起HTTP请求
* 如果返回的server not found会向Nacos Server发起注册请求重新注册
  

**UDP接受服务端推送** 

* Client通过UDP接受到nacos server推动的消息
* 如果服务端推送的为服务信息通过processServiceInfo处理逻辑见上篇，主要实例变更时的通知机制
* 如果dump类型，则客户端发送服务信息serviceInfoMap的ack信息到服务端



### gRPC代理初始化

**gRPC初始化逻辑概览** 

*  gRPC 客户端代理的初始化主要逻辑为创建gRPC Client并启动
* 并注册ServerRequestHandler用于处理Nacos Server推送的NotifySubscriberRequest请求
* 注册ConnectionListener用于处理gRPC建立和断开连接事件
* 请求超时时间可以通过「namingRequestTimeout」设置，默认为3秒



**gRPC Client启动逻辑** 

* gRPC Client启动逻辑主要在于建立与nacos server的grpc连接，其中两个守护线程一直在运行
* 守护线程1用于处理grpc连接的建立和关闭事件
* 守护线程2用于与nacos server的心跳保鲜，并负责异步建立grpc连接
* 守护线程2同时负责当nacos server的地址信息发生变更时重新与新server建立连接
* nacos server的地址变更通过grpc通道由server推送ConnectResetRequest到client
* grpc client只与nacos server集群中一台建立grpc连接。



# 源码分析

```java
public NamingClientProxyDelegate(String namespace, ServiceInfoHolder serviceInfoHolder, Properties properties,
            InstancesChangeNotifier changeNotifier) throws NacosException {
        this.serviceInfoUpdateService = new ServiceInfoUpdateService(properties, serviceInfoHolder, this,
                changeNotifier);
  		  this.serverListManager = new ServerListManager(properties);
  		  this.serviceInfoHolder = serviceInfoHolder;
        this.securityProxy = new SecurityProxy(properties, NamingHttpClientManager.getInstance().getNacosRestTemplate());
        initSecurityProxy();
  		  // @注解7.4
        this.httpClientProxy = new NamingHttpClientProxy(namespace, securityProxy, serverListManager, properties,serviceInfoHolder);
  			// @注解7.5
        this.grpcClientProxy = new NamingGrpcClientProxy(namespace, securityProxy, serverListManager, properties,serviceInfoHolder);
}
```

 

# HTTP代理初始化

**@注解7.4** Http代理的初始化，该代理主要在nacos 2.0以前版本使用，2.0之后通过grpc与nacos server通信。

```java
public NamingHttpClientProxy(String namespaceId, SecurityProxy securityProxy, ServerListManager serverListManager,
            Properties properties, ServiceInfoHolder serviceInfoHolder) {
        super(securityProxy, properties);
        this.serverListManager = serverListManager;
        this.setServerPort(DEFAULT_SERVER_PORT);
        this.namespaceId = namespaceId;
  			// @注解7.4.1
        this.beatReactor = new BeatReactor(this, properties);
  			// @注解7.4.2
        this.pushReceiver = new PushReceiver(serviceInfoHolder);
  			// @注解7.4.3
        this.maxRetry = ConvertUtils.toInt(properties.getProperty(PropertyKeyConst.NAMING_REQUEST_DOMAIN_RETRY_COUNT,
                String.valueOf(UtilAndComs.REQUEST_DOMAIN_RETRY_COUNT)));
 }
```



### HTTP心跳检测器

**@注解7.4.1** 初始化BeatReactor，用于向nacos server发送心跳

```java
public BeatReactor(NamingHttpClientProxy serverProxy, Properties properties) {
        this.serverProxy = serverProxy;
        // 心跳线程池大小，默认为核数的二分之一，最小为1，可通过properties参数「namingClientBeatThreadCount」设置
        int threadCount = initClientBeatThreadCount(properties);
  			// 初始化线程执行器
        this.executorService = new ScheduledThreadPoolExecutor(threadCount, new ThreadFactory() {
            @Override
            public Thread newThread(Runnable r) {
                Thread thread = new Thread(r);
                thread.setDaemon(true);
                thread.setName("com.alibaba.nacos.naming.beat.sender");
                return thread;
            }
        });
}
```

接着一下这个执行器再做什么事情。

```java
public void addBeatInfo(String serviceName, BeatInfo beatInfo) {
        NAMING_LOGGER.info("[BEAT] adding beat: {} to beat map.", beatInfo);
        String key = buildKey(serviceName, beatInfo.getIp(), beatInfo.getPort());
        BeatInfo existBeat = null;
        //fix #1733
        if ((existBeat = dom2Beat.remove(key)) != null) {
            existBeat.setStopped(true);
        }
        dom2Beat.put(key, beatInfo);
        // 默认延迟5秒
        executorService.schedule(new BeatTask(beatInfo), beatInfo.getPeriod(), TimeUnit.MILLISECONDS);
        MetricsMonitor.getDom2BeatSizeMonitor().set(dom2Beat.size());
}
```

当通过addBeatInfo增加一个心跳信息BeatInfo时，执行器会创建BeatTask（Runnable）延迟5秒运行。

```java
class BeatTask implements Runnable {
			  BeatInfo beatInfo;
			  public BeatTask(BeatInfo beatInfo) {
            this.beatInfo = beatInfo;
        }
			  @Override
        public void run() {
            if (beatInfo.isStopped()) {
                return;
            }
            long nextTime = beatInfo.getPeriod();
            try {
              	// 向nacos server「/nacos/v1/ns/instance/beat」发送心跳
                JsonNode result = serverProxy.sendBeat(beatInfo, BeatReactor.this.lightBeatEnabled);
                long interval = result.get("clientBeatInterval").asLong();
                boolean lightBeatEnabled = false;
                if (result.has(CommonParams.LIGHT_BEAT_ENABLED)) {
                    lightBeatEnabled = result.get(CommonParams.LIGHT_BEAT_ENABLED).asBoolean();
                }
                BeatReactor.this.lightBeatEnabled = lightBeatEnabled;
                if (interval > 0) {
                    nextTime = interval;
                }
                int code = NamingResponseCode.OK;
                if (result.has(CommonParams.CODE)) {
                    code = result.get(CommonParams.CODE).asInt();
                }
              	// 如果nacos server返回NOT FOUND则重新发起注册请求
                if (code == NamingResponseCode.RESOURCE_NOT_FOUND) {
                    Instance instance = new Instance();
                    instance.setPort(beatInfo.getPort());
                    instance.setIp(beatInfo.getIp());
                    instance.setWeight(beatInfo.getWeight());
                    instance.setMetadata(beatInfo.getMetadata());
                    instance.setClusterName(beatInfo.getCluster());
                    instance.setServiceName(beatInfo.getServiceName());
                    instance.setInstanceId(instance.getInstanceId());
                    instance.setEphemeral(true);
                    try {
                        serverProxy.registerService(beatInfo.getServiceName(),
                                NamingUtils.getGroupName(beatInfo.getServiceName()), instance);
                    } catch (Exception ignore) {
                    }
                }
            } catch (NacosException ex) {
                NAMING_LOGGER.error("[CLIENT-BEAT] failed to send beat: {}, code: {}, msg: {}",
                        JacksonUtils.toJson(beatInfo), ex.getErrCode(), ex.getErrMsg());

            }
          	// 默认为5秒，可以通过PreservedMetadataKeys.HEART_BEAT_INTERVAL设置
            executorService.schedule(new BeatTask(beatInfo), nextTime, TimeUnit.MILLISECONDS);
        }
}
```

addBeatInfo调用时机，当节点在注册时如果实例为临时节点，则会创建心跳任务发起

```java
public void registerService(String serviceName, String groupName, Instance instance) throws NacosException {
        
        NAMING_LOGGER.info("[REGISTER-SERVICE] {} registering service {} with instance: {}", namespaceId, serviceName,instance);
        String groupedServiceName = NamingUtils.getGroupedName(serviceName, groupName);
        if (instance.isEphemeral()) {
            BeatInfo beatInfo = beatReactor.buildBeatInfo(groupedServiceName, instance);
          	// 添加心跳任务
            beatReactor.addBeatInfo(groupedServiceName, beatInfo);
        }
        final Map<String, String> params = new HashMap<String, String>(16);
        params.put(CommonParams.NAMESPACE_ID, namespaceId);
        params.put(CommonParams.SERVICE_NAME, groupedServiceName);
        params.put(CommonParams.GROUP_NAME, groupName);
        params.put(CommonParams.CLUSTER_NAME, instance.getClusterName());
        params.put("ip", instance.getIp());
        params.put("port", String.valueOf(instance.getPort()));
        params.put("weight", String.valueOf(instance.getWeight()));
        params.put("enable", String.valueOf(instance.isEnabled()));
        params.put("healthy", String.valueOf(instance.isHealthy()));
        params.put("ephemeral", String.valueOf(instance.isEphemeral()));
        params.put("metadata", JacksonUtils.toJson(instance.getMetadata()));
        
        reqApi(UtilAndComs.nacosUrlInstance, params, HttpMethod.POST);
        
}
```

再跟踪下注册入口，判读使用哪个ClientProxy

```java
@Override
public void registerService(String serviceName, String groupName, Instance instance) throws NacosException {
    getExecuteClientProxy(instance).registerService(serviceName, groupName, instance);
}

private NamingClientProxy getExecuteClientProxy(Instance instance) {
        // 是否为临时节点，临时节点使用grpc，持久节点使用http；默认为true，也就是默认使用grpcClientProxy
        return instance.isEphemeral() ? grpcClientProxy : httpClientProxy;
}
```



**小结：** HTTP心跳检测只适用于注册的节点持久节点，临时节点会使用grpc代理，即HTTP的心跳检测默认废弃由grpc替代；在初始化时客户端注册代理NamingClientProxy时，初始化了一个HTTP心跳器用于向Nacos Server发起心跳；在注册节点时通过向心跳执行器添加心跳任务addBeatInfo触发；心跳执行器通过每隔五秒中向Nacos Server发起HTTP请求，如果返回的server not found会向Nacos Server发起注册请求重新注册；



### UDP接受服务端推送

**@注解7.4.2** 初始化PushReceiver用于接受nacos server信息推送，使用UDP协议。

```java
 public PushReceiver(ServiceInfoHolder serviceInfoHolder) {
        try {
            this.serviceInfoHolder = serviceInfoHolder;
            this.udpSocket = new DatagramSocket();
            this.executorService = new ScheduledThreadPoolExecutor(1, new ThreadFactory() {
                @Override
                public Thread newThread(Runnable r) {
                    Thread thread = new Thread(r);
                    thread.setDaemon(true);
                    thread.setName("com.alibaba.nacos.naming.push.receiver");
                    return thread;
                }
            });
            
            this.executorService.execute(this);
        } catch (Exception e) {
            NAMING_LOGGER.error("[NA] init udp socket failed", e);
        }
}
```

**备注：** PushReceiver实现Runnable接口，在构造方法中通过守护线程运行。

```java
public void run() {
  while (!closed) {
    try {
      // byte[] is initialized with 0 full filled by default
      byte[] buffer = new byte[UDP_MSS];
      DatagramPacket packet = new DatagramPacket(buffer, buffer.length);
      // 接受nacos server推送
      udpSocket.receive(packet);
      // 将推送内容转换为json字符串
      String json = new String(IoUtils.tryDecompress(packet.getData()), UTF_8).trim();
      NAMING_LOGGER.info("received push data: " + json + " from " + packet.getAddress().toString());
      PushPacket pushPacket = JacksonUtils.toObj(json, PushPacket.class);
      String ack;
      // 推送类型服务信息（例如订阅实例的变更）会通知订阅者逻辑已在上篇分析
      if ("dom".equals(pushPacket.type) || "service".equals(pushPacket.type)) {
        serviceInfoHolder.processServiceInfo(pushPacket.data);
        // send ack to server
        ack = "{\"type\": \"push-ack\"" + ", \"lastRefTime\":\"" + pushPacket.lastRefTime + "\", \"data\":"
          + "\"\"}";
      } else if ("dump".equals(pushPacket.type)) {
        // dump data to server
        ack = "{\"type\": \"dump-ack\"" + ", \"lastRefTime\": \"" + pushPacket.lastRefTime + "\", \"data\":"
          + "\"" + StringUtils.escapeJavaScript(JacksonUtils.toJson(serviceInfoHolder.getServiceInfoMap()))
          + "\"}";
      } else {
        // do nothing send ack only
        ack = "{\"type\": \"unknown-ack\"" + ", \"lastRefTime\":\"" + pushPacket.lastRefTime
          + "\", \"data\":" + "\"\"}";
      }
			// 向Server发送ack消息
      udpSocket.send(new DatagramPacket(ack.getBytes(UTF_8), ack.getBytes(UTF_8).length,
                                        packet.getSocketAddress()));
    } catch (Exception e) {
      if (closed) {
        return;
      }
      NAMING_LOGGER.error("[NA] error while receiving push data", e);
    }
  }
}
```

**小结：** Client通过UDP接受到nacos server推动的消息：@1如果推送的为服务信息通过processServiceInfo处理，逻辑见上篇；@2 如果dump类型，则客户端发送服务信息serviceInfoMap的ack信息到服务端。



### HTTP重试次数

**@注解7.4.3**  client通过HTTP向Nacos Server请求的重试次数，默认为3次。可以通过「namingRequestDomainMaxRetryCount」指定

```java
 public String reqApi(String api, Map<String, String> params, Map<String, String> body, List<String> servers,
            String method) throws NacosException {
 		// ...
 		if (serverListManager.isDomain()) {
            String nacosDomain = serverListManager.getNacosDomain();
            for (int i = 0; i < maxRetry; i++) { // 请求发送异常最大重试次数
                try {
                    return callServer(api, params, body, nacosDomain, method);
                } catch (NacosException e) {
                    exception = e;
                    if (NAMING_LOGGER.isDebugEnabled()) {
                        NAMING_LOGGER.debug("request {} failed.", nacosDomain, e);
                    }
                }
            }
 		}
 		//....
 }
```





# gRPC代理初始化



### gRPC初始化逻辑概览



**@注解7.5** 下面接着gRPC 客户端代理的初始化逻辑

```java
public NamingGrpcClientProxy(String namespaceId, SecurityProxy securityProxy, ServerListFactory serverListFactory,
            Properties properties, ServiceInfoHolder serviceInfoHolder) throws NacosException {
        super(securityProxy, properties);
        this.namespaceId = namespaceId;
        this.uuid = UUID.randomUUID().toString();
        // 设置请求超时时间，默认为3秒。可以通过参数「namingRequestTimeout」设置
        this.requestTimeout = Long.parseLong(properties.getProperty(CommonParams.NAMING_REQUEST_TIMEOUT, "-1"));
        Map<String, String> labels = new HashMap<String, String>();
        // 设置source=sdk，module=naming
        labels.put(RemoteConstants.LABEL_SOURCE, RemoteConstants.LABEL_SOURCE_SDK);
        labels.put(RemoteConstants.LABEL_MODULE, RemoteConstants.LABEL_MODULE_NAMING);
        // 创建gRPC Client：clientName=uuid，ConnectionType=GRPC
        this.rpcClient = RpcClientFactory.createClient(uuid, ConnectionType.GRPC, labels);
        // 创建ConnectionEventListener用于建立和断开gRPC连接时的事件响应
        this.namingGrpcConnectionEventListener = new NamingGrpcConnectionEventListener(this);
 			  // 启动grpc client
        start(serverListFactory, serviceInfoHolder);
}
```

```java
private void start(ServerListFactory serverListFactory, ServiceInfoHolder serviceInfoHolder) throws NacosException {
        rpcClient.serverListFactory(serverListFactory);
  			// @注解7.5.1 gRPC Client启动
        rpcClient.start();
  			// 注册registerServerRequestHandler用于处理从Nacos Push到Client的请求
        rpcClient.registerServerRequestHandler(new NamingPushRequestHandler(serviceInfoHolder));
  			// 注册连接事件Listener，当连接建立和断开时处理事件
        rpcClient.registerConnectionListener(namingGrpcConnectionEventListener);
}
```

**小结：** gRPC 客户端代理的初始化主要逻辑为创建gRPC Client并启动；并注册ServerRequestHandler用于处理Nacos Server推送的NotifySubscriberRequest请求；注册ConnectionListener用于处理gRPC建立和断开连接事件；另外，请求超时时间可以通过「namingRequestTimeout」设置，默认为3秒。



### gRPC Client启动逻辑



**@注解7.5.1**  gRPC Client启动逻辑

```java
public final void start() throws NacosException {
  // 将Client状态由INITIALIZED变更为STARTING
  boolean success = rpcClientStatus.compareAndSet(RpcClientStatus.INITIALIZED, RpcClientStatus.STARTING);
  if (!success) {
    return;
  }
  // -------------------------@1 satart---------------------------------------------
  // 守护线程执行器
  clientEventExecutor = new ScheduledThreadPoolExecutor(2, new ThreadFactory() {
    @Override
    public Thread newThread(Runnable r) {
      Thread t = new Thread(r);
      t.setName("com.alibaba.nacos.client.remote.worker");
      t.setDaemon(true);
      return t;
    }
  });
	
  // 从BlockingQueue中不断获取连接Event，根据事件类型回调onConnected()/onDisConnect()事件
  clientEventExecutor.submit(new Runnable() {
    @Override
    public void run() {
      while (true) {
        ConnectionEvent take = null;
        try {
          take = eventLinkedBlockingQueue.take();
          if (take.isConnected()) {
            notifyConnected();
          } else if (take.isDisConnected()) {
            notifyDisConnected();
          }
        } catch (Throwable e) {
          //Do nothing
        }
      }
    }
  });
	// -------------------------@1 end---------------------------------------------
  
  
  // -------------------------@2 start---------------------------------------------
  clientEventExecutor.submit(new Runnable() {
    @Override
    public void run() {
      while (true) {
        try {
          // 获取重定向连接上下文，指重新连接到其他server节点
          ReconnectContext reconnectContext = reconnectionSignal
            .poll(keepAliveTime, TimeUnit.MILLISECONDS);
          if (reconnectContext == null) {
            // check alive time.
            // client活动时间超过5秒钟，向Nacos Server发起健康检测
            if (System.currentTimeMillis() - lastActiveTimeStamp >= keepAliveTime) {
              // 发送健康检查
              boolean isHealthy = healthCheck();
              // 非健康节点
              if (!isHealthy) {
                if (currentConnection == null) {
                  continue;
                }
                LoggerUtils.printIfInfoEnabled(LOGGER,
                                               "[{}]Server healthy check fail,currentConnection={}", name,
                                               currentConnection.getConnectionId());
                // 标记客户端状态为unhealthy
                rpcClientStatus.set(RpcClientStatus.UNHEALTHY);
                // 重置ReconnectContext移除serverInfo
                reconnectContext = new ReconnectContext(null, false);

              } else {
                // 健康连接更新时间戳
                lastActiveTimeStamp = System.currentTimeMillis();
                continue;
              }
            } else {
              // 心跳保鲜未过期，跳过本次检测
              continue;
            }

          }

          if (reconnectContext.serverInfo != null) {
            // clear recommend server if server is not in server list.
            boolean serverExist = false;
            // 判断连接上下文的reconnectContext.serverInfo是否在我们推荐设置的列表中
            for (String server : getServerListFactory().getServerList()) {
              ServerInfo serverInfo = resolveServerInfo(server);
              if (serverInfo.getServerIp().equals(reconnectContext.serverInfo.getServerIp())) {
                serverExist = true;
                reconnectContext.serverInfo.serverPort = serverInfo.serverPort;
                break;
              }
            }
            // 不再推荐的列表中则移除，改为随机选择
            if (!serverExist) {
              LoggerUtils.printIfInfoEnabled(LOGGER,
                                             "[{}] Recommend server is not in server list ,ignore recommend server {}", name,
                                             reconnectContext.serverInfo.getAddress());

              reconnectContext.serverInfo = null;

            }
          }
          // 发起重新连接
          reconnect(reconnectContext.serverInfo, reconnectContext.onRequestFail);
        } catch (Throwable throwable) {
          //Do nothing
        }
      }
    }
  });
  // -------------------------@2 end---------------------------------------------
  
  
  // -------------------------@3 start---------------------------------------------
  // 异步连接nacos server失败，改为同步连接
  //connect to server ,try to connect to server sync once, async starting if fail.
  Connection connectToServer = null;
  rpcClientStatus.set(RpcClientStatus.STARTING);

  int startUpRetryTimes = RETRY_TIMES;
  while (startUpRetryTimes > 0 && connectToServer == null) {
    try {
      startUpRetryTimes--;
      ServerInfo serverInfo = nextRpcServer();

      LoggerUtils.printIfInfoEnabled(LOGGER, "[{}] Try to connect to server on start up, server: {}", name,
                                     serverInfo);

      connectToServer = connectToServer(serverInfo);
    } catch (Throwable e) {
      LoggerUtils.printIfWarnEnabled(LOGGER,
                                     "[{}]Fail to connect to server on start up, error message={}, start up retry times left: {}",
                                     name, e.getMessage(), startUpRetryTimes);
    }

  }

  // -------------------------@3 end---------------------------------------------
  
  
	// -------------------------@4 start---------------------------------------------
  
  if (connectToServer != null) {
    LoggerUtils.printIfInfoEnabled(LOGGER, "[{}] Success to connect to server [{}] on start up,connectionId={}",name, connectToServer.serverInfo.getAddress(),connectToServer.getConnectionId());
      this.currentConnection = connectToServer;
      rpcClientStatus.set(RpcClientStatus.RUNNING);
      // 连接成功添加ConnectionEvent
      eventLinkedBlockingQueue.offer(new ConnectionEvent(ConnectionEvent.CONNECTED));
    } else {
    	// 未成功建立连接重新发起异步建立连接需求
      switchServerAsync();
    }
  
	// 注册ConnectResetRequestHandler用于处理nacos server推送的重置连接请求
  registerServerRequestHandler(new ConnectResetRequestHandler());

  //register client detection request.
  registerServerRequestHandler(new ServerRequestHandler() {
    @Override
    public Response requestReply(Request request) {
      if (request instanceof ClientDetectionRequest) {
        return new ClientDetectionResponse();
      }

      return null;
    }
  });
  Runtime.getRuntime().addShutdownHook(new Thread() {
    @Override
    public void run() {
      try {
        RpcClient.this.shutdown();
      } catch (NacosException e) {
        LoggerUtils.printIfErrorEnabled(LOGGER, "[{}]RpcClient shutdown exception, errorMessage ={}", name,
                                        e.getMessage());
      }
    }
  });
  // -------------------------@4 end---------------------------------------------
}
```

**备注：** grpc client启动时的逻辑：**逻辑块@1**  守护线程不断从阻塞队列eventLinkedBlockingQueue获取grpc连接/断开事件，并调用上文中注册的namingGrpcConnectionEventListener回调其onConnected/onDisConnect方法。 其中事件添加时机为：

grpc连接建立时，添加连接事件：

```java
// 连接成功添加ConnectionEvent
eventLinkedBlockingQueue.offer(new ConnectionEvent(ConnectionEvent.CONNECTED));
```

grpc连接关闭时，添加关闭事件：

```java
private void closeConnection(Connection connection) {
        if (connection != null) {
            connection.close();
            // 断开连接添加DISCONNECTED事件
            eventLinkedBlockingQueue.add(new ConnectionEvent(ConnectionEvent.DISCONNECTED));
        }
}
```

**逻辑块@2**  守护线程不断从阻塞队列reconnectionSignal获取重新连接事件（ReconnectContext）也就是更换nacos server的连接grpc通道：

阻塞队列没有重新连接事件：则做心跳保鲜检测，心跳频率为5秒。当超过5秒时会向Nacos Server发起健康检查，当返回不健康时，将grpc client标记为unhealthy；返回健康则刷新心跳时间lastActiveTimeStamp。

阻塞队列有重新连接事件：重连事件上下文reconnectContext的的server ip在我们设置的nacos server 列表则使用，否则改为随机选择nacos server ip地址，并与新server建立连接。

```java
protected void reconnect(final ServerInfo recommendServerInfo, boolean onRequestFail) {

  try {

    AtomicReference<ServerInfo> recommendServer = new AtomicReference<ServerInfo>(recommendServerInfo);
    // onRequestFail=true表示当健康检查失败grpcClient被设置为unhealthy，重连时重新发起健康检查，如果检查通过则不再执行重连
    if (onRequestFail && healthCheck()) {
      LoggerUtils.printIfInfoEnabled(LOGGER, "[{}] Server check success,currentServer is{} ", name,
                                     currentConnection.serverInfo.getAddress());
      rpcClientStatus.set(RpcClientStatus.RUNNING);
      return;
    }

    LoggerUtils.printIfInfoEnabled(LOGGER, "[{}] try to re connect to a new server ,server is {}", name,
                                   recommendServerInfo == null ? " not appointed,will choose a random server."
                                   : (recommendServerInfo.getAddress() + ", will try it once."));

    // loop until start client success.
    boolean switchSuccess = false;

    int reConnectTimes = 0;
    int retryTurns = 0;
    Exception lastException = null;
    // 切换nacos server没有成功则会一直重试
    while (!switchSuccess && !isShutdown()) {

      //1.get a new server
      ServerInfo serverInfo = null;
      try {
        // 获取需要重新连接的server地址
        serverInfo = recommendServer.get() == null ? nextRpcServer() : recommendServer.get();
        //2.create a new channel to new server
        // 与新的server建立grpc连接，如果连接失败返回null
        Connection connectionNew = connectToServer(serverInfo);
        // 关闭缓存的当前连接并重定向到新的连接
        if (connectionNew != null) {
          LoggerUtils.printIfInfoEnabled(LOGGER, "[{}] success to connect a server  [{}],connectionId={}",
                                         name, serverInfo.getAddress(), connectionNew.getConnectionId());
          //successfully create a new connect.
          if (currentConnection != null) {
            LoggerUtils.printIfInfoEnabled(LOGGER,"[{}] Abandon prev connection ,server is  {}, connectionId is {}", name,currentConnection.serverInfo.getAddress(), currentConnection.getConnectionId());
            //set current connection to enable connection event.
            currentConnection.setAbandon(true);
            closeConnection(currentConnection);
          }
          currentConnection = connectionNew;
          rpcClientStatus.set(RpcClientStatus.RUNNING);
          switchSuccess = true;
          // 添加连接成功时间到阻塞队列
          boolean s = eventLinkedBlockingQueue.add(new ConnectionEvent(ConnectionEvent.CONNECTED));
          return;
        }

        //close connection if client is already shutdown.
        if (isShutdown()) {
          closeConnection(currentConnection);
        }

        lastException = null;

      } catch (Exception e) {
        lastException = e;
      } finally {
        // 清理本次重连请求
        recommendServer.set(null);
      }

      // 执行到这里表示上面没有成功建立连接，打印重试次数日志
      if (reConnectTimes > 0
          && reConnectTimes % RpcClient.this.serverListFactory.getServerList().size() == 0) {
        LoggerUtils.printIfInfoEnabled(LOGGER,"[{}] fail to connect server,after trying {} times, last try server is {},error={}", name,reConnectTimes, serverInfo, lastException == null ? "unknown" : lastException);
        if (Integer.MAX_VALUE == retryTurns) {
          retryTurns = 50;
        } else {
          retryTurns++;
        }
      }

      reConnectTimes++;
			// 重试时等待特定的时间
      try {
        //sleep x milliseconds to switch next server.
        if (!isRunning()) {
          // first round ,try servers at a delay 100ms;second round ,200ms; max delays 5s. to be reconsidered.
          Thread.sleep(Math.min(retryTurns + 1, 50) * 100L);
        }
      } catch (InterruptedException e) {
        // Do  nothing.
      }
    }

    if (isShutdown()) {
      LoggerUtils.printIfInfoEnabled(LOGGER, "[{}] Client is shutdown ,stop reconnect to server", name);
    }

  } catch (Exception e) {
    LoggerUtils.printIfWarnEnabled(LOGGER, "[{}] Fail to  re connect to server ,error is {}", name, e);
  }
}
```

**备注：** 重新切换连接server逻辑：@1当检查失败grpc client会被标记为unhealthy这类型onRequestFail为true，重连时重新发起健康检查，如果检查成功，则退出本次重连。@2 获取重连的server地址和端口，并建立grpc连接，关闭当前缓存的旧连接并重定向到新连接，同时添加连接成功时间到阻塞队列。@3 一直重试直到连接建立成功，每次重试等待一些时间（100ms,200ms...最大为5秒）。



**逻辑块@3** 当异步与nacos server建立失败时，改为尝试同步建立连接。



**逻辑块@4** 如果连接建立成功添加连接事件到阻塞队列；连接建立失败发起异步建立连接请求；注册ConnectResetRequestHandler用于处理nacos server推送的重置连接请求；jvm退出时通过hook关闭grpc client。



**小结：**  gRPC Client启动逻辑主要在于建立与nacos server的grpc连接，其中两个守护线程一直在运行。一个用于处理grpc连接的建立和关闭事件；一个用于与nacos server的心跳保鲜，并负责异步建立grpc连接，当nacos server的地址信息发生变更时负责重新与新server建立连接；grpc client只与nacos server集群中一台建立grpc连接。































   



















