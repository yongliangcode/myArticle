

```
title: Mesh2# 第三方注册中心集成istio
categories: Mesh
tags: Mesh
date: 2021-09-21 11:55:01
```



# 引言



公司往往有自己的注册中心，有的使用Nacos、zookeeper等，还有自研的。这些在istio体系外的注册中心需要融入网格体系，让注册中心以及配置中心事件通知到istio，进而通过istio下发到数据面去。



第三方注册中心集成到istio通常有三种做法：

* 方式一  修改源码实现serviceregistry.Instance接口适配到service controller

* 方式二  通过自定义MCP Server，例如：Nacos提供了这部分实现

* 方式三  将第三方注册中心事件封装成istio的ServiceEntry和WorkloadEntry资源写入Kubernetes的api server

  istiod收到监听后完成转换



方式一需要修改istio源码，重度耦合后续升级istio比较困难；方式二看着比较简单却调试比较困难给开发造成障碍；方式三从业界实践来看采用的最为广泛。



本文将分析第三种方式如何集成istio的，在此之前需要先走查下kubernetes的大体架构。



# 一、k8s架构简述



### 架构与概念



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/k8s%E6%9E%B6%E6%9E%84%E5%8E%9F%E7%90%86.png)



**kube-apiserver：** 与Kubernetes资源交互的入口，可以通过kubectl或者client-go其他语言类库进行访问

**kube-scheduler:**   负责资源调度与计算，将Pod按照特定策略分发到计算节点

**etcd：** 键值存储数据库，保存Kubernetes集群相关数据

**kube-controller-manager:**  运行一系列列控制器的组件，比如：节点控制器、任务控制器、端点控制器等

**kubelete：** 运行在计算节点中，通过监听控制面接受指令，在节点内执行操作

**kube-proxy:** 运行在计算节点的网络代理，负责Pod内外的网络通信代理



### Pods创建流程

* 通过kubectl或者client-go类库向kube-apiserver发起创建请求
* kube-apiserver将请求信息持久化在etcd数据库
* kube-scheduler检测到请求后，计算Pod应该分配到哪个Node，并将分配策略写入etcd数据库
* Kubelet检测到etcd的分配策略后，执行该策略调用docker相关api创建container



# 二、第三方注册中心集成



### 架构图





![](https://gitee.com/laoliangcode/md-picture/raw/master/img/%E6%B3%A8%E5%86%8C%E4%B8%AD%E5%BF%83%E4%BA%8B%E4%BB%B6%E8%BD%AC%E6%8D%A2%E4%B8%BAistio.png)





**转换流程：** 

从注册中心（Zookeeper）获取变更事件，将其转换为ServiceEntry写入kube-apiserver；Istiod（Pilot）通过监听kube-apiserver收到ServiceEntry后经过转换通过xDS下发给数据面。



### 代码说明



直接去写比较耗时，快速掌握的方式是参考别人已经实现的，下面以社区项目dubbo2istio跟踪其如何将zookeeper转换的。

```
https://github.com/aeraki-framework/dubbo2istio
```

另外，还用到了dubbo 示例中的dubbo-demo-api-provider，地址如下。

```
https://github.com/apache/dubbo/tree/3.0/dubbo-demo/dubbo-demo-api/dubbo-demo-api-provider
```



**分析过程为：** 

* 通过dubbo-demo-api-provider注册节点到zookeeper注册中心
* 通过跟踪dubbo2istio代码观察其转换逻辑并通过client-go类库写入kube-apiserver
* 通过相关命令验证其的确已经写入到kube-apiserver



### 转换分析



**@1 ** 运行服务提供者dubbo-demo-api-provider，让其在注册中心完成注册。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/dubbo%E6%B3%A8%E5%86%8C%E4%BF%A1%E6%81%AF.png)



**@2** 运行dubbo2istio跟踪其逻辑



**@3** 获取zookeeper注册的节点将其转换为ServiceEntry，转换使用的类库为「istio.io/client-go」

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210910145430.png)



**@4** 将转换好的注册信息写入kube-apiserver，写入成功无错误返回

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210910192312.png)



**@5** 再通过代码查询是否写入成功，能够查到说明写入成功

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210910193902.png)



**@6** 通过命令行查询验证是否写入到kube-apiserver



**登陆istiod容器** 

```
kubectl -n istio-system exec -it istiod-56f8cc6cb5-xkg4m -- /bin/bash
```



**通过registryz命令查看** 

```
curl http://127.0.0.1:15014/debug/registryz
```

```

[
	{
        "Attributes": {
            "ServiceRegistry": "External",
            "Name": "org.apache.dubbo.demo.demoservice",
            "Namespace": "dubbo",
            "Labels": {
                "manager": "aeraki",
                "registry": "dubbo2istio"
            },
            "UID": "",
            "ExportTo": null,
            "LabelSelectors": null,
            "ClusterExternalAddresses": null,
            "ClusterExternalPorts": null
        },
        "ports": [
            {
                "name": "tcp-dubbo",
                "port": 20880,
                "protocol": "TCP"
            }
        ],
        "creationTime": "2021-09-10T09:58:18Z",
        "hostname": "org.apache.dubbo.demo.demoservice",
        "address": "0.0.0.0",
        "autoAllocatedAddress": "240.240.0.5",
        "Mutex": {},
        "Resolution": 0,
        "MeshExternal": false
    },
    // ....
]
```



备注：可以在istio中查到服务提供者「org.apache.dubbo.demo.demoservice」即从zookeeper注册中心成功将事件通过kube-apiserver通知到istio。

