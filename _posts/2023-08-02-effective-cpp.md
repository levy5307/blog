---
layout: post
title: Effective C++复习笔记
date: 2023-08-02
Author: levy5307
tags: []
comments: true
toc: true
---

## 条款一：视C++为一个语言联邦

C++主要由4个次语言组成：

- C。C++仍是以C为基础

- Object-Oriented C++。这部分就是C with Classes

- Template C++。这部分是C++的模板编程

- STL

需要注意的是，每个部分的高效编程守则可能不太一样。例如：

- 对于C-like类型，pass-by-value比pass-by-reference更高效

- 对Object-Oriented C++部分，则pass-by-reference则会更高效

- Template C++更需要使用pass-by-reference，因为有时候你都不知道你在处理什么类型的对象

- STL部分，迭代器和函数对象都是在C指针上构造出来的，则使用pass-by-value会更高效。

## 条款二：尽量以`const`, `enum`, `inline`替换`#define`
