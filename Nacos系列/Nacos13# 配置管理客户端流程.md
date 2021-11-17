---
title: Nacos13# 配置管理客户端流程
categories: Nacos
tags: Nacos
date: 2021-08-15 11:55:01
---



# 引言

Nacos注册中心的主要流程基本上撸完了，下面开始撸配置中心。本文从示例入手走查了客户端的初始化流程，Listener的注册逻辑和执行逻辑。



# 内容提要



### 示例

* 通过示例构建ConfigService、注册了Listener分析其流程



### Client初始化概览

* 支持多种获取server地址方式（本地、endpoint）
* 支持多种namespace设置（本地、阿里云）
* 支持超时时间、重试时间等参数设置
* 支持用户名和密码验证
* 长轮询会从BlockingQueue中获取元素，队列有元素立即执行executeConfigListen，队列无元素阻塞5秒钟执行executeConfigListen()



### Listener注册逻辑 

* client添加Listener后会在cacheMap中缓存CacheData
* cacheMap中key由「dataId+group+tenant」拼接而成
* 每个CacheData会绑定注册的Listener列表
* 每个CacheData会绑定taskId，3000个不同的CacheData对应一个taskId
* 设置isSyncWithServer=false表示 cache md5 data不是来自server同步
* BlockingQueue中添加new Object() 供长轮询判断立即执行使用



### 配置变更执行逻辑

* 执行逻辑由executeConfigListen方法实现
* 当CacheData从Server同步后，会校验md5是否变更了，变更则回调注册的Listener完成通知
* 注册Listener后会构建与server的RPC通道rpcClient
* 向server发起变更查询请求configChangeListenRequest
* Server端通过比较缓存的md5值，返回client变更的key列表
* Client通过变更的key列表向server发起配置查询请求ConfigQueryRequest
* 获取变更内容，并回调注册的Listener完成通知
* 回调注册的Listener是通过线程池异步执行Runnble Job实现的



<!--more-->



# **示例** 

```java
@Test
public void test01() throws Exception {
  String serverAddr = "localhost:8848";
  String dataId = "test";
  String group = "DEFAULT_GROUP";
  Properties properties = new Properties();
  properties.put("serverAddr", serverAddr);
  // 构建ConfigService
  ConfigService configService = NacosFactory.createConfigService(properties);
  configService.addListener(dataId, group, new Listener() {
    @Override
    public void receiveConfigInfo(String configInfo) {
      System.out.println("receive:" + configInfo);
    }

    @Override
    public Executor getExecutor() {
      return null;
    }
  });
  System.in.read();
}
```

**备注：** 示例中构建了ConfigService，注入Listener接受server配置变更通知。



# Client初始化概览

**NacosConfigService构造方法**

```java
public NacosConfigService(Properties properties) throws NacosException {
  ValidatorUtils.checkInitParam(properties);
  // 注解@1
  initNamespace(properties);
  // 注解@2
  ServerListManager serverListManager = new ServerListManager(properties);
  // 注解@3
  serverListManager.start();
  // 注解@4
  this.worker = new ClientWorker(this.configFilterChainManager, serverListManager, properties);
  // 将被废弃HttpAgent，先忽略
  // will be deleted in 2.0 later versions
  agent = new ServerHttpAgent(serverListManager);
}
```

**注解@1** 设置namespace可以通过properties.setProperty(PropertyKeyConst.NAMESPACE)，代码中会兼容阿里云环境，在此忽略，默认为空。

**注解@2** 初始化namespace、server地址等信息

**注解@3** 启动主要用于endpoint方式定时获取server地址，当本地传入isFixed=true

**注解@4** clientWorker初始化



**ClientWorker初始化**

```java
public ClientWorker(final ConfigFilterChainManager configFilterChainManager, ServerListManager serverListManager,
            final Properties properties) throws NacosException {

  this.configFilterChainManager = configFilterChainManager;

  // 注解@5
  init(properties);

  // 注解@6
  agent = new ConfigRpcTransportClient(properties, serverListManager);

  // 调度线程池，「处理器核数」
  ScheduledExecutorService executorService = Executors
    .newScheduledThreadPool(Runtime.getRuntime().availableProcessors(), new ThreadFactory() {
      @Override
      public Thread newThread(Runnable r) {
        Thread t = new Thread(r);
        t.setName("com.alibaba.nacos.client.Worker");
        t.setDaemon(true);
        return t;
      }
    });
  agent.setExecutor(executorService);

  // 注解@7
  agent.start();

}
```

**注解@5** 初始化超时时间、重试时间等

```java
private void init(Properties properties) {

    // 超时时间，默认30秒
    timeout = Math.max(ConvertUtils.toInt(properties.getProperty(PropertyKeyConst.CONFIG_LONG_POLL_TIMEOUT),
            Constants.CONFIG_LONG_POLL_TIMEOUT), Constants.MIN_CONFIG_LONG_POLL_TIMEOUT);


    // 重试时间，默认2秒
    taskPenaltyTime = ConvertUtils
            .toInt(properties.getProperty(PropertyKeyConst.CONFIG_RETRY_TIME), Constants.CONFIG_RETRY_TIME);


    // 开启配置删除同步，默认false
    this.enableRemoteSyncConfig = Boolean
            .parseBoolean(properties.getProperty(PropertyKeyConst.ENABLE_REMOTE_SYNC_CONFIG));
}
```

**注解@6** gRPC config agent初始化

```java
public ConfigTransportClient(Properties properties, ServerListManager serverListManager) {

    // 默认编码UTF-8
    String encodeTmp = properties.getProperty(PropertyKeyConst.ENCODE);
    if (StringUtils.isBlank(encodeTmp)) {
        this.encode = Constants.ENCODE;
    } else {
        this.encode = encodeTmp.trim();
    }
    // namespace租户，默认空
    this.tenant = properties.getProperty(PropertyKeyConst.NAMESPACE);

    this.serverListManager = serverListManager;

    // 用户名和密码验证
    this.securityProxy = new SecurityProxy(properties,
            ConfigHttpClientManager.getInstance().getNacosRestTemplate());

}
```

**注解@7** gRPC agent启动

```java
public void start() throws NacosException {
    // 简单用户名和密码验证
    if (securityProxy.isEnabled()) {
        securityProxy.login(serverListManager.getServerUrls());

        this.executor.scheduleWithFixedDelay(new Runnable() {
            @Override
            public void run() {
                securityProxy.login(serverListManager.getServerUrls());
            }
        }, 0, this.securityInfoRefreshIntervalMills, TimeUnit.MILLISECONDS);

    }

    startInternal();
}
```

```java
@Override
public void startInternal() throws NacosException {
    executor.schedule(new Runnable() {
        @Override
        public void run() {
            while (true) { // 一直运行
                try {
                    // 最长等待5秒
                    listenExecutebell.poll(5L, TimeUnit.SECONDS);
                    executeConfigListen();
                } catch (Exception e) {
                    LOGGER.error("[ rpc listen execute ] [rpc listen] exception", e);
                }
            }
        }
    }, 0L, TimeUnit.MILLISECONDS);

}
```

**小结：** 线程会一直运行，从BlockingQueue中获取元素。队里不为空，获取后立即执行executeConfigListen()；队列为空等待5秒后执行

executeConfigListen()。



# Listener注册逻辑 

executeConfigListen的逻辑有点复杂，先看示例代码中的添加Listener部分。

```java
configService.addListener(dataId, group, new Listener() {
    @Override
    public void receiveConfigInfo(String configInfo) {
        System.out.println("receive:" + configInfo);
    }

    @Override
    public Executor getExecutor() {
        return null;
    }
});
```

```java
public void addTenantListeners(String dataId, String group, List<? extends Listener> listeners)
        throws NacosException {

    // 默认DEFAULT_GROUP
    group = null2defaultGroup(group);

    // 租户，默认空
    String tenant = agent.getTenant();

    // 注解@8
    CacheData cache = addCacheDataIfAbsent(dataId, group, tenant);

    synchronized (cache) {
        for (Listener listener : listeners) {
            cache.addListener(listener);
        }
        // cache md5 data是否来自server同步
        cache.setSyncWithServer(false);
      	// BlockingQueue中添加new Object()
        agent.notifyListenConfig();
    }
	
}
```

**注解@8**  构建缓存数据CacheData并放入cacheMap中，缓存的key为 「dataId+group+tenant」例如：test+DEFAULT_GROUP。

每个CacheData会绑定对应的taskId，每3000个CacheData对应一个taskId。其实从后面的代码中可以看出，每个taskId会对应一个gRPC Client。

```java
public CacheData addCacheDataIfAbsent(String dataId, String group, String tenant) throws NacosException {

    // 从缓存中获取
    CacheData cache = getCache(dataId, group, tenant);
    if (null != cache) {
        return cache;
    }
    // 构造缓存key以+连接，test+DEFAULT_GROUP
    String key = GroupKey.getKeyTenant(dataId, group, tenant);
    synchronized (cacheMap) {
        CacheData cacheFromMap = getCache(dataId, group, tenant);
        // multiple listeners on the same dataid+group and race condition,so
        // double check again
        // other listener thread beat me to set to cacheMap
        if (null != cacheFromMap) { // 再检查一遍
            cache = cacheFromMap;
            // reset so that server not hang this check
            cache.setInitializing(true); // 缓存正在初始化
        } else {
            // 构造缓存数据对象
            cache = new CacheData(configFilterChainManager, agent.getName(), dataId, group, tenant);
            // 初始值taskId=0，注意此处每3000个CacheData共用一个taskId
            int taskId = cacheMap.get().size() / (int) ParamUtil.getPerTaskConfigSize();
            cache.setTaskId(taskId);
            // fix issue # 1317
            if (enableRemoteSyncConfig) { // 默认false
                String[] ct = getServerConfig(dataId, group, tenant, 3000L, false);
                cache.setContent(ct[0]);
            }
        }
        Map<String, CacheData> copy = new HashMap<String, CacheData>(this.cacheMap.get());
        // key = test+DEFAULT_GROUP
        copy.put(key, cache);
        // cacheMap = {test+DEFAULT_GROUP=CacheData [test, DEFAULT_GROUP]}
        cacheMap.set(copy);

    }
    LOGGER.info("[{}] [subscribe] {}", agent.getName(), key);

    MetricsMonitor.getListenConfigCountMonitor().set(cacheMap.get().size());

    return cache;
}
```

**具体缓存内容** 

| 属性                     | 含义                                             |
| ------------------------ | ------------------------------------------------ |
| name                     | ConfigTransportClient名称，config_rpc_client     |
| configFilterChainManager | filter拦截链条，可以执行一些列拦截器             |
| dataId                   | dataId                                           |
| group                    | group名称，默认为DEFAULT_GROUP                   |
| tenant                   | 租户名称                                         |
| listeners                | 添加的Listener列表，线程安全CopyOnWriteArrayList |
| content                  | 启动时会从本地文件读入，默认为null               |
| md5                      | content的md5字符串                               |

**小结**：添加监听器逻辑如下：构建CacheData，并缓存在cacheMap中，key是由「dataId+group+tenant」组成；每个CacheData会绑定了Listener列表，也绑定了taskId，3000个不同的CacheData对应一个taskId，对应一个gRPC通道实例；设置isSyncWithServer=false表示 cache md5 data不是来自server同步，BlockingQueue中添加new Object() 供前面提到的长轮询判断使用。



# 配置变更执行逻辑 

上文中提到一个线程一直在轮询，轮询执行executeConfigListen方法，这个方法比较关键。

```java
public void executeConfigListen() {

    Map<String/*taskId*/, List<CacheData>> listenCachesMap = new HashMap<String, List<CacheData>>(16);
    Map<String, List<CacheData>> removeListenCachesMap = new HashMap<String, List<CacheData>>(16);
    long now = System.currentTimeMillis();
    // 超过5分钟
    boolean needAllSync = now - lastAllSyncTime >= ALL_SYNC_INTERNAL;
    for (CacheData cache : cacheMap.get().values()) {
        synchronized (cache) {
            // --------注解@9开始--------
            if (cache.isSyncWithServer()) { 
                cache.checkListenerMd5(); // 内容有变更通知Listener执行
                if (!needAllSync) { // 不超过5分钟则不再全局校验
                    continue;
                }
            }
						// --------注解@9结束--------
            if (!CollectionUtils.isEmpty(cache.getListeners())) { // 有添加Listeners
                // get listen config 默认 false
                if (!cache.isUseLocalConfigInfo()) {
                    List<CacheData> cacheDatas = listenCachesMap.get(String.valueOf(cache.getTaskId()));
                    if (cacheDatas == null) {
                        cacheDatas = new LinkedList<CacheData>();
                        listenCachesMap.put(String.valueOf(cache.getTaskId()), cacheDatas);
                    }
                    // CacheData [test, DEFAULT_GROUP]
                    cacheDatas.add(cache);

                }
            } else if (CollectionUtils.isEmpty(cache.getListeners())) { // 没有添加Listeners

                if (!cache.isUseLocalConfigInfo()) {
                    List<CacheData> cacheDatas = removeListenCachesMap.get(String.valueOf(cache.getTaskId()));
                    if (cacheDatas == null) {
                        cacheDatas = new LinkedList<CacheData>();
                        removeListenCachesMap.put(String.valueOf(cache.getTaskId()), cacheDatas);
                    }
                    cacheDatas.add(cache);

                }
            }
        }

    }

    boolean hasChangedKeys = false;
  
		//-------------------注解@10开始---------------------------------
  
    if (!listenCachesMap.isEmpty()) { // 有Listeners
        for (Map.Entry<String, List<CacheData>> entry : listenCachesMap.entrySet()) {
            String taskId = entry.getKey();
            List<CacheData> listenCaches = entry.getValue();

            ConfigBatchListenRequest configChangeListenRequest = buildConfigRequest(listenCaches);
            configChangeListenRequest.setListen(true);
            try {
                // 注解@10.1 每个taskId构建rpcClient
                RpcClient rpcClient = ensureRpcClient(taskId);
                // 注解@10.2
                ConfigChangeBatchListenResponse configChangeBatchListenResponse = (ConfigChangeBatchListenResponse) requestProxy(
                        rpcClient, configChangeListenRequest);
                if (configChangeBatchListenResponse != null && configChangeBatchListenResponse.isSuccess()) {

                    Set<String> changeKeys = new HashSet<String>();
                    // handle changed keys,notify listener
                    // 有变化的configContext
                    if (!CollectionUtils.isEmpty(configChangeBatchListenResponse.getChangedConfigs())) {
                        hasChangedKeys = true;
                        for (ConfigChangeBatchListenResponse.ConfigContext changeConfig : configChangeBatchListenResponse.getChangedConfigs()) {
                            String changeKey = GroupKey
                                    .getKeyTenant(changeConfig.getDataId(), changeConfig.getGroup(),
                                            changeConfig.getTenant());
                            changeKeys.add(changeKey);
                            boolean isInitializing = cacheMap.get().get(changeKey).isInitializing();
                            // 注解@10.3 回调Listener
                            refreshContentAndCheck(changeKey, !isInitializing);
                        }

                    }

                    //handler content configs
                    for (CacheData cacheData : listenCaches) {
                        String groupKey = GroupKey
                                .getKeyTenant(cacheData.dataId, cacheData.group, cacheData.getTenant());
                        if (!changeKeys.contains(groupKey)) { // 注解@10.4
                            //sync:cache data md5 = server md5 && cache data md5 = all listeners md5.
                            synchronized (cacheData) {
                                if (!cacheData.getListeners().isEmpty()) {
                                    cacheData.setSyncWithServer(true);
                                    continue;
                                }
                            }
                        }

                        cacheData.setInitializing(false);
                    }

                }
            } catch (Exception e) {

                LOGGER.error("Async listen config change error ", e);
                try {
                    Thread.sleep(50L);
                } catch (InterruptedException interruptedException) {
                    //ignore
                }
            }
        }
    }
	//-------------------注解@10结束---------------------------------
    if (!removeListenCachesMap.isEmpty()) {
        for (Map.Entry<String, List<CacheData>> entry : removeListenCachesMap.entrySet()) {
            String taskId = entry.getKey();
            List<CacheData> removeListenCaches = entry.getValue();
            ConfigBatchListenRequest configChangeListenRequest = buildConfigRequest(removeListenCaches);
            configChangeListenRequest.setListen(false);
            try {
                // 向server发送Listener取消订阅请求ConfigBatchListenRequest#listen为false
                RpcClient rpcClient = ensureRpcClient(taskId);
                boolean removeSuccess = unListenConfigChange(rpcClient, configChangeListenRequest);
                if (removeSuccess) {
                    for (CacheData cacheData : removeListenCaches) {
                        synchronized (cacheData) {
                            if (cacheData.getListeners().isEmpty()) {
                                // 移除本地缓存
                                ClientWorker.this
                                        .removeCache(cacheData.dataId, cacheData.group, cacheData.tenant);
                            }
                        }
                    }
                }

            } catch (Exception e) {
                LOGGER.error("async remove listen config change error ", e);
            }
            try {
                Thread.sleep(50L);
            } catch (InterruptedException interruptedException) {
                //ignore
            }
        }
    }
    if (needAllSync) {
        lastAllSyncTime = now;
    }
    //If has changed keys,notify re sync md5.
    if (hasChangedKeys) { // key有变化触发下一轮轮询
        notifyListenConfig();
    }
}
```

**注解@9** isSyncWithServer初始为false，在下文代码中校验结束后会设置为true，表示md5 cache data同步来自server。如果为true会校验Md5.

```java
void checkListenerMd5() {
    for (ManagerListenerWrap wrap : listeners) {
        if (!md5.equals(wrap.lastCallMd5)) { // 注解@9.1
            safeNotifyListener(dataId, group, content, type, md5, wrap);
        }
    }
}
```

**注解@9.1**  配置内容有变更时，回调到我们示例中注册的Listener中。

```java
private void safeNotifyListener(final String dataId, final String group, final String content, final String type,
        final String md5, final ManagerListenerWrap listenerWrap) {
    final Listener listener = listenerWrap.listener;
    if (listenerWrap.inNotifying) {
        // ...
        return;
    }
    Runnable job = new Runnable() {
        @Override
        public void run() {
            long start = System.currentTimeMillis();
            ClassLoader myClassLoader = Thread.currentThread().getContextClassLoader();
            ClassLoader appClassLoader = listener.getClass().getClassLoader();
            try {
                if (listener instanceof AbstractSharedListener) {
                    AbstractSharedListener adapter = (AbstractSharedListener) listener;
                    adapter.fillContext(dataId, group);
                   // ...
                }
                Thread.currentThread().setContextClassLoader(appClassLoader);

                ConfigResponse cr = new ConfigResponse();
                cr.setDataId(dataId);
                cr.setGroup(group);
                cr.setContent(content);
                // filter拦截继续过滤
                configFilterChainManager.doFilter(null, cr);

                String contentTmp = cr.getContent();
                listenerWrap.inNotifying = true;

                // 注解@9.2
                listener.receiveConfigInfo(contentTmp);
                // compare lastContent and content
                if (listener instanceof AbstractConfigChangeListener) {
                    Map data = ConfigChangeHandler.getInstance()
                            .parseChangeData(listenerWrap.lastContent, content, type);
                    ConfigChangeEvent event = new ConfigChangeEvent(data);
                    // 回调变更事件方法
                    ((AbstractConfigChangeListener) listener).receiveConfigChange(event);
                    listenerWrap.lastContent = content;
                }

                listenerWrap.lastCallMd5 = md5;
                // ..
            } catch (NacosException ex) {
               // ...
            } catch (Throwable t) {
               // ...
            } finally {
                listenerWrap.inNotifying = false;
                Thread.currentThread().setContextClassLoader(myClassLoader);
            }
        }
    };

    final long startNotify = System.currentTimeMillis();
    try {
      	// 注解@9.3
        if (null != listener.getExecutor()) {
            listener.getExecutor().execute(job);
        } else {
            try {
                INTERNAL_NOTIFIER.submit(job); // 默认线程池执行，为5个线程
            } catch (RejectedExecutionException rejectedExecutionException) {
                // ...
                job.run();
            } catch (Throwable throwable) {
                // ...
                job.run();
            }
        }
    } catch (Throwable t) {
       // ...
    }
    final long finishNotify = System.currentTimeMillis();
    // ...
}
```

**注解@9.2** 回调注册Listener的receiveConfigInfo方法或者receiveConfigChange逻辑

**注解@9.3** 优先使用我们示例中注册提供的线程池执行job，如果没有设置使用默认线程池「INTERNAL_NOTIFIER」，默认5个线程



**备注：** 当CacheData从server同步后，会校验md5是否变更了，当变更时会回调到我们注册的Listener完成通知。通知任务被封装成Runnable任务，执行线程池可以自定义，默认为5个线程。



**注解@10.1** 每个taskId构建rpcClient，例如：taskId= config-0-c70e0314-4770-43f5-add4-f258a4083fd7；结合上下文每3000个CacheData对应一个rpcClient。

**注解@10.2** 向server发起configChangeListenRequest，server端由ConfigChangeBatchListenRequestHandler处理，还是比较md5

是否变更了，变更后server端返回变更的key列表。

**注解@10.3** 当server返回变更key列表时执行refreshContentAndCheck方法。

```java
private void refreshContentAndCheck(CacheData cacheData, boolean notify) {
    try {
        // 注解@10.3.1
        String[] ct = getServerConfig(cacheData.dataId, cacheData.group, cacheData.tenant, 3000L, notify);
        cacheData.setContent(ct[0]);
        if (null != ct[1]) {
            cacheData.setType(ct[1]);
        }
        if (notify) { // 记录日志
           // ...
        }
        // 注解@10.3.2
        cacheData.checkListenerMd5();
    } catch (Exception e) {
        //...
    }
}
```

**注解@10.3.1** 向server发起ConfigQueryRequest，查询配置内容

**注解@10.3.2** 回调注册的Listener逻辑见 **注解@9** 

 **注解@10.4**  key没有变化的，内容由server同步，设置SyncWithServer=true，下一轮逻辑会由 **注解@9** 部分执行



**备注：** 从整个**注解@10** 注册Listener后，会构建与server的RPC通道rpcClient；向server发起变更查询请求configChangeListenRequest，server端通过比较缓存的md5值，返回client变更的key列表；client通过变更的key列表向server发起配置查询请求ConfigQueryRequest，获取变更内容，并回调我们注册的Listener。







