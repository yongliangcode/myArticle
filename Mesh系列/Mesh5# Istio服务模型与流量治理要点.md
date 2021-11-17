

```
title: Mesh5# Istio服务模型与流量治理要点
categories: Mesh
tags: Mesh
date: 2021-10-16 11:55:01
```



# Istio服务模型



服务（Service）与版本（Version）：Istio中的服务在kubernetes中以service形式存在，可定义不同的服务版本。通过Deployment创建工作负载，通过Service关联这些负载，域名或者虚拟IP访问后端Pod。



服务实例（ServiceInstance）: 一个服务可以包含一组实例，在Kubernetes中用Endpoints实现，一组域名或者IP地址。





![](https://gitee.com/laoliangcode/md-picture/raw/master/img/Istio%E6%9C%8D%E5%8A%A1%E6%A8%A1%E5%9E%8B.drawio.png)





**服务（Service）示例：** 

```
apiVersion: v1
kind: Service
metadata:
  name: helloworld
  labels:
    app: helloworld
    service: helloworld
spec:
  ports:
  - port: 5000
    name: http
  selector:
    app: helloworld
```

备注：创建一个名称为helloworld的Service，指向“app: helloworld”的Pods，Kubernetes会自动创建一个和Service同名的Endpoints对象，Selector会持续跟踪映射属于helloworld的Pods。



**工作负载示例：** 

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld-v1
  labels:
    app: helloworld
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: helloworld
      version: v1
  template:
    metadata:
      labels:
        app: helloworld
        version: v1
    spec:
      containers:
      - name: helloworld
        image: docker.io/istio/examples-helloworld-v1
        resources:
          requests:
            cpu: "100m"
        imagePullPolicy: IfNotPresent #Always
        ports:
        - containerPort: 5000
```

备注：以Deployment方式创建工作负载，关联的版本与镜像。





# Istio流量治理



### 治理原理



通过Isito中VirtualService、DestinationRule、ServiceEntry等配置实现流量治理，即Istio将流量配置通过xDS下发给Enovy，通过拦截Inbound和Outbound流量，在流量经过时执行规则，实现流量治理。

通常流量治理有：动态变更负载均衡策略、不同版本灰度发布、服务治理限流熔断和故障注入演练等。



### 概念说明



#### 1.VirtualService

含义：形式上为虚拟服务，将流量转发到对应的后端服务。



**1.1 重要参数说明** 

* hosts
  必选字段，用于匹配访问地址，建议用字母的域名而不是IP地址

* gateways
  流量规则网关Gateway，可作用于网格中的SideCar和入口处的Gateway
  网格内部访问可以省略；网格外流量配置关联的Gateway表示执行该规则；网格内外都需要访问：需要配置Gateway和mesh两个字段

* http
  用于处理HTTP流量

* tls
  用于处理非终结的TLS和HTTPS流量

* tcp
  用于处理TCP流量，如果未定义http和tls所有流量将走tcp路由

* exportTo
  用于控制命名空间的可见性，可以控制一个命名空间下的VirtualService是否被其他命名SideCar和Gateway使用，未赋值表示全局可见

  

备注：VirtualService规则是一个数组，当第一个规则生效后将会跳出，不再检查后面的规则



**1.2 VirtualService典型应用** 

* 不同服务组合通过不同路径映射
* 不同版本映射通过不同URI映射到不同的服务版本



**1.3 示例** 

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: helloworld
spec:
  hosts:
  - "*"
  gateways:
  - helloworld-gateway
  http:
  - match:
    - uri:
        exact: /hello
    route:
    - destination:
        host: helloworld
        port:
          number: 5000   
```



#### 2.DestinationRule

含义：通常和VirtualService结合使用，VirtualService描述满足什么条件将流量转发到后端服务，DestinationRule描述到达后端了如何处理，类似于方法内部逻辑。



**2.1 重要参数说明** 

* hosts
  必填，表示规则使用的对象
* trafficPolicy
  规则具体内容，可包括负载均衡策略、异常点检查、连接池策略等
* subsets
  服务子集，常用于定义服务的版本
* exportTo
  用于控制命名空间的可见性，未赋值全局可见



**2.2 DestinationRule典型应用** 

* 负载均衡策略规则
* 不同版本灰度流量，例如：通过subSet
* 服务熔断限流，例如：通过请求量和请求超时等



**2.3 示例** 

```
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: bookinfo-ratings
spec:
  host: ratings.prod.svc.cluster.local
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN
  subsets:
  - name: testversion
    labels:
      version: v3
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
```



#### 3.ServiceEntry

含义：将网格外服务纳入istio网格。



**3.1 重要参数说明** 

* hosts
  必填，与ServiceEntry关联的主机名，主要用于http协议，其他协议不生效
* address
  表示与服务关联的地址
* port
  表示与服务关联的端口
* Location
  用于设置服务是在网格内还是网格外
  MESH_EXTERNAL：表示在网格外部，通过API访问外部服务
  MESH_INTERNAL：表示在网格内部，不能直接注册到网格注册中心的服务
* resolution
  服务发现的方式，NONE、STATIC、DNS等
* SubjectAltNames
  表示这个服务负载的SAN列表
* endpoints
  表示与网格服务关联的网络地址，可以是IP或者域名



**3.2 ServiceEntry典型应用** 

* 配置访问外部服务



**3.3 示例** 

```
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: baidu-external
spec:
  hosts:
    - www.baidu.com
  ports:
    - number: 80
      name: HTTP
      protocol: HTTP
  resolution: DNS
  location: MESH_EXTERNAL
```

