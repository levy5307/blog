---
layout: post
title: Facebook Cassandra
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

对于运行在生产环境中的存储系统是非常复杂的。出了实际数据的持久化之外，存储系统还需要包括如下特性：scalable and robust solutions of load balance, membership and failure detection, failure recovery, replica synchronization, overload handling, state transfer, concurrency and job scheduling, request marshalling, request routing, system monitoring and alarming, and configuration management. 在本文中不讲述上述这些细节，主要集中在Cassandra中应用的核心的分布式技术： partitioning, replication, membership, failure handling and scaling。所有这些模块协同起来处理读写请求。一个对于key的读/写请求呗路由到Cassandra中的节点，该节点决定该key所在的副本。对于写入，系统将该请求路由到所有的副本，并且等待quorum数量的副本的写入完成的ack。对于读取，根据client需要的一致性保证，系统可能会将请求路由到最近的副本，或者路由到所有的副本，等待quorum数量的response。

### Partitioning

Cassandra的一个关键特性是其拥有逐渐扩展的能力，这就需要能够动态的为数据在集群的一系列的节点上分区的能力。Cassandra使用一致性hash来对数据进行分区，该hash函数使用了保序hash函数。一致性hash的细节在这里就不细说了，可以参考文章[数据分片](https://levy5307.github.io/blog/thinking-about-partition/)。

基础的一致性hash有几个挑战：

- 每个节点在环上的位置时随机的，这会带来不同节点不统一的数据和负载分布

- 它忽略了节点的异构性问题

面对这些问题，通常有两种解决办法：

1. 为每个节点分配环上的多个位置

2. 分析环上的负载情况，然后具有轻负载的节点从环上向前移动，用以减轻高负载节点的负载

Cassandra选择第二种方法，因为它的设计和实现更容易驾驭，并且可以帮助负载均衡作出更加确定的选择。

另外需要说明，每一个节点都知道系统中的所有其他节点以及其负责的数据range。

### Replication

Cassandra使用Replication来实现高可用以及持久性。每个数据都被复制到N个节点上，其中N是每个Cassandra自己的replication配置。如Partitioning一节所述，每个key都会被分配到一个coordinator节点（该coordinator负责以个range的key，该key刚好落入该range内）。除了将数据在本地持久化以外，coordinator还需要将数据复制到环上的N-1个节点之上。Cassandra为客户端提供了数据复制的多种不同选择，包括：Rack Unaware, Rack ware, Datacenter aware，前两者在一个data center，后者跨越多个data center。根据客户端选择的replication policy来选择副本:

- Rack Unaware。如果客户端选择了Rack Unware，那么选择coordinator在环上的下游的N-1个节点作为non-coordinator副本。

- Rack Aware和Datacenter aware。这两种情况稍微复杂。Cassandra在集群中通过zookeeper选择一个leader节点出来，该leader节点告诉一个节点其需要作为副本的环上的range。并且leader会保证每个节点不会为超过N-1个range作副本。关于一个节点负责作为副本的range的meta信息会缓存在本地，并且持久化在zk上，这样当一个节点重启的时候可以从zk拉取该信息。我们借用了Dynamo的说法，将负责一个指定range的节点叫做preference list。需要说明的一点是，Rack Aware和Datacenter aware的区别在于前者的副本在同一个data center，而后者需要跨data center。

如上一节所述，每一个节点都知道系统中的所有其他节点，以及其负责的range。通过放款quorum要求，即使在节点发生故障或者发生网络分区的情况下，Cassandra仍然可以提供持久化保证。Cassandra可以通过配置，让每一个row都复制到跨越多个data center，本质上，一个key的preference list是有跨越多个datacenter的存储节点构成的，这些datacenter通过高速网络连接，这种设计让我们可以在整个data center都挂掉的时候仍然可以正常提供服务。

在Cassandra中，并没有使用一致性协议来进行replication，所以Cassandra是一个最终一致性的系统。

### Membership

Cassandra的cluster membership是基于Scuttlebutt实现的，Scuttlebutt是一个非常高效的、反熵的Gossip协议。其突出特性是其高效的CPU利用率以及高效的gossip channel利用率。在Cassandra中，Gossip不仅应用在membership，也应用在传播其他系统相关控制状态。

#### Failure Detection

failure detection是一个节点用来判断其他节点up或者down的机制。在Cassandra中，failure detection用来防止尝试向已经下线的机器发送请求。Cassandra使用了&Phi; Accrual Failure Detector的变种。Accrual Failure Detector的思想是，不简单的使用Bool值来判断节点的up或者down，而是使用一个value来代表每个节点的怀疑度。该value是&phi;，其实动态调整的，用来表示当前被监控节点的网络和负载条件。

其判断方法如下：

1. 一个给定的&Phi;

2. 每个节点维护一个窗口，用于记录gossip message从其他节点的到达间隔时间，从而获得一个时间分布，并且根据分布获取&phi;，

3. 当&phi; > &Phi;时，就可以认为该主机已经宕机了。

当然这里可能会产生误判，误判的可能如下：

- &Phi;=1，10%

- &Phi;=2, 1%

- &Phi;=3, 0.1%

通过查阅Cassandra手册，Cassandra默认采用8。并且Cassandra建议，在不稳定的网络中可以将其提高到10-12，用于避免false failure。值大于12或者小于5都是不推荐的。

另外需要说明的一点，在Scuttlebutt的paper中，其使用的是Gaussian分布，而Cassandra采用的是指数分布，因为Cassandra认为由于gossip channel以及其对延迟的影响，指数分布更加相似。

### Bootstraping

当一个节点第一次启动时，它首先选择一个随机的token，用以匹配环上的位置。为了容错的需要，该信息会被持久化到本地和zk中。随后，该信息会通过gossip传输到整个集群。这就是我们怎么知道所有节点以及其对应的环上位置的。这使得所有的节点都可以转发一个请求到正确的节点。在启动时，当一个节点需要加入集群时，它首先读取配置，该配置中包括集群中的几个联络点，我们称这个初始的联络点叫做集群的种子（seed）。种子也可以来自一个类似于zk的服务。

在Facebook的网络环境中，节点故障通常是很短暂的，但是有时也会延长一段时间。故障有多种形式，例如：磁盘故障、cpu损坏等。节点停机很少代表永久离开，因此不会导致rebalance操作以及不会导致unreachable副本的repari操作。同样的，手工错误可能会导致以外的启动新的Cassandra节点。为了避免这种情况，所有的消息都包含Cassandra实例（集群）名字，当配置中的手工错误导致一个节点尝试加入一个错误的Cassandra实例中，可以根据集群名字来组织它。由于上述原因，使用一种明确的机制向Cassandra实例中添加或者删除节点则更加合适。管理员使用命令行工具或者浏览器连接到Cassandra节点，发起一个成员变成来加入或离开集群。

### Scaling the Cluster

当一个新节点加入系统时，会被分配一个token，这样便可以缓解负载较高的节点的负载。这样导致新的节点会分担部分先前其他节点负责的范围。Cassandra的boostrap算法是由系统中的其他节点发起，通过命令行工具或者Cassandra的web dashboard。准备转移数据的节点通过内核到内核的copy技术将数据copy到新节点。根据运维经验，单个节点的数据传输速率可以达到40MB/sec，我们还在努力通过多个副本参与并行化传输来进行改善，类似于Bittorrent技术。

### Local Persistence

Cassandra依赖本地文件系统做持久化，数据以一种有助于高效地数据检索的格式存储在磁盘上。通常一个写入操作包括一次commit log写入（该写入用于持久化以及恢复使用）、以及一次内存写入。内存写入执行要在commit log写入之后。我们有一个专门的磁盘用于顺序写入日志，因此可以最大化磁盘吞吐。当内存中的数据结构的大小（根据数据量大小和对象数量）超过一个给定的阈值，它将会把自己dump到磁盘中。这个dump写入时执行在系统中的一个磁盘上。所有的写入是顺序的，并且会产生一个基于row key的index（block index）用于高效的检索。该index文件将会与data文件一起持久化。经过一段时间，很多文件可能会存储在磁盘上，然后一个后台的merge操作会将这些文件合并整理成一个文件。这个过程与Bigtable的compaction过程非常类似。

通常，在查询磁盘文件之间，读操作首先查询in-memory数据结构。查询是根据从新到旧的顺序。当磁盘查询时，我们需要从磁盘上多个文件中查询该key。为了避免查询不包含该key的文件，针对每个文件的bloom filter需要保存在内存中，查询文件之前，需要首先查询bloom filter查看该key是否在该文件中。一个column family可能包含很多个column，当需要检索的column举例key较远时需要利用一些特殊的索引。为了避免扫描磁盘上的没一列，我们维护了一份列索引来帮助我们直接定位到正确的磁盘chunk。由于指定key的列已经被序列化并写入到磁盘，我们按照每个磁盘256K的chunk boundary创建索引的。该boundary是可以配置的，但是我们发现256K大小对我们来说运行的很好。

### Implementation Details

单台机器上的Cassandra主要包括如下几个模块：partitioning module, the cluster membership and failure detection module and the storage engine module。所有这些模块依赖一个事件驱动模块，其将消息处理pipeline和task pipeline划分成了多个阶段。另外，该事件驱动模块是按照SEDA实现的。the cluster membership and failure detection module建立在非阻塞IO的网络层之上。所有的系统控制信息都依赖于基于UDP协议的消息传输，而用于replication和request routing等应用相关的消息则依赖于TCP协议传输。请求路由模块的实现使用了一个状态机，当集群的任一节点收到请求时，状态机都会在一下几种状态之间切换：

1. 定位拥有这个key的节点

2. 将请求路由到此节点并等待响应到达

3. 如果response没有在配置的超时时间内到达，则将此请求设置为失败并返回给客户端

4. 根据时间戳计算出最新的response

5. 为任何数据不是最新的副本安排数据修复。

为了论述起见，我们再这里不讨论故障的详细情况。系统可以被配置为同步写入或者异步写入。对于特定的需要高吞吐的系统，我们会选择异步replication。对于使用同步的情况，我们需要等待quorum数量的response返回后才会返回结果给客户端。

任何的日志系统都存在一个清除commit log的机制。在Cassandra中我们使用一种滚动的提交日志，在一个旧的提交日志超过一个特定的可配置大小后，就推出一个新的提交日志。我们发现在我们的线上生产环境中，128MB的大小运行的特别好。每个commit log都有一个header，该header是一个固定大小的bit vector，其大小大于可能的column family数量。每个commit log都有一个bit vector并且会在内存中维护。对该bit vector有如下几个操作：

1. 修改。在我们的实现中，对每个column family都有一个in-memory数据结构和一个磁盘上的data file。每次当一个column family的in-memory数据结构被持久化到磁盘中，我们在commit header中设置一个bit，用于表示该column family已经成功持久化到磁盘上。

2. 检查。每次滚动提交日志时，都会检查其bit vector，并检查之前滚动的提交日志的所有bit vector。如果发现所有的column family都持久化到了磁盘上，那么该commit log就可以清除了。

写入到commit log的操作可以分为normal模式和fast sync模式两种。在fast sync模式中，写入commit log的数据会被缓存起来。这意味着当机器宕机时，是有可能数据丢失的。在这种模式下，in-memory数据结构被持久化到磁盘时，也是会进行缓存的。

传统的数据库并不是设计用来处理高写入吞吐的，在Cassandra中，所有向磁盘的写入都是顺序的，以最大化磁盘写吞吐。由于文件dump到磁盘后不会再修改了，所以在读取时无需进行加锁。Cassandra的服务实例的读写操作实际上都是无锁操作，因此我们并不需要应付基于B-Tree的数据库实现中存在的并发问题。

Cassandra是通过primary key来索引数据的。磁盘上的data file会被分成一系列的block。每个block最多包含128个key，并根据block index进行区分。block index记录block内键的相对偏移及其数据的大小。当in-memory数据结构被dump到磁盘上时，一个block index将会生成。为了快速存取，该block index同样会在内存中维护。

读操作同样会先查询in-memory数据结构，如果找到则返回给客户端，因为该in-memory数据结构肯定会包含最新的数据。如果没有找到则会以反向时间顺序去依次查找磁盘，因为这样先去查询最新的文件，一旦找到数据便可以返回给客户端。随着时间，磁盘上的数据文件数量将会增多，我们会运行一个非常类似于Bigtable系统的Compaction进程，通过它将多个文件合并成一个。merge操作是在一系列排好序的文件进行合并，系统总会对大小相近的两个文件进行compaction。例如，永远不会出现一个100GB的文件和一个50GB的文件进行compaction的情况。每隔一段时间，一个major compaction将会执行，将所有相关的文件合并成一个大的文件。compaction是一个IO密集型操作，需要对此进行大量的优化以做到不影响后续的读请求。

