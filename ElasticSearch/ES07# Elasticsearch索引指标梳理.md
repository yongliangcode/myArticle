---
title: ES07# Elasticsearch索引指标梳理
categories: Elasticsearch
tags: Elasticsearch
date: 2022-05-10 11:55:01
---



# 引言

前面梳理了集群和节点的指标，索引指标也很重要。含义说明与前面有重复，只不过是在索引级别。还是撸一遍，索引状态命令GET my-index/_stats。文章主要内容有：

* 一、索引概览统计指标
* 二、索引具体操作指标
* 三、索引缓存类指标



# 一、索引概览统计指标

### 1.索引分片统计

返回示例：

```json
"_shards" : {
    "total" : 40,
    "successful" : 40,
    "failed" : 0
}
```

指标说明：

| 属性               | 说明               |
| ------------------ | ------------------ |
| _shards.total      | 索引总的分片数量   |
| _shards.successful | 成功响应的分片数量 |
| _shards.failed     | 失败响应的分片数量 |

### 2.索引文档统计

返回示例：

```json
"docs" : {
  "count" : 3428951086,
  "deleted" : 0
}
```

指标说明：

| 属性         | 说明                 |
| ------------ | -------------------- |
| docs.count   | 索引总的文档数量     |
| docs.deleted | 索引被删除的文档数量 |

### 3.索引存储统计

返回示例：

```json
"store" : {
  "size_in_bytes" : 848984127692,
  "reserved_in_bytes" : 0
}
```

指标说明：

| 属性                    | 说明                   |
| ----------------------- | ---------------------- |
| store.size_in_bytes     | 索引总的存储大小       |
| store.reserved_in_bytes | 预计快照恢复增长的大小 |

### 4.索引操作统计

返回示例：

```json
"indexing" : {
        "index_total" : 3431981662,
        "index_time_in_millis" : 897140520,
        "index_current" : 22,
        "index_failed" : 0,
        "delete_total" : 0,
        "delete_time_in_millis" : 0,
        "delete_current" : 0,
        "noop_update_total" : 0,
        "is_throttled" : false,
        "throttle_time_in_millis" : 0
 }
```

指标说明：

| 属性                    | 说明                   |
| ----------------------- | ---------------------- |
| index_total             | 操作索引总的次数       |
| index_time_in_millis    | 操作索引总的耗时       |
| index_current           | 当前正在操作索引的次数 |
| index_failed            | 操作索引失败的次数     |
| delete_total            | 索引删除操作总的次数   |
| delete_time_in_millis   | 索引删除操作总的耗时   |
| delete_current          | 当前正在删除索引的次数 |
| noop_update_total       | 索引空更新的次数       |
| is_throttled            | 是否被限流             |
| throttle_time_in_millis | 限流操作所耗用的时间   |



# 二、索引具体操作指标

### 1.Get操作指标

返回示例：

```json
"get" : {
    "total" : 0,
    "time_in_millis" : 0,
    "exists_total" : 0,
    "exists_time_in_millis" : 0,
    "missing_total" : 0,
    "missing_time_in_millis" : 0,
    "current" : 0
}
```

指标说明：

| 属性                   | 说明                        |
| ---------------------- | --------------------------- |
| total                  | 该索引get操作总的次数       |
| time_in_millis         | 该索引get操作总的耗时       |
| exists_total           | 该索引get操作成功总的次数   |
| exists_time_in_millis  | 该索引get操作成功总耗时     |
| missing_total          | 该索引get操作失败总次数     |
| missing_time_in_millis | 该索引get操作失败总耗时     |
| current                | 该索引正在执行get操作的数量 |

### 2.Search操作指标

返回示例：

```json
"search" : {
    "open_contexts" : 0,
    "query_total" : 0,
    "query_time_in_millis" : 0,
    "query_current" : 0,
    "fetch_total" : 0,
    "fetch_time_in_millis" : 0,
    "fetch_current" : 0,
    "scroll_total" : 0,
    "scroll_time_in_millis" : 0,
    "scroll_current" : 0,
    "suggest_total" : 0,
    "suggest_time_in_millis" : 0,
    "suggest_current" : 0
}
```

指标说明：

| 属性                   | 说明                              |
| ---------------------- | --------------------------------- |
| open_contexts          | 该索引打开查询上下文总的数量      |
| query_total            | 该索引Query操作总的数量           |
| query_time_in_millis   | 该索引Query操作总的耗时           |
| query_current          | 该索引当前正在运行的Query操作数量 |
| fetch_total            | 该索引fetch操作总的数量           |
| fetch_time_in_millis   | 该索引fetch操作总的耗时           |
| fetch_current          | 该索引当前运行fetch操作的数量     |
| scroll_total           | 该索引scroll操作总的数量          |
| scroll_time_in_millis  | 该索引scroll操作总的耗时          |
| scroll_current         | 该索引当前运行scroll操作的数量    |
| suggest_total          | 该索引suggest操作总的数量         |
| suggest_time_in_millis | 该索引suggest操作总的耗时         |
| suggest_current        | 该索引当前运行suggest操作的数量   |

### 3.Merges操作指标

返回示例：

```json
"merges" : {
          "current" : 36,
          "current_docs" : 251604906,
          "current_size_in_bytes" : 63670970546,
          "total" : 107906,
          "total_time_in_millis" : 696446968,
          "total_docs" : 12497658332,
          "total_size_in_bytes" : 3109854688497,
          "total_stopped_time_in_millis" : 0,
          "total_throttled_time_in_millis" : 515392248,
          "total_auto_throttle_in_bytes" : 209715200
        }
```

指标说明：

| 属性                           | 说明                                  |
| ------------------------------ | ------------------------------------- |
| current                        | 该索引正在运行merge操作的数量         |
| current_docs                   | 该索引正在运行merge文本的数量         |
| current_size_in_bytes          | 该索引正在运行merge文本占用的内存大小 |
| total                          | 该索引merge操作总的数量               |
| total_time_in_millis           | 该索引merge操作总的耗时               |
| total_docs                     | 该索引merge文档总的数量               |
| total_size_in_bytes            | 该索引merge文档总的大小               |
| total_stopped_time_in_millis   | 该索引merge操作停止总的时间           |
| total_throttled_time_in_millis | 该索引merge操作限流总的耗时           |
| total_auto_throttle_in_bytes   | 超过该阈值自动触发merge操作限流       |

### 4.refresh操作指标

返回示例：

```json
"refresh" : {
          "total" : 159194,
          "total_time_in_millis" : 22205194,
          "external_total" : 154671,
          "external_total_time_in_millis" : 22427426,
          "listeners" : 0
        }
```

指标说明：

| 属性                          | 说明                          |
| ----------------------------- | ----------------------------- |
| total                         | 该索引refresh操作总的数量     |
| total_time_in_millis          | 该索引refresh操作总的耗时     |
| external_total                | 该索引额外refresh操作总的数量 |
| external_total_time_in_millis | 该索引额外refresh操作总的耗时 |
| listeners                     | 该索引refresh listeners的数量 |

### 5.flush操作指标

返回示例：

```json
"flush" : {
  "total" : 4508,
  "periodic" : 4468,
  "total_time_in_millis" : 3194273
}
```

指标说明：

| 属性                 | 说明                                |
| -------------------- | ----------------------------------- |
| total                | 该索引flush刷盘操作总的次数         |
| periodic             | 该索引周期性触发flush刷盘操作的次数 |
| total_time_in_millis | 该索引flush刷盘操作总的耗时         |

### 6.warmer操作指标

返回示例：

```json
"warmer" : {
  "current" : 0,
  "total" : 154631,
  "total_time_in_millis" : 1910
}
```

指标说明：

| 属性                 | 说明                         |
| -------------------- | ---------------------------- |
| current              | 该索引正在运行预热索引的数量 |
| total                | 该索引总共预热索引的数量     |
| total_time_in_millis | 该索引总共预热索引的耗时     |



# 三、索引缓存类指标

### 1.query_cache指标

返回示例：

```json
"query_cache" : {
        "memory_size_in_bytes" : 0,
        "total_count" : 0,
        "hit_count" : 0,
        "miss_count" : 0,
        "cache_size" : 0,
        "cache_count" : 0,
        "evictions" : 0
      }
```

指标说明：

| 属性                 | 说明                                      |
| -------------------- | ----------------------------------------- |
| memory_size_in_bytes | 该索引查询缓存占用总的大小                |
| total_count          | 该索引查询缓存总的次数（包括命中+未命中） |
| hit_count            | 该索引查询缓存命中的次数                  |
| miss_count           | 该索引查询缓存未命中的次数                |
| cache_size           | 该索引当前查询缓存中文档的数量            |
| cache_count          | 该索引查询缓存中总的文档的数量            |
| evictions            | 该索引查询缓存中被驱逐的数量              |

### 2.translog指标

返回示例：

```json
"translog" : {
          "operations" : 19562907,
          "size_in_bytes" : 13294819243,
          "uncommitted_operations" : 19562907,
          "uncommitted_size_in_bytes" : 13294819243,
          "earliest_last_modified_age" : 0
        }
```

指标说明：

| 属性                       | 说明                                        |
| -------------------------- | ------------------------------------------- |
| operations                 | 该索引transaction log操作次数               |
| size_in_bytes              | 该索引transaction log的大小                 |
| uncommitted_operations     | 该索引未提交transaction操作的数量           |
| uncommitted_size_in_bytes  | 该索引未提交transaction日志的大小           |
| earliest_last_modified_age | 该索引transaction日志存的最久的日志条目时间 |

### 3.request_cache指标

返回示例：

```json
"request_cache" : {
          "memory_size_in_bytes" : 0,
          "evictions" : 0,
          "hit_count" : 0,
          "miss_count" : 0
        }
```

指标说明：

| 属性                 | 说明                       |
| -------------------- | -------------------------- |
| memory_size_in_bytes | 该索引请求缓存的大小       |
| evictions            | 该索引请求缓存被驱逐的数量 |
| hit_count            | 该索引请求缓存的命中数量   |
| miss_count           | 该索引请求缓存的未命中数量 |

### 4.recovery指标

返回示例：

```json
"recovery" : {
  "current_as_source" : 0,
  "current_as_target" : 0,
  "throttle_time_in_millis" : 272139765
}
```

指标说明：

| 属性                    | 说明                       |
| ----------------------- | -------------------------- |
| current_as_source       | 源索引分片恢复操作的数量   |
| current_as_target       | 目标索引分片恢复操作的数量 |
| throttle_time_in_millis | 恢复操作的延迟时长         |

### 5.索引segments统计指标

返回示例：

```json
"segments" : {
          "count" : 1857,
          "memory_in_bytes" : 9678652,
          "terms_memory_in_bytes" : 7318248,
          "stored_fields_memory_in_bytes" : 1624176,
          "term_vectors_memory_in_bytes" : 0,
          "norms_memory_in_bytes" : 475008,
          "points_memory_in_bytes" : 0,
          "doc_values_memory_in_bytes" : 261220,
          "index_writer_memory_in_bytes" : 557751636,
          "version_map_memory_in_bytes" : 114448838,
          "fixed_bit_set_memory_in_bytes" : 0,
          "max_unsafe_auto_id_timestamp" : -1,
          "file_sizes" : { }
        }
```

指标说明：

| 属性                          | 说明                                                         |
| ----------------------------- | ------------------------------------------------------------ |
| count                         | 该索引segments的数量总数                                     |
| memory_in_bytes               | 该索引segments使用的缓存总和                                 |
| terms_memory_in_bytes         | 该索引terms query使用的缓存大小                              |
| stored_fields_memory_in_bytes | 该索引fields使用缓存大小                                     |
| term_vectors_memory_in_bytes  | 该索引Term Vectors（词条向量）使用缓存大小                   |
| norms_memory_in_bytes         | 该索引norms（标准信息）使用的缓存大小                        |
| points_memory_in_bytes        | 该索引points使用的缓存大小                                   |
| doc_values_memory_in_bytes    | 该索引doc values占用缓存大小                                 |
| index_writer_memory_in_bytes  | 该索引index writer占用缓存大小                               |
| version_map_memory_in_bytes   | 该索引version maps（描述document、fields包含的内容）占用的缓存大小 |
| fixed_bit_set_memory_in_bytes | 该索引BitSet（带标状态的数组）占用缓存的大小                 |
| max_unsafe_auto_id_timestamp  | 该索引documents自动生成IDs最新时间戳                         |

### 6.列数据缓存指标

列数据缓存主要用于对字段进行排序以及计算的聚合，将字段加载到缓存方便快速访问，通过参数indices.fielddata.cache.size控制。

返回示例：

```json
"fielddata" : {
          "memory_size_in_bytes" : 0,
          "evictions" : 0
        }
```

指标说明：

| 属性                 | 说明                                                         |
| :------------------- | ------------------------------------------------------------ |
| memory_size_in_bytes | 该索引列数据缓存总大小                                       |
| evictions            | 该索引驱逐缓存的大小，当超过堆内存阈值为了安全保护时会被驱逐，查询抛出Data too large异常 |

### 7.complete缓存指标

Linux内核中用于唤醒等待队列中睡眠线程，等待队列占用的缓存大小。

返回示例：

```json
"completion" : {
   "size_in_bytes" : 0
}
```

指标说明：

| 属性          | 说明                       |
| ------------- | -------------------------- |
| size_in_bytes | 该索引complete缓存使用大小 |

备注：官方文档说明：

```
https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-stats.html
```









