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

基于key范围的分片方式对于范围查询的支持比较好，不过需要注意的是，该方式比较容易导致热点问题，例如：用户使用时间作为key，并且总是查询最近的数据，这样将会导致查询总是落在最新的key上面。

下面针对两种不同的分片方式进行了调研，看看行业内的实践是怎样的。

## hash分片

顾名思义，hash分片就是按照数据的某一特征（例如key），利用选定的hash算法，并将hash值与系统中的节点建立映射关系，从而将hash值不同的数据分布到不同的节点上。

其大致算法如下：

```
node index = hash(data) % server_count
```

举个例子。

在一个拥有3个节点的集群中，有如下一组数据data(key=6), data(key=7), data(key=8), data(key=11), data(key=12), data(key=13)，假设其采用的hash算法如下：`hash(data) = key`。那么`node index = key % 3`。因此当前的数据分布图如下：

![](../images/partition-hashkey.png)

这种方式的优点有：

- 实现简单，无需额外维护其他的元数据，根据公式便可以直接找到数据所在的server。

- 根据hash的特性，数据可以均匀的分布在不同的节点中，这样可以比较有效的避免数据倾斜。

当然，其缺点也是很明显的：

- 由于数据被打散，如果要进行范围查询则是比较吃力的，如上图中，如果要查询key在[11,13]范围之间的数据，则需要分别向node 1和node 2两个node发送请求，如文章开头所讲，不仅浪费资源，事务性也很难实现。

- 由于数据的分布与节点数高度相关，那么当扩缩容时，数据的搬迁量也是很巨大的。例如当我们扩容一台机器时，其映射算法变成如下：`node index = key % 4`，其数据分布如下：

![](../images/partition-hashkey-add-node.png)

从图中可以看到，数据搬迁的量还是比较大的。所以很多采用hash分片的实现中，节点扩展需要成倍增加，这样只移动50%的数据即可（Pegasus中便是如此，partition split需要将分片数翻倍，无法做到partition数量+1或者+2）。

为了克服这个问题，redis引入了逻辑节点的概念，用以屏蔽真实的server node。

### 逻辑节点


## 范围分片

