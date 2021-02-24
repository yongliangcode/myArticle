---
title: Kafka6# 常用JMX监控指标整理
categories: Kafka
tags: Kafka
date: 2019-05-26 11:55:01
---



本文盘点下到Kafka 2.4.1版本以来的一些亮点，这些亮点或笔者实际中踩过的坑、或可能将来会在实践中使用、或个人关注的，点击官方发布日志连接查看全貌。



# 0.11.0.3



0.11.0.2于2017年11月17日发布；0.11.0.3于2018年6月2日发布修订版本。

其中修复了0.11.0.2以前的一个BUG，该Bug曾导致过生产事故；即堆内存不能正常回收，频繁Full GC。详见：Kafka(0.11.0.2版本)堆内存不能正常回收问题分析【实战笔记】[[KAFKA-6307](https://issues.apache.org/jira/browse/KAFKA-6307)]

[0.11.0.3官方发布日志](https://archive.apache.org/dist/kafka/0.11.0.3/RELEASE_NOTES.html)



<!--more-->



# 1.0.0



1.0.0于2017年11月1日发布；1.0.1于2018年3月5日发布；1.0.2于2018年6月8日发布。

增强各个组件的稳定性。可以容忍JBOD磁盘故障，故障时不再导致broker崩溃，会保留可用磁盘上的日志文件。[[KAFKA-4763]](https://issues.apache.org/jira/browse/KAFKA-4763)

幂等生产者或者我们要保证消息顺序性时需要设置max.in.flight.requests.per.connection=1；1.0.0之后可以最大设置为5，从而提升投递性能。[[KAFKA-5494]](https://issues.apache.org/jira/browse/KAFKA-5494)

[1.0.0官方发布日志](https://archive.apache.org/dist/kafka/1.0.0/RELEASE_NOTES.html)

[1.0.1官方发布日志](https://archive.apache.org/dist/kafka/1.0.1/RELEASE_NOTES.html)

[1.0.2官方发布日志](https://archive.apache.org/dist/kafka/1.0.2/RELEASE_NOTES.html)



# 1.1.0



1.1.0于2018年3月28日发布；1.1.1于2018年6月19日发布

1.1.0通过将同步方式修改为异步方式，提升了KafkaI使Controller的shutdown速度；由于Controller性能的改进促使集群可以支持20万个分区。[[KIP-227]](https://cwiki.apache.org/confluence/display/KAFKA/KIP-227%3A+Introduce+Incremental+FetchRequests+to+Increase+Partition+Scalability?S_TACT=M16103KW)

[Apache Kafka Supports 200K Partitions Per Cluster](https://www.confluent.io/blog/apache-kafka-supports-200k-partitions-per-cluster/)

[Apache Kafka支持单集群20万分区](https://www.cnblogs.com/huxi2b/p/9984523.html)

增加了对单broker日志目录之间的数据迁移，例如：一个broker下挂了多个磁盘，当各个分区出现不均衡时，1.1.0之后支持该broker磁盘将分区迁移实现数据均衡。[[KAFKA-5163]](https://issues.apache.org/jira/browse/KAFKA-5163)

[1.1.0官方发布日志](https://archive.apache.org/dist/kafka/1.1.0/RELEASE_NOTES.html)

[1.1.1官方发布日志](https://archive.apache.org/dist/kafka/1.1.1/RELEASE_NOTES.html)



# 2.0.0



2.0.0于2018年6月30日发布；2.0.1于2018年11月9日发布；增加了主题前缀或通配符的ACL的支持，从而简化了大型安全部署中的访问控制管理。[[KAFKA-6841]](https://issues.apache.org/jira/browse/KAFKA-6841)

支持OAuth 2.0认证[[KAFKA-6562]](https://issues.apache.org/jira/browse/KAFKA-6562)

[2.0.0官方发布日志](https://archive.apache.org/dist/kafka/2.0.0/RELEASE_NOTES.html)

[2.0.1官方发布日志](https://archive.apache.org/dist/kafka/2.0.1/RELEASE_NOTES.html)



# 2.1.0



2.1.0于2018年11月20日发布；2.1.1于2019年2月15日发布。

支持Zstandard压缩算法

[[KAFKA-4514]](https://issues.apache.org/jira/browse/KAFKA-4514)

[2.1.0官方发布日志](https://archive.apache.org/dist/kafka/2.1.0/RELEASE_NOTES.html)

[2.1.1官方发布日志](https://archive.apache.org/dist/kafka/2.1.1/RELEASE_NOTES.html)



# 2.2.0



2.2.0于2019年3月22日发布；2.2.1于2019年6月1日发布；2.2.2于2019年12月1日发布。

改进消费组管理，默认group.id为null，以前为空字符串。[[KAFKA-6774]](https://issues.apache.org/jira/browse/KAFKA-6774)

[2.2.0官方发布日志](https://archive.apache.org/dist/kafka/2.2.0/RELEASE_NOTES.html)

[2.2.1官方发布日志](https://archive.apache.org/dist/kafka/2.2.1/RELEASE_NOTES.html)

[2.2.2官方发布日志](https://www.apache.org/dist/kafka/2.2.2/RELEASE_NOTES.html)



# 2.3.0



2.3.0于2019年6月25日发布；2.3.1于2019年10月24日发布。

提供命令查看哪些topic的分区小于最小ISR的数量。[[KAFKA-7236]](https://issues.apache.org/jira/browse/KAFKA-7236)

[2.3.0官方发布日志](https://www.apache.org/dist/kafka/2.3.0/RELEASE_NOTES.html)

[2.3.1官方发布日志](https://www.apache.org/dist/kafka/2.3.1/RELEASE_NOTES.html)



# 2.4.0



2.4.0于2019年12月16日发布；2.4.1于2020年3月12日发布。

允许消费者从最近的副本(follower)获取数据 [[KAFKA-8443]](https://issues.apache.org/jira/browse/KAFKA-8443)

跨机房数据同步引擎MirrorMaker 2.0 [[KAFKA-7500]](https://issues.apache.org/jira/browse/KAFKA-7500)

升级ZooKeeper到3.5.7该版本fix了21个issue [[KAFKA-9515]](https://issues.apache.org/jira/browse/KAFKA-9515)

[2.4.0官方发布日志](https://www.apache.org/dist/kafka/2.4.0/RELEASE_NOTES.html)

[2.4.1官方发布日志](https://www.apache.org/dist/kafka/2.4.1/RELEASE_NOTES.html)