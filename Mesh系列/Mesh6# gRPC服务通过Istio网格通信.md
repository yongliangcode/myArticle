

```
title: Mesh6# gRPC服务通过Istio网格通信
categories: Mesh
tags: Mesh
date: 2021-11-07 11:55:01
```





# 前言



本文通过gRCP服务消费方mesha和gRPC服务提供方meshb，验证其部署在Istio网格的通信过程。通过该示例可以将外部注册中心接入网格，完成通信不再困难。



# 一、Isito配置提点



在Kubernetes集群中已安装了Isito，并查验下面几个参数是否正确设置。



**istio-config.yaml内容** 

```
---
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: default
  values:
    global:
      logging:
        level: default:debug
  meshConfig:
    accessLogFile: /dev/stdout
    defaultConfig:
      holdApplicationUntilProxyStarts: true
      proxyMetadata:
        ISTIO_META_DNS_CAPTURE: "true"
        ISTIO_META_DNS_AUTO_ALLOCATE: "true"
  components:
    pilot:
      hub: istio
      tag: 1.10.4

```



**关键参数说明** 

| 参数                            | 说明                                                         |
| ------------------------------- | ------------------------------------------------------------ |
| holdApplicationUntilProxyStarts | 默认false，设置为true表示业务容器需在sidecar Proxy容器启动完毕后启动 |
| ISTIO_META_DNS_CAPTURE          | 默认false，设置为true表示开启DNS代理，DNS的请求将被转发到sidecar |
| ISTIO_META_DNS_AUTO_ALLOCATE    | 默认false，设置为true表示DNS代理将自动为ServiceEntrys分配IP，不需要指定 |



**执行如下命令生效配置** 

```
istioctl install -y -f istio-config.yaml
```



**通过如下命令验证生效后的配置** 

```
kubectl describe IstioOperator installed-state -n istio-system >> istioOperator.conf
```



# 二、示例说明



**定义Proto** 

通过简单方法SayHello完成client与server通信。

```
syntax = "proto3";

option java_multiple_files = true;
option java_package = "com.melon.test.client.grpc";
option java_outer_classname = "HelloMesh";

package meshgrpc;

service Mesher {

  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

message HelloRequest {
  string name = 1;
}

message HelloReply {
  string message = 1;
}
```



**服务消费方mesha** 

```java
@RestController
public class MeshSender {

    @GetMapping("demo")
    public String meshWorker(){

         String target = "dns:///AppMeshClient.mesh:50000";
        // String  target = "127.0.0.1:50000";
        ManagedChannel channel = ManagedChannelBuilder.forTarget(target)
                .usePlaintext()
                .build();
        MesherGrpc.MesherBlockingStub blockingStub = MesherGrpc.newBlockingStub(channel);
        HelloRequest request = HelloRequest.newBuilder().setName("mesh demo!").build();
        HelloReply reply = blockingStub.sayHello(request);
        System.out.println(reply.getMessage());
        return reply.getMessage();

    }
}
```



**服务提供方meshb** 

```java
@Component
public class MeshReceiver {


  @PostConstruct
  public void receiverWorker() throws IOException {
    int port = 50000;
    Server server = ServerBuilder.forPort(port)
      .addService(new MeshBService())
      .build()
      .start();
    System.out.println("Server started.");
  }

  class  MeshBService extends MesherGrpc.MesherImplBase {

    public void sayHello(HelloRequest request,
                         io.grpc.stub.StreamObserver<HelloReply> responseObserver) {

      System.out.println("receiver client message: " + request.getName());
      HelloReply reply = HelloReply.newBuilder().setMessage("I'm from server " + request.getName()).build();
      responseObserver.onNext(reply);
      responseObserver.onCompleted();
    }
  }

}
```



**镜像推送** 

```xml
<plugin>
  <groupId>com.google.cloud.tools</groupId>
  <artifactId>jib-maven-plugin</artifactId>
  <version>3.1.4</version>
  <configuration>
    <from>
      <image>
        harbor.x.x/x/java:8u212-full
      </image>
      <auth>
        <username>${harbor.username}</username>
        <password>${harbor.password}</password>
      </auth>
    </from>
    <to>
      <image>x.x.x/x/${project.name}</image>
      <auth>
        <username>${harbor.username}</username>
        <password>${harbor.password}</password>
      </auth>
      <tags>
        <tag>${project.version}</tag>
        <tag>latest</tag>
      </tags>
    </to>
    <container>
      <ports>
        <port>x</port>
      </ports>
      <creationTime>USE_CURRENT_TIMESTAMP</creationTime>
    </container>
  </configuration>
</plugin>
```



说明：通过maven插件jib，执行命令如下命令「mvn compile jib:build」将镜像推到harbor仓库



# 三、gRPC服务提供方部署



设置meshb服务的Deployment

```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: meshb
  labels:
    app: meshb
spec:
  selector:
    matchLabels:
      app: meshb
  replicas: 1
  template:
    metadata:
      labels:
        app: meshb
    spec:
      imagePullSecrets:
        - name: xxxx #仓库名称
      containers:
        - name: meshb
          image: x.x.x.x/x/meshb:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 50000
```



执行下面命令生效

```
kubectl apply -f meshb.yaml
deployment.apps/meshb created
```



查看运行状态正常

```
# kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
meshb-565945d794-wcb8z              2/2     Running   0          9m19s
```



查看启动日志启动成功

```
# kubectl logs meshb-565945d794-wcb8z -n default

 Server started.
2021-11-05 09:21:19.828  INFO 1 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8181 (http) with context path ''
2021-11-05 09:21:19.847  INFO 1 --- [           main] com.melon.test.MeshbApplication          : Started MeshbApplication in 3.859 seconds (JVM running for 4.344)
```



备注：至此gRPC服务提供方已部署完成。



# 四、gRPC服务消费方部署



设置mesha服务的Deployment

```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mesha
  labels:
    app: mesha
spec:
  selector:
    matchLabels:
      app: mesha
  replicas: 1
  template:
    metadata:
      labels:
        app: mesha
    spec:
      imagePullSecrets:
        - name: middleware
      containers:
        - name: mesha
          image: x.x.x/x/mesha:latest
          ports:
            - containerPort: 7171
          imagePullPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: mesha
spec:
  selector:
    app: mesha
  type: LoadBalancer
  ports:
    - name: web
```

执行命令部署

```
# kubectl apply -f mesha.yaml
deployment.apps/mesha created
service/mesha created
```

查看服务状态

```
# kubectl get service
NAME                      TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
mesha                     LoadBalancer   10.x.61.x    <pending>     7171:30514/TCP   20s
```

查看Pod运行状态

```
# kubectl get pods
NAME                                READY   STATUS     RESTARTS   AGE
mesha-b559fc4f4-m9752               2/2     Running    0          87s
```



备注：至此服务消费方已部署完成。



# 五、发起gRPC调用



### 尝试发起访问



通过域名访问

```
http://x.x.x.x:30514/demo
```



页面打印如下错误

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20211105170804.png)

查看日志发现域名无法解析：

```
kubectl logs -f mesha-b559fc4f4-m9752  -n default

2021-11-05 09:01:37.372  WARN 1 --- [ault-executor-0] io.grpc.internal.ManagedChannelImpl      : [Channel<1>: (dns:///AppMeshClient.mesh:50000)] Failed to resolve name. status=Status{code=UNAVAILABLE, description=Unable to resolve host AppMeshClient.mesh, cause=java.lang.RuntimeException: java.net.UnknownHostException: AppMeshClient.mesh: Name or service not known
	at io.grpc.internal.DnsNameResolver.resolveAll(DnsNameResolver.java:436)
	at io.grpc.internal.DnsNameResolver$Resolve.resolveInternal(DnsNameResolver.java:272)
	at io.grpc.internal.DnsNameResolver$Resolve.run(DnsNameResolver.java:228)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
Caused by: java.net.UnknownHostException: AppMeshClient.mesh: Name or service not known
	at java.net.Inet4AddressImpl.lookupAllHostAddr(Native Method)
	at java.net.InetAddress$2.lookupAllHostAddr(InetAddress.java:929)
	at java.net.InetAddress.getAddressesFromNameService(InetAddress.java:1324)
	at java.net.InetAddress.getAllByName0(InetAddress.java:1277)
	at java.net.InetAddress.getAllByName(InetAddress.java:1193)
	at java.net.InetAddress.getAllByName(InetAddress.java:1127)
	at io.grpc.internal.DnsNameResolver$JdkAddressResolver.resolveAddress(DnsNameResolver.java:646)
	at io.grpc.internal.DnsNameResolver.resolveAll(DnsNameResolver.java:404)
	... 5 more
}
```



### 通过ServiceEntry映射服务提供方IP



查看服务提供方meshb的IP为x.x.0.17

```
# kubectl get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE     IP             NODE                NOMINATED NODE   READINESS GATES
mesha-b559fc4f4-m9752               2/2     Running   0          140m    x.x.1.117   k8s-servicemesh-3   <none>           <none>
meshb-565945d794-wcb8z              2/2     Running   0          118m    x.x.0.17    k8s-servicemesh-1   <none>           <none>
```

meshb-service-entry.yaml内容

```
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: meshb
spec:
  endpoints:
    - address: x.x.0.17
  hosts:
    - AppMeshClient.mesh
  location: MESH_INTERNAL
  ports:
    - name: grpc
      number: 50000
      protocol: grpc
  resolution: STATIC
```

执行如下命令ServiceEntry生效

```
kubectl apply -f meshb-service-entry.yaml
serviceentry.networking.istio.io/meshb created
```



再访问页面发现已经正常

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20211105192844.png)

备注：至此服务消费方在网格中向服务提供方发起调用。



# 六、日志校验追踪



**查看服务消费方mesha日志** 

```
kubectl logs -f mesha-b559fc4f4-m9752  -n default

I'm from server mesh demo!
```

备注：服务消费方mesha从服务提供方meshb返回的信息。



**查看服务提供方meshb日志** 

```
# kubectl logs -f meshb-565945d794-wcb8z -n default

2021-11-05 09:21:19.828  INFO 1 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8181 (http) with context path ''
2021-11-05 09:21:19.847  INFO 1 --- [           main] com.melon.test.MeshbApplication          : Started MeshbApplication in 3.859 seconds (JVM running for 4.344)
receiver client message: mesh demo!
```

备注：服务提供方meshb收到了服务消费方mesha的请求。



**查看服务消费方mesha的envoy日志** 

```
kubectl logs -f -l app=mesha -c istio-proxy -n default
2021-11-05T10:25:17.772317Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-11-05T10:52:35.856445Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, transport is closing
2021-11-05T10:52:36.093048Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
[2021-11-05T11:06:23.510Z] "GET /demo HTTP/1.1" 500 - via_upstream - "-" 0 x 37 x "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/95.0.4638.54 Safari/537.36" "49cd74d8-86eb-4ef5-bd71-8236d188c2f9" "x.x.x.x:30514" "x.x.1.x:7171" inbound|7171|| 127.0.0.6:41007 10.x.1.x:7171 x.x.0.0:20826 - default
2021-11-05T11:22:58.633956Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, transport is closing
2021-11-05T11:22:59.100387Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
[2021-11-05T11:27:23.842Z] "POST /meshgrpc.Mesher/SayHello HTTP/2" 200 - via_upstream - "-" 17 33 371 319 "-" "grpc-java-netty/1.28.0" "79f2edbd-9c31-4265-87fb-38a594d9383b" "AppMeshClient.mesh:50000" "10.x.0.x:50000" outbound|50000||AppMeshClient.mesh 10.x.1.x:51914 240.240.0.63:50000 10.166.1.117:40544 - default
[2021-11-05T11:27:23.527Z] "GET /demo HTTP/1.1" 200 - via_upstream - "-" 0 x x 728 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/95.0.4638.54 Safari/537.36" "859344b6-012d-4457-a391-2b4d13aa3e35" "10.69.31.x:30514" "10.x.1.x:7171" inbound|7171|| 127.0.0.6:47065 10.x.1.x:7171 x.x.0.0:2410 - default
```



**查看服务消费方meshb的envoy日志** 

```
# kubectl logs -f -l app=meshb -c istio-proxy -n default
2021-11-05T09:57:12.490428Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, transport is closing
2021-11-05T09:57:12.765887Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-11-05T10:24:48.767511Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, transport is closing
2021-11-05T10:24:48.914475Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-11-05T10:57:52.604281Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, transport is closing
2021-11-05T10:57:52.757736Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-11-05T11:26:56.551824Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, transport is closing
2021-11-05T11:26:56.987345Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
[2021-11-05T11:27:23.844Z] "POST /meshgrpc.Mesher/SayHello HTTP/2" 200 - via_upstream - "-" 17 33 333 316 "-" "grpc-java-netty/1.28.0" "79f2edbd-9c31-4265-87fb-38a594d9383b" "AppMeshClient.mesh:50000" "10.x.0.x:50000" inbound|50000|| 127.0.0.6:44221 10.x.0.x:50000 10.x.1.x:51914 - default
```



**登陆mesha的Pod中验证** 

```
kubectl describe pod/mesha-b559fc4f4-m9752 -n default
```

```
[12:04:44root@mesha-b559fc4f4-m9752 /] C:2
# curl -v AppMeshClient.mesh
* About to connect() to AppMeshClient.mesh port 80 (#0)
*   Trying 240.240.0.63...
* Connected to AppMeshClient.mesh (240.240.0.63) port 80 (#0)
> GET / HTTP/1.1
> User-Agent: curl/7.29.0
> Host: AppMeshClient.mesh
> Accept: */*
>
< HTTP/1.1 503 Service Unavailable
< content-length: 91
< content-type: text/plain
< date: Fri, 05 Nov 2021 12:04:59 GMT
< server: envoy
<
* Connection #0 to host AppMeshClient.mesh left intact
upstream connect error or disconnect/reset before headers. reset reason: connection failure
```

备注：登陆mesha的Pod，通过访问域名AppMeshClient.mesh，发现其自动分配IP「240.240.0.63」并指向sidecar「envoy」，即会把业务容器的流量通过DNS Proxy重定向到sidecar。



说明：通过查看服务消费方日志、服务提供方日志以及其数据面enovy日志，说明其调用在istio网格中进行。







