---
title: Pulsar#1 Pulsar部署与线上配置
categories: Pulsar
tags: Pulsar
date: 2022-02-24 11:55:01
---



# 引言





# 引言

{persistent|non-persistent}://tenant/namespace/topic

* persistent|non-persistent 是否为持久主题，非持久主题不落盘，只在缓存中存储，broker重启或者宕机消息丢失
* tenant 租户逻辑隔离，可以是不同组织、一个组内内的不同部门、组织内的不同业务形态
* namespace 命名空间，一个租户下可以有多个命名空间，用于更细粒度隔离
* topic 主题名称，分为分区主题和普通主题，分区主题是一个逻辑概念，由一组主题组成，每个主题隶属于一个broker



```
bin/pulsar-admin namespaces create public/melon-namespace --clusters pulsar-cluster-1 --bundles 16
```



```
bin/pulsar-admin topics create-partitioned-topic --partitions 16 public/melon-namespace/melon-topic
```



```java
@Test
public void test01() throws Exception {

  String serviceHttpUrl = "http://127.0.0.1:8080";

  PulsarAdmin admin = PulsarAdmin.builder().serviceHttpUrl(serviceHttpUrl).build();

  final String partitionedTopic = "public/melon-namespace/melon-topic-test02";

  admin.topics().createPartitionedTopic(partitionedTopic, 3);

}
```



PulsarAdminImpl类

通过Jersey创建REST的客户端



Client请求调用链：

```
Topics类createPartitionedTopic


org.apache.pulsar.client.admin.internal.BaseResource#asyncPutRequest


javax.ws.rs.client.WebTarget#request

```





服务端处理过程，Jersey构建REST服务端



JerseyWebTarget {



 http://127.0.0.1:8080/admin/v2/persistent/public/melon-namespace/melon-topic-test02/partitions 



}



addServlet



org.apache.pulsar.broker.web.WebService



```
org.apache.pulsar.broker.admin.v2.PersistentTopics#createPartitionedTopic
```









```
/managed-ledgers/
```

```java
public CompletableFuture<Void> createPersistentTopicAsync(TopicName topic) {
  String path = MANAGED_LEDGER_PATH + "/" + topic.getPersistenceNamingEncoding();;
  return store.put(path, new byte[0], Optional.of(-1l))
    .thenApply(__ -> null);
}
```



ZKMetadataStore























