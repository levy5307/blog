---
layout: post
title: pegasus cure
date: 2020-03-29
Author: Levy5307
tags: [pegasus load balance]
comments: true
toc: true
---

pegasus对于每个partition采用一主两备份的方式来进行管理，该一主两备被称作一个replica group。当meta server对replica group进行修改时，有可能会产生primary不可用、secondary数量确实等情况，此时则需要对该replica group进行修复，该过程叫做cure。
meta server中使用定时任务来定期检查各个replica group的状态信息以对replica group进行修复。

## Primary不可用
当primary不可用时，首先要考虑的是从secondary中选择一个将其提升为primary。然而此时secondary也有可能不存在，所以也要根据实际情况进行分类解决。

### Secondary数量>0
此时则在所有的secondary中选择一个提升为primary，选取的规则如下：
1. 对比该app下的primary数量，选择拥有较少数量的节点对应的secondary
2. 如果两个相等，则选择该app的partition数量最少的
3. 还是相等的话，则选择所有app下primary数量最少的
4. 如果还是相等，则选择所有app下partition数量最少的
因为产生决策时，而真正执行则需要一段时间。因此，这里的数量统计不仅仅要考虑正在服务的replica，也要包括将来要服务的replica。

### 新创建的partition(last_drop为空)
当secondary数量为0时，也有可能是该partition是新创建的，此时只需要在partition中选择一个提升为primary就可以了。具体的选择方法和上述相同。

### 该partition的所有replica都不可用
当secondary数量为0，并且该partition并非是新创建的，则表明此时该partition中的所有replica都不可用。
***待补充***

## 缺少Secondary

## 多Secondary
这种情况是由于负载均衡策略导致的，此时只要选择出这些secondary所在的node中partition数量最少的一个node，将其对应的secondary remove就可以了。


## 未完待续

