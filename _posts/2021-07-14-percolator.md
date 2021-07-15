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
