---
layout: post
title: Doris数据读取流程
date: 2023-07-18
Author: levy5307
tags: []
comments: true
toc: true
---

Doris的底层数据读取是通过`OlapScanNode`发起的。当调用`OlapScanNode::get_next`时，如果当前该`OlapScanNode`是非start状态时，则开始执行start_scan流程。
