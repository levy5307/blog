---
layout: post
title: Key-Value Storage Engines
date: 2021-10-29
Author: levy5307
tags: [论文]
comments: true
toc: true
---

Key-value存储支持key-value API，以及管理key-value pairs，其应用十分广泛。

## KEY-VALUE STORAGE ENGINES, AND APPLICATIONS

***Storage Engine Design*** 

KV存储的核心是实现一个存储key-value pairs的数据结构，每个数据结构的设计都实现了读取、更新和内存放大的特定trade-off。例如，读放大被定义为读取一个特定的key时，需要读取的数据。实际上，读、更新和内存放大进一步细分为更细粒度的指标，如点读、范围读、更新、删除、插入和缓存所需的内存。核心数据结构的设计影响着每一个性能指标以及系统的每一个特性和属性。例如：

- 为了支持time-travel查询，需要决定如何存储timestamp信息。

- 为了加速对最近访问数据的查询，我们需要在缓存和其他结构之间平衡可用内存，这些结构需要加速对基本数据的处理，例如，内存bloom过滤器，fence pointers。

- 为了支持高效的并发读写请求，存储引擎可能必须将数据布局从“in-place”更改为“out-of-place”。

***Rapidly Changing System Requirements***

今天我们比以往任何时候都更希望快速构建、更改和调整数据系统，以便能够跟上不断变化的应用程序和硬件的需求。使用新workload patterns的新应用程序或现有应用程序中的新特性经常出现，单个系统无法有效地支持不同的workload。这一问题有几个日益紧迫的原因：

- 首先，出现了新的应用程序，其中许多应用程序引入了以前非典型的新workload pattern。

- 其次，现有的应用程序不断重新定义它们的服务和特性，这将直接影响它们的workload pattern，并且在许多情况下使现有的底层存储决策不是最优的，甚至是糟糕的。

- 第三，硬件不断变化，影响CPU/带宽/延迟平衡;

最大化性能要求底层存储设计更改，这些问题归结为：one size does not fit all problem，这适用于整个系统设计和存储层。特别是，在当今基于云计算的世界中，即使是略低于最优水平的设计，也会导致能源利用以及成本方面的巨大损失。

***There is no Perfect Design***

不存在完美的数据结构可以最小化所有性能trade-off。

例如，如果我们添加一个日志来支持高效的out-of-place写，我们就会牺牲内存/空间成本，因为我们可能有重复的条目，并牺牲读取成本，因为将来的查询必须同时搜索核心数据结构和日志。反过来，这意味着不存在能够满足不同性能需求的完美KV存储。***每一个设计都是一种妥协***。但是，我们如何知道哪种设计最适合应用程序？我们是否有足够的设计和系统来满足新兴和不断变化的数据驱动应用程序的需求？特别是在数据科学和机器学习的新机遇推动下，应用程序越来越多样化和庞大的今天，在系统内核适当设计以匹配目标应用程序的需求是非常关键的。

***Tutorial Structure***

本论文由三部分组成：

- 第一部分回顾了本节中描述的整个问题设置，1)定义键值存储系统，2)描述跨数据科学领域的传统和新兴应用程序，3)给出关键的开放挑战，特别是针对不同的workload和动态应用程序场景。

- 第二部分讨论了过去几年的主要研究和行业趋势，这些趋势已经形成了键值存储引擎的最先进水平。具体来说，我们将讨论1)读优化的存储引擎，2)写优化的存储引擎，3)系统组件之间分配内存的设计选项和注意事项，4)如何存储时间戳，5)如何存储大value。

- 在第三部分，我们提出了一个统一的框架，它封装了所有最先进的设计，并允许我们推理存储引擎的可能设计空间。我们还讨论了如何描述和交流存储引擎设计，以及为什么这是一个关键问题

- 最后，我们提出了self-designed的KV存储引擎的挑战。我们解释了1)与过去的解决方案相比，它们带来了新的机会，2)它们如何能够被用于解决跨许多类型的数据密集型应用的实际问题，3)它们与过去的工作相融合而产生的新的研究机会。self-designed系统知道关键组件（如data storage）可能的设计选择及其组合，并能在截然不同的选择中选择最合适的设计

## STATE-OF-THE-ART ENGINE DESIGN

***The Big Three***

KV存储主要使用三种数据结构来管理数据。为了了解它们提供的不同设计目标和性能平衡，我们简要介绍它们的核心设计特征。

第一个是***B+树***。B+树由页节点和索引组成，该页节点具有有序的kv pair，所以由fence pointers组成。例如，BerkeleyDB是由B+树实现的，现在作为MongoDB的主要存储引擎。FoundationDB同样依赖于B+树。总之，B+树实现了读写读写性能之间的平衡，并且具有合理的内存开销。

在2000年代初期出现了新的应用浪潮，其需要更快的写入，同时仍然需要良好的读取性能，同时，基于闪存的SSD的出现使得写入IO的成本比读取IO高1-2个数量级。这些workload和硬件趋势导致了KV存储的两个数据结构设计决策:

1. 在内存中缓冲数据，并向二级存储中批量写入

2. 避免全局顺序维护。

这是由***LSM-tree***所开创的设计，其将数据划分成一系列level。每个键值条目都首先进入最top level（内存中的写入缓冲区），并随着更多数据到达而在较低level进行排序合并。诸如bloom filter、fence pointers等内存结构有助于减少磁盘IO。LSM-tree已经被很多工业环境所采用，包括LevelDB，Google Bigtable，Facebook RocksDB，Cassandra，HBase等。以及更多的学术研究，例如SlimDB，WiscKey，Monkey，Dostoevsky和LSM-bush等。一些关系型数据库，例如MySQL和SQLite4通过将primary key作为key，row作为value而对其进行了支持。总而言之，LSM-tree比B+-tree实现了更好的写入，但是由于必须通过多个level来查找数据，其放弃了一些读性能，并且还会导致内存放大，以容纳足够的in-memory filters来支持高效的点查询。

最近，为了支持更快的导入速度，出现了***Log-structured Hash Table***。数据首先在in-memory写入缓冲区积累，当其满了之后，就将其push到二级存储中，作为持续增长日志的一个node。内存索引（例如Hash表）使得可以轻松定位任何kv pair，同时定期合并这些日志，以控制过期entry的上限。这种***Log and Index***设计在[Riak BitCask](https://levy5307.github.io/blog/BitCask/)、[Spotify Sparkey](https://github.com/spotify/sparkey)，[Microsoft FASTER](https://github.com/microsoft/FASTER)以及一些学术研究中均有运用。大多数系统都使用Hash表作为日志索引。总的来说，这种设计实现了出色的写入性能，但是牺牲了读取性能（对于范围查询），而内存占用也通常很高，因为现在所有的key都需要在内存中建立索引，以最大限度地减少每个key的IO。

***Memory Management***

KV存储最关键的决定之一是如何在各种in-memory组件之间分配内存。例如，在LSM-tree设计中，通常内存中有Bloom Filter和其他helper structure来帮助跳过磁盘读取。同样，缓存用于访问最近的数据。保存最近更新/插入的Buffer同样也会竞争内存使用。总体而言，这些组件中的每一个都可以对系统性能产生积极影响，但在固定预算、workload和存储引擎基础设计的情况下，如何最好地分配内存并不总是很清楚。

***Compactions and Splits***

根据具体的设计，NoSQL引擎需要频繁地重新组织数据，以保持某些性能不变。例如，LSM-tree需要在新数据到达时执行Compaction，以维护数据顺序并删除无效数据。根据Compaction的频率，系统可以对读或者写进行优化。此外，compaction可能是in-place或者out-of-place的。后者允许在compaction执行时同时提供查询服务（按照compaction前的数据layout查询）。这是以临时复制相关数据为代价的。in-place不需要任何额外的内存，但是需要在compaction时阻止查询。

***Concurrency Control***

KV存储需要支持大量的并发查询。当读取和写入同时到达时，根据存储引擎的确切设计，可以有很多方法来提高吞吐量。例如，与典型的B-tree设计相比，LSM-tree本质上更能够处理并发请求，因为它们以out-of-place形式更新数据。同样，log-structured hash table的设计更进一步，其执行更少的compaction，从而导致更少的读写冲突。B-tree可以采用out-of-place的形式，具体可以参考BW-tree或者B<sup>ϵ</sup>。

***Time-travel Queries***

在商业中非常有价值的应用程序能够根据系统在特定时间点的状态查询数据。这意味着kv pair应该与时间戳相关联。但是，即使对数据支持少量的版本，在存储方面也可能是一笔巨大的开销。同时，时间戳的存储方式至关重要。例如，如果时间戳和base data存储在一起，那么这可能导致查询的大量开销（因为时间戳需要和base data一起读书）

***CPU vs IO Cost***

KV存储引擎主要处理大数据，因此它们的大部分性能成本来自从磁盘拷贝数据和跨内存层次移动数据。然而，成本的很大一部分仍然来自CPU。例如，由于混合了不同访问模式的workload，存储引擎会密集的进行IO操作。同样，使用压缩会导致CPU消耗增大，所使用的压缩形式决定了节省的I/O与牺牲的CPU之间的平衡。这种权衡在云计算中变得尤为重要，因为CPU和I/O成本都是预算的一部分，而不同的云提供商提供不同（并且经常变化）的成本策略。

***Adaptive Indexing and Layouts***

一个为不同workload调节各种性能属性的方式是通过自适应，虽然在NoSQL存储中还没有研究过这个概念，我们将在讨论NoSQL引擎的挑战时使用这些概念。自适应索引（Adaptive indexing）是self-tuning database中的轻量级方法，它解决了dynamic workload的离线和在线索引的局限性。对于工作负载的变化，它通过部分地、增量地构建或改进索引来做出反应。也就是说，不需要DBA或者离线处理。通过对每个查询做出轻量级操作的反应，自适应索引设法立即适应不断变化的workload。随着越来越多的查询到达，索引被细化得越多、性能提高得越多。最近，该领域受到了相当多的关注，许多工作研究了关系存储、NoSQL系统等基础存储的自适应。通常在这些工作中，layout会适应收到的请求。类似地，通过实验和学习进行调优，通过机器学习进行调优，可以使用来自测试的反馈来调整设计。

## SELF-DESIGNED NOSQL STORAGE

***Self-designed Systems***

self-designed的系统使用设计空间自动生成最适合目标workload和硬件的设计。要做到这一点，我们需要知道设计空间中不同的点对系统的性能有何不同影响。例如，learned cost models是一种能够学习基本访问模式(随机访问、扫描、排序搜索)成本的方法，从中我们可以为给定的数据结构的复杂算法的计算成本

此外，design continuums是另一个方向，它允许准确快速地寻找最佳设计。它在所有可能的设计集合中连接特定的设计子集。design continuums实际上是设计空间的投影，是设计的“口袋”

***Storage Engine Description***

存储引擎设计的巨大设计空间及其复杂性引发的另一个关键问题是我们如何传达特定设计的细节。这对于系统架构师和工程师在系统演进时理解和维护系统是至关重要的。我们认为，需要一种高级语言来描述存储引擎，并提出了实现这一目标的想法和未决问题

***Applications: Big Data, Data Science***

在本教程的最后一部分，我们将讨论如何将所描述的新方向应用于众多数据密集型领域。我们讨论广泛的数据科学应用，如统计处理和机器学习系统。实际上，所有这些领域都需要处理不断增长的数据量。支持的数据存储和精确处理算法需要根据高级算法/workload和硬件所需的精确访问模式而变化。随着数据的增长，即使是这些决策中最次优的选择，也可能需要数小时甚至数天的处理时间。本教程中提出的新思想展示了开放的研究问题，旨在为此类数据密集型应用程序生成接近最优的存储系统。

 