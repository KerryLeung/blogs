---
layout:     post 
title:      Clickhouse集成kafka流数据实践
subtitle:   实时数据导入
date:       2020-10-29             
author:     KL                  
header-img: img/post-bg-2015.jpg    
catalog: true                      
tags:                            
    - clickhouse
    - kafka
---

## Background
最近的项目中，有业务场景用到导入实时数据到Clickhouse的需求，再实践的过程中，由于clickhouse本身文档的一些不完善，有些经验和教训值得记录下。
Clickhouse的介绍可以参考 https://clickhouse.tech/docs/en/
ClickHouse is a column-oriented database management system (DBMS) for online analytical processing of queries (OLAP).

具备参考价值的文档：
1. 官方文档，kafka engine是一种特殊的table engine https://clickhouse.tech/docs/en/engines/table-engines/integrations/kafka/
2. 插件作者维护的文档： https://altinity.com/blog/2020/5/21/clickhouse-kafka-engine-tutorial 介绍了很多官方文档没有提及的用例，已经一些FAQ问题，非常有用。
3. 头条的实践分享： https://mp.weixin.qq.com/s?__biz=MzI1MzYzMjE0MQ==&mid=2247486616&idx=1&sn=bf430bf24a31427099c10b58677b1513&chksm=e9d0c77adea74e6c55f3c2c3f8740b8cb21cd6d06136163d0010c225869581d8ae6b0f656df9&mpshare=1&scene=23&srcid=08279Hm50pvjEqddah1L4zVm&sharer_sharetime=1598497600046&sharer_shareid=a899e7b0540fc4bd14e13a68570e1619%23rd 

## 实践与使用案例分析
此方案是基于Clickouse 支持的Kafka Engine table 实现的，优点是配置简单，可以快速实现，无引入额外的组件或服务。缺点主要集中在资源管理不灵活，配置繁琐，需要三张clickhouse表，且消费kafka的语义为at least once，缺乏监控机制。先来看看官方文档对于Kafka Engine 的定义: https://clickhouse.tech/docs/en/engines/table-engines/integrations/kafka/
```
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
(
    name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1],
    name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2],
    ...
) ENGINE = Kafka()
SETTINGS
    kafka_broker_list = 'host:port',
    kafka_topic_list = 'topic1,topic2,...',
    kafka_group_name = 'group_name',
    kafka_format = 'data_format'[,]
    [kafka_row_delimiter = 'delimiter_symbol',]
    [kafka_schema = '',]
    [kafka_num_consumers = N,]
    [kafka_max_block_size = 0,]
    [kafka_skip_broken_messages = N,]
    [kafka_commit_every_batch = 0,]
    [kafka_thread_per_consumer = 0]
```
除了建表Engine部分的kafka参数外，其他语句与正常engine并无差别,使用也非常简单，只需要指定broker list和topic name, consumer group name即可，kafka message的格式支持常见的csv,json,pb,avro,orc等等。

实现原理为C++实现的librdkafka包，可支持的配置参数可参见https://github.com/edenhill/librdkafka/blob/master/CONFIGURATION.md 。

kafka engine table消费topic的原理是，每台机器上的一个table即是一个consumer，同一个cluster中的不同node上的同一个topic的table分属与同一个consumer group中的不同consumer，topic中的一条message只能被一个consumer消费成功。所以有一点要注意，如果对kafka engine table run select 语句，那么消息被显示之后，数据就会消失。所以文档也说，kafka engine一张表本身是没有什么用处的，需要配合其他表一起工作。

### use case
典型用法是kafka engine table + materialized view table + mergetree table, 其中kafka table负责数据ingest，mv table 负责数据搬移，mergetree table负责数据存储和对外提供查询，下面是官方提供的样例
```
CREATE TABLE queue (
    timestamp UInt64,
    level String,
    message String
  ) ENGINE = Kafka('localhost:9092', 'topic', 'group1', 'JSONEachRow');

  CREATE TABLE daily (
    day Date,
    level String,
    total UInt64
  ) ENGINE = SummingMergeTree(day, (day, level), 8192);

  CREATE MATERIALIZED VIEW consumer TO daily
    AS SELECT toDate(toDateTime(timestamp)) AS day, level, count() as total
    FROM queue GROUP BY day, level;

  SELECT level, sum(total) FROM daily GROUP BY level;
```
queue定义kafka数据来源于格式，daily表即对外提供查询服务的table，mv表consumer负责数据搬运。其中值得强调的是，mv table其实是进行了日期对齐操作的，kafka table的timestamp往往是非常细粒度的数据，如果不在时间维度就行数据压缩，那么mergetree table的数据量会非常大，对于存储，网络和查询的压力会非常大。
为了解决数据量的问题，mergetree table这里用了SummingMergeTree engine，这个engine可以对于数字类型的列自动apply sum聚合操作，以减少数据量，由于在mv的sql中对于时间进行了toDate运算，所以在daily table中可对数据进行数据聚合，以减少数据量，mv table的SQL也可以实现其他的transform操作，这里看业务需求的定义，而SummingMergeTree也可以替换成其他的engine，达到类似的效果，比如aggregateMergeTree。

### 集群场景的配置
参见 https://altinity.com/blog/clickhouse-kafka-engine-faq  简单说有两种方法
1. 在集群的每个node上都配置一个kafka table，然后通过mv写入到local table中，对外通过distributed table提供query 访问
The best practice is to create a Kafka engine table on every ClickHouse server, so that every server consumes some partitions and flushes rows to the local ReplicatedMergeTree table.

2. 在集群的每个node上都配置一个kafka table，直接通过mv写入distributed table
Another possibility is to flush data from a Kafka engine table into a Distributed table.

第二种方法不推荐，直接写入分布式表有数据不一致的风险，且增加网络消耗。第一种方法要注意kafka分区数据数量和Clickhouse cluster的数量的大小关系。

## 总结
总体来看，通过Clickhouse的kafka engine table插件，ingest实时数据并提供数据访问是一个比较简单易配置的workflow，但是也要注意对于实时数据的聚合，ingest job的监控，消费kafka的分布式语义等等问题。在项目中碰到的问题，有机会可以在之后的blog中继续分析。
