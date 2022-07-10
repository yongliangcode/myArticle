---
title: CK02# ClickHouse分布式表读写原理梳理
categories: Elasticsearch
tags: Elasticsearch
date: 2022-06-12 11:55:01
---



# 引言

本文主要梳理了ClickHouse分布式表，也就是是Distributed表引擎基本工作原理。主要内容有：

* 分布式表分片算法规则
* 分布式表写入基本流程
* 分布式表读出数据流程
* 非分布式表写入本地表



# 一、分布式表分片算法规则

使用分布式表时，数据应该落到哪个分片节点上呢？ClickHouse有一套自己的分片算法，下面从概念开始就一探究竟。

**分片键（sharding_key）：** 要求返回一个整数类型的取值，下面语法中sharding_key需整数类型

```
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
(
    name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1],
    name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2],
    ...
) ENGINE = Distributed(cluster, database, table[, sharding_key[, policy_name]])
[SETTINGS name=value, ...]
```

**权重（Weight）** ：  ClickHouse中一个节点一个分片，可以给分片配置权重，权重越大数据分配越多，默认权重为1

**槽（Slot）：** 槽的数量为集群中所有分片的权重之和

![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/%E5%88%86%E7%89%87%E7%AE%97%E6%B3%95.png)



小结：通过权重与槽联合使用的一种简单分片算法。



# 二、分布式表写入基本流程

在使用ClickHouse分布式表写入数据时，大体流程是这样的。

@1 数据先写入一个分片（例如：分片1）

@2 属于本分片的数据写入本地表，属于其他分片的数据先写入本分片的临时目录

​      例如：其他分片的数据先写入分片1的临时目录

@3 该分片与集群中其他分片建立连接

​       例如：分片1与分片2、分片1与分片3建立连接

@4 将写入本地临时文件的数据异步发送到其他分片

​       例如：分片1将临时目录数据发送到分片2与分片3

![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/%E5%88%86%E5%B8%83%E5%BC%8F%E8%A1%A8%E5%86%99%E5%85%A5%E6%B5%81%E7%A8%8B22.png)



小结：使用分布式表直接写入数据时，集群中各个节点彼此会建立连接，彼此之间传输数据；性能低于直接写入本地表。



# 三、分布式表读出数据流程

使用ClickHouse的分布式表查询，大体流程如下：

* 集群多副本时根据负载均衡选择一个副本，也就是说副本是可以承担查询功能的
* 将分布式查询语句转换为本地查询语句
* 将本地插叙语句发送到各个分片节点执行查询
* 再将返回的结果执行合并



![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/%E5%88%86%E5%B8%83%E5%BC%8F%E8%A1%A8%E6%9F%A5%E8%AF%A2%E6%B5%81%E7%A8%8B.png)



小结：分布式表的查询类似于分库分表中间件，逻辑也很类似。





# 四、非分布式表写入本地表

写入时通过分布式表，由于先写入本地临时目录，集群节点之间会有数据传输。那如果再写入时直接写入本地表，性能要高于通过分布式表。



![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/%E5%85%B6%E4%BB%96%E6%96%B9%E5%BC%8F%E5%86%99%E5%85%A5%E6%9C%AC%E5%9C%B0%E8%A1%A8.png)



负载均衡的方式：

* 分片节点前挂载SLB等方式负载均衡，注意带宽限制
* 客户端写入时轮训与各个分片建立连接，在客户端进行负载均衡选择分片















