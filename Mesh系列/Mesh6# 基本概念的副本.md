

```
title: Mesh4# Envoy中查看ServiceEntry注入信息
categories: Mesh
tags: Mesh
date: 2021-10-16 11:55:01
```



# 引言



# 治理原理



通过Isito中VirtualService、Destination、Gateway、ServiceEntry等配置实现流量治理，即通过Enovy拦截Inbound和Outbound流量，在流量经过时执行规则，实现流量治理。



通常流量治理有：动态修改负载均衡策略、不同版本灰度发布、服务治理限流熔断和故障演练等。

















**负载均衡** 

Envoy根据Istiod下发的负载均衡策略选择实例转发，Istio当前支持的负载均衡算法包括：轮询、随机和最小连接算法。



**Istio熔断** 

Istio通过连接池和故障实例隔离对应到Envoy的熔断和异常点隔离来实现

连接池：最大连接数和连接超时等管理

异常数检查：错误数量超过阈值将实例移除，移除一段时间再次加入尝试，失败继续隔离，另外支持驱逐的比例。



**Istio故障注入** 

在网络协议中注入对应协议的故障，干预协议之间的调用，不修改代码

例如：针对HTTP注入特定的code码或注入特定的延时



**灰度发布** 

Istio通过切分不同版本的分流策略来实现，80%流量版本到V1，20%流量到V2。同时支持根据Header内容将请求分发到不同的版本上。



蓝绿发布：新版本独立部署在另一套独立的资源上，新版本可用后将所有流量从老版本切换过来，新版本可用时删除老版本，新版本有问题快速切换到老版本。



AB测试：同时线上不上部署A/B两个版本接受流量，按照一定的策略让一部分用户使用A版本，一部分用户使用B版本，收集两部分用户的使用反馈，通过反馈决定使用哪个版本。



金丝雀发布：上线一个新版本，从老版本中切一部分流量到新版本中判断运行情况。





**服务访问入口**



入口代理服务，通过该入口将请求转发为后端服务，入口可以做权限、限流等。

1.Kubenetes服务访问入口

* 将服务发布成Loadbalancer类型的Service，通过外部端口访问指定的服务
* Ingress方式，Ingress是一个总入口根据不同的路径转发到后端不同的服务 xxx/A 转发到A服务，xxx/B转发到B服务。Ingress Controller作为kubernetes的控制器监听Kube-apiserver的Ingress对应后端服务的变化，实时获取后端Service和Endpoint的变化，结合规则动态负载均衡、

2.Istio服务访问入口

* istio通过Gateway访问网格内的服务，该Gateway与其他SideCar一样也是一个Envoy
* Gateway通常发布成Loadbalancer类型的Service
* Gateway资源四六层端口、TLS的基本功能，只定义了入口点
* VirtualService则定义了七层路由丰富内容，外部和内部访问规则使用VirtualService定义的规则

3.流量出口

* 大多数情况下通过SideCar可以执行治理功能
* 有时需要Egress Gateway支持要求所有出口流量经过该网关



外部服务接入网格和治理通过ServiceEntry实现。



# 基本概念



### VirtualService



**路由规则定义** 

定义了对特定目标服务的一组流量规则，形式为虚拟服务，后端可以为服务，也可以为DestinationRule定义的服务子集。

Service：服务

Service Version：服务版本

Source：发起调用的服务

Host：调用目标服务的地址



hosts：必选字段，用于匹配访问地址，建议用字母的域名而不是IP地址

gateways：流量规则网关Gateway，可作用于网格中的SideCar和入口处的Gateway
网格内部访问可以省略

网格外流量配置关联的Gateway表示执行该规则

网格内外都需要访问：需要配置Gateway和mesh两个字段

http：用于处理HTTP流量

tls：用于处理非终结的TLS和HTTPS流量

tcp：用于处理TCP流量，如果未定义http和tls所有流量将走tcp路由

VirtualService规则是一个数组，当第一个规则生效后将会跳出，不再检查后面的规则

exportTo：用于控制VirtualService跨命名空间的可见性，可以控制一个命名空间下的VirtualService是否被其他命名SideCar和Gateway使用。未赋值表示全局可见支持”.“和”*“。





### DestinationRule

Istio目标配置规则

DestinationRule经常和VirtualService结合使用，VirtualService用到的服务子集subset在DestinationRule上也有定义。

两者的定位：

VirtualService是一个虚拟服务，描述满足什么条件被哪个后端处理，除了路径还可以更开放的Match匹配。

DestinationRule描述这个请求到达后端后如何处理，类似于方法内部逻辑。

DestinationRule定义了访问到后端服务的访问策略，负载均衡和熔断策略定义在DestinationRule中。

hosts：必选字段，注册中心注册的服务名，也可以是网格外通过serviceEntry注入的外部注册中心



VirtualService典型应用：

* 不同服务组合通过不同路径映射
* 不同版本映射通过不同URI映射到不同的服务版本



**HTTP规则解析：** 满足HTTPMatchRequest条件的流量会被路由到HTTPRouteDestination、执行重定向（HTTPRewrite）、重试（HTTPRetry）、故障注入（HTTPFaultInjection）、跨站（CorsPolicy）策略等。



**HTTP规则解析：** HTTPRoute中最重要的字段是match，为一个HTTPMatchRequest类型数组，表示HTTP请求满足的条件，支持HTTP的uri、schema、method、authority、port等作为条件匹配请求。

Header、Port、

SourceLabels：请求来源的负载均衡标签，kubernetes上表示Pod上的标签。

Gateways：语义同VirtualService的定义，会覆盖VirtualService。

在HTTPRoute路由规则中，每个HTTPMatchRequest中的诸多属性都是逻辑与，几个元素之间的关系是逻辑或

例如：source取值”north“并且uri以“/hello”开头。



**HTTP路由目标** 

HTTPRoute上的route字段是一个HTTPRouteDestination类型的数组。HTTPRouteDestination定义了如何描述路由目标。包含三个字段：

* Destination：表示请求的目标，通过host、subset和port三个属性来描述。

  host表示istio中的服务名，可以是网格内或者网格外ServiceEntry注册的服务名

  如果在kubernetes中使用短域名，Istio会根据规则的空间来解析服务名，而不是根据service的命名空间来解析，建议通过全名指定。

  subset是在host定义一个子集，比如灰度流量转发到不同的版本上

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

  

  Weight: 表示流量分配比例

  

  **HTTP重定向** 从一个url到另一个url的永久重定向

  

  **HTTP重写** 重写url地址

  

  **HTTP重试** 

  

  **DestinationRule典型应用** 

  * 定义Subset 例如：不同版本的流量访问

  * 服务熔断  例如：请求数量和超时

  * 配置负载均衡规则，针对不同的版本执行不同的策略

  * TLS认证配置

    

    

  # Gateway

  

  Gateway接受网格边缘访问，并将流量接入到网格内服务。

  比如：下面示例通过80端口访问内部服务

  ```
  apiVersion: networking.istio.io/v1alpha3
  kind: Gateway
  metadata:
    name: helloworld-gateway
  spec:
    selector:
      istio: ingressgateway # use istio default controller
    servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
      - "*"
  ```

  

  Gateway通常和VirtualService配合使用，Gateway定义从服务外面怎么访问，VirtualService定义匹配到服务内部如何流转。

* selector 必选字段，表示Gateway负载，为入口处Envoy运行的Pod标签，通过该标签找到运行Gateway规则的Envoy
* server 必选字段，表示开放的一组服务列表
  Port: 表示哪个端口堆外开放
  hosts：必选字段，Gateway的发布服务地址
* DefaultEndpoint：表示流量转发的默认后端
* tls：在实际应用中考虑的安全问题，在入口处通过https访问协议



Gateway的典型应用：

1.将网格内的HTTP服务发布为HTTP外部访问

2.将网格内的HTTPS服务发布为HTTPS外部访问



# ServiceEntry



将网格外服务加入网格中，像网格一样管理。

* hosts：是一个必选字段，表示与ServiceEntry相关联的主机名
  HTTP流量匹配HTTP的Header的host或者Autority
  HTTPS或者TLS流量，这个字段匹配SNI
  其他协议这个字段不生效使用下面的address和port字段
* address：表示服务关联的虚拟IP地址
* port：表示与外部服务关联的端口
* Location: 用于设置服务是在网格内还是网格外
  MESH_EXTERNAL：表示在网格外部，通过API访问外部服务
  MESH_INTERNAL：表示在网格内部，不能直接注册到网格注册中心的服务
* resolution：服务发现的方式，NONE、STATIC、DNS等
* SubjectAltNames：表示这个服务负载的SAN列表
* endpoints：表示与网格服务关联的网络地址，可以是IP或者域名



**典型应用** 

* 配置访问外部服务



# SideCar

用于对数据面的资源更精细的控制，可以更精细的控制Enovy转发和接口的接口和协议；

* workloadselector：表示工作负载选择器，未配置则应用于整个命名空间

* egress：可以配置SideCar对网格内其他服务的访问，如果没有配置，表示命名空间可见

* ingress：配置SideCar对应工作负载的Inbound流量

  















VirtualService是Istio流量治理的核心配置，在Istio流量治理中最重要最复杂的规则。

基于Kubenertes CRD的方式描述，

 ```
 apiVersion: networking.istio.io/v1alpha3
 kind: Gateway
 metadata:
   name: helloworld-gateway
 spec:
   selector:
     istio: ingressgateway # use istio default controller
   servers:
   - port:
       number: 80
       name: http
       protocol: HTTP
     hosts:
     - "*"
 ---
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

网关”helloworld-gateway“拦截所有请求，当uri匹配到/hello时转发到helloworld服务。

























































# Istio服务与版本



**服务** 



服务是Istio主要管理的资源对象，包含：hostsName、ports、并指定Service的域名和端口列表，每个端口包含端口名称、端口号和端口协议。

以Kubernetes的Service形式存在，

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

创建一个名称为helloworld的Service，通过ClusterIP访问该Service，指向「app: helloworld」的Pods 。Kubernetes会自动创建一个和Service同名的Endpoints对象，Service的selector会持续关注属于Service的Pod，结果会更新到相应的Endpoints对象。

指定服务「helloworld」的端口是5000协议HTTP



```
kubectl get endpoints -n default
NAME          ENDPOINTS                                            AGE
helloworld    172.17.0.14:5000,172.17.0.16:5000                    47h
```



**版本** 

不同的服务的版本的灰度发布，Istio通过Deployment定义不同的版本。下面定义了两个版本V1和V2拥有共同的标签「app: helloworld」与Service的选择器标签一致，从而实现Service关联到这两个Deployment对应的Pod。

两个Deployment拥有不同的镜像版本（创建不同的Pod）、不同的服务版本、

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
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld-v2
  labels:
    app: helloworld
    version: v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: helloworld
      version: v2
  template:
    metadata:
      labels:
        app: helloworld
        version: v2
    spec:
      containers:
      - name: helloworld
        image: docker.io/istio/examples-helloworld-v2
        resources:
          requests:
            cpu: "100m"
        imagePullPolicy: IfNotPresent #Always
        ports:
        - containerPort: 5000
```





```
istioctl proxy-config routes helloworld-v1-776f57d5f6-jvwr5 --name 5000 -o json
[
    {
        "name": "5000",
        "virtualHosts": [
            {
                "name": "allow_any",
                "domains": [
                    "*"
                ],
                "routes": [
                    {
                        "name": "allow_any",
                        "match": {
                            "prefix": "/"
                        },
                        "route": {
                            "cluster": "PassthroughCluster",
                            "timeout": "0s",
                            "maxGrpcTimeout": "0s"
                        }
                    }
                ],
                "includeRequestAttemptCount": true
            },
            {
                "name": "helloworld.default.svc.cluster.local:5000",
                "domains": [
                    "helloworld.default.svc.cluster.local",
                    "helloworld.default.svc.cluster.local:5000",
                    "helloworld",
                    "helloworld:5000",
                    "helloworld.default.svc",
                    "helloworld.default.svc:5000",
                    "helloworld.default",
                    "helloworld.default:5000",
                    "10.108.83.210",
                    "10.108.83.210:5000"
                ],
                "routes": [
                    {
                        "name": "default",
                        "match": {
                            "prefix": "/"
                        },
                        "route": {
                            "cluster": "outbound|5000||helloworld.default.svc.cluster.local",
                            "timeout": "0s",
                            "retryPolicy": {
                                "retryOn": "connect-failure,refused-stream,unavailable,cancelled,retriable-status-codes",
                                "numRetries": 2,
                                "retryHostPredicate": [
                                    {
                                        "name": "envoy.retry_host_predicates.previous_hosts"
                                    }
                                ],
                                "hostSelectionRetryMaxAttempts": "5",
                                "retriableStatusCodes": [
                                    503
                                ]
                            },
                            "maxStreamDuration": {
                                "maxStreamDuration": "0s",
                                "grpcTimeoutHeaderMax": "0s"
                            }
                        },
                        "decorator": {
                            "operation": "helloworld.default.svc.cluster.local:5000/*"
                        }
                    }
                ],
                "includeRequestAttemptCount": true
            }
        ],
        "validateClusters": false
    }
]
```



```
istioctl proxy-config endpoints helloworld-v1-776f57d5f6-jvwr5 --cluster "outbound|5000||helloworld.default.svc.cluster.local"
ENDPOINT             STATUS      OUTLIER CHECK     CLUSTER
172.17.0.14:5000     HEALTHY     OK                outbound|5000||helloworld.default.svc.cluster.local
172.17.0.16:5000     HEALTHY     OK                outbound|5000||helloworld.default.svc.cluster.local
```



