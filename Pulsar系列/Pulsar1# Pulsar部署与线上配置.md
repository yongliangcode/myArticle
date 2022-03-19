---
title: Pulsar#1 Pulsar部署与线上配置
categories: Pulsar
tags: Pulsar
date: 2022-02-24 11:55:01
---



# 引言

Apache Pulsar越来越多的公司使用，与Apache Kafka、Apache RocketMQ并列成为消息领域三家马车，有必要对其研究一番。下面以笔者曾在生产环境使用的配置梳理，内容提要：

* Pulsar的安装与部署
* Pulsar集群的验证
* 生产环境机器配置
* 生产环境内存分配
* 生产环境broker配置项调整
* 生产环境bookie配置项调整



# 一、Pulsar安装与部署



### 1.下载安装包

Pulsar安装包包含了zookeeper、broker、bookie三个组件。

下载Pulsar二进制包

```
https://pulsar.apache.org/download/
```

解压压缩包

```
tar -zvxf apache-pulsar-2.9.1-bin.tar.gz
```



### 2.部署zookeeper

#### 2.1 修改zookeeper配置

创建目录

```
mkdir -p data/zookeeper

echo 1 > data/zookeeper/myid
```

修改zk配置，文件位于conf/zookeeper.conf 

```
# 数据目录
dataDir=data/zookeeper
# 日志目录
dataLogDir=data/zookeeper/logs
# zk集群配置，server.1~n
server.1=127.0.0.1:2888:3888
```

#### 2.2 后台启动zookeeper

```
bin/pulsar-daemon start zookeeper
doing start zookeeper ...
starting zookeeper, logging to /Users/admin/work/software_install/apache-pulsar-2.9.1/logs/pulsar-zookeeper-M-C02GL1NTQ05P.log
Note: Set immediateFlush to true in conf/log4j2.yaml will guarantee the logging event is flushing to disk immediately. The default behavior is switched off due to performance considerations.
```

通过pulsar-daemon管理pulsar组件

```
bin/pulsar-daemon help
Error: no enough arguments provided.
Usage: pulsar-daemon (start|stop|restart) <command> <args...>
where command is one of:
    broker              Run a broker server
    bookie              Run a bookie server
    zookeeper           Run a zookeeper server
    configuration-store Run a configuration-store server
    websocket           Run a websocket proxy server
    functions-worker    Run a functions worker server
    standalone          Run a standalone Pulsar service
    proxy               Run a Proxy Pulsar service
```

**备注：**可以通过pulsar-daemon命令对broker、bookie、zookeeper等组件启动、关闭或者重启。

#### 2.3 检查zookeeper是否启动成功

zookeeper启动日志和查看zookeeper进程

```
ps axu | grep zookeeper
```



### 3.元数据初始化

#### 3.1 初始化命令说明

```
bin/pulsar initialize-cluster-metadata \
--cluster pulsar-cluster-1 \
--zookeeper 127.0.0.1:2181 \
--configuration-store 127.0.0.1:2181 \
--web-service-url http://127.0.0.1:8080 \
--web-service-url-tls https://127.0.0.1:8443 \  
--broker-service-url pulsar://127.0.0.1:6650 \
--broker-service-url-tls pulsar+ssl://127.0.0.1:6651
```

**参数说明** 

| 参数                   | 说明                                                         |
| ---------------------- | ------------------------------------------------------------ |
| cluster                | 默认集群名称                                                 |
| zookeeper              | 本地集群使用的zk地址                                         |
| configuration-store    | 多个集群全局的zk集群地址，各个集群之间同步数据，单机群地址同上面参数zookeeper即可 |
| web-service-url        | Broker的管理流地址，例如创建删除主题等                       |
| web-service-url-tls    | Broker开启TLS，管理流则使用该地址                            |
| broker-service-url     | Broker数据流地址，发送接受消息等                             |
| broker-service-url-tls | Broker开启TLS，数据流则使用该地址                            |

备注：生产环境可以使用域名。

#### 3.2 查看初始化结果

```
bin/pulsar zookeeper-shell

[zk: localhost:2181(CONNECTED) 1] ls /
[admin, bookies, ledgers, pulsar, stream, zookeeper]
```



### 4.部署BookKeeper集群

#### 4.1 配置修改

```
bindAddress=127.0.0.1
advertisedAddress=127.0.0.1
zkServers=127.0.0.1:2181
```

**参数说明** 

| 参数              | 说明                                                         |
| ----------------- | ------------------------------------------------------------ |
| bindAddress       | 服务监听的地址，默认 0.0.0.0                                 |
| advertisedAddress | 服务向外发布的主机名或者IP，默认为IntetAddress.getLocalHost().getHostName |
| zkServers         | zk集群地址，可与broker共用                                   |

#### 4.2 命令启动

```
bin/pulsar-daemon start bookie
doing start bookie ...
starting bookie, logging to /Users/admin/work/software_install/apache-pulsar-2.9.1/logs/pulsar-bookie-M-C02GL1NTQ05P.log
Note: Set immediateFlush to true in conf/log4j2.yaml will guarantee the logging event is flushing to disk immediately. The default behavior is switched off due to performance considerations.
```



#### 4.3 测试bookie集群

```
bin/bookkeeper shell simpletest --ensemble 1 --writeQuorum 1 --ackQuorum 1 -- numEntries 1000

...
2022-02-19T23:43:03,391+0800 [main] INFO  org.apache.bookkeeper.tools.cli.commands.client.SimpleTestCommand - 722 entries written
2022-02-19T23:43:03,983+0800 [main] INFO  org.apache.bookkeeper.tools.cli.commands.client.SimpleTestCommand - 1000 entries written to ledger 0
2022-02-19T23:43:04,041+0800 [main] INFO  org.apache.bookkeeper.proto.PerChannelBookieClient - Closing the per channel bookie client for 127.0.0.1:3181
...
```

备注：通过simpletest命令向bookie集群写入测试数据，完成测试后会自动删除。



### 5.部署Broker集群

#### 5.1 修改配置

```
zookeeperServers=127.0.0.1:2181
configurationStoreServers=127.0.0.1:2181
bindAddress=127.0.0.1
# 默认InetAddress.getLocalHost().getHostName()
advertisedAddress=127.0.0.1
clusterName=pulsar-cluster-1
```

#### 5.2 启动broker

```
bin/pulsar-daemon start broker
doing start broker ...
starting broker, logging to /Users/admin/work/software_install/apache-pulsar-2.9.1/logs/pulsar-broker-M-C02GL1NTQ05P.log
Note: Set immediateFlush to true in conf/log4j2.yaml will guarantee the logging event is flushing to disk immediately. The default behavior is switched off due to performance considerations.

```

#### 5.3 验证集群

查看集群节点

```
bin/pulsar-admin brokers list cluster-1
"172.17.13.184:8080"
```

发送测试消息

```
bin/pulsar-client produce persistent://public/default/test -n 1 -m "Hello Pulsar"
...
2022-02-20T13:31:18,469+0800 [main] INFO  org.apache.pulsar.client.cli.PulsarClientTool - 1 messages successfully produced
...
```

消费测试消息

```
bin/pulsar-client consume persistent://public/default/test -n 100 -s "consumer-test" -t "Exclusive"
...
----- got message -----
key:[null], properties:[], content:Hello Pulsar
...
```



小结：至此测试集群搭建完成，下文将介绍生产环境配置的调整项。



# 二、生产环境配置

### 1.机器配置

下面为生产环境搭建Pulsar集群，由3个zookeeper节点、3个broker节点和5个bookie节点构成。

| 组件      | 配置           |
| --------- | -------------- |
| zookeeper | 4C8G100G * 3   |
| broker    | 16C64G500G * 3 |
| bookie    | 16C64G500G * 5 |

备注：每个组件集群部署时可以同城跨可用区部署，提高高可用。broker不存储消息100G即可，bookie存储消息通常需要较大磁盘，比如3T，具体根据消息量计算。

### 2.内存优化

| 配置项            | 内存大小或者比例，总大小                                     |
| ----------------- | ------------------------------------------------------------ |
| 系统OS缓存        | 1~2G                                                         |
| Jvm内存和堆外内存 | 1/2（除去系统缓存后剩余缓存的一半），其中Jvm heap占1/3，堆外内存Direct Memory占2/3 |
| PageCache内存大小 | 1/2（除去系统缓存后剩余缓存的一半）                          |

**2.1 broker内存调整** 

以内存64G大小，在文件conf/pulsar_env.sh修改如下内容：

```
PULSAR_MEM=${PULSAR_MEM:-"-Xms10g -Xmx10g -XX:MaxDirectMemorySize=20g"}
```

**2.2 bookie内存调整** 

以内存64G大小，在文件conf/bkenv.sh修改如下内容：

```
BOOKIE_MEM=${BOOKIE_MEM:-${PULSAR_MEM:-"-Xms10g -Xmx10g -XX:MaxDirectMemorySize=20g"}}
```

### 3.Broker调整项

| 配置项                                                       | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| zookeeperServers=x.x.x.x:2181,x.x.x.x:2181,x.x.x.x:2181      | 本地zookeeper集群地址                                        |
| configurationStoreServers=x.x.x.x:2181,x.x.x.x:2181,x.x.x.x:2181 | 配置存储Zookeeper集群地址                                    |
| bindAddress=x.x.x.x                                          | 服务监听的地址，可以为本机IP，默认为0.0.0.0                  |
| advertisedAddress=x.x.x.x                                    | 服务向外发布的主机名或者IP，默认为IntetAddress.getLocalHost().getHostName |
| clusterName=cluster-xxx                                      | 集群名称                                                     |
| brokerDeleteInactiveTopicsEnabled=false                      | 关闭自动删除不活动的主题                                     |
| defaultNumberOfNamespaceBundles=12                           | Bundle的数量应为broker数量的整数倍，默认为4                  |
| defaultRetentionSizeInMB=1T                                  | 消费确认过的消息超过该⼤⼩后会触发删除策略                   |
| defaultRetentionTimeInMinutes=1w                             | 消费确认过的消息超过指定时间后触发删除策略                   |
| backlogQuotaDefaultLimitGB=-1                                | 保持默认，未被消费确认的消息⼤存储⼤⼩<br />默认为-1表示没有限制，可以通过set-message-ttl设置过期时间，防⽌磁盘爆满 |
| backlogQuotaDefaultRetentionPolicy=producer_request_hold     | 保持默认，未被消费确认的消息超过存储⼤⼩的策略               |
| managedLedgerDefaultEnsembleSize=3                           | 创建Ledger时指定Ensemble的⼤⼩                               |
| managedLedgerDefaultWriteQuorum=3                            | 创建Ledger时指定Quorum的⼤⼩                                 |
| managedLedgerDefaultAckQuorum=2                              | 创建Ledger时指定ack Quorum的⼤⼩                             |
| dispatcherMaxReadBatchSize=500                               | ⼀次从bookkeeper读取的数量，默认为100条                      |
| loadBalancerAutoBundleSplitEnabled=false                     | 关闭auto bundle split功能，提⾼客户端稳定性                  |
| loadBalancerAutoUnloadSplitBundlesEnabled=false              | 关闭auto bundle split功能，提⾼客户端稳定性                  |
| loadBalancerSheddingEnabled=false                            | 禁⽌Pulsar⾃动均衡                                           |
| loadBalancerEnabled=false                                    | 禁⽌Pulsar⾃动均衡                                           |

备注：参数根据实际情况调整，在线上开启负载均衡时，发现有重复消息，此处将其关闭。

### 4.Bookie调整项

| 配置项                                                       | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| zkServers=x.x.x.x:2181,x.x.x.x:2181,x.x.x.x:2181             | 本地zookeeper集群地址                                        |
| journalDirectory=/data/bookkeeper/journal01,/data/bookkeeper/journal02 | BookKeeper存储其预写⽇志的⽬录，多个⽬录逗号进⾏分割，防⽌线程阻塞 |
| ledgerDirectories=/data/bookkeeper/ledgers01,/data/bookkeeper/ledgers02 | 指定存储BookKeeper输出ledger的⽬录。多个ledger⽬录，需要使⽤逗号分割 |

备注：journalDirectory和ledgerDirectories在条件允许的情况可以配置到不同的磁盘。







