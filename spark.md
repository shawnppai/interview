##### 1. spark shuffle
有些运算需要将各节点上的同一类数据汇集到某一节点进行计算，把这些分布在不同节点的数据按照一定的规则汇集到一起的过程称为 Shuffle。spark的shuffle是在DAGSchedular划分Stage的时候产生的。
shuffle操作伴随着磁盘IO，数据的序列化和反序列，以及网络IO。
分为Shuffle Write和Shuffle Read
* Shuffle Write
    * BypassMergeSortShuffleWriter：当shuffle reduce task(即partition)数量小于spark.shuffle.sort.bypassMergeThreshold参数的值且没有map side aggregations触发
    > 每个task都会为下游的reduce task创建一个临时文件，按照key的hash存入对应的临时文件中，首先会将数据存缓存再内存中，当缓存满溢时写入磁盘。最后会对所有临时文件合并成一个磁盘文件，并创建一个索引文件标识下游各个reduce task的数据在文件中的start offset和end offset。
    * SortShuffleWriter
    > 该模式下，数据首先写入一个内存数据结构中，此时根据不同的shuffle算子，可能选用不同的数据结构。如果是reduceByKey这种聚合类的shuffle算子，那么会选用Map数据结构，一边通过Map进行聚合，一边写入内存；如果是join这种普通的shuffle算子，那么会选用Array数据结构，直接写入内存。如果数据到达了阈值，就会触发写入磁盘。在写入磁盘前，会根据key对内存数据结构中已有的数据进行排序。最后对溢写磁盘形成的多个临时文件进行合并，还会单独写一份索引文件，其中标识了下游各个task的数据在文件中的start offset与end offset。
    * UnsafeShuffleWriter
    > Serializer支持relocation。这是指Serializer可以对已经序列化的对象进行排序，这种排序起到的效果和先对数据排序再序列化一致。没有map side aggregations。shuffle reduce task(即partition)数量不能大于支持的上限(2^24)
    UnsafeShuffleWriter 将record序列化后插入sorter，然后对已经序列化的record进行排序，并在排序完成后写入磁盘文件作为spill file，再将多个spill file合并成一个输出文件。在合并时会基于spill file的数量和IO compression codec选择最合适的合并策略。

* Shuffle Read
通过请求 Driver 端的 MapOutputTrackerMaster 询问 ShuffleMapTask 输出的数据位置，当 Parent Stage 的所有 ShuffleMapTasks 结束后再 fetch。因为 Spark 不要求 Shuffle 后的数据全局有序，因此没必要等到全部数据 shuffle 完成后再处理，所以是边 fetch 边处理。刚获取来的 FileSegment 存放在 softBuffer 缓冲区，经过处理后的数据放在内存 + 磁盘上。
内存使用的是AppendOnlyMap ，类似 Java 的HashMap，内存＋磁盘使用的是ExternalAppendOnlyMap，如果内存空间不足时，ExternalAppendOnlyMap可以将 records 进行 sort 后 spill（溢出）到磁盘上，等到需要它们的时候再进行归并


##### 2. spark join
* shuffle hash join：适合一张大表和一张小表jion
    1. shuffle阶段：分别将两个表按照join key进行分区，将相同join key的记录重分布到同一节点，两张表的数据会被重分布到集群中所有节点。这个过程称为shuffle
    
    2. hash join阶段：每个分区节点上的数据单独执行单机hash join算法。
* broadcast hash join：适合一张大表和一张极小表join
    1. broadcast阶段：将小表广播分发到大表所在的所有主机。广播算法可以有很多，最简单的是先发给driver，driver再统一分发给所有executor；要不就是基于bittorrete的p2p思路；

    2. hash join阶段：在每个executor上执行单机版hash join，小表映射，大表试探；
* sort merge join：SparkSQL对两张大表join采用了全新的算法
    1. shuffle阶段：将两张大表根据join key进行重新分区，两张表数据会分布到整个集群，以便分布式并行处理

    2. sort阶段：对单个分区节点的两表数据，分别进行排序
    
    3. merge阶段：对排好序的两张分区表数据执行join操作。join操作很简单，分别遍历两个有序序列，碰到相同join key就merge输出，否则取更小一边
