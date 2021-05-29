---
title: FA4# 哈啰在分布式消息治理和微服务治理中的实践
categories: 方案设计
tags: 方案设计
date: 2021-05-04 11:55:01
---



# 偶然

在今年3月初的时候敏捷教练陈文博应该是接到大领导指示让去召集人去投稿全球架构师大会（ArchSummit），目的在于提升公司的影响力和增进技术交流。



文博发我一个大会议题链接说让投稿，老梁扫了一眼没发现没合适的主题。这家伙找益伟去了，益伟给了个列表，微服务架构方向让我试试看。领导说让试试，你还能说不？这个大会老梁以前作为听众参加过，被选中其实不太容易，不太确定的事花太多时间划不来，老梁花了十来分钟拟了个提纲发给了他。



过了几天反馈说被选中了，让继续细化提纲、突出难点、听众受益点，就这么稀里糊涂的被选中去参加了这个大会。



江湖很大，圈子很小，来去匆匆少了联系，再见依旧是故人。在这个过程中也跟业内优秀的专家们建立一些链接，特别感谢：

* 感谢敏捷教练陈文博方方面面的支持和推动
* 感谢组员李莹在PPT动画制作过程中的多次修改
* 感谢领导鹏哥（杨鹏）、益伟在准备PPT过程中的指导和提点
* 感谢大会主编薛梁和专题出品人蚂蚁资深技术专家黄挺（鲁直）的邀请



老梁参加大会主要两部分，第一天是一个现场分享、第二天是一个视频录制。下面是参加大会分享的内容，其实也没啥，大伙随便看看。





# 内容



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210504130210.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210504130355.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210504130412.png)



![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210504130442.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210504130510.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210504130529.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210504130550.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210504130608.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210504130625.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210504130641.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210504130701.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210504130718.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210504130742.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210504130804.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210504130837.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210504130930.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210504130950.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210504131010.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210504131029.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210504131044.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210504131106.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210504131126.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210504131144.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210504131201.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210504131219.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210504131235.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210504131303.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210504131347.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210504131403.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210504131422.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210504131505.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210504131524.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210504131539.png)

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210504183140.png)

