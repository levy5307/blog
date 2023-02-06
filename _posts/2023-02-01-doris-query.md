---
layout: post
title: Doris查询原理 
date: 2023-02-01
Author: levy5307
tags: []
comments: true
toc: true
---

同大多数数据库一样，Doris的查询主要分为查询接收、Parse（词法分析和语法分析）、Analyze（语义分析）、Rewrite（查询重写）、逻辑计划生成（单机执行计划）、物理计划生成（分布式执行计划）、Schedule、查询计划执行、查询结果返回等步骤。其中查询计划执行由BE负责，其他均由FE负责。

## 查询接收

在Doris中，`AcceptListener`负责监听mysql连接。当一个新连接到来时`AcceptListener`会创建一个`ConnectProcessor`类型对象。`ConnectProcessor`周期性的获取该连接上的请求，针对不同的command执行不同的操作，包括use database操作（COM_INIT_DB）、查询操作（COM_QUERY）、quit（COM_QUIT）等

## Parse

`ConnectProcessor::handleQuery`主要处理查询操作，随后使用Java CUP Parser对输入的字符串进行词法和语法分析。词法分析主要将输入的字符串解析成一系列token（例如select、from等），而语法分析则根据词法分析生成的token，生成一颗抽象语法树（Abstract Syntax Tree）

## Analyze

## Rewrite

## 逻辑计划生成

## 物理计划生成

## Schedule

## 查询计划执行

查询计划是由BE负责执行的，其执行引擎采用Batch模式的Volcano模型，相对于Tuple模式的Volcano，执行效率更高。

## 查询结果返回
