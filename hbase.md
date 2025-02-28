##### 1. Hbase处理的有那两种文件？
WAL预写日志和数据文件
##### 2. Hbase查询一次数据的流程是什么？
在zookeeper中查询行键，Zookeeper返回-ROOT- region服务器，服务器返回.META.表中对应的region服务器，region返回数据并缓存
##### 3. HBASE写数据的流程是什么样的
写的请求会被交付给对应HRegion实例，预写日志，写到Memstore，HDFS中的Hfile持久化
##### 4. 预写日志有什么用？
预写日志包括了序列号和实例数据，可以在HBASE发生问题的时候恢复数据

##### 5. HBASE最小的存储单元是什么？其结构如何
Hbase最小的存储单元是HFile，HFile中的数据已Key-Value的形式存储，Key-Value本质上是一个较为低级的字节数组，key的字段包括row_length, row, column family length, column family, column qualifier, time stamp, key type

##### 6. Hbase对于列中的控制是如何处理的？
Hbase中没有None的概念，如果没有值，不会再对应的地方存储None，而是会将真个列省略。

##### 6. Hbase底层存储的时候，是如何存储单元格的？
底层数据在存储的时候，按列族线性地存储了单元格，同时单元格包含了所有它必要的信息。一个列族下的所有单元格都存储在一个存储文件中，store file，不同列族的单元格是不会出现在同一个存储文件中的。一个列族的所有列都存储在HFile中。

##### 7. key-value存储是的排序规则？
先拿行键排序，当一行有多个单元格时内部按列键排序。

##### 8. 高表和宽表？
选择该表，尽可能的将信息隐藏到行键中，因为Hbase是按行键进行分片的，按行键将不同的数据写到不同的机器上

##### 9. 什么叫部分键扫描？
部分键扫描是指通过行键的一部分来获取和所有包含这部分行键的完整行键对应的数据
比如，行键设计为real_disease_id+patient_id + uniq_record_id
那么，通过real_disease_id + patient_id可以获取该疾病和该病人所有的病例数据。
注意，必须要保证行键中的每个字段达到统一的长度，否则可能会溢出。
需要指定开始和结束行键。

##### 10. 为什么Hbase过滤器会很慢？
Hbase过滤器，甚至是行过滤器，大多数情况下，会对全表进行扫描，然后对这些结构进行过滤。但是，行键范围扫描会快的多，他们相当于过滤后的扫描。

##### 11. 描述 Hbase 的 rowKey 的设计原则
* rowkey 长度原则
    > rowkey 是一个二进制码流，可以是任意字符串，最大长度 64kb，实际应用中一般为 10-100bytes，以 byte[]形式保存，一般设计成定长。建议越短越好，不要超过 16 个字节， 原因如下: 数据的持久化文件 HFile 中是按照 KeyValue 存储的，如果 rowkey 过长会极大影响 HFile 的存储效率 MemStore 将缓存部分数据到内存，如果 rowkey 字段过长，内存的有效利用率就会降低，系统不能缓存更多的数据，这样会降低检索效率。

* rowkey 散列原则
    > 如果 rowkey 按照时间戳的方式递增，不要将时间放在二进制码的前面，建议将 rowkey 的高位作为散列字段，由程序随机生成，低位放时间字段，这样将提高数据均衡分布在每个 RegionServer，以实现负载均衡的几率。如果没有散列字段，首字段直接是时间信息，所有的数据都会集中在一个 RegionServer 上，这样在数据检索的时候负载会集中在个别的 RegionServer 上，造成热点问题，会降低查询效率。
* rowkey 唯一原则
    > 必须在设计上保证其唯一性，rowkey 是按照字典顺序排序存储的，因此， 设计 rowkey 的时候，要充分利用这个排序的特点，将经常读取的数据存储到一块，将最近可能会被访问的数据放到一块。




