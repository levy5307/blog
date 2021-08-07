---
layout: post
title: Cassandra
date: 2021-08-07
Author: levy5307
tags: [论文]
comments: true
toc: true
---

Cassandra的目标是构建在上百台的节点之上（可能会跨越多个不同的data center）。Cassandra用于设计满足Facebook的Inbox Search的存储需求，这要求该存储系统需要能够处理非常高的写吞吐，每天数十亿的写入，以及随着用户规模增长而扩容。并且由于用户根据地理位置可能被不同的data center服务，所以在不同的data center之间复制数据是很关键的。

## Data Model

Cassandra中的表示一个分布式的、由key索引的多维map。value是一个高度结构化的object。row key是一个string类型数据，没有大小限制（通常是16-36 bytes大小）。对于一个replica中的单行读写，不管其涉及到的列有多少个，其操作都是原子的。Column可以组成一个set叫做column family，这与Bigtable很像。Cassandra有两种column family，分别是Simple column family和Super column family。其中Super column family可被视为在一个column family之中的column family

此外，应用可以指定Super column family和Simple column faily中的column排列顺序，包括按时间排序和按名字排序。时间排序在Inbox Search中得到了应用，因为其需要按照时间顺序展示结果。column family中的任一column都需要通过column_family:column的形式来访问。在一个super column family中的列需要通过column_family:super_column:column的形式来访问。代表性的应用使用一个专用的Cassandra集群，并且将其作为他们服务的一部分，尽管Cassandra支持多个表，并且每个表都有其自己的schema。

## API

Cassandra API包括以下三种简单的方法：

```insert(table, key, rowMutation)```

```get(table, key, columnName)```

```delete(table, key, columnName)```

columnName可以是一个column family中的指定的column、一个column family，一个super column family或者一个super column中的column

## System Architecture

对于运行在生产环境中的存储系统是非常复杂的。出了实际数据的持久化之外，存储系统还需要包括如下特性：scalable and robust solutions of load balance, membership and failure detection, failure recovery, replica synchronization, overload handling, state transfer, concurrency and job scheduling, request marshalling, request routing, system monitoring and alarming, and configuration management. 在本文中不讲述上述这些细节，主要集中在Cassandra中应用的核心的分布式技术： partitioning, replication, membership, failure handling and scaling。所有这些模块协同起来处理读写请求。一个对于key的读/写请求呗路由到Cassandra中的节点，该节点决定该key所在的副本。对于写入，系统将该请求路由到所有的副本，并且等待一定数量的副本的写入完成的ack。对于读取，根据client需要的一致性保证，系统可能会将请求路由到最近的副本，或者路由到所有的副本，等待一定数量的response。

### Partitioning

Cassandra的一个关键特性是其拥有逐渐扩展的能力，这就需要能够动态的为数据在集群的一系列的节点上分区的能力。Cassandra使用一致性hash来对数据进行分区，该hash函数使用了保序hash函数。一致性hash的细节在这里就不细说了，可以参考文章[数据分片](https://levy5307.github.io/blog/thinking-about-partition/)。

基础的一致性hash有几个挑战：

- 每个节点在环上的位置时随机的，这会带来不同节点不统一的数据和负载分布

- 它忽略了节点的异构性问题

面对这些问题，通常有两种解决办法：

1. 为每个节点分配环上的多个位置

2. 分析换上的负载情况，然后具有轻负载的节点从环上向前移动，用以减轻高负载节点的负载

Cassandra选择第二种方法，因为它的设计和实现更容易驾驭，并且可以帮助负载均衡作出更加确定的选择。

另外需要说明，每一个节点都知道系统中的所有其他节点以及其负责的数据range。

### Replication

Cassandra使用Replication来实现高可用以及持久性。每个数据都被复制到N个节点上，其中N是一个per-instance的replication配置。如Partitioning一节所述，每个key都会被分配到一个coordinator节点（该coordinator负责以个range的key，该key刚好落入该range内）。除了将数据在本地持久化以外，coordinator还需要将数据复制到环上的N-1个节点之上。Cassandra为客户端提供了数据复制的多种不同选择，包括：Rack Unaware, Rack ware, Datacenter aware，前两者在一个data center，后者跨越多个data center。根据客户端选择的replication policy来选择副本:

- Rack Unaware。如果客户端选择了Rack Unware，那么选择coordinator在环上的下游的N-1个节点作为non-coordinator副本。

- Rack Aware和Datacenter aware。这两种情况稍微复杂。Cassandra在集群中通过zookeeper选择一个leader节点出来，该leader节点告诉一个节点其需要作为副本的环上的range。并且leader会保证每个节点不会为超过N-1个range作副本。关于一个节点负责作为副本的range的meta信息会缓存在本地，并且持久化在zk上，这样当一个节点重启的时候可以从zk拉取该信息。我们借用了Dynamo的说法，将负责一个指定range的节点叫做preference list。需要说明的一点是，Rack Aware和Datacenter aware的区别在于前者的副本在同一个data center，而后者需要跨data center。

如上一节所述，每一个节点都知道系统中的所有其他节点，以及其负责的range。通过放款quorum要求，即使在节点发生故障或者发生网络分区的情况下，Cassandra仍然可以提供持久化保证。Cassandra可以通过配置，让每一个row都复制到跨越多个data center，本质上，一个key的preference list是有跨越多个datacenter的存储节点构成的，这些datacenter通过高速网络连接，这种设计让我们可以在整个data center都挂掉的时候仍然可以正常提供服务。

### Membership

