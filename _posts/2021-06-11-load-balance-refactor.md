---
layout: post
title: pegasus load balance重构
date: 2021-06-11
Author: levy5307
tags: []
comments: true
toc: true
---

## 背景

现有的实现如下所示：

`simple_load_balancer`：主要用于cure，cure的目的是维护分片的健康（一主两备份）

`greedy_load_balancer`：用于生成load balancer计划。当前只有app load balance，后续要添加cluster load balance

`checker_load_balancer`：用于功能测试

![](../images/load-balancer-background.svg)

## 问题

现有实现有如下几个问题：

1. cure功能严格意义上来说已经不是load balance的一部分，在这里将其放入`simple_load_balancer`，使`server_load_balancer`的接口变得异常复杂，既包括cure的接口，也包括load balance的接口。从而间接导致`greedy_load_balancer`和`checker_load_balancer`变得很复杂。简单来说，这样***不符合单一职责原则***。

2. 目前`greedy_load_balancer`实现了针对表的负载均衡，其基于这样一个思想：当集群中所有的表都负载均衡时，整个集群便是负载均衡的。因此在对每个表做负载均衡时并没有考虑到集群当前的负载情况。这是绝对有问题的。因此我们想要加入一个cluster load balance的功能。由于`greedy_load_balancer`已经很臃肿了，无法往里面再堆逻辑了。

## 重构

如上所述，cure功能严格意义上来说已经不是load balance的一部分了，所以首先把`simple_load_balancer`中cure的功能抽出来放入一个新创建类`partition_healer`，专门用于cure。load balance相关的功能放入`greedy_load_balancer`。这样`simple_load_balancer`便可以删掉。

![](../images/load-balancer-refactor-step1.svg)

另外，由于要加入cluster load balancer，不希望将这部分功能加入`greedy_load_balancer`了，因为这样会导致`greedy_load_balancer`过于臃肿。并且加入了cluster load balancer之后，暂时先不希望把原先的load balance删掉，所以短期内两者应该会并存。

所以想将其抽出来放入一个单独的类中。

考虑了如下几种实现思路：

### 继承

![](../images/load-balancer-inherit.svg)

将`greedy_load_balancer`的功能迁移到`app_load_balancer`中，集群负载均衡功能添加到`cluster_load_balancer`中。对于两者之间的公共代码，放入`greedy_load_balancer`。

***问题：***由于app load balance和cluster load balance策略同时存在，需要通过配置在两者之间切换，这样便将这些细节暴露给了`meta_service`。我希望这些细节对`meta_service`是透明的，load balance的问题，在自己内部解决，与`meta_service`无关。

### Decorator

![](../images/load-balancer-decorator.svg)

由于原有的负载均衡策略还保留，只是新增了一个cluster负载均衡功能，自然就想到了Decorator。实现一个新类`cluster_load_balancer`，作为`greedy_load_balancer`的Decorator，其内部可以通过一个变量来配置是使用`cluster_load_balancer::balance`还是`greedy_load_balancer::balance`，该变量可以通过远程变量控制。`meta_service`内部创建一个`cluster_load_balancer`就可以了。

***问题：***`cluster_load_balancer`和`greedy_load_balancer`里有大量重复代码和公用成员变量，使用这种方式不好复用。对于这种复用的情况，最好还是用继承

### Strategy

![](../images/load-balancer-strategy.svg)

实现一个新的接口`load_balance_policy`，其有两个子类，分别实现app load balance和cluster load balance，在`greedy_load_balance`内部通过一个配置来控制使用哪个policy。
`meta_service`持有一个`greedy_load_balancer`对象，无需关心使用哪个policy。

## 最后

最后实现类图如下：

![](../images/load-balancer-final.svg)

另外，在load balance的执行过程中，包括primary copy和secondary copy操作，这两个的操作流程大体是相同的，只有部分细节的差异，可以使用模板方法模式来实现，具体可以参考[模板方法模式在Pegasus load balance中的应用](https://levy5307.github.io/blog/code-refactor/#%E6%A8%A1%E6%9D%BF%E6%96%B9%E6%B3%95%E6%A8%A1%E5%BC%8F)

