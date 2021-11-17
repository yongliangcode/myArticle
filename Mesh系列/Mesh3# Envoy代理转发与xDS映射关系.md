

```
title: Mesh3# Envoy代理转发与xDS映射关系
categories: Mesh
tags: Mesh
date: 2021-10-15 11:55:01
```



# 引言



Envoy作为Istio默认数据面代理，它的工作流程是怎么样的？本文通过示例运行，走查其运行流程，以及xDS协议映射。



# xDS



**xDS**  协议是“X Discovery Service”的简写，这里的“X”表示它不是指具体的某个协议，是一组基于不同数据源的服务发现协议的总称，包括 CDS、LDS、EDS、RDS等。在Istio架构中，基于xDS协议提供了标准的控制面规范，并以此向数据面传递服务信息和治理规则。在Envoy中，xDS被称为数据平面 API，并且担任控制平面Pilot和数据平面Envoy的通信协议。



**CDS**  是 Cluster Discovery Service的缩写，Envoy使用它在进行路由的时候发现上游Cluster。Envoy通常会优雅地添加、更新和删除 Cluster。有了 CDS 协议，Envoy在初次启动的时候不一定要感知拓扑里所有的上游Cluster。在做路由 HTTP 请求的时候通过在 HTTP 请求头里添加 Cluster信息实现请求转发。



**EDS** 即Endpoint Discovery Service 的缩写。在Envoy术语中，Endpoint即Cluster的成员。Envoy 通过 EDS API可以更加智能地动态获取上游Endpoint。



**LDS**  即Listener Discovery Service的缩写。基于此，Envoy 可以在运行时发现所有的Listener，包括 L3 和 L4 filter 等所有的 filter 栈，并由此执行各种代理工作，如认证、TCP 代理和 HTTP 代理等。添加 LDS 使得 Envoy 的任何配置都可以动态执行。



**RDS**  即 Router Discovery Service 的缩写，用于 Envoy 在运行时为 HTTP 连接管理 filter 获取完整的路由配置，比如 HTTP 头部修改等。并且路由配置会被优雅地写入而无需影响已有的请求。当 RDS 和 EDS、CDS 共同使用时，可以帮助构建一个复杂的路由拓扑蓝绿发布等。



**ADS**  EDS，CDS 等每个独立的服务都对应了不同的 gRPC 服务名称。对于需要控制不同类型资源抵达 Envoy 顺序的需求，可以使用聚合发现服务，即 Aggregated xDS，它可以通过单一的 gRPC 服务流支持所有的资源类型，借助于有序的配置分发，从而解决资源更新顺序的问题。



```
备注：上述概念摘自 https://www.servicemesher.com/istio-handbook/ecosystem/xds.html
```



# Envoy代理示例



**安装部署** 

以mac版本为例，安装看看

```
brew update
brew install envoy
```



**版本检查** 

```
envoy --version

envoy  version: a2a1e3eed4214a38608ec223859fcfa8fb679b14/1.19.1/Modified/RELEASE/BoringSSL

```



下载示例yaml文件

```
https://www.envoyproxy.io/docs/envoy/latest/_downloads/92dcb9714fb6bc288d042029b34c0de4/envoy-demo.yaml
```



**示例运行** 

```
envoy -c envoy-demo.yaml
```



访问以下地址会路由转发到Envoy官方地址「www.envoyproxy.io」

```
http://localhost:10000/
```

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210924223926.png)





# 逻辑走查

envoy-demo.yaml文件走查

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210924192135.png)



概念：

LDS（Listener Discovery Service）：监听发现服务

RDS（Route Discovery Service）：路由发现服务

CDS（Cluster Discovery Service)：集群发现服务

EDS（Endpoint Discovery Service）：集群成员发现服务



流程：

1.Listener通过监听端口（10000）将请求根据Route提供的策略转发

2.Route可以配置路由规则，示例中转发到名字为「service_envoyproxy_io」的cluster

3.Cluster中可以配置行为相同的多个EndPoint，多个EndPoint可以配置负载均衡策略

4.EndPoint最终转发的节点地址





