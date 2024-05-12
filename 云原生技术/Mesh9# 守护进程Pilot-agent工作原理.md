```
title: Mesh9# 守护进程Pilot-agent工作原理
categories: Mesh
tags: Mesh
date: 2021-11-07 11:55:01
```



# Envoy与Istio交互



Envoy以SideCar的容器与业务容器部署在一个Pod中，在Pod内部通过iptables转发流量。Envoy容器包含两个进程pilot-agent和envoy，pilot-agent为istiod与enovy的沟通桥梁。

```
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
istio-p+       1  0.0  0.2 744004 49024 ?        Ssl  Nov16   0:57 /usr/local/bin/pilot-agent proxy sidecar --domain
istio-p+      34  0.3  0.4 33824816 79800 ?      Sl   Nov16   5:30 /usr/local/bin/envoy -c etc/istio/proxy/envoy-rev0
```
