##### 1. Kafka中有哪几个组件?
* Topic：Kafka主题是一堆或一组消息。
* Producer：在Kafka，生产者发布通信以及向Kafka主题发布消息。
* Consumer：Kafka消费者订阅了一个主题，并且还从主题中读取和处理消息。
* Broker：一个Borker就是Kafka集群中的一个实例，或者说是一个服务单元
##### 2. offset的作用是什么？
每个partition都由一系列有序的、不可变的消息组成，这些消息被连续的追加到partition中。partition中的每个消息都有一个连续的序列号叫做offset,用于partition唯一标识一条消息。消费者通过offset来记录消费进度。
##### 3. 数据传输的事物定义有哪三种？
数据传输的事务定义通常有以下三种级别：
* At most once: 消息不会被重复发送，最多被传输一次，但也有可能一次不传输
* At least one: 消息不会被漏发送，最少被传输一次，但也有可能被重复传输.
* Exactly once: 不会漏传输也不会重复传输,每个消息都传输被一次而
且仅仅被传输一次，这是大家所期望的

##### 4. 什么是消费者组？
consumer group是kafka提供的可扩展且具有容错性的消费者机制。既然是一个组，那么组内必然可以有多个消费者或消费者实例(consumer instance)，它们共享一个公共的ID，即group ID。组内的所有消费者协调在一起来消费订阅主题(subscribed topics)的所有分区(partition)。当然，每个分区只能由同一个消费组内的一个consumer来消费。
* 一个组可以有多个消费者
* 主题中的消息只能被同一个组中的一个消费者消费
* 一个主题可以被多个消费者组消费

##### 5. 解释leader和follower的概念
Kafka每个topic的partition有N个副本，其中N是topic的复制因子。Kafka通过多副本机制实现故障自动转移，当Kafka集群中一个Broker失效情况下仍然保证服务可用。在Kafka中发生复制时确保partition的预写式日志有序地写到其他节点上。N个replicas中。其中一个replica为leader，其他都为follower，leader处理partition的所有读写请求，与此同时，follower会被动定期地去复制leader上的数据。

##### 6. Kafka的那些设计让它有如此高的性能？
* 利用Partition 实现并行处理
* 基于ISR 的数据复制方案
* 高效使用磁盘： 零拷贝，页缓存，顺序写
* 减少网络开销：批处理，数据压缩降低网络负载，高效的序列化方式

##### 7. kafka消息是怎样被删除的？
kafka的消息被保存为文件，文件被划分为1G大小的Segment，默认保留7天的数据，到时会被删除。正在写入的Segment被称为活跃段，如果消息过期时间设置为2天，但是消息所在的段是活跃段，那么在删除的时候是7天，而不是2天


##### 8. 为什么Kafka不支持读写分离？
Kafka不是做数据存储的，是消息队列。读写分离会产生数据一致性和延时的问题。
对于Kafka来说，必要性不是很高，因为在Kafka集群中，如果存在多个副本，经过合理的配置，可以让leader副本均匀的分布在各个broker上面，使每个 broker 上的读写负载都是一样的。

##### 9. 什么情况下，leader 会认为一条消息 commit了
kafka认为生产者写入消息完成的标志：
* acks=0：消息发送完毕，生产者认为消息写入成功；
* acks=1：主副本写入成功，生产者认为消息写入成功；
* acks=all：所有in-sync副本写入成功，生产者认为消息写入成功

当ISR中所有Replica都向Leader发送ACK时，leader才commit，这时候producer才能认为一个请求中的消息都commit了。


##### 10. 创建topic时如何选择合适的分区数？
根据集群的机器数量和需要的吞吐量来决定适合的分区数，创建一个只有1个分区的topic，然后测试这个topic的producer吞吐量和consumer吞吐量。假设它们的值分别是Tp和Tc，单位可以是MB/s。然后假设总的目标吞吐量是Tt，那么`分区数 =  Tt / max(Tp, Tc)`

##### 11. 有哪些情形会造成重复消费？
消费者消费后没有commit offset(程序崩溃/强行kill/消费耗时/自动提交偏移情况下unscrible)

##### 12. 那些情景下会造成消息漏消费？
消费者没有处理完消息 提交offset(自动提交偏移 未处理情况下程序异常结束)

##### 13. Kafka为什么为什么要分区呢？
为了性能考虑，如果不分区每个topic的消息只存在一个broker上，那么所有的消费者都是从这个broker上消费消息，那么单节点的broker成为性能的瓶颈，如果有分区的话生产者发过来的消息分别存储在各个broker不同的partition上，这样消费者可以并行的从不同的broker不同的partition上读消息，实现了水平扩展。


##### 14. Kafka中是怎么体现消息顺序性的？

Kafka分布式的单位是partition，同一个partition用一个write ahead log组织，所以可以保证FIFO的顺序。不同partition之间不能保证顺序。
但是绝大多数用户都可以通过message key来定义，因为同一个key的message可以保证只发送到同一个partition，比如说key是user id，table row id等等，所以同一个user或者同一个record的消息永远只会发送到同一个partition上，保证了同一个user或record的顺序。
