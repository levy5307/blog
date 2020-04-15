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
当secondary数量为0，并且该partition并非是新创建的，则表明此时该partition中的所有replica都不可用。此时则从inactive中选取一个最新的replica，将其promote为primary

***NOTE：*** inactive的replica是指config_context的dropped成员。当replica发生状态变化时，例如：downgrade to secondary、upgrade to secondary、assign primary时，replica server会将该replica状态同步给meta server，当收到某个replica变为inactive的通知时，meta server会将其保存到config_context::dropped中
调用链路：
```
               通知
replica server ----> meta server --> simple_load_balancer --> config_context
```
另外, partition_context::last_drops与config_context::dropped是相对应的，dropped保存inactive的replica, last_drops保存的是inactive的replica所在的node

## 缺少Secondary

对于缺少secondary的情况，也是从inactive中选取一个，将其提升为secondary。但是具体选取哪个inactive replica，却有两种情况。
### emergency

所谓emergency，就是满足以下四个条件之一：
- 一个partition允许最大的replica数量> 2pc要求的最小数量，并且当前partition的replica数量 < 2pc要求的最小数量。也就是说，挂掉的节点过多，导致不满足2pc要求
- inactive列表为空
- 最后一个inactive replica挂掉的时间过长
- 最后一个inactive replica所在的节点在黑名单中
 
此时，系统中为每个config_context维护了一个prefered_dropped变量，该变量表示上次选取的inactive replica在dropped中的坐标，下次选取的时候从prefer_dropped-1开始选取。此时选取的逻辑如下所示：
- 首先，从prefer_dropped-1开始，选取所在的节点状态为active(即没有挂掉)的第一个inactive replica
- 如果上述选取的inactive replica没有在黑名单中，并且其地址是valid，则选取成功
- 否则，则需要从系统中的所有节点中，则选取出partition所在的节点，从中选取选取replica最少的节点，将该节点上的inactive replica提升为secondary
	
### non-emergency

当不满足以上四个条件时，此时就是non-emergency。此时只需要简单的选取最后一个inactive replica，将其提升为secondary就可以了。

## 多Secondary
这种情况是由于负载均衡策略导致的，此时只要选择出这些secondary所在的node中partition数量最少的一个node，将其对应的secondary remove就可以了。

