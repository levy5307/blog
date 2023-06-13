---
layout: post
title: Doris stream load
date: 2023-02-01
Author: levy5307
tags: []
comments: true
toc: true
---

Stream Load是Doris的一种同步的导入方式, 允许用户通过Http访问的方式批量地将CSV或者JSON数据导入Doris，并返回数据导入的结果。用户可以直接通过Http请求的返回体判断数据导入是否成功，也可以通过在客户端执行查询SQL来查询历史任务的结果。Stream Load是是最常用的一种导入方式，在小米内部占了约80%以上场景。 

## 执行过程


用户执行stream load主要有两种方式：

- 将请求直接提交给be，并由该节点作为本次stream load任务的coordinator。

- 将http请求提交给fe，fe再通过http重定向将数据导入请求转发给某一个be节点，该be节点作为本次stream load任务的coordinator，此时的fe主要起到请求转发的作用。

本文中主要介绍第二种方式。其主要执行流程如下：

- 用户提交stream load请求到fe

- fe对http请求进行解析，然后进行鉴权。鉴权通过后，根据策略选取一台be作为coordinator，并将stream load请求转发给coordinator be（`StreamLoadAction::on_header`）

- coordinator be在接到请求后，会对其header信息进行校验，包括body长度、format类型等。

- coordinator be向fe发送`begin transaction的请求`，fe在接收到该请求时会开启一个事务，并向coordinator be返回事务id

- coordinator be向fe发送`TStreamLoadPutRequest`请求，fe在接收到该请求时，会产生导入执行计划，并向coordinator be返回。该执行计划非常简单，由`OlapTableSink`和`StreamLoadScanNode`两个算子组成，且只有一个`PlanFragment`

[](../images/streamload-plan.jpg)

- coordinator be在接收到导入计划之后，开始执行导入计划。
