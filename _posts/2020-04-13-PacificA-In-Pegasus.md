---
layout: post
title: PacificA In Pegasus
date: 2020-04-14
Author: Levy5307
tags: [pegasus, PacificA]
comments: true
toc: true
---

PacificA是微软实现的一款强一致性的分布式共识协议，具有简单易实现、可用性高的优点。Pegasus就是使用PacificA协议来维护多副本之间的复制。

之前有篇文章讲过PacificA的原理与理论，如有疑惑请移步PacificA(https://levy5307.github.io/blog/PacificA/)

## Read & Write
根据PacificA算法，读操作的处理是比较简单的，其只需要primary在本地读取到最新的值就可以了，secondary并不参与其中，这里不再赘述。但是对于写则会复杂很多，因为需要经过主从之间的2PC来实现，这里主要对写做一些讲解。

![Write流程](../images/pegasus-pacifica-write-process.png)

写入流程如上图所示。这里加几点说明：

1. 对于同一个replica的写处理是单线程的，也就是说，对于每一个replica，只有一个线程在处理写请求。

2. 上图中的WAL分为private log和shared log，其中private log保存在内存中，而shared log在磁盘上。在pegasus中，每个replica对应一个RocksDB，由于一个机器上有多个replica，那么如果打开了RocksDB的WAL的话，写入的时候将会在多个文件之间切换，导致随机写。所以在Pegasus中采用了shared log的机制来替代RocksDB的WAL。但是使用shared log也有个问题，就是当recover一个replica的时候需要做log split操作。所以Pegasus采用了shared log + private log的方式，private log保存在内存中，当添加potential secondary时，直接使用private log，对于rivate log缺失的部分通过重放shared log补全。

## failure detector

PacificA中，错误探测是通过primary定期向secondary发送beacon来实现，这里是假设primary和secondary在不同的机器上，这样发送beacon才有意义。然而在pegasus里却不满足这种条件，即：一台机器上既有primary、也有secondary，所以Pegasus对于错误探测机制这里做了一些简单的修改。beacon的发送不是在primary和secondary之间，而是修改成了在meta server和primary server之间，由primary server主动向meta server发送beacon，时间间隔默认为3s。具体执行如下图所示：

```
                             |--- lease period ----| lease IsExpired, commit suicide
                 |---- lease period ----|
    replica: ---------------------------------------------------------------->
                 \      /    \     /       _\
               beacon ack   beacon ack       x (beacon deliver failed)
                  _\/         _\/
     meta : ---------------------------------------------------------------->
                    |----- grace period -----|
                                |---- grace period -----| grace IsExpired, declare worker dead
```

对于lease period和grace period是否expired，pegasus分别在replica server和meta server的failure detector中创建了一个定时任务去定时检查，该定时任务的时间间隔会比较小，便于及时发现expired的情况。

### meta server侧

当超过lease period的时间没有收到meta group的ack时，replica server则认为meta group不可用了。此时该replica server会将其之上的所有的replica（不论是primary还是secondary）状态都设置成暂时性不可用（PS_INACTIVE和_inactive_is_transient，但是ballot不变）, 这里这样做主要是为了维持PacificA中的***Primary Invariant***, 防止出现多主。(NOTE：为什么secondary也要设置为inactive)

这里需要对meta group作一下解释: replica是与整个meta server group发送beacon的，但是发送不是发给该group中的所有meta，而是在meta group中选择出一个leader并与其通信。当与其通信过程中发生通信错误时，则切换leader，与另外的meta server进行通信。当grace period的时间内没有收到leader的ack信息时，则认为整个meta group不可用。

当与meta server恢复通信后，则将与meta server同步最新配置，获取replica server上所归属的replica(primary+secondary)

### replica server侧

当meta server发现某replica server的grace period过期时，会认为该replica server已经宕机了，此时meta会将该replica server上的所有primary和secondary降级为inactive。

对于primary降为inactive的情况: 
1. 首先meta需要在replica group中将当前primary设置为不可用，同时将ballot + 1。由于metaserver使用zookeeper对数据进行持久化, 所以需要将该partition的最新配置发送至zookeeper去更新
2. 更新本地配置，即更新node_state，从node_state上移除该primary
3. 更新load balancer。当前primary移除掉后，需要修改load balancer的信息。该信息是指：每个gpid都有其所在的server列表(三副本则为三台server)，这里修改信息是指将该primary对应的server从上述列表中移除。
4. 触发cure操作，由于该replica group没有了primary，需要触发cure操作来"治愈"该replica group。

***NOTE:*** 这里先通过cure获取“治愈”所需要执行的迁移动作（目标server node、动作类型等等），然后通过向该目标server node发送send_proposal来执行该迁移动作。例如：这里就是选取一个secondary，并向其发送一个CT_UPGRADE_TO_PRIMARY类型的proposal

对于secondary降为inactive的情况则较为简单: 
1. 向该secondary所在的primary发送CT_DOWNGRADE_TO_INACTIVE的proposal（ballot没有变化）
2. primary接收到该请求时，从secondaries中移除该secondary
3. 该primary向meta server发送更新配置的请求，更新最新配置。

发送proposal的流程：
1. meta向replica发送proposal
2. replica根据自身状态以及proposal信息做一些检查
3. 由于replica使用meta server进行持久化，所以如果检查通过，replica向meta server发送持久化新配置的请求
4. 当第3步的请求成功后，replica更新本地配置

而当replica server恢复正常后，此时则仅将该replica server标记为active，等待下次进行load balance时会将一部分primary和secondary迁移过来。

和PacificA算法一样，Pegasus同样令grace period > lease period，所以一定是replica server先发现beacon通信失败、而先于meta server做出响应。这样做是为了达到***Primary Invariant***，使replica先设置其为inactive，从而防止出现多primiary的情况发生。

