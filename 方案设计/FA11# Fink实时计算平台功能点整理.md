---
title: FA11# Fink实时计算平台功能点整理
categories: 方案设计
tags: 方案设计
date: 2022-02-22 11:55:01
---



# 引言



本文简要梳理下Flink实时计算平台提供的能力和功能点。



# 一、实时计算场景与特性



### 1.常见实时计算场景归纳

* 实时推荐：千人千面个性化推送
* 实时监控：反欺诈以及触发风控的异常与预警
* 实时报表：促销活动实时大屏等
* 实时检索：实时索引的构建和检索等
* 实时处理：数据实时清洗和汇总的其他场景



### 2.Flink实时计算框架特点

* 低延迟：毫秒级延迟

* 高吞吐：千万每秒吞吐

* 准确性：Exactly-once 状态一致性

* 易用性：作业开发使用高阶的Flink SQL API&Table API

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20220206204329.png)



# 二、实时计算平台架构



### 1.数据流线路图

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/Flink实时计算平台架构图.png)

**备注：** 

* 数据源：Binlog数据库数据增量订阅、SDK的埋点数据、Agent的上报数据、以及事件总线类数据
* 消息队列：数据源的数据被收集到消息队列中，通常选型Kafka
* Fink实时平台：Flink从消息队列消费数据跑作业任务
* 业务场景：由Flink任务类型支撑各类场景




### 2.架构示意图



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/Flink架构图.png)

备注：Table & SQL API通过Apache Calcite进行SQL解析，并转换成Calcite执行计划，最终调用Flink DataStream/DataSet API。





# 三、功能整理



* 资源上传：上传执行作业的jar文件或者gitlab地址平台进行打包编译成jar包
* 创建作业：配置作业的jar、并发以及一些策略等信息
* 作业信息：展现作业的运行状态、重启等
* 事件日志：记录操作的详细日志
* 异常日志：展现作业的日志和错误信息
* SQL作业：任务SQL化，提供Flink SQL作业调试、语法校验、编辑提交等功能






