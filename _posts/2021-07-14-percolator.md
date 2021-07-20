---
layout: post
title: Percolator
date: 2021-07-14
Author: levy5307
tags: [论文]
comments: true
toc: true
---

## Background

在Google公司内部，对于海量索引数据的创建和实时更新是必须面对的问题。Map Reduce解决了海量索引数据的批量创建问题，但是却不能支持对增量数据的实时更新，每次需要对全量索引数据进行一次重新创建。并且由于文档在Google上能否被检索到取决于全量索引的创建时间，因此导致其被检索到的时间间隔较长。

综上，Google内部缺少一个支持海量数据存储、支持并行随机读写、支持跨行事务的分布式数据库。Percolator便因此而生。

## Design

Percolator为大规模的增量计算提供了两个抽象：基于随机访问仓库的的ACID事务、以及用于组织增量计算的observers

一个Percolator系统由三个二进制文件组成，他们在集群中的每台机器上运行：

- 一个Percolator Worker。所有的obserers都链接到Percolator Worker，Percolator Worker用于scan Bigtable的修改的column，并唤醒对应的observers。

- 一个Bigtable tablet server。observer通过向Bigtable的tablet server发送读写RPC来执行事务。

- 一个GFS chunkserver。Bigtable tablet server将其读写请求发送至GFS chunkserver，因为Bigtable是使用GFS做存储层，具体可查看bigtable论文

此外，Percolator还依赖两个小服务：时间戳oracle和轻量级锁服务。时间戳oracle用于提供严格线性增长的timestamp，snapshot隔离级别协议需要该线性增长的timestamp。轻量级锁服务使搜索dirty notification更加高效。

在程序员的视角来看，Percolator是由少量的table组成。table是由行和列索引的单元格cell组成。每个cell都对应一个value，该value是由一组uninterpreted bytes组成。（在Percolator内部，为了支持snapshot隔离级别，每个单元格内包含由一系列时间戳索引的values）

Percolator的设计基于两个前提：

1. 运行在大规模数据上

2. 并不要求非常低的延迟。不严格的延迟要求允许我们采用一种懒惰的做法来清理在失败机器上运行的事务所遗留的锁。这种懒惰的做法比较容易实现，但是可能会让事务提交延迟数十秒。这种延迟在OLTP系统上通常是难以接受的，但是在构建web索引的增量处理系统中是可以容忍的。

Percolator缺少一个中央总控用于事务管理，尤其它缺少一个全局的死锁检测器。这增加了产生冲突事务的延迟，但是却允许系统可以扩展到上千台机器。

Percolator是基于Bigtable构建的，它需要将一些元数据存储在旁边特殊的列中，其实现的挑战主要在于提供Bigtable所没有的功能：多行事务和observer framework

### 事务

Percolator提供了跨行、跨表的ACID快照隔离级别的事务。其充分利用的Bigtable的timestamp，对每个数据项都维护多个版本（MVCC）以实现快照隔离，快照隔离可以有效的解决写-写冲突：拥有较大提交版本号的事务将会正确提交，另外一个则会回滚。另外，Percolator并不提供串行隔离。根据之前的文章中介绍，MVCC并不能解决write skew的问题，但是其相对于串行隔离的优势是性能更高，具体可以参考[分布式事务](https://levy5307.github.io/blog/distributed-transaction/)。

Percolator需要维护锁，其对锁有如下几个要求：

- 锁必须是持久化的，以应付宕机导致锁失效

- 锁服务一定要支持高吞吐量，因为可能会有几千台机器并行请求锁

- 锁服务一定要是低延迟的，每个Get（）操作都会申请读锁，以最大化降低锁服务带来的延迟

- 高可用。需要冗余备份，以防异常故障

基于这些要求，Percolator采用了Bigtable，将锁和数据存储在同一行，采用特殊的列来存放锁。在访问某行数据时，Percolator将在一个Bigtable行事务中对同行的锁执行读取与修改。

下表展示了Percolator中名为c的column在Bigtable中所对应的columns

| Column   | Use                                  |
|----------|--------------------------------------|
| c:lock   | 未提交的事务才会写入该cell, 包含primary lock的位置   |
| c:write  | 已提交的事务才会写入。存储bigtable中data的timestamp |
| c:data   | data itself                          |
| c:notify | Hint: Observers可以开始运行了               |
| c:ack_O  | Observer “O”已经运行过了。存储其最后一次运行成功的时间戳   |

在percolator的事务中，每个事务有两个对应的timestamp：start timestamp和commit timestamp。分别代表事务开始时间和事务提交时间。

下面以Bog向Joe转账7元为例，看一下Percolator事务的执行流程：

- 初始状态，Joe的账户中有2元，Bob的账户有10元

![](../images/percolator-initial-state.jpg)

- 该事务获取其start timestamp: 7。并且获取Bob这一行的锁（该锁是primary lock），同时向该start timestamp中写入数据3

![](../images/percolator-step-1.jpg)

- 该事务现在为Joe这一行加锁，并且写入Joe新的余额，该锁是一个secondary lock，其包含一个指向primary lock的引用。当事务crash时，该事务需要清理，此时其需要获取primary lock。

![](../images/percolator-step-2.jpg)

- 此时，该事务达到commit point。此时事务获取其commit timestamp: 8。并且清空primary lock，并在timestamp 8的行的balance:write列中写入一条record，该record记录了data在哪个timestamp行中存储。这样后续的reader便可以读取到Bob的新余额3

![](../images/percolator-step-3.jpg)

- 该事务完成向Joe这一行中write records写入以及删除锁

![](../images/percolator-step-4.jpg)





