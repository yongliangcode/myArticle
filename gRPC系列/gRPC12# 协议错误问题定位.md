---
title: gRPC12# 协议错误问题定位
categories: gRPC
tags: gRPC
date: 2021-12-05 11:55:01
---



# 错误日志一

### 日志分析

收到业务同学反馈发现有SOA错误，但是对业务没有什么影响，错误内容如下：

```
io.grpc.StatusRuntimeException: INTERNAL: HTTP/2 error code: PROTOCOL_ERROR
Received Goaway
Stream 99 does not exist
  at io.grpc.stub.ClientCalls.toStatusRuntimeException(ClientCalls.java:262)
  at io.grpc.stub.ClientCalls.getUnchecked(ClientCalls.java:243)
  at io.grpc.stub.ClientCalls.blockingUnaryCall(ClientCalls.java:156)
  at com.hellobike.soa.protobuf.internal.SoaInvokerServiceGrpc$SoaInvokerServiceBlockingStub.call(SoaInvokerServiceGrpc.java:222)
  at com.hellobike.soa.rpc.invoker.v1.block.BlockingGrpcClientInvoker.invokeInternal(BlockingGrpcClientInvoker.java:31)
  at com.hellobike.soa.rpc.invoker.v1.block.AbstractBlockingGrpcClientInvoker.doInvoke(AbstractBlockingGrpcClientInvoker.java:42)
  at com.hellobike.soa.core.client.invoke.ClientInvoker.lambda$invoke$0(ClientInvoker.java:37)
  at com.hellobike.otter.context.Context.supplier(Context.java:623)
  at com.hellobike.soa.core.client.invoke.ClientInvoker.invoke(ClientInvoker.java:37)
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
```



**Goaway帧含义** 

先看下这个帧的含义：用于关闭连接或者发出错误， 端点必须将带有0x0以外的流标识符的GOAWAY帧视为类型为PROTOCOL_ERROR的连接错误

Goway帧抓包格式如下图所示：

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20211126195234.png)



小结：现象分析，该服务未客户端收到的HTTP/2二进制帧为Goaway，并抛出协议错误PROTOCOL_ERROR以及Stream 99 does not exist。



###  源码跟踪

先跟踪Stream 99 does not exist，在netty包DefaultHttp2ConnectionDecoder类中找到：

```java
private void verifyStreamMayHaveExisted(int streamId) throws Http2Exception {
  if (!connection.streamMayHaveExisted(streamId)) {
    throw connectionError(PROTOCOL_ERROR, "Stream %d does not exist", streamId);
  }
}
```

跟下streamMayHaveExisted方法逻辑：

```java
public boolean streamMayHaveExisted(int streamId) {
  return remoteEndpoint.mayHaveCreatedStream(streamId) || localEndpoint.mayHaveCreatedStream(streamId);
}

// 判断服务端帧是否为合法帧：1.是否大于0 2.是否为服务端帧（2的倍数）3.是否为合法区间的帧 
public boolean mayHaveCreatedStream(int streamId) {
  return isValidStreamId(streamId) && streamId <= lastStreamCreated();
}

// 在HTTP/2中客户端发起的StreamID必须是奇数，服务器发起的StreamID必须是偶数
public boolean isValidStreamId(int streamId) {
  return streamId > 0 && server == ((streamId & 1) == 0);
}

// 计算服务端合法帧区间
public int lastStreamCreated() {
    return nextStreamIdToCreate > 1 ? nextStreamIdToCreate - 2 : 0;
}
```

看下哪里调用了streamMayHaveExisted方法，在shouldIgnoreHeadersOrDataFrame方法进行调用。

```java
 private boolean shouldIgnoreHeadersOrDataFrame(ChannelHandlerContext ctx, int streamId, Http2Stream stream,
                                                String frameName) throws Http2Exception {
   if (stream == null) {
     if (streamCreatedAfterGoAwaySent(streamId)) {
       logger.info("{} ignoring {} frame for stream {}. Stream sent after GOAWAY sent",
                   ctx.channel(), frameName, streamId);
       return true;
     }

     // Make sure it's not an out-of-order frame, like a rogue DATA frame, for a stream that could
     // never have existed.
     verifyStreamMayHaveExisted(streamId);
		// ...
} 
```

调用shouldIgnoreHeadersOrDataFrame的地方有onDataRead和onHeadersRead方法，分别解析请求的header和Data部分。

**Data解析** 

```java
/**
 * Handles all inbound frames from the network.
 */
private final class FrameReadListener implements Http2FrameListener {
  @Override
  public int onDataRead(final ChannelHandlerContext ctx, int streamId, ByteBuf data, int padding,
                        boolean endOfStream) throws Http2Exception {
    Http2Stream stream = connection.stream(streamId);
    Http2LocalFlowController flowController = flowController();
    int bytesToReturn = data.readableBytes() + padding;
		
    final boolean shouldIgnore;
    try {
      // 调用点
      shouldIgnore = shouldIgnoreHeadersOrDataFrame(ctx, streamId, stream, "DATA");
    } catch (Http2Exception e) {
    }
    
   // ...  
}
```

**Header解析** 

```java
@Override
public void onHeadersRead(ChannelHandlerContext ctx, int streamId, Http2Headers headers, int streamDependency,
                          short weight, boolean exclusive, int padding, boolean endOfStream) throws Http2Exception {
  Http2Stream stream = connection.stream(streamId);
  boolean allowHalfClosedRemote = false;
  if (stream == null && !connection.streamMayHaveExisted(streamId)) {
    // 创建服务端Stream
    stream = connection.remote().createStream(streamId, endOfStream);
    // Allow the state to be HALF_CLOSE_REMOTE if we're creating it in that state.
    allowHalfClosedRemote = stream.state() == HALF_CLOSED_REMOTE;
  }
	// 调用点
  if (shouldIgnoreHeadersOrDataFrame(ctx, streamId, stream, "HEADERS")) {
    return;
  }
  // ...
}

private void incrementExpectedStreamId(int streamId) {
   if (streamId > nextReservationStreamId && nextReservationStreamId >= 0) {
     nextReservationStreamId = streamId;
   }
   nextStreamIdToCreate = streamId + 2; // 服务端帧偶数递增数
   ++numStreams;
}
```

**小结：**

1.代码调用链条如下
  onHeadersRead/onDataRead->shouldIgnoreHeadersOrDataFrame->streamMayHaveExisted->mayHaveCreatedStream->lastStreamCreated

2.从代码来看，在解析Header或者Data部分出现帧乱序，当前帧ID超过下一个帧预期的大小

3.疑问到底是解析header出现问题还是解析data？



# 错误日志二

```
Caused by: io.grpc.netty.shaded.io.netty.handler.codec.http2.Http2Exception$HeaderListSizeException: Header size exceeded max allowed size (8192)
at io.grpc.netty.shaded.io.netty.handler.codec.http2.Http2Exception.headerListSizeError(Http2Exception.java:189) ~[grpc-netty-shaded-1.33.1.jar:1.33.1]
at io.grpc.netty.shaded.io.netty.handler.codec.http2.Http2CodecUtil.headerListSizeExceeded(Http2CodecUtil.java:233) ~[grpc-netty-shaded-1.33.1.jar:1.33.1]
at io.grpc.netty.shaded.io.netty.handler.codec.http2.HpackEncoder.encodeHeadersEnforceMaxHeaderListSize(HpackEncoder.java:133) ~[grpc-netty-shaded-1.33.1.jar:1.33.1]
at io.grpc.netty.shaded.io.netty.handler.codec.http2.HpackEncoder.encodeHeaders(HpackEncoder.java:117) ~[grpc-netty-shaded-1.33.1.jar:1.33.1]
at io.grpc.netty.shaded.io.netty.handler.codec.http2.DefaultHttp2HeadersEncoder.encodeHeaders(DefaultHttp2HeadersEncoder.java:74) ~[grpc-netty-shaded-1.33.1.jar:1.33.1]
at io.grpc.netty.shaded.io.netty.handler.codec.http2.DefaultHttp2FrameWriter.writeHeadersInternal(DefaultHttp2FrameWriter.java:501) ~[grpc-netty-shaded-1.33.1.jar:1.33.1]
at io.grpc.netty.shaded.io.netty.handler.codec.http2.DefaultHttp2FrameWriter.writeHeaders(DefaultHttp2FrameWriter.java:268) ~[grpc-netty-shaded-1.33.1.jar:1.33.1]
at io.grpc.netty.shaded.io.netty.handler.codec.http2.Http2OutboundFrameLogger.writeHeaders(Http2OutboundFrameLogger.java:60) ~[grpc-netty-shaded-1.33.1.jar:1.33.1]
at io.grpc.netty.shaded.io.netty.handler.codec.http2.DecoratingHttp2FrameWriter.writeHeaders(DecoratingHttp2FrameWriter.java:53) ~[grpc-netty-shaded-1.33.1.jar:1.33.1]
at io.grpc.netty.shaded.io.grpc.netty.NettyClientHandler$PingCountingFrameWriter.writeHeaders(NettyClientHandler.java:966) ~[grpc-netty-shaded-1.33.1.jar:1.33.1]
at io.grpc.netty.shaded.io.netty.handler.codec.http2.DefaultHttp2ConnectionEncoder.sendHeaders(DefaultHttp2ConnectionEncoder.java:180) ~[grpc-netty-shaded-1.33.1.jar:1.33.1]
```

这个错误日志很明显，Header大小超过8KB，Header size exceeded max allowed size (8192)，这个大小时gPRC设定的。

```
private int maxHeaderListSize = GrpcUtil.DEFAULT_MAX_HEADER_LIST_SIZE;
public static final int DEFAULT_MAX_HEADER_LIST_SIZE = 8192;
```

**小结：** 该错误为gRPC设置了Header大小为8KB，超过该大小具体错误是Netty抛出的。



### 源码跟踪

下面是报错的地方

```java
public static void headerListSizeExceeded(int streamId, long maxHeaderListSize,
                                              boolean onDecode) throws Http2Exception {
        throw headerListSizeError(streamId, PROTOCOL_ERROR, onDecode, "Header size exceeded max " +
                                  "allowed size (%d)", maxHeaderListSize);
}
```

最后跟踪写入header时进行的校验

```java
 public ChannelFuture writeHeaders(ChannelHandlerContext ctx, int streamId, Http2Headers headers, int padding,
                                      boolean endStream, ChannelPromise promise) {
        return delegate.writeHeaders(ctx, streamId, headers, padding, endStream, promise);
}
```



# 根因截图



通过解析内容发现在gRPC header部分传入了大的链路ID导致，需要重构该链路生产方案。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20211129201438.png)





**总结：** 在gRPC通信时由于前面一条消息header头过大抛出异常Header size exceeded max allowed size (8192)，导致后续帧发生乱序。























