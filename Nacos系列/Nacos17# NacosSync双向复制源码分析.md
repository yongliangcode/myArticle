---
title: Nacos17# NacosSync双向复制源码分析
categories: Nacos
tags: Nacos
date: 2021-09-02 11:55:01
---



# 引言

通过开源同步工具NacosSync的分析，对我们实现自定义的同步工具提供参考。文本就同步任务分发与Nacos集群之间、从zk到Nacos的同步源码做个分析。



# 内容提要



### 任务和配置入库

* 集群配置入库
* 同步任务入库



### 同步任务分发

* 每三秒调度一次任务列表
* 新增任务发布同步任务事件SyncTaskEvent并由listenerSyncTaskEvent处理
* 删除任务发布删除任务事件DeleteTaskEvent并由listenerDeleteTaskEvent处理
* 任务的发布和订阅使用Guava的EventBus



### Nacos集群之间同步逻辑

* 两个Nacos集群之间进行同步，同步任务在Service维度（AppId）建立
* 对源集群注册监听获取注册节点列表，通过剔除无效节点后，将新的节点注册到目标集群



### 从zk集群同步到Nacos集群

* NacosSync从zk集群同步到Nacos只支持dubbo路径
* 第一次先同步所有节点过去，再监听源集群路径变化，同步到目标集群



# 任务和配置入库

入库部分比较简单，只列出入口和处理类。



**集群配置入库** 

请求入口：ClusterApi#clusterAdd

入库处理：ClusterAddProcessor#process

```java
clusterAccessService.insert(clusterDO);
```



**同步任务入库** 

请求入口：TaskApi#taskAdd

入库处理：TaskAccessService#addTask

```java
 taskAccessService.addTask(taskDO);
```



# 同步任务分发

同步任务入库了，紧着需要任务进行分发。代码翻到QuerySyncTaskTimer实现了springboot的CommandLineRunner接口。



**定时任务调度** 

```java
public void run(String... args) {
	scheduledExecutorService.scheduleWithFixedDelay(new CheckRunningStatusThread(), 0, 3000,
	TimeUnit.MILLISECONDS);
}
```

**备注：** 定时任务每3秒钟调度一次。



**调度任务执行** 

```java
private class CheckRunningStatusThread implements Runnable {

    @Override
    public void run() {

        Long start = System.currentTimeMillis();
        try {
          	// 注解@1
            Iterable<TaskDO> taskDOS = taskAccessService.findAll();
            taskDOS.forEach(taskDO -> {
              	// 注解@2 
                if ((null != skyWalkerCacheServices.getFinishedTask(taskDO))) {
                    return;
                }
              	// 注解@3
                if (TaskStatusEnum.SYNC.getCode().equals(taskDO.getTaskStatus())) {
                    eventBus.post(new SyncTaskEvent(taskDO));
                    log.info("从数据库中查询到一个同步任务，发出一个同步事件:" + taskDO);
                }
								// 注解@4
                if (TaskStatusEnum.DELETE.getCode().equals(taskDO.getTaskStatus())) {
                    eventBus.post(new DeleteTaskEvent(taskDO));
                    log.info("从数据库中查询到一个删除任务，发出一个同步事件:" + taskDO);
                }
            });

        } catch (Exception e) {
            log.warn("CheckRunningStatusThread Exception", e);
        }
				// 注解@5
        metricsManager.record(MetricsStatisticsType.DISPATCHER_TASK, System.currentTimeMillis() - start);
    }
}
```

**注解@1** 查询所有同步任务

**注解@2** 过滤已完成的任务

**注解@3** 发布一个同步任务事件SyncTaskEvent

**注解@4** 发布一个删除任务事件DeleteTaskEvent

**注解@5** 通过metric统计本次调度任务执行的耗时情况



**小结：** 当有新增任务或者删除任务时通过Guava的EventBus发布一个同步事件或删除事件，该检测3秒执行一次。



# 同步事件处理

代码EventListener#listenerSyncTaskEvent订阅了同步事件SyncTaskEvent。

```java
@Subscribe
public void listenerSyncTaskEvent(SyncTaskEvent syncTaskEvent) {

    try {
        long start = System.currentTimeMillis();
      	// 注解@6
        if (syncManagerService.sync(syncTaskEvent.getTaskDO())) {    
          	// 注解@7
            skyWalkerCacheServices.addFinishedTask(syncTaskEvent.getTaskDO());
          	// 注解@8
            metricsManager.record(MetricsStatisticsType.SYNC_TASK_RT, System.currentTimeMillis() - start);
        } else {
            log.warn("listenerSyncTaskEvent sync failure");
        }                
    } catch (Exception e) {
        log.warn("listenerSyncTaskEvent process error", e);
    }

}
```

**注解@6** 执行同步任务

**注解@7** 标记该同步任务完成

**注解@8** 记录任务执行时间



代码EventListener#listenerDeleteTaskEvent订阅了删除任务事件DeleteTaskEvent。

```java
@Subscribe
public void listenerDeleteTaskEvent(DeleteTaskEvent deleteTaskEvent) {

    try {
        long start = System.currentTimeMillis();
        if (syncManagerService.delete(deleteTaskEvent.getTaskDO())) {
            skyWalkerCacheServices.addFinishedTask(deleteTaskEvent.getTaskDO());
            metricsManager.record(MetricsStatisticsType.DELETE_TASK_RT, System.currentTimeMillis() - start);
        } else {
            log.warn("listenerDeleteTaskEvent delete failure");
        }                
    } catch (Exception e) {
        log.warn("listenerDeleteTaskEvent process error", e);
    }

}
```



**小结：**  listenerSyncTaskEvent和listenerDeleteTaskEvent代码结构一致，执行任务逻辑，执行完缓存已完成任务，最后记录耗时情况。



# Nacos集群之间同步逻辑

先看下Nacos集群之间的同步，代码在NacosSyncToNacosServiceImpl#sync部分。

**执行同步逻辑** 

```java
@Override
public boolean sync(TaskDO taskDO) {
  String taskId = taskDO.getTaskId();
  try {
    // 注解@7
    NamingService sourceNamingService =
      nacosServerHolder.get(taskDO.getSourceClusterId(), taskDO.getNameSpace());

    // 注解@8
    NamingService destNamingService = nacosServerHolder.get(taskDO.getDestClusterId(), taskDO.getNameSpace());


    this.listenerMap.putIfAbsent(taskId, event -> {
      if (event instanceof NamingEvent) {
        try {
          // 注解@9
          List<Instance> sourceInstances = sourceNamingService.getAllInstances(taskDO.getServiceName(),
                                                                               getGroupNameOrDefault(taskDO.getGroupName()), new ArrayList<>(), true);

          // 注解@10
          this.removeInvalidInstance(taskDO, destNamingService, sourceInstances);

          // 注解@11
          if (sourceInstances.isEmpty()) {
            sourceInstanceSnapshot.remove(taskId);
            return;
          }

          // 注解@12
          this.syncNewInstance(taskDO, destNamingService, sourceInstances);
        } catch (Exception e) {
          log.error("event process fail, taskId:{}", taskId, e);
          metricsManager.recordError(MetricsStatisticsType.SYNC_ERROR);
        }
      }
    });

    sourceNamingService.subscribe(taskDO.getServiceName(), getGroupNameOrDefault(taskDO.getGroupName()),
                                  listenerMap.get(taskId));
  } catch (Exception e) {
    log.error("sync task from nacos to nacos was failed, taskId:{}", taskId, e);
    metricsManager.recordError(MetricsStatisticsType.SYNC_ERROR);
    return false;
  }
  return true;
}
```

**注解@7** 创建源集群的NameService

**注解@8** 创建目标集群的NameService

**注解@9** 获取服务注册的实例

**注解@10** 先删除已失效的节点

```java
private void removeInvalidInstance(TaskDO taskDO, NamingService destNamingService,
    List<Instance> sourceInstances) throws NacosException {

    String taskId = taskDO.getTaskId();
    if (this.sourceInstanceSnapshot.containsKey(taskId)) {
        // 注解@10.1
        Set<String> oldInstanceKeys = this.sourceInstanceSnapshot.get(taskId);
        List<String> newInstanceKeys = sourceInstances.stream().map(this::composeInstanceKey)
            .collect(Collectors.toList());
        // 注解@10.2
        Collection<String> instanceKeys = Collections.subtract(oldInstanceKeys, newInstanceKeys);
        for (String instanceKey : instanceKeys) {
            log.info("任务Id:{},移除无效同步实例:{}", taskId, instanceKey);
            String[] split = instanceKey.split(":", -1);
            // 注解@10.3
            destNamingService
                .deregisterInstance(taskDO.getServiceName(), getGroupNameOrDefault(taskDO.getGroupName()), split[0],
                    Integer.parseInt(split[1]));

        }
    }
}
```

**注解@10.1**   缓存的旧节点信息

**注解@10.2**  从旧节点中剥离出废弃无效的节点

**注解@10.3** 将废弃无效节点注销

**注解@11** 如果同步实例已经为空代表该服务所有实例已经下线,清除本地持有快照

**注解@12**  同步新节实例到目标集群并更新缓存

```java
private void syncNewInstance(TaskDO taskDO, NamingService destNamingService,
    List<Instance> sourceInstances) throws NacosException {
    Set<String> latestSyncInstance = new TreeSet<>();
    // 再次添加新实例
    String taskId = taskDO.getTaskId();
    // 注解@12.1
    Set<String> instanceKeys = sourceInstanceSnapshot.get(taskId);
    // 注解@12.2
    for (Instance instance : sourceInstances) {
        if (needSync(instance.getMetadata())) {
            String instanceKey = composeInstanceKey(instance);
            // 注解@12.3
            if (CollectionUtils.isEmpty(instanceKeys) || !instanceKeys.contains(instanceKey)) {
                destNamingService.registerInstance(taskDO.getServiceName(),
                    getGroupNameOrDefault(taskDO.getGroupName()),
                    buildSyncInstance(instance, taskDO));
            }
            // 注解@12.4
            latestSyncInstance.add(instanceKey);
        }
    }
    if (CollectionUtils.isNotEmpty(latestSyncInstance)) {

        log.info("任务Id:{},已同步实例个数:{}", taskId, latestSyncInstance.size());
        // 注解@12.5
        sourceInstanceSnapshot.put(taskId, latestSyncInstance);
    }
}
```

**注解@12.1** 缓存的旧节点信息

**注解@12.2** 遍历新节点信息

**注解@12.3** 当新节点信息不为空并且旧节点不存在，则注册到目标集群

**注解@12.4** 收集新节点

**注解@12.5** 更新缓存节点信息



**小结：** 在两个Nacos集群之间进行同步，同步任务在Service维度（AppId）建立。通过对源集群注册监听获取注册节点列表，通过剔除无效节点后，将新的节点注册到目标集群的过程。



**执行删除任务逻辑** 

代码翻到NacosSyncToNacosServiceImpl#delete部分

```java
public boolean delete(TaskDO taskDO) {
    try {
        NamingService sourceNamingService =
            nacosServerHolder.get(taskDO.getSourceClusterId(), taskDO.getNameSpace());
        NamingService destNamingService = nacosServerHolder.get(taskDO.getDestClusterId(), taskDO.getNameSpace());
        // 注解@13
        sourceNamingService
            .unsubscribe(taskDO.getServiceName(), getGroupNameOrDefault(taskDO.getGroupName()),
                listenerMap.remove(taskDO.getTaskId()));
        sourceInstanceSnapshot.remove(taskDO.getTaskId());

        // 注解@14
        List<Instance> sourceInstances = sourceNamingService
            .getAllInstances(taskDO.getServiceName(), getGroupNameOrDefault(taskDO.getGroupName()),
                new ArrayList<>(), false);
        for (Instance instance : sourceInstances) {
            if (needSync(instance.getMetadata())) {
              	// 注销操作
                destNamingService
                    .deregisterInstance(taskDO.getServiceName(), getGroupNameOrDefault(taskDO.getGroupName()),
                        instance.getIp(),
                        instance.getPort());
            }
        }
    } catch (Exception e) {
        log.error("delete task from nacos to nacos was failed, taskId:{}", taskDO.getTaskId(), e);
        metricsManager.recordError(MetricsStatisticsType.DELETE_ERROR);
        return false;
    }
    return true;
}
```

**注解@13** 移除该任务（service）源集群订阅

**注解@14** 删除目标集群中同步的实例列表



**小结：** 删除逻辑比较简单，取消源集群订阅，将目标集群的注册节点移除。



# 从zk集群同步到Nacos集群

再看从zk集群同步到Nacos集群，代码翻到ZookeeperSyncToNacosServiceImpl#sync()

```java
@Override
public boolean sync(TaskDO taskDO) {
    try {
        if (treeCacheMap.containsKey(taskDO.getTaskId())) {
            return true;
        }
        // 注解@1
        TreeCache treeCache = getTreeCache(taskDO);
        // 注解@2
        NamingService destNamingService = nacosServerHolder.get(taskDO.getDestClusterId(), null);
        // 注解@3
        registerAllInstances(taskDO, destNamingService);
        // 注解@4
        Objects.requireNonNull(treeCache).getListenable().addListener((client, event) -> {
            try {
                String path = event.getData().getPath();
                Map<String, String> queryParam = parseQueryString(path);
                if (isMatch(taskDO, queryParam) && needSync(queryParam)) {
                    processEvent(taskDO, destNamingService, event, path, queryParam);
                }
            } catch (Exception e) {
                // ...
            }
        });
    } catch (Exception e) {
       	// ...
        metricsManager.recordError(MetricsStatisticsType.SYNC_ERROR);
        return false;
    }
    return true;
}
```

**注解@1** 监听zk源集群 路径为「/dubbo」

**注解@2** 目标Nacos集群构建

**注解@3** 初次执行任务统一注册所有实例 

```java
private void registerAllInstances(TaskDO taskDO, NamingService destNamingService) throws Exception {
    CuratorFramework zk =  zookeeperServerHolder.get(taskDO.getSourceClusterId(), "");
    // 注解@3.1
    if(!ALL_SERVICE_NAME_PATTERN.equals(taskDO.getServiceName())) {
        registerALLInstances0(taskDO, destNamingService, zk, taskDO.getServiceName());
    } else {
        // 注解@3.2
        List<String> serviceList = zk.getChildren().forPath(DUBBO_ROOT_PATH);
        for(String serviceName : serviceList) {
            registerALLInstances0(taskDO, destNamingService, zk, serviceName);
        }
    }
}
```

**注解@3.1** 同步特定服务注册节点（Dubbo）

**注解@3.2** 同步全部所有的zk节点到Nacos

**注解@4** 注册zk监听监听新增和更新的同步

```java
private void processEvent(TaskDO taskDO, NamingService destNamingService, TreeCacheEvent event, String path,
                          Map<String, String> queryParam) throws NacosException {
    if(!com.alibaba.nacossync.util.StringUtils.isDubboProviderPath(path)) {
        return;
    }

    Map<String, String> ipAndPortParam = parseIpAndPortString(path);
    Instance instance = buildSyncInstance(queryParam, ipAndPortParam, taskDO);
    String serviceName = queryParam.get(INTERFACE_KEY);
    switch (event.getType()) {
        case NODE_ADDED:
        case NODE_UPDATED:
            // 注解@4.1
            destNamingService.registerInstance(
                getServiceNameFromCache(serviceName, queryParam), instance);
            break;
        case NODE_REMOVED:
            // 注解@4.2
            destNamingService.deregisterInstance(
                getServiceNameFromCache(serviceName, queryParam),
                ipAndPortParam.get(INSTANCE_IP_KEY),
                Integer.parseInt(ipAndPortParam.get(INSTANCE_PORT_KEY)));
            nacosServiceNameMap.remove(serviceName);
            break;
        default:
            break;
    }
}
```

**注解@4.1** 同步节点新增更新到目标集群

**注解@4.2** 源集群节点被删除同步注销目标集群



**小结：** NacosSync从zk集群同步到Nacos只支持dubbo路径，可参考基于二次改造。第一次先同步所有节点过去，再监听源集群路径变化，同步到目标集群。









































