---
layout: post
title: tcp slow start调研
date: 2021-07-22
Author: levy5307
tags:
comments: true
toc: true
---

## 背景

有用户反应在系统刚启动时的请求耗时会比较长，所以针对这个问题进行了调研。了解到了tcp的slow start特性，并对此进行了测试。

将测试分为同机房测试和跨机房测试。

## 跨机房测试

使用get operation, value=1kB大小，使用c3 client to c4 server进行了跨机房访问，请求100次。

```
recv/send buffer大小: 512 Bytes                                                                                                                                                                                                         
average = 190478 us, max = 214527 us, min = 1601 us
耗时比较大，前面的请求(约几百个)基本都在200ms左右，后面的有所减少，慢慢减少到了1.6ms
 
recv/send buffer大小: 默认情况(4 * 1024)。其中，{1: latency}代表第一个请求的延时
1: latency = 12102 us
2: latency = 1871 us
average = 1789 us, max = 12102 us, min = 1528 us
 
recv/send buffer大小: 8 * 1024
1: latency = 14661 us
2: latency = 2015 us
average = 1830 us, max = 14661 us, min = 1581 us
 
recv/send buffer大小: 16 * 1024
1: latency = 12206 us
2: latency = 1906 us
average = 1750 us, max = 12206 us, min = 1534 us
 
recv/send buffer大小: 32 * 1024
1: latency = 12390 us
2: latency = 1975 us
average = 1823 us, max = 12390 us, min = 1571 us
 
recv/send buffer大小: 64 * 1024
1: latency = 13539 us
2: latency = 1931 us
average = 1831 us, max = 13539 us, min = 1550 us
```

根据上面的测试，发现两个情况：

1. 当buffer大小 < 数据大小时，最开始的一批请求都会非常慢，随后的请求会变快

2. 当buffer大小 > 数据大小时，第一个请求会比较慢，第2个rpc之后的耗时明显变小。此时即使再增大buffer大小，请求耗时也不会减少，即此时的请求耗时和设置的recv/send buffer大小没关系

根据第2条，可以看出第2个rpc之后的耗时明显减少，所以考虑加入fake rpc来做warm up。加入之后，请求100次情况：

```
1: latency = 3723 us
2: latency = 1896 us
average = 1714 us, max = 3904 us, min = 1561 us
```

所以添加fake rpc，对于提升第一次请求速度确实有帮助。

***总结：*** 

1. 通过调研得知，tcp协议是有slow start算法的，即最初的时候会设置滑动窗口比较小，后续根据网络状况再调整滑动窗口大小。但是仅仅是改变客户端和服务端两边的buffer大小是没有太大意义的，因为我们无法改变中途所经过的路由器的滑动窗口大小。此时整个链路的瓶颈是在中途所经过的路由器上，所以即使再增大buffer大小，请求耗时也不会减少。但是如果将服务断/客户端的buffer设置过小时（远小于系统默认值），此时的瓶颈在服务端/客户端，所以请求最开始会很慢，随后随着窗口变大也会变得越来越快。

2. 发送fake rpc是非常有效的，因为该fake rpc会经过整个请求的链路，且tcp的slow start算法对于滑动窗口是指数级增长的，所以会将整个链路的滑动窗口大小都调大，从而减少后续的请求耗时。

## 同机房测试

另外，使用ycsb也进行了同机房访问测试(c4 client to c4 server)：


未使用opentable（用于拉取表路由信息）进行warm up(value大小=1KB):

```
recv/send buffer大小: 512 Bytes                                                                                                                                                                                                         
average = 211.22 us, max = 136703 us, min = 97 us, count = 572286
 
recv/send buffer大小: 默认情况(4 * 1024)
average = 195.31 us, max = 136959 us, min = 96 us, count = 614644
 
recv/send buffer大小: 8 * 1024
average = 201.28 us, max = 140543 us, min = 97 us, count = 595120
 
recv/send buffer大小: 16 * 1024
average = 202.53 us, max = 136063 us, min = 96 us, count = 594949
 
recv/send buffer大小: 32 * 1024
average = 197.24 us, max = 131711 us, min = 95 us, count = 606562
 
recv/send buffer大小: 64 * 1024
average = 204.27 us, max = 143231 us, min = 100 us, count = 590427
```

可见请求耗时和设置的窗口大小无关。

使用opentable进行warmup后，没有fake rpc

```
recv/send buffer大小: 64 * 1024
average = 223.86 us, max = 36991 us, min = 79 us, count = 535692
```

可见，进行warmup后，相比未进行warmup，效果有了很大提升。

使用opentable进行warmup并发送fake rpc

```
recv/send buffer大小: 64 * 1024
average = 302.69 us, max = 48895 us, min = 82 us, count = 266561
```

结果显示请求耗时和设置的窗口大小无关，和是否发送fake rpc也无关。但是提前opentable拉去表配置的效果提升很大

***总结：*** 由于同机房内部网络状况都会比较好，tcp协议默认此时不会出现丢包等情况，所以滑动窗口一开始就会设置成最大，即使我们手动设置也是如此。所以此时不管发送fake rpc还是改变buffer大小，都对请求耗时没有影响

