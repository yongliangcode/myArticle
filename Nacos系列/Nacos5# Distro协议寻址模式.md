---
title: Nacos5# Distro协议寻址模式
categories: Nacos
tags: Nacos
date: 2021-06-20 11:55:01
---



# 引言



在Nacos服务端分析服务注册逻辑，就绕不开Distro协议。该协议为临时一致性协议，数据存储在缓存中。阿里专门为注册中心而设计的。后面文章逐步还原该协议承担的职责，本文先分析寻址模式。



# 内容提要



### 寻址概念

* 寻址是指如何发现Nacos集群中节点变化的，当检测到变化时能后及时更新节点信息。

  

### 寻址模式

* Nacos支持两种寻址模式分别为「文件寻址」和「地址服务器寻址」
* 默认为文件寻址，可以通过参数「nacos.core.member.lookup.type」设置取值为「file」或者「address-server」
* 文件寻址路径默认为 「${user.home}/nacos/conf/cluster.conf」
* 文件寻址cluster.conf配置文件的内容格式为「ip1:port,ip2:port」
* 地址服务器寻址默认为：http://jmenv.tbsite.net:8080/serverlist；其中域名、端口、url均可自定义
* 检测到集群节点变更时会更新缓存并发布MembersChangeEvent事件
* 为防止新节点没有初始化好，当检测到新节点加入时先设置该节点状态为DOWN，该节点不参与通信
* 过几秒通过节点之间通信将已初始化的新节点状态由DOWN设置为UP，该节点正式参与通信



<!--more-->



# 寻址初始化

寻址是指如何发现Nacos集群中节点变化的，当检测到变化时能后及时更新节点信息。Nacos提供了两种寻址模式，分别为 **文件寻址** 和**地址服务器寻址**。如果单机启动就本机一个节点也无所谓寻址。



接下来看下源码部分如何实现的。在DistroProtocol类中有一个成员变量ServerMemberManager memberManager，寻址的逻辑即封装在ServerMemberManager中。



坐标：ServerMemberManager#init()

```java
protected void init() throws NacosException {
  Loggers.CORE.info("Nacos-related cluster resource initialization");

  // 注解@1 
  this.port = EnvUtil.getProperty("server.port", Integer.class, 8848);
  // 注解@2
  this.localAddress = InetUtils.getSelfIP() + ":" + port;
  // 注解@3
  this.self = MemberUtil.singleParse(this.localAddress);
  // 注解@4
  this.self.setExtendVal(MemberMetaDataConstants.VERSION, VersionUtils.version);

  this.self.setAbilities(initMemberAbilities());
  // 注解@5
  serverList.put(self.getAddress(), self);
 	// 注解@6
  registerClusterEvent();
  // 注解@7
  initAndStartLookup();

  if (serverList.isEmpty()) {
    throw new NacosException(NacosException.SERVER_ERROR, "cannot get serverlist, so exit.");
  }

  Loggers.CORE.info("The cluster resource is initialized");
}
```

**注解@1** 可以通过server.port指定服务端端口，默认8848

**注解@2** 获取本地地址

**注解@3** 拆分IP和Port组装Member对象

**注解@4** 设置版本取自pom文件 version=${project.version}

**注解@5** 缓存本节点信息

**注解@6** 发布MembersChangeEvent事件并订阅IPChangeEvent事件

```java
private void registerClusterEvent() {
  // 发布MembersChangeEvent事件
  NotifyCenter.registerToPublisher(MembersChangeEvent.class,
                                   EnvUtil.getProperty("nacos.member-change-event.queue.size", Integer.class, 128));
  // 订阅IPChangeEvent事件
  NotifyCenter.registerSubscriber(new Subscriber<InetUtils.IPChangeEvent>() {
    @Override
    public void onEvent(InetUtils.IPChangeEvent event) {
      String newAddress = event.getNewIP() + ":" + port;
      ServerMemberManager.this.localAddress = newAddress;
      EnvUtil.setLocalAddress(localAddress);

      Member self = ServerMemberManager.this.self;
      self.setIp(event.getNewIP());

      String oldAddress = event.getOldIP() + ":" + port;
      ServerMemberManager.this.serverList.remove(oldAddress);
      ServerMemberManager.this.serverList.put(newAddress, self);

      ServerMemberManager.this.memberAddressInfos.remove(oldAddress);
      ServerMemberManager.this.memberAddressInfos.add(newAddress);
    }

    @Override
    public Class<? extends Event> subscribeType() {
      return InetUtils.IPChangeEvent.class;
    }
  });

}
```



# 寻址适配器



**注解@7** 初始化寻址模式适配器并启动；寻址模式分别为单机、配置文件、地址服务

```java
private void initAndStartLookup() throws NacosException {
  // 注解@7.1
  this.lookup = LookupFactory.createLookUp(this);
  isUseAddressServer = this.lookup.useAddressServer();
  // 注解@7.2
  this.lookup.start();
}
```

**注解@7.1** 获取寻址模式适配器

```java
public static MemberLookup createLookUp(ServerMemberManager memberManager) throws NacosException {
  if (!EnvUtil.getStandaloneMode()) {
    // 注解@7.1.1
    String lookupType = EnvUtil.getProperty(LOOKUP_MODE_TYPE);
    LookupType type = chooseLookup(lookupType);
    // 注解@7.1.2
    LOOK_UP = find(type);
    currentLookupType = type;
  } else {
    // 注解@7.1.3
    LOOK_UP = new StandaloneMemberLookup();
  }
  LOOK_UP.injectMemberManager(memberManager);
  Loggers.CLUSTER.info("Current addressing mode selection : {}", LOOK_UP.getClass().getSimpleName());
  return LOOK_UP;
}
```

**注解@7.1.1** 寻址类型可以通过「nacos.core.member.lookup.type」参数指定，取值为「file」或者「address-server」

**注解@7.1.2** 根据不同的类型实例化不同的MemberLookup分别为：FileConfigMemberLookup和AddressServerMemberLookup

```java
private static MemberLookup find(LookupType type) {
    if (LookupType.FILE_CONFIG.equals(type)) {
        LOOK_UP = new FileConfigMemberLookup();
        return LOOK_UP;
    }
    if (LookupType.ADDRESS_SERVER.equals(type)) {
        LOOK_UP = new AddressServerMemberLookup();
        return LOOK_UP;
    }
    throw new IllegalArgumentException();
}
```

**注解@7.1.3** 如果采用standalone模式实例化StandaloneMemberLookup

**注解@7.2** 寻址适配器启动

**standalone寻址适配器启动** 

```java
public void start() {
  if (start.compareAndSet(false, true)) {
    String url = InetUtils.getSelfIP() + ":" + EnvUtil.getPort();
    afterLookup(MemberUtil.readServerConf(Collections.singletonList(url)));
  }
}
```

**备注：** 坐标StandaloneMemberLookup#start()，获取本地地址执行afterLookup



**文件寻址适配器启动** 

```java
public void start() throws NacosException {
        if (start.compareAndSet(false, true)) {
            readClusterConfFromDisk();
            try {
                WatchFileCenter.registerWatcher(EnvUtil.getConfPath(), watcher);
            } catch (Throwable e) {
                Loggers.CLUSTER.error("An exception occurred in the launch file monitor : {}", e.getMessage());
            }
        }
 }

 private void readClusterConfFromDisk() {
   Collection<Member> tmpMembers = new ArrayList<>();
   try {
     List<String> tmp = EnvUtil.readClusterConf(); // 从磁盘文件读取节点列表
     tmpMembers = MemberUtil.readServerConf(tmp);
   } catch (Throwable e) {
     Loggers.CLUSTER
       .error("nacos-XXXX [serverlist] failed to get serverlist from disk!, error : {}", e.getMessage());
   }
	
   afterLookup(tmpMembers);
 }
```

**备注：** 默认从 ${user.home}/nacos/conf/cluster.conf文件中读取集群地址信息，文件格式为：「ip1:port,ip2:port」。读取后执行afterLookup。并注册FileWatcher监听cluster.conf的变化，有变更会被监听并更新缓存地址列表。



**地址服务器寻址适配器** 

```java
 public void start() throws NacosException {
   if (start.compareAndSet(false, true)) {
     this.maxFailCount = Integer.parseInt(EnvUtil.getProperty("maxHealthCheckFailCount", "12"));
     initAddressSys();
     run();
   }
 }
```

每5秒定时请求地址服务器

```java
private void run() throws NacosException {
    boolean success = false;
    Throwable ex = null;
    int maxRetry = EnvUtil.getProperty("nacos.core.address-server.retry", Integer.class, 5);
    for (int i = 0; i < maxRetry; i++) {
        try {
            syncFromAddressUrl();
            success = true;
            break;
        } catch (Throwable e) {
            ex = e;
            Loggers.CLUSTER.error("[serverlist] exception, error : {}", ExceptionUtil.getAllExceptionMsg(ex));
        }
    }
    if (!success) {
        throw new NacosException(NacosException.SERVER_ERROR, ex);
    }
    
    GlobalExecutor.scheduleByCommon(new AddressServerSyncTask(), 5_000L);
}
```

处理地址列表

```java
private void syncFromAddressUrl() throws Exception {
    RestResult<String> result = restTemplate
            .get(addressServerUrl, Header.EMPTY, Query.EMPTY, genericType.getType());
    if (result.ok()) {
        isAddressServerHealth = true;
        Reader reader = new StringReader(result.getData());
        try {
            afterLookup(MemberUtil.readServerConf(EnvUtil.analyzeClusterConf(reader)));
        } catch (Throwable e) {
            Loggers.CLUSTER.error("[serverlist] exception for analyzeClusterConf, error : {}",
                    ExceptionUtil.getAllExceptionMsg(e));
        }
        addressServerFailCount = 0;
    } else {
        addressServerFailCount++;
        if (addressServerFailCount >= maxFailCount) {
            isAddressServerHealth = false;
        }
        Loggers.CLUSTER.error("[serverlist] failed to get serverlist, error code {}", result.getCode());
    }
}
```

**备注：** 域名默认为「jmenv.tbsite.net」可以通过参数「address.server.domain」指定服务器地址；端口默认为「8080」可以通过参数「address.server.port」指定；url默认为「/serverlist」可以通过参数指定「address.server.url」。

默认为：http://jmenv.tbsite.net:8080/serverlist；每5秒钟定时向地址服务器请求获取地址列表；获取列表后执行afterLookup。



# 节点变更

三种适配器寻址最后都调用到了afterLookup，接下来看下这块逻辑。

```java
public void afterLookup(Collection<Member> members) {
    this.memberManager.memberChange(members);
}

synchronized boolean memberChange(Collection<Member> members) {

  if (members == null || members.isEmpty()) {
    return false;
  }
  // 是否包含本地地址
  boolean isContainSelfIp = members.stream()
    .anyMatch(ipPortTmp -> Objects.equals(localAddress, ipPortTmp.getAddress()));

  if (isContainSelfIp) {
    isInIpList = true;
  } else {
    isInIpList = false;
    members.add(this.self);
    Loggers.CLUSTER.warn("[serverlist] self ip {} not in serverlist {}", self, members);
  }

  // 集群中地址列表是否有变化
  boolean hasChange = members.size() != serverList.size();
  ConcurrentSkipListMap<String, Member> tmpMap = new ConcurrentSkipListMap<>();
  Set<String> tmpAddressInfo = new ConcurrentHashSet<>();
  for (Member member : members) {
    final String address = member.getAddress();
    Member existMember = serverList.get(address);
    if (existMember == null) { // 有新的节点加入
      hasChange = true;
     // 新增的节点先设置状态为DOWN，过几秒中通过心跳更改状态UP。防止新节点未成功启动而发请求
     member.setState(NodeState.DOWN); 
      tmpMap.put(address, member);
    } else {
     // 已存在，还会被更新
      tmpMap.put(address, existMember); 
    }

    if (NodeState.UP.equals(member.getState())) {
      tmpAddressInfo.add(address);
    }
  }

  serverList = tmpMap;
  memberAddressInfos = tmpAddressInfo;

  Collection<Member> finalMembers = allMembers();

  Loggers.CLUSTER.warn("[serverlist] updated to : {}", finalMembers);

  if (hasChange) { // 集群节点有变更
    MemberUtil.syncToFile(finalMembers); // 同步写入磁盘文件cluster.conf中
    Event event = MembersChangeEvent.builder().members(finalMembers).build();
    NotifyCenter.publishEvent(event); // 发布MembersChangeEvent事件
  }

  return hasChange;
}
```

**备注：** 通过寻址适配器获取的集群节点列表，会与缓存的节点信息进行比较。如果有变更会更新缓存、把全部节点写入磁盘文件cluster.conf、同时发布MembersChangeEvent事件。



**小结：** Nacos集群中的节点变更了怎么发现呢？Nacos提供两种模式一个是通过动态监听配置文件cluster.conf；另外一种是通过定时5秒去地址中心获取。



















