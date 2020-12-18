---
title: MQ12# RocketMQ性能优化
categories: RocketMQ
tags: RocketMQ
date: 2020-12-20 18:14:01
---



# 目录

```
一、系统优化

1.最大文件数

2.系统参数调整

二、RocketMQ性能调优

1.开启异步刷盘

2.开启堆外内存设置

3.开启文件预热

4.开启Slave读权限

5.关闭堆内存据传输
```



# 系统优化



## 最大文件数

limits.conf 设置用户能打开的最大文件数

```
vim /etc/security/limits.conf

\# End of file

baseuser soft nofile 655360

baseuser hard nofile 655360

\* soft nofile 655360

\* hard nofile 655360
```



## 系统参数调整

```
sudo vim /etc/sysctl.conf

vm.overcommit_memory=1

vm.drop_caches=1

vm.zone_reclaim_mode=0

vm.max_map_count=655360

vm.dirty_background_ratio=50

vm.dirty_ratio=50

vm.dirty_writeback_centisecs=360000

vm.page-cluster=3

vm.swappiness=1

vm.zone_reclaim_mode=1 // 主节点

vm.min_free_kbytes=512000

sudo sysctl -p
```



## 参数说明

### overcommit_memory

```
是否允许内存的过量分配

当为0的时候，当用户申请内存的时候，内核会去检查是否有这么大的内存空间

当为1的时候，内核始终认为，有足够大的内存空间，直到它用完了为止

当为2的时候，内核禁止任何形式的过量分配内存
```

### drop_caches

```
写入的时候，内核会清空缓存，腾出内存来，相当于sync

写1的时候，会清空页缓存，就是文件

写2的时候，会清空inode和目录树

写3的时候，都清空

This is a non-destructive operation and will only free things that are completely unused.

Dirty objects will continue to be in use until written out to disk and are not freeable.

If you run "sync" first to flush them out to disk, these drop operations will tend to free more memory.
```

### zone_reclaim_mode

```
如果为0的话，那么系统会倾向于从其他节点分配内存

如果为1的话，那么系统会倾向于从本地节点回收Cache内存多数时候
```

### max_map_count

```
定义了一个进程能拥有的最多的内存区域，默认为65536
```



### dirty_background_bytes/dirty_background_ratio

```
当dirty cache到了多少的时候，就启动pdflush进程，将dirty cache写回磁盘

当有dirty_background_bytes存在的时候，dirty_background_ratio是被自动计算的
```



### dirty_bytes/dirty_ratio

```
当一个进程的dirty cache到了多少的时候，启动pdflush进程，将dirty cache写回磁盘

当dirty_bytes存在的时候，dirty_ratio是被自动计算的
```



### dirty_writeback_centisecs

```
pdflush每隔多久，自动运行一次（单位是百分之一秒）
```



### page-cluster

```
每次swap in或者swap out操作多少内存页为2的指数。

等于0的时候，为1页；等于1的时候，为2页；等于2的时候，为4页
```



### swappiness

```
swappiness=0 仅在内存不足的情况下，当剩余空闲内存低于vm.min_free_kbytes limit时，使用交换空间

swappiness=1 内核版本3.5及以上、Red Hat内核版本2.6.32-303及以上，进行最少量的交换，而不禁用交换

swappiness=10 当系统存在足够内存时，推荐设置为该值以提高性能

swappiness=60 默认值

swappiness=100 内核将积极的使用交换空间
```



<!--more-->



#  RocketMQ性能调优

线上RocketMQ的JVM未做调优，堆内存使用8G；主要从RocketMQ配置参数方面梳理下。



## 开启异步刷盘

```
flushDiskType=ASYNC_FLUSH

同步刷盘TPS过低，较难满足业务发展需求
```



## 开启堆外内存设置

```
transientStorePoolEnable=true

消息写入到堆外内存，消费时从pageCache消费，读写分离，提升集群性能
```



## 开启文件预热

```
warmMapedFileEnable=true

开启文件预热，避免日志文件在分配内存时缺页中断
```



## 开启Slave读权限

```
slaveReadEnable=true

消息占用物理内存的大小通过accessMessageInMemoryMaxRatio来配置默认为40%；

如果消费的消息不在内存中，开启slaveReadEnable时会从slave节点读取；提高Master内存利用率
```



## 关闭堆内存据传输

```
transferMsgByHeap默认true设置为false

Broker响应消费请求时，不必将数据重新读到堆内存再发送给客户端；

直接从PageCache将数据发送给客户端
```



## 延长发送队列等待时间

```
waitTimeMillsInSendQueue默认200ms；适当延长到1000ms

降低由于等待超时客户端快速失败抛出[TIMEOUT_CLEAN_QUEUE]broker busy频率
```

