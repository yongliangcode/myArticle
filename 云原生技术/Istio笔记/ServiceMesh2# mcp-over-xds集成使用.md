

```
title: ServiceMesh2# mcp-over-xds集成使用
categories: ServiceMesh
tags: ServiceMesh
date: 2021-09-31 11:55:01
```



# 引言



# 一、准备工作



Naocs开启MCP Server配置

```
nacos.istio.mcp.server.enabled=true
```



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210904184537.png)





**监听端口查看** 

```
lsof -i:18848
COMMAND   PID      USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
java    25973 yongliang  385u  IPv6 0xffb40159335a3441      0t0  TCP *:18848 (LISTEN)
```









