

```
title: Mesh5# EnvoyFilter使用
categories: Mesh
tags: Mesh
date: 2021-10-28 11:55:01
```



# 引言



# ServiceEntry注入工作原理

helloworld.yaml文件内容

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
---
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



helloworld-gateway.yaml 文件内容

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







```
kubectl apply -f helloworld.yaml
service/helloworld created
deployment.apps/helloworld-v1 created
deployment.apps/helloworld-v2 created
```



```
kubectl apply -f helloworld-gateway.yaml
gateway.networking.istio.io/helloworld-gateway created
virtualservice.networking.istio.io/helloworld created
```



```
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
curl http://$GATEWAY_URL/hello
```





```
curl http://$GATEWAY_URL/hello
Hello version: v1, instance: helloworld-v1-776f57d5f6-jvwr5
```



 ```
 curl http://$GATEWAY_URL/hello
 Hello version: v2, instance: helloworld-v2-54df5f84b-h979n
 ```



 ```
 curl -s -I -X HEAD http://$GATEWAY_URL/hello   
 HTTP/1.1 200 OK
 content-type: text/html; charset=utf-8
 content-length: 60
 server: istio-envoy
 date: Mon, 18 Oct 2021 08:27:23 GMT
 x-envoy-upstream-service-time: 143
 ```



```
kubectl apply -f envoyfilter.yaml
envoyfilter.networking.istio.io/header-filter created
```



```
kubectl apply -f envoyfilter.yaml -n istio-system
envoyfilter.networking.istio.io/header-filter created
```

