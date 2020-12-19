---
title: MQ21# 为何建议关闭RocketMQ预热配置
categories: RocketMQ
tags: RocketMQ
date: 2020-12-20 20:19:01
---



RocketMQ提供了一个预热配置项warmMapedFileEnable默认为关闭状态。曾在文章RocketMQ存储--映射文件预热【源码笔记】分析过文件预热流程。在预热文件时会填充1个G的假值0作为占位符，提前分配物理内存，防止消息写入时发生缺页异常。如此特性正如文章标题所说，为何建议关闭RocketMQ预热配置呢？

# 服务端监控

## 日志监控

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219081057.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219081114.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219081131.png)



## CPU情况

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219081205.png)

由服务端日志可以看出，在预热时broker会发生较长的耗时，10～30秒不等，CPU也会有小幅抖动，这会造成什么影响呢？接着看下文



<!--more-->



# 客户端发送监控

在broker预热时，客户端耗时长达5秒。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219081226.png)



在broker预热时，客户端耗时长达6秒

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20201219081243.png)



在此时间段，业务应用伴随着大量超时报警。



# 小结

在broker预热时，往往伴随着磁盘写入耗时过长、CPU小幅抖动、业务具体表现为发送耗时过长，超时错误增多。关闭预热配置从集群TPS摸高情况来看并未有明显的差异，但是从稳定性角度关闭却很有必要。