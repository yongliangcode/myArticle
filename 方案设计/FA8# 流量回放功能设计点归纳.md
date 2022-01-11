---
title: FA8# 流量录制回放功能设计点归纳
categories: 方案设计
tags: 方案设计
date: 2022-01-14 11:55:01
---



# 引言



本文对流量录制和回放常见的方案、用途以及设计原理做个归纳整理。



# 一、解决的问题



### 1.回归测试覆盖率

 测试用例不足或者遗漏难以覆盖所有场景，导致回归测试费时费力，线上稳定存在隐患，通过真实流量录制在回归测试时进行覆盖。

* 回归特定接口和链路
* 回归特定业务场景
* 全量回归特定业务线



### 2.与全链路压测闭环

 解决全链路压测的数据准备问题，通过流量录制和回放系统与压测系统打通，形成从流量录制到压测闭环。

* 定向录制某个链路接口线上流量
* 对录制流量进行压测打标
* 增压发起全链路压测



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/%E5%BD%95%E5%88%B6%E6%B5%81%E9%87%8F%E4%B8%8E%E5%8E%8B%E6%B5%8B%E6%B5%81%E9%87%8F%E9%97%AD%E7%8E%AF.png)



### 3.数据的其他用处

* 抽取线上流量测试环境调试复现
* 其他用到线上请求数据的地方



# 二、常用方案



流量录制的方案和采用技术各种各样，下面梳理两种常用的技术方案。

### 1.GoReplay

```
https://github.com/buger/goreplay
```

**实现原理** 

依赖数据包捕获函数库（Packet Capture library）通过抓网络流量包，实现流量录制功能，go语言编写。



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20220110110439.png)



**优点归纳** 

* 支持流量录制
* 支持流量回放
* 支持流量过滤
* 支持插件机制
* 支持重写（URL、参数、Header等）
* 支持录制限流
* 抓包实现与服务语言无关



**缺点归纳** 

* 只支持HTTP，其他协议需要二次开发



### 2.jvm-sandbox-repeater

```
https://github.com/alibaba/jvm-sandbox-repeater
```

**实现原理** 

实现Java Instrumentation接口编写Agent，通过jvm对外编程接口规范JVMTI，实现对jvm运行信息的获取以及执行程序的加载，java开发。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/jvm-sandbox-repeater%E5%8E%9F%E7%90%86.png)



**优点归纳** 

* 流量录制和回放
* 快速扩展插件机制
* 已支持众多插件支持http/dubbo/mybatis/java/redis等



**缺点归纳** 

* 需要侵入运行服务的jvm
* 仅支持java服务



# 三、实现架构图

下图为基于上述两种方案的设计简图，通过运行一个录制代理ReplayAgent的方式实现。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/%E6%B5%81%E9%87%8F%E5%BD%95%E5%88%B6%E6%9E%B6%E6%9E%84%E5%9B%BE.png)





**功能点归纳** 

* 录制代理ReplayAgent负责接收控制台指令对GoReplay或sandbox-repeater管控
* 录制代理上报录制数据流量和监控信息

* 控制台对流量录制管理
  例如：数据完整性、录制任务状态和结果、录制时间、录制流量过滤
* 控制台对流量回放管理
  例如：回放结果状态、时长设定、回放速度等
* 控制台与压测平台、回归测试平台的通信





