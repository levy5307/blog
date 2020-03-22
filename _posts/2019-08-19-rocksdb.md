rocksdb\_verbose\_log:是否打印与rocksdb相关的一些调试日志                                                                                                                                                            

rocksdb\_abnormal\_get\_time\_threshold_ns: 如果get操作的时长超过了该数值，那么将会被写入日志。0代表不会写入

rocksdb\_abnormal\_get\_size\_threshold: 如果get操作获取的value的长度大于了该数值，那么将会被写入日志。0代表不会写入

rocksdb\_abnormal\_multi\_get\_time\_threshold\_ns:  如果multi-get操作的时长超过了该数值，那么将会被写入日志。0代表不会写入

rocksdb\_abnormal\_multi\_get\_size_threshold: 如果multi-get操作的key-value的size之和超过了该数值，那么将会被写入日志。0代表不会写入

rocksdb\_abnormal\_multi\_get\_iterate\_count\_threshold: 如果multi-get操作的key-value的数量超过了该数值，那么将会被写入日志。0代表不会写入

rocksdb\_write\_buffer_size:单个memtable的最大size。一旦memtable大小超过该数值，将会被标记为不可修改的，并且会创建一个新的memtable。然后，一个后台线程会把memtable的内容落盘到一个SST文件。

rocksdb\_max\_write\_buffer\_number: memtable的最大数量，包括active-memtable和immutable-memtable，如果active memtable被填满，并且memtable的总数量大于该数值，那么将会被延缓写入。

当刷新进程执行的比写入更慢时，有可能会发生这种情况。

rocksdb\_max\_background_flushes:支持的并发的后台flush线程数量。flush线程在高优先级的线程池中

rocksdb\_max\_background_compactions:支持的并发的后台compaction线程数量。conpaction线程在低优先级的线程池中

rocksdb\_num\_levels: rocksdb的LSM tree的层数

rocksdb\_target\_file\_size\_base and rocksdb\_target\_file\_size\_multiplier:

level 1层的文件大小最大为target\_file\_size\_base字节。每高一层其文件大小比前面一层大的target\_file\_size\_multiplier倍。默认情况下rocksdb\_target\_file\_size\_multiplier是１，也就是说每层的文件大小相同。

增大rocksdb\_target\_file\_size\_base将会导致每层的文件数量减少。我们推荐将rocksdb\_target\_file\_size\_base设置为max\_bytes\_for\_level\_base / 10，这样level 1层将最大有10个文件

rocksdb\_max\_bytes\_for\_level\_base and rocksdb\_max\_bytes\_for\_level\_multiplier:

rocksdb\_level0\_file\_num\_compaction_trigger: 如果level 0中的文件数量超过了该指定数值，L0->L1 compaction将会被触发

rocksdb\_level0\_slowdown\_writes\_trigger:  如果level 0中的文件数量超过了该指定数值，那么写入速度将会被降低到rocksdb.delayed-write-rate

rocksdb\_level0\_stop\_writes\_trigger:如果level 0中的文件数量超过了该指定数值，那么写入将会被禁止

rocksdb\_compression\_type:压缩算法类型。

压缩算法类型, 支持 none，snappy，lz4，zstd几种选项。支持为每一层单独配置压缩算法，用逗号分隔，如:“none,none,snappy,zstd” 表示L0，L1不进行压缩，L2使用snappy压缩，L3往下使用zstd压缩。

rocksdb\_disable\_table\_block\_cache: 如果该值被设置为true,则表示禁用block cache功能

rocksdb\_block\_cache\_capacity and rocksdb\_block\_cache\_num\_shard\_bits:

rocksdb\_block\_cache_capacity: block cache总的内存使用量

rocksdb\_block\_cache\_num\_shard\_bits: shard block cache所使用的bit数量. block cache分很多shard, 便于并发操作, shard的数目为2^rocksdb\_block\_ache\_num\_shard\_bits.

rocksdb\_disable\_bloom_filter: 如果该值被设置为true,则表示禁用bloom filter功能
