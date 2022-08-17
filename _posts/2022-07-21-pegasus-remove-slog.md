---
layout: post
title: pegasus dual-WAL架构优化
date: 2022-07-21
Author: levy5307
tags: []
comments: true
toc: true
---

读WiscKey论文的时候，了解到SSD的写入具有一定的并行性。遂对SSD做了一些调研，发现果真如此。另外考虑到Pegasus的双WAL架构，以slog写入为准，而slog是单线程写入的。这导致完全利用不上SSD的并行性。因此考虑对slog做移除。

## 可行性调研

经过深入研究pegasus代码，发现涉及到slog的功能主要有learn、duplication、partition split以及2pc

### 2pc

需要将plog提交改成同步提交。

### learn

- learnee

learnee只会使用plog，不会从slog同步数据跟leaner，因为plog没有的数据可以从prepare_list中获取，不会有数据遗漏。

- learner

learner（on_learn_reply函数）会将获取的mutation apply到plog和slog，这里需要做修改。

### 热备（duplication）

使用的完全是plog，与slog无关。

### partition split

copy parent时使用private log，只是在child写入时需要同时写入plog和slog。

### 版本升级&回退

- 升级

true data应该是在slog里的，在升级时的启动过程，log回放还是需要从slog中获取。```因此slog相关代码还需要保留，只是不再写入。```

- 回退

回退时会导致plog中数据比slog中更新，需要能正确回放log数据。

在replica_init.cpp的replay_mutation主要用于执行log mutation回放：

- 对每个replica回放其对应plog

- 当所有replica回放完plog之后，读取slog，对其中保存的mutation进行回放。如果某个mutation的decree小于plog中max decree，则直接忽略该mutation。

因此该机制可以保证，在回退到老版本时，依然可以通过回放plog获取最新数据。

需要注意的点：在回放时，不会将slog缺失的数据补足，即slog中间会有空洞。因此在回放完成后，slog+plog才代表true data

- 升级中途

此时，部分replica server的true data在slog，另外一部分在plog。

只要确保log数据能够正确写入和读取就没有问题。

- 升级中途回退

如果升级过程遇到问题，需要回退到老版本。此时会导致如下情况：

- 部分replica的true data在slog（这部分replica没有被正常打开）

- 部分replica的true data在plog（这部分replica被正常open且对写请求进行了响应）

同版本回退，replay_mutation机制可以保证回放到最新的数据。

### slog gc

gc逻辑主要是匹配所有slog文件的```max_decree```与min{```plog_durable_decree```，```app_durable_decree```}。如果小于则该文件可以删除；否则需要保留。

***所以即使有回退等情况导致的slog空洞，理论上也不影响数据的正确性***，只要slog文件中的mutation的decree是有序的即可。

Note: 升级之后，系统会逐渐gc掉slog文件，直到还剩最后一个。

### 其他

```replica_init_info```保存了有每个replica在slog中的起始地址。主要包括如下几个方面：

- 持久化到.info文件中

- 从.info文件中load数据

- 更新slog起始地址

可以维持原有逻辑不变。

### 结论

目前可以做到不向slog写入，并将plog写入改成同步写，无法全部将slog相关代码删除。

## 性能测试

在移除slog后，plog的保存依然有两种方案：

- plog统一放在一个ssd盘(home盘)上

- plog放在多个ssd盘(data盘)，这样不仅可以利用ssd的并行性，还可以利用多盘的并行性，提高写入吞吐

因此测试性能时，对这两种方案分别进行了测试。

### master分支（保留slog）

|      operation_case      | run_time | throughput | length | read_write | thread_count |       read(qps|ave|min|max|95|99|999|9999)       |     write(qps|ave|min|max|95|99|999|9999)      |
| --- | --- | --- | --- | --- | --- | --- | --- |
| write=single,read=single | 12982    | 23108      | 1000   | 0 : 1      | 15           | {0 0 0 0 0 0 0 0}                                | {23108 1944 412 515241 5265 10553 24484 38153} |
| write=single,read=single | 3695     | 244278     | 1000   | 1 : 0      | 50           | {244316 612 119 353535 969 2084 20393 35273}     | {0 0 0 0 0 0 0 0}                              |
| write=single,read=single | 5627     | 42651      | 1000   | 1 : 1      | 30           | {21328 1424 129 1111380 5469 20425 95060 155604} | {21326 2784 418 550228 7383 12399 37279 68009} |
| write=single,read=single | 5013     | 29915      | 1000   | 1 : 3      | 15           | {7479 1084 128 1347924 3065 15969 84169 141567}  | {22439 1639 407 255615 3979 7665 21596 37012}  |
| write=single,read=single | 3488     | 25793      | 1000   | 1 : 30     | 15           | {830 987 153 811689 2795 12929 75935 143828}     | {24966 1765 412 271700 4196 8500 19433 28137}  |
| write=single,read=single | 4093     | 73317      | 1000   | 3 : 1      | 30           | {54992 845 118 457727 2798 9303 36057 56009}     | {18328 2360 415 456191 5900 10385 30020 50996} |
| write=single,read=single | 4560     | 197394     | 1000   | 30 : 1     | 50           | {191035 732 119 172628 1461 4051 17439 27193}    | {6369 1470 440 130943 2480 5995 19623 30948}   |


### plog in data

|      operation_case      | run_time | throughput | length | read_write | thread_count |     read(qps|ave|min|max|95|99|999|9999)      |     write(qps|ave|min|max|95|99|999|9999)      |
| --- | --- | --- | --- | --- | --- | --- | --- |
| write=single,read=single | 10538    | 28464      | 1000   | 0 : 1      | 15           | {0 0 0 0 0 0 0 0}                             | {28466 1577 339 447572 4808 9529 21057 31903}  |
| write=single,read=single | 3420     | 265243     | 1000   | 1 : 0      | 50           | {265272 567 118 166548 884 1799 16427 23087}  | {0 0 0 0 0 0 0 0}                              |
| write=single,read=single | 4398     | 54556      | 1000   | 1 : 1      | 30           | {27282 724 123 179284 2433 7693 23727 33321}  | {27280 2567 351 474623 7309 12553 34591 60809} |
| write=single,read=single | 4071     | 36835      | 1000   | 1 : 3      | 15           | {9209 617 131 253225 1626 6852 21545 30025}   | {27629 1418 344 564393 4054 7904 19201 30495}  |
| write=single,read=single | 2831     | 31775      | 1000   | 1 : 30     | 15           | {1025 702 155 248788 1721 8023 29839 43103}   | {30757 1436 341 308393 4029 8521 19097 27551}  |
| write=single,read=single | 3410     | 87997      | 1000   | 3 : 1      | 30           | {66002 680 120 168404 1912 6281 21188 32169}  | {21999 2038 344 432895 5556 10164 30020 54239} |
| write=single,read=single | 4143     | 217312     | 1000   | 30 : 1     | 50           | {210345 668 116 179284 1374 3876 16865 23241} | {7010 1233 376 133353 2054 5217 18625 24799}   |

### plog in home

|      operation_case      | run_time | throughput | length | read_write | thread_count |     read(qps|ave|min|max|95|99|999|9999)      |     write(qps|ave|min|max|95|99|999|9999)      |
| --- | --- | --- | --- | --- | --- | --- | --- |
| write=single,read=single | 12674    | 23668      | 1000   | 0 : 1      | 15           | {0 0 0 0 0 0 0 0}                             | {23669 1898 387 575828 5411 10455 23983 36287} |
| write=single,read=single | 3347     | 269283     | 1000   | 1 : 0      | 50           | {269348 555 121 151508 867 1438 16021 19000}  | {0 0 0 0 0 0 0 0}                              |
| write=single,read=single | 5126     | 46811      | 1000   | 1 : 1      | 30           | {23407 600 128 146943 1426 5945 20548 30708}  | {23408 3236 399 587775 9396 15871 42324 74004} |
| write=single,read=single | 4859     | 30863      | 1000   | 1 : 3      | 15           | {7712 527 130 320297 919 5117 19105 27636}    | {23152 1763 394 464639 4712 9924 23199 34889}  |
| write=single,read=single | 3371     | 26690      | 1000   | 1 : 30     | 15           | {859 566 155 190377 960 4981 21252 31743}     | {25831 1719 391 152831 4335 9175 20751 29716}  |
| write=single,read=single | 3750     | 80008      | 1000   | 3 : 1      | 30           | {60016 569 122 310868 1179 4841 17724 27908}  | {20003 2777 415 577364 7352 13444 33855 54121} |
| write=single,read=single | 4118     | 218765     | 1000   | 30 : 1     | 5            | {211735 656 116 176937 1313 3533 16956 23524} | {7053 1471 456 150100 2435 5804 20839 30159}   |

### 总结对比

***plog in data(remove slog) v.s. master分支（保留slog）***

以mater分支为基准：

- 吞吐提高约20%

利用了ssd的并行性，符合理论预期

- 写延迟降低约8%

写延迟降低，是因为以前单线程写时，每个写向磁盘flush时，都需要排队很长时间。remove slog之后，可以多队列并行写，排队时间减少。

- 读延迟降低约36%

读延迟降低这么多确实是始料未及的，猜测原因是由于写延迟降低、写对集群的负载减少，从而间接提高了读的性能

***plog in home v.s. plog in data***

以plog in data为基准：

- 吞吐下降约10%

- 读延迟降低约16%

- 写延迟升高约15%

***plog in home（remove slog） v.s. master（保留slog）***

以master分支为基准：

- 吞吐提高约7%

- 读延迟降低约45%

- 写延迟升高约10%

***结论***

以master分支（保留slog）为基准，得出如下表格

|  Scenario    |  throughput  |   读延迟    |   写延迟   |
| ---- | ---- | ---- | ---- |
| plog in data |    +20%      |    -36%    |    -8%    |
| plog in home |     +7%      |    -45%    |   +10%    |

从而可得：

- plog in data(remove slog)性能最好

- master分支（保留slog）和plog in home(remove slog)各有优劣，master分支写性能高，plog in home读性能高。

