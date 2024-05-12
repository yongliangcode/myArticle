---
title: MQ47# RocketMQ集群磁盘被拉满
categories: RocketMQ
tags: RocketMQ
date: 2023-11-21 20:55:01
---



# 现象反馈



收到RocketMQ集群磁盘IO告警，有反馈业务流量下跌受损。



# 问题定位



![image-20231117105525219](/Users/admin/Library/Application Support/typora-user-images/image-20231117105525219.png)



原因：RocketMQ集群磁盘被拉满，有个业务场景消息组大量消费历史数据，整个集群磁盘被拉满、导致集群15分钟不可用。



# 解决方案



* 止血措施： 给拉取历史消息的消费组增加限流、限流后恢复
* 其他措施：
  * 全局限流： rocketmq节点增加全局限流、避免被拉挂
  * 从节点消费：消费历史消息去从节点去消费、避免影响主节点
  * 积压治理：及时跟进消费加压巡检和治理（例如：超过1000万的积压及时跟进业务场景和接入）



