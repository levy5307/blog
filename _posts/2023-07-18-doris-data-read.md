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

我们把过滤条件按照最外层的AND拆分之后的单元叫做conjunction，比如`((A & B) | C) & D & E`就是由`((A & B) | C)`，`D`，`E`三个conjunction组成的。之所以这么定义是因为，conjunction是是否下推到存储的最小单元。一个conjunction里面的条件要么都下推，要么都不下推。

- 根据一系列参数获取`ranges_per_scanner`，并根据`ranges_per_scanner`以及scan ranges，构建一些`OlapScanner`，每个`OlapScanner`负责某一确定的tablet的部分ranges

- 对新创建出来的`OlapScanner`，分别执行prepare。在prepare中，获取tablet reader，并根据fe传递过来的数据version，获取[0, version]之间的所有rowset的reader

- 然后，创建一个线程，执行`transfer_thread`，该线程在后台异步的scan底层数据。

- 在`transfer_thread`中，首先根据`_max_materialized_row_batches`（最大物化batch数） / `config::doris_scanner_row_num`（一次read最多读取的行数） / `state->batch_size()`（一个batch的大小）得到执行scan的最大线程数量。然后在确保不超过前述最大线程数的前提下，给`OlapScanner`分配一个线程执行`OlapScanNode::scanner_thread`。

- 当`transfer_thread`中创建完`scanner_thread`后，等待`_scan_row_batches`中有row batch时，将row batch放入到`_materialized_row_batches`中，供`OlapScanNode::get_next`消费。

