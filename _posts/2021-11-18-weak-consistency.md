---
layout: post
title: 分布式系统中的弱一致性
date: 2021-11-18
Author: levy5307
tags: []
comments: true
toc: true
---

之前专门花时间调研过各种[强一致性的分布式协议](https://levy5307.github.io/blog/consensus-protocol-summary/)，却并未对弱一致性进行过深入研究，然而前段时间学习论文时候发现，其实很多存储系统还是采用的弱一致性，以获取较高的性能或者可用性。所以这里决定研究一下弱一致性。

## Dynamo

Dynamo是由Amazon开发的一款分布式KV存储，其设计目标是：

- 高可用性

因此Dynamo采用的一致性模型是最终一致性，当然最终一致可能会带来冲突的问题

- 永远可写

因为Amazon有很多购物场景，对于像“加入购物车”这种行为，Amazon肯定希望永远执行成功，因此其需要底层存储系统永远可写。

另外前面讲到，最终一致模型可能会带来冲突问题，所以何时解决冲突是需要考虑的一个问题。这里有两个选择：写时解决和读时解决。写时解决大部分系统所采取的方案，在这种方案中，读流程是比较简单。不过在这种系统中，如果写入不能到达全部或者大部分副本，写入将会被拒绝（如果只写入一个节点，冲突永远不会被发现）由于Dynamo需要保证永远可写，因此冲突解决是由读来完成的。

另外为了保证永远可写，Dynamo还采用了[Hinted Handoff](https://levy5307.github.io/blog/Dynamo/#handling-failures-hinted-handoff)等处理方法。


