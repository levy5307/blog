---
layout: post
title: pegasus load balance
date: 2020-03-29
Author: Levy5307
tags: [pegasus load balance]
comments: true
toc: true
---

Pegasus的负载均衡由meta server进行全局控制，其最小单元是replica。具体分为以下两个方面：

* cure：当某些原因导致一个replica groupo不满足一主两备时，meta server会根据相应的策略进行调整，其中包括把不足的备份补全，或者把多余的备份剔除。这种策略叫做cure
* balancer: meta server会定期所有replica server节点的replica情况做评估，当其认为replica在节点分布不均衡时，会将相应replica进行迁移。

如下两篇文章分别对这两种情况进行了介绍。

* [cure](https://levy5307.github.io/blog/pegasus-cure/)
* [balancer](https://levy5307.github.io/blog/pegasus-balancer/)
