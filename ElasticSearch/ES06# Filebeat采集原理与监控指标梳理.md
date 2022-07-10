---
title: ES06# Filebeat采集原理与监控指标梳理
categories: Elasticsearch
tags: Elasticsearch
date: 2022-05-04 11:55:01
---



# 引言

当Filebeat作为日志采集的agent铺开时，对其自身agent的监控以确保稳定就尤为的重要，有几种方式监控agent运行。

* 第一种 filebeat自己将监控埋点上报
* 第二种 filebeat暴露埋点接口，另外一个agent定时采集后上报

第二种能够监测filebeat的进程状况，例如官方提供的Metricbeat，也可以自己实现agent上报监控指标。本文就其如何监控Filebeat以及指标含义进行梳理，主要内容有：

* 一、filebeat日志采集原理
* 二、filebeat暴露endpoint
* 三、beat监控指标
* 四、filebeat监控指标
* 五、libbeat监控指标
* 六、监控指标完整示例



# 一、filebeat日志采集原理



filebeat采集原理如下图所示，数据流从左到右流转。

![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/%E9%87%87%E9%9B%86%E5%8E%9F%E7%90%86.png)

组件描述：

Input：负责输入源，每启动一个文件会创建一个Harvester负责读取

Filebeat.Harvester：负责文件数据的读取

Filebeat.Registrar：负责记录文件以及对应的偏移量，记录在registry/filebeat/log.json中，格式如下所示，filebeat启动时会读取该文件

```
{"k":"filebeat::logs::native::4349261-16777229","v":{"id":"native::4349261-16777229","prev_id":"","source":"/Users/admin/logs/csp/sentinel-server.log","offset":1150,"type":"log","identifier_name":"native","timestamp":[2062583329080,1648206113],"ttl":-1,"FileStateOS":{"inode":4349261,"device":16777229}}}
```

Libbeat.Pipeline：负责管理Harvester的写入、缓存、输出等，queue事件队列（缓存or磁盘），acker输出后的确认回调



# 二、filebeat暴露endpoint

### 1.HTTP endpoint

在filebeat.yml文件中开启http.enabled，默认端口为5066，本文将其修改成了5067。

```
#Enable the HTTP endpoint to allow external collection of monitoring data
http.enabled: true
http.port: 5067
```



### 2.filebeat基本信息

请求命令

```
http://localhost:5067/?pretty
```

返回示例

```
{
  "beat": "filebeat",
  "hostname": "M-C02GL1NTQ05P",
  "name": "M-C02GL1NTQ05P",
  "uuid": "d9823622-46a6-4f4d-8dda-f2efeee82e35",
  "version": "7.15.2"
}
```

指标说明

| 指标     | 说明                       |
| -------- | -------------------------- |
| beat     | beat的类型，本文为filebeat |
| hostname | 主机名称                   |
| uuid     | filebeat的uuid             |
| version  | filebeat的版本             |



另外下面命令返回了filebeat的众多指标信息，下文中根据不同的类型分拆梳理走查。

```
http://localhost:5067/stats?pretty
```



# 三、beat监控指标

beat为通用模块，filebeat继承该模块，侧重于整体概览。主要包括：基本信息、CPU运行状态信息、缓存指标信息、运行时指标。

### 1.基本信息

返回示例

```json
"info": {
      "ephemeral_id": "7ab1d8ba-098d-46a8-a116-bfc350493f40",
      "uptime": {
        "ms": 623789
      },
      "version": "7.15.2"
 }
```

指标说明

| 指标         | 说明                                  |
| ------------ | ------------------------------------- |
| ephemeral_id | 临时ID用于标识此agent，在重启后会变化 |
| uptime       | agent的运行时间                       |
| version      | agent的版本信息                       |



### 2.CPU运行状态

返回示例

```json
"cpu": {
      "system": {
        "ticks": 17,
        "time": {
          "ms": 737
        }
      },
      "total": {
        "ticks": 34,
        "time": {
          "ms": 1464
        },
        "value": 34
      },
      "user": {
        "ticks": 17,
        "time": {
          "ms": 727
        }
      }
 }
```

指标说明

| 指标         | 说明                                  |
| ------------ | ------------------------------------- |
| system.ticks | 运行CPU处于系统状态的时间             |
| user.ticks   | 运行CPU处于用户状态的时间             |
| total.ticks  | 运行CPU处于系统状态和用户状态总的时间 |



### 3.缓存指标

返回示例

```json
"memstats": {
  "gc_next": 23418608,
  "memory_alloc": 16693968,
  "memory_sys": 76366856,
  "memory_total": 181068824,
  "rss": 46481408
}
```

指标说明

| 指标         | 说明                                                    |
| ------------ | ------------------------------------------------------- |
| gc_next      | 下一次GC目标堆的大小                                    |
| memory_alloc | Go语言堆空间分配的字节数                                |
| memory_sys   | 服务当前使用系统的内存大小                              |
| memory_total | 服务运行至今总共分配的堆内存大小（只递增）              |
| rss          | Resident Set Size（常驻内存大小）表示进程使用了多少内存 |



### 4.运行时指标

返回示例

```json
"runtime": {
  "goroutines": 31
}
```

指标说明

| 指标       | 说明                       |
| ---------- | -------------------------- |
| goroutines | 运行时调度管理“线程池”大小 |





# 四、filebeat监控指标

### 1.events指标

返回示例

```json
"events": {
      "active": 0,
      "added": 3,
      "done": 3
    }
```

指标说明

| 指标   | 说明               |
| ------ | ------------------ |
| active | 正在活动事件的数量 |
| added  | 已添加事件的数量   |
| done   | 已完成事件的数量   |

备注：通过events了解filebeat的吞吐情况。



### 2.harvester指标

返回示例

```json
"harvester": {
      "closed": 0,
      "open_files": 0,
      "running": 0,
      "skipped": 0,
      "started": 0
    }
```

指标说明

| 指标       | 说明                  |
| ---------- | --------------------- |
| closed     | harvester已关闭的数量 |
| open_files | 已打开的文件数量      |
| running    | 正在harvester的数量   |
| skipped    | harvester已忽略的数量 |
| started    | harvester已启动的数量 |

备注：每个文件都会通过一个harvester读取，通过harvester监控读取文件数量情况。



### 3.input指标

返回示例

```json
"input": {
      "log": {
        "files": {
          "renamed": 0,
          "truncated": 0
        }
      },
      "netflow": {
        "flows": 0,
        "packets": {
          "dropped": 0,
          "received": 0
        }
      }
```

指标说明

| 指标             | 说明                      |
| ---------------- | ------------------------- |
| files.renamed    | 被改名文件的数量          |
| files.truncated  | 被截断文件的数量          |
| netflow.flows    | IP网络流量统计            |
| packets.dropped  | 丢弃的分组（packets）数量 |
| packets.received | 接受的分组（packets）数量 |

备注：每个文件都会通过一个harvester读取，通过harvester监控读取文件数量情况，netflow网络数据包分析工具。



# 五、libbeat监控指标

libbeat为公共模块，filebeat继承了该模块。可以output的速率、pipeline中队列积压以及registrar记录位移情况。

### 1.config指标

返回示例

```json
"config": {
      "module": {
        "running": 0,
        "starts": 0,
        "stops": 0
      },
      "reloads": 1,
      "scans": 62
    }
```

指标说明

| 指标           | 说明               |
| -------------- | ------------------ |
| module.running | 正在运行组件的数量 |
| module.starts  | 已启动组件的数量   |
| module.stops   | 已停止组件的数量   |
| reloads        | 加载配置文件的数量 |
| scans          | 扫描配置文件的数量 |



### 2.output指标

返回示例

```json
"output": {
      "events": {
        "acked": 0,
        "active": 0,
        "batches": 0,
        "dropped": 0,
        "duplicates": 0,
        "failed": 0,
        "toomany": 0,
        "total": 0
      },
      "read": {
        "bytes": 0,
        "errors": 0
      },
      "type": "elasticsearch",
      "write": {
        "bytes": 0,
        "errors": 0
      }
    }
```

指标说明

| 指标           | 说明         |
| -------------- | ------------ |
| events.acked   | 确认事件数量 |
| events.active  | 活动事件数量 |
| events.dropped | 丢弃事件数量 |
| events.failed  | 失败事件数量 |
| read.bytes     | 读入的字节数 |
| write.bytes    | 写出的字节数 |



### 3.pipeline指标

返回示例

```
"pipeline": {
      "clients": 1,
      "events": {
        "active": 0,
        "dropped": 0,
        "failed": 0,
        "filtered": 3,
        "published": 0,
        "retry": 0,
        "total": 3
      },
      "queue": {
        "acked": 0,
        "max_events": 4096
      }
    }
```

指标说明

| 指标             | 说明                        |
| ---------------- | --------------------------- |
| clients          | 连接到pipeline的client数量  |
| queue.acked      | queue中已被output确认的数量 |
| queue.max_events | queue中存储事件的最大数量   |

备注：Filebeat使用内部queue存储publishing前的事件，这些事件会被outputs消费。



### 4.registrar指标

返回示例

```json
"registrar": {
    "states": {
      "cleanup": 0,
      "current": 3,
      "update": 3
    },
    "writes": {
      "fail": 0,
      "success": 3,
      "total": 3
    }
  }
```

指标说明

| 指标           | 说明                       |
| -------------- | -------------------------- |
| states.cleanup | 被清理registrar的数量      |
| states.current | 正在运行registrar的数量    |
| states.update  | 已更新registrar的数量      |
| writes.fail    | 写入registry文件失败的数量 |
| writes.success | 写入registry文件成功的数量 |

备注：registrar会记录日志文件采集的位移。



### 5.CPU负载指标

返回示例

```
"system": {
    "cpu": {
      "cores": 8
    },
    "load": {
      "1": 2.749,
      "15": 2.5459,
      "5": 2.5659,
      "norm": {
        "1": 0.3436,
        "15": 0.3182,
        "5": 0.3207
      }
    }
```

指标说明

| 指标  | 说明    |
| ----- | ------- |
| cores | CPU核数 |
| load  | CPU负载 |



# 六、监控指标完整示例

**指标端点** 

```
http://localhost:5067/stats?pretty
```

**返回示例** 

```
{
  "beat": {
    "cpu": {
      "system": {
        "ticks": 17,
        "time": {
          "ms": 737
        }
      },
      "total": {
        "ticks": 34,
        "time": {
          "ms": 1464
        },
        "value": 34
      },
      "user": {
        "ticks": 17,
        "time": {
          "ms": 727
        }
      }
    },
    "info": {
      "ephemeral_id": "7ab1d8ba-098d-46a8-a116-bfc350493f40",
      "uptime": {
        "ms": 623789
      },
      "version": "7.15.2"
    },
    "memstats": {
      "gc_next": 23418608,
      "memory_alloc": 16693968,
      "memory_sys": 76366856,
      "memory_total": 181068824,
      "rss": 46481408
    },
    "runtime": {
      "goroutines": 31
    }
  },
  "filebeat": {
    "events": {
      "active": 0,
      "added": 3,
      "done": 3
    },
    "harvester": {
      "closed": 0,
      "open_files": 0,
      "running": 0,
      "skipped": 0,
      "started": 0
    },
    "input": {
      "log": {
        "files": {
          "renamed": 0,
          "truncated": 0
        }
      },
      "netflow": {
        "flows": 0,
        "packets": {
          "dropped": 0,
          "received": 0
        }
      }
    }
  },
  "libbeat": {
    "config": {
      "module": {
        "running": 0,
        "starts": 0,
        "stops": 0
      },
      "reloads": 1,
      "scans": 62
    },
    "output": {
      "events": {
        "acked": 0,
        "active": 0,
        "batches": 0,
        "dropped": 0,
        "duplicates": 0,
        "failed": 0,
        "toomany": 0,
        "total": 0
      },
      "read": {
        "bytes": 0,
        "errors": 0
      },
      "type": "elasticsearch",
      "write": {
        "bytes": 0,
        "errors": 0
      }
    },
    "pipeline": {
      "clients": 1,
      "events": {
        "active": 0,
        "dropped": 0,
        "failed": 0,
        "filtered": 3,
        "published": 0,
        "retry": 0,
        "total": 3
      },
      "queue": {
        "acked": 0,
        "max_events": 4096
      }
    }
  },
  "registrar": {
    "states": {
      "cleanup": 0,
      "current": 3,
      "update": 3
    },
    "writes": {
      "fail": 0,
      "success": 3,
      "total": 3
    }
  },
  "system": {
    "cpu": {
      "cores": 8
    },
    "load": {
      "1": 2.749,
      "15": 2.5459,
      "5": 2.5659,
      "norm": {
        "1": 0.3436,
        "15": 0.3182,
        "5": 0.3207
      }
    }
  }
}
```



备注：官方参考文档

```

https://www.elastic.co/guide/en/beats/filebeat/current/http-endpoint.html

# Monitor Filebeat
https://www.elastic.co/guide/en/beats/filebeat/current/monitoring.html#monitoring

# Filebeat internal collection
https://www.elastic.co/guide/en/beats/filebeat/current/configuration-monitor.html

# Metricbeat collection
https://www.elastic.co/guide/en/beats/filebeat/current/monitoring-metricbeat-collection.html
```

