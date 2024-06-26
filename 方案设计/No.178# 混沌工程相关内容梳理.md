---
title: No.178# 混沌工程相关内容梳理
categories: 方案设计
tags: 方案设计
date: 2022-11-27 11:55:01
---



# 引言

随着公司规模业务的快速增长，数以千计甚至万计的微服务，依赖的各类组件越来越多。



分布式体系架构体系越来越复杂，没有任何一个人能够掌控所有复杂的耦合性。



也就是说复杂性无法避免，不可能再回到单体应用，也无法彻底消除这种复杂性。



需要考虑的是如何应对这种复杂性问题。



# 一、混沌工程要点梳理



在《混沌工程--复杂系统韧性实现之道》给出了对了混沌工程的定义：



混沌工程在分布式系统上进行实验的学科，目的是建立对该系统能够能够承受生产环境的动荡条件的信心。



混沌工程原则5条黄金标准：

* 建立关于稳态行为的假说
* 多样化的引入现实世界的事件
* 在生产环境中进行实验
* 持续运行自动化实验
* 最小化爆炸半径



稳态行为需要关键指标衡量，所以监控的关键指标就特别重要。



多样化的引入现实世界的事件，注意演练case的质量，多个视角观测，不拘役简单实现，也不忽略用户体验。



在生产环境中进行实验，其实刚起步阶段基本都是在非生产环境执行的，随着成熟度的提升，逐步在生产环境演练。



持续运行自动化实验，从最开始拉着运维手动干，逐步沉淀到混动系统中。



最小化爆炸半径，提升精准打击能力，某个集群、某个节点、某个应用、某个接口、某次调用。



通过混沌工程建立一种文化，在不确定的结果出现时保持系统的韧性。



通过混沌工程不断探索未知、将未知变为所知、提升应对不确定风险的韧性。



# 二、混动工程平台建设



构建蓝绿对抗平台：案例编排、演练自动化、沉淀案例、统计可视化、成熟度度量。



#### 故障点注入

* 注入的故障类型
* 注入的目标服务
* 注入的故障内容
* 注入的故障监控



备注：例如故障监控关联日志、链路、操作记录等。



#### 可视化统计

* 故障演练执行CASE数量
* 攻防成功失败CASE数量
* 演练发现问题数量
* 演练故障类型分布
* 演练问题类型分布



#### 成熟度模型

* 成熟程度的等级
* 引发的故障等级
* 演练的时间范围
* 故障的持续时间
* 故障的恢复能力





# 三、精准打击实现方法



「最小化爆炸半径」 是混沌工程的5条黄金法则之一，下面梳理两种精准打击方法。



#### 1、通过流量染色实现精准打击



下面对服务C做故障演练，例如：注入对某个方法注入超时5秒故障，此时对A服务和B服务均会造成故障。



可以通过对故障流量染色，只对属于故障演练的请求注入故障。

![请求级别故障注入 (1)](/Users/admin/Downloads/请求级别故障注入 (1).png)



另外可以对iptables出口流量干扰，制造请求级故障。



#### 2、通过输入参数控制爆炸半径



例如：通过拦截器分析入参，对输入用户ID在特定范围的请求，篡改返回参数制造故障。





# 四、强弱依赖自动判定



强弱依赖治理   平台能力在强弱依赖治理演进上可以考虑下：梳理强弱依赖主要是**保障核心**服务的稳定性 ，避免由于弱依赖服务故障对核心服务造成拖累。  



服务之间的依赖：微服务之间的上下游调用形成服务之间的依赖。

强依赖： 服务A调用服务B，B服务出现故障时A服务也不可用


弱依赖： 服务A调用服务B，B服务出现故障时A服务仍然可用



#### 方式一：强弱依赖人工梳理



通过走查服务代码核心逻辑，识别哪些链路接口是核心服务，哪些是非核心链路。



针对这些非核心链路逻辑能够拆分到非核心服务中去，针对非核心链路可以配置降级措施。



#### 方式二、通过对依赖接口注入故障

步骤1：为选定服务或接口拉取依赖关系        

步骤2：为接口依赖设置预判预期        

步骤3：为依赖接口注入故障并引入流量        

步骤4：监控指标并观测影响        

步骤5：强弱依赖结果判定



![强弱依赖感知 (2)](/Users/admin/Downloads/强弱依赖感知 (2).png)





# 五、常见通用故障Case梳理



| 故障类型      | 事项说明                                                     |
| ------------- | ------------------------------------------------------------ |
| 节点故障      | 杀掉ecs节点、进程、Pod、容器、节点重启等                     |
| 网络故障      | 网络分区、网络中断、网络丢包、网络延迟、网络阻塞、带宽限制   |
| IO类故障      | 磁盘损坏、写入IO延迟、写入返回错误等、IO瞬时飙高             |
| CPU/缓存/域名 | CPU瞬时飙高、内核内存分配异常、内存/CPU使用率过高、DNS域名解析错误 |
| 集群类故障    | 扩缩容、频繁变更节点角色                                     |
| 应用类故障    | 请求超时、网络异常、重复请求、入参修改、返回值修改、限流熔断异常、线程池填满、频繁GC、内存溢出等 |



















