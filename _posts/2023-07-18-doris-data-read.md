---
layout: post
title: Doris数据读取流程
date: 2023-07-18
Author: levy5307
tags: []
comments: true
toc: true
---

Doris的底层数据读取是通过`OlapScanNode`发起的。当调用`OlapScanNode::get_next`时：

- 如果当前该`OlapScanNode`是非start状态时，则开始执行start_scan流程。

- 如果`_materialized_row_batches`为empty，并且传输没有完成时，则等待1秒钟之后继续查看，直到不为空

- 获取到row batch后，放入到返回参数`row_batch`中

前面提到，第一次执行`get_next`时，会触发start_scan流程：

- 根据conjuncts构建`ColumnValueRange`、olap filter以及scan keys 

- 根据一系列参数获取`ranges_per_scanner`，并根据`ranges_per_scanner`以及scan ranges，构建一些`OlapScanner`，每个`OlapScanner`负责某一确定的tablet的部分ranges

