---
title: Sentinel20# 针对调用来源限流的分析
categories: Sentinel
tags: Sentinel
date: 2021-05-01 12:25:01
---



# 引言 



当我们SOA的服务提供方S的某个资源（接口方法）想针不同的服务消费方（A与B）设置不同的限流阈值时，这时就需要用到针对调用来源的限流。那我们可以大规模去使用这种限流方式吗？



# 内容提要

**1.应用场景** 

示意图如下，针对A服务methodA1调用服务S的methodS1设置QPS限流300，针对B服务methodB1调用S服务的methodS1设置限流1000。也就是需要在S服务中对相同的资源（methodS1）针对不同的来源A与B设置不同的限流阈值。



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/%E9%92%88%E5%AF%B9%E8%B0%83%E7%94%A8%E6%9D%A5%E6%BA%90%E9%99%90%E6%B5%81%E5%9C%BA%E6%99%AF.png)

**2.实现原理** 

* Sentinel在统计请求流量时会为每个调用来源构建统计信息（StatisticNode）

* 在请求通过时获取调用来源origin对应的统计信息判决请求是否放行

  

**3.为什么不能大量使用针对调用来源的限流？** 

备注：由于需要为每个调用来源origin的资源建立统计信息StatisticNode，大量使用会造成内存占用过多。这点官方faq中也给出了警示。**“注意 origin 数量不能太多，否则会导致内存暴涨，并且目前不支持模式匹配。” ** 

下面举例说明，下面的例子中针对来源限流是不针对来源限流内存占用的30倍。

不针对来源限流：S服务有15个对外提供的服务接口，如果不针对来源限流，只需要15个统计StatisticNode即可

针对来源限流：如果调用S服务的消费者有30个，那么需要统计的StatisticNode的数量=30 * 15 = 450个

在实际中如果支持了其实难以控制数量是否是太多的，所以这是一个权衡的过程。首先针对调用来源的限流这个场景是不是很普遍，如果只是偶尔出现，这个功能应该考虑被禁用。



<!--more-->



# 代码示例

**注入规则**

注入同一个资源“consumer-source-test“针对不同来源AppId1、AppId2的两条规则。

```
[
    {
        "clusterMode": false,
        "controlBehavior": 0,
        "count": 20,
        "grade": 1,
        "limitApp": "AppId1",
        "maxQueueingTimeMs": 500,
        "resource": "consumer-source-test",
        "strategy": 0,
        "warmUpPeriodSec": 10
    },
    {
        "clusterMode": false,
        "controlBehavior": 0,
        "count": 40,
        "grade": 1,
        "limitApp": "AppId2",
        "maxQueueingTimeMs": 500,
        "resource": "consumer-source-test",
        "strategy": 0,
        "warmUpPeriodSec": 10
    }
]
```



**限流日志**

针对AppId2的运行日志显示限流40，针对调用来源限流生效。

```
1619830979000|2021-05-01 09:02:59|consumer-source-test|42|1135|42|0|0|0|0|0
1619830980000|2021-05-01 09:03:00|consumer-source-test|40|1152|40|0|0|0|0|0
1619830981000|2021-05-01 09:03:01|consumer-source-test|42|1180|42|0|0|0|0|0
1619830982000|2021-05-01 09:03:02|consumer-source-test|40|1172|40|0|0|0|0|0
1619830983000|2021-05-01 09:03:03|consumer-source-test|40|1152|40|0|0|0|0|0
1619830984000|2021-05-01 09:03:04|consumer-source-test|43|1159|43|0|0|0|0|0
1619830985000|2021-05-01 09:03:05|consumer-source-test|41|1195|41|0|0|0|0|0
1619830986000|2021-05-01 09:03:06|consumer-source-test|40|1160|40|0|0|0|0|0
1619830987000|2021-05-01 09:03:07|consumer-source-test|40|1209|40|0|0|0|0|0
1619830988000|2021-05-01 09:03:08|consumer-source-test|41|1159|41|0|0|0|0|0
1619830989000|2021-05-01 09:03:09|consumer-source-test|40|1136|40|0|0|0|0|0
```





# 源码分析

**针对调用来源构建统计信息**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210501082237.png)



为每个origin构建统计信息StatisticNode

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210430100704.png)



**请求通过时的校验**

当请求判断是否允许放行时，需要统计信息Node与规则阈值比较，此时获取的是origin的统计信息。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210501085359.png)



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210501085530.png)

