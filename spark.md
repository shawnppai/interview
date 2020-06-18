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
##### 3. 宽依赖和窄依赖
* 宽依赖：宽依赖是指父RDD的每个分区都可能被多个子RDD分区所使用。会有shuffle产生。宽依赖算子：reduceByKey grupByKey combineByKey，sortByKey, join(no copartition)
* 窄依赖：窄依赖是指父RDD的每个分区只被子RDD的一个分区所使用。不会有shuffle产生。窄依赖算子：filter map flatmap mapPartitions

    窄依赖在划分Stage时，可以划分在一起，而且可以并行计算，并且在数据恢复时只需要重新计算父RDD即可，恢复方便。而宽依赖则不然，因为宽依赖的范围较广，必须重新计算所有的父RDD依赖，计算量大，不容易恢复。

    窄依赖是一对一的，所以不需要shuffle，可以在一个节点内完成。
宽依赖是多对多或者多对一的，需要不同节点的数据。
##### 4. action和transform
* transform产生一个新的RDD，只会记录，而不会立即执行，遇到action才会从头开始计算。map，flatmap，groupby等
* action返回一个结果，会立即执行。count，first，take，reduce

##### 5. Spark 经常说的Repartition 有什么作用
一般上来说有多少个 Partition，就有多少个 Task，Repartition 的理解其实很简单，就是把原来 RDD 的分区重新安排。这样做避免小文件，减少 Task 个数，但是会增加每个Task处理的数据量，Task运行时间可能会增加

##### 6. 说说 Worker 和 Executor 的区别
Worker 是指每个工作节点，启动的一个进程，负责管理本节点，jps 可以看到 Worker 进程在运行，对应的概念是 Master 节点。 Executor 每个 Spark 程序在每个节点上启动的一个进程，专属于一个 Spark 程序，与 Spark 程序有相同的生命周期，负责 Spark 在节点上启动的 Task，管理内存和磁盘。如果一个节点上有多个 Spark 程序，那么相应就会启动多个执行器。所以说一个 Worker 节点可以有多个 Executor 进程。

##### 7. RDD, DAG, Stage, Task 和 Job 怎么理解？
* RDD： 是弹性分布式数据集。一个 RDD 代表一个可以被分区的只读数据集。RDD 内部可以有许多分区(partitions)，每个分区又拥有大量的记录(records)。
* DAG： Spark 中使用 DAG 对 RDD 的关系进行建模，描述了 RDD 的依赖关系，这种关系也被称之为 lineage（血缘），RDD 的依赖关系使用 Dependency 维护。
* Stage： 在 DAG 中又进行 Stage 的划分，划分的依据是依赖是否是 shuffle 的，每个 Stage 又可以划分成若干 Task。接下来的事情就是 Driver 发送 Task 到 Executor，Executor 线程池去执行这些 task，完成之后将结果返回给 Driver。
* Job： Spark 的 Job 来源于用户执行 action 操作（这是 Spark 中实际意义的 Job），就是从 RDD 中获取结果的操作，而不是将一个 RDD 转换成另一个 RDD 的 transformation 操作。
* Task： 一个 Stage 内，最终的 RDD 有多少个 partition，就会产生多少个 task

##### 8. 简单说说 Spark 支持的4种集群管理器

* Standalone 模式: 资源管理器是 Master 节点，调度策略相对单一，只支持先进先出模式，固定任务资源。
* Hadoop Yarn 模式: 资源管理器是 Yarn 集群，主要用来管理资源。Yarn 支持动态资源的管理，还可以调度其他实现了 Yarn 调度接口的集群计算，非常适用于多个集群同时部署的场景，是目前最流行的一种资源管理系统。
* Apache Mesos: Mesos 是专门用于分布式系统资源管理的开源系统，可以对集群中的资源做弹性管理。
* Kubernetes: K8S 是自 Apache Spark 2.3.0 引入的集群管理器，Docker 作为基本的 Runtime 方式。  
目前来说 Spark 的 Cluster Mode，Yarn 还是主流，K8S 则迎头赶上。

##### 9. Spark 作业提交流程是怎么样的
spark-submit 提交代码，执行 new SparkContext()，在 SparkContext 里构造 DAGScheduler 和 TaskScheduler。
TaskScheduler 会通过后台的一个进程，连接 Master，向 Master 注册 Application。
Master 接收到 Application 请求后，会使用相应的资源调度算法，在 Worker 上为这个 Application 启动多个 Executor
Executor 启动后，会自己反向注册到 TaskScheduler 中。所有 Executor 都注册到 Driver 上之后，SparkContext 结束初始化，接下来往下执行我们自己的代码。
每执行到一个 Action，就会创建一个 Job。Job 会提交给 DAGScheduler。
DAGScheduler 会将 Job 划分为多个 Stage，然后每个 Stage 创建一个 TaskSet。
TaskScheduler 会把每一个 TaskSet 里的 Task，提交到 Executor 上执行。
Executor 上有线程池，每接收到一个 Task，就用 TaskRunner 封装，然后从线程池里取出一个线程执行这个 task。(TaskRunner 将我们编写的代码，拷贝，反序列化，执行 Task，每个 Task 执行 RDD 里的一个 partition)

##### 10. Spark为什么快，Spark SQL 一定比 Hive 快吗


##### 11. RDD 容错方式