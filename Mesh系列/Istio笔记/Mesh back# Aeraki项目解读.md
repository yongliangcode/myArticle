

```
title: Mesh3# Aeraki项目解读
categories: Mesh
tags: Mesh
date: 2021-10-08 11:55:01
```



# 引言





# 源码走查



Controller初始化



初始化了envoyFilterController、configController、serviceEntryController、routeController、xdsServer、crdController



EnvoyFilter： 描述了针对代理服务的过滤器，用来定制由 Istio Pilot 生成的代理配置



# NewServer初始化

1.读取kubenertes配置 $HOME/.kube/config

2.初始化Clientset封装了操作api-server的资源操作API

3.ConfigController用于监听 istio xds server 地址并将变更通知listeners

4.构造了ServiceEntry Controller 封装了serviceEntry的add/update/delete事件监听

   4.1 其中使用了client-go的informer机制可以将变化带给controller

   4.2 将ServiceEntry变更封装在NewRateLimitingQueue队列中

5.构造EnvoyFilterController

   5.1 监听ConfigController变更，并将事件写入pushChannel

   5.2 监听crdController（自定义资源例如dubbo资源等）变更，并将事件写入pushChannel

6.构建xdsServer



备注：ServiceEntry接受的变更会写入队列NewRateLimitingQueue。



# Envoy Filter Controller 运行逻辑



整个流程由pushEnvoyFilters2APIServer负责



### EnvoyFilter生成流程

1.通过client-go从kube api server获取serviceEntries列表

2.找出该serviceEntry关联的路由信息VirtualService

3.将ServiceEntry和VirualService封装成EnvoyFilterContext

4.根据协议类型(例如：ServiceEntry.ports.name=dubbo）获取不同的协议envoyFilter生成器，并生成相应的EnvoyFilter

5.将协议类型和对应的EnvoyFilter存入缓存map



### EnvoyFilter变更操作

1.根据新配置和老配置执行新增、删除和更新操作，更新到kube api server

2.均调用client-go提供的EnvoyFilterInterface实现



备注：pushEnvoyFilters2APIServer的运行由NewRateLimitingQueue的事件触发，即当监听到ServiceEntry的变化时触发EnvoyFilter的变更推送操作。















