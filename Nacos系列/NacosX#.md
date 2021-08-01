





# 订阅事件的处理



### ClientEvent.ClientChangedEvent

坐标：DistroClientComponentRegistry#doRegister()

```java
@PostConstruct
public void doRegister() {
  // 注解@17
  DistroClientDataProcessor dataProcessor = new DistroClientDataProcessor(clientManager, distroProtocol,
                                                                          upgradeJudgement);
 
}
```

DistroClientDataProcessor构造函数

```java
public DistroClientDataProcessor(ClientManager clientManager, DistroProtocol distroProtocol,
                                 UpgradeJudgement upgradeJudgement) {
  this.clientManager = clientManager;
  this.distroProtocol = distroProtocol;
  this.upgradeJudgement = upgradeJudgement;
  NotifyCenter.registerSubscriber(this); // 注解@18
}
```

坐标：DistroClientDataProcessor#subscribeTypes()

```java
@Override
public List<Class<? extends Event>> subscribeTypes() {
  List<Class<? extends Event>> result = new LinkedList<>();
  // 注解@19
  result.add(ClientEvent.ClientChangedEvent.class);
  result.add(ClientEvent.ClientDisconnectEvent.class);
  result.add(ClientEvent.ClientVerifyFailedEvent.class);
  return result;
}
```

事件的处理，坐标：DistroClientDataProcessor#onEvent()

```java
@Override
public void onEvent(Event event) {
    if (EnvUtil.getStandaloneMode()) {
        return;
    }
    if (!upgradeJudgement.isUseGrpcFeatures()) {
        return;
    }
    if (event instanceof ClientEvent.ClientVerifyFailedEvent) {
        syncToVerifyFailedServer((ClientEvent.ClientVerifyFailedEvent) event);
    } else {
        syncToAllServer((ClientEvent) event); // 注解@20
    }
}
```

**注解@17** 在容器启动时注册了DistroClientDataProcessor

**注解@18** 在DistroClientDataProcessor构造方法中将其注册为订阅者

**注解@19** 订阅者监听的事件类型，其中包括了ClientChangedEvent事件

**注解@20** 将客户端变更事件同步给集群中的其他server

```java
 private void syncToAllServer(ClientEvent event) {
        Client client = event.getClient();
        // Only ephemeral data sync by Distro, persist client should sync by raft.
        if (null == client || !client.isEphemeral() || !clientManager.isResponsibleClient(client)) {
            return;
        }
        if (event instanceof ClientEvent.ClientDisconnectEvent) {
            DistroKey distroKey = new DistroKey(client.getClientId(), TYPE);
            distroProtocol.sync(distroKey, DataOperation.DELETE);
        } else if (event instanceof ClientEvent.ClientChangedEvent) {
            DistroKey distroKey = new DistroKey(client.getClientId(), TYPE);
            distroProtocol.sync(distroKey, DataOperation.CHANGE); 
        }
    }
```



**小结：** ClientEvent.ClientChangedEvent事件由DistroClientDataProcessor订阅处理，会把变更事件同步给集群中的其他server。



###  ClientOperationEvent.ClientRegisterServiceEvent







### MetadataEvent.InstanceMetadataEvent



### 











