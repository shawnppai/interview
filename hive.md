##### 1. hive执行过程

1. 用户提交查询等任务给Driver。
2. 编译器获得该用户的任务Plan。
3. 编译器Compiler根据用户任务去MetaStore中获取需要的Hive的元数据信息。
4. 编译器Compiler得到元数据信息，对任务进行编译，先将HiveQL转换为抽象语法树，然后将抽象语法树转换成查询块，将查询块转化为逻辑的查询计划，重写逻辑查询计划，将逻辑计划转化为物理的计划（MapReduce）, 最后选择最佳的策略。
5. 将最终的计划提交给Driver。
6. Driver将计划Plan转交给ExecutionEngine去执行，获取元数据信息，提交给JobTracker或者SourceManager执行该任务，任务会直接读取HDFS中文件进行相应的操作。
7. 获取执行的结果。
8. 取得并返回执行结果

##### 2. hive编译过程
1. Antlr定义SQL的语法规则，完成SQL词法，语法解析，将SQL转化为抽象语法树AST Tree
2. 遍历AST Tree，抽象出查询的基本组成单元QueryBlock
3. 遍历QueryBlock，翻译为执行操作树OperatorTree
4. 逻辑层优化器进行OperatorTree变换，合并不必要的ReduceSinkOperator，减少shuffle数据量
5. 遍历OperatorTree，翻译为MapReduce任务
6. 物理层优化器进行MapReduce任务的变换，生成最终的执行计划

##### 3. hive内部表和外部表的区别

建表:
* 内部表 create table a，数据加载过程时数据会被移动到hive的仓库中
* 外部表 create external table，数据加载时数据被保存在location指定的位置，并不会被移动到hive的仓库中

删除时:
* 内部表 hdfs上的数据和元数据信息会被一起删除
* 外部表 hdfs上的的数据信息不会被删除，只删除元素信息，实际数据信息存储再location指定的位置

其他操作都相同

##### 4. hive的order by,sort by, distribute by和cluster by的区别

* order by: 全局排序，针对的是全部的数据，所有数据在一个reducer中进行排序。如果在严格模式下(hive.mapred.mode=strict)，则必须配合limit使用
* sort by：数据进入reducer前排序，保证数据在每个reducer中的顺序。如果有多个reducer，那么最终结果可能部分有序。
* distribute by：数据分组，将数据按照by后的字段进行分组，属于同一个分组的进入一个reducer。保证相同的key进入同一个reducer。但是不能保证数据的顺序。
* cluster by：是distribute by和sort by的结合。
Cluster By 和 Distribute By一般用在transform中较常使用。

##### 5. hive的metastore的三种模式

* 内嵌Derby方式：这个是Hive默认的启动模式，一般用于单元测试，这种存储方式有一个缺点：在同一时间只能有一个进程连接使用数据库。
* Local方式：本地MySQL。需要确保hive执行的节点可以访问到Mysql。本地模式不需要单独起metastore服务，用的是跟hive在同一个进程里的metastore服务。每一个hive客户端都会链接到数据库进行元数据信息的查询。
* Remote方式：远程MySQL,一般常用此种方式。远程元存储需要单独起metastore服务，然后每个客户端都在配置文件里配置连接到该metastore服务。远程模式的metastore服务和hive运行在不同的进程里，metastore和hive客户端使用Thrift通信。

##### 6. hive的metastore有什么作用

* hive的database，table等元数据信息都是通过metastore来访问的。
客户端连接metastore服务，metastore再去连接MySQL数据库来存取元数据。
* 有了metastore服务，就可以有多个客户端同时连接，而且这些客户端不需要知道MySQL数据库的用户名和密码，只需要连接metastore服务即可。

##### 7. hive中join都有哪些

Hive中除了支持和传统数据库中一样的内关联（JOIN）、左关联（LEFT JOIN）、右关联（RIGHT JOIN）、全关联（FULL JOIN），还支持左半关联（LEFT SEMI JOIN）

##### 8. left semi join是什么，怎么使用
left semi join：作伴链接，相当于in条件句，以join的方式实现，不过select子句中只能有一个列，且会自动过滤，条件需要写道on子句中，不能写在where子句中。
例如:`select user_id, user_name from tbl where user_id in (select t.user_id from t where login_date > '2020-01-01')`可以写成 `select user_id, user_name from tbl left semi join t on tbl.user_id = t.user_id and t.login_date > '2020-01-01'`

##### 9. Hive 中的压缩格式TextFile、SequenceFile、RCfile 、ORCfile各有什么区别？

* TextFile: 默认存储方式，行储存，数据不做压缩，磁盘开销大，数据解析开销大，压缩后无法分片，无法对数据进行并行操作。可以直接存储，加载数据的速度最高。
* SquenceFile：二进制存储，行存储，以key-value的形式序列化到文件中。可以分片，可以压缩。
* RCFile：一种行相结合的存储方式。数据按行分块，按列存储。同一行的数据位于同一节点，因此元组重构的开销很低，块内列存储，可以进行列维度的数据压缩，跳过不必要的列读取
* ORC：列式存储，以二进制方式保存。是RCFile的改良版。
> 1. ORCFile按行分为多个stripes，然后在每个stripe内数据以列为单位进行存储，所有列的内容保存在同一个文件中。每个stripe默认为250MB。
> 2. ORC支持多种压缩(NONE, ZLIB, SNAPPY，默认为ZLIB)，并且可以切分.
> 3. ORC可以支持复杂的数据结构(如map，struct，list)
> 4. 并且提供了多种索引(row group index、bloom filter index)，使数据可以快速读取。

##### 10. 所有的Hive任务都会有MapReduce的执行吗？
不是，类似`select * from tbl where partition = patition1 limit n`的这种就不需要起MapReduce job，直接通过Fetch task获取数据。

##### 11. Hive的函数：UDF、UDAF、UDTF的区别？
* UDF(User Defined Function)：普通用户自定义函数，单行进，单行出。继承UDF或者GenericUDF，后者比前者可以处理复杂的数据类型，如Map、List、Struct。
* UDAF(User Defined Aggregate Function)：用户自定义聚合函数，多行进，单行出。实现GenericUDAFEvaluator接口。
* UDTF(User Defined Table-Generating Functions): 用户自定义表生成函数，单行进，多行出。继承GenericUDTF实现。

##### 12. 说说对Hive分区和分桶的理解？
* 分区：分区HDFS表现就是文件夹，是将数据按照数据特点进行区分，不同的文件夹中保存不同的数据，可以避免hive查询中扫描全部文件，起到隔离数据和缩小查询范围的作用。比如常见的时间分区和业务分区，不同时间分区下，保存不同时间点产生的数据。
> 1. 单值分区：可以划分为静态分区和动态分区，静态分区在数据导入时需要手动指定分区值，动态分区系统可以自动判断。动态分区需要开启`set hive.exec.dynamic.partition=true`。有多个分区的时候，动态分区需要在静态分区之后，如果所有分区都是动态分区，需要闭关严格模式`set hive.exec.dynamic.partition.mode=nonstrict`。
> 2. 范围分区：单值分区每个分区对应于分区键的一个取值，而每个范围分区则对应分区键的一个区间，只要落在指定区间内的记录都被存储在对应的分区下。分区范围需要手动指定，分区的范围为前闭后开区间 [最小值, 最大值)。最后出现的分区可以使用 MAXVALUE 作为上限，MAXVALUE 代表该分区键的数据类型所允许的最大值

* 分桶：分桶是更细粒度的数据划分，是通过哈希值来将数据均匀的划分为多个桶。
> 1. hash值除以桶的个数进行求余，决定该条记录存放在哪个桶中.
> 2. 物理上，每个桶就是表(或分区）目录里的一个文件。
> 3. 分桶可以获得更高的查询处理效率，比如join尤其是map-side join的时候，保存相同的列值的桶进行join就可以。另一个是非常适合数据抽样。
> 4 .分桶插入数据时需要开启hive.enforce.bucketing属性；或者需要设置和分桶数相同的reducer的数量mapred.reduce.tasks，使用Distribute by … Sort by进行排序。

##### 13. hive有那些抽样方式?
* 数据块抽样:
   1. tablesample(n percent) 根据hive表数据的大小按比例抽取数据，不适用于所有的格式，这种抽样最小的单元时一个HDFS数据块，如果表大小小于数据块大小的话，就会返回所有行。可以通过`set hive.sample.seednumber=<INTEGER>;`从不同的数据块进行抽样。
   2. tablesample(n M) 指定抽样数据的大小，单位为M；和 n percent采样有同样的限制。
   3. tablesample(n rows) 指定抽样数据的行数，其中n代表每个map任务均取n行数据，map数量可通过hive表的简单查询语句确认
* 分桶抽样：`TABLESAMPLE (BUCKET x OUT OF y [ON colname])`，利用分桶表，随机分到多个桶里，然后抽取指定的一个桶，如果已经分桶且on colname为cluster by的列，那么查询只会扫描对应桶中的数据。随机且速度快。
* 随机抽样：利用rand()函数，进行抽样。比如`order by rand() limit n`或者`cluster by rand() limit n`或者`where rand() < float_num distribute by rand() sort by rand() limit n`

##### 14. 说一下hive的窗口函数
聚合函数是分组后，将多行的值聚集为一行，每组的结果只有一个。有时候既想要聚合前的数目，也想要聚合后的值，这便引入了窗口函数。
窗口函数分析/序列函数 + over函数组成，over函数中函数中的分区、排序、指定窗口范围可组合使用也可以不指定。
窗口函数在join，select，having之后执行
* 分析函数：sum，avg，max，min，count
* 序列函数：row_number，rank，dense_rank，ntile，lag，lead，first_value和last_value


##### 15. hive row_number，rank和dense_rank
* row_number的排序是连续且不重复的，即使值相等，序号也不会相同。比如对A A B排序，序号依次为1，2，3
* rank的排序是间断且重复的，值相等，则序号相同，但是重复几次，后面的序号就空出几个。比如对A A B排序，序号依次为1，1，3
* dense_rank的排序是连续且重复的，值相等，则序号相同，后面的序号在前面的基础上加1。比如对A A B排序，序号依次为1，1，2


##### 16. 描述数据中的null,在hive底层如何存储
null在hive底层默认是用"\N"来存储的

##### 17. 了解hive server2吗？
HS2是一个服务端接口，能够使远程客户端链接并执行查询操作并返回结果，目前的实现版本是基于thrift rpc调用，hiveserver2是hiveserver的改良版本，支持多客户端的并发和认证，能够更好的支持jdbc及odbc等开放的客户端api

##### 18. 知道CBO吗？
* RBO基于规则优化
   * 谓词下推：将过滤数据的操作放在join之前，减小join的数据量
   * 常量累加：将多个常量的值进行累加，得出结果替代原来的计算式，以减少计算量。如：x+2+1直接替换为x+3
   * 列值裁剪：只扫描需要的列值，对所需要的列之外的列进行那个裁剪，减少网络和内存的消耗，提高扫描效率
* CBO基于代价优化：评估SQL语法树每个节点的执行代价，然后将所以节点的代价累计，选出执行代价最小的一条路径。
   1. 采集原始表的基本信息：输出数据大小，总条数，基本列信息等
   2. 定义核心算子的基数推导规则： 在当前子节点统计信息的基础上，计算父节点相关统计信息的一条推导规则。
   3. 核心算子实际代价计算：CPU COST和IO COST
   4. 选择最优执行路径(代价最小执行路径)，动态规划

##### 19. mapper，reducer数量由什么决定的？如何调整？
* mapper的调节：mapper无法直接调节，但是可以设置分片的大小来间接的影响。
   * `mapreduce.input.fileinputformat.split.minsize` 文件分片最小的有效字节数
   * `mapreduce.input.fileinputformat.split.maxsize` 文件分片最大的有效字节数
   * `dfs.blocksize` HDFS中文件块的大小

   利用公式`splitSize = max(minimumSize, min(maximumSize, blockSize))`，可以通过改变上述三个参数来调节最终的分片大小。
   > 需要注意点的是，在调节分片大小时，如果分片过小，会增加管理分片的总时间和构建map任务的总时间，则会增加 整个任务的执行时间。 同时分片大小也不应大于块大小，因为此时无法确保存储有该切片的多个数据块位于单个节点中，会增加执行map任务中的网络传输。 因此对于大多数作业来说，一个合理的分片大小趋向于HDFS的一个块的大小，默认是128MB

* reducer的调节：可以通过设置`hive.exec.reducers.bytes.per.reducer`每个reducer处理的数据大小来间接影响数量，或者可以通过`set mapreduce.job.reduces=number`来设置reducer的数量。Reducer个数应该设置为0.95或者1.75乘以节点数与每个节点的容器数的乘积
   > 当乘数为0.95时，map任务结束后所有的reduce将会立刻启动并开始转移数据， 此时队列中无等待任务，该设置适合reudce任务执行时间短或者reduce任务在个节点的执行时间相差不大的情况; 当乘数为1.75时，运行较快的节点将在完成第一轮reduce任务后，可以立即从队列中取出新的reduce任务执行， 由于该reduce个数设置方法减轻了单个reduce任务的负载，并且运行较快的节点将执行新的reduce任务而非空等执行较慢的节点，其拥有更好的负载均衡特性。

##### 20. 什么情况下只有一个reducer
* 用了Order by
* 笛卡尔积
* 数据量小于hive.exec.reducers.bytes.per.reducer参数值

##### 21. shuffle的作用是什么？简单描述一下原理
按照partition、key对map生成的中间结果进行排序合并，输出给reduce线程。
* map端：map端输出首先写入一个环形内存缓存区，当达到一个百分比时，数据开始写入磁盘，形成溢出文件spill file，写入过程由另外的线程来执行，不影响缓存正常的缓存写入。 在写入磁盘之前，会按照partition、再按照key进行排序。 如果设置了combine操作，如果至少存在3个spill file文件，那么会再merge过程执行combine操作。 merge是对多个spill file文件进行合并，确保一个map task最终只生成一个中间文件数据。 决定数据写入那个partition的方法是，对key进行hash,再以reduce task数量取模，然后到指定的job
* reduce端：map端完成后，会使用心跳机制通知master，master有map和主机的对应关系，reduce会定期询问master以获取map位置。reduce进程启动copy线程，通过HTTP方式请求map task输在的Task Tracker获取map task的输出文件。获取的会先放入内存缓存区中，达到一定的阈值后，开发时merge，把merge输出写到磁盘。该reduce的map输出全部拷贝完成后，开始进行合并操作，即磁盘到磁盘merge，merge进行合并排序

##### 22. 列转行和行转列的函数分别有那些？
* 列转行：collect_set，concat_ws，
* 行转列：主要通过lateral view explode来实现

##### 23. 数据倾斜的原因以及解决方法