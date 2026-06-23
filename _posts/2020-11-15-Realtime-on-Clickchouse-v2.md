---
layout:     post 
title:      Clickhouse集成kafka流数据的不足
subtitle:   避免踩坑
date:       2020-11-15             
author:     KL                  
header-img: img/post-bg-2015.jpg    
catalog: true                      
tags:                            
    - clickhouse
    - kafka
---

## Background
Clickhouse提供了集成kafka的解决方案，从而使在clickhouse中集成流数据变为可能，但是如果要使用在生产环境中，有以下几个方面的问题不得不考虑。
1. 消费Kafka数据的语义是at least once, 消费的数据会有重复
2. 维护的成本与job监控
3. 扩展性

## EOS support
官方文档并未说明kafka engine消费Clickhouse的语义，但是从插件原作者维护的[文档|https://altinity.com/blog/clickhouse-kafka-engine-faq] 中可见详细的说明:
```
Duplicates are theoretically possible, since current implementation ensures the at-least-once contract.
That should not happen in normal circumstances if the recommended ClickHouse version is used. It can happen in rare corner cases, though. For example, if the data was inserted into ClickHouse, and immediately after the connection to the Kafka broker has been lost, then ClickHouse was not able to commit a new offset.
```
可知，kafka engine table消费Kafka数据的时候是有可能重复的，但是从实践经验来看，重复发生的比例并不高，鉴于实时数据本身对于数据准确性的要求就有妥协，如果针对要使用的使用场景，diff ratio在可接受范围内，此方案还是非常值得采用的，毕竟流程简单，配置容易。

如果对于数据可能重复非常敏感或者不容容忍的话，只能期待插件作者在将来优化，在文档中也有人问及此问题，作者的答复是
```
We have some design concepts of how to avoid that (store commits / offsets on ClickHouse side, to avoid commits in 2 system, and / or introduce transactions/rollbacks into ClickHouse).

That may appear in a few months (we have several other things to fix before).

Support for producing messages in EOS semantics is not planned in the nearest future.
```
作者在消费kakfa数据不重复方面，其实是有一些短期的改进意见的，可以持续关注下社区的发展与更新。其改进意见与Druid support EOS的方式类似，毕竟这两个query engine管理数据的方式类似，即在组件内部维护kakfa consumer的offset，并与clickhouse数据写入的part文件对应，从而实现数据的不重复写入，具体实现可参考Druid的KIS（Kafka Indexing Service）。

在作者完成功能支持之前，一个可选的方案是在通过ReplacingMergeTree 的去重机制来实现，官方文档说明如下
```
The engine differs from MergeTree in that it removes duplicate entries with the same sorting key value.
...
Thus, ReplacingMergeTree is suitable for clearing out duplicate data in the background in order to save space, but it doesn’t guarantee the absence of duplicates.

```
将数据通过materialized view导入到ReplacingMergeTree table之后，设置好去重的key，然后clickhouse会在合适的时间进行数据去重，这个方案有两个问题
1. 去重时机不可控，即重复数据又被查询到的可能，如对数据准备性能容忍，则问题不大
2. 只考虑去重，未考虑数据聚合，会导致数据量急剧增加，增加存储的负担，已经降低查询效率
针对第二点，也考虑过在ReplacingMergeTree table之后再通过materialized view导入数据到SummingMergeTree的做法，但是由于materialized view的设计机制，导致ReplacingMergeTree table数据去重之后，不会触发SummingMergeTree table的更新，所以不能同时即实现去重又实现聚合。

## 维护的成本与job监控
数据导入的方式是通过
kafka table -> materialized view -> mergetree table
实现的，所以一个Kafka topic需要三张clickhouse的table来实现，维护负担略重，同时kafka table本质是一个consumer，数据被读取一次之后就不能重复读取，在配置Kafka topic table的时候，一定要注意用户与表权限的配置，否则万一有人在kafka table执行了select语句，那么这段数据就完全丢失了，不会写入到final table中的。

对于监控方案，不得不说，这套解决方案的监控近乎于没有，既不能知道某个topic消费到的offset是什么，也不知道一共消费了多少行数据，速率是什么样的。对于产品的维护和运维非常的不友好，经过调研之后，有两个方向可以抢救下
1. 通过kafka manager查看clickhouse consumer的offset与消费信息。 因为目前clickhouse的consumer的offset是保留在kafka side的
2. 通过Clickhouse view的机制与kafka engine的virtual column结合在一起，定制化记录监控所需要的信息。

从kafka读取数据的时候，可得到的数据本身的信息有
```
_topic — Kafka topic.
_key — Key of the message.
_offset — Offset of the message.
_timestamp — Timestamp of the message.
_partition — Partition of Kafka topic.

```
已上信息可用来做监控。同时view的机制决定，多个view如果同时读取同一个kafka table的数据，当有数据来临的时候，所有的view都会被触发，可以通过链接多个view的方式，自定义化处理监控信息。

## 扩展性
如果是一个集群，那么可通过选择在其中几台机器上配置kafka table消费topic的数据，具体的consumer数量可结合业务与性能测试得到，之后若需水平扩展，增加consumer的数量即可。

## 总结
总体来看，通过Clickhouse的kafka engine table的稳定性还是不错的，但是在监控方面还有很多不足，会对生产环境的adoption产生很多困扰，仍有很多改进的空间。

