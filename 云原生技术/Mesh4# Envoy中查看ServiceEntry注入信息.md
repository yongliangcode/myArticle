

```
title: Mesh4# Envoy中查看ServiceEntry注入信息
categories: Mesh
tags: Mesh
date: 2021-10-16 11:55:01
```



# 引言

在Istio中提供了ServiceEntry的配置，将网格外的服务纳入网格管理。将第三方注册中心zookeeper、nacos等纳入Istio网格可以通过ServiceEntry纳入Istio的管理。这些如何注入的，流程是怎么样，下面通过示例将整个流程窜起来。



# ServiceEntry注入工作原理



ServiceEntry注入的流程图

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/istio%E7%9B%91%E5%90%ACServiceEntry.png)





**备注：注入流程如下** 

@1 将ServiceEntry注入到kube-apiserver中

@2 Istiod中通过kubeConfigController监听ServiceEntry配置的变化

@3 Istiod将ServiceEntry封装成PushRequest发送给XDSServer

@4 XDSServer转换为xDS格式下发给Envoy



# Envoy中查看ServiceEntry



**1.组织ServiceEntry配置** 

通过ServiceEntry配置baidu域名，将其作为网格服务的一部分serviceentry.yaml

```
---
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
  location: MESH_INTERNAL
```



**2.部署ServiceEntry配置** 

通过下面命令部署到Kubernetes api server中

```
kubectl apply -f serviceentry.yaml -n default
serviceentry.networking.istio.io/baidu-external created
```



**3.Istio中查看ServiceEntry信息**

登陆istiod容器

```
kubectl -n istio-system exec -it istiod-5c4b9cb6b5-6n68m -- /bin/bash
```



通过registryz命令查看，已经注入到istio中。

````
istio-proxy@istiod-5c4b9cb6b5-6n68m:/$ curl http://127.0.0.1:15014/debug/registryz
[
  {
    "Attributes": {
      "ServiceRegistry": "External",
      "Name": "www.baidu.com",
      "Namespace": "default",
      "Labels": null,
      "UID": "",
      "ExportTo": null,
      "LabelSelectors": null,
      "ClusterExternalAddresses": null,
      "ClusterExternalPorts": null
    },
    "ports": [
      {
        "name": "HTTP",
        "port": 80,
        "protocol": "HTTP"
      }
    ],
    "creationTime": "2021-10-14T03:01:24Z",
    "hostname": "www.baidu.com",
    "address": "0.0.0.0",
    "autoAllocatedAddress": "240.240.0.5",
    "Mutex": {},
    "Resolution": 1,
    "MeshExternal": false
  },
  // ...
]
````



**4.在Envoy查看xDS信息**

```
istioctl proxy-config route productpage-v1-6b746f74dc-2c55l -n default -o json
[
  //...
  {
                  "name": "www.baidu.com:80",
                  "domains": [
                      "www.baidu.com",
                      "www.baidu.com:80"
                  ],
                  "routes": [
                      {
                          "name": "default",
                          "match": {
                              "prefix": "/"
                          },
                          "route": {
                              "cluster": "outbound|80||www.baidu.com",
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
                              "operation": "www.baidu.com:80/*"
                          }
                      }
                  ],
                  "includeRequestAttemptCount": true
   }
   // ...         
]
```



小结：通过上面的命令追踪，ServiceEntry的示例下发到了数据面Envoy中。
