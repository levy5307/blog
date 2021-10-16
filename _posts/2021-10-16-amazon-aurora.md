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

IT workloads正在逐渐向公有云上迁移，这种迁移的重要原因在于能够按需提供灵活容量的能力。在当前的分布式云服务中，弹性和扩展性是由存储和计算层分离来实现的。

传统数据库当前面临的IO瓶颈已经发生了变化，因为IO可以分布在多个节点和多个磁盘上，因此单个的磁盘和节点不再过热，***当前的瓶颈变为了存储和计算层之间的IO***。除了基本的每秒包数(PPS)和带宽瓶颈之外，由于性能，数据库将并行地向存储系统发出写操作，这将导致流量放大的问题。离群存储节点、磁盘或网络路径的性能可能会影响响应时间。

尽管很多数据库操作可以互相重叠，但是很多视乎我们需要同步操作，这将导致stall和上下文切换。例如，当缓存未命中所导致的磁盘读取操作，当读取完成之前，该读取线程无法继续执行。cache miss同样会导致其他penalty，例如cache evict和flush dirty page，以安置new page。background processing例如checkpointing和dirty page writing可以减轻这种penalty的发生，但是同样会导致stall，context switch和resource contention。

事务提交是另一个干扰源，提交一个事务时的暂停可能会阻碍其他事务的进展。在cloud-scale的分布式系统中，使用多阶段同步协议(如两阶段提交2PC)处理提交是一个挑战。这些协议不能容忍故障，而大规模分布式系统有持续不断的硬故障和软故障。同时它们还具有高延迟的问题，因为大规模系统分布在多个数据中心。

