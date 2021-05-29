

### 客户端

支持对用户透明的连接重连、故障切换、未ack消息的缓存、消息重传等。



### 跨集群复制



操作手册

https://pulsar.apache.org/docs/zh-CN/administration-geo



### 多租户

租户是topic的最基本单位，比命名空间和topic名称更为基本。

**租户**

* 授权机制
* 适用于租户配置的集群配置

**命名空间**

* pulsar为指定的多个租户配置了合适的容量
* 命名空间是租户的管理单元



 ```
persistent://tenant/app1/topic-1
 ```



### 主题压缩

消息高度可扩展的持久存储，是Pulsar构建的主要目标。



### Multiple advertised listeners

针对不同的网络暴露多个可访问的地址。

* advertisedListeners 用来指定多个对外暴露的监听地址
  例如：advertisedListeners=internal:pulsar://192.168.1.11:6660,internal:pulsar+ssl://192.168.1.11:6651
* internalListenerName 用来指定Broker使用的内部服务的URL，可以使用advertisedListeners中的一个值



只要advertisedListeners是可用的，客户端就能选择一个监听器作为服务的URl，建立起与broker的连接。



### Pulsar schema

可以通过schema定义复杂对象，通过pulsar api直接发送对象



### Pulsar Function

* 从一个或者多个topic中消费消息，并将提供的规则用于每条消息，并将运行结果发布到另一个topic



### Pulsar connector（Pulsar IO）

连接器，从一个数据源通过pulsar导入到另外的数据源。



### Pulsar SQL

Apache Pulsar用于存储事件数据流，事件数据使用预定义的数据结构化的，使用Presto的SQL查询内容。



### Tiered storage

分层存储，可以把消息导入到第三方廉价存储中，例如：S3



### 事务

事务语义允许事件应用将消费、处理、生产消息整个过程定义为一个原子操作。在Pulsar中，生产或消费者能够跨多个主题和分区处理的消息。并确保消息作为一个单元处理。

* 事务调度器 负责事务相关联的主题和订阅信息。事务提交后，事务调度器与主题与所在的broker交互完成事务
* 事务元数据  事务的元数据信息存储在事务日志中，也会备份在Pulsar的一个Topic中



### Helm Chart

在K8s中操作Pulsar集群。



### 系统管理

**Zookeeper** 负责本集群协调和配置相关服务 & 负责整个系统配置实现跨集群服务

**Bookeeper** 

```
前台启动bookie
$ bin/bookkeeper bookie

后台启动
$ bin/pulsar-daemon start bookie

校验bookie是否正常工作
$ bin/bookkeeper shell bookiesanity

使用这个命令的时候，底层的机制是在本地的 bookie 创建新的 ledger ，往里面写一些新的条目，然后读取它，最后删除这个ledger。

```



**Bookie下线注意事项**

1.确保EnsembleSize >= Write Quorum >= Ack Quorum

2.确保目标Bookie在listbookies命令列出的Bookie列表中

3.确保没有其他程序正在运行

```
1.登陆到bookie节点，检查是否有重复的分类账，如果存在则停用命令强制将他们复制
$ bin/bookkeeper shell listunderreplicated

2.运行退役命令
# 登陆到停用节点运行
$ bin/bookkeeper shell decommissionbookie

# 登陆到其他节点运行
$ bin/bookkeeper shell decommissionbookie -bookieid <target bookieid>

3.验证有没有分类账
$ bin/bookkeeper shell listledgers -bookieid <target bookieid>

4.验证是否还在列表中
$ bin/bookkeeper shell listbookies -rw -h 
$ bin/bookkeeper shell listbookies -ro -h

```







































