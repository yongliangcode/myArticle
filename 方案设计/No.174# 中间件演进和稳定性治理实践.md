---
title: No.174# 中间件演进和稳定性治理实践
categories: 方案设计
tags: 方案设计
date: 2022-10-04 11:55:01
---



# 引言



如果把带团队看作是驾驭一辆马车，想把这辆马车驶向何方？



如果把带团队看作是驾驭一辆马车，如何让马车走的稳一点？



把  “看方向” 和 “稳定性治理” 体系化，保障驾驭的马车平稳行进。

​																											——老梁



通过对中间件功能、架构以及关键能力的定期聚焦，暴露中间件存在的问题和风险，把控未来演进方向，呈现中间件现状和未来演进的清晰画像。



每个中间件从功能、架构、关键能力入手，根据公司战略，延伸到业务赋能、降本提效、用户体验等方向，通过定期聚焦，确保演进规范能指导中间件未来演进。



通过对中间件分类分级梳理，制定不同的变更规范，让风险暴露在停留期、小范围、低等级服务，保障中间件的平稳运行，降低对业务的影响。



通过容灾能力设计、遵守变更规范、落实代码评审、完善监控告警、蓝绿攻防演练、事故案例复盘等方面构建 “稳定性治理”  体系。



本文容灾能力设计方面主要拓展了异地双活实践方案和注意事项，每页PPT可以拓展为一篇组件的具体实现的文章。



本文虽围绕中间件领域展开，其他技术领域数据智能、运维保障、开发测试等稍加变通也可参考。





![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20221004102459.png)





# 一、文章目录与个人介绍



![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20221004105741.png)



![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20221004105953.png)





# 二、中间件演进规范实践



![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20221004110615.png)



![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20221004110207.png)



![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20221004110250.png)



![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20221004110316.png)



![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20221004110348.png)



![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20221004110415.png)



![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20221004110445.png)



![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20221004110535.png)



# 三、中间件变更规范实践



![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20221004110714.png)

![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20221004110743.png)

![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20221004110813.png)



![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20221004110848.png)

![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20221004110913.png)

![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20221004110940.png)

![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20221004111014.png)



# 四、中间件异地双活实践



![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20221004111129.png)

![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20221004111158.png)

![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20221004111227.png)

![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20221004111302.png)

![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20221004111340.png)

![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20221004111405.png)

![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20221004111431.png)

![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20221004111653.png)

![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20221004111744.png)

![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20221004111820.png)

![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20221004111850.png)

![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20221004111930.png)

![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20221004112003.png)

![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20221004112036.png)

![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20221004112101.png)

![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20221004112134.png)



# 五、稳定性治理内容提点



![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20221004112249.png)

![](https://raw.githubusercontent.com/yongliangcode/md-picture/master/img2/20221004112329.png)

