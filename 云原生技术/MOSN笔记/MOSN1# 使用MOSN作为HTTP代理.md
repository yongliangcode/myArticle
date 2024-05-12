---
title: MOSN1# 使用MOSN作为HTTP代理
categories: MOSN
tags: MOSN
date: 2021-09-12 11:55:01
---



# 引言





# 内容提要





# 静态配置注解

MOSN中有静态配置和动态配置，静态配置用于简单场景，下面的示例即静态配置，需包含一个Server和Cluster。动态配置需要MOSN启动时访问控制面板获取配置，控制表面也可在运行时推送MOSN运行配置需包含DynamicResources和StaticResources。

```json
{
  // 注解@1 是否优雅关闭，默认为false
	"close_graceful" : true, 
  // 注解@2 静态配置需包含一个需包含一个Server，当前只支持一个 
	"servers":[
		{
      // 注解@3 默认错误日志文件路径，支持stdout、stderr，默认为stderr
			"default_log_path":"stdout",
      // 注解@4 MOSN的路由设置
			"routers":[
				{
          // 注解@5 唯一路由配置标识
					"router_config_name":"server_router",
          // 注解@6  路由规则细节
					"virtual_hosts":[{
            // 注解@7 virtual host的唯一标识
						"name":"serverHost",
            // 注解@8 domain配置，支持通配符
						"domains": ["*"],
            // 注解@9 一组具体的路由匹配规则
						"routers": [
							{
                // 注解@10 路由匹配参数
								"match":{"prefix":"/"}, // 注解@11 表示匹配的路径前缀
                // 注解@12 将被路由的upstream信息
								"route":{"cluster_name":"serverCluster"}
							}
						]
					}]
				}
			],
      // 注解@13 配置MOSN启动监听的端口,配置可以通过Listener动态接口进行添加和修改
			"listeners":[
				{
          // 注解@14 Listener名称，如果配置为空会生成UUID
					"name":"serverListener",
          // 注解@15 监听的地址
					"address": "127.0.0.1:2046",
          // 注解@16 默认为true表示地址会被占用
					"bind_port": true,
          // 注解@17 配置核心逻辑，描述Listener如何处理请求
					"filter_chains": [{
						"filters": [
							{
								"type": "proxy",
								"config": {
									"downstream_protocol": "Http1",
									"upstream_protocol": "Http1",
									"router_config_name":"server_router"
								}
							}
						]
					}]
				}
			]
		}
	],
	"cluster_manager":{
		"clusters":[
			{
				"name":"serverCluster",
				"type": "SIMPLE",
        // 负载均衡策略
				"lb_type": "LB_RANDOM",
				"max_request_per_conn": 1024,
				"conn_buffer_limit_bytes":32768,
				"hosts":[
					{"address":"127.0.0.1:8080"}
				]
			}
		]
	},
	"admin": {
		"address": {
			"socket_address": {
				"address": "0.0.0.0",
				"port_value": 34902
			}
		}
	}
}
```



# 示例



**1.下载源码** 

```
https://github.com/mosn/mosn
```



**2.编译MOSN** 

```
cd ${project}/mosn/cmd/mosn/main

go build
```



**3.将MOSN移动到示例目录**

```
mv main ${project}/mosn/examples/codes/http-sample
```



**4.启动HTTP Server** 

```
cd ${project}/mosn/examples/codes/http-sample

go run server.go
```



**5.启动MOSN** 

启动Client

```
./main start -c client_config.json
```



启动Server

```
./main start -c server_config.json
```



**6.访问代理** 

```
curl http://127.0.0.1:2045/
Method: GET
Protocol: HTTP/1.1
Host: 127.0.0.1:2045
RemoteAddr: 127.0.0.1:51677
RequestURI: "/"
URL: &url.URL{Scheme:"", Opaque:"", User:(*url.Userinfo)(nil), Host:"", Path:"/", RawPath:"", ForceQuery:false, RawQuery:"", Fragment:"", RawFragment:""}
Body.ContentLength: 0 (-1 means unknown)
Close: false (relevant for HTTP/1 only)
TLS: (*tls.ConnectionState)(nil)

Headers:
Accept: */*
Content-Length: 0
User-Agent: curl/7.64.1
```



**7.直接访问Server**

```
curl 127.0.0.1:8080
Method: GET
Protocol: HTTP/1.1
Host: 127.0.0.1:8080
RemoteAddr: 127.0.0.1:53346
RequestURI: "/"
URL: &url.URL{Scheme:"", Opaque:"", User:(*url.Userinfo)(nil), Host:"", Path:"/", RawPath:"", ForceQuery:false, RawQuery:"", Fragment:"", RawFragment:""}
Body.ContentLength: 0 (-1 means unknown)
Close: false (relevant for HTTP/1 only)
TLS: (*tls.ConnectionState)(nil)

Headers:
Accept: */*
User-Agent: curl/7.64.1
```



**备注：** 通过MOSN访问代理（端口2045）和直接访问访问Server（端口8080）返回内容一致，MOSN成功代理了HTTP的访问。

