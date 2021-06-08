---
layout: post
title: 关于数据分区的一些思考
date: 2021-06-06
Author: levy5307
tags: []
comments: true
---

目前团队所做的kv存储，对于数据分区所采用的方式是hash分区：对key取hash值，获取的hash值再对partition取模。即：```hash(key) % partition_count```

最近在思考这种方式的实现问题。

显然，由于这种方式对于连续的key可能获取到不同的hash值, 导致并不一定能分配到同一个分片上。所以对于范围查询非常不友好，需要向多个分片发送请求才能保证获取到该范围内的所有key，并且由于跨多个节点，事务性比较难保证。为了解决这个问题，团队项目采用了两级key的方式，即将key分为hashkey和sortkey：hashkey用于分区映射，同一个hashkey下数据按照sortkey排序。这样同一个hashkey下的sortkey便可以存储于同一个partition下，并且同一个hashkey下的sortkey便可以支持范围查询。

但是这样也带来一些问题：

- hashkey+sortkey的方式并不符合主流的kv接口设计(如redis)，导致接口设计完全不同。

- 这种方式相当于一定程度上把数据分区交给了用户，也就是说，如果用户使用不当，将大量sortkey放置于同一个hashkey下，很容易产生热点partition。

- 对于跨hashkey的范围查询支持依然不够好。

其实要想支持范围查询，尚不如使用基于key范围的分片方式（HBase所采用的方案）。

不过需要注意的是，基于key范围分片的方式也存在热点问题，例如：用户使用时间作为key，并且总是查询最近的数据，这样将会导致查询总是落在最新的key上面。

