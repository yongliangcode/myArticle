---
title: Nacos16# NacosSync双向复制使用指南
categories: Nacos
tags: Nacos
date: 2021-08-26 11:55:01
---



# 引言

Nacos的注册发现和配置中心的源码基本录完了，还有一块是不同集群之间的同步。zk同步到Nacos集群，nacos集群之间做多活需要数据复制等。那本文就先看下如何使用它，进而后面文章分析如何实现。



# 配置指南



**1.源码部署配置** 

创建数据库 **nacos_sync** 源码启动后自动生成了三张表。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210804115647.png)



**2.添加集群配置** 

以本地和UAT两个集群为例。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210804142614.png)



**3.创建双向同步** 

配置local和uat两个集群之间的双向同步。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210804142720.png)



**4.检查同步效果** 

nacos.test.4为从Local复制过来的。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210804142255.png)

nacos.test.3为从Uat复制过来的。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210804142348.png)





# 小结

通过上面简单配置即可完成集群之前的双向复制，然而原生zk复制功能只支持dubbo zk复制到nacos，未使用dubbo的避免不了自己改造实现。







