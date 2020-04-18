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

## 未完成