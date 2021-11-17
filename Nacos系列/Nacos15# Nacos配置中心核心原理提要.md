---
title: Nacos15# Nacos配置中心核心原理提要
categories: Nacos
tags: Nacos
date: 2021-08-26 11:55:01
---



# 引言

通过对Nacos配置中心源码阅读，将其核心原理归纳提炼。包含：客户端逻辑和服务端逻辑。



# 配置中心客户端逻辑 



### 1.客户端流程概览



客户端整体流程可以进一步简化为：

- 客户端通过长轮询的方式比较配置内容md5变更
- 长轮询通过从阻塞队列不断获取元素判断是否立即执行

- 阻塞队列无元素等待5秒执行



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210803194741.png)

### 2.Listener注册逻辑



 客户端Listener注册逻辑可以进一步简化为：

- 客户端缓存了CacheData
- 阻塞队列中添加了元素new Object()



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210803194818.png)



### 3.配置变更检测逻辑



客户端长轮询逻辑，可以进一步简化为：



- 客户端收到服务端推送的变更事件后发起MD5校验
- 客户端主动向服务端发起MD5校验



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/%E9%85%8D%E7%BD%AE%E4%B8%AD%E5%BF%83%E5%AE%A2%E6%88%B7%E7%AB%AFListener%E6%89%A7%E8%A1%8C%E9%80%BB%E8%BE%912.png)

### 4.阻塞队列添加时机



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/%E5%AE%A2%E6%88%B7%E7%AB%AF%E5%90%91%E9%98%BB%E5%A1%9E%E9%98%9F%E5%88%97%E6%B7%BB%E5%8A%A0%E6%97%B6%E6%9C%BA2.png)



# 配置中心服务端逻辑



### 1.服务端变更发布流程



服务端变更发布流程可以进一步简化为：

- 将变更内容写入数据库
- 向本节点连接的Client发送变更通知

- 向集群中其他节点发送变更通知



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/%E6%9C%8D%E5%8A%A1%E7%AB%AF%E5%8F%98%E6%9B%B4%E5%8F%91%E5%B8%83%E6%B5%81%E7%A8%8B2.png)

### 2.向Client发送变更通知



向Client发送变更通知进一步简化为：

- 每个节点只负责直连到本节点的Client发送通知
- 通知通过缓存的gRPC连接向Client发送



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/%E5%90%91Client%E5%8F%91%E9%80%81%E5%8F%98%E6%9B%B4%E9%80%9A%E7%9F%A52.png)





### 3.向其他节点发送变更通知

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210803195329.png)



# 小结



如何检测到配置内容的变更？无非以下两种方式，上文是具体细节。


1.客户端通过长轮询向服务端查询 
2.服务端向客户端发送变更通知

