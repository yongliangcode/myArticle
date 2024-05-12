---
title: k8s01# K8s日志采集与服务质量QoS
categories: Kubernetes
tags: Kubernetes
date: 2022-07-17 11:55:01
---



# 引言

应用容器化后的日志采集该选择何种方式？该如何权衡？不同的服务质量QoS对Node的稳定性影响是怎么样的，本文就捋一捋这个。主要内容有：

* 日志采集三种方式
* 日志采集方式权衡
* Pod服务质量QoS



# 一、日志采集三种方式



K8s日志采集方式主要有原生方式、DaemonSet采集方式、Sidecar采集方式。



原生方式，日志写入标准输出和标准错误流，可通过 ` kubectl logs` 命令查看输出，如下图。

![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20220717232014.png)



通过日志轮替工具logrotate实现日志分割、压缩、删除、以及创建新的日志文件。



DaemonSet采集方式，在k8s的node节点上运行日志代理，由日志代理将日志采集到后端服务。

![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20220717112029.png)



SideCar采集方式，在一个POD中运行一个单独的日志采集代理容器，用于采集容器的日志。

![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20220717113607.png)

官方也说明了SideCar会带来更多的资源损耗，日志没有被kubelet接管，不能使用kubectl logs访问日志。



**小结：基于官方提供的采集方式，是应该选择DaemonSet还是SideCar呢？如何权衡？** 



官方日志管理说明文档：

```
https://kubernetes.io/zh-cn/docs/concepts/cluster-administration/logging/
```





# 二、日志采集方式权衡



**资源占用** ，SideCar采集模式每个POD运行一个日志代理容器，DaemonSet则是Node运行一个日志代理。



因此，DaemonSet会更充分共享资源，SideCar模式会占用更多的资源。



**运维难度**，如果业务POD内已经拥有多种辅助容器比如安全、链路等，再增加日志采集容器，POD中容器数量将会增加。



因此，众多SideCar容器要比DaemonSet增加更多的运维投入。



**隔离情况**，DaemonSet的日志代理，可以配置不同应用的采集路径配置，SideCar一对一单向定制。



因此，总的来说SideCar隔离性更好些。



**集群规模**，SideCar没有限制，DaemonSet模式实际支持的应用数与场景有关系，大体范围在几百到千级别。



因此，集群规模更多日志代理采集的能力是否跟得上的问题。



**升级问题**，DaemonSet模式日志采集升级业务无感知，SideCar模式的升级可能导致业务POD的重建，POD的重建还是对业务有感知。



因此，DaemonSet模式对业务更为透明无感。



如果想SideCar采集模式业务无感，可以使用OpenKruise提供的SidecarSet管理sidecar容器。



SidecarSet负责注入和升级k8s集群的sideCar容器，对业务无感。



**小结：在日志采集代理能力能满足需求的情况下，DaemonSet模式在运维复杂性、资源节省、升级方面更好的选择。**





# 三、Pod服务质量QoS



K8s使用服务质量QoS来决定Pod的调度和驱逐策略。



QoS分为三类，Guaranteed、Burstable、BestEffort。



**Guaranteed**类的POD，该POD中的每个容器内存和CPU必须指定，而且request和limit必须相等。



**Burstable**类的POD，POD里至少一个容器的内存或者CPU请求不满足Guaranteed要求，request和limit设置的不相同。



**BestEffort**类的POD， 容器没有设置内存和CPU的request或limit。



优先级Guaranteed>Burstable>BestEffort。



K8s资源回收驱逐策略，当Node上的内存或者CPU耗尽时，为了保护Node会驱逐POD，优先级低的POD会优先被驱逐。



日志采集类filebeat、ilogtail等类型的POD适合使用Burstable或者BestEffort。



**小结：为了Node的稳定性，日志采集类的POD设置为BestEffort类型的QoS更为安全。**



官方Pod 的服务质量文档

```
https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/quality-service-pod/
```





























