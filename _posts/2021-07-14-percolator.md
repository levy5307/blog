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

