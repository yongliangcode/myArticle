# 背景介绍

哈啰已进化为包括两轮出行（哈啰单车、哈啰助力车、哈啰电动车、小哈换电）、四轮出行（哈啰顺风车、全网叫车、哈啰打车）等的综合化移动出行平台，并向酒店、到店团购等众多本地生活化生态探索。



随着公司业务的不断发展，流量也在不断增长。我们发现生产中的一些重大事故，往往是被突发的流量冲跨的，对流量的治理和防护，保障服务高可用就尤为重要。



服务之间的交互通常可以分为直接调用和通过消息队列解耦的异步调用。他们都可以归为流量调用范畴，本文就哈啰在消息流量和微服务调用的治理中踩过的坑、积累的经验一个分享。



# 个人介绍

梁勇（老梁）就职于哈啰出行，任职高级技术专家。当前主要从事中间件的相关开发和治理工作，在公众号「瓜农老梁」发表百余篇原创文章，涵盖开源中间件源码分析、实战笔记、性能优化、方案设计等。《RocketMQ实战与进阶》专栏联合作者、参与了《RocketMQ技术内幕》审稿工作。





# 聊聊治理这件事



**治理在干一件什么事？**

​	*让环境变得美好一些*



**需要知道哪些地方还不够好？** 

* 以往经验

* 用户反馈

* 业内对比



**还需要知道是不是一直都是好的？**

* 监控跟踪
* 告警通知



**不好的时候如何再让其变好？**

* 治理措施
* 应急方案



# 微服务调用链路模型



尽管业务生态越来越多，每个公司都能画出庞大的架构图。其服务之间的核心链路可以简化如下：

* A请求通过RPC直接调用B
* A将消息发送给消息队列，B从消息队列消费，完成A和B的交互（异步RPC）



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/%E5%BE%AE%E6%9C%8D%E5%8A%A1%E8%B0%83%E7%94%A8%E6%A0%B8%E5%BF%83%E9%93%BE%E8%B7%AF%E6%A8%A1%E5%9E%8B.png)

今天分享的主题也将围绕这两种调用交互方式展开。



# 目录

* 裸奔时代的痛
* 打造分布式消息治理平台
* 打造微服务高可用治理平台





# 裸奔时代的痛



*回忆下那些曾经不太美好的日子* 



**裸奔的RabbitMQ**

* 积压过多是清理还是不清理？这是个问题，我再想想
* 积压过多触发集群流控？那是真的影响业务了
* 想消费前两天的数据？请您重发一遍吧
* 要统计哪些服务接入了？您要多等等了，我得去捞IP看看
* 有没有使用风险比如大消息？这个我猜猜



**裸奔的服务** 

  曾经有这么一个故障，多个业务共用一个数据库。在一次晚高峰流量陡增，把数据库打挂了。

* 数据库单机升级到最高配依然无法解决

* 重启后缓一缓，不一会就又被打挂了
* 如此循环着、煎熬着、默默等待着高峰过去



**思考：需要完善的治理措施**



# 打造分布式消息治理平台



*哪些是我们的关键指标，哪些是我们的次要指标，这是消息治理的首要问题* 



**目标** 

​	旨在屏蔽底层各个中间件（RocketMQ/Kafka）的复杂性，通过唯一标识动态路由消息。同时打造集资源管控、检索、监控、告警、巡检、容灾、可视化运维等一体化的消息治理平台，保障消息中间件平稳健康运行。



**架构** 



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/HMS-%E6%9E%B6%E6%9E%84%E5%9B%BE%E5%88%86%E4%BA%AB%20(2).png)





# 迁移RabbitMQ到RocketMQ



*不好用就换个好用的* 



* 推动接入统一SDK
* 一键迁移（主题 + 关联消费组一起迁移）
* 自助迁移 （根据进度自行迁移）





![](https://gitee.com/laoliangcode/md-picture/raw/master/img/%E8%BF%81%E7%A7%BBRabbitMQ%E5%88%B0RocketMQ.png)





# RocketMQ实战中踩过的坑和解决方案



*我们总会遇到坑，遇到就把它填了*



**RocketMQ集群CPU毛刺** 

问题描述：RocketMQ从节点、主节点频繁CPU飙高，很明显的毛刺，很多次从节点直接挂掉了。



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210325151305.png)



只有系统日志有错误提示

 ```
2020-03-16T17:56:07.505715+08:00 VECS0xxxx kernel: <IRQ>  [<ffffffff81143c31>] ? __alloc_pages_nodemask+0x7e1/0x960
2020-03-16T17:56:07.505717+08:00 VECS0xxxx kernel: java: page allocation failure. order:0, mode:0x20
2020-03-16T17:56:07.505719+08:00 VECS0xxxx kernel: Pid: 12845, comm: java Not tainted 2.6.32-754.17.1.el6.x86_64 #1
2020-03-16T17:56:07.505721+08:00 VECS0xxxx kernel: Call Trace:
2020-03-16T17:56:07.505724+08:00 VECS0xxxx kernel: <IRQ>  [<ffffffff81143c31>] ? __alloc_pages_nodemask+0x7e1/0x960
2020-03-16T17:56:07.505726+08:00 VECS0xxxx kernel: [<ffffffff8148e700>] ? dev_queue_xmit+0xd0/0x360
2020-03-16T17:56:07.505729+08:00 VECS0xxxx kernel: [<ffffffff814cb3e2>] ? ip_finish_output+0x192/0x380
2020-03-16T17:56:07.505732+08:00 VECS0xxxx kernel: [<ffffffff811862e2>] ? 
 ```



各种调试系统参数只能减缓缺不能根除



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210325151334.png)





**解决方案**

将集群所有系统升级从centos6升级到centos7，内核版本也从从2.6升级到3.10。



**备注**

[详见RocketMQ社区帖子](https://github.com/apache/rocketmq/issues/1910)



**RocketMQ集群线上延迟消息失效**

问题描述：RocketMQ社区版默认本支持18个延迟级别，每个级别在设定的时间都被会消费者准确消费到。为此也专门测试过消费的间隔是不是准确，测试结果显示很准确。然而，如此准确的特性居然出问题了，接到业务同学报告线上某个集群延迟消息消费不到，诡异！



**解决方案**

将"delayOffset.json"和"consumequeue/SCHEDULE_TOPIC_XXXX"移到其他目录，相当于删除；逐台重启broker节点。重启结束后，经过验证，延迟消息功能正常发送和消费。



**备注** 

[详见RocketMQ社区帖子](https://github.com/apache/rocketmq/issues/1869)

[详细复盘过程](https://mp.weixin.qq.com/s/f27tcOlm0_BTvL8725RPTQ)



# 消息治理平台设计需要考虑的点

* 提供简单易用API
* 有哪些关键点能衡量客户端的使用没有安全隐患
* 有哪些关键指标能衡量集群健康不健康
* 有哪些常用的用户/运维操作将其可视化
* 有哪些措施应对这些不健康





# 尽可能简单易用



*把复杂的问题搞简单，那是能耐*



**极简统一API** 

提供统一的SDK封装了（Kafka/RocketMQ）两种消息中间件。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/02.API%E8%AE%BE%E8%AE%A1.png)



**一次申请**

主题/消费组申请--->工单审批（钉钉）--->多环境实时生效---> 自动生成告警规则



**发送示例**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210326114005.png)



**消费示例** 

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210326114051.png)





# 客户端治理



*监控客户端使用是否规范，找到合适的措施治理*



**抓手** 



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/%E5%AE%A2%E6%88%B7%E7%AB%AFSDK%E5%9F%8B%E7%82%B9.png)



**影响**

* 瞬时流量过大触发集群流控？
* 发送消息过大触发IO时间过长？
* 先摘流量后排查？
* 发送/消费耗时过长影响性能？
* 链路估计提高排查效率？





**措施** 

通过巡检将存在隐患的发送或者消费推送给告警系统，通知应用负责人，专项治理跟踪治理结果。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/%E5%AE%A2%E6%88%B7%E7%AB%AF%E6%B2%BB%E7%90%86%E6%8E%AA%E6%96%BD2.png)





# 主题/消费组治理



*监控主题消费组资源使用情况*



**抓手** 



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/%E4%B8%BB%E9%A2%98%E6%B6%88%E8%B4%B9%E7%BB%84%E7%9B%91%E6%8E%A7.png)



**影响**

* 消费积压影响业务？
* 业务与消费/发送速度有关系？
* 节点掉线通知？
* 队列分区不均衡影响性能？



**措施**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/%E4%B8%BB%E9%A2%98%E6%B6%88%E8%B4%B9%E7%BB%84%E6%B2%BB%E7%90%86%E6%8E%AA%E6%96%BD.png)



# 集群健康治理



*度量集群健康的核心指标有哪些？*



**抓手** 

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/04.%E9%9B%86%E7%BE%A4%E7%9B%91%E6%8E%A7.png)



**影响**

* 集群不可用影响大的去了？公司要不要你还两说
* 抖动往往意味着客户端发送会超时
* 集群流控最好不要有
* 集群抖动最好不要有



如果说这些关键指标中哪一个最重要？我会选择响应时间（RT），下面看看影响RT可能哪些原因。



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/%E5%BD%B1%E5%93%8DRT%E7%9A%84%E5%9B%A0%E7%B4%A0.png)





**措施** 

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/%E9%9B%86%E7%BE%A4%E6%B2%BB%E7%90%86%E6%8E%AA%E6%96%BD%E6%96%B0.png)



# 关于告警

* 监控指标大多是秒级探测
* 触发阈值的告警推送到公司统一告警系统、实时通知
* 巡检的风险通知推送到公司巡检系统、每周汇总通知



# 说说消息的流量分离



**哪些会用到流量分离？**

* 全链路压测分离压测流量
* 测试场调试分离不同流量
* 灰度发布分离灰度流量



**消息流量分离有哪些措施？**

* 消息头部打标区分不同流量
* 通过不同主题/消费组将流量分离
* 通过不同的集群将流量分离



**测试场流量分离举例**

* 紫色线是测试场STABLE环境部署的Master分支服务
* 红色线是本次新功能的调试的服务被拉到了测试场
* 拉入测试场的服务的流量会在链路中透传测试标记
* 具体到消息会通过不同的主题和消费组+头部打标实现

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/%E6%B5%8B%E8%AF%95%E5%9C%BA%E6%B5%81%E9%87%8F%E5%88%86%E7%A6%BB%E4%B8%BE%E4%BE%8B.png) 





# 聊聊延迟消息投递



延迟消息使用场景太多了，通常有哪些做法？

* 定时轮询，定时从存储介质（MySQL/RocksDB/Redis等）
* 时间轮？+ 本地存储索引？
* 开源中间件不能满足需求？



**有没有一种简单高效的方案？供参考** 

* 扩增RocketMQ的延迟等级，例如：扩增7个等级让RocketMQ存储支持35天
  messageDelayLevel = 1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h 1d 3d 7d 14d 21d 28d 35d

* 将一天内的消息存储在Pulsar集群，利用其任意时刻时间特性进行投递





![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210327191135.png)

**特点**

* 充分利用RocketMQ和Pulsar特性
* 存储时间2个月？3个月？1年均可
* 投递能力可伸缩，每秒10万？20万？你说了算
* 投递进度秒级？不是梦





# 消息平台看板截图



![RocketMQ集群看板截图3](/Users/yongliang/work/架构师大会文档/消息治理/RocketMQ集群看板截图3.png)



# 深入研究 RocketMQ 源码为治理保驾护航



**阅读源码有啥用？老梁的理解**

* 遇到问题不慌

* 解决问题有思路

* 能够链接到一些专业同学一起探讨

* 为技术方案的设计提供素材

* 知行合一（源码+实战）





# 打造微服务高可用治理平台



*哪些是我们的核心服务，哪些是我们的非核心服务，这是服务治理的首要问题*



**目标**

服务能应对突如其来的陡增流量，尤其保障核心服务的平稳运行。





# 应用分级和分组部署



**应用分级** 

根据用户和业务影响两个纬度来进行评估设定的，将应用分成了四个等级。

* 业务影响：应用故障时影响的业务范围
* 用户影响：应用故障时影响的用户数量



S1： 核心产品，产生故障会引起外部用户无法使用或造成较大资损，比如主营业务核心链路，如单车、助力车开关锁、顺风车的发单和接单核心链路，以及其核心链路强依赖的应用。



S2:  不直接影响交易，但关系到前台业务重要配置的管理与维护或业务后台处理的功能。



S3:  服务故障对用户或核心产品逻辑影响非常小，且对主要业务没影响，或量较小的新业务；面向内部用户使用的重要工具，不直接影响业务，但相关管理功能对前台业务影响也较小。



S4:  面向内部用户使用，不直接影响业务，或后续需要推动下线的系统。



**分组部署**

S1服务是公司的核心服务，是重点保障的对象，需保障其不被非核心服务流量意外冲击。

* S1服务分组部署，分为Stable和Standalone两套环境
* 非核心服务调用S1服务流量路由到Standalone环境
* S1服务调用非核心服务需配置熔断策略



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/%E5%BA%94%E7%94%A8%E5%88%86%E7%BB%84%E9%83%A8%E7%BD%B22.png)

# 多种限流熔断能力建设



**我们想要** 



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/%E6%9C%8D%E5%8A%A1%E6%B2%BB%E7%90%86%E7%9B%AE%E6%A0%87.jpg)



**能力建设**



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/%E5%A4%9A%E7%BB%B4%E9%AB%98%E5%8F%AF%E7%94%A8%E8%83%BD%E5%8A%9B.png)



**部分效果图**



**预热图示** 

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/%E9%A2%84%E7%83%AD%E6%95%88%E6%9E%9C.jpg)



**排队等待**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/%E6%8E%92%E9%98%9F%E6%95%88%E6%9E%9C.jpg)



**预热+排队**

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/%E9%A2%84%E7%83%AD+%E6%8E%92%E9%98%9F.jpg)





**界面截图**

* 中间件全部接入
* 动态配置实时生效



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/%E9%AB%98%E5%8F%AF%E7%94%A8%E5%B9%B3%E5%8F%B0%E6%88%AA%E5%9B%BE.jpg)





# 统一卡点核心服务强制接入



**发布门禁** 

* 通过发布系统设置卡点，对于小于特定版本的SDK禁止应用发布
* 先从非核心S4应用卡点运行一周后逐步向上一级别服务卡点
* 对于发布频次过低的服务通过专项推动升级



# 线上冷/惰应用定期巡检整治



**冷/惰应用** 

* 冷应用是指线上的应用长期没有什么流量，空占着机器资源的一种情况
* 惰应用是指线上流量过低甚至个位数的请求数



**治理措施**

* 定期巡检线上服务流量，生成冷/惰应用报告
* 针对冷应用可以采取下线措施
* 针对惰应用可以采取合并措施



# 思考：从应用到集群拓广治理维度



* 服务限流与集群健康状况联动
* 高可用手动配置向预测自动化设置发展







