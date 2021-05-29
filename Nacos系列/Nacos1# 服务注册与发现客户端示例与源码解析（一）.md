---
title: Nacos1# 服务注册与发现客户端示例与源码解析（一）
categories: Nacos
tags: Nacos
date: 2021-05-16 11:55:01
---



# 引言

Nacos在业界注册中心的选型中举足轻重，值得去深入分析和研究。本文就注册和发现客户端的初始话逻辑从源码角度分析其做了什么事情，另外，其服务发现的设计架构可作为我们相似场景设计的模型作为参考。



# 内容提要



### 可配置参数

* 默认命名空间public，可通过System.setProperty(PropertyKeyConst.NAMESPACE, ""）设置
* 默认web根目录为/nacos/v1/ns，可通过System.setProperty(PropertyKeyConst.CONTEXT_PATH, ""）设置
* 默认日志文件名称为naming.log，可通过System.setProperty(UtilAndComs.NACOS_NAMING_LOG_NAME, ""）设置
* 支持通过动态刷新EndPoint获取server地址列表，EndPoint地址可通过properties.setProperty(PropertyKeyConst.ENDPOINT,"")设置，EndPoint刷新的频率是30秒
* 支持直接传入Server地址properties.setProperty(PropertyKeyConst.SERVER_ADD,"")



### 服务发现逻辑

服务发现逻辑也就是当实例变更时通知给订阅者逻辑，详细逻辑如下：

* 当我们开启订阅时subscribe时，会通过调度器生成一个UpdateTask；UpdateTask每个6秒钟（最长为1分钟）会从注册中心获取实例Instance列表，当检测到实例Instance列表有变更时会通过NotifyCenter.publishEvent发布实例变更事件
* NotifyCenter是个门面类，对DefaultPublisher的操作，以及DefaultPublisher与关联事件的映射，例如：会绑定ChangeEvent与EventPublisher的关系；上面发布的实例变更事件实际为添加到DefaultPublisher的阻塞队列
* DefaultPublisher中维护一个订阅者集合subscribers；DefaultPublisher中维护一个事件阻塞队列queue默认大小为16384；DefaultPublisher同时也是一个线程类初始化时通过for死循环从阻塞队列queue中获取Event，并循环回调订阅者subscribers执行该Event
* subscribers执行Event，具体回调到InstancesChangeNotifier#onEvent，进而回调到我们订阅时提供的AbstractEventListener#onEvent，从而实现我们的发现逻辑。



### 故障转移逻辑

* 在ServiceInfoHolder初始化初始化时，会生成本地缓存目录 ${user.home}/nacos/naming
* 每10秒钟将ServiceInfo备份到缓存文件中
* 故障转移开启生效实例化延迟5秒钟会从本地文件将ServiceInfo读入缓存serviceMap
* 如果配置参数「namingLoadCacheAtStart」设置为true启动时会从本地缓存文件读取ServiceInfo信息，默认为false



# 注册与发现示例

**服务注册示例**

```java
@Test
public void registerTest() throws Exception {

  System.setProperty("serverAddr", "127.0.0.1:8848");
  System.setProperty("namespace", "public");

  Properties properties = new Properties();
  properties.setProperty("serverAddr", System.getProperty("serverAddr"));
  properties.setProperty("namespace", System.getProperty("namespace"));

  NamingService naming = NamingFactory.createNamingService(properties);

  naming.registerInstance("nacos.test.3", "11.11.11.11", 8888, "TEST1");
  
  System.out.println(naming.getAllInstances("nacos.test.3"));

	System.in.read();
}
```

**输出**

```
[Instance{instanceId='null', ip='11.11.11.11', port=8888, weight=1.0, healthy=true, enabled=true, ephemeral=true, clusterName='TEST1', serviceName='DEFAULT_GROUP@@nacos.test.3', metadata={}}]
```



**服务发现示例**

```java
@Test
public void subscribeTest() throws Exception {

        System.setProperty("serverAddr", "127.0.0.1:8848");
        System.setProperty("namespace", "public");

        Properties properties = new Properties();
        properties.setProperty("serverAddr", System.getProperty("serverAddr"));
        properties.setProperty("namespace", System.getProperty("namespace"));

        NamingService naming = NamingFactory.createNamingService(properties);
        Executor executor = new ThreadPoolExecutor(1, 1, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>(),
            new ThreadFactory() {
                @Override
                public Thread newThread(Runnable r) {
                    Thread thread = new Thread(r);
                    thread.setName("test-thread");
                    return thread;
                }
            });

        naming.subscribe("nacos.test.3", new AbstractEventListener() {

            @Override
            public Executor getExecutor() {
                return executor;
            }

            @Override
            public void onEvent(Event event) {
                System.out.println("订阅到的服务：" + ((NamingEvent) event).getServiceName());
                System.out.println("订阅到的实例：" + ((NamingEvent) event).getInstances());
            }
        });

        System.in.read();

}
```

**输出**

```
订阅的服务：nacos.test.3
订阅的实例：[Instance{instanceId='null', ip='11.11.11.11', port=8888, weight=1.0, healthy=true, enabled=true, ephemeral=true, clusterName='TEST1', serviceName='DEFAULT_GROUP@@nacos.test.3', metadata={}}]
```





# NacosNamingService初始化源码分析

反射实例化

```java
Class<?> driverImplClass = Class.forName("com.alibaba.nacos.client.naming.NacosNamingService");
Constructor constructor = driverImplClass.getConstructor(Properties.class);
NamingService vendorImpl = (NamingService) constructor.newInstance(properties);
```

```java
public NacosNamingService(Properties properties) throws NacosException {
        init(properties);
}
```

```java
private void init(Properties properties) throws NacosException {
        ValidatorUtils.checkInitParam(properties); // 注解@1 
  
				this.namespace = InitUtils.initNamespaceForNaming(properties); // 注解@2

        InitUtils.initSerialization();

        InitUtils.initWebRootContext(properties); // 注解@3

        initLogName(properties); // 注解@4

        // 注解@5...............................................
  			this.changeNotifier = new InstancesChangeNotifier();

        NotifyCenter.registerToPublisher(InstancesChangeEvent.class, 16384);

        NotifyCenter.registerSubscriber(changeNotifier);
  			// ................................................
				
  			// 注解@6
        this.serviceInfoHolder = new ServiceInfoHolder(namespace, properties);
  		  // 注解@7
        this.clientProxy = new NamingClientProxyDelegate(this.namespace, serviceInfoHolder, properties, changeNotifier);
    }

```

代码注解：

**注解@1**  校验contextPath非法字符，默认路径为/nacos

**注解@2** 获取命名空间，可以通过System.setProperty和Properties设置命名空间，默认为public 

```java
public static String initNamespaceForNaming(Properties properties) {
        String tmpNamespace = null;
        /**
         * 阿里云上也提供注册发现产品服务，兼容云上的命名空间设置
         */
        String isUseCloudNamespaceParsing = properties.getProperty(PropertyKeyConst.IS_USE_CLOUD_NAMESPACE_PARSING,
                System.getProperty(SystemPropertyKeyConst.IS_USE_CLOUD_NAMESPACE_PARSING,
                        String.valueOf(Constants.DEFAULT_USE_CLOUD_NAMESPACE_PARSING)));

        if (Boolean.parseBoolean(isUseCloudNamespaceParsing)) {

            tmpNamespace = TenantUtil.getUserTenantForAns();
            tmpNamespace = TemplateUtils.stringEmptyAndThenExecute(tmpNamespace, new Callable<String>() {
                @Override
                public String call() {
                    String namespace = System.getProperty(SystemPropertyKeyConst.ANS_NAMESPACE);
                    LogUtils.NAMING_LOGGER.info("initializer namespace from System Property :" + namespace);
                    return namespace;
                }
            });

            tmpNamespace = TemplateUtils.stringEmptyAndThenExecute(tmpNamespace, new Callable<String>() {
                @Override
                public String call() {
                    String namespace = System.getenv(PropertyKeyConst.SystemEnv.ALIBABA_ALIWARE_NAMESPACE);
                    LogUtils.NAMING_LOGGER.info("initializer namespace from System Environment :" + namespace);
                    return namespace;
                }
            });
        }

        /**
         * 非阿里云注册产品，使用自定义的
         * 获取服务启动设置的namespace
         */
        tmpNamespace = TemplateUtils.stringEmptyAndThenExecute(tmpNamespace, new Callable<String>() {
            @Override
            public String call() {
                String namespace = System.getProperty(PropertyKeyConst.NAMESPACE);
                LogUtils.NAMING_LOGGER.info("initializer namespace from System Property :" + namespace);
                return namespace;
            }
        });

        /**
         * 也可以通过properties获取namespace
         */
        if (StringUtils.isEmpty(tmpNamespace) && properties != null) {
            tmpNamespace = properties.getProperty(PropertyKeyConst.NAMESPACE);
        }

        /**
         * 如果System.getProperty和Properties都没有设置命名空间，使用默认的public
         */
        tmpNamespace = TemplateUtils.stringEmptyAndThenExecute(tmpNamespace, new Callable<String>() {
            @Override
            public String call() {
                return UtilAndComs.DEFAULT_NAMESPACE_ID;
            }
        });
        return tmpNamespace;
}
```

**注解@3** 设置web root context，其中：

webContext：/nacos

nacosUrlBase：webContext + "/v1/ns"，默认 /nacos/v1/ns

nacosUrlInstance：nacosUrlBase + "/instance"，默认为 /nacos/v1/ns/instance

```java
public static void initWebRootContext(Properties properties) {
        final String webContext = properties.getProperty(PropertyKeyConst.CONTEXT_PATH);
        TemplateUtils.stringNotEmptyAndThenExecute(webContext, new Runnable() {
            @Override
            public void run() {
                UtilAndComs.webContext = ContextPathUtil.normalizeContextPath(webContext);
                UtilAndComs.nacosUrlBase = UtilAndComs.webContext + "/v1/ns";
                UtilAndComs.nacosUrlInstance = UtilAndComs.nacosUrlBase + "/instance";
            }
        });
        // 已废弃：通过-Dnacos.naming.web.context设置contextPath
        initWebRootContext();
}
```

**@注解4**  自定义日志名称，可以通过properties或者System中设置com.alibaba.nacos.naming.log.filename指定名称，默认为naming.log

```java
private void initLogName(Properties properties) {
        logName = System.getProperty(UtilAndComs.NACOS_NAMING_LOG_NAME);
        if (StringUtils.isEmpty(logName)) {

            if (properties != null && StringUtils
                    .isNotEmpty(properties.getProperty(UtilAndComs.NACOS_NAMING_LOG_NAME))) {
                logName = properties.getProperty(UtilAndComs.NACOS_NAMING_LOG_NAME);
            } else {
                logName = "naming.log";
            }
        }
}
```



<u>**小结：** 默认为命名空间为public，可以通过**System.setProperty(PropertyKeyConst.NAMESPACE, ""）**和Properties设置；默认web root context为 「/nacos/v1/ns」，可以通过参数**System.setProperty(PropertyKeyConst.CONTEXT_PATH, ""）**设置；nacos日志文件名称默认为naming.log，可以通过参数**System.setProperty(UtilAndComs.NACOS_NAMING_LOG_NAME, ""）**设置。</u>



**@注解5**  通过NotifyCenter注册了一个Publisher和Subscriber，另起一小节。



# NotifyCenter与DefaultPublisher源码分析

**NotifyCenter静态块赋值** 

```java
static {
        // @注解5.1 默认为DefaultPublisher中的BlockingQueue长度
        // 长度默认16384，可以通过System.setProperty("nacos.core.notify.ring-buffer-size","intx")设置
        String ringBufferSizeProperty = "nacos.core.notify.ring-buffer-size";
        ringBufferSize = Integer.getInteger(ringBufferSizeProperty, 16384);

        // @注解5.2 BlockingQueue长度默认1024
  		  // 可以通过System.setProperty("nacos.core.notify.share-buffer-size","intx")设置
        String shareBufferSizeProperty = "nacos.core.notify.share-buffer-size";
        shareBufferSize = Integer.getInteger(shareBufferSizeProperty, 1024);
				
        // @注解5.3 指定Class EventPublisher默认为DefaultPublisher，可以通过SPI设置
  		  final Collection<EventPublisher> publishers = NacosServiceLoader.load(EventPublisher.class);
        Iterator<EventPublisher> iterator = publishers.iterator();

        if (iterator.hasNext()) {
            clazz = iterator.next().getClass(); // 赋值 SPI EventPublisher
        } else {
            clazz = DefaultPublisher.class; // 赋值默认值
        }
				
        // @注解5.4 EventPublisher初始化，默认为DefaultPublisher的初始化
        publisherFactory = new BiFunction<Class<? extends Event>, Integer, EventPublisher>() {

            @Override
            public EventPublisher apply(Class<? extends Event> cls, Integer buffer) {
                try {
                    // 实例化EventPublisher（默认DefaultPublisher）
                    EventPublisher publisher = clazz.newInstance();
                    publisher.init(cls, buffer);
                    return publisher;
                } catch (Throwable ex) {
                    LOGGER.error("Service class newInstance has error : {}", ex);
                    throw new NacosRuntimeException(SERVER_ERROR, ex);
                }
            }
        };

        try {

            // @注解5.5 DefaultSharePublisher初始化
            INSTANCE.sharePublisher = new DefaultSharePublisher();
            INSTANCE.sharePublisher.init(SlowEvent.class, shareBufferSize);

        } catch (Throwable ex) {
            LOGGER.error("Service class newInstance has error : {}", ex);
        }

        ThreadUtils.addShutdownHook(new Runnable() {
            @Override
            public void run() {
                shutdown();
            }
        });
}
```

**@注解5.4** DefaultPublisher的初始化，其本身继承了Thread，初始化了ArrayBlockingQueue其大小为ringBufferSize默认16384

```java
@Override
public void init(Class<? extends Event> type, int bufferSize) {
        setDaemon(true); // 守护线程
        setName("nacos.publisher-" + type.getName()); // 设置线程名字
        this.eventType = type;
        this.queueMaxSize = bufferSize;
        // 阻塞队列初始化
        this.queue = new ArrayBlockingQueue<Event>(bufferSize);
        start();
}
```

再看下其线程启动时在做什么事情，可以看到一个for死循环不断的从队列中取出Event，并通知订阅者Subscriber执行Event

```java
void openEventHandler() {
        try {

            int waitTimes = 60;
            for (; ; ) {
                if (shutdown || hasSubscriber() || waitTimes <= 0) {
                    break;
                }
                ThreadUtils.sleep(1000L);
                waitTimes--;
            }

            for (; ; ) {
                if (shutdown) {
                    break;
                }
                final Event event = queue.take(); // 从队列取出Event
                receiveEvent(event);
                UPDATER.compareAndSet(this, lastEventSequence, Math.max(lastEventSequence, event.sequence()));
            }
        } catch (Throwable ex) {
            LOGGER.error("Event listener exception : {}", ex);
        }
}
```

```java
 void receiveEvent(Event event) {
        final long currentEventSequence = event.sequence();
        // 通知订阅者执行Event
        for (Subscriber subscriber : subscribers) {
            // Whether to ignore expiration events
            if (subscriber.ignoreExpireEvent() && lastEventSequence > currentEventSequence) {
                LOGGER.debug("[NotifyCenter] the {} is unacceptable to this subscriber, because had expire",
                        event.getClass());
                continue;
            }
            notifySubscriber(subscriber, event);
        }
    }
    
    @Override
    public void notifySubscriber(final Subscriber subscriber, final Event event) {

        LOGGER.debug("[NotifyCenter] the {} will received by {}", event, subscriber);

        final Runnable job = () -> subscriber.onEvent(event); // 执行订阅者Event
        final Executor executor = subscriber.executor();

        if (executor != null) {
            executor.execute(job);
        } else {
            try {
                job.run();
            } catch (Throwable e) {
                LOGGER.error("Event callback exception: ", e);
            }
        }
    }
```



**@注解5.5** DefaultSharePublisher继承自DefaultPublisher，处理SlowEvent事件，处理架构与DefaultPublisher一致。



**绑定ChangeEvent与Publisher** 

```java
this.changeNotifier = new InstancesChangeNotifier();

NotifyCenter.registerToPublisher(InstancesChangeEvent.class, 16384)
```

Publisher的注册过程在于建立InstancesChangeEvent.class与EventPublisher的关系。

默认为Map<String, EventPublisher> publisherMap，key为com.alibaba.nacos.client.naming.event.InstancesChangeEvent，value为DefaultPublisher实例。

```java
public static EventPublisher registerToPublisher(final Class<? extends Event> eventType, final int queueMaxSize) {
        if (ClassUtils.isAssignableFrom(SlowEvent.class, eventType)) {
            return INSTANCE.sharePublisher;
        }
        // topic = com.alibaba.nacos.client.naming.event.InstancesChangeEvent
        final String topic = ClassUtils.getCanonicalName(eventType);
        synchronized (NotifyCenter.class) {
            // MapUtils.computeIfAbsent is a unsafe method.
            MapUtil.computeIfAbsent(INSTANCE.publisherMap, topic, publisherFactory, eventType, queueMaxSize);
        }
        return INSTANCE.publisherMap.get(topic);
}
```

**将Subscribe注册到Publisher**

```java
this.changeNotifier = new InstancesChangeNotifier();

NotifyCenter.registerSubscriber(changeNotifier);
```

上面提到Publisher中维护了一个subscribers集合，这行代码即将InstancesChangeNotifier,添加到该集合，InstancesChangeNotifier继承了Subscriber。

```java
private static void addSubscriber(final Subscriber consumer, Class<? extends Event> subscribeType) {
        final String topic = ClassUtils.getCanonicalName(subscribeType);
   synchronized (NotifyCenter.class) {
   		// MapUtils.computeIfAbsent is a unsafe method.
   		MapUtil.computeIfAbsent(INSTANCE.publisherMap, topic, publisherFactory, subscribeType, ringBufferSize);
   }
   // 获取时间对应的Publisher
   // key = com.alibaba.nacos.client.naming.event.InstancesChangeEvent
   // value = DefaultPublisher
   EventPublisher publisher = INSTANCE.publisherMap.get(topic);
   // 添加到subscribers集合
   publisher.addSubscriber(consumer);
}
```



<u>**小结：** DefaultPublisher中维护一个订阅者集合subscribers；DefaultPublisher中维护一个事件阻塞队列queue默认大小为16384；DefaultPublisher同时也是一个线程类初始化时通过for死循环从阻塞队列queue中获取Event，并循环回调订阅者subscribers执行该Event；NotifyCenter是操作DefaultPublisher的门面类，会绑定ChangeEvent与EventPublisher的关系，并将InstancesChangeNotifier添加到了DefaultPublisher的subscribers集合。</u>



**注解@6** ServiceInfoHolder初始化，另起一小节分析



# ServiceInfoHolder初始化源码分析

```java
public ServiceInfoHolder(String namespace, Properties properties) {
    // 注解@6.1 
    initCacheDir(namespace);
    // 注解@6.2
    if (isLoadCacheAtStart(properties)) {
        this.serviceInfoMap = new ConcurrentHashMap<String, ServiceInfo>(DiskCache.read(this.cacheDir));
    } else {
        this.serviceInfoMap = new ConcurrentHashMap<String, ServiceInfo>(16);
    }
    // 注解@6.3
    this.failoverReactor = new FailoverReactor(this, cacheDir);
    this.pushEmptyProtection = isPushEmptyProtect(properties);
}
```

**@注解6.1** 生成缓存目录：默认为${user.home}/nacos/naming/public，可以通过System.setProperty("JM.SNAPSHOT.PATH")自定义根目录

**@注解6.2** 启动时是否从缓存目录读取信息，默认false。设置为true会读取缓存文件

**@注解6.3** 故障转移相关

故障转移目录：${user.home}/nacos/naming/public/failover

故障转移开关文件：${user.home}/nacos/naming/public/failover/00-00---000-VIPSRV_FAILOVER_SWITCH-000---00-00

故障转移关闭：当故障转移开关文件不存在时或者文件的值为0

故障转移开启：当故障转移开关文件存在时或者文件的值为1

故障转移检查：延迟5秒将缓存文件ServiceInfo信息读入缓存（由FailoverReactor#SwitchRefresher负责）

当故障转移开关开启，更新缓存switchParams.put("failover-mode", "true")，同时启动FailoverFileReader线程读取目录failover文件ServiceInfo内容。例如：DEFAULT_GROUP%40%40nacos.test.3，这些信息被读入到内存Map<String, ServiceInfo> serviceMap中。

```json
{
    "name": "nacos.test.3",
    "groupName": "DEFAULT_GROUP",
    "clusters": "",
    "cacheMillis": 10000,
    "hosts": [
        {
            "ip": "11.11.11.11",
            "port": 8888,
            "weight": 1,
            "healthy": true,
            "enabled": true,
            "ephemeral": true,
            "clusterName": "TEST1",
            "serviceName": "DEFAULT_GROUP@@nacos.test.3",
            "metadata": {},
            "instanceHeartBeatTimeOut": 15000,
            "ipDeleteTimeout": 30000,
            "instanceIdGenerator": "simple",
            "instanceHeartBeatInterval": 5000
        }
    ],
    "lastRefTime": 1618601660155,
    "checksum": "",
    "allIPs": false,
    "reachProtectionThreshold": false,
    "valid": true
}
```

故障数据备份：每10秒钟备份一次（FailoverReactor#DiskFileWriter），会把ServiceInfo即上面json内容备份到文件中。



SwitchRefresher工作过程

```java
class SwitchRefresher implements Runnable {
        
        long lastModifiedMillis = 0L;
        
        @Override
        public void run() {
            try {
                File switchFile = new File(failoverDir + UtilAndComs.FAILOVER_SWITCH);
                // 文件不存在退出
                if (!switchFile.exists()) {
                    switchParams.put("failover-mode", "false");
                    NAMING_LOGGER.debug("failover switch is not found, " + switchFile.getName());
                    return;
                }
                
                long modified = switchFile.lastModified();
                
                if (lastModifiedMillis < modified) {
                    lastModifiedMillis = modified;
                    // 获取故障转移文件内容
                    String failover = ConcurrentDiskUtil.getFileContent(failoverDir + UtilAndComs.FAILOVER_SWITCH,Charset.defaultCharset().toString());
                    if (!StringUtils.isEmpty(failover)) {
                        String[] lines = failover.split(DiskCache.getLineSeparator());
                        for (String line : lines) {
                            String line1 = line.trim();
                            // 1表示开启故障转移模式
                            if ("1".equals(line1)) {
                                switchParams.put("failover-mode", "true");
                                NAMING_LOGGER.info("failover-mode is on");
                                new FailoverFileReader().run();
                            // 0表示关闭故障转移模式
                            } else if ("0".equals(line1)) {
                                switchParams.put("failover-mode", "false");
                                NAMING_LOGGER.info("failover-mode is off");
                            }
                        }
                    } else {
                        switchParams.put("failover-mode", "false");
                    }
                }
            } catch (Throwable e) {
                NAMING_LOGGER.error("[NA] failed to read failover switch.", e);
            }
        }
}
```



FailoverFileReader工作过程，主要将Json内容读取缓存

```java
 class FailoverFileReader implements Runnable {
 	 // ...
   String dataString = ConcurrentDiskUtil
                                .getFileContent(file, Charset.defaultCharset().toString());
   reader = new BufferedReader(new StringReader(dataString));
                        
   String json;
   if ((json = reader.readLine()) != null) {
     try {
       dom = JacksonUtils.toObj(json, ServiceInfo.class);
     } catch (Exception e) {
       NAMING_LOGGER.error("[NA] error while parsing cached dom : " + json, e);
     }
   }
   // ... 读入缓存
   if (!CollectionUtils.isEmpty(dom.getHosts())) {
        domMap.put(dom.getKey(), dom);
   }
   if (domMap.size() > 0) {
       serviceMap = domMap;
   }
 }
```



DiskFileWriter工作过程

```java
class DiskFileWriter extends TimerTask {
        
   @Override
   public void run() {
     Map<String, ServiceInfo> map = serviceInfoHolder.getServiceInfoMap();
          for (Map.Entry<String, ServiceInfo> entry : map.entrySet()) {
                ServiceInfo serviceInfo = entry.getValue();
                if (StringUtils.equals(serviceInfo.getKey(), UtilAndComs.ALL_IPS) || StringUtils
                        .equals(serviceInfo.getName(), UtilAndComs.ENV_LIST_KEY) || StringUtils
                        .equals(serviceInfo.getName(), "00-00---000-ENV_CONFIGS-000---00-00") || StringUtils
                        .equals(serviceInfo.getName(), "vipclient.properties") || StringUtils
                        .equals(serviceInfo.getName(), "00-00---000-ALL_HOSTS-000---00-00")) {
                    continue;
                }
                
                DiskCache.write(serviceInfo, failoverDir);
       	 }
    }
}

// 将缓存内容写入磁盘文件
public static void write(ServiceInfo dom, String dir) {

        try {
            makeSureCacheDirExists(dir);

            File file = new File(dir, dom.getKeyEncoded());
            if (!file.exists()) {
                // add another !file.exists() to avoid conflicted creating-new-file from multi-instances
                if (!file.createNewFile() && !file.exists()) {
                    throw new IllegalStateException("failed to create cache file");
                }
            }

            StringBuilder keyContentBuffer = new StringBuilder();

            String json = dom.getJsonFromServer();

            if (StringUtils.isEmpty(json)) {
                json = JacksonUtils.toJson(dom);
            }

            keyContentBuffer.append(json);
					 
            ConcurrentDiskUtil.writeFileContent(file, keyContentBuffer.toString(), Charset.defaultCharset().toString());

        } catch (Throwable e) {
            NAMING_LOGGER.error("[NA] failed to write cache for dom:" + dom.getName(), e);
        }
}

```



<u>**小结：** 在ServiceInfoHolder初始化初始化时，会生成本地缓存目录 ${user.home}/nacos/naming；每10秒钟将ServiceInfo备份到缓存文件中；故障转移开启生效实例化延迟5秒钟会从本地文件将ServiceInfo读入缓存serviceMap；如果配置参数「namingLoadCacheAtStart」设置为true启动时会从本地缓存文件读取ServiceInfo信息，默认为false。</u>



**注解@7** 注册客户端委派代理类初始化

```java
public NamingClientProxyDelegate(String namespace, ServiceInfoHolder serviceInfoHolder, Properties properties,
            InstancesChangeNotifier changeNotifier) throws NacosException {
  		  // @注解7.1
        this.serviceInfoUpdateService = new ServiceInfoUpdateService(properties, serviceInfoHolder, this,
                changeNotifier);
  			// @注解7.2
        this.serverListManager = new ServerListManager(properties);
  		  this.serviceInfoHolder = serviceInfoHolder;
        this.securityProxy = new SecurityProxy(properties, NamingHttpClientManager.getInstance().getNacosRestTemplate());
        initSecurityProxy();
        this.httpClientProxy = new NamingHttpClientProxy(namespace, securityProxy, serverListManager, properties,serviceInfoHolder);
        this.grpcClientProxy = new NamingGrpcClientProxy(namespace, securityProxy, serverListManager, properties,serviceInfoHolder);
}
```

**注解@7.1** ServiceInfoUpdateService初始化，另起一章分析



# ServiceInfoUpdateService初始化源码分析



```java
public ServiceInfoUpdateService(Properties properties, ServiceInfoHolder serviceInfoHolder,
            NamingClientProxy namingClientProxy, InstancesChangeNotifier changeNotifier) {
  			// @注解7.1.1
  			this.executor = new ScheduledThreadPoolExecutor(initPollingThreadCount(properties),
                new NameThreadFactory("com.alibaba.nacos.client.naming.updater"));
        this.serviceInfoHolder = serviceInfoHolder;
        this.namingClientProxy = namingClientProxy;
        this.changeNotifier = changeNotifier;
 }
```

**注解@7.1.1**  定时任务调度执行器，线程池大小为处理器核数的一半，可以通过参数"namingPollingThreadCount”指定

职责：调度器用于执行UpdateTask，延迟1秒执行。

```java
private synchronized ScheduledFuture<?> addTask(UpdateTask task) {
        return executor.schedule(task, DEFAULT_DELAY, TimeUnit.MILLISECONDS);
}
```

UpdateTask执行逻辑：

```java
public void run() {
    long delayTime = DEFAULT_DELAY;   
    try {
      // 判断该注册的Service是否被订阅，如果没有订阅则不再执行
      if (!changeNotifier.isSubscribed(groupName, serviceName, clusters) && !futureMap.containsKey(serviceKey)) 			{
        NAMING_LOGGER.info("update task is stopped, service:" + groupedServiceName + ", clusters:" + clusters);
        return;
      }
      // 获取缓存的service信息          
      ServiceInfo serviceObj = serviceInfoHolder.getServiceInfoMap().get(serviceKey);
      if (serviceObj == null) {
        // 根据serviceName从注册中心服务端获取Service信息
        serviceObj = namingClientProxy.queryInstancesOfService(serviceName, groupName, clusters, 0, false);
        serviceInfoHolder.processServiceInfo(serviceObj);
        lastRefTime = serviceObj.getLastRefTime();
        return;
      }
			// 过期服务（服务的最新更新时间小于等于缓存刷新时间），从注册中心重新查询
      if (serviceObj.getLastRefTime() <= lastRefTime) {
        serviceObj = namingClientProxy.queryInstancesOfService(serviceName, groupName, clusters, 0, false);
        // 处理Service消息
        serviceInfoHolder.processServiceInfo(serviceObj);
      }
      // 刷新更新时间
      lastRefTime = serviceObj.getLastRefTime();
      if (CollectionUtils.isEmpty(serviceObj.getHosts())) {
        incFailCount();
        return;
      }
      // 下次更新缓存时间设置，默认为6秒
      delayTime = serviceObj.getCacheMillis() * DEFAULT_UPDATE_CACHE_TIME_MULTIPLE;
      // 重置失败数量为0
      resetFailCount();
    } catch (Throwable e) {
      incFailCount();
      NAMING_LOGGER.warn("[NA] failed to update serviceName: " + groupedServiceName, e);
    } finally {
      // 下次调度刷新时间，下次执行的时间与failCount有关
      // failCount=0，则下次调度时间为6秒，最长为1分钟
      // 即当无异常情况下缓存实例的刷新时间是6秒
      executor.schedule(this, Math.min(delayTime << failCount, DEFAULT_DELAY * 60), TimeUnit.MILLISECONDS);
    }
}
```

**备注：** UpdateTask主要逻辑为如果服务缓存刷新时间过期，则会从注册中心查询最新服务信息，同时刷新缓存更新时间。并定时调度去更新服务注册信息，更新的频率最小为6秒，最长为1分钟。当更新无异常时更新频率为6秒，当发生异常时最长频率为1分钟。



另外如果过期还会调用serviceInfoHolder#processServiceInfo处理服务信息，下面看下其执行逻辑：

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

**备注：** 服务实例信息会被缓存在serviceInfoMap中，key为「goupName@@ServiceName」例如：DEFAULT_GROUP@@nacos.test.3；serviceInfoMap的大小会通过prometheus simpleclient统计监控；如果服务信息有更新，会通过 NotifyCenter.publishEvent发布实例变更事件，订阅该服务的的订阅者Subscribes将会处理该事件；将缓存服务信息保存到本地文件容灾。



下面看下如何判断服务实例信息变更的：实例信息修改、删除、新增均属于实例变更。

```java
private boolean isChangedServiceInfo(ServiceInfo oldService, ServiceInfo newService) {
        if (null == oldService) {
            NAMING_LOGGER.info("init new ips(" + newService.ipCount() + ") service: " + newService.getKey() + " -> "
                    + JacksonUtils.toJson(newService.getHosts()));
            return true;
        }
        if (oldService.getLastRefTime() > newService.getLastRefTime()) {
            NAMING_LOGGER
                    .warn("out of date data received, old-t: " + oldService.getLastRefTime() + ", new-t: " + newService
                            .getLastRefTime());
        }
        boolean changed = false;
        Map<String, Instance> oldHostMap = new HashMap<String, Instance>(oldService.getHosts().size());
        for (Instance host : oldService.getHosts()) {
            oldHostMap.put(host.toInetAddr(), host);
        }
        Map<String, Instance> newHostMap = new HashMap<String, Instance>(newService.getHosts().size());
        for (Instance host : newService.getHosts()) {
            newHostMap.put(host.toInetAddr(), host);
        }

        // 变更的实例集合
        Set<Instance> modHosts = new HashSet<Instance>();
        // 新增的实例集合
        Set<Instance> newHosts = new HashSet<Instance>();
        // 删除的实例集合
        Set<Instance> remvHosts = new HashSet<Instance>();

        List<Map.Entry<String, Instance>> newServiceHosts = new ArrayList<Map.Entry<String, Instance>>(
                newHostMap.entrySet());
        for (Map.Entry<String, Instance> entry : newServiceHosts) {
            Instance host = entry.getValue();
            String key = entry.getKey();
            if (oldHostMap.containsKey(key) && !StringUtils.equals(host.toString(), oldHostMap.get(key).toString())) {
                modHosts.add(host);
                continue;
            }

            if (!oldHostMap.containsKey(key)) {
                newHosts.add(host);
            }
        }

        for (Map.Entry<String, Instance> entry : oldHostMap.entrySet()) {
            Instance host = entry.getValue();
            String key = entry.getKey();
            if (newHostMap.containsKey(key)) {
                continue;
            }

            if (!newHostMap.containsKey(key)) {
                remvHosts.add(host);
            }

        }

        if (newHosts.size() > 0) {
            changed = true;
            NAMING_LOGGER
                    .info("new ips(" + newHosts.size() + ") service: " + newService.getKey() + " -> " + JacksonUtils
                            .toJson(newHosts));
        }

        if (remvHosts.size() > 0) {
            changed = true;
            NAMING_LOGGER.info("removed ips(" + remvHosts.size() + ") service: " + newService.getKey() + " -> "
                    + JacksonUtils.toJson(remvHosts));
        }

        if (modHosts.size() > 0) {
            changed = true;
            NAMING_LOGGER.info("modified ips(" + modHosts.size() + ") service: " + newService.getKey() + " -> "
                    + JacksonUtils.toJson(modHosts));
        }
        return changed;
}
```



接着看下UpdateTask是什么时候被加入的：

```java
public void scheduleUpdateIfAbsent(String serviceName, String groupName, String clusters) {
        String serviceKey = ServiceInfo.getKey(NamingUtils.getGroupedName(serviceName, groupName), clusters);
        if (futureMap.get(serviceKey) != null) {
            return;
        }
        synchronized (futureMap) {
            if (futureMap.get(serviceKey) != null) {
                return;
            }
						// 构建UpdateTask
            ScheduledFuture<?> future = addTask(new UpdateTask(serviceName, groupName, clusters));
            futureMap.put(serviceKey, future);
        }
 }
```

```java
@Override
public ServiceInfo subscribe(String serviceName, String groupName, String clusters) throws NacosException {
    String serviceNameWithGroup = NamingUtils.getGroupedName(serviceName, groupName);
    String serviceKey = ServiceInfo.getKey(serviceNameWithGroup, clusters);
    ServiceInfo result = serviceInfoHolder.getServiceInfoMap().get(serviceKey);
    if (null == result) {
    	result = grpcClientProxy.subscribe(serviceName, groupName, clusters);
    }
  	// 定时调度UpdateTask
    serviceInfoUpdateService.scheduleUpdateIfAbsent(serviceName, groupName, clusters);
    serviceInfoHolder.processServiceInfo(result);
    return result;
}
```

**备注：** 也就是在我们开启订阅subscribe时就会生成一个UpdateTask被调度。



实例列表变更时会生成实例变更事件并通知订阅者执行，下面看下Subscribe是如何执行该事件的：

```java
 naming.subscribe("nacos.test.3", new AbstractEventListener() {
					 @Override
            public Executor getExecutor() {
                return executor;
            }

            @Override
            public void onEvent(Event event) {
                System.out.println("订阅到的1：" + ((NamingEvent) event).getServiceName());
                System.out.println("订阅到的2：" + ((NamingEvent) event).getInstances());
            }
        });
```

```java
public void registerListener(String groupName, String serviceName, String clusters, EventListener listener) {
        String key = ServiceInfo.getKey(NamingUtils.getGroupedName(serviceName, groupName), clusters);
        ConcurrentHashSet<EventListener> eventListeners = listenerMap.get(key);
        if (eventListeners == null) {
            synchronized (lock) {
                eventListeners = listenerMap.get(key);
                if (eventListeners == null) {
                    eventListeners = new ConcurrentHashSet<EventListener>();
                    listenerMap.put(key, eventListeners); // 将EventListener缓存到listenerMap
                }
            }
        }
        eventListeners.add(listener);
}
```

**备注：** 示例中传入了AbstractEventListener，同时将该EventListener缓存到listenerMap，key为「goupName@@ServiceName」例如：DEFAULT_GROUP@@nacos.test.3。



变更事件会通知到Subcribes，具体由InstancesChangeNotifier#onEvent执行，具体为使用示例中的getExecutor()执行Event。

```java
@Override
    public void onEvent(InstancesChangeEvent event) {
        String key = ServiceInfo
                .getKey(NamingUtils.getGroupedName(event.getServiceName(), event.getGroupName()), event.getClusters());
        ConcurrentHashSet<EventListener> eventListeners = listenerMap.get(key);
        if (CollectionUtils.isEmpty(eventListeners)) {
            return;
        }
        for (final EventListener listener : eventListeners) {
            final com.alibaba.nacos.api.naming.listener.Event namingEvent = transferToNamingEvent(event);
            if (listener instanceof AbstractEventListener && ((AbstractEventListener) listener).getExecutor() != null) {
              	// 调用AbstractEventListener的线程执行该Evnet
                ((AbstractEventListener) listener).getExecutor().execute(() -> listener.onEvent(namingEvent));
            } else {
                listener.onEvent(namingEvent);
            }
        }
}
```



<u>**小结：** 当我们开启订阅时subscribe时，会通过调度器生成一个UpdateTask；UpdateTask每个6秒钟（最长为1分钟）会从注册中心获取实例Instance列表，如果有变更会通过NotifyCenter.publishEvent发布实例变更事件，相关订阅者Subscribe执行该事件，也就是回调到了我们自己的onEvent方法中；另外serviceInfoMap大小通过prometheus simpleclient暴露监控指标</u>



# ServerListManager 初始化 

```java
public NamingClientProxyDelegate(String namespace, ServiceInfoHolder serviceInfoHolder, Properties properties,
            InstancesChangeNotifier changeNotifier) throws NacosException {
  		  // @注解7.1
        this.serviceInfoUpdateService = new ServiceInfoUpdateService(properties, serviceInfoHolder, this,
                changeNotifier);
  			// @注解7.2
        this.serverListManager = new ServerListManager(properties);
  		  this.serviceInfoHolder = serviceInfoHolder;
  			// @注解7.3
        this.securityProxy = new SecurityProxy(properties, NamingHttpClientManager.getInstance().getNacosRestTemplate());
        initSecurityProxy();
        this.httpClientProxy = new NamingHttpClientProxy(namespace, securityProxy, serverListManager, properties,serviceInfoHolder);
        this.grpcClientProxy = new NamingGrpcClientProxy(namespace, securityProxy, serverListManager, properties,serviceInfoHolder);
}
```

**@注解7.2:** 东西有点多，接着来ServerListManager初始化：

```java
public ServerListManager(Properties properties) {
        initServerAddr(properties); // 获取Nocas Server地址
        if (!serverList.isEmpty()) {
            currentIndex.set(new Random().nextInt(serverList.size())); // @注解7.2.2
        }
}
```

```java
private void initServerAddr(Properties properties) {
  		  // 注解@7.2.1
        this.endpoint = InitUtils.initEndpoint(properties);
        if (StringUtils.isNotEmpty(endpoint)) {
            this.serversFromEndpoint = getServerListFromEndpoint();
          	// 每30秒刷新server地址
            refreshServerListExecutor = new ScheduledThreadPoolExecutor(1,
                    new NameThreadFactory("com.alibaba.nacos.client.naming.server.list.refresher"));
            refreshServerListExecutor
                    .scheduleWithFixedDelay(this::refreshServerListIfNeed, 0, refreshServerListInternal,
                            TimeUnit.MILLISECONDS);
        } else {
          	// 直接将Nacos Server地址传入
            String serverListFromProps = properties.getProperty(PropertyKeyConst.SERVER_ADDR);
            if (StringUtils.isNotEmpty(serverListFromProps)) {
                this.serverList.addAll(Arrays.asList(serverListFromProps.split(",")));
                if (this.serverList.size() == 1) {
                    this.nacosDomain = serverListFromProps;
                }
            }
        }
}
```

**注解@7.2.1**  可配置固定Endpoint的方式获取Nacos Server地址，可以通过properties.setProperty(PropertyKeyConst.ENDPOINT,"")来设置。Endpoint可以是一个服务的域名，client每隔30秒会向「http://" + endpoint + "/nacos/serverlist」发送请求获取server list并更新列表。除了配置Endpoint外，可以通过properties.setProperty(PropertyKeyConst.SERVER_ADD,"")将nacos server地址传入到客户端。



**@注解7.2.2** 客户端会随机选择nacos server的一个地址



<u>**小结：** 在获取Nacos Server地址列表时，支持直接传入properties.setProperty(PropertyKeyConst.SERVER_ADD,"")和通过动态刷新EndPoint来更新，刷新频率为30秒。</u>



**@注解7.3** 安全代理SecurityProxy初始化，解析用户名和密码，并登陆每台server获取token；这块先不做深入分析



得，本文有点长了，剩下两个初始化，NamingHttpClientProxy和NamingGrpcClientProxy下篇接着撸。

















