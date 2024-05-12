

```
title: Mesh1# istio安装与部署
categories: ServiceMesh
tags: ServiceMesh
date: 2021-09-21 11:55:01
```



# 引言

Istio作为service mesh控制面的实施标准，先部署起来。然而会有一个坑要注意，否则无法访问到页面。这个坑是个示例的bug，已被人提了issue，我也被坑了一把。



# 一、准备工作



**1.安装Docker** 

通过命令行或者直接下载，由于网络原因我直接下载安装 ，下载地址：

```
https://hub.docker.com/editions/community/docker-ce-desktop-mac
```



**2.驱动安装**

```
curl -LO https://storage.googleapis.com/minikube/releases/latest/docker-machine-driver-hyperkit
```

```
chmod +x docker-machine-driver-hyperkit
```

```
sudo mv docker-machine-driver-hyperkit /usr/local/bin/
```

```
sudo chown root:wheel /usr/local/bin/docker-machine-driver-hyperkit
```

```
sudo chmod u+s /usr/local/bin/docker-machine-driver-hyperkit
```



**3.安装minikube** 

```
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
```

**验证版本** 

```
$ minikube version
minikube version: v1.22.0
```



**4.启动minikube** 

```
$ minikube start
😄  Darwin 10.15.7 上的 minikube v1.22.0
✨  根据现有的配置文件使用 docker 驱动程序
👍  Starting control plane node minikube in cluster minikube
🚜  Pulling base image ...
🏃  Updating the running docker "minikube" container ...
❗  This container is having trouble accessing https://k8s.gcr.io
💡  To pull new external images, you may need to configure a proxy: https://minikube.sigs.k8s.io/docs/reference/networking/proxy/
🐳  正在 Docker 20.10.7 中准备 Kubernetes v1.21.2…
🔎  Verifying Kubernetes components...
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
🌟  Enabled addons: storage-provisioner, default-storageclass
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```



# 二、安装与部署



**1.下载istio**

还是直接下载安装包，当前最新版本为1.11.0

```
https://github.com/istio/istio/releases/tag/1.11.0
```



**2.设置环境变量** 

```
vim ~/.bash_profile

export PATH=$PATH:/Users/yongliang/istio/istio-1.11.0/bin

source ~/.bash_profile
```



**3.安装istio** 

```
$ istioctl install --set profile=demo -y
✔ Istio core installed
✔ Istiod installed
✔ Egress gateways installed
✔ Ingress gateways installed
✔ Installation complete
```



**4.创建istio命名空间** 

```
kubectl create namespace istio-system
```



**5.设置自动注入envoy** 

```
$ kubectl label namespace default istio-injection=enabled
namespace/default labeled
```



**6.验证istio版本** 

```
$ istioctl version
client version: 1.11.0
control plane version: 1.11.0
data plane version: 1.11.0 (8 proxies)
```



小结：输出可以看出安装的istio客户端版本、控制面板版本和数据面版本。



# 三、部署示例程序



**1.部署示例** 

示例在安装目录sample目录下

```
-rw-r--r--@  1 yongliang  staff  11348  8 13 00:17 LICENSE
-rw-r--r--@  1 yongliang  staff   5866  8 13 00:17 README.md
drwxr-x---@  3 yongliang  staff     96  8 13 00:17 bin
-rw-r-----@  1 yongliang  staff    854  8 13 00:17 manifest.yaml
drwxr-xr-x@  5 yongliang  staff    160  8 13 00:17 manifests
drwxr-xr-x@ 21 yongliang  staff    672  8 13 00:17 samples
drwxr-xr-x@  5 yongliang  staff    160  8 13 00:17 tools
```

```
$ kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
service/details created
serviceaccount/bookinfo-details created
deployment.apps/details-v1 created
service/ratings created
serviceaccount/bookinfo-ratings created
deployment.apps/ratings-v1 created
service/reviews created
serviceaccount/bookinfo-reviews created
deployment.apps/reviews-v1 created
deployment.apps/reviews-v2 created
deployment.apps/reviews-v3 created
service/productpage created
serviceaccount/bookinfo-productpage created
deployment.apps/productpage-v1 created
```



**2.服务启动情况** 

```
$ kubectl get services
NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
details       ClusterIP   10.100.65.41     <none>        9080/TCP   4d2h
kubernetes    ClusterIP   10.96.0.1        <none>        443/TCP    4d4h
productpage   ClusterIP   10.107.21.144    <none>        9080/TCP   4d2h
ratings       ClusterIP   10.110.139.187   <none>        9080/TCP   4d2h
reviews       ClusterIP   10.106.238.130   <none>        9080/TCP   4d2h
```

pods为Running状态

```
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
details-v1-79f774bdb9-bkrbp       2/2     Running   4          4d2h
productpage-v1-6b746f74dc-2c55l   2/2     Running   4          4d2h
ratings-v1-b6994bb9-7nvs2         2/2     Running   4          4d2h
reviews-v1-545db77b95-mffvg       2/2     Running   4          4d2h
reviews-v2-7bf8c9648f-pmqw8       2/2     Running   4          4d2h
reviews-v3-84779c7bbc-sztp8       2/2     Running   4          4d2h
```



**3.把应用关联到istio网关** 

```
$ kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
gateway.networking.istio.io/bookinfo-gateway created
virtualservice.networking.istio.io/bookinfo created
```



**4.分析istio配置信息** 

```
$ istioctl analyze

✔ No validation issues found when analyzing namespace: default.
```



**5.设置入站IP和端口** 



**端口设置** 

```
$ export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')

$ export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
```

打印出来看看

```
$ echo "$INGRESS_PORT"
31688

$ echo "$SECURE_INGRESS_PORT"
31908
```



**设置入站IP** 

在官方提供的命令中是下面一段:

```
$ export INGRESS_HOST=$(minikube ip)
```

```
$ minikube ip
192.168.49.2
```

注意：照着执行后发现最后无法访问，下面有修正。



**启动minikube隧道** 

```
$ minikube tunnel
❗  The service istio-ingressgateway requires privileged ports to be exposed: [80 443]
🔑  sudo permission will be asked for it.
🏃  Starting tunnel for service istio-ingressgateway.
```



**修正网关地址** 

官方为命令：

```
$ export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
```

需要修正为：

```
$ export GATEWAY_URL=127.0.0.1
```

```
$ echo "$GATEWAY_URL"
127.0.0.1
```



备注：修正原因参见issue地址 https://github.com/istio/istio.io/issues/9340



**6.浏览器访问页面** 

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210821123126.png)

**7.安装Kiali仪表盘** 

```
$ kubectl apply -f samples/addons
$ kubectl rollout status deployment/kiali -n istio-system
deployment "kiali" successfully rolled out
```

**启动仪表盘** 

```
$ istioctl dashboard kiali
http://localhost:20001/kiali
```



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210821123244.png)



备注：当访问http://127.0.0.1/productpage时可以在仪表盘中观察到流量的流向和服务之间的关系。
