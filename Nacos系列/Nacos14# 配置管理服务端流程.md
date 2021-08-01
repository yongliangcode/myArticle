---
title: Nacos14# 配置管理服务端流程
categories: Nacos
tags: Nacos
date: 2021-08-21 11:55:01
---



# 引言

在上文分析中客户端会有长轮询，不断轮询阻塞队列「listenExecutebell」去比较客户端和服务端配置内容md5是否一致，不一致则通知我们注册的Listener完成回调。当阻塞队里有元素时会立即执行，没有元素等待5秒钟执行。那都在什么时候往队列中添加元素的从而触发立即执行。



# 内容提要



### 长轮询回顾

* 监听配置内容的变更通过长轮询实现
* 具体实现为轮询阻塞队列



### 阻塞队列添加时机

* 客户端添加Listener时添加
* 客户端删除Listener时添加
* 服务端通知内容变更时添加
* 建立gRPC连接时添加



### 服务端变更发布流程

* 通过配置中心发起变更请求
* 配置变更内容被写入数据库
* 向本节点连接的Client发送变更通知
* 向集群中的其他节点发送变更通知



### 向Client发送变更通知

* 首先构建DumpTask
* 处理DumpTask时发布ConfigDumpEvent事件
* 处理ConfigDumpEvent时发布LocalDataChangeEvent事件
* 处理LocalDataChangeEvent事件：@1 先从缓存中拿出注册Client列表 @2 根据client从缓存获取gRPC连接 @3 构建ConfigChangeNotifyRequest @4 通过gRPC向客户端发送变更通知
* 客户端处理变更通知，客户端接到请求后向阻塞队列添加new Object元素，详细客户端的长轮询流程见上篇



### 向其他节点发送变更通知

* 通过gRPC向集群中其他节点发送变更通知
* 集群节点收到变更通知后再下发给连接自己的Client





# 长轮询回顾

**长轮询判断**

```java
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

**队列添加元素** 

```java
@Override
public void notifyListenConfig() {
    listenExecutebell.offer(bellItem);
}
```

**备注：** 配置内容变更机制是通过长轮询实现的。



# **阻塞队列添加时机** 

**1.客户端添加Listener时添加** 

```java
public void addListeners(String dataId, String group, List<? extends Listener> listeners) {
    // ...
  	// 通过队列中增加一个对象，长轮询立即执行
  	agent.notifyListenConfig();
  }
}
```

**2.客户端删除Listener时添加** 

```java
public void removeListener(String dataId, String group, Listener listener) {
    group = null2defaultGroup(group);
    CacheData cache = getCache(dataId, group);
    if (null != cache) {
        synchronized (cache) {
            cache.removeListener(listener);
            if (cache.getListeners().isEmpty()) {
                cache.setSyncWithServer(false);
                agent.removeCache(dataId, group);
            }
        }

    }
}

public void removeCache(String dataId, String group) {
  // Notify to rpc un listen ,and remove cache if success.
  notifyListenConfig();
}
```

**3.服务端通知内容变更时添加** 

在与服务端建立gRPC通道时会添加Handler用于处理服务端推送的请求，收到推送请求后向阻塞队列添加元素。

```java
private void initRpcClientHandler(final RpcClient rpcClientInner) {
    /*
     * Register Config Change /Config ReSync Handler
     */
    rpcClientInner.registerServerRequestHandler((request) -> {
        if (request instanceof ConfigChangeNotifyRequest) {
            ConfigChangeNotifyRequest configChangeNotifyRequest = (ConfigChangeNotifyRequest) request;
            LOGGER.info("[{}] [server-push] config changed. dataId={}, group={},tenant={}",
                    rpcClientInner.getName(), configChangeNotifyRequest.getDataId(),
                    configChangeNotifyRequest.getGroup(), configChangeNotifyRequest.getTenant());
            String groupKey = GroupKey
                    .getKeyTenant(configChangeNotifyRequest.getDataId(), configChangeNotifyRequest.getGroup(),
                            configChangeNotifyRequest.getTenant());

            CacheData cacheData = cacheMap.get().get(groupKey);
            if (cacheData != null) {
                cacheData.setSyncWithServer(false);
                notifyListenConfig(); // 向阻塞队列中添加元素
            }
            return new ConfigChangeNotifyResponse();
        }
        return null;
    });
  	// ....
 }
```

**4.建立gRPC连接时添加**

在建立gRPC连接时会向阻塞队列中添加元素。

```java
rpcClientInner.registerConnectionListener(new ConnectionEventListener() {

    @Override
    public void onConnected() {
        LOGGER.info("[{}] Connected,notify listen context...", rpcClientInner.getName());
        notifyListenConfig(); // 向队列中添加元素
    }

    @Override
    public void onDisConnect() {
        String taskId = rpcClientInner.getLabels().get("taskId");
        LOGGER.info("[{}] DisConnected,clear listen context...", rpcClientInner.getName());
        Collection<CacheData> values = cacheMap.get().values();

        for (CacheData cacheData : values) {
            if (StringUtils.isNotBlank(taskId)) {
                if (Integer.valueOf(taskId).equals(cacheData.getTaskId())) {
                    cacheData.setSyncWithServer(false);
                }
            } else {
                cacheData.setSyncWithServer(false);
            }
        }
    }
```

**备注：** 阻塞队列添加元素的四种情况，添加后长轮询立即执行配置内容校验。



# **服务端变更发布流程** 

**1.发布配置变更请求** 

服务端的发布内容变更，通过变更请求跟踪下，后端处理请求通过ConfigController#publishConfig()实现。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210730172935.png)



**2.处理配置变更请求** 

* 将变更配置更新到数据库
* 发布ConfigDataChangeEvent事件

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210730174331.png)



**3.订阅ConfigDataChangeEvent** 

翻到订阅该事件的处理逻辑，看看其干了什么事情，AsyncNotifyService构造函数中的onEvent()方法。从代码中可以看出当收到ConfigDataChangeEvent时，逻辑如下：

* 获取集群中所有节点列表
* 每个节点封装NotifySingleRpcTask
* 异步向各个节点通知配置信息变更

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210730193829.png)



**4.NotifySingleRpcTask逻辑** 

在代码AsyncNotifyService#AsyncRpcTask部分查看逻辑框架，任务运行主要干了两件事：

* 向连接本节点的*Client*发送配置变更通知
* 向集群中的其他节点发送变更通知

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210731111536.png)



**小结：** 当我们从Nacos配置中心界面提交变更配置时：1.入库； 2.向本节点连接的Client发送变更通知；3.向集群中的其他节点发送变更通知。



# 向Client发送变更通知

**1.首先构建DumpTask** 

代码坐标DumpService#dump()

```java
public void dump(String dataId, String group, String tenant, String tag, long lastModified, String handleIp,
        boolean isBeta) {
    String groupKey = GroupKey2.getKey(dataId, group, tenant);
    dumpTaskMgr.addTask(groupKey, new DumpTask(groupKey, tag, lastModified, handleIp, isBeta));
}
```

**2.处理DumpTask时发布ConfigDumpEvent事件** 

代码坐标DumpProcessor#process()

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210731193633.png)

**3.处理ConfigDumpEvent时发布LocalDataChangeEvent事件** 

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210731194211.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210731194350.png)

**4.处理LocalDataChangeEvent事件** 

关注RpcConfigChangeNotifier#onEvent处理过程

```java
@Override
public void onEvent(LocalDataChangeEvent event) {
    String groupKey = event.groupKey;
    boolean isBeta = event.isBeta;
    List<String> betaIps = event.betaIps;
    String[] strings = GroupKey.parseKey(groupKey);
    String dataId = strings[0];
    String group = strings[1];
    String tenant = strings.length > 2 ? strings[2] : "";
    String tag = event.tag;
    configDataChanged(groupKey, dataId, group, tenant, isBeta, betaIps, tag);
}
```

例如：注册在缓存的client=1627734181270_127.0.0.1_53522

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210801091854.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210801092420.png)

**5.客户端处理变更通知** 

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210801094245.png)



**小结：** 走查了服务端如何将变更通知送达到连接该节点的客户端的。



# 向其他节点发送变更通知

**1.接上面「服务端变更发布流程」最后步骤「向集群中的其他节点发送变更通知」** 

```java
configClusterRpcClientProxy
        .syncConfigChange(member, syncRequest, new AsyncRpcNotifyCallBack(task));
```

从缓存中获取与其他节点的rpcClient连接，key=Cluster-1.2.3.4:1111

```java
public void asyncRequest(Member member, Request request, RequestCallBack callBack) throws NacosException {
    RpcClient client = RpcClientFactory.getClient(memberClientKey(member));
    if (client != null) {
        client.asyncRequest(request, callBack);
    } else {
        throw new NacosException(CLIENT_INVALID_PARAM, "No rpc client related to member: " + member);
    }
}
```

**2.其他节点处理请求入口** 

GrpcRequestAcceptor#request只列出部分代码

```java
// 执行RequestHandler逻辑
Response response = requestHandler.handleRequest(request, requestMeta);
```

**3.其他节点由ConfigChangeClusterSyncRequestHandler处理ConfigChangeClusterSyncRequest** 

即：完成向客户端发送变更通知

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210801102703.png)



**小结：** 当集群其他节点收到变更通知后，会向连接到本节点的Client发送变更通知，逻辑同上一小结「向Client发送变更通知」，从而完成全部向所有Client下发变更通知。



