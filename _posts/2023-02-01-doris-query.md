---
layout: post
title: Doris查询原理 
date: 2023-02-01
Author: levy5307
tags: []
comments: true
toc: true
---

同其他数据库一样，Doris的查询主要分为Parse（词法分析和语法分析）、Analyze（语义分析）、Rewrite（查询重写）、逻辑计划生成（单机执行计划）、物理计划生成（分布式执行计划）、Schedule、查询计划执行等步骤。其中查询计划执行由BE负责，其他均由FE负责。

## 查询接收

## Parse

## Analyze

## Rewrite

## 逻辑计划生成

## 物理计划生成

## Schedule

## 查询计划执行

查询计划是由BE负责执行的，其执行引擎采用Batch模式的Volcano模型，相对于Tuple模式的Volcano，执行效率更高。
