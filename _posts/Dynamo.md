---
layout: post
title: Dynamo
date: 2021-08-09
Author: levy5307
tags: [论文]
comments: true
toc: true
---

Dynamo是Amazon实现的一款KV存储。为了支持Amazon大规模且持续增长的用户，具有高可用、高扩展性的特性。Dynamo有一个重要的设计目标：允许应用自己控制自己的系统特性（例如持久性和一致性）让应用自己决定如何在功能、性能和成本效率之间取得折中。

## System Assumptions and Requirements
