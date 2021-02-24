---
title: Kafka4# 从0.11.0.2升级到2.2.1
categories: Kafka
tags: Kafka
date: 2019-05-18 11:55:01
---



# 升级记录

消息格式没有变化，只需要更改borker版本即可。

下载2.2.1客户端，保持原有节点配置不变增加设置0.11版本，逐台重启机器，在流入流出正常后再重启下一台，重要客户端做好观察。

inter.broker.protocol.version=0.11.0

全部正常后，再设置broker版本到2.2，逐台重启机器，在流入流出正常后再重启下一台，重要客户端做好观察。

inter.broker.protocol.version=2.2



<!--more-->



# 注意事项

在客户端版本使用0.9或者更低版本时会出现以下错误，升级后异常可解决。

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210224101154.png)



# 官方升级文档

https://kafka.apache.org/22/documentation.html#upgrade



1.5 Upgrading From Previous Versions

Upgrading from 0.8.x, 0.9.x, 0.10.0.x, 0.10.1.x, 0.10.2.x, 0.11.0.x, 1.0.x, 1.1.x, 2.0.x or 2.1.x to 2.2.0

If you are upgrading from a version prior to 2.1.x, please see the note below about the change to the schema used to store consumer offsets. Once you have changed the inter.broker.protocol.version to the latest version, it will not be possible to downgrade to a version prior to 2.1.

For a rolling upgrade:

Update server.properties on all brokers and add the following properties. CURRENT_KAFKA_VERSION refers to the version you are upgrading from. CURRENT_MESSAGE_FORMAT_VERSION refers to the message format version currently in use. If you have previously overridden the message format version, you should keep its current value. Alternatively, if you are upgrading from a version prior to 0.11.0.x, then CURRENT_MESSAGE_FORMAT_VERSION should be set to match CURRENT_KAFKA_VERSION.

inter.broker.protocol.version=CURRENT_KAFKA_VERSION (e.g. 0.8.2, 0.9.0, 0.10.0, 0.10.1, 0.10.2, 0.11.0, 1.0, 1.1).

log.message.format.version=CURRENT_MESSAGE_FORMAT_VERSION (See potential performance impact following the upgrade for the details on what this configuration does.)

If you are upgrading from 0.11.0.x, 1.0.x, 1.1.x, or 2.0.x and you have not overridden the message format, then you only need to override the inter-broker protocol version.

inter.broker.protocol.version=CURRENT_KAFKA_VERSION (0.11.0, 1.0, 1.1, 2.0).

Upgrade the brokers one at a time: shut down the broker, update the code, and restart it. Once you have done so, the brokers will be running the latest version and you can verify that the cluster's behavior and performance meets expectations. It is still possible to downgrade at this point if there are any problems.

Once the cluster's behavior and performance has been verified, bump the protocol version by editing inter.broker.protocol.version and setting it to 2.2.

Restart the brokers one by one for the new protocol version to take effect. Once the brokers begin using the latest protocol version, it will no longer be possible to downgrade the cluster to an older version.

If you have overridden the message format version as instructed above, then you need to do one more rolling restart to upgrade it to its latest version. Once all (or most) consumers have been upgraded to 0.11.0 or later, change log.message.format.version to 2.2 on each broker and restart them one by one. Note that the older Scala clients, which are no longer maintained, do not support the message format introduced in 0.11, so to avoid conversion costs (or to take advantage of exactly once semantics), the newer Java clients must be used.