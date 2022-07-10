---
title: ES09# Filebeat配置项以及吞吐调优项梳理
categories: filebeat
tags: filebeat
date: 2022-06-25 11:55:01
---



# 引言

公司有使用filebeat作为日志采集的agent，然而最近发现其在一些node采集吞吐不足，现就其配置项与吞吐调优进行梳理。本文的主要内容有：

* Input输入配置项
* 通用以及全局配置项
* Output输出配置项



# 一、Input输入配置项

Filebeat支持众多的Inputs，以日志文本类为例梳理其配置项，主要配置项如下：

| 配置项                          | 说明                                                         |
| ------------------------------- | ------------------------------------------------------------ |
| type                            | 取值log或者filestream，7.16.0以后log声明为废弃               |
| enabled                         | 是否开启，默认关闭                                           |
| paths                           | 抓取的日志文件配置，例如：- /var/log/*.log                   |
| encoding                        | 读取使用的编码，默认为plain，可选utf-8、gbk等                |
| exclude_lines                   | 读取文件时丢掉哪些行，默认没有丢弃。例如：['^DBG'] 排除以DBG开头的行 |
| include_lines                   | 指定需要读取的行，默认所有行均会读取。例如：['^ERR', '^WARN']读取以ERR和WARN开头的行 |
| exclude_files                   | 排除哪些文件不采集，例如： ['.gz$']排除.gz结尾的文件         |
| file_identity.native            | 判断两个文件是否相同，默认使用inode和device id               |
| fields                          | 在输出的每条日志增加额外的信息，默认会在fields新建子目录     |
| fields_under_root               | 表示新增的字段fields为顶级目录                               |
| keep_null                       | 是否在事件中发布具有null的字段，默认false                    |
| publisher_pipeline.disable_host | 是否禁止设置host.name，默认false                             |
| ignore_older                    | 超过指定时间段未更新的文件将被忽略，例如：2h，日志文件修改时间超过2h将被filebeat忽略；默认为0，不忽略任何文件 |
| scan_frequency                  | 监测新文件产生的频率，默认为10s                              |
| harvester_buffer_size           | 单个文件采集器harvester每次使用缓存区的大小，也就是读取文件的大小；默认为16KB；提高吞吐的调优项 |
| max_bytes                       | 限制一条日志的大小，超出部分将被丢弃，默认为10M              |
| line_terminator                 | 行的分割符，默认auto                                         |
| recursive_glob.enabled          | 扩展"**"的文件递归模式，默认开启                             |
| json.message_key                | 可选设置，用于在行过滤和多行合并时指定json key，需json对象中顶层字符串 |
| json.keys_under_root            | 默认false，json解码后以”json“为key，设置为true，该key将被设置为顶级 |
| json.overwrite_keys             | 默认false，设置为true，keys_under_root开启的情况下，解码后的json字段将覆盖Filebeat字段 |
| json.expand_keys                | 默认false，设置为true递归去点。例如：{"a.b.c": 123}转换为{"a":{"b":{"c":123}}} |
| json.add_error_key              | 默认false，设置为true，如果json编译失败将添加错误key"error.message" 和 "error.key: json"。 |
| multiline.pattern               | 多行合并可以讲堆栈信息合并成一条发送，此配置未多行合并正则表达式。例如：'^[[:space:]]' 将空格开头的合并发送 |
| multiline.negate                | 默认false，是否定义否定模式，上面的正则表达式语义相反        |
| multiline.match                 | 默认after，多行合并一行事件的模式。可选after和before         |
| multiline.max_lines             | 多行合并中的最大行数，超过该设置将被丢弃。默认为500          |
| multiline.timeout               | 多行合并模式匹配中，一次合并的超时时间，默认为5秒            |
| tail_files                      | 默认false从头读取新文件，设置为true从尾部读取新文件          |
| symlinks                        | 默认false，不处理常规文件的符号链接。                        |
| backoff                         | 默认1秒，Filebeat检测到EOF后，再次检查文件时的等待时间       |
| max_backoff                     | 默认10秒，Filebeat检测到EOF后，再次检查文件时的等待最长时间  |
| backoff_factor                  | 默认2，等待时间系数，表示每次等待时间是上一次的两倍，最长默认为10秒 |
| harvester_limit                 | 默认0，没有限制。用于限制一个input中harvester的启动数量      |
| close_eof                       | 默认false，当读到文件末尾harvester会继续工作不关闭，true表示读到文件末尾后结束 |
| close_inactive                  | 当close_eof为false时有效，表示多长时间没消息时harvester退出  |
| close_renamed                   | 默认false，文件更名（日志文件轮替）时不退出                  |
| close_removed                   | 默认true，表示文件被删除时harvester停止工作                  |
| clean_inactive                  | 默认0，被禁用。当文件修改时间超过clean_inactive，registry的state将被移除 |
| clean_removed                   | 默认true，从registry移除不存在的日志文件                     |
| close_timeout                   | 默认0，不限制。harvester每次读取文件的超时时间。             |



备注：当filebeat性能不足时可以通过调优harvester_buffer_size的大小来提高读取日志的能力，需要指定不同的文件，可以定义多个input。



# 二、通用以及全局配置项

| 配置项                             | 说明                                                         |
| ---------------------------------- | ------------------------------------------------------------ |
| filebeat.registry.path             | Registry数据存储路径，默认${path.data}/registry              |
| filebeat.registry.file_permissions | registry文件权限，默认0600                                   |
| filebeat.registry.flush            | 控制registry entries刷盘的频率，默认为1秒                    |
| filebeat.shutdown_timeout          | 默认为0，不等待。filebeat等待publisher关闭的时长             |
| name                               | filebeat指定名字，默认为hostname                             |
| tags                               | 给每条日志加标签，便于过滤                                   |
| fields                             | 给每条日志加字段，保存在fields字段中                         |
| fields_under_root                  | 默认false，是否将fields的字段保存为顶级字段                  |
| timestamp.precision                | filebeat时间戳精度，默认millisecond                          |
| queue                              | 存储事件的内部缓存队列，当队列中事件达到最大值，input将不能想queue中写入数据，直到output将数据从队列拿出去消费。 |
| mem.events                         | 内部缓存队列queue最大事件数，默认为4096                      |
| flush.min_events                   | queue中的最小事件，达到后将被发送给output，默认为2048        |
| flush.timeout                      | 定时刷新queue中的事件到output中，默认为1s                    |



备注：调整mem.events、flush.min_events、flush.timeout，增加内存，牺牲一些实时性，可提高吞吐。



# 三、Output输出配置项

filebeat支持众多不同的output作为目标输出源，本文以kafka为例梳理其配置项。

| 配置项                     | 说明                                                         |
| -------------------------- | ------------------------------------------------------------ |
| output.kafka               | 输出类型为kafka                                              |
| hosts                      | kafka集群broker地址                                          |
| topic                      | 用于生成事件的kafka主题                                      |
| key                        | kafka的事件key，需唯一，默认不生成                           |
| partition.hash             | 发送到kafka分区的策略，默认通过key has，未设置key则随机      |
| reachable_only             | 默认false，设置为true则发送到kafka 可用的 leaders分区        |
| metadata.retry.max         | Leader选举元数据请求的重试最大次数                           |
| metadata.retry.backoff     | Leader选举期间重试的时间间隔，默认为250ms                    |
| metadata.refresh_frequency | 元数据刷新频率，默认为10分钟                                 |
| worker                     | 并发负载均衡 Kafka output workers的数量，默认为1             |
| max_retries                | 发布失败后的重试次数，默认为3                                |
| backoff.init               | 当发送kafka发生网络错误，经过多久重新发送，默认1秒           |
| backoff.max                | 发生网络错误后会重试，每次递增直到最大值后丢弃，默认最大值为60s |
| bulk_max_size              | 单次kafka request请求批量的消息数量，默认2048                |
| bulk_flush_frequency       | 批量发送kafka request需要等待的时间，默认0不等待，与linger.ms功能相同 |
| timeout                    | 等待broker返回的超时时间，默认30s                            |
| required_acks              | kafka broker的确认机制，1:leader确认，0:无需确认，-1:所有broker确认 |



备注：降低发送到broker频率，提高一次发送的数量，通过bulk_max_size、bulk_flush_frequency以及required_acks可以调优发送到kafka的吞吐。
