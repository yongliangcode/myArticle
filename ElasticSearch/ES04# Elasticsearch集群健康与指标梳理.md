---
title: ES04# Elasticsearch集群健康与指标梳理
categories: Elasticsearch
tags: Elasticsearch
date: 2022-04-17 11:55:01
---



# 引言



对于Elasticsearch运维管理员来讲集群平稳运行非常重要，Elasticsearch提供了health命令和stats统计指标来说明集群是否正常。这两个命令返回大量的指标信息，本文就一探究竟拨开主要指标的含义，文章主要内容有：

* 集群健康状况说明

* 集群统计指标说明

* 文章小结

  



# 一、集群健康状况说明

### 1.查询命令

通过「_cluster/health」命令能快速了解集群、索引、分片的健康状况，以及这些不健康大体是怎么引起的。

```
GET _cluster/health
```



### 2.查询参数说明

| 参数                            | 说明                                                         |
| ------------------------------- | ------------------------------------------------------------ |
| level                           | 指定查询的层级，可选`cluster`, `indices` 和 `shards` ，默认cluster |
| local                           | 是否从本地node查询，默认false从master节点查询                |
| master_timeout                  | 与master node建立连接超时时间，默认30秒                      |
| timeout                         | response的超时时间，默认30秒                                 |
| wait_for_active_shards          | 等待活动分片的数量，默认为0                                  |
| wait_for_events                 | 等待指定队列的优先级处理完成后返回，可选`immediate`, `urgent`, `high`, `normal`, `low`, `languid` |
| wait_for_no_initializing_shards | 是否等待没有initializations的分片时返回，默认false表示不等待 |
| wait_for_no_relocating_shards   | 是否等待不存在relocating的分片时返回，默认false表示不等待    |
| wait_for_nodes                  | 指定可用的节点数N，（>=N, <=N, >N 以及 <N等）                |
| wait_for_status                 | 等待集群变成指定状态后返回， `green`, `yellow` , `red`，默认不等待任何状态 |



### 3.查询示例

**输入命令** 

```
GET /_cluster/health
```

**输出结果** 

```json
{
  "cluster_name" : "elastic-log-xxx",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 24,
  "number_of_data_nodes" : 21,
  "active_primary_shards" : 27777,
  "active_shards" : 27804,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
```



### 4.返回参数说明

| 参数                             | 说明                                                         |
| -------------------------------- | ------------------------------------------------------------ |
| cluster_name                     | 集群名称                                                     |
| status                           | 集群状态<br />green：所有主分片和副本均已分配<br />yellow：所有主分片均已分配，至少一个副本分配缺失<br />red：至少一个主分片分配缺失 |
| timed_out                        | 监控查询是否超时，默认超时时间为30秒                         |
| number_of_nodes                  | 集群中所有节点数量                                           |
| number_of_data_nodes             | 集群中数据节点数量                                           |
| active_primary_shards            | 集群中活动的主分片数                                         |
| active_shards                    | 集群中活动的总分片数                                         |
| relocating_shards                | 正在从一个节点迁往其他节点总的分片数                         |
| initializing_shards              | 集群中处在initializing状态分片的数量，索引刚创建时分片的状态 |
| unassigned_shards                | 集群中处在unassigned状态分片的数量，表示集群不健康，有分片未被分配 |
| delayed_unassigned_shards        | 集群中因分片超时的分片数                                     |
| number_of_pending_tasks          | 集群中处于等待状态未被执行任务的数量                         |
| number_of_in_flight_fetch        | 集群中正在运行状态任务的数量                                 |
| task_max_waiting_in_queue_millis | 任务在队列中等待的最长时间                                   |
| active_shards_percent_as_number  | 集群中活动分片的占比                                         |



### 5.其他示例

```
GET /_cluster/health?level=indices&timeout=50s
```

```
GET /_cluster/health?level=shards&timeout=50s
```



# 二、集群统计指标说明

### 1.查询命令

该命令返回方方面面的指标，涵盖集群、节点、索引、系统、Jvm

```
GET _cluster/stats
```



### 2.节点指标 

stats命令返回的节点指标格式：

```
"_nodes" : {
    "total" : 33,
    "successful" : 33,
    "failed" : 0
}
```

下面是返回的指标说明：

| 属性       | 说明                       |
| :--------- | -------------------------- |
| _nodes     | 集群中节点数量             |
| total      | 本次查询选中的总节点数     |
| successful | 本次请求中成功响应的节点数 |
| failed     | 本次请求中失败响应的节点数 |



### 3.集群指标 

stats命令返回的集群指标格式：

```
{
  "cluster_name" : "elastic-log-trade2-prd",
  "cluster_uuid" : "iLSahkk-TaiP9g-FQKWsDQ",
  "timestamp" : 1650100202175,
  "status" : "green"
}
```

下面是返回的指标说明：

| 属性         | 说明                     |
| :----------- | ------------------------ |
| cluster_name | 集群名称                 |
| cluster_uuid | 集群唯一标识             |
| timestamp    | 集群最新刷新指标的时间戳 |
| status       | 集群健康状态             |



### 4.索引指标

stats命令返回的索引指标格式：

```json
"indices" : {
    "count" : 736,
    "shards" : {
      "total" : 21324,
      "primaries" : 21297,
      "replication" : 0.0012677841949570363,
      "index" : {
        "shards" : {
          "min" : 2,
          "max" : 30,
          "avg" : 28.972826086956523
        },
        "primaries" : {
          "min" : 1,
          "max" : 30,
          "avg" : 28.936141304347824
        },
        "replication" : {
          "min" : 0.0,
          "max" : 1.0,
          "avg" : 0.036684782608695655
        }
      }
    }
```

下面是返回的指标说明：

| 属性               | 说明                                           |
| :----------------- | ---------------------------------------------- |
| indices.count      | 总的索引数量                                   |
| shards.total       | 总的分片数                                     |
| shards.primaries   | 总的主分片数                                   |
| shards.replication | 副本分片数与主分片数的比率                     |
| index.shards.min   | 一个索引允许的最小分片数量                     |
| index.shards.max   | 一个索引允许的最大分片数量                     |
| index.shards.avg   | 索引的平均分片数量                             |
| primaries.min      | 一个索引允许的最小主分片数                     |
| primaries.max      | 一个索引允许的最大主分片数                     |
| primaries.avg      | 索引的平均主分片数量                           |
| replication.min    | 一个索引允许的最小副本因子，即最小允许几个副本 |
| replication.max    | 一个索引允许的最大副本因子，即最大允许几个副本 |
| replication.avg    | 索引的平均副本因子，即平均有几个副本           |



### 5.文档指标

stats命令返回的文档指标格式：

```json
"docs" : {
  "count" : 133086059854,
  "deleted" : 628015654
}
```

下面是返回的指标说明：

| 属性    | 说明                     |
| :------ | ------------------------ |
| count   | 主分片中未删除的文档数量 |
| deleted | 主分片中已删除的文档数量 |



### 6.存储指标

stats命令返回的存储指标格式：

```json
"store" : {
  "size_in_bytes" : 63035363353788,
  "reserved_in_bytes" : 0
}
```

下面是返回的指标说明：

| 属性              | 说明                          |
| :---------------- | ----------------------------- |
| size_in_bytes     | 分片占用总大小，示例中约为57T |
| reserved_in_bytes | 预测恢复快照将增长多少        |



### 7.列数据缓存指标

列数据缓存主要用于对字段进行排序以及计算的聚合，将字段加载到缓存方便快速访问，通过参数indices.fielddata.cache.size控制。

stats命令返回的列数据缓存指标格式：

```json
"fielddata" : {
  "memory_size_in_bytes" : 280480,
  "evictions" : 0
}
```

下面是返回的指标说明：

| 属性                 | 说明                                                         |
| :------------------- | ------------------------------------------------------------ |
| memory_size_in_bytes | 列数据缓存总大小                                             |
| evictions            | 驱逐缓存的大小，当超过堆内存阈值为了安全保护时会被驱逐，查询抛出Data too large异常 |



### 8.查询缓存指标

查询缓存用于缓存查询结果，默认为堆内存的10%，可以通过参数indices.queries.cache.size。调优该参数可以提高命中率，提高查询性能。

stats命令返回的查询缓存指标格式：

```json
 "query_cache" : {
   "memory_size_in_bytes" : 3407568767,
   "total_count" : 24089971,
   "hit_count" : 5694545,
   "miss_count" : 18395426,
   "cache_size" : 33826,
   "cache_count" : 160975,
   "evictions" : 127149
 }
```

下面是返回的指标说明：

| 属性                 | 说明                                                         |
| :------------------- | ------------------------------------------------------------ |
| memory_size_in_bytes | 查询缓存总大小                                               |
| total_count          | 查询缓存中命中和未命中的总数                                 |
| hit_count            | 查询缓存中总的命中数量                                       |
| miss_count           | 查询缓存中总的未命中数量                                     |
| cache_size           | 查询缓存中当前总的条目总数                                   |
| cache_count          | 查询缓存中总的条目数包含被驱逐的，是cache_size与evictions之和 |
| evictions            | 查询缓存中被驱逐的条目总数                                   |



### 9.complete缓存指标

Linux内核中用于唤醒等待队列中睡眠线程，等待队列占用的缓存大小。

stats命令返回的complete缓存指标格式：

```json
"completion" : {
   "size_in_bytes" : 0
}
```

下面是返回的指标说明：

| 属性          | 说明                 |
| ------------- | -------------------- |
| size_in_bytes | complete缓存使用大小 |



### 10.segments指标

stats命令返回的segments缓存指标格式：

```json
"segments" : {
      "count" : 314594,
      "memory_in_bytes" : 1698599230,
      "terms_memory_in_bytes" : 1229694816,
      "stored_fields_memory_in_bytes" : 233711144,
      "term_vectors_memory_in_bytes" : 0,
      "norms_memory_in_bytes" : 79423936,
      "points_memory_in_bytes" : 0,
      "doc_values_memory_in_bytes" : 155769334,
      "index_writer_memory_in_bytes" : 3707801916,
      "version_map_memory_in_bytes" : 281886805,
      "fixed_bit_set_memory_in_bytes" : 69337608,
      "max_unsafe_auto_id_timestamp" : 1650067201523,
      "file_sizes" : { }
    }
```

下面是返回的指标说明：

| 属性                          | 说明                                                         |
| ----------------------------- | ------------------------------------------------------------ |
| count                         | segments的数量总数                                           |
| memory_in_bytes               | segments使用的缓存总和                                       |
| terms_memory_in_bytes         | terms query使用的缓存大小                                    |
| stored_fields_memory_in_bytes | fields使用缓存大小                                           |
| term_vectors_memory_in_bytes  | Term Vectors（词条向量）使用缓存大小                         |
| norms_memory_in_bytes         | norms（标准信息）使用的缓存大小                              |
| points_memory_in_bytes        | points使用的缓存大小                                         |
| doc_values_memory_in_bytes    | doc values占用缓存大小                                       |
| index_writer_memory_in_bytes  | index writer占用缓存大小                                     |
| version_map_memory_in_bytes   | version maps（描述document、fields包含的内容）占用的缓存大小 |
| fixed_bit_set_memory_in_bytes | BitSet（带标状态的数组）占用缓存的大小                       |
| max_unsafe_auto_id_timestamp  | documents自动生成IDs最新时间戳                               |



### 11.mappings指标

统计集群中使用的字段数据类型，以及使用该字段类型的索引数量

stats命令返回的mappings指标格式：

```json
"mappings" : {
  "field_types" : [
    {
      "name" : "binary",
      "count" : 15,
      "index_count" : 4
    },
    {
      "name" : "boolean",
      "count" : 100,
      "index_count" : 22
    }
    //...
}        
```

下面是返回的指标说明：

| 属性                    | 说明                         |
| ----------------------- | ---------------------------- |
| field_types.name        | 字段数据类型                 |
| field_types.count       | 该字段数据类型在集群中的数量 |
| field_types.index_count | 使用该字段类型的索引数量     |



### 12.Analyzer指标

Analyzer组件用于将文本解析成单词，即分词过程。

stats命令返回的analysis指标格式：

```json
"analysis" : {
      "char_filter_types" : [ ],
      "tokenizer_types" : [ ],
      "filter_types" : [
        {
          "name" : "pattern_capture",
          "count" : 1,
          "index_count" : 1
        }
      ],
      // ....
      ]
    }
  }
```

下面是返回的指标说明：

| 属性                    | 说明                                                         |
| ----------------------- | ------------------------------------------------------------ |
| filter_type.name        | 使用的token filter类型，示例为模式匹配词元过滤器（pattern_capture） |
| filter_type.count       | 使用该类型分析器的数量                                       |
| field_types.index_count | 使用该类型索引的数量                                         |



### 13.nodes指标

该指标包含集群中各个节点类型数量统计、使用的ES版本、操作系统处理器数量、操作系统名称、物理内存情况

stats命令返回的nodes指标格式：

```json
"nodes" : {
    "count" : {
      "total" : 33,
      "coordinating_only" : 0,
      "data" : 0,
      "data_cold" : 15,
      "data_content" : 15,
      "data_hot" : 15,
      "data_warm" : 0,
      "ingest" : 30,
      "master" : 3,
      "ml" : 3,
      "remote_cluster_client" : 3,
      "transform" : 0,
      "voting_only" : 0
    },
    "versions" : [
      "7.10.0"
    ],
    "os" : {
      "available_processors" : 780,
      "allocated_processors" : 780,
      "names" : [
        {
          "name" : "Linux",
          "count" : 33
        }
      ],
      "pretty_names" : [
        {
          "pretty_name" : "CentOS Linux 8 (Core)",
          "count" : 33
        }
      ],
      "mem" : {
        "total_in_bytes" : 3002182139904,
        "free_in_bytes" : 122349514752,
        "used_in_bytes" : 2879832625152,
        "free_percent" : 4,
        "used_percent" : 96
      }
    }
```

下面是返回的指标说明：

| 属性                          | 说明                           |
| ----------------------------- | ------------------------------ |
| nodes.count.total             | 总的节点数量                   |
| nodes.count.coordinating_only | 协作节点（coordinating）的数量 |
| nodes.count.data              | data节点的数量                 |
| nodes.count.data_cold         | data冷节点的数量               |
| nodes.count.data_hot          | data热节点的数量               |
| nodes.versions                | 使用的elasticsearch版本        |
| os.available_processors       | 可用的处理器核数               |
| os.allocated_processors       | 已分配的处理器核数             |
| os.names.name                 | 操作系统类型                   |
| os.pretty_names               | 操作系统名称                   |
| nodes.mem.total_in_bytes      | 总的物理内存                   |
| nodes.mem.free_in_bytes       | 空闲的物理内存                 |
| nodes.mem.used_in_bytes       | 已使用的物理内存               |
| nodes.mem.free_percent        | 空闲内存占比                   |
| nodes.mem.used_percent        | 已使用内存占比                 |



### 14.处理器指标

stats命令返回的处理器指标格式：

```json
"process" : {
      "cpu" : {
        "percent" : 186
      },
      "open_file_descriptors" : {
        "min" : 1120,
        "max" : 15304,
        "avg" : 10030
      }
    }
```

下面是返回的指标说明：

| 属性                  | 说明            |
| --------------------- | --------------- |
| cpu.percent           | cpu使用的百分比 |
| open_file_descriptors | 文件描述符指标  |



### 15.Jvm指标

stats命令返回的Jvm指标格式：

```json
 "jvm" : {
   "max_uptime_in_millis" : 1406090815,
   "versions" : [
     {
       "version" : "15.0.1",
       "vm_name" : "OpenJDK 64-Bit Server VM",
       "vm_version" : "15.0.1+9",
       "vm_vendor" : "AdoptOpenJDK",
       "bundled_jdk" : true,
       "using_bundled_jdk" : true,
       "count" : 33
     }
   ],
   "mem" : {
     "heap_used_in_bytes" : 592482887640,
     "heap_max_in_bytes" : 1629940088832
   },
   "threads" : 6632
 }
```

下面是返回的指标说明：

| 属性                       | 说明            |
| -------------------------- | --------------- |
| jvm.max_uptime_in_millis   | jvm启动了多久   |
| jvm.versions               | jvm版本信息     |
| jvm.mem.heap_used_in_bytes | 已使用的堆内存  |
| jvm.mem.heap_max_in_bytes  | 最大堆内存      |
| jvm.threads                | jvm运行的线程数 |



### 16.file stores指标

stats命令返回的file stores指标格式：

```json
"fs" : {
      "total_in_bytes" : 175453039177728,
      "free_in_bytes" : 111910350888960,
      "available_in_bytes" : 111909797236736
    }
```

下面是返回的指标说明：

| 属性                  | 说明                           |
| --------------------- | ------------------------------ |
| fs.total_in_bytes     | 集群中文件存储占用总的磁盘大小 |
| fs.free_in_bytes      | 未分配的磁盘空间大小           |
| fs.available_in_bytes | jvm可用的磁盘空间余量          |



# 三、文章小结

health命令可以监测集群、索引、分片三个维度的健康状况。而stats统计的指标就更多了，本文梳理了16个不同方面的指标。虽然有些指标字面含义能大体猜出来，然而还有一些指标从字面上并不清晰，对其梳理一遍很有必要，在遇到问题时方便快速定位。



备注：官方文档

```
https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-health.html

https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-stats.html
```







