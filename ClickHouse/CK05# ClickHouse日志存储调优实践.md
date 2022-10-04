



# 引言



随着生产环境日志查询切换到ClickHouse，原先的ElasticSearch集群释放回收也进入尾声。本文对过去一段时间的实践做一个复盘整理，主要内容有：



- 重要术语特性说明
- 分布式表读写原理
- 日志存储调优实践
- 文章要点回顾总结



# 一、重要术语特性说明



研究一个组件通常从其约定的术语、有哪些吸引人的特性开始，本节梳理了ClickHouse重要术语和亮眼的特性。



### **1、重要性能原因**



众多的设计和优化成就了ClickHouse的高性能，下面列举几个比较突出的点：

特性说明列式存储数据按列组织，同一列的数据保存在一起，不同的列分不同的文件保存压缩算法默认使用LZ4压缩算法，压缩比与数据相关，压缩比1:4~1:8不等向量化执行引擎1、利用CPU的SIMD（Single Instruction Multiple Data）单条指令操作多条数据

2、在寄存器层面实现数据并行执行，寄存器访问数据的速度是内存的300倍，是磁盘的3000万倍众多表引擎1、提供近30种的表引擎供选择，选择表表引擎意味着选择了不同的存储查询方式

2、MergeTree系列为官方主流系列



小结：在寄存器层面实现数据并行执行，SIMD大量用于文本转换、数据过滤、数据解压以及JSON转换等场景。





### **2、重要术语释义**



下表就ClickHouse的多主架构、数据分片、数据副本进行说明，为下文分布式表读写原理奠定基础。

术语说明多主架构1、ClickHouse采用多主架构，而不是主从架构

2、意味着不像ElasticSearch有Master、Data、Coordinating等角色的区分

3、访问中集群中的任何节点均可获得相同的结果数据副本1、Clickhouse的副本其他组件并无差异，多一分相同的冗余数据

2、副本是表级别的，创建表时需要使用ReplicatedMergeTree系列引擎

3、基于多主架构通过zookeeper将执行语句分发到副本本地执行数据分片1、ClickHouse集群中每个节点称为分片

2、可通过集群借助于ZooKeeper执行分布式DDL语句CREATE/DROP TABLE ON CLUSTER my_cluster

3、数据通过本地表（可以使用_local后缀命名）存储，使用Distributed以外的引擎

4、分布式表不存储数据，为本地表的代理，类似于分库分表组件，需使用Distributed引擎

5、分片规则需要声明分片键，否则分布式表中只包含一个分片，失去分片的意义

#### 

小结：数据切片在中间件设计提高吞吐的重要实现方式，数据副本是保障可靠性的重要实现方式；当然clickhouse也不例外。



### **3、MergeTree系列表引擎**



选择什么样的表引擎意味着选择了不同的数据存储组织方式，ClickHouse中有合并树、外部存储、内存、文件、接口与其他六个系列引擎，其中MergeTree合并树系列为其核心引擎。



合并树表引擎描述MergeTree1、MergeTree的基础引擎，该系列的其他引擎继承了其能力 

2、具备数据分区、一级索引、二级索引、数据TTL一级存储能力ReplacingMergeTree1、具备删除本分区重复数据的能力 

2、通过ORDER BY排序键判断数据是否重复 

3、在分区合并的时候删除本分区重复数据，跨分区无法删除重复数据 

4、手动执行分区合并消耗大量时间SummingMergeTree1、合并分区时按照定义条件合并汇总数据，降低查询开销 

2、通过ORDER BY排序键作为聚合条件 

3、数据的合并和汇总在分区合并时进行，跨分区不会汇总合并AggregatingMergeTree1、SummingMergeTree的升级版 

2、根据ORDER BY排序键聚合数据，并写入表中，本分区相同数据合并 

3、在分区合并的时候执行聚合计算，跨分区不计算CollapsingMergeTree1、折叠合并树通过增加不同sign标志的数据代替删除的方式，实现行数据的修改与删除

2、在合并分区的时候触发 

3、对写入的数据有严格的顺序要求VersionedCollapsingMergeTree1、与CollapsingMergeTree作用相同通过对数据折叠，完成数据的删除与修改 

2、通过标志位sign与版本号ver共同完成数据折叠 

3、对写入的数据没有顺序要求，内部通过ver倒序判断



小结：基于MergeTree衍生引擎提供删除重复数据、汇总聚合、删除与修改的能力，然而他们只适合特定的场景，都在分区合并的执行，不支持跨分区。





# 二、分布式表读写原理



如果说术语和特性是进入一个组件的前置条件，那对该组件主要基本原理的理解和掌握是用好、指导实践的前提。本小节就分布式表分片算法规则、写入基本流程、读出数据流程、非分布式表写入本地表的基本原理进行阐述。



### **1、分布式表分片算法规则**



使用分布式表时，数据应该落到哪个分片节点上呢？ClickHouse有一套自己的分片算法，下面从概念入手一探究竟。



**分片键（sharding_key）**要求返回一个整数类型的取值，下面语法中sharding_key需整数类型

```SQL
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]

(

    name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1],

    name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2],

    ...

) ENGINE = Distributed(cluster, database, table[, sharding_key[, policy_name]])

[SETTINGS name=value, ...]
```



**权重（Weight）**ClickHouse中一个节点一个分片，可以给分片配置权重，权重越大数据分配越多，默认权重为1

**槽（Slot）**槽的数量为集群中所有分片的权重之和

![image-20220831193139381](/Users/admin/Library/Application Support/typora-user-images/image-20220831193139381.png)





小结：通过权重与槽联合使用的一种简单分片算法。



### **2、分布式表写入基本流程**



在使用ClickHouse分布式表写入数据时，大体流程如下图：

![image-20220831193151092](/Users/admin/Library/Application Support/typora-user-images/image-20220831193151092.png)





**流程一** 数据先写入一个分片（例如：分片1）



**流程二** 属于本分片的数据写入本地表，属于其他分片的数据先写入本分片的临时目录

例如：其他分片的数据先写入分片1的临时目录



**流程三**  该分片与集群中其他分片建立连接

例如：分片1与分片2、分片1与分片3建立连接



**流程四**  将写入本地临时文件的数据异步发送到其他分片

例如：分片1将临时目录数据发送到分片2与分片3



小结：使用分布式表直接写入数据时，集群中各个节点彼此会建立连接，彼此之间传输数据；性能低于直接写入本地表。



### 3、**分布式表读出数据流程**



使用ClickHouse的分布式表查询，大体流程如下图：

![image-20220831193203168](/Users/admin/Library/Application Support/typora-user-images/image-20220831193203168.png)









**流程一**  集群多副本时根据负载均衡选择一个副本，也就是说副本是可以承担查询功能的



**流程二**  将分布式查询语句转换为本地查询语句



**流程三**  将本地查询语句发送到各个分片节点执行查询



**流程四**  再将返回的结果执行合并



小结：分布式表的查询类似于分库分表中间件，逻辑也很类似。



### 4、**非分布式表写入本地表**



写入时通过分布式表，由于先写入本地临时目录，集群节点之间会有数据传输。那如果再写入时直接写入本地表，性能要高于通过分布式表。

![image-20220831193213903](/Users/admin/Library/Application Support/typora-user-images/image-20220831193213903.png)





小结：负载均衡的方式：一种实现分片节点前挂载SLB等负载均衡，需注意带宽限制；另一种实现则可以在客户端写入时轮训与各个分片建立连接，在客户端进行负载均衡选择分片。





# 三、日志存储调优实践



知道了术语和基本原理只是第一步，在实战中用好、把事给办了还有很长的路。本节就实战中的一些调优点进行了归纳，涵盖数据的冷热存储、数据迁移与删除、集群规划与调优、表设计调优点等内容。



### 1、**数据的冷热存储**



公司所有的应用存储日志时长统一设置固定存储时长，比如：1个月、2个月。这种策略也常被公司采用，优点是整体设计简单。缺点是不能满足业务需要以及可能存在成本浪费。



有的业务场景日志存储3天，例如用户推荐、行为分析等。很多应用要求存储7天，核心链路场景却希望存储1个月。对于逆向退货场景类的却要2个月，甚至一些特殊场景要求3个月、6个月的。





根据应用设置不同的存储时长，是一个不错的方案，即能满足不同场景需求也兼顾存储成本。



用ClickHouse做日志存储，通过冷/热盘来存储数据。热盘可以使用ESSD，存储7天的数据。冷盘可以使用普通盘，存储7~30的数据，OSS存储大于30天的数据。



日志架构示意图如下：

![image-20220831193223457](/Users/admin/Library/Application Support/typora-user-images/image-20220831193223457.png)





#### **1.1 冷热存储配置** 

下面是ClickHouse配置冷热存储的配置。

```XML
<storage_configuration>

  <disks>

    <hot>

      <path>/data1/clickhouse/hot/data/</path>

      <max_data_part_size_bytes>1073741824</max_data_part_size_bytes>

    </hot>

    <cold>

      <path>/data2/clickhouse/cold/data/</path>

    </cold>

  </disks>

  <policies>

    <ttl>

      <volumes>

        <hot>

          <disk>hot</disk>

        </hot>

        <cold>

          <disk>cold</disk>

        </cold>

      </volumes>

      <move_factor>0.1</move_factor>

    </ttl>

  </policies>

</storage_configuration>
```



说明：

- 合并分区或者一次性写入的分区大小超过max_data_part_size_bytes，也会被写入到COLD卷中
- 当存储磁盘空间小**move_factor**时，默认为0.1即磁盘小于10%时，数据会自动移动到下一个磁盘volume组，也就是热盘磁盘剩余10%时，clickhouse会将热数据向冷盘迁移。



#### **1.2 命令使用说明** 

通过下面命令查看冷热存储策略以及对应的磁盘空间和剩余空间。

```SQL
SELECT name, path, formatReadableSize(free_space) AS free, formatReadableSize(total_space) AS total, formatReadableSize(keep_free_space) AS reserved FROM system.disks;
```

![image-20220831193242473](/Users/admin/Library/Application Support/typora-user-images/image-20220831193242473.png)



通过下面命令查看max_data_part_size以及move_factor的配置。

```SQL
SELECT policy_name, volume_name, volume_priority, disks, formatReadableSize(max_data_part_size) AS max_data_part_size, move_factor FROM system.storage_policies;
```

![image-20220831193250706](/Users/admin/Library/Application Support/typora-user-images/image-20220831193250706.png)



小结：通过冷热多路径存储策略，非常适合日志的场景。



### 2、**数据迁移与删除**



尽管ClickHouse提供表级别和列级别的数据TTL迁移和数据过期策略，然而在数据迁移和删除不可避免造成磁盘IO增加，影响集群的整体性能。



那我们在业务低峰期比如凌晨两点，定时执行指定的分区迁移，能将影响降到最小。



#### 2.1 数据分区迁移



在系统表system.parts中可查看分区所在磁盘，命令如下：

```SQL
SELECT partition, name, disk_name FROM system.parts WHERE table='tb_logs_local';
```

![image-20220831193300645](/Users/admin/Library/Application Support/typora-user-images/image-20220831193300645.png)

备注：分区Part (000125d45a217a0d121f99b0cdfda94c_908003_1097483_4) 存储在hot磁盘中。



通过move命令将hot盘的分区迁移到冷盘中，命令如下：

```SQL
alter table dw_log.tb_logs_local on cluster default move partition ('xxx-clasp','xx','2022-07-01 19:00:00') to disk 'cold'
```

![image-20220831193311789](/Users/admin/Library/Application Support/typora-user-images/image-20220831193311789.png)

再次查看，该分区已被迁移到冷盘中了。

![image-20220831193319848](/Users/admin/Library/Application Support/typora-user-images/image-20220831193319848.png)





#### 2.2 数据分区删除



ClickHouse提供了drop命令，可以将分区进行删除，命令如下：

```SQL
alter table dw_log.tb_logs_local on cluster default drop partition ('scp-pink-clasp','MF7','2022-07-01 19:00:00') ;
```

执行后，再次结果查询，该分区已检索不到。

![image-20220831193329053](/Users/admin/Library/Application Support/typora-user-images/image-20220831193329053.png)





小结：在创建表时可以设置应用名称、日期为分区键，在system.parts有详细的应用以及创建日期，进而通过move/drop命令执行分区的转移和删除。



### 3、**集群规划与调优**



一个集群多少节点，节点使用什么样的配置，总共需要多少个集群。在规划时首先需要考虑的，并在实践中也需要相互验证与调整。



使用冷热分离架构，一个节点挂2T的热盘以及5T的冷盘。每个节点热盘使用SSD，冷盘使用普通盘。一个ClickHouse集群20个节点。



#### 3.1 集群规模划分



日志存储划分为几个集群，有的公司会将所有的日志存储在一个集群，根据公司情况将ClickHouse集群划分了4个，具体规模需要根据实际情况评估。



**集群1**  存储支撑域相关日志

**集群2** 存储非交易业务日志

**集群3** 存储交易相关日志

**集群4** 存储算法推荐相关日志



小结：划分为多个集群，可根据不同的业务域方便针对性的治理；一个集群有问题时方便将日志流量调度到其他日志集群应急处理。



#### 3.2 集群配置选择



为了不必要浪费和降低成本，通常会先选择一个中等偏低的配置开始，在不能满足需求时才进一步提高配置，下面就描述下在实践中尝试过程。



**配置一**  刚开始规划使用16C64G的配置，然而查询确遇到了问题：

- 测试精确查找一条日志，需要30秒
- 模糊查询一条最近5小时内的日志，需要60秒

投产需要查询耗时无明显的等待感，5秒内为宜，小部分超过5秒也可以接受，否则无法在生产环境投入使用。



**配置二**  通过与业内ClickHouse专家与公司同事反复对焦，将配置提升到48C192G。

- 精确查找一条日志，几百毫秒返回
- 布隆查询一条最近5小时内的日志，秒级返回
- 模糊查询一条最近5小时内的日志，3秒内返回

该配置基本满足了业务支撑类场景的使用，然而对于推荐算法这种高吞吐、大消息（高达20KB一条）依然不能解决。



**配置三** 针对算法推荐场景针对性治理，单条消息20KB左右，吞吐也很高。首先是升盘将SSD PL1升级到PL2，进一步提高IOPS吞吐。

- 模糊查询一条最近5小时内的日志，整个集群IPOS被打满，耗时超过30秒，无法投产



**配置四**   提高磁盘配置，将PL2升级为PL3，对应的配置也提高128C256G

- 模糊查询一条最近5小时内的日志，大部分3~5内返回
- 模糊查询一条最近1小时内的日志，大部分2内返回
- 精确查找一条日志消息，大部分1秒左右返回

该配置单条消息20KB，查询基本都在5秒内，没有特别明显的等待感，具备上线条件。



#### 3.3 集群调优参数



除了集群规模、集群配置外，集群调优也很重要，ClickHouse有几个参数需要调优，如下：


参数说明示例硬件配置硬件配置至关重要，过低配置无法投产48C192G/热盘SSD 2T/冷盘HDD 5T * 20max_threads用于控制一个用户的查询线程数可配置核数的80%，例如：40Cmax_memory_usage单个查询最多能够使用内存大小可配置内存的80%，例如：80Gmax_execution_time单个查询最大执行时间根据实际设置，例如：60查询耗时不超过1分钟skip_unavailable_shards当某一个shard无法访问时，其他shard的数据仍然可以查询例如：设置1



小结：根据不同业务场景划分多个集群，方便针对性调优治理，当然集群规模也不易过多，灵活权衡。





### 4、**表设计调优点**



表结构的设计也至关重要，笔者团队在不断尝试时，表结构推了重来次数不下5次，不断调整提高查询性能。



#### 4.1 **重视字段索引创建**



将公共约定的常用名称，独立成字段，方便添加索引，提升查询效率，例如：链路ID设置为String类型，trace_id添加索引。

```SQL
`trace_id` String,

INDEX trace_id trace_id TYPE SET(100) GRANULARITY 2,

INDEX idx_trace_id trace_id TYPE tokenbf_v1(512, 2, 0) GRANULARITY 2
```

时间类型的字段log_time添加索引。

```Ada
`log_time` DateTime64(3),

INDEX logtime log_time Type minmax GRANULARITY 2
```



小结：ClickHouse的MergeTree提供了4种类型的跳数索引（二级索引），4中类型跳数索引为：minmax、set、ngrambf_v1和tokenbf_v1，充分为字段建立索引提高查询性能，只要能提高查询性能均可尝试。



#### **4.2 合理选择分区字段**



选择什么样的分区字段是非常重要的，直接影响集群的整体性能，笔者在此过程中也反复尝试了多种方式。



**4.2.1 应用和天分区** 



是指每个应用每天一个分区，也方便各个应用的日志成本的核算和分摊，通过测试存在以下问题：

- 几百个应用意味着一天有几百个分区
- Flink在写入时导致ClickHouse的整体IOPS居高不下
- 严重时写入的IPOS占整体的30%以上，甚至50%



小结：写入占用了过多的磁盘IOPS资源，严重影响查询性能，需要将更多的CPU/IO资源留个查询。 



**4.2.2 按天设置分区**



是指一个集群的所有应用共用一个分区，每天创建一个。通过测试有效降低磁盘IOPS。



为了能够根据分摊存储成本，将消息提大小、存储时长，提成独立字段解决。 



分区字段示例

```SQL
PARTITION BY (toDate(log_time),log_save_time)  
```



小结：按天分区能解决绝大多数业务日志，然而凡事总有例外。



**4.2.3 按小时设置分区**



按天分区在业务场景能满足需求，也降低了写入的IPOS。然而在数据智能算法推荐场景，由于其日志量和消息大小均很大。



尽管将分区的迁移放在了凌晨2点之后，一个分区一个分区迁移。当一个分区迁移时把整个集群的IPOS打满，而且是持续打满。



对半夜日志排查造成严重困扰，所以决定对该集群使用按小时创建分区。

```SQL
PARTITION BY (toStartOfHour(log_time),log_save_time)  
```



小结：根据业务实际场景，灵活定制设置分区键，针对性优化提高性能。 



#### **4.3 合并树引擎与排序字段**



在生产环境使用了MergeTree，而没有采用ReplacingMergeTree。尽管日志可能重复数据，然而合并相同数据必然消耗集群性能。



一切都在取舍之中，一切让位与集群性能。



排序字段尽可能是查询的字段，充分利用主键索引。

```Visual%20Basic
ORDER BY (application, environment, log_time, ip, file_offset)
```



小结：选择合适的合并树引擎、利用好排序字段和主键索引。



#### **4.4 选择合适的压缩算法**



更强悍的压缩算法，往往需要牺牲一定的性能为代价。ClickHouse的压缩算法LZ4和ZSTD也不例外。



经测试LZ4查询响应要比ZSTD快30%左右，LZ4的磁盘占用空间要比ZSTD多2.5倍左右。



LZ4的压缩比约为1:4，ZSTD的压缩比约为1:10。



如果使用LZ4查询耗时为1秒，而ZSTD查询性能为1.5秒左右。



秒级的影响对使用方来说，体感并不明显，高达2.5倍的存储开销，缺耗费不少。



ZSTD压缩示例

```SQL
`message` String  CODEC(ZSTD),

INDEX idx_message message TYPE tokenbf_v1(512, 2, 0) GRANULARITY 2,
```



小结：选择合适的压缩算法，笔者更倾向经济实用型，会选择ZSTD压缩算法。





# 四、文章要点回顾总结



本文从ClickHouse术语、分布式表读写原理等理论入手，在进入实战前先行预热。逐步过度到日志存储实战，理论知识部分简明扼要在帮助快速理解，实战部突出设计调优要点，配有命令实战部分。



ClickHouse向量化引擎、列式存储、分布式表的读写原理以及如何负载均衡写入本地表。ClickHouse用于日志存储时的数据冷热存储、数据迁移与删除、集群规划与调优、表设计调优点，相信大家有了自己的思考。
