---
title: gRPC5# Reactor线程模型
categories: gRPC
tags: gRPC
date: 2020-12-08 11:55:01
---





现象直接报错：

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20211110163257.png)



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20211110163338.png)





![image-20211110163401076](/Users/yongliang/Library/Application Support/typora-user-images/image-20211110163401076.png)





```
客户端错误
io.grpc.StatusRuntimeException: INTERNAL: Panic! This is a bug!
	at io.grpc.stub.ClientCalls.toStatusRuntimeException(ClientCalls.java:262)
	at io.grpc.stub.ClientCalls.getUnchecked(ClientCalls.java:243)
	at io.grpc.stub.ClientCalls.blockingUnaryCall(ClientCalls.java:156)
	at com.hellobike.soa.protobuf.internal.SoaInvokerServiceGrpc$SoaInvokerServiceBlockingStub.call(SoaInvokerServiceGrpc.java:222)
	at com.hellobike.soa.rpc.invoker.v1.block.BlockingGrpcClientInvoker.invokeInternal(BlockingGrpcClientInvoker.java:31)
	at com.hellobike.soa.rpc.invoker.v1.block.AbstractBlockingGrpcClientInvoker.doInvoke(AbstractBlockingGrpcClientInvoker.java:42)
	at com.hellobike.soa.core.client.invoke.ClientInvoker.lambda$invoke$0(ClientInvoker.java:37)
	at com.hellobike.otter.context.Context.supplier(Context.java:623)
	at com.hellobike.soa.core.client.invoke.ClientInvoker.invoke_aroundBody0(ClientInvoker.java:37)
	at com.hellobike.soa.core.client.invoke.ClientInvoker$AjcClosure1.run(ClientInvoker.java:1)
	at org.aspectj.runtime.reflect.JoinPointImpl.proceed(JoinPointImpl.java:167)
	at com.hellobike.cat.core.cat.CatSoaHeaderContext.callWithHeader(CatSoaHeaderContext.java:23)
	at com.hellobike.cat.core.aspect.SoaRemoteCallAspect.lambda$doAction$0(SoaRemoteCallAspect.java:41)
	at com.hellobike.cat.core.aspect.AbstractAspect.processWithCatCatch(AbstractAspect.java:59)
	at com.hellobike.cat.core.aspect.SoaRemoteCallAspect.doAction(SoaRemoteCallAspect.java:33)
	at com.hellobike.soa.core.client.invoke.ClientInvoker.invoke(ClientInvoker.java:35)
	at com.hellobike.soa.core.invoke.filter.FilterInvoker.invoke(FilterInvoker.java:42)
	at com.hellobike.soa.core.client.filter.RouteFilter.invoke(RouteFilter.java:32)
	at com.hellobike.soa.core.invoke.filter.FilterInvoker.invoke(FilterInvoker.java:43)
	at com.hellobike.soa.core.client.filter.RateLimitExceptionFilter.invoke(RateLimitExceptionFilter.java:26)
	at com.hellobike.soa.core.invoke.filter.FilterInvoker.invoke(FilterInvoker.java:43)
	at com.hellobike.soa.core.client.filter.SentinelFilter.invoke(SentinelFilter.java:49)
	at com.hellobike.soa.core.invoke.filter.FilterInvoker.invoke(FilterInvoker.java:43)
	at com.hellobike.soa.core.client.filter.ServiceDowngradeFilter.invoke(ServiceDowngradeFilter.java:31)
	at com.hellobike.soa.core.invoke.filter.FilterInvoker.invoke(FilterInvoker.java:43)
	at com.hellobike.soa.core.invoke.filter.FilterChain.invoke(FilterChain.java:64)
	at com.hellobike.soa.core.proxy.jdk.JDKProxyHandler.invokeInternal(JDKProxyHandler.java:73)
	at com.hellobike.soa.core.proxy.jdk.JDKProxyHandler.invoke(JDKProxyHandler.java:44)
	at com.sun.proxy.$Proxy112.getShopSkuIdByShopId(Unknown Source)
	at sun.reflect.GeneratedMethodAccessor1294.invoke(Unknown Source)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at com.hellobike.soa.core.client.invoke.BaseInvokeHandler.lambda$invoke$0(BaseInvokeHandler.java:57)
	at com.hellobike.otter.context.Context.call(Context.java:614)
	at com.hellobike.soa.core.client.invoke.BaseInvokeHandler.invoke(BaseInvokeHandler.java:57)
	at com.sun.proxy.$Proxy112.getShopSkuIdByShopId(Unknown Source)
	at com.hellobike.hippo.shop.web.application.ifaces.impl.UserLoginIfaceImpl.getPoolId(UserLoginIfaceImpl.java:216)
	at com.hellobike.hippo.shop.web.application.ifaces.impl.UserLoginIfaceImpl.userLoginByPhone_aroundBody2(UserLoginIfaceImpl.java:138)
	at com.hellobike.hippo.shop.web.application.ifaces.impl.UserLoginIfaceImpl$AjcClosure3.run(UserLoginIfaceImpl.java:1)
	at org.aspectj.runtime.reflect.JoinPointImpl.proceed(JoinPointImpl.java:167)
	at com.hellobike.cat.core.aspect.AbstractAspect.processWithCatCatch(AbstractAspect.java:59)
	at com.hellobike.cat.core.aspect.SoaServiceAspect.doAction(SoaServiceAspect.java:32)
	at com.hellobike.hippo.shop.web.application.ifaces.impl.UserLoginIfaceImpl.userLoginByPhone(UserLoginIfaceImpl.java:103)
	at sun.reflect.GeneratedMethodAccessor1229.invoke(Unknown Source)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at com.hellobike.soa.rpc.server.v1.GrpcServerInvokerV1.doInvoke(GrpcServerInvokerV1.java:35)
	at com.hellobike.soa.core.server.invoke.ServerInvoker.invoke_aroundBody0(ServerInvoker.java:16)
	at com.hellobike.soa.core.server.invoke.ServerInvoker$AjcClosure1.run(ServerInvoker.java:1)
	at org.aspectj.runtime.reflect.JoinPointImpl.proceed(JoinPointImpl.java:167)
	at com.hellobike.cat.core.aspect.AbstractAspect.processWithCatCatch(AbstractAspect.java:59)
	at com.hellobike.cat.core.aspect.SoaProviderAspect.doAction(SoaProviderAspect.java:44)
	at com.hellobike.soa.core.server.invoke.ServerInvoker.invoke(ServerInvoker.java:15)
	at com.hellobike.soa.core.invoke.filter.FilterInvoker.invoke(FilterInvoker.java:42)
	at com.hellobike.soa.core.invoke.filter.FilterChain.invoke(FilterChain.java:64)
	at com.hellobike.soa.rpc.server.v1.RpcServerHandler.recordRun(RpcServerHandler.java:127)
	at com.hellobike.soa.rpc.server.v1.RpcServerHandler.call(RpcServerHandler.java:89)
	at com.hellobike.soa.protobuf.internal.SoaInvokerServiceGrpc$MethodHandlers.invoke(SoaInvokerServiceGrpc.java:271)
	at io.grpc.stub.ServerCalls$UnaryServerCallHandler$UnaryServerCallListener.onHalfClose(ServerCalls.java:182)
	at com.hellobike.soa.rpc.server.interceptor.SentinelServerFlowControlInterceptor$1.onHalfClose(SentinelServerFlowControlInterceptor.java:62)
	at io.grpc.PartialForwardingServerCallListener.onHalfClose(PartialForwardingServerCallListener.java:35)
	at io.grpc.ForwardingServerCallListener.onHalfClose(ForwardingServerCallListener.java:23)
	at com.hellobike.soa.rpc.server.interceptor.SoaServerAuthInterceptor$1.onHalfClose(SoaServerAuthInterceptor.java:45)
	at io.grpc.PartialForwardingServerCallListener.onHalfClose(PartialForwardingServerCallListener.java:35)
	at io.grpc.ForwardingServerCallListener.onHalfClose(ForwardingServerCallListener.java:23)
	at io.grpc.ForwardingServerCallListener$SimpleForwardingServerCallListener.onHalfClose(ForwardingServerCallListener.java:40)
	at com.hellobike.soa.rpc.server.interceptor.SoaServerSysTagInterceptor$SoaServerCallListenerForSysTag.lambda$onHalfClose$1(SoaServerSysTagInterceptor.java:126)
	at com.hellobike.soa.rpc.server.interceptor.SoaServerSysTagInterceptor$SoaServerCallListenerForSysTag.invokeAndSetTag(SoaServerSysTagInterceptor.java:64)
	at com.hellobike.soa.rpc.server.interceptor.SoaServerSysTagInterceptor$SoaServerCallListenerForSysTag.onHalfClose(SoaServerSysTagInterceptor.java:126)
	at io.grpc.PartialForwardingServerCallListener.onHalfClose(PartialForwardingServerCallListener.java:35)
	at io.grpc.ForwardingServerCallListener.onHalfClose(ForwardingServerCallListener.java:23)
	at io.grpc.ForwardingServerCallListener$SimpleForwardingServerCallListener.onHalfClose(ForwardingServerCallListener.java:40)
	at com.hellobike.soa.rpc.server.interceptor.SoaServerContextInterceptor$ContextualizedServerCallListener.onHalfClose(SoaServerContextInterceptor.java:117)
	at io.grpc.PartialForwardingServerCallListener.onHalfClose(PartialForwardingServerCallListener.java:35)
	at io.grpc.ForwardingServerCallListener.onHalfClose(ForwardingServerCallListener.java:23)
	at io.grpc.ForwardingServerCallListener$SimpleForwardingServerCallListener.onHalfClose(ForwardingServerCallListener.java:40)
	at io.grpc.PartialForwardingServerCallListener.onHalfClose(PartialForwardingServerCallListener.java:35)
	at io.grpc.ForwardingServerCallListener.onHalfClose(ForwardingServerCallListener.java:23)
	at io.grpc.ForwardingServerCallListener$SimpleForwardingServerCallListener.onHalfClose(ForwardingServerCallListener.java:40)
	at io.grpc.Contexts$ContextualizedServerCallListener.onHalfClose(Contexts.java:86)
	at io.grpc.PartialForwardingServerCallListener.onHalfClose(PartialForwardingServerCallListener.java:35)
	at io.grpc.ForwardingServerCallListener.onHalfClose(ForwardingServerCallListener.java:23)
	at io.grpc.ForwardingServerCallListener$SimpleForwardingServerCallListener.onHalfClose(ForwardingServerCallListener.java:40)
	at com.hellobike.soa.rpc.tracing.SoaServerTracingInterceptor$2.onHalfClose(SoaServerTracingInterceptor.java:189)
	at io.grpc.internal.ServerCallImpl$ServerStreamListenerImpl.halfClosed(ServerCallImpl.java:331)
	at io.grpc.internal.ServerImpl$JumpToApplicationThreadServerStreamListener$1HalfClosed.runInContext(ServerImpl.java:814)
	at io.grpc.internal.ContextRunnable.run(ContextRunnable.java:37)
	at io.grpc.internal.SerializingExecutor.run(SerializingExecutor.java:123)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
Caused by: java.util.concurrent.RejectedExecutionException: Task java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask@193c1c64 rejected from java.util.concurrent.ScheduledThreadPoolExecutor@28e53909[Terminated, pool size = 0, active threads = 0, queued tasks = 0, completed tasks = 3]
	at java.util.concurrent.ThreadPoolExecutor$AbortPolicy.rejectedExecution(ThreadPoolExecutor.java:2063)
	at java.util.concurrent.ThreadPoolExecutor.reject(ThreadPoolExecutor.java:830)
	at java.util.concurrent.ScheduledThreadPoolExecutor.delayedExecute(ScheduledThreadPoolExecutor.java:326)
	at java.util.concurrent.ScheduledThreadPoolExecutor.schedule(ScheduledThreadPoolExecutor.java:533)
	at com.hellobike.soa.core.utils.EventDebouncer.run(EventDebouncer.java:46)
	at com.hellobike.soa.rpc.nr.SoaNameResolver.refreshFromExternal(SoaNameResolver.java:125)
	at com.hellobike.soa.rpc.nr.SoaNameResolver.processUnitConfig(SoaNameResolver.java:116)
	at com.hellobike.soa.core.configcenter.PriorityDynamicConfigClient.processListener(PriorityDynamicConfigClient.java:142)
	at com.hellobike.soa.core.configcenter.PriorityDynamicConfigClient.lambda$addListener$0(PriorityDynamicConfigClient.java:68)
	at com.hellobike.soa.configcenter.zookeeper.DataCacheListener.lambda$internalDataChanged$1(DataCacheListener.java:49)
	at java.util.concurrent.CopyOnWriteArrayList.forEach(CopyOnWriteArrayList.java:891)
	at java.util.concurrent.CopyOnWriteArraySet.forEach(CopyOnWriteArraySet.java:404)
	at com.hellobike.soa.configcenter.zookeeper.DataCacheListener.internalDataChanged(DataCacheListener.java:49)
	at com.hellobike.soa.configcenter.zookeeper.AbstractDataListener.dataChanged(AbstractDataListener.java:113)
	at com.hellobike.soa.configcenter.zookeeper.ZookeeperClientDynamicConfigClient.triggerDataChanged(ZookeeperClientDynamicConfigClient.java:106)
	at com.hellobike.soa.core.configcenter.AbstractDynamicConfigClient.addListener(AbstractDynamicConfigClient.java:59)
	at com.hellobike.soa.core.configcenter.DynamicConfigManagement$ReferenceDynamicConfigClient.addListener(DynamicConfigManagement.java:76)
	at com.hellobike.soa.core.configcenter.PriorityDynamicConfigClient.addListener(PriorityDynamicConfigClient.java:69)
	at com.hellobike.soa.core.configcenter.client.CompositeClientDynamicConfigClient.addListener(CompositeClientDynamicConfigClient.java:26)
	at com.hellobike.soa.core.configcenter.DynamicConfigClientFactory$ReferenceDynamicConfigClientManagement.addListener(DynamicConfigClientFactory.java:116)
	at com.hellobike.soa.rpc.nr.SoaNameResolver.start(SoaNameResolver.java:91)
	at io.grpc.internal.ManagedChannelImpl.exitIdleMode(ManagedChannelImpl.java:397)
	at io.grpc.internal.ManagedChannelImpl$ChannelStreamProvider$1ExitIdleModeForTransport.run(ManagedChannelImpl.java:484)
	at io.grpc.SynchronizationContext.drain(SynchronizationContext.java:95)
	at io.grpc.SynchronizationContext.execute(SynchronizationContext.java:127)
	at io.grpc.internal.ManagedChannelImpl$ChannelStreamProvider.getTransport(ManagedChannelImpl.java:488)
	at io.grpc.internal.ManagedChannelImpl$ChannelStreamProvider.newStream(ManagedChannelImpl.java:518)
	at io.grpc.internal.ClientCallImpl.startInternal(ClientCallImpl.java:277)
	at io.grpc.internal.ClientCallImpl.start(ClientCallImpl.java:192)
	at io.grpc.ForwardingClientCall.start(ForwardingClientCall.java:32)
	at com.hellobike.soa.rpc.tracing.SoaClientTracingInterceptor$1.start(SoaClientTracingInterceptor.java:176)
	at io.grpc.ForwardingClientCall.start(ForwardingClientCall.java:32)
	at io.grpc.stub.MetadataUtils$HeaderAttachingClientInterceptor$HeaderAttachingClientCall.start(MetadataUtils.java:88)
	at io.grpc.stub.ClientCalls.startCall(ClientCalls.java:332)
	at io.grpc.stub.ClientCalls.asyncUnaryRequestCall(ClientCalls.java:306)
	at io.grpc.stub.ClientCalls.futureUnaryCall(ClientCalls.java:218)
	at io.grpc.stub.ClientCalls.blockingUnaryCall(ClientCalls.java:146)
```



