

```
title: Mesh8# Envoy原理提点与常用命令
categories: Mesh
tags: Mesh
date: 2021-12-12 11:55:01
```





# 基本概念



Istio的核心组件，作为sideCar与应用部署在一个Pod中，作为代理流量的进出均需经过Envoy所在的容器，除了代理外还可根据规则进行流量治理、监控等功能。



**Upstream Host:** 上游主机，接受envoy的连接和请求并返回响应

**Downstream Host:** 下游主机，向envoy发起请求并接受响应

**Enovy Mesh:** 由一组Envoy组成的拓扑网络

**Listener: **  监听器负责监听数据端口，接受下游的连接和请求，下游主机通过Listener连接Envoy

**Cluster: ** 集群管理后端服务服务的连接池、服务的健康检查、服务熔断等

**Filter：** 支持多种过滤器Listener Filter、Network Filter、L7 Filter等，组成filter链条，执行不同的流量治理逻辑。



**协议支持：** 

L3/L4网络代理，支持TCP、HTTP代理和TLS认证

L7代理，支持Buffer、限流等高级功能

L7路由，支持通过路径、权限、请求内容、运行时间等参数重定向路由请求

在HTTP模式下支持HTTP1.1和HTTP/2，同时支持基于HTTP/2的gRPC



**线程模型** 

一个Envoy进程包括一个Server主线程和一个GuardDog守护线程

Server主线程：负责管理Access Log以及解析上游主机的DNS。Access Log根据配置信息访问来处理Enovy访问记录，DNS解析将统一配置的域名解析成IP并缓存在本地DNS缓存中。



一个Envoy进程可以配置多个Listener，推荐配置一个，每个Listener可创建若干线程默认为核数，每个线程对应一个Worker。

一旦某个客户端连接进入Envoy中的某个线程，则连接断开之前的逻辑都在该线程内处理。例如：处理Client请求对应的TCP filter，解析协议和重新编码，与上游主机建立连接并处理返回数据等。



**内存管理** 

内存管理分为变量管理和Buffer管理：

* 变量管理：C++运行过程中创建的实例
* Buffer管理：数据接收、编解码等过程中临时存储数据的Buffer，通过malloc分配
  

**流量控制** 

* 如果上游主机处理过慢会在buffer积压，通过设置上下水位的阈值来控制
* 通过Envoy设置全局连接数来限制



**主要模块** 

* Network模块 抽象Socket提供读写功能
* Network Filter模块 过滤数据流量Listener Filter、Read Filter、Write Filter
* L7 protocol模块包含HTTP、HTTP/2、gRPC
* L7 filters模块与HTTP相关的认证、限流、路由等
* Server Manager模块管理Worker管理、启动管理、配置管理和日志等
* L7 Connection Manager模块包括建立连接、复用连接等功能
* Cluster Manger模块集群管理模块包括hosts管理、负载均衡、健康检查等





# Enovy命令汇总

**1.help命令** 
说明：打印命令帮助信息

```
curl 127.0.0.1:15000/help
admin commands are:
  /: Admin home page
  /certs: print certs on machine
  /clusters: upstream cluster status
  /config_dump: dump current Envoy configs (experimental)
  /contention: dump current Envoy mutex contention stats (if enabled)
  /cpuprofiler: enable/disable the CPU profiler
  /drain_listeners: drain listeners
  /healthcheck/fail: cause the server to fail health checks
  /healthcheck/ok: cause the server to pass health checks
  /heapprofiler: enable/disable the heap profiler
  /help: print out list of admin commands
  /hot_restart_version: print the hot restart compatibility version
  /init_dump: dump current Envoy init manager information (experimental)
  /listeners: print listener info
  /logging: query/change logging levels
  /memory: print current allocation/heap usage
  /quitquitquit: exit the server
  /ready: print server state, return 200 if LIVE, otherwise return 503
  /reopen_logs: reopen access logs
  /reset_counters: reset all counters to zero
  /runtime: print runtime values
  /runtime_modify: modify runtime values
  /server_info: print server version/status information
  /stats: print server stats
  /stats/prometheus: print server stats in prometheus format
  /stats/recentlookups: Show recent stat-name lookups
  /stats/recentlookups/clear: clear list of stat-name lookups and counter
  /stats/recentlookups/disable: disable recording of reset stat-name lookup names
  /stats/recentlookups/enable: enable recording of reset stat-name lookup names
```



**2.certs命令** 

说明：打印证书地址、序列号和有效期

```json
curl 127.0.0.1:15000/certs

{
 "certificates": [
  {
   "ca_cert": [
    {
     "path": "\u003cinline\u003e",
     "serial_number": "ade6c290ffcee6ded23b26cb367b258c",
     "subject_alt_names": [],
     "days_until_expiration": "3648",
     "valid_from": "2021-11-15T08:34:55Z",
     "expiration_time": "2031-11-13T08:34:55Z"
    }
   ],
   // ...
 ]
}
```



**3.Clsuters命令** 

说明：打印所有服务发现的Cluster地址、请求统计、最大连接数、最大重试次数等

```json
curl 127.0.0.1:15000/clusters
outbound|50000||AppMeshClient.mesh::default_priority::max_connections::4294967295
outbound|50000||AppMeshClient.mesh::default_priority::max_pending_requests::4294967295
outbound|50000||AppMeshClient.mesh::default_priority::max_requests::4294967295
outbound|50000||AppMeshClient.mesh::default_priority::max_retries::4294967295
outbound|50000||AppMeshClient.mesh::high_priority::max_connections::1024
outbound|50000||AppMeshClient.mesh::high_priority::max_pending_requests::1024
outbound|50000||AppMeshClient.mesh::high_priority::max_requests::1024
outbound|50000||AppMeshClient.mesh::high_priority::max_retries::3
outbound|50000||AppMeshClient.mesh::added_via_api::true
outbound|50000||AppMeshClient.mesh::x.x.x.x:50000::cx_active::0
outbound|50000||AppMeshClient.mesh::x.x.x.x:50000::cx_connect_fail::0
outbound|50000||AppMeshClient.mesh::x.x.x.x:50000::cx_total::0
outbound|50000||AppMeshClient.mesh::x.x.x.x:50000::rq_active::0
outbound|50000||AppMeshClient.mesh::x.x.x.x:50000::rq_error::0
outbound|50000||AppMeshClient.mesh::x.x.x.x:50000::rq_success::0
outbound|50000||AppMeshClient.mesh::x.x.x.x:50000::rq_timeout::0
outbound|50000||AppMeshClient.mesh::x.x.x.x:50000::rq_total::0
outbound|50000||AppMeshClient.mesh::x.x.x.x:50000::hostname::
outbound|50000||AppMeshClient.mesh::x.x.x.x:50000::health_flags::healthy
outbound|50000||AppMeshClient.mesh::x.x.x.x:50000::weight::1
outbound|50000||AppMeshClient.mesh::x.x.x.x:50000::region::
outbound|50000||AppMeshClient.mesh::x.x.x.x:50000::zone::
outbound|50000||AppMeshClient.mesh::x.x.x.x:50000::sub_zone::
outbound|50000||AppMeshClient.mesh::x.x.x.x:50000::canary::false
outbound|50000||AppMeshClient.mesh::x.x.x.x:50000::priority::0
outbound|50000||AppMeshClient.mesh::x.x.x.x:50000::success_rate::-1.0
outbound|50000||AppMeshClient.mesh::x.x.x.x:50000::local_origin_success_rate::-1.0
// ...
```



**4.config_dump命令** 

说明：打印Envoy中所有的配置信息

```json
curl 127.0.0.1:15000/config_dump
{
  "name": "appofcservice:2222",
  "domains": [
    "appofcservice",
    "appofcservice:2222",
    "240.240.0.1",
    "240.240.0.1:2222"
  ],
  "routes": [
    {
      "match": {
        "prefix": "/"
      },
      "route": {
        "cluster": "outbound|2222||appofcservice",
        "timeout": "0s",
        "retry_policy": {
          "retry_on": "connect-failure,refused-stream,unavailable,cancelled,retriable-status-codes",
          "num_retries": 2,
          "retry_host_predicate": [
            {
              "name": "envoy.retry_host_predicates.previous_hosts"
            }
          ],
          "host_selection_retry_max_attempts": "5",
          "retriable_status_codes": [
            503
          ]
        },
        "max_stream_duration": {
          "max_stream_duration": "0s"
        }
      },
      "decorator": {
        "operation": "appofcservice:2222/*"
      },
      "name": "default"
    }
  ],
  "include_request_attempt_count": true
}
```



**5.contention命令** 

说明：打印互斥锁Mutex连接信息，默认关闭

```
curl 127.0.0.1:15000/contention
Mutex contention tracing is not enabled. To enable, run Envoy with flag --enable-mutex-tracing
```



**6.cpuprofiler命令** 

说明：打开关闭cpuprofiler，检查应用程序的CPU使用率和线程使用情况

```
curl -X POST http://127.0.0.1:15000/cpuprofiler?enable=y
OK
```



**7.drain_listeners命令** 

说明：断开listeners，可以指定断开入口流量和优雅关闭

```
curl -X POST http://127.0.0.1:15000/drain_listeners
OK
```

```
POST /drain_listeners?inboundonly
```

```
POST /drain_listeners?graceful
```



**8.healthcheck命令** 

说明：获取健康检查情况

 ```
 curl -X POST http://127.0.0.1:15000/healthcheck/fail
 OK
 
 curl -X POST http://127.0.0.1:15000/healthcheck/ok
 OK
 ```



**9.heapprofiler命令** 

说明：启动或禁用heapprofier

```
curl -X POST http://127.0.0.1:15000//heapprofiler?enable=y
Starting heap profiler
```



**10.hot_restart_version命令** 

说明：查看热重启的版本

```
curl -X POST http://127.0.0.1:15000/hot_restart_version
11.104
```



**11.init_dump命令** 

说明：dump当前Envoy init manager的信息

```
curl -X POST http://127.0.0.1:15000/init_dump
{}
```



**12.listeners命令** 

说明：打印Envoy中所有listener的地址

```
curl -X POST http://127.0.0.1:15000/listeners
6de837ce-6c33-4993-9264-59838f8a01c5::0.0.0.0:15090
8a421b9f-ebf5-4cb4-9035-e7cc6cc42493::0.0.0.0:15021
x.x.0.1_443::10.156.0.1:443
x.x.108.214_443::x.x.108.214:443
x.x.220.84_15012::x.x.220.84:15012
x.x.187.8_15443::x.x.187.8:15443
x.x.187.8_15012::x.x.187.8:15012
```



**13.logging命令** 

说明：可以更改模块日志级别

```
curl -X POST http://127.0.0.1:15000/logging?assert=error
active loggers:
admin: warning
aws: warning
assert: error
backtrace: warning
```



**14.memory命令** 

说明：打印内存分配信息

 ```
 curl -X POST http://127.0.0.1:15000/memory
 {
  "allocated": "21303344",
  "heap_size": "31457280",
  "pageheap_unmapped": "0",
  "pageheap_free": "557056",
  "total_thread_cache": "8057088",
  "total_physical_bytes": "34078720"
 }
 ```



**15.quitquitquit命令** 

说明：退出Envoy服务

```
curl -X POST http://127.0.0.1:15000/quitquitquit
```



**16.ready命令** 

说明：检查envoy服务是否正常活着

```
curl -X POST http://127.0.0.1:15000/ready
LIVE
```



**17.reopen_logs命令** 

说明：开启access log

```
curl -X POST http://127.0.0.1:15000/reopen_logs
OK
```



**18.reset_counters命令** 

说明：重置计数器

```
curl -X POST http://127.0.0.1:15000/reset_counters
OK
```



**19.runtime命令** 

说明：获取runtime信息数据

```json
{
 "entries": {
  "re2.max_program_size.error_level": {
   "final_value": "1024",
   "layer_values": [
    "1024",
    "",
    ""
   ]
  },
  // ...
 "layers": [
  "deprecation",
  "global config",
  "admin"
 ]
}
```



**20.runtime_modify命令** 

说明：修改运行时参数

```
curl -X POST http://127.0.0.1:15000/runtime_modify?key1=value1&key2=value2&keyN=valueN
```



**21.server_info命令** 

说明：获取Envoy服务信息

```json
{
    "name": "envoy.ratelimit",
    "category": "envoy.filters.network",
    "type_descriptor": "",
    "disabled": false
   },
   {
    "name": "envoy.redis_proxy",
    "category": "envoy.filters.network",
    "type_descriptor": "",
    "disabled": false
   },
   {
    "name": "envoy.tcp_proxy",
    "category": "envoy.filters.network",
    "type_descriptor": "",
    "disabled": false
},
```



**22.stats命令** 

说明：打印Envoy相关统计数据

```
curl -X POST http://127.0.0.1:15000/stats
cluster_manager.cds.version_text: "2021-11-16T06:54:31Z/19"
listener_manager.lds.version_text: "2021-11-16T06:54:31Z/19"
cluster.xds-grpc.assignment_stale: 0
cluster.xds-grpc.assignment_timeout_received: 0
cluster.xds-grpc.bind_errors: 0
cluster.xds-grpc.circuit_breakers.default.cx_open: 0
cluster.xds-grpc.circuit_breakers.default.cx_pool_open: 0
cluster.xds-grpc.circuit_breakers.default.rq_open: 0
cluster.xds-grpc.circuit_breakers.default.rq_pending_open: 0
cluster.xds-grpc.circuit_breakers.default.rq_retry_open: 0
cluster.xds-grpc.circuit_breakers.high.cx_open: 0
cluster.xds-grpc.circuit_breakers.high.cx_pool_open: 0
cluster.xds-grpc.circuit_breakers.high.rq_open: 0
```



**23.prometheus命令** 

说明：以prometheus的格式展现统计数据

```json
curl -X POST http://127.0.0.1:15000/stats/prometheus
istio_request_duration_milliseconds_count{response_code="200",reporter="destination",source_workload="mesha",source_workload_namespace="default",source_principal="unknown",source_app="mesha",source_version="unknown",source_cluster="Kubernetes",destination_workload="meshb",destination_workload_namespace="default",destination_principal="unknown",destination_app="meshb",destination_version="unknown",destination_service="AppMeshClient.mesh",destination_service_name="AppMeshClient.mesh",destination_service_namespace="default",destination_cluster="Kubernetes",request_protocol="grpc",response_flags="-",grpc_response_status="0",connection_security_policy="none",source_canonical_service="mesha",destination_canonical_service="meshb",source_canonical_revision="latest",destination_canonical_revision="latest"} 1
```



**24.stats/recentlookups命令** 

说明：协助envoy开发同学定位连接问题

```
curl -X POST http://127.0.0.1:15000//stats/recentlookups/clear
```

```
curl -X POST http://127.0.0.1:15000/stats/recentlookups/disable
```

```
curl -X POST http://127.0.0.1:15000/stats/recentlookups/enable
OK
```

```
curl -X POST http://127.0.0.1:15000/stats/recentlookups
   Count Lookup

total: 22
```





