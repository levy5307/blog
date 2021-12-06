---
layout: post
title: Pegasus rpc调研
date: 2021-12-06
Author: levy5307
tags: []
comments: true
toc: true
---

最近pegasus考虑做rpc替换，因此对业内上一些rpc进行调研。我认为rpc选型需要满足以下几点：

1. 应用广泛。我们不希望选取了一个比较小众的rpc，过了一段时间之后，由于种种原因其不再维护，这样再切换的成本比较高。

2. 性能高。Pegasus作为一款KV存储，在公司内部定位就是快，即低延时。

3. 支持各种不同语言。Pegasus有各种语言的客户端，需要该rpc这些不同语言都有所支持。

鉴于第1点，我们将调研范围锁定在了thrift、grpc、brpc和dubbo之间。

## Dubbo

dubbo目前[主力支持golang和java](https://dubbo.apache.org/zh/docs/v2.7/user/languages/)，另外也支持Erlang。对cpp都不支持，显然不符合我们的需求。

## Grpc

我在这里将grpc放入考虑范围主要是因为：

1. grpc业内名气大、应用范围广

2. 鉴于Google的技术口碑，很多公司对grpc的发展十分看好，这其中就包括TiDB。

但是grpc的性能实在太差了，基本是在各个测评中处于被吊打的位置，甚至在brpc的benchmark中，grpc几乎在所有test case中垫底。

并且在调研的过程中，我们就发现TiDB就有篇[文章](https://cloud.tencent.com/developer/article/1372855)在考虑将grpc替换掉。另外Pegasus与TiDB不同，Pegasus对于延迟的敏感要明显强于TiDB，所以grpc过低的性能表现对我们来说是致命的。

另外grpc和thrift不兼容，如果引入grpc可能会带来兼容性问题。

因此建议忽略grpc。

## Brpc

brpc有如下几个优点：

1. 应用广。在百度内部就有数百万实例在使用，外部也有些产品在使用，比如字节跳动的bytegraph

2. 性能高。详细可以看[官方benchmark](https://github.com/apache/incubator-brpc/blob/master/docs/cn/benchmark.md)，总结来说，brpc相对于thrift和grpc，性能明显要高很多。

3. 兼容性好。brpc本身不绑定协议，可以使用很多种不同的协议。这意味着我们可以继续沿用thrift进行序列化，不用担心与老版本的兼容性问题。

4. 支持的语言足够多。当前支持Java、C++和Python。虽然对Pegasus来说缺少golang的支持，但是根据上一条，golang客户端可以继续沿用原有实现方式。

5. [对长尾友好](https://github.com/apache/incubator-brpc/blob/master/docs/cn/benchmark.md#%E6%B5%8B%E8%AF%95%E6%96%B9%E6%B3%95)。brpc的底线认为RPC必须能够处理长尾，这一点对以性能为卖点的KV存储非常重要。

6. [文档](https://github.com/apache/incubator-brpc/tree/master/docs/cn)丰富，便于理解与调优，易用性高（关于易用性，brpc可以通过浏览器或者curl[查看server内部状态](https://github.com/apache/incubator-brpc/blob/master/docs/cn/overview.md)，分析在线服务的cpu热点，内存分配等）。

7. 百度开源了基于brpc的[braft](https://github.com/baidu/braft)，如果后续Pegasus想要替换一致性协议，使用brpc可以更好的支持braft。

brpc存在的问题：

1. 之前Doris使用的brpc，经常导致线上机器coredump，需要去深入了解下具体情况。

不过总体来看，brpc还是很值得推荐的。

## Thrift

thrift有如下优点：

1. 支持[丰富的语言](https://github.com/apache/thrift/blob/master/LANGUAGES.md)。支持高达28种语言，包括C++/Java/Python/golang等

2. 目前我们的序列化使用的是thrift，如果rpc也使用thrift的话，与老集群的兼容性会比较好。

3. 使用广泛，很多互联网公司都使用了thrift。

缺点：

1. 性能不如brpc

2. [官方文档](https://thrift.apache.org/docs/)缺失。这一点对于用户不太友好

因此相较于thrift，还是更推荐brpc。

