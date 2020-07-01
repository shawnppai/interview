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
1. spark-submit 提交代码，执行 new SparkContext()，在 SparkContext 里构造 DAGScheduler 和 TaskScheduler。
2. TaskScheduler 会通过后台的一个进程，连接 Master，向 Master 注册 Application。
3. Master 接收到 Application 请求后，会使用相应的资源调度算法，在 Worker 上为这个 Application 启动多个 Executor
4. Executor 启动后，会自己反向注册到 TaskScheduler 中。所有 Executor 都注册到 Driver 上之后，SparkContext 结束初始化，接下来往下执行我们自己的代码。
5. 每执行到一个 Action，就会创建一个 Job。Job 会提交给 DAGScheduler。
6. DAGScheduler 会将 Job 划分为多个 Stage，然后每个 Stage 创建一个 TaskSet。
7. TaskScheduler 会把每一个 TaskSet 里的 Task，提交到 Executor 上执行。
8. Executor 上有线程池，每接收到一个 Task，就用 TaskRunner 封装，然后从线程池里取出一个线程执行这个 task。(TaskRunner 将我们编写的代码，拷贝，反序列化，执行 Task，每个 Task 执行 RDD 里的一个 partition)

##### 10. Spark为什么快，Spark SQL 一定比 Hive 快吗
* 消除了冗余的 HDFS 读写: Hadoop 每次 shuffle 操作后，必须写到磁盘，而 Spark 在 shuffle 后不一定落盘，可以 persist 到内存中，以便迭代时使用。如果操作复杂，很多的 shufle 操作，那么 Hadoop 的读写 IO 时间会大大增加，也是 Hive 更慢的主要原因了。
* 消除了冗余的 MapReduce 阶段: Hadoop 的 shuffle 操作一定连着完整的 MapReduce 操作，冗余繁琐。而 Spark 基于 RDD 提供了丰富的算子操作，且 reduce 操作产生 shuffle 数据，可以缓存在内存中。
* JVM 的优化: Hadoop 每次 MapReduce 操作，启动一个 Task 便会启动一次 JVM，基于进程的操作。而 Spark 每次 MapReduce 操作是基于线程的，只在启动 Executor 是启动一次 JVM，内存的 Task 操作是在线程复用的。每次启动 JVM 的时间可能就需要几秒甚至十几秒，那么当 Task 多了，这个时间 Hadoop 不知道比 Spark 慢了多少。

考虑一种极端查询：Select month_id,sum(sales) from T group by month_id;

这个查询只有一次shuffle操作，此时，也许Hive HQL的运行时间也许比Spark还快。

##### 11. RDD 容错方式
Spark 选择记录更新的方式。但是，如果更新粒度太细太多，那么记录更新成本也不低。因此，RDD只支持粗粒度转换，即只记录单个块上执行的单个操作，然后将创建 RDD 的一系列变换序列（每个 RDD 都包含了他是如何由其他 RDD 变换过来的以及如何重建某一块数据的信息。因此 RDD 的容错机制又称血统容错）记录下来，以便恢复丢失的分区。lineage本质上很类似于数据库中的重做日志（Redo Log），只不过这个重做日志粒度很大，是对全局数据做同样的重做进而恢复数据。
相比其他系统的细颗粒度的内存数据更新级别的备份或者 LOG 机制，RDD 的 lineage 记录的是粗颗粒度的特定数据 transformation 操作行为。当这个 RDD 的部分分区数据丢失时，它可以通过 lineage 获取足够的信息来重新运算和恢复丢失的数据分区。

Spark checkpoint通过将RDD写入Disk作检查点，是Spark lineage容错的辅助，lineage过长会造成容错成本过高，这时在中间阶段做检查点容错，如果之后有节点出现问题而丢失分区，从做检查点的RDD开始重做Lineage，就会减少开销。  
　　checkpoint主要适用于以下两种情况：  
　　1. DAG中的Lineage过长，如果重算，开销太大，如PageRank、ALS等。  
　　2. 尤其适合在宽依赖上作checkpoint，这个时候就可以避免为Lineage重新计算而带来的冗余计算。

##### 12. 简述SparkSQL中RDD、DataFrame、DataSet三者的区别与联系?
* RDD: 一个RDD就是你的数据的一个不可变的分布式元素集合，在集群中跨节点分布，可以通过若干提供了转换和处理的底层API进行并行处理。
* DataFrame: DataFrame是一种以RDD为基础的分布式数据集，类似于传统数据库中的二维表格。DataFrame引入了schema。DataFrame的数据集都是按指定列存储，即结构化数据。类似于传统数据库中的表。
* DataSet：dataset整合了rdd和dataframe的优点，支持结构化和非结构化数据。Dataset是一个强类型的特定领域的对象，这种对象可以函数式或者关系操作并行地转换

联系与区别：
* RDDs 适合非结构化数据的处理，而 DataFrame & DataSet 更适合结构化数据和半结构化的处理；
* DataFrame & DataSet 可以通过统一的 Structured API 进行访问，而 RDDs 则更适合函数式编程的场景；
* 相比于 DataFrame 而言，DataSet 是强类型的 (Typed)，有着更为严格的静态类型检查；
* DataSets、DataFrames、SQL 的底层都依赖了 RDDs API，并对外提供结构化的访问接口

##### 13. map与flatMap的区别
* map：对RDD每个元素转换，文件中的每一行数据返回一个数组对象
* flatMap：对RDD每个元素转换，然后再扁平化，将所有的对象合并为一个对象，会抛弃值为null的值


##### 14. Repartition和Coalesce关系与区别
* 关系：
两者都是用来改变RDD的partition数量的，repartition底层调用的就是coalesce方法：coalesce(numPartitions, shuffle = true)
* 区别：
repartition一定会发生shuffle，coalesce根据传入的参数来判断是否发生shuffle

##### 15. cache和persist的区别
1. cache和persist都是用于将一个RDD进行缓存的，这样在之后使用的过程中就不需要重新计算了，可以大大节省程序运行时间；
2. cache只有一个默认的缓存级别MEMORY_ONLY ，cache调用了persist，而persist可以根据情况设置其它的缓存级别；
    * MEMORY_ONLY : 将 RDD 以反序列化 Java 对象的形式存储在 JVM 中。如果内存空间不够，部分数据分区将不再缓存，在每次需要用到这些数据时重新进行计算。这是默认的级别。
    * MEMORY_AND_DISK : 将 RDD 以反序列化 Java 对象的形式存储在 JVM 中。如果内存空间不够，将未缓存的数据分区存储到磁盘，在需要使用这些分区时从磁盘读取。
    * MEMORY_ONLY_SER : 将 RDD 以序列化的 Java 对象的形式进行存储（每个分区为一个 byte 数组）。这种方式会比反序列化对象的方式节省很多空间，尤其是在使用 fast serializer时会节省更多的空间，但是在读取时会增加 CPU 的计算负担。
    * MEMORY_AND_DISK_SER : 类似于 MEMORY_ONLY_SER ，但是溢出的分区会存储到磁盘，而不是在用到它们时重新计算。
    DISK_ONLY : 只在磁盘上缓存 RDD。
    * MEMORY_ONLY_2，MEMORY_AND_DISK_2，等等 : 与上面的级别功能相同，只不过每个分区在集群中两个节点上建立副本。
    * OFF_HEAP（实验中）: 类似于 MEMORY_ONLY_SER ，但是将数据存储在 off-heap memory，这需要启动 off-heap 内存。

##### 16. 如何理解spark中的血统概念
RDD在Lineage依赖方面分为两种Narrow Dependencies与Wide Dependencies用来解决数据容错时的高效性以及划分任务时候起到重要作用。

##### 17. SparkStreaming有哪几种方式消费Kafka中的数据，它们之间的区别是什么？
1. 基于Receiver的方式  
    这种方式使用Receiver来获取数据。Receiver是使用Kafka的高层次Consumer API来实现的。receiver从Kafka中获取的数据都是存储在Spark Executor的内存中的（如果突然数据暴增，大量batch堆积，很容易出现内存溢出的问题），然后Spark Streaming启动的job会去处理那些数据。

    然而，在默认的配置下，这种方式可能会因为底层的失败而丢失数据。如果要启用高可靠机制，让数据零丢失，就必须启用Spark Streaming的预写日志机制（Write Ahead Log，WAL）。该机制会同步地将接收到的Kafka数据写入分布式文件系统（比如HDFS）上的预写日志中。所以，即使底层节点出现了失败，也可以使用预写日志中的数据进行恢复。

2. 基于Direct的方式  
    这种新的不基于Receiver的直接方式，是在Spark 1.3中引入的，从而能够确保更加健壮的机制。替代掉使用Receiver来接收数据后，这种方式会周期性地查询Kafka，来获得每个topic+partition的最新的offset，从而定义每个batch的offset的范围。当处理数据的job启动时，就会使用Kafka的简单consumer api来获取Kafka指定offset范围的数据。

    优点如下：

    * 简化并行读取：如果要读取多个partition，不需要创建多个输入DStream然后对它们进行union操作。Spark会创建跟Kafka partition一样多的RDD partition，并且会并行从Kafka中读取数据。所以在Kafka partition和RDD partition之间，有一个一对一的映射关系。

    * 高性能：如果要保证零数据丢失，在基于receiver的方式中，需要开启WAL机制。这种方式其实效率低下，因为数据实际上被复制了两份，Kafka自己本身就有高可靠的机制，会对数据复制一份，而这里又会复制一份到WAL中。而基于direct的方式，不依赖Receiver，不需要开启WAL机制，只要Kafka中作了数据的复制，那么就可以通过Kafka的副本进行恢复。

    一次且仅一次的事务机制。

3. 对比：  
    基于receiver的方式，是使用Kafka的高阶API来在ZooKeeper中保存消费过的offset的。这是消费Kafka数据的传统方式。这种方式配合着WAL机制可以保证数据零丢失的高可靠性，但是却无法保证数据被处理一次且仅一次，可能会处理两次。因为Spark和ZooKeeper之间可能是不同步的。

    基于direct的方式，使用kafka的简单api，Spark Streaming自己就负责追踪消费的offset，并保存在checkpoint中。Spark自己一定是同步的，因此可以保证数据是消费一次且仅消费一次。

在实际生产环境中大都用Direct方式

##### 18. 简述Spark中共享变量（广播变量和累加器）的基本原理与用途。
累加器（accumulator）是Spark中提供的一种分布式的变量机制，其原理类似于mapreduce，即分布式的改变，然后聚合这些改变。累加器的一个常见用途是在调试时对作业执行过程中的事件进行计数。而广播变量用来高效分发较大的对象。

共享变量出现的原因：

通常在向 Spark 传递函数时，比如使用 map() 函数或者用 filter() 传条件时，可以使用驱动器程序中定义的变量，但是集群中运行的每个任务都会得到这些变量的一份新的副本，更新这些副本的值也不会影响驱动器中的对应变量。

Spark的两个共享变量，累加器与广播变量，分别为结果聚合与广播这两种常见的通信模式突破了这一限制。

##### 19. spark性能优化
[Spark性能优化指南——基础篇](https://tech.meituan.com/2016/04/29/spark-tuning-basic.html)  
[Spark性能优化指南——高级篇](https://tech.meituan.com/2016/05/12/spark-tuning-pro.html)


#### 20. 参数调优
spark参数设置：
1. num-executors：应用运行时executor的数量，推荐50-100左右比较合适
2. executor-memory：应用运行时executor的内存，那么申请的内存量最好不要超过资源队列最大总内存的1/3~1/2，推荐4-8G比较合适
3. executor-cores：应用运行时executor的CPU核数，推荐2-4个比较合适
4. driver-memory：应用运行时driver的内存量，Driver的内存通常来说不设置，或者设置1G左右应该就够了，主要考虑如果使用map side join或者一些类似于collect的操作，那么要相应调大内存量
5. spark.default.parallelism：每个stage默认的task数量，推荐参数为num-executors * executor-cores的2~3倍较为合适
6. spark.storage.memoryFraction：每一个executor中用于RDD缓存的内存比例，如果程序中有大量的数据缓存，可以考虑调大整个的比例，默认为60%
7. spark.shuffle.memoryFraction：每一个executor中用于Shuffle操作的内存比例，默认是20%，如果程序中有大量的Shuffle类算子，那么可以考虑其它的比例。