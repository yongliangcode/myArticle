### Message



属性：用户自定义属性的键值对（可选）



TypedMessageBuilder，最佳的选择是将 key 设置为字符串，key 设置为其他类型（例如，AVRO 对象），则 key 会以字节形式发送，这时 consumer 就很难使用了。



### Producer

发送 分为同步

异步（发送队列可以配置）



压缩：压缩后性能提高10倍 支持四种类型 LZ4、ZLIB、ZSTD、SNAPPY



**批量发送** 

发送：累计到一批发送，由最大消息数和最大发布延迟定义

消费：确认了一批消息后才算确认成功，其中部分否定的确认以及不可预料的失败，导致消息会重新发送



分块：需要禁止批量发送，一条消息可能被分成不同的块进行发送，ledger里面可能不是连续的；这种方式给内存带来了一定的负担。





### Consuemr



Consumer向Broker发送消息获取申请，consumer有个队列用于接收broker发送来的消息，receiverQueueSize控制接收队列的大小。`consumer.receive()` 被调用一次，就从缓冲区（buffer）获取一条消息。

接收消息分为同步接收receive和异步接收receiveAsync



**监听**

接收到消息自动回调MessageListener

```java
 pulsarClient.newConsumer() //
                .topic("persistent://my-tenant/my-ns/my-topic") //
                .subscriptionName("my-subscription-name") //
                .messageListener((consumer, msg) -> {
                    log.info("Received message: {}", msg);
                    consumer.acknowledgeAsync(msg);
							 }).subscribe();

```



**确认** 

只有订阅的消息全部确认后才会删除，可以通过set-message-ttl设置过期时间

累计确认和单条确认：在共享订阅模式下，消息都是单条确认模式



**取消确认**

某条消息没有成功消费，可以发送取消请求到broker，broker重新推送该消息。

在独占消费模式和灾备订阅模式中，消费者仅仅只能对收到的最后一条消息进行取消确认，共享模式下可以单条取消确认。



### 死信队列



消费失败的进入死信队里



### 重试队列



指定重试时间和重试主题



### 主题



命名规范

```
{persistent|non-persistent}://tenant/namespace/topic

持久化/非持久化
租户
命名空间
topic
```



### 订阅

独占：默认订阅模式，至于一个consumer订阅。

共享：消息会被发送到一个消费者去类跟Kafka/RocketMQ集群消费一样

灾备：一个消费者挂了，另一个消费者接管

key共享：相同key会发送到同一个消费者消费，也就是顺序消费



### 订阅多个主题

* 支持正则表达式订阅多个主题
* 制定明确的Topic列表



### 分区主题



* 普通主题也就是单个主题，被保存在单个broker中，这个会影响吞吐量

* 分区主题，通过N个内部主题，N的数量等于分区的数量，每个分区隶属于集群的一个broker

  分区的分布式pulsar自动处理的；由路由模式决定消息发送到哪个分区；订阅模式决定消息发送到哪个consumer

  分区数在用admin api创建的时候指定

  

### 路由模式



* RoundRobinPartition  没有key时循环轮转round-robin发送；如果有key则会根据key进行取hash路由到不同的分区中
* SinglePartition 消息被发送到单个分区；如果没有指定key，所有消息被随记发送一个分区；如果指定key会根据key分散到不同的分区分钟
* CustomPartition 自定义路由规则用户自定义实现消息该进入什么分区



### 顺序保证

* 分区有序  使用SinglePartition和RoundRobinPartition策略，设置key即可，会根据key的hash路由
* 全局有序  同一个发送的消费者都是有序的，使用SinglePartition策略不要设置key



### 散列scheme

两种散列函数JavaStringHash和Murmur3_32Hash，默认为JavaStringHash，多语言客户端时建议Murmur3_32Hash。



### 非持久topic

只保存在内存中，不落盘，broker进程消失内存消息也会随之丢失

```
non-persistent://tenant/namespace/topic
```





### 消息保留和过期

* 默认策略 立即删除被消费者确认的消息，以backlog的形式持久化保存所有未被确认的消息

* 可以通过TTL（time to live）在指定的时间内不被确认的话会自动确认

* 可以通过保留策略覆盖默认策略 通过set-retention命令设置，通过get-retention获取保留策略

  示例一

  ```
  $ pulsar-admin namespaces set-retention my-tenant/my-ns \
    --size 10G \
    --time 3h
  ```

  说明：对命名空间进行设置，包含其下面的每个主题。

  1.3小时内当topic的存储的消息超过10G时，已经被consumer确认的消息不再保存

  2.超过3小时即使消息大小小于10G，已经被consumer确认的消息也会被删除

  

  示例二

  ```
  $ pulsar-admin namespaces set-retention my-tenant/my-ns \
    --size 1T \
    --time -1
  ```

  说明：只设置消息存储大小；对存储时间没有限制


  示例三

  ```
  $ pulsar-admin namespaces set-retention my-tenant/my-ns \
    --size -1 \
    --time 3h
  ```

  说明：只设置了存储时间；对存储消息大小没有限制

  

### 消息去重

消息去重保证一条消息在pulsar中存储一次，比如开启了去重功能，发送端重复发送后，重复消息将不会保存在bookeeper中。



### 生产中幂等

也就是发送时消息去重



### 延迟消息

在消费组Shared subscription mode下，延迟消息才有效。在未来一段时间消费的消息









































