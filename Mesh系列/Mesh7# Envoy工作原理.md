

```
title: Mesh6# gRPC服务通过Istio网格通信
categories: Mesh
tags: Mesh
date: 2021-11-07 11:55:01
```





# 基本概念



Istio的核心组件，作为sideCar与应用部署在一个Pod中，作为代理流量的进出均需经过Envoy所在的容器，除了代理外还可根据规则进行流量治理、监控等功能。



**Upstream Host:** 上游主机，接受envoy的连接和请求并返回响应

**Downstream Host:** 下游主机，向envoy发起请求并接受响应

**Enovy Mesh:** 由一组Envoy组成的拓扑网络

**Listener: **  监听器负责监听数据端口，接受下游的连接和请求，下游主机通过Listener连接Envoy

**Cluster: ** 集群管理后端服务服务的连接池、服务的健康检查、服务熔断等

**Filter：** 支持多种过滤器Listener Filter、Network Filter、L7 Filter等，组成filter链条，执行不同的流量治理逻辑。



**协议支持：** 

L3/L4网络代理，支持TCP、HTTP代理和TLS认证

L7代理，支持Buffer、限流等高级功能

L7路由，支持通过路径、权限、请求内容、运行时间等参数重定向路由请求

在HTTP模式下支持HTTP1.1和HTTP/2，同时支持基于HTTP/2的gRPC





**线程模型** 

一个Envoy进程包括一个Server主线程和一个GuardDog守护线程

Server主线程：负责管理Access Log以及解析上游主机的DNS。Access Log根据配置信息访问来处理Enovy访问记录，DNS解析将统一配置的域名解析成IP并缓存在本地DNS缓存中。



GuardDog守护线程：负责看门狗业务



一个Envoy进程可以配置多个Listener，推荐配置一个，每个Listener可创建若干线程默认为核数，每个线程对应一个Worker。

一旦某个客户端连接进入Envoy中的某个线程，则连接断开之前的逻辑都在该线程内处理。例如：处理Client请求对应的TCP filter，解析协议和重新编码，与上游主机建立连接并处理返回数据等

























