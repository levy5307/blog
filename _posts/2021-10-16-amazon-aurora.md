---
layout: post
title: Amazon Aurora
date: 2021-10-16
Author: levy5307
tags: [论文]
comments: true
toc: true
---

## Introduction

在当前的分布式云服务中，村算分离提供了弹性和扩展性。但是仍然存在几个问题：

- 当前的瓶颈变为了存储和计算层之间的IO

传统数据库当前面临的IO瓶颈已经发生了变化，因为IO可以分布在多个节点和多个磁盘上，因此单个的磁盘和节点不再过热

- 数据库将并行地向存储系统发出写操作，这将导致流量放大的问题

- 很多时候我们需要同步操作，这将导致stall和上下文切换

例如: 当缓存未命中所导致的磁盘读取操作，当读取完成之前，该读取线程无法继续执行；提交一个事务时的暂停可能会阻碍其他事务的进展

相较于传统的数据库服务，Aurora的架构有如下三个显著的优点：

1. 使用了一个独立的，能容忍错误并自我修复的跨数据中心storage服务，保证了database不受网络层或数据存储层问题的影响。

2. 在数据存储中，只写入redo log记录，可以从量级上减少网络IOPS。

3. 将耗时复杂的功能从一次昂贵的操作转变为连续的异步操作，保证这些操作不会影响前台处理

## DURABILITY AT SCALE

据库系统必须满足数据一旦写入就可以读取的约定，然而并不是所有的系统都是这样。在本节中，我们将讨论quorum model（仲裁模型）背后的基本原理，为什么要对存储进行分段，以及如何将这两种方法结合起来，不仅提供持久性、可用性和减少抖动，而且还帮助我们解决大规模存储的管理操作问题。

### Replication and Correlated Failures

instance生命周期与storage生命周期没有很好的相关性。实例失败、客户关闭了它们、根据负载调整它们的大小，这些原因有助于将存储层与计算层分离。这样做的话，这些存储节点和磁盘同样也可能出现故障。因此，它们必须以某种形式复制以提供故障恢复能力。在大规模的云环境中，将会持续的存在来自节点、磁盘和网络的backupgroup noise。每次失败可能有不同的持续时间和不同的爆炸半径。例如，一个节点可能暂时网络不可用，重新启动时临时停机，或者磁盘、节点、机架或主干网络交换机甚至数据中心的永久性故障。

在replicated system中容忍故障的一种方法是使用quorum-based投票协议。对于一个V个副本的数据，为了实现一致性必须遵守两个规则：

1. 每次读取必须知道最新的写入，公式为：R + W > V。该公式确保用于读取的节点集与用于写入的节点集相交

2. 每次写入必须知道最近的写入以避免写入冲突。公式为：W > V / 2

容忍单个节点丢失的常见方法是令(R，W，V) = (3, 2, 2)。然而，Aurora认为这是不够的。要了解原因，让我们首先了解AWS中可用区(AZ)的概念。AZ是Region的一个子集，通过低延迟链路连接到该Region中的其他AZ，但对大多数故障（包括电源、网络、软件部署、泛洪等）与其他AZ是隔离的。跨AZ分布数据副本可确保典型的大规模故障模式只影响一个数据副本。这意味着可以简单地将三个副本中的每一个放置在不同的AZ中，并且除了较小的单个故障之外还可以容忍大规模事件。

然而，在大型存储群中，故障的background noise意味着，在任何给定的时间，某些磁盘或节点子集可能已经发生故障并正在修复中。这些故障可能独立地分布在每个AZ A、B和C中的节点上。但是，由于火灾、屋顶故障、洪水等原因，这样background noise与AZ故障一起将会破坏仲裁模型，导致无法正确读取与写入。***仲裁模型需要容忍AZ故障以及同时发生的background noise故障***。

在Aurora中，我们需要满足一下两点：

1. 整个AZ故障加上一个额外节点故障（AZ+1），不会导致数据丢失

2. 整个AZ故障不会影响数据写入

Aurora的仲裁模型使用6个副本，跨越3个AZ，每个AZ有两个copies。即(R，W，V) = (3, 4, 6)，在这种模型下，即使一个AZ+1发生仍然可以读；即使两个节点故障（或者整个AZ故障）仍然可写。

### Segmented Storage

让我们考虑AZ+1是否提供足够的持久性的问题。为了提供足够的持久性，我们需要在故障修复期间尽量没有新的故障发生。即MTTF(Mean Time to Failure) > MTTR(Mean Time to repair)。如果双故障发生的概率足够高，这样当遇上AZ故障时，将会破坏仲裁模型。降低MTTF是很困难的，Aurora专注于降低MTTR。Aurora采用的策略如下：将数据分区成固定大小（10GB）的segment，每个segment有6个副本（称为一个PG: Protection Group），分布在3个AZ中。数据存储在带有SSD的EC2上（亚马逊的虚拟服务器）。

Segment就是系统探测失效和修复的最小单元。10GB的分段数据在10Gbps的网络连接上只需要10s就能传输完毕。根据我们的观察，两个副本同时失效、以及一个不包含这两个副本的AZ失效的概率非常非常低。

### Operational Advantages of Resilience

如果一个系统设计为可以适应long failures，那它自然的就可以适应shorter failures。例如，如果一个系统可以处理长时期的AZ故障，那么同样可以处理短期的停电故障或者因为软件问题而导致的回滚。

由于整个Aurora的设计针对故障保证了高可用性，那么利用这个高可用性也可以做很多运维操作。例如回滚一次错误的部署；标记过hot节点上的segment为已损坏，由系统重新将其数据分配到colder节点中去；以及停止节点为其打上系统和安全补丁。打补丁时会一个个AZ依次进行，保证同一时间同一PG内最多只有一个副本（所在节点）进行打补丁操作。

## THE LOG IS THE DATABASE

在这一节，我们将解释为什么在segmented replicated存储系统上使用传统数据库，由于网络IO和sync stall导致的性能负担。然后解释了我们的解决方法：将log处理流程下放到存储服务，并且通过实验证明了我们的方法可以大幅减少网络IO。最后，我们描述了我们在存储服务使用的各种技术来最小化sync stalls和不必要的写入。

### The Burden of Amplified Writes

我们将存储切分成segment并将每个segment复制6份，并以4/6的写入仲裁使得Aurora具有很大的弹性。不幸的是，这种方法会导致传统的数据库（例如MySQL）的性能不稳定，因为其每个application写入都会导致很多不同的实际IO。replication将会导致IO放大，从而导致沉重的PPS（packets per second）负担。同时，IO会导致同步点，而这些同步点将会导致pipeline stall并扩大latency。

我们来查看一下传统数据库是如何写入的。传统数据库（例如MysQL）向其objects（例如heap file、b-tree等）写入data pages，以及向WAL写入redo log。每个redo log record包括before-image和after-image之间修改page的difference，也就是说，一个log record可以用于before-image以生产出after-image。

实际上，其他的数据也必须写入。例如，考虑为了达到高可用性而做的跨data-center的MySQL同步镜像，该镜像以active-standby形式工作，如下图所示:

![](../images/mysql-mirror.jpg)

在图中，AZ1有一个active MySQL实例，其拥有一个位于EBS的网络存储。AZ2中有一个standby的MySQL实例，同样其也拥有位于EBS的网络存储。向primary EBS的写入会使用software mirroring的方式同步至standby EBS。

图中也展现了传统数据库需要写入的各种类型数据：redo log，写入Amazon S3用以定点恢复的二进制log，修改的data pages，临时的double write以及metadata files。IO flow的顺序如下：

- 在step1和step2，向EBS进行写入，并将其同步到AZ-local镜像。当所有写入都完成时将会收到通知

- 在step3，写入将会通过sync block-level软件镜像暂存至standby实例

- 最后，在step4和step5，将会写入standby EBS和其对应的镜像。

上面所讲的mirrored MySQL模式是不可取的，不仅仅是因为数据是如何写入的，还因为写入了什么样的数据。

- 首先，step 1/3/5是顺序的、同步的，因为很多写入是顺序的，所以latency是叠加的。这样的话，抖动将会放大，因为即使是异步写入也需要等待最慢的操作完成。对于一个分布式系统来说，该模型可以视为拥有4/4的写仲裁。

- 其次，user operations将会导致很多不同的写入，而这些写入以不同的形式代表着相同的信息。例如，为避免page撕裂，而向double write buffer的写入。

### Offloading Redo Processing to Storage

当传统的database修改一个data page时，其产生一个redo log record，并且唤醒log applicator来应用该log record，将其作用于before-image，以产生相应的after-image。事务提交需要log真正写入，但是data page的写入可以推迟。

***在Aurora中，唯一跨网络的写入是redo log records。从来没有从datatabase层写入pages。这一点对比传统的database优化还是很大的***

log applicator被下推到存储层，其可以在后台或者按需生产database pages。当然，根据完整的modification链去生产page是花费很高的，因此，我们持续在后台具化database pages以避免每次都根据需要重新生产他们。请注意，从正确性的角度来说，在后台进行具化操作是完全可选的：从引擎的角度来说，log就是数据库，存储系统具化的任意pages都是log的cache。另外需要注意，与checkpoint不同，仅仅有modification长链的pages需要具化。checkpoint是由整个redo log链的长度来控制，而Aurora page的具化是由该指定page的redo log链长度来控制。

尽管由于replication导致了写入放大，Aurora的方法依然大幅减少了网络负载，并且提供了性能和可用性。下图展示了一个Aurora集群，该集群中有一个primary instance和跨越多个AZ的多个副本。primary仅仅向存储服务写入log records，并将这些log records以及metadata更新发送到replica instance。IO基于共同的目的地（逻辑段，即一个PG），批量的发送全局有序的log records，并将每个batch发送至所有的6个副本，这些副本在本地对batch进行持久化，并且database engine等待4/6的ack（此时代表这些log records已经持久化或者harden）。replicas使用redo log对buffer caches进行apply。

![](../images/aurora-network-io.jpg)

经过实验对比，Aurora优化达成了其同一时间处理的事务量提高到了35倍，优化前每个事务处理的I/O数量是优化后的7.7倍，以及更多数据上的性能提高。性能的提高也意味着系统可用性的升级，降低了故障恢复时间。


