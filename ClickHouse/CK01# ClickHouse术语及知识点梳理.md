---
title: CK01# ClickHouse术语及知识点梳理
categories: Elasticsearch
tags: Elasticsearch
date: 2022-06-12 11:55:01
---



# 引言

尽管使用ElasticSearch冷热存储架构来存储日志，成本依旧高昂，而ElasticSearch的存储成本占用70%以上，寻找新的低成本存储方案也就成了主要解决方式。



根据测评ClickHouse存储成本可以降低到ElasticSearch的三分之一以上，本文就ClickHouse的术语与知识点做个梳理，主要内容有：

* 日志成本构成
* ClickHouse高性能特性
* 多主架构、分片与副本
* MergeTree系列表引擎



# 一、日志成本构成

当前的日志平台的成本主要由下面几个方面构成：

* 采集agent消耗的CPU和内存
* 日志Kafka集群成本
* Flink集群消费的计算资源
* ElasticSearch的存储成本
* OSS的存储成本

其中ElasticSearch的存储成本是主要部分，如何优化ES的存储成本呢？当前使用的冷热存储架构，第一天的数据存储在高配的热节点中，磁盘ESSD，之后的数据存在在低配的普通云盘中。



![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/%E5%86%B7%E7%83%AD%E5%88%86%E7%A6%BB%E6%9E%B6%E6%9E%84.png)ElasticSearch存储成本优化点：

* 推动业务减少不必要的日志输出
* 持续聚焦缩短存储时间
* 持续聚焦提高ElasticSearch的资源使用率
* 使用低成本ClickHouse的存储替换ElasticSearch



小结：尽管使用ElasticSearch冷热存储架构来存储日志，成本依旧高昂，而ElasticSearch的存储成本占用70%以上，寻找新的低成本存储方案也就成了主要解决方式。



根据测评ClickHouse存储成本可以降低到ElasticSearch的三分之一以上，下面梳理下ClickHouse特性与知识点。



# 二、ClickHouse高性能特性



众多的设计和优化成就了ClickHouse的高性能，下面找一些比较突出的点梳理下：

| 特性           | 描述                                                         |
| -------------- | ------------------------------------------------------------ |
| 列式存储       | 数据按列组织，同一列的数据保存在一起，不同的列分不同的文件保存 |
| 压缩算法       | 默认使用LZ4压缩算法，压缩比与数据相关，压缩比1:4~1:8不等     |
| 向量化执行引擎 | 1、利用CPU的SIMD（Single Instruction Multiple Data）单条指令操作多条数据<br />2、在寄存器层面实现数据并行执行，寄存器访问数据的速度是内存的300倍，是磁盘的3000万倍 |
| 众多表引擎     | 1、提供近30种的表引擎供选择，选择表表引擎意味着选择了不同的存储查询方式<br />2、MergeTree系列为官方主流系列 |



备注：在寄存器层面实现数据并行执行，SIMD大量用于文本转换、数据过滤、数据解压以及JSON转换等场景。



# 三、多主架构、分片与副本

### 1、多主架构

* ClickHouse采用多主架构，而不是主从架构

* 意味着不像ElasticSearch有Master、Data、Coordinating等角色的区分
* 访问中集群中的任何节点均可获得相同的结果



### 2、数据副本

* Clickhouse的副本其他组件并无差异，多一分相同的冗余数据
* 副本是表级别的，创建表时需要使用ReplicatedMergeTree系列引擎
* 基于多主架构通过zookeeper将执行语句分发到副本本地执行



### 3、数据分片

* ClickHouse集群中每个节点称为分片

* 可通过集群借助于ZooKeeper执行分布式DDL语句

  ```
  CREATE/DROP TABLE ON CLUSTER my_cluster
  ```

* 数据通过本地表（可以使用_local后缀命名）存储，使用Distributed以外的引擎
* 分布式表不存储数据，为本地表的代理，类似于分库分表组件，需使用Distributed引擎
* 分片规则需要声明分片键，否则分布式表中只包含一个分片，失去分片的意义





# 四、MergeTree系列表引擎



选择什么样的表引擎意味着选择了不同的数据存储组织方式，ClickHouse中有合并树、外部存储、内存、文件、接口与其他六个系列引擎，其中MergeTree合并树系列为其核心引擎。



| 合并树表引擎                 | 描述                                                         |
| ---------------------------- | ------------------------------------------------------------ |
| MergeTree                    | 1、MergeTree的基础引擎，该系列的其他引擎继承了其能力<br />2、具备数据分区、一级索引、二级索引、数据TTL一级存储能力 |
| ReplacingMergeTree           | 1、具备删除本分区重复数据的能力<br />2、通过ORDER BY排序键判断数据是否重复<br />3、在分区合并的时候删除本分区重复数据，跨分区无法删除重复数据<br />4、手动执行分区合并消耗大量时间 |
| SummingMergeTree             | 1、合并分区时按照定义条件合并汇总数据，降低查询开销<br />2、通过ORDER BY排序键作为聚合条件<br />3、数据的合并和汇总在分区合并时进行，跨分区不会汇总合并 |
| AggregatingMergeTree         | 1、SummingMergeTree的升级版<br />2、根据ORDER BY排序键聚合数据，并写入表中，本分区相同数据合并<br />3、在分区合并的时候执行聚合计算，跨分区不计算 |
| CollapsingMergeTree          | 1、折叠合并树通过增加不同sign标志的数据代替删除的方式，实现行数据的修改与删除<br />2、在合并分区的时候触发<br />3、对写入的数据有严格的顺序要求 |
| VersionedCollapsingMergeTree | 1、与CollapsingMergeTree作用相同通过对数据折叠，完成数据的删除与修改<br />2、通过标志位sign与版本号ver共同完成数据折叠<br />3、对写入的数据没有顺序要求，内部通过ver倒序判断 |



小结：基于MergeTree衍生引擎提供删除重复数据、汇总聚合、删除与修改的能力，然而他们只适合特定的场景，都在分区合并的执行，不支持跨分区。





















