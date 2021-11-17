---
title: Nacos3# 服务注册与发现服务端启动源码解析
categories: Nacos
tags: Nacos
date: 2021-06-05 11:55:01
---



# 引言

本文从gRPC的.proto文件解读其暴露的服务，由此生成gRPC的客户端/服务端存根。进而分析服务端加载启动过程。最近家里事情较多，本文短了点，大伙随便看看。



# 内容提要



### gRPC Service.proto解读

* 暴露用于服务端到客户端流式RPC的服务RequestStream#requestStream
* 暴露用于简单RPC调用的服务Request#request
* 暴露用于双向流式RPC调用的服务BiRequestStream#requestBiStream
* 三种方式入参均为Payload



### Server启动流程

* 定义了拦截器获取客户端的ip、port、connectId等
* 装配了.proto定义的两种调用方式，用于接受客户端请求
  简单调用方式Request#request和双向流调用方式BiRequestStream#biRequestStream
* 设置了服务启动端口、线程、接受消息的限制、压缩/解压缩类型



<!--more-->



# gRPC Service .proto解读

客户端和服务端通过gRPC通信，基于.proto生成响应的通信代码，那先看看.proto暴露了哪些服务。

**api/proto/nacos_grpc_service.proto** 

```java
syntax = "proto3"; // 注解@1 

// 注解@2
import "google/protobuf/any.proto";
import "google/protobuf/timestamp.proto";

option java_multiple_files = true; 
option java_package = "com.alibaba.nacos.api.grpc.auto"; // 注解@3

message Metadata { // 注解@4
  string type = 3;
  string clientIp = 8;
  map<string, string> headers = 7; 
}

message Payload { // 注解@5
  Metadata metadata = 2;
  google.protobuf.Any body = 3;
}

service RequestStream { // 注解@6
  // build a streamRequest
  rpc requestStream (Payload) returns (stream Payload) {
  }
}

service Request { // 注解@7
  // Sends a commonRequest
  rpc request (Payload) returns (Payload) {
  }
}

service BiRequestStream { // 注解@8
  // Sends a commonRequest
  rpc requestBiStream (stream Payload) returns (stream Payload) {
  }
}
```

**注解@1 ** 定义proto的版本

**注解@2** 导入其他的.proto文件

**注解@3** option可选的；指java类生成所在的包

**注解@4**  定义Metadata消息格式，生成对应Metadata类，包含了字符串类型type和clientIp、map类型的headers

**注解@5** 定义Payload消息格式，生成对应Payload类，包含了Metadata的引用、Any类型（对应java中Object）body

**注解@6** 定义service RequestStream会生产客户端和服务端存根用于grpc通信，暴露的服务为requestStream，类型为：服务端到客户端流式RPC，接受Payload对象参数，返回批量Payload数据

**注解@7**  定义service Request会生产客户端和服务端存根用于grpc通信，暴露的服务为request，类型为：简单RPC调用，接受Payload参数返回Payload类型对象

**注解@8** 定义service BiRequestStream会生产客户端和服务端存根用于grpc通信，暴露的服务为requestBiStream，类型为：双向流式RPC，接受批量Payload类型数据，返回批量Payload类型数据



**小结：** 我们从.proto的描述中能够发现，nacos server将暴露三个服务。@1 RequestStream#requestStream用于服务端到客户端流式RPC；@2 Request#request用于简单RPC调用；@3 BiRequestStream#requestBiStream用于双向流式RPC调用。三种的出入参均为Payload。



# Server启动流程

坐标com.alibaba.nacos.core.remote.BaseRpcServer，在nacos启动时执行

```java
@PostConstruct
public void start() throws Exception {
	 String serverName = getClass().getSimpleName();
   Loggers.REMOTE.info("Nacos {} Rpc server starting at port {}", serverName, getServicePort());

   startServer();
}
```

**源码解读**

```java
@Override
public void startServer() throws Exception {
  final MutableHandlerRegistry handlerRegistry = new MutableHandlerRegistry();

  // 注解@9
  ServerInterceptor serverInterceptor = new ServerInterceptor() {
    @Override
    public <T, S> ServerCall.Listener<T> interceptCall(ServerCall<T, S> call, Metadata headers,
                                                       ServerCallHandler<T, S> next) {
      Context ctx = Context.current()
        .withValue(CONTEXT_KEY_CONN_ID, call.getAttributes().get(TRANS_KEY_CONN_ID))
        .withValue(CONTEXT_KEY_CONN_REMOTE_IP, call.getAttributes().get(TRANS_KEY_REMOTE_IP))
        .withValue(CONTEXT_KEY_CONN_REMOTE_PORT, call.getAttributes().get(TRANS_KEY_REMOTE_PORT))
        .withValue(CONTEXT_KEY_CONN_LOCAL_PORT, call.getAttributes().get(TRANS_KEY_LOCAL_PORT));
      if (REQUEST_BI_STREAM_SERVICE_NAME.equals(call.getMethodDescriptor().getServiceName())) {
        Channel internalChannel = getInternalChannel(call);
        ctx = ctx.withValue(CONTEXT_KEY_CHANNEL, internalChannel);
      }
      return Contexts.interceptCall(ctx, call, headers, next);
    }
  };
  // 注解@10
  addServices(handlerRegistry, serverInterceptor);
  // 注解@11
  server = ServerBuilder.forPort(getServicePort()).executor(getRpcExecutor())
    .maxInboundMessageSize(getInboundMessageSize()).fallbackHandlerRegistry(handlerRegistry)
    .compressorRegistry(CompressorRegistry.getDefaultInstance())
    .decompressorRegistry(DecompressorRegistry.getDefaultInstance())
    .addTransportFilter(new ServerTransportFilter() {
      @Override
      public Attributes transportReady(Attributes transportAttrs) {  // transport/connection 建立回调
        InetSocketAddress remoteAddress = (InetSocketAddress) transportAttrs
          .get(Grpc.TRANSPORT_ATTR_REMOTE_ADDR);
        InetSocketAddress localAddress = (InetSocketAddress) transportAttrs
          .get(Grpc.TRANSPORT_ATTR_LOCAL_ADDR);
        int remotePort = remoteAddress.getPort();
        int localPort = localAddress.getPort();
        String remoteIp = remoteAddress.getAddress().getHostAddress();
        Attributes attrWrapper = transportAttrs.toBuilder()
          .set(TRANS_KEY_CONN_ID, System.currentTimeMillis() + "_" + remoteIp + "_" + remotePort)
          .set(TRANS_KEY_REMOTE_IP, remoteIp).set(TRANS_KEY_REMOTE_PORT, remotePort)
          .set(TRANS_KEY_LOCAL_PORT, localPort).build();
        String connectionId = attrWrapper.get(TRANS_KEY_CONN_ID);
        Loggers.REMOTE_DIGEST.info("Connection transportReady,connectionId = {} ", connectionId);
        return attrWrapper;

      }

      @Override
      public void transportTerminated(Attributes transportAttrs) { // transport/connection 关闭回调
        String connectionId = null;
        try {
          connectionId = transportAttrs.get(TRANS_KEY_CONN_ID);
        } catch (Exception e) {
          // Ignore
        }
        if (StringUtils.isNotBlank(connectionId)) {
          Loggers.REMOTE_DIGEST
            .info("Connection transportTerminated,connectionId = {} ", connectionId);
          connectionManager.unregister(connectionId);
        }
      }
    }).build();
	
  
  // 注解@12
  server.start();
}
```

**注解@9** 定义server的拦截器，可以从请求中获取connection id、ip、port等

**注解@10** 添加处理服务

```java
private void addServices(MutableHandlerRegistry handlerRegistry, ServerInterceptor... serverInterceptor) {

  // unary common call register.
  // 注解@10.1
  final MethodDescriptor<Payload, Payload> unaryPayloadMethod = MethodDescriptor.<Payload, Payload>newBuilder()
    .setType(MethodDescriptor.MethodType.UNARY)  // 服务调用方式UNARY
    .setFullMethodName(MethodDescriptor.generateFullMethodName(REQUEST_SERVICE_NAME, REQUEST_METHOD_NAME)) // 服务的接口名和方法名「request」
    .setRequestMarshaller(ProtoUtils.marshaller(Payload.getDefaultInstance())) // 请求序列化类
    .setResponseMarshaller(ProtoUtils.marshaller(Payload.getDefaultInstance())).build(); // 响应序列化类

  // 注解@10.2
  final ServerCallHandler<Payload, Payload> payloadHandler = ServerCalls
    .asyncUnaryCall((request, responseObserver) -> {
      grpcCommonRequestAcceptor.request(request, responseObserver);
    });

  // 注解@10.3
  final ServerServiceDefinition serviceDefOfUnaryPayload = ServerServiceDefinition.builder(REQUEST_SERVICE_NAME)
    .addMethod(unaryPayloadMethod, payloadHandler).build();

  // 注解@10.4
  handlerRegistry.addService(ServerInterceptors.intercept(serviceDefOfUnaryPayload, serverInterceptor));

  // bi stream register.
  // 注解@10.5
  final ServerCallHandler<Payload, Payload> biStreamHandler = ServerCalls.asyncBidiStreamingCall(
    (responseObserver) -> grpcBiStreamRequestAcceptor.requestBiStream(responseObserver));

  // 注解@10.6
  final MethodDescriptor<Payload, Payload> biStreamMethod = MethodDescriptor.<Payload, Payload>newBuilder()
    .setType(MethodDescriptor.MethodType.BIDI_STREAMING).setFullMethodName(MethodDescriptor
                                                                           .generateFullMethodName(REQUEST_BI_STREAM_SERVICE_NAME, REQUEST_BI_STREAM_METHOD_NAME))
    .setRequestMarshaller(ProtoUtils.marshaller(Payload.newBuilder().build()))
    .setResponseMarshaller(ProtoUtils.marshaller(Payload.getDefaultInstance())).build();

  // 注解@10.7
  final ServerServiceDefinition serviceDefOfBiStream = ServerServiceDefinition
    .builder(REQUEST_BI_STREAM_SERVICE_NAME).addMethod(biStreamMethod, biStreamHandler).build();


  // 注解@10.8
  handlerRegistry.addService(ServerInterceptors.intercept(serviceDefOfBiStream, serverInterceptor));

}
```

**注解@10.1** 构造MethodDescriptor，包括：服务调用方式简单RPC即UNARY、服务的接口名和方法名、请求序列化类、响应序列化类

**注解@10.2** 服务接口处理类，接受到request请求将调用执行

**注解@10.3** 构建暴露的服务「Request」

**注解10.4** 注册到内部的注册中心（Registry）中，可以根据服务定义信息查询实现类（普通对象request/response调用）

**注解@10.5** 服务接口处理类，接收到biRequestStream请求将调用执行

**注解@10.6** 构造MethodDescriptor，包括：服务双向流调用方式BIDI_STREAMING、服务的接口名和方法名、请求序列化类、响应序列化类

**注解@10.7** 构建暴露的服务「BiRequestStream」

**注解@10.8** 注册到内部的注册中心（Registry）中，可以根据服务定义信息查询实现类（双向流调用）

**注解@11** 设置server启动的端口（默认为 8848 + 1001 = 9849），getRpcExecutor线程执行器（线程数默认为 = 处理器核数*16） ，maxInboundMessageSize最大限制为10M，压缩解压缩使用gzip。

**注解@12** 注册发现server启动（grpc）



**小结：** server启动过程中主要干了三件事 @1定义了拦截器获取客户端的ip、port、connectId等；@2装配了.proto定义的两种调用方式，简单调用方式Request#request和双向流调用方式BiRequestStream#biRequestStream；@3设置了服务启动端口、线程、接受消息的限制、压缩/解压缩类型。





