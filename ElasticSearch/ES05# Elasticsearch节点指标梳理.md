---
title: ES05# Elasticsearch节点指标梳理
categories: Elasticsearch
tags: Elasticsearch
date: 2022-04-17 11:55:01
---



# 引言

节点状态GET _nodes/stats，简单的命令返回大量的指标信息，本文就一探究竟拨开主要指标的含义，文章主要内容有：

* 节点信息说明
* 操作指标说明
* 缓存&事务&恢复指标



# 一、节点信息说明

### 1.节点数量

```json
"_nodes" : {
    "total" : 33,
    "successful" : 33,
    "failed" : 0
  }
```

指标说明：

| 属性              | 说明               |
| ----------------- | ------------------ |
| _nodes.total      | 集群的节点数量     |
| _nodes.successful | 正确响应的节点数量 |
| _nodes.failed     | 失败响应的节点数量 |



### 2.IP&角色&属性

```json
"nodes" : {
    "fSoa6g9FQNWOD1upVGrJUg" : {
      "timestamp" : 1650288221571,
      "name" : "elastic-log-xxx-xxx-es-data-7",
      "transport_address" : "x.x.x.x:9300",
      "host" : "x.x.x.x",
      "ip" : "x.x.x.x:9300",
      "roles" : [
        "data_content",
        "data_hot",
        "ingest"
      ],
      "attributes" : {
        "k8s_node_name" : "cn-hangx.x.x.x.x",
        "xpack.installed" : "true",
        "transform.node" : "false"
      }
      //...
    }
  	//...
 }
```

指标说明：

| 属性              | 说明                 |
| ----------------- | -------------------- |
| timestamp         | 收集指标的时间戳     |
| name              | 节点名称             |
| transport_address | 集群内部通信地址端口 |
| host              | host地址             |
| IP                | IP+端口              |
| roles             | 该节点被赋予的角色   |
| attributes        | 节点属性信息         |



### 3.文档数量与存储

```json
"indices" : {
  "docs" : {
    "count" : 1096432845,
    "deleted" : 286918
  },
  "store" : {
    "size_in_bytes" : 543284041812,
    "reserved_in_bytes" : 0
  },
	// ...
}
```

指标说明：

| 属性                    | 说明                   |
| ----------------------- | ---------------------- |
| docs.count              | 该节点存储的文档数量   |
| docs.deleted            | 该节点删除的文档数量   |
| store.size_in_bytes     | 该节点分片存储大小     |
| store.reserved_in_bytes | 预测恢复快照将增长多少 |

##### 

# 二、操作指标说明

### 1. 索引操作

```json
"indexing" : {
  "index_total" : 22717470659,
  "index_time_in_millis" : 8039662582,
  "index_current" : 11,
  "index_failed" : 0,
  "delete_total" : 390,
  "delete_time_in_millis" : 587,
  "delete_current" : 0,
  "noop_update_total" : 0,
  "is_throttled" : false,
  "throttle_time_in_millis" : 0
}
```

指标说明：

| 属性                    | 说明                                 |
| ----------------------- | ------------------------------------ |
| index_total             | 该节点索引操作总次数                 |
| index_time_in_millis    | 该节点索引操作总的耗时               |
| index_current           | 该节点当前正在执行索引操作的个数     |
| index_failed            | 该节点索引操作执行失败的个数         |
| delete_total            | 该节点索引删除操作的总数             |
| delete_time_in_millis   | 该节点索引删除操作的总耗时           |
| delete_current          | 该节点当前正在执行索引删除操作的个数 |
| noop_update_total       | 该节点空操作（更新）的总数           |
| is_throttled            | 是否被限流                           |
| throttle_time_in_millis | 限流操作所耗用的时间                 |



### 2.Get操作指标

返回示例：

```json
"get" : {
  "total" : 217898,
  "time_in_millis" : 24594,
  "exists_total" : 211213,
  "exists_time_in_millis" : 24306,
  "missing_total" : 6685,
  "missing_time_in_millis" : 288,
  "current" : 0
}
```

指标说明：

| 属性                   | 说明                            |
| ---------------------- | ------------------------------- |
| total                  | 该节点Get操作总次数             |
| time_in_millis         | 该节点Get操作总的耗时           |
| exists_total           | 该节点Get操作成功总次数         |
| exists_time_in_millis  | 该节点Get操作成功总耗时         |
| missing_total          | 该节点Get操作失败总次数         |
| missing_time_in_millis | 该节点Get操作失败总耗时         |
| current                | 该节点当前正在执行Get操作的数量 |



### 3.Search操作指标

返回示例：

```json
"search" : {
  "open_contexts" : 0,
  "query_total" : 2810350,
  "query_time_in_millis" : 37625703,
  "query_current" : 0,
  "fetch_total" : 1386124,
  "fetch_time_in_millis" : 15092328,
  "fetch_current" : 0,
  "scroll_total" : 122754,
  "scroll_time_in_millis" : 1515856,
  "scroll_current" : 0,
  "suggest_total" : 0,
  "suggest_time_in_millis" : 0,
  "suggest_current" : 0
}
```

指标说明：

| 属性                   | 说明                              |
| ---------------------- | --------------------------------- |
| open_contexts          | 该节点打开查询上下文总的数量      |
| query_total            | 该节点Query操作总的数量           |
| query_time_in_millis   | 该节点Query操作总的耗时           |
| query_current          | 该节点当前正在运行的Query操作数量 |
| fetch_total            | 该节点fetch操作总的数量           |
| fetch_time_in_millis   | 该节点fetch操作总的耗时           |
| fetch_current          | 该节点当前运行fetch操作的数量     |
| scroll_total           | 该节点scroll操作总的数量          |
| scroll_time_in_millis  | 该节点scroll操作总的耗时          |
| scroll_current         | 该节点当前运行scroll操作的数量    |
| suggest_total          | 该节点suggest操作总的数量         |
| suggest_time_in_millis | 该节点suggest操作总的耗时         |
| suggest_current        | 该节点当前运行suggest操作的数量   |



### 4.Merges操作指标

返回示例：

```json
"merges" : {
  "current" : 8,
  "current_docs" : 17109224,
  "current_size_in_bytes" : 9829070126,
  "total" : 3074176,
  "total_time_in_millis" : 10028444483,
  "total_docs" : 96464444178,
  "total_size_in_bytes" : 47030059786323,
  "total_stopped_time_in_millis" : 11215,
  "total_throttled_time_in_millis" : 6133172861,
  "total_auto_throttle_in_bytes" : 72584133625
}
```

指标说明：

| 属性                           | 说明                                  |
| ------------------------------ | ------------------------------------- |
| current                        | 该节点正在运行merge操作的数量         |
| current_docs                   | 该节点正在运行merge文本的数量         |
| current_size_in_bytes          | 该节点正在运行merge文本占用的内存大小 |
| total                          | 该节点merge操作总的数量               |
| total_time_in_millis           | 该节点merge操作总的耗时               |
| total_docs                     | 该节点merge文档总的数量               |
| total_size_in_bytes            | 该节点merge文档总的大小               |
| total_stopped_time_in_millis   | 该节点merge操作停止总的时间           |
| total_throttled_time_in_millis | 该节点merge操作限流总的耗时           |
| total_auto_throttle_in_bytes   | 超过该阈值自动触发merge操作限流       |



### 5.refresh操作指标

返回示例：

```json
"refresh" : {
  "total" : 15285785,
  "total_time_in_millis" : 738659952,
  "external_total" : 15153381,
  "external_total_time_in_millis" : 758721356,
  "listeners" : 0
}
```

指标说明：

| 属性                          | 说明                        |
| ----------------------------- | --------------------------- |
| total                         | 该节点refresh操作总的数量   |
| total_time_in_millis          | 该节点refresh操作总的耗时   |
| external_total                | 该节额外refresh操作总的数量 |
| external_total_time_in_millis | 该节额外refresh操作总的耗时 |
| listeners                     | 该节refresh listeners的数量 |



### 6.flush操作指标

返回示例：

```json
"flush" : {
  "total" : 90832,
  "periodic" : 50676,
  "total_time_in_millis" : 71006569
}
```

指标说明：

| 属性                 | 说明                                |
| -------------------- | ----------------------------------- |
| total                | 该节点flush刷盘操作总的次数         |
| periodic             | 该节点周期性触发flush刷盘操作的次数 |
| total_time_in_millis | 该节点flush刷盘操作总的耗时         |



### 7.warmer操作指标

返回示例：

```json
"warmer" : {
  "current" : 0,
  "total" : 1186361,
  "total_time_in_millis" : 45855
}
```

指标说明：

| 属性                 | 说明                         |
| -------------------- | ---------------------------- |
| current              | 该节点正在运行预热索引的数量 |
| total                | 该节点总共预热索引的数量     |
| total_time_in_millis | 该节点总共预热索引的耗时     |



# 三、缓存&事务&恢复指标

### 1.query_cache指标

返回示例：

```json
"query_cache" : {
  "memory_size_in_bytes" : 11514288,
  "total_count" : 21172337,
  "hit_count" : 7241011,
  "miss_count" : 13931326,
  "cache_size" : 78,
  "cache_count" : 26881,
  "evictions" : 26803
}
```

指标说明：

| 属性                 | 说明                                |
| -------------------- | ----------------------------------- |
| memory_size_in_bytes | 查询缓存占用总的大小                |
| total_count          | 查询缓存总的次数（包括命中+未命中） |
| hit_count            | 查询缓存命中的次数                  |
| miss_count           | 查询缓存未命中的次数                |
| cache_size           | 当前查询缓存中文档的数量            |
| cache_count          | 查询缓存中总的文档的数量            |
| evictions            | 查询缓存中被驱逐的数量              |



### 2.translog指标

返回示例：

```json
"translog" : {
  "operations" : 22091013,
  "size_in_bytes" : 25272012418,
  "uncommitted_operations" : 22091013,
  "uncommitted_size_in_bytes" : 25272012418,
  "earliest_last_modified_age" : 0
}
```

指标说明：

| 属性                       | 说明                                  |
| -------------------------- | ------------------------------------- |
| operations                 | transaction log操作次数               |
| size_in_bytes              | transaction log的大小                 |
| uncommitted_operations     | 未提交transaction操作的数量           |
| uncommitted_size_in_bytes  | 未提交transaction日志的大小           |
| earliest_last_modified_age | transaction日志存的最久的日志条目时间 |



### 3.request_cache指标

返回示例：

```json
"request_cache" : {
  "memory_size_in_bytes" : 151103,
  "evictions" : 0,
  "hit_count" : 22922,
  "miss_count" : 42233
}
```

指标说明：

| 属性                 | 说明                 |
| -------------------- | -------------------- |
| memory_size_in_bytes | 请求缓存的大小       |
| evictions            | 请求缓存被驱逐的数量 |
| hit_count            | 请求缓存的命中数量   |
| miss_count           | 请求缓存的未命中数量 |



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

| 属性                    | 说明                     |
| ----------------------- | ------------------------ |
| current_as_source       | 源索引分片恢复操作的数量 |
| current_as_target       | 目标引分片恢复操作的数量 |
| throttle_time_in_millis | 恢复操作的延迟时长       |





备注：其他fielddata、completion、segments以及系统、jvm等指标在上一篇已梳理，不再重复。
