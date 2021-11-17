---
title: MQ45# 实战|RocketMQ不同可用区导致消费不均衡
categories: RocketMQ
tags: RocketMQ
date: 2021-08-15 20:55:01
---



# 现象反馈



业务同学反馈有个服务在部署容器后不间断收到积压告警，该服务对积压敏感，影响派单的时效性。原来部署到ECS上的服务没有积压情况，准备往容器迁移。下面是业务同学做的排除测试，另外容器当前在J/K可用区部署，而MQ集群部署在B/G/F区。



* 回退到原ECS部署积压消失
* 在原可用区申请扩容ECS未出现积压
* 在新的可用区J/K申请ECS出现积压



**备注：** 很明显该积压与可用区有关系。



# 积压监控

在迁移容器的过程中，同时有容器消费和ECS消费的节点，通过分区积压进行对比。

**ECS消费分区积压监控** 

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210807112009.png)



**备注：** 明显ECS的节点没有什么积压。



**容器消费分区积压监控** 

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210807113327.png)



**备注：** 积压较多的分区分布在容器节点。



# 可用区耗时监控

**J/F可用区延迟** 

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210810184231.png)



**G/B/K可用区延迟** 

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210810184328.png)



**备注：** J/K区的延迟比其他可用区多0.5ms左右。



# 解决措施



既然由于可用区延迟引起，可以考虑一下几种措施：



**1.将MQ集群迁移到J/K可用区** 

由于其他可用区还有重要业务，明显不可行。



**2.将容器发布部署非J/K可用区** 

容器可以相对考虑可用区的均衡性，但是难以避免不同可用区混部，也不太可行。



**3.提高消费能力** 

通过提高部署容器节点和增加消费线程池大小来提高消费能力可以起到立竿见影的效果。

