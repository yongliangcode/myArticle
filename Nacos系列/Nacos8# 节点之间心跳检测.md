---
title: Nacos8# 集群中节点之间健康检查
categories: Nacos
tags: Nacos
date: 2021-07-03 11:55:01
---



# 引言

当新的节点加入集群或者集群中有节点下线了，集群之间可以通过健康检查发现。健康检查的频率是怎么样的？节点的状态又是如何变动的？状态的变动又会触发什么动作。带着这些问题本文捋一捋。



# 内容提要



### 健康检查

* Nacos节点会向集群其他节点发送健康检查心跳，每一轮频率为2秒
* 当健康检查异常时设置为不信任「SUSPICIOUS」状态，超过失败最大次数3次设置为下线「DOWN」状态
* 健康检查成功设置该节点为科通信「UP」状态
* 无论成功还是失败当节点状态变更时均发布MembersChangeEvent事件



### 成员变更事件

* 当集群节点成员变更时，MemberChangeListener会收到该事件
* 例如回调ClusterRpcClientProxy#onEvent触发refresh
* 刷新本节点与集群中其他节点的RPC状态，关闭无效的或者增加新的RPC连接



# 健康检查



代码翻到ServerMemberManager#onApplicationEvent，在Nacos启动的时候会启动一个定时任务，第一次延迟5秒执行，该定时任务即负责节点之间的心跳。

```java
@Override
public void onApplicationEvent(WebServerInitializedEvent event) {
    getSelf().setState(NodeState.UP);
    if (!EnvUtil.getStandaloneMode()) { // 注解@1
        GlobalExecutor.scheduleByCommon(this.infoReportTask, 5_000L);
    }
    EnvUtil.setPort(event.getWebServer().getPort());
    EnvUtil.setLocalAddress(this.localAddress);
    Loggers.CLUSTER.info("This node is ready to provide external services");
}
```

**注解@1** 非单机模式延迟5秒执行，执行的infoReportTask为MemberInfoReportTask。

```java
public abstract class Task implements Runnable {
    
    protected volatile boolean shutdown = false;
    
    @Override
    public void run() { // 注解@2
        if (shutdown) {
            return;
        }
        try {
            executeBody();
        } catch (Throwable t) {
            Loggers.CORE.error("this task execute has error : {}", ExceptionUtil.getStackTrace(t));
        } finally {
            if (!shutdown) {
                after();
            }
        }
    }
  
}
```

**注解@2** 看下这个Task执行逻辑，先执行 executeBody()，执行结束后执行after()。

```java
class MemberInfoReportTask extends Task {

    private final GenericType<RestResult<String>> reference = new GenericType<RestResult<String>>() {
    };

    private int cursor = 0;

    @Override
    protected void executeBody() {
        // ----------注解@1 start---------------
      	// 获取集群中除了自身以外的其他节点列表
        List<Member> members = ServerMemberManager.this.allMembersWithoutSelf();
        if (members.isEmpty()) {
            return;
        }
        // 定义一个游标
        this.cursor = (this.cursor + 1) % members.size();
        // 获取每个节信息
        Member target = members.get(cursor);
				//-----------注解@1 end-----------------
        Loggers.CLUSTER.debug("report the metadata to the node : {}", target.getAddress());
        // 注解@2
        final String url = HttpUtils
                .buildUrl(false, target.getAddress(), EnvUtil.getContextPath(), Commons.NACOS_CORE_CONTEXT,
                        "/cluster/report");
        try {
          	// 注解@3
            asyncRestTemplate
                    .post(url, Header.newInstance().addParam(Constants.NACOS_SERVER_HEADER, VersionUtils.version),
                            Query.EMPTY, getSelf(), reference.getType(), new Callback<String>() { 
                                @Override
                                public void onReceive(RestResult<String> result) { // 注解@4
                                    // 注解@5 返回版本不一致
                                    if (result.getCode() == HttpStatus.NOT_IMPLEMENTED.value()
                                            || result.getCode() == HttpStatus.NOT_FOUND.value()) {
                                        // ...
                                        Member memberNew = target.copy();
                                        if (memberNew.getAbilities() != null
                                                && memberNew.getAbilities().getRemoteAbility() != null && memberNew
                                                .getAbilities().getRemoteAbility().isSupportRemoteConnection()) {
                                            memberNew.getAbilities().getRemoteAbility()
                                                    .setSupportRemoteConnection(false);
                                            update(memberNew); // 更新节点属性
                                        }
                                        return;
                                    }
                                    // 注解@6
                                    if (result.ok()) {
                                        MemberUtil.onSuccess(ServerMemberManager.this, target);
                                    } else {
                                    	// 注解@7 处理失败上报
                                        MemberUtil.onFail(ServerMemberManager.this, target);
                                    }
                                }

                                @Override
                                public void onError(Throwable throwable) {
                                  	// 注解@8 处理失败上报
                                     MemberUtil.onFail(ServerMemberManager.this, target, throwable);
                                }

                                @Override
                                public void onCancel() {

                                }
                            });
        } catch (Throwable ex) {
            // ...
        }
    }

    @Override
    protected void after() {
        GlobalExecutor.scheduleByCommon(this, 2_000L); // 注解@9
    }
}
```

**注解@1** 获取集群中除了自身以外的其他节点列表，通过游标循环每个节点。

**注解@2** 构造每个节点的上报url请求路径为「/cluster/report」

**注解@3** 发起Post健康检查请求，请求内容为自身信息Member

**注解@4** 处理健康检查返回结果，有以下三种类型

**注解@5** 版本过低错误，这个可能在集群中版本不一致出现

**注解@6** 处理成功上报，更新该节点member的状态为UP表示科通信，设置失败次数为0，并发布成员变更事件

```java
public static void onSuccess(final ServerMemberManager manager, final Member member) {
    final NodeState old = member.getState();
    manager.getMemberAddressInfos().add(member.getAddress());
    member.setState(NodeState.UP); // 状态为UP可通信状态
    member.setFailAccessCnt(0); // 失败次数为0
    if (!Objects.equals(old, member.getState())) {
        manager.notifyMemberChange(); // 发布成员变更事件
    }
}
```

**注解@7&注解@8** 均为处理失败的上报，例如：集群中一个节点被kill -9 杀掉后。在nacos-cluster.log日志文件中会打印如下日志，并发布成员变更事件

```
2021-07-0x 16:30:24,994 ERROR failed to report new info to target node : x.x.x.x:8848, error : caused: Connection refused;

:2021-07-0x 16:30:30,995 ERROR failed to report new info to target node : x.x.x.x:8848, error : caused: Connection refused;
```

```java
public static void onFail(final ServerMemberManager manager, final Member member, Throwable ex) {
    manager.getMemberAddressInfos().remove(member.getAddress());
    final NodeState old = member.getState();

    // 设置该节点为「不信任」
    member.setState(NodeState.SUSPICIOUS);
    // 失败次数递增+1
    member.setFailAccessCnt(member.getFailAccessCnt() + 1);
    // 默认最大失败重试次数为3
    int maxFailAccessCnt = EnvUtil.getProperty("nacos.core.member.fail-access-cnt", Integer.class, 3);

    // If the number of consecutive failures to access the target node reaches
    // a maximum, or the link request is rejected, the state is directly down
    // 超过重试次数设置节点状态为「下线」
    if (member.getFailAccessCnt() > maxFailAccessCnt || StringUtils
            .containsIgnoreCase(ex.getMessage(), TARGET_MEMBER_CONNECT_REFUSE_ERRMSG)) {
        member.setState(NodeState.DOWN);
    }

    if (!Objects.equals(old, member.getState())) {
        manager.notifyMemberChange(); // 发布成员变更事件
    }
}
```

被kill -9 杀掉的节点显示状态为下线DOWN

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210702164901.png)



**注解@9** 执行完executeBody后延迟2秒继续执行executeBody，也就是检查健康检查的心跳频率为2秒，一轮全部节点检查结束后延迟2秒接着下一轮



无论检查成功还是失败，当节点状态变更时，发布成员变更事件。

```java
if (!Objects.equals(old, member.getState())) {
    manager.notifyMemberChange();
}

void notifyMemberChange() {
    NotifyCenter.publishEvent(MembersChangeEvent.builder().members(allMembers()).build());
}
```



**小结：** Nacos节点会向集群其他节点发送健康检查心跳，每一轮频率为2秒；当健康检查异常时设置为不信任「SUSPICIOUS」状态，超过失败最大次数3次设置为下线「DOWN」状态；健康检查成功设置该节点为科通信「UP」状态；无论成功还是失败当节点状态变更时均发布MembersChangeEvent事件。



# 成员变更事件

当集群中有节点下线或者新节点上线都会通过心跳健康检查探测对节点状态进行改变。而状态的变更均会触发成员变更事件MembersChangeEvent。那订阅到这个事件干啥呢？



ClusterRpcClientProxy继承了MemberChangeListener，当有MembersChangeEvent事件时会回调其onEvent方法。

```java
@Override
public void onEvent(MembersChangeEvent event) {
    try {
        List<Member> members = serverMemberManager.allMembersWithoutSelf();
        refresh(members);
    } catch (NacosException e) {
        // ...
    }
}
```

那接着看refresh方法。

```java
private void refresh(List<Member> members) throws NacosException {

    for (Member member : members) {
        if (MemberUtil.isSupportedLongCon(member)) {
            // 注解@10
            createRpcClientAndStart(member, ConnectionType.GRPC);
        }
    }
    Set<Map.Entry<String, RpcClient>> allClientEntrys = RpcClientFactory.getAllClientEntries();
    Iterator<Map.Entry<String, RpcClient>> iterator = allClientEntrys.iterator();
    List<String> newMemberKeys = members.stream().filter(a -> MemberUtil.isSupportedLongCon(a))
            .map(a -> memberClientKey(a)).collect(Collectors.toList());
    // 关闭旧的grpc连接
    while (iterator.hasNext()) {
        Map.Entry<String, RpcClient> next1 = iterator.next();
        if (next1.getKey().startsWith("Cluster-") && !newMemberKeys.contains(next1.getKey())) {
            Loggers.CLUSTER.info("member leave,destroy client of member - > : {}", next1.getKey());
            RpcClientFactory.getClient(next1.getKey()).shutdown();
            iterator.remove();
        }
    }

}
```

**注解@10** 为集群中每个节点member创建rcp client，在client启动时会先目标节点发送HealthCheckRequest，如果非健康节点将会被移除。见RpcClient类部分代码。

```java
boolean isHealthy = healthCheck();
// 非健康节点
if (!isHealthy) {
    if (currentConnection == null) {
        continue;
    }
    LoggerUtils.printIfInfoEnabled(LOGGER,
            "[{}]Server healthy check fail,currentConnection={}", name,
            currentConnection.getConnectionId());
    // 标记客户端状态为unhealthy
    rpcClientStatus.set(RpcClientStatus.UNHEALTHY);
    // 重置ReconnectContext移除serverInfo
    reconnectContext = new ReconnectContext(null, false);
```

这个意味着如果集群中有节点下线，与下线节点的rpc将会失效；同样如果集群中有新节点加入将会建立新的rpc通道。



**小结：** 当集群节点成员变更时，MemberChangeListener会收到该事件。例如回调ClusterRpcClientProxy#onEvent触发refresh。刷新本节点与集群中其他节点的RPC状态，关闭无效的或者增加新的RPC连接。





 





























