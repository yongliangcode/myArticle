---
title:Q3# ZK集群内存使用率风险评估
categories: 问题整理
tags: 问题整理
date: 2021-10-08 11:55:01
---



# 背景



在集群升级发生了Leader选举和切换，当前时期集群处于不稳定，客户端连接的节点有倾斜。有两个节点x.x.x.88和x.x.x.15内存使⽤率过⾼，需要评估其能否扛得住。由于未全部完成升级，除了节点x.x.x.122和节点x.x.x16高配机（32C64G）外，其他均为低配机（4C8G）。



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210929112618.png)





# 风险分析



### 集群架构 



注册中心由9台ZK节点构成，为了分担直接连接Leader节点的连接压力，通过域名分成三组，写操作由其内部转发到Leader操作。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210929103635.png)



### 服务注册域名组



x.x.x.15内存使⽤率为73%

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210929114156.png)



x.x.x.89内存使⽤率41%

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210929114303.png)



x.x.x.45使⽤率27%

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210929115710.png)



结论：该组均为低配节点，如果节点不能提供服务（FULL GC、挂掉、假死等）重连到其他节点，该组在极限值如果⼤部分连接到x.x.x.89可能导致该节点不可⽤，再全部重连到x.x.x.45，导致整个soazk配置的域名不可⽤。



该组是存在风险最大的一组：

* 经过两天观察运行平稳，缓存x.x.x.15节点尚有25%空间，不可用概率较低
* 当x.x.x.15节点不可用，全部冲跨剩余节点的概率也较低
* 该组域名为负责注册，按照当前故障演练测试情况来看，即使全部挂掉服务能正常调用





### 服务发现域名组



节点x.x.x.16内存使用率78%



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210929132515.png)



节点x.x.x.46内存使用率27.4%

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210929113021.png)





节点x.x.x.16内存使用率12%

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210929113559.png)



结论：节点x.x.x.88内存使⽤率78%，x.x.x.46使⽤率为27.5%，其中x.x.x.16为⾼配机（32C64G）使⽤率为12%，⼀旦x.x.x.88节点不能提供服务（FULL GC、挂掉、假死等）剩余两个节点能够扛得住客户端的重新连接。



### 配置域名组

节点x.x.x.122内存使用率11.2%

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210929133341.png)



节点x.x.x.47内存使用率20.2%

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210929133544.png)



节点x.x.x.47内存使用率20.2%

![image-20210929141418872](/Users/yongliang/Library/Application Support/typora-user-images/image-20210929141418872.png)



结论：配置域名集群无风险。



# 总结



当前风险较大集中在注册域名组节点，但是发生的不可用的概率较小，所以以观察为主，节后低峰期再处理。



# 应急预案



### 1.告警与观测



   做好告警设置和观察，特别监控内存使用率在90%时，申请执行应急预案。约在使用超过95%执行该预案



### 2.应急操作



**定向爆破**

| 步骤 | 操作过程                              |
| ---- | ------------------------------------- |
| 1    | 将节点高风险域名指向高配机器x.x.x.122 |
| 2    | 下线该高风险节点迫使客户端触发重连    |
| 3    | 升级该高风险节点为高配机              |

备注：其他节点均以此轮换进行。
