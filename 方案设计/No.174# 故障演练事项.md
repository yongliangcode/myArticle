---

title: No.174# 中间件演进和稳定性治理实践
categories: 方案设计
tags: 方案设计
date: 2022-10-04 11:55:01
---



# 引言



方向思维

混沌工程

《Chaos Engineering》



![image-20221031134048347](/Users/admin/Library/Application Support/typora-user-images/image-20221031134048347.png)





![image-20221031134555970](/Users/admin/Library/Application Support/typora-user-images/image-20221031134555970.png)





1、故障可以打到实例、请求、集群，指哪打哪

2、RPC框架+流量染色 只影响特定流量



故障扩散： 对A-->做故障演练，如果对公共服务C注入故障，同时影响了A和B。

A--->C

B--->C

A服务向公共服务C发起请求时干扰，例如：iptables出口流量，只对出流量进行干扰。

或者通过流量染色只对A调用C的流量进行植入故障。



1、生产环境演练的必要性

2、时间成本很高

3、一年都很稳定

4、故障演练收益不明显



设计可扩展的故障中心，实现精准可控的爆炸半径控制，降低风险成本。



![image-20221031142903955](/Users/admin/Library/Application Support/typora-user-images/image-20221031142903955.png)





![image-20221031161101522](/Users/admin/Library/Application Support/typora-user-images/image-20221031161101522.png)



![image-20221031161123065](/Users/admin/Library/Application Support/typora-user-images/image-20221031161123065.png)



1、贴近生产环境去模拟故障

* 强弱依赖自动化分析与红蓝对抗平台

* 自动化指标分析强弱依赖识别

2、自演练融入到日常迭代中

* 无通知注入
* 在规定的时间定位故障并恢复
* 人的混动工程

3、故障恢复的手段

* 混动工程有预案



![image-20221031163551014](/Users/admin/Library/Application Support/typora-user-images/image-20221031163551014.png)





![image-20221031163652825](/Users/admin/Library/Application Support/typora-user-images/image-20221031163652825.png)



最佳实践：与预案平台联动，预案执行手册



![image-20221031164411257](/Users/admin/Library/Application Support/typora-user-images/image-20221031164411257.png)



![image-20221031164703505](/Users/admin/Library/Application Support/typora-user-images/image-20221031164703505.png)



服务框架



![image-20221031165019396](/Users/admin/Library/Application Support/typora-user-images/image-20221031165019396.png)









红蓝军SOP体系



固化蓝军操作、降低演练风险



chaosblade 故障模型不清晰

Wukonglab 自动化程度高、建模抽象度高

Google DIRT （Dizaster Recovery Testing）演练前、中半结构化描述，沉淀与检索。

Netfix ChAP可进行自动化持续演练以及自动化稳态分析，实现Continuous Verification（CV），其自动化演练case来自服务治理分析。

LinkedIn LinkedOut：精细化爆炸半径控制、业务级filter（userId=xxx），作为框架层插件。



云原生方向发展：

1、Chaos Mesh以云原生形态出现

2、ChaosBlade 完成云原生改造



持续验证（自动化）

验证用户应急能力



沉淀持续验证库、将无副作用的故障注入加入持续验证库。
