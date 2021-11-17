---
title:Q2# ZK SYN Flood与参数优化
categories: 问题整理
tags: 问题整理
date: 2021-09-30 11:55:01
---



# 引言



Zookeeper集群部分节点连接数量瞬时跌零，导致不少服务发生重连，对业务造成了影响（惊吓），本文就发生的现象进行分析和整理。



# 一、ZK监控情况



zk集群部分节点服务发生重连

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210927211003.png)



伴随着集群CPU也发生抖动

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210927211443.png)



ZK日志主要为Socket关闭等

```
 [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2182:NIOServerCnxn@368] - caught end of stream exception
EndOfStreamException: Unable to read additional data from client sessionid 0x0, likely client has closed socket
	at org.apache.zookeeper.server.NIOServerCnxn.doIO(NIOServerCnxn.java:239)
	at org.apache.zookeeper.server.NIOServerCnxnFactory.run(NIOServerCnxnFactory.java:203)
	at java.lang.Thread.run(Thread.java:745)
```



**小结：** zk节点不稳定，导致服务发生重新注册。





# 二、ZK系统日志



节点x.x.x.15 系统日志在「2021-09-26T19:16:50」「2021-09-26T19:17:50」均发生SYN flooding。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210927103606.png)





x.x.x.89 系统日志在「2021-09-26T19:16:50」和 「2021-09-26T19:17:50」均发生SYN flooding。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210927103703.png)





x.x.x.45 系统日志如下「2021-09-26T19:17:50」发生SYN flooding。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210927103924.png)





**小结：** 在问题发生时间段三个节点均在「2021-09-26T19:17:50」附近有系统错误日志发生SYN flooding。另外，这三个节点「2021-09-26T12:54:03」均有SYN flooding错误日志，但是未发现ZK节点连接断开情况。那是由于zk抖动客户端重连触发了SYN flooding ? 还是由于先发生SYN flooding导致客户端zk的客户端。

可以确定的是发送了大量请求到zk节点，节点处理不过来了。



# 三、解决方式



**SYN flood维基释意**  

```
SYN flood或称SYN洪水、SYN洪泛是一种拒绝服务攻击，起因于攻击者发送一系列的SYN（TCP握手）请求到目标系统。
SYN flood攻击目前有两种方法，不过都与服务端没收到ACK有关。恶意用户可以跳过发送最后的ACK信息；或者在SYN里透过欺骗来源IP地址，这让服务器送SYN-ACK到假造的IP地址，因此永不可能收到ACK。这两个案例服务器会花点时间等抄收通知，故一个简单的网络壅塞可能是由于没有ACK造成的。
```



也就是客户端发送大量TCP连接，TCP的等待队列被塞满，导致CPU内存等资源不足，无法提供服务。关于TCP三次握手详见以前文章[《HTTP/2协议之连接前言【原理笔记】》](https://mp.weixin.qq.com/s/QtI9ono4ssnroEO6u2R6JQ)有回顾，文中通过抓报文分析了三次握手过程。

第一步：Client发送[SYN]报文到Server。Client进入SYN_SENT状态，等待Server响应。[SYN]报文序号Seq=x《备注：截图中Seq=0》
第二步：Server收到后发送[SYN,ACK]报文给Client，ACK为x+1(备注：截图中ACK=1); [SYN,ACK]报文序号为y(备注：截图中Seq=0),Server进入SYN_RECV状态
第三步：Client收到后，发送[ACK]报文到Server，包序号Seq=x+1，ACK=y+1。Server收到后Client/Server进入ESTABLISHED状态。



**解决方式：** 

参考下面文章主要对tcp_max_syn_backlog（能接受SYN同步包的最大客户端数）和somaxconn（服务端所能accept即处理数据的最大客户端数量）参数做了调大处理。

```
https://access.redhat.com/solutions/30453
```



**小结：** 通过调整系统参数和升级zk集群配置来应对，当前的4C8G配置过低，出现的该系统错误日志总体来说是资源处理不过来了。不升级可能在未来某个时间对公司所有服务造成不可计量和不可承受的损失，在相对低峰期升级，同样会对全部服务影响，但是可控可应急，两害相权取其轻。



# 四、ZK参数优化



**ZK配置调整** 

```
# ZK中的一个时间单元
tickTime=2000
 
# 默认10，Follower在启动过程中，会从Leader同步所有最新数据，将时间调大些
initLimit=30000

# 默认5，Leader与flower的心跳检测，超过后flower表示下线，略微调大一些   
syncLimit=10

# 默认60，单客户端IP级别与单zk节点的连接数限制，调整为2000    
maxClientCnxns=2000

# 最大的会话超时时间，其实交给客户端了
# 默认的Session超时时间是在2 * tickTime ~ 20 * tickTime这个范围    
maxSessionTimeout=60000000    
    
# 两次事务快照之间可执行事务的次数，默认的配置值为100000
snapCount=3000000

# 日志文件大小kb，切换快照生成日志
preAllocSize=131072    

# 自动清理snapshot和事务日志，清理频率，单位是小时    
autopurge.purgeInterval=6  

保留的文件数目，默认3个    
autopurge.snapRetainCount=10

# 存储快照文件snapshot的目录    
dataDir=/data/zookeeper

# 事务日志输出目录
dataLogDir=/data/zookeeper/log    
```



**Jvm参数** 

在zookeeper/conf创建java.env文件

```
export JVMFLAGS="-Xms10G -Xmx10G -Xmn4G -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:/data/zkout/gc/gc-%t.log $JVMFLAGS"
```



**日志滚动** 

conf/log4j.properties调整为滚动输出

```
zookeeper.root.logger=INFO, ROLLINGFIL

zookeeper.log.dir=/data/zkout/logs

log4j.appender.ROLLINGFILE=org.apache.log4j.DailyRollingFileAppender
#log4j.appender.ROLLINGFILE.Threshold=${zookeeper.log.threshold}
#log4j.appender.ROLLINGFILE.File=${zookeeper.log.dir}/${zookeeper.log.file}
log4j.appender.ROLLINGFILE.DatePattern='.'yyyy-MM-dd

log4j.appender.ROLLINGFILE.MaxFileSize=100MB
```

zkEnv.sh调整日志输出方式

```
if [ "x${ZOO_LOG_DIR}" = "x" ]
then
    ZOO_LOG_DIR="/data/zkout/logs/"
fi

if [ "x${ZOO_LOG4J_PROP}" = "x" ]
    
then
    # ZOO_LOG4J_PROP="INFO,CONSOLE"
    ZOO_LOG4J_PROP="INFO,ROLLINGFILE"
fi
```



**系统参数调整** 

limits.conf 设置用户能打开的最大文件数

```
vim /etc/security/limits.conf
# End of file
root soft nofile 655360  #根据部署用户
root hard nofile 655360
* soft nofile 655360
* hard nofile 655360
```

系统参数调整

```
vim /etc/sysctl.conf

# 表示开启SYN Cookies。当出现SYN等待队列溢出时，启用cookies来处理，可防范少量SYN攻击，默认为0，表示关闭；
net.ipv4.tcp_syncookies = 1

# 表示开启重用。允许将TIME-WAIT sockets重新用于新的TCP连接，默认为0，表示关闭；为1，开启    
net.ipv4.tcp_tw_reuse = 1

# 表示开启TCP连接中TIME-WAIT sockets的快速回收，默认为0，表示关闭；为1，开启；    
net.ipv4.tcp_tw_recycle = 1  
 
# 修改系統默认的 TIMEOUT 时间    
net.ipv4.tcp_fin_timeout = 5    

# the kernel's socket backlog limit    
net.core.somaxconn = 65535

# the application's socket listen backlog    
net.ipv4.tcp_max_syn_backlog = 100000
 
sysctl -p    
```































