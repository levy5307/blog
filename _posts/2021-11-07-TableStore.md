---
layout: post
title: TableStore
date: 2021-11-07
Author: levy5307
tags: []
comments: true
toc: true
---

之前抽时间研究了下当前比较火热的两款表格存储，BigTable和Cassandra。这篇文章主要对两者做一个对比概括。

## Data Model

Cassandra的数据模型如下图所示：

![](../images/cassandra-table.png)

Cassandra有两种column family，分别是Simple column family和Super column family。其中Super column family可被视为在一个column family之中的column family。column family中的任一column都需要通过column_family:column的形式来访问。在一个super column family中的列需要通过column_family:super_column:column的形式来访问。

bigtable数据模型：(row:string, column:string,time:int64)->string

其中，column命名语法为：{列族：限定词}。同时bigtable支持时间戳，可以用时间戳来索引同一数据的不同版本。

同Cassandra一样，BigTable对于单行数据的读写是原子的。

![](../images/bigtable-data-model.jpg)

## Partition

Cassandra采用的是一致性hash搭配负载分析的做法，其中负载分析主要用于解决一致性hash可能带来的负载不均的情况。而BigTable采用的是数据范围分片，由此可以看出，Cassandra能够更好的解决热点问题，而BigTable对于范围查询则更加友好。

## Replication

在Cassandra中，并没有使用一致性协议来进行replication，其将数据复制到N个节点上，并当发生故障或者网络分区的时候，通过放宽quorum的方式来提供持久化保证，所以Cassandra是一个最终一致性的系统。


