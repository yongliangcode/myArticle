---
title: Nacos10# 健康检查类型与场景
categories: Nacos
tags: Nacos
date: 2021-07-08 11:55:01
---



# 引言

Nacos支持众多健康检查类型，心跳、HTTP、TCP、MySQL等类型，这些都作用于什么场景？他们又是如何事项的呢？本文就撸一撸这个。



# 内容提要



### 临时节点续约

* 临时节点续约通过gRPC连接保鲜实现
* 执行频率5秒一次
* 检查结果健康刷新保鲜时间
* 检查结果不可用标记节点不健康
* 当节点不健康时重新连接时会从server列表选择下一个节点连接



### 持久节点心跳检测

* 心跳执行器通过每隔五秒中向Nacos Server发起HTTP请求
* 如果返回的server not found会向Nacos Server发起注册请求重新注册



### 持久节点探活

* Nacos探活只有在持久节点注册时才会支持
* 探活支持HTTP、TCP、Mysql三种探活类型
* HTTP通过检测返回200状态码标记是否健康
* TPC通过Channel连接方式标记是否健康
* Mysql则保证当前节点为主节点，可用于主从切换场景



# 临时节点续约

在《Nacos2# 服务注册与发现客户端示例与源码解析（二）》分析gRPC Client启动逻辑时有分析连接健康检查逻辑。具体代码在RpcClient#start()中，下面再次聚焦下。



### client连接心跳检查

```java
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

                            // 健康连接更新时间戳
                            lastActiveTimeStamp = System.currentTimeMillis();
                            continue;
                        }
                    } else {
                        // 心跳保鲜未过期，跳过本次检测
                        continue;
                    }

                }
               // ...
            } catch (Throwable throwable) {
                //Do nothing
            }
        }
    }
});
```



### 服务端响应

代码翻到GrpcRequestAcceptor#request部分，执行RequestHandler逻辑。

```java
Response response = requestHandler.handleRequest(request, requestMeta);
```

```java
@TpsControl(pointName = "HealthCheck")
public HealthCheckResponse handle(HealthCheckRequest request, RequestMeta meta) {
    return new HealthCheckResponse();
}
```

当服务端收到健康检查请求时，通过HealthCheckRequestHandler#handle返回HealthCheckResponse。



### 节点选择

```java
while (startUpRetryTimes > 0 && connectToServer == null) {
    try {
        startUpRetryTimes--;
        ServerInfo serverInfo = nextRpcServer(); // 需连接的节点

        LoggerUtils.printIfInfoEnabled(LOGGER, "[{}] Try to connect to server on start up, server: {}", name,
                serverInfo);

        connectToServer = connectToServer(serverInfo); // 发起rpc连接，连接到集群中其他节点
    } catch (Throwable e) {
        LoggerUtils.printIfWarnEnabled(LOGGER,
                "[{}]Fail to connect to server on start up, error message={}, start up retry times left: {}",
                name, e.getMessage(), startUpRetryTimes);
    }

}
```

在选择节点时从server地址列表中自增选择下一个。

```java
@Override
public String genNextServer() {
    int index = currentIndex.incrementAndGet() % getServerList().size(); // 获取集群中下一个节点
    return getServerList().get(index);
}
```



### 清理无效连接

详见：RpcScheduledExecutor#start()；定时任务会清理无效连接

```java
// 定时任务每3秒执行一次
RpcScheduledExecutor.COMMON_SERVER_EXECUTOR.scheduleWithFixedDelay(new Runnable() {
    @Override
    public void run() {
    	// 超过保鲜时间的连接，重新异步发起连接
        connection.asyncRequest(clientDetectionRequest, new RequestCallBack() {
        
        // 刷新激活时间
        connection.freshActiveTime();
        successConnections.add(outDateConnectionId);
          
        //当失效时（保鲜时间超过20秒），注销关闭connection
        unregister(outDateConnectionId);
   	}
 }, 1000L, 3000L, TimeUnit.MILLISECONDS);
```



**小结：** 在临时节点注册时，客户端gRPC启动时会启动一个守护线程用户健康检查；健康检查的频率为5秒执行一次；当检查结果健康则刷新保鲜时间；检查结果不可用标记gRPC客户端状态为unhealthy；不健康节点在发起连接时会从server地址列表中选择下一个发起连接；

服务端也会定时清理超过保鲜时间连接。



# 永久节点心跳检测

永久节点心跳检测在《Nacos2# 服务注册与发现客户端示例与源码解析（二）》HTTP心跳检测器有详细分析，这里把内容要点摘录如下：

* HTTP心跳检测只适用于注册的节点持久节点，临时节点会使用grpc代理
* 心跳执行器通过每隔五秒中向Nacos Server发起HTTP请求
* 如果返回的server not found会向Nacos Server发起注册请求重新注册



# 持久节点探活

持久节点探活支持HTTP、TCP和Mysql几种类型，下面以HTTP为例分析其运行逻辑。

### 探活示例

当注册时节点设置为不健康即Healthy设置为false，当服务端探活正常后将节点设置为健康即Healthy为true。

```java
@SpringBootApplication
public class BootApplication {
public static void main(String[] args) throws Exception {
  SpringApplication.run(BootApplication.class, args);
  System.setProperty("serverAddr", "127.0.0.1:8848");
  System.setProperty("namespace", "public");


  Properties properties = new Properties();
  properties.setProperty("serverAddr", System.getProperty("serverAddr"));
  properties.setProperty("namespace", System.getProperty("namespace"));

  Instance instance = new Instance();
  instance.setClusterName("clusterDemo3");
  instance.setHealthy(false);//默认健康检查通过后，才认为是真正的健康
  instance.setIp(getIpAddress());
  instance.setPort(8282);
  instance.setWeight(100);
  instance.setServiceName("AppNacosDemo3");
  instance.setEphemeral(false);

  Map<String, String> map = new HashMap<String, String>();
  map.put("unit", "shunit");
  instance.setMetadata(map);

  NamingService naming = NamingFactory.createNamingService(properties);
  naming.registerInstance("AppNacosDemo3", instance);

}

private static String getIpAddress() throws SocketException {
  Enumeration<NetworkInterface> allNetInterfaces = NetworkInterface.getNetworkInterfaces();
  while (allNetInterfaces.hasMoreElements()) {
    NetworkInterface netInterface = allNetInterfaces.nextElement();
    if (!netInterface.isLoopback() && !netInterface.isVirtual() && netInterface.isUp()) {
      Enumeration<InetAddress> addresses = netInterface.getInetAddresses();
      while (addresses.hasMoreElements()) {
        InetAddress ip = addresses.nextElement();
        final String hostAddress = ip.getHostAddress();
        if (ip instanceof Inet4Address) {
          return hostAddress;
        }
      }
    }
  }
  return "";
}
}

@RestController
public class HelloController {
    @RequestMapping(value = "/bike",method = RequestMethod.GET)
    public void hello(){
        System.out.println("receive message from server.");
    }
}
```



### 探活地址设置

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210714202054.png)



探活输出

```
receive message from server.
receive message from server.
```



**小结：** 启动时注册节点为非健康节点，Nacos通过检查路径请求返回200正确后将节点设置为健康状态。



### 源码分析

当持久节点注册时，会请求到InstanceController#register方法。

```java
@Override
public void registerInstance(String namespaceId, String serviceName, Instance instance){
  boolean ephemeral = instance.isEphemeral();
  String clientId = IpPortBasedClient.getClientId(instance.toInetAddr(), ephemeral);
  createIpPortClientIfAbsent(clientId, ephemeral); // 创建IpPortClient
  Service service = getService(namespaceId, serviceName, ephemeral);
  clientOperationService.registerInstance(service, instance, clientId);
}
```

```java
public IpPortBasedClient(String clientId, boolean ephemeral) {
    this.ephemeral = ephemeral;
    this.clientId = clientId;
    this.responsibleId = getResponsibleTagFromId();
    if (ephemeral) {
        beatCheckTask = new ClientBeatCheckTaskV2(this);
        HealthCheckReactor.scheduleCheck(beatCheckTask);
    } else {
      	// 持久节点创建HealthCheckTaskV2并定时任务调度
        healthCheckTaskV2 = new HealthCheckTaskV2(this);
        HealthCheckReactor.scheduleCheck(healthCheckTaskV2);
    }
}
```

```java
@Override
public void doHealthCheck() {
  try {
    // 获取Client的service列表
    for (Service each : client.getAllPublishedService()) {
      // 开启健康检查
      if (switchDomain.isHealthCheckEnabled(each.getGroupedServiceName())) {
        // 注册节点信息
        InstancePublishInfo instancePublishInfo = client.getInstancePublishInfo(each);
        ClusterMetadata metadata = getClusterMetadata(each, instancePublishInfo);
        // 调用
        ApplicationUtils.getBean(HealthCheckProcessorV2Delegate.class).process(this, each, metadata);
        // ...
      }
    }
  } catch (Throwable e) {
   	
  } finally {
    if (!cancelled) {
      // 下一轮调度
      HealthCheckReactor.scheduleCheck(this);
      if (this.getCheckRtWorst() > 0) {
          // ...
          }
        }
      }
    }
  }
}
```

备注：定时任务调度，不断向服务的注册节点发送探活请求。



**探活处理器选择**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210714191750.png)

备注：由运行时缓存情况可以看出，支持TPC、HTTP、MYSQL三种类型探活处理。



**HTTP探活** 

代码坐标HttpHealthCheckProcessor#process

```java
@Override
public void process(HealthCheckTaskV2 task, Service service, ClusterMetadata metadata) {
    HealthCheckInstancePublishInfo instance = (HealthCheckInstancePublishInfo) task.getClient()
            .getInstancePublishInfo(service);
    if (null == instance) {
        return;
    }
    try {
        // TODO handle marked(white list) logic like v1.x.
        if (!instance.tryStartCheck()) {
           	// ...
            return;
        }
        Http healthChecker = (Http) metadata.getHealthChecker();
        // 默认使用注册实例端口
        int ckPort = metadata.isUseInstancePortForCheck() ? instance.getPort() : metadata.getHealthyCheckPort();
        // 组织url请求
        URL host = new URL("http://" + instance.getIp() + ":" + ckPort);
        URL target = new URL(host, healthChecker.getPath());
        Map<String, String> customHeaders = healthChecker.getCustomHeaders();
        Header header = Header.newInstance();
        header.addAll(customHeaders);
        // 发送HTTP请求
        ASYNC_REST_TEMPLATE.get(target.toString(), header, Query.EMPTY, String.class,
                new HttpHealthCheckCallback(instance, task, service));
        MetricsMonitor.getHttpHealthCheckMonitor().incrementAndGet();
    } catch (Throwable e) {
        instance.setCheckRt(switchDomain.getHttpHealthParams().getMax());
        healthCheckCommon.checkFail(task, service, "http:error:" + e.getMessage());
        healthCheckCommon.reEvaluateCheckRT(switchDomain.getHttpHealthParams().getMax(), task,
                switchDomain.getHttpHealthParams());
    }
}
```

```java
@Override
public void onReceive(RestResult<String> result) {
    instance.setCheckRt(System.currentTimeMillis() - startTime);
    int httpCode = result.getCode();
    // 返回200
    if (HttpURLConnection.HTTP_OK == httpCode) {
        // 将节点变更为健康
        healthCheckCommon.checkOk(task, service, "http:" + httpCode);
        healthCheckCommon.reEvaluateCheckRT(System.currentTimeMillis() - startTime, task,
                switchDomain.getHttpHealthParams());
    } else if (HttpURLConnection.HTTP_UNAVAILABLE == httpCode
            || HttpURLConnection.HTTP_MOVED_TEMP == httpCode) {
       
    } else {
      
    }
}

 public void checkOk(HealthCheckTaskV2 task, Service service, String msg) {
   // 节点设置为健康状态
   healthStatusSynchronizer.instanceHealthStatusChange(true, task.getClient(), service, instance);
 }
```

备注：向节点发起HTTP请求，返回状态码为200表示，将节点标志为健康状态；否则标记非健康状态。



**TCP探活**

TPC探活的代码详见TcpHealthCheckProcessor，不再详细分析。大体逻辑为通过与注册实例建立channel，不断ping 注册实例的端口是否可用，从而判断服务是否健康。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210715192801.png)



### MYSQL探活

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210715173009.png)

备注：主要检查当前节点为主库，不能访问到从库，可能在主从切换中使用。



**总结：** 本文就临时节点续约、持久节点心跳、持久节点的探活代码实现做了熟练。相信通过代码走查，对其使用场景和实现不再陌生。











