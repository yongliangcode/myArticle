### Brokers



一个无状态的组件主要负责运行其他两个组件。

* 一个HTTP服务器， 暴露了REST系统接口以及生产者和消费者之间查找的API
* 一个调度分发器，它是异步的TCP服务器，通过自定义二进制协议应用于所有的相关数据传输



消息会缓存一部分在broker侧的managed ledger，超过会从bookeeper中捞取。

broker的管理

https://pulsar.apache.org/docs/zh-CN/admin-api-brokers/





### 集群

由3部分组成

* 一个多个Pulsar broker

* 一个Zookeeper协调器
* 一组Bookeeper的Bookies用于存储消息



### 元数据存储

* 配置与仲裁存储    存储租户/命名域等全局一致的配置项
* 每个集群有独立的Zookeeper保存集群内部配置信息。例如：归属信息，broker负载报告，Bookeeper ledger信息等等



### Apache Bookeeper

Bookeeper是一种分布式的预写日志系统（WAL）。

```
预写式日志（Write-ahead logging，缩写 WAL）是关系数据库系统中用于提供原子性和持久性（ACID属性中的两个）的一系列技术。在使用WAL的系统中，所有的修改在提交之前都要先写入log文件中
```

* pulsar可以利用其建立很多独立的日志，成为ledgers。随着时间的推移，topic的多个ledgers会被创建
* 保证多系统挂掉，ledger的读取一致性
* 提供了不同的Bookies之间均匀的IO特性
* 容量和吞吐量都可以水品扩展，添加新节点能立即生效
* Bookies被设计成数以千计的并发读写ledgers。多个磁盘一个用于写日志，另外一个用于日志存储。这样可以将读操作的影响和写操作的延迟分离开

消费组的的位点（cursors）也会存储在bookeeper中，可以以扩张的方式存储消费位置。



### Ledgers

Ledger是一个只追加的数据结构，并且只要一个写入器，这个写入器负责多个Bookeeper存储节点（Bookie）的写入，Ledger的条目被复制到多个bookies。

* Broker可以创建、删除、添加内容到Ledger
* 一个Ledger被关闭后，这个Ledger只以只读模式打开（除非明确的要写数据或者是因为写入器挂掉导致ledger关闭）
* Ledger中的条目无用后，整个Ledger可以被删除



**Ledger读写一致性**

Bookeeper主要优势在于能够在系统故障时能够保障读写一致性。Ledger只能被一个进程写入，进程在写入时不会有冲突，写入也很高效。

故障后，ledger会启动一个恢复进程来确定Ledger的最终状态，并确认最后提交到日志是哪个条目。保证所有ledger读进程读到相同的内容。



**Managed ledgers**

消息流的抽象，有一个进程不断在流的末尾添加消息，并且有多个cursors消费这个流，每个cursor都要自己的消费位点。

单个Manager Ledger使用多个Bookeeper Ledgers存储数据，原因：

* 发生故障后原有的Ledger不能再写了需要创建新的
* 默认当消息都被确认后，Ledger将被移除



### 日志存储

*journal* files 存储Bookeeper的事务日志。更新到ledger之前，bookie需要确保更新事务被存储。

日志文件大小通过journalMaxSizeMB设置



### Pulsar Proxy

通常Pulsar客户端和Pulsar Broker直连。然而在某些情况下，客户端不知道broker的地址，例如云环境或者k8s上运行Pulsar。

选择代理模式后（可选），客户端的连接将通过该代理与broker通信。

* 使用Proxy不需要客户端更改任何配置
* Pulsar Proxy支持TLS加密和认证

代理操作文档

```
https://pulsar.apache.org/docs/zh-CN/administration-proxy/
```



### 服务发现

客户端使用单个URL与整个Pulsar实例进行通信，Pulsar内部提供服务发现机制。



















