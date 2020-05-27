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

##### 8. Hive 中的压缩格式TextFile、SequenceFile、RCfile 、ORCfile各有什么区别？