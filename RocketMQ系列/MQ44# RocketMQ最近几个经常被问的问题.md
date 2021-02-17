---
title: MQ44# RocketMQ最近几个经常被问的问题
categories: RocketMQ
tags: RocketMQ
date: 2021-01-25 20:55:01
---



# 常见问题一

**问：** RocketMQ消费者订阅了tag，但却收不到消息无法消费，并且根据 msgid 去查询，发现这条消息的状态为 CONSUMED_BUT_FILTERED，那这是为什么？

**答：** 在RocketMQ中，一个消费组能同时订阅多个 tag，但一个消费组的不同消费者不能分开订阅不同的tag，即同一个消费组的订阅关系必须保持一样。例如：常见错误使用方式同一个项目中，一段消费代码订阅tagA，然后拷贝到这段代码再更改为tagB。



<!--more-->

**正确用法** 

```java
public void subscribe(){
	DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("melon_online_test_consumer");
	consumer.subscribe("melon_online_test","tag1 || tag2 || tag3");
}  
```

**错误用法**

```java
public class SubscribeTest {
  public void subscribeA(){
    DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("melon_online_test_consumer");
    consumer.subscribe("melon_online_test","tag1");
  } 

  public void subscribeB(){
    DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("melon_online_test_consumer");
    consumer.subscribe("melon_online_test","tag2");
  } 
}
```





# 常见问题二

**问：** 发现大量的RocketMQ client 大量的info日志输出，我不关心，如何禁用呢？

![](https://gitee.com/laoliangcode/md-picture/raw/master/img/20210125093344.png)

**答：** 尝试以下设置，项目中使用了Slf4j
@1 可以配置RocketmqClient的logger设置优先级为warn

@2 也可以通过-Drocketmq.client.logUseSlf4j=false 和 -Drocketmq.client.logLevel=WARN 关闭MQ客户端使用Slf4j并提高日志等级

项目中没有使用Slf4j，可以通过-Drocketmq.client.logLevel=WARN调高日志等级。





#  常见问题三

**问：** 我的服务消费后需要调用第三方接口，别人的接口调用有限制，Rocketmq消费可以限流吗？

**答：** RocketMQ本身没有类似每秒消费多少条数据的精确限流，我们可以结合Sentienl来实现，示例代码如下：

```java
    private String KEY = "melon_topic:melon_consumer"; // 资源名称由topic和消费组构成
    
    public static void main(String[] args) throws InterruptedException, MQClientException {
        initFlowControlRule(); // Sentinel流控规则
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("melon_consumer");
        consumer.setNamesrvAddr("localhost:9876");
        consumer.subscribe("melon_topic", "*");
        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
        consumer.registerMessageListener(new MessageListenerConcurrently() {
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
                for (MessageExt msg : msgs) {
                    Entry entry = null;
                    try {
                        ContextUtil.enter(KEY); // 定义资源
                        entry = SphU.entry(KEY, EntryType.OUT);
                        System.out.printf("%s Receive New Messages: %s %n", Thread.currentThread().getName(), msg);
                    } catch (BlockException ex) {
                        // Blocked.被限流后消息重试
                        return ConsumeConcurrentlyStatus.RECONSUME_LATER;
                    } finally {
                        if (entry != null) {
                            entry.exit();
                        }
                        ContextUtil.exit();
                    }
                }
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
        consumer.start();
        System.out.printf("Consumer Started.%n");
    }

    private static void initFlowControlRule() {
        FlowRule rule = new FlowRule();
        rule.setResource(KEY);
        rule.setCount(5);// 每秒通过5条消息
        rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
        rule.setControlBehavior(RuleConstant.CONTROL_BEHAVIOR_RATE_LIMITER);
        rule.setMaxQueueingTimeMs(5 * 1000); // 排队超时时间5秒
        FlowRuleManager.loadRules(Collections.singletonList(rule));
    }

```



<!--more-->



#  常见问题四

**问：**RocketMQ默认延迟等级有18个，我可以扩增吗？

**答：** 可以的，但是不建议扩增太多等级，可以通过修改broker属性messageDelayLevel来实现，注意修改了后需要重启broker。例如：

```java
messageDelayLevel = 1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h 1d 3d 7d 14d 21d
```







