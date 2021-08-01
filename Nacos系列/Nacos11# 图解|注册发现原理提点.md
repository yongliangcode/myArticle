---
title: Nacos11# 图解|注册发现核心原理提点
categories: Nacos
tags: Nacos
date: 2021-07-28 11:55:01
---



# 引言

通过对Nacos注册与发现源码阅读，将其核心原理归纳提炼。包含：注册、发现、节点之间通信、健康检查类型。



# 服务注册原理

当客户端发起注册时，注册原理逻辑见下图，进一步简化主要有：

- 将新注册的实例信息推送给订阅该服务的订阅者
- 将新注册的实例信息增量同步给集群中的其他节点



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210716135220.png)



# 服务发现原理



服务发现的逻辑进一步简化为：

- 定时从注册中心查询最新服务实例列表信息
- 定时频率通常为6秒，发生异常为60秒

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210716135323.png)





# 集群节点通信原理



集群中节点通信原理可以进一步简化为：

- 每个节点用于全量的注册快照信息
- 新节点加入集群时会从集群中某节点发起全量同步

- 节点之间每隔5秒校验缓存的注册快照信息
- 节点之间每隔2秒进行一轮健康检查用于关闭/新建/刷新gRPC连接

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210716135412.png)



# 健康检查类型与场景

健康检查类型与场景进一步可以简化为：



- 临时节点通过gRPC连接保鲜实现，保鲜频率为5秒
- 临时节点注册使用Distro协议，持久节点注册使用Raft协议

- 持久节点支持客户端心跳和服务端探活两种方式
- 持久节点探活支持HTTP、TCP等探活类型



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210716135453.png)



