## Kafka介绍

## 1 基础架构

![](http://dist415.oss-cn-beijing.aliyuncs.com/kafkaarc.png)

- **Producer** ：消息生产者，就是向kafka broker发消息的客户端；
- **Consumer** ：消息消费者，向kafka broker取消息的客户端；
- **ConsumerGroup （CG）**：消费者组，由多个consumer组成。**消费者组内每个消费者负责消费不同分区的数据，一个分区只能由一个组内消费者消费；消费者组之间互不影响**。所有的消费者都属于某个消费者组，即消费者组是**逻辑上的一个订阅者**。
- **Broker** ：一台kafka服务器就是一个broker。一个集群由多个broker组成。一个broker 可以容纳多个topic。
- **Topic** ：可以理解为一个队列，**生产者和消费者面向的都是一个topic**；
- **Partition**：为了实现扩展性，一个非常大的topic可以分布到多个broker（即服务器）上，**一个topic可以分为多个partition，每个partition是一个有序的队列**；
- **Replica**：副本，为保证集群中的某个节点发生故障时，**该节点上的partition数据不丢失，且kafka仍然能够继续工作**，kafka提供了副本机制，一个topic的每个分区都有若干个副本，一个leader和若干个follower。
- **leader**：每个分区多个副本的“主”，生产者发送数据的对象，以及消费者消费数据的对象都是leader。
- **follower**：每个分区多个副本中的“从”，实时从leader中同步数据，保持和leader数据的同步。leader发生故障时，某个follower会成为新的follower。

## 2 Kafka工作流程

![](http://dist415.oss-cn-beijing.aliyuncs.com/kafkaworkflow.png)

Kafka中消息是以**topic**进行分类的，生产者生产消息，消费者消费消息，都是面向topic的

topic是逻辑上的概念，而partition是物理上的概念，每个partition对应于一个log文件，该log文件中存储的就是producer生产的数据。Producer生产的数据会被不断追加到该log文件末端，且每条数据都有自己的offset。**消费者组中的每个消费者，都会实时记录自己消费到了哪个offset**，以便出错恢复时，从上次的位置继续消费。

## 3 Kafka文件储存机制

![](http://dist415.oss-cn-beijing.aliyuncs.com/kafkafilearc.png)

由于生产者生产的消息会不断追加到 log 文件末尾，为防止 log 文件过大导致数据定位
效率低下，Kafka 采取了**分片和索引**机制，将每个 partition 分为多个 segment。每个 segment
对应两个文件——“.index”文件和“.log”文件。这些文件位于一个文件夹下，该文件夹的命名
规则为：**topic 名称+分区序号**。例如，first 这个 topic 有三个分区，则其对应的文件夹为 first-
0,first-1,first-2。

```ini
00000000000000000000.index
00000000000000000000.log
00000000000000170410.index
00000000000000170410.log
00000000000000239430.index
00000000000000239430.log
```

index 和 log 文件以当前 segment 的**第一条消息的 offset** 命名。下图为 index 文件和 log
文件的结构示意图

![](http://dist415.oss-cn-beijing.aliyuncs.com/kafkafileindex.png)

**“.index”文件存储大量的索引信息，“.log”文件存储大量的数据，索引文件中的元数据指向对应数据文件中message的物理偏移地址。**



## 4 Kafka生产者

### 4.1 分区策略

#### 4.1.1 分区原因

- **方便在集群中扩展**，每个Partition可以通过调整以适应它所在的机器，而一个topic 又可以有多个Partition组成，因此整个集群就可以适应任意大小的数据了；
- **可以提高并发**，因为可以以Partition为单位读写了

#### 4.1.2 分区原则

在生产端发送数据的时候,发送到的分区根据以下顺序选择:

1. 指明partition 的情况下，直接将指明的值直接作为partiton 值；
2. 没有指明partition 值但有key 的情况下，将key 的hash 值与topic 的partition 数进行取余得到partition 值；
3. 既没有partition 值又没有key 值的情况下，第一次调用时随机生成一个整数（后面每次调用在这个整数上自增），将这个值与topic 可用的partition 总数取余得到partition 值，也就是常说的round-robin 算法。

### 4.2 数据可靠性

**为保证producer发送的数据，能可靠的发送到指定的topic，topic的每个partition收到producer发送的数据后，都需要向producer发送ack（acknowledgement确认收到），如果producer收到ack，就会进行下一轮的发送，否则重新发送数据。**

#### 4.2.1 发送方案

![](http://dist415.oss-cn-beijing.aliyuncs.com/kafkaack.png)

Kafka 选择了**第二种**方案，原因如下：

- 同样为了容忍 n 台节点的故障，第一种方案需要 2n+1 个副本，而第二种方案只需要 n+1个副本，而 Kafka 的每个分区都有大量的数据，**第一种方案会造成大量数据的冗余**。

- 虽然第二种方案的网络延迟会比较高，但网络延迟对 Kafka 的影响较小。

#### 4.2.2 ISR

采用第二种方案之后，设想以下情景：leader 收到数据，所有 follower 都开始同步数据，
但有一个 follower，因为某种故障，迟迟不能与 leader 进行同步，那 leader 就要一直等下去，
直到它完成同步，才能发送 ack。这个问题怎么解决呢？

**Leader维护了一个动态的in-syncreplicaset(ISR)，意为和leader保持同步的follower集合。当ISR中的follower完成数据的同步之后，leader就会给follower发送ack。如果follower长时间未向leader同步数据，则该follower将被踢出ISR，该时间阈值由replica.lag.time.max.ms参数设定。Leader发生故障之后，就会从ISR中选举新的leader。**

#### 4.2.3 ACK应答机制

对于某些不太重要的数据，对数据的可靠性要求不是很高，能够容忍数据的少量丢失，所以没必要等ISR中的follower全部接收成功。所以**Kafka为用户提供了三种可靠性级别**，用户根据对可靠性和延迟的要求进行权衡，选择以下的配置。


- **0**：**producer不等待broker的ack**，这一操作提供了一个最低的延迟，broker一接收到还没有写入磁盘就已经返回，当broker故障时有可能丢失数据；
- **1**：**producer等待broker的ack，partition的leader落盘成功后返回ack**，如果在follower同步成功之前leader故障，那么将会丢失数据；
- **-1**（all）：**producer等待broker的ack，partition的leader和follower全部落盘成功后才返回ack**。但是如果在follower同步完成后，broker发送ack之前，leader发生故障，那么会造成**数据重复**

#### 4.2.4 HW和LEO

**LEO：指的是每个副本最大的 offset；**
**HW：指的是消费者能见到的最大的 offset，ISR 队列中最小的 LEO。**

![](http://dist415.oss-cn-beijing.aliyuncs.com/kafkahw.png)

- **follower故障**：follower发生故障后会被临时踢出ISR，待该follower恢复后，follower会读取本地磁盘记录的上次的HW，并将log文件高于HW的部分截取掉，从HW开始向leader进行同步。等该**follower的LEO大于等于该Partition的HW**，即follower追上leader之后，就可以重新加入ISR了。

- **leader故障**:leader发生故障之后，会从ISR中选出一个新的leader，之后，为保证多个副本之间的数据一致性，其余的follower会先将各自的log文件高于**HW的部分截掉**，然后从新的leader 同步数据。

  

**注意：这只能保证副本之间的数据一致性，并不能保证数据不丢失或者不重复。**



### 4.3 ExactlyOnce语义

将服务器的ACK级别设置为-1,可以保证Producer到Server之间不会丢失数据，即**At Least Once语义**。相对的，将服务器ACK级别设置为0,可以保证生产者每条消息只会被发送一次，即**AtMostOnce语义**。

**AtLeastOnce可以保证数据不丢失，但是不能保证数据不重复**

**AtLeastOnce 可以保证数据不重复，但是不能保证数据不丢失**

但是，对于一些非常重要的信息，比如说交易数据，下游数据消费者要求数据既不重复也不丢失，即ExactlyOnce语义。在0.11版本以前的Kafka，对此是无能为力的，只**能保证数据不丢失，再在下游消费者对数据做全局去重**。对于多个下游应用的情况，每个都需要单独做全局去重，这就对性能造成了很大影响。

0.11版本的Kafka，引入了一项重大特性：**幂等性**。**所谓的幂等性就是指Producer不论向Server发送多少次重复数据，Server端都只会持久化一条**。幂等性结合AtLeastOnce语义，就构成了Kafka的ExactlyOnce语义。即：

```
AtLeastOnce+幂等性=ExactlyOnce
```

要启用幂等性，只需要将**Producer的参数中enable.idompotence设置为true即可**。

**Kafka 的幂等性实现其实就是将原来下游需要做的去重放在了数据上游**。开启幂等性的Producer在初始化的时候会被分配一个PID，发往同一Partition的消息会附带SequenceNumber。**而Broker端会对<PID,Partition,SeqNumber>做缓存，当具有相同主键的消息提交时，Broker只会持久化一条**。

**但是PID重启就会变化，同时不同的Partition也具有不同主键，所以幂等性无法保证跨分区跨会话的ExactlyOnce。**



## 5 Kafka消费者

### 5.1 消费方式

**consumer采用pull（拉）模式从broker中读取数据**。

push（推）模式很难适应消费速率不同的消费者，因为消息发送速率是由broker决定的。它的目标是尽可能以最快速度传递消息，但是这样很容易造成consumer来不及处理消息，典型的表现就是拒绝服务以及网络拥塞。而pull模式则可以根据consumer的消费能力以适当的速率消费消息。

**pull模式不足之处是，如果kafka没有数据，消费者可能会陷入循环中，一直返回空数据**。针对这一点，Kafka的消费者在消费数据时会传入一个时长参数**timeout**，如果当前没有数据可供消费，consumer会等待一段时间之后再返回，这段时长即为timeout。

### 5.2 offset维护

由于consumer在消费过程中可能会出现断电宕机等故障，consumer恢复后，需要从故障前的位置的继续消费，所以**consumer需要实时记录自己消费到了哪个offset，以便故障恢复后继续消费**。

**Kafka0.9版本之前，consumer默认将offset保存在Zookeeper中，从0.9版本开始，consumer默认将offset保存在Kafka一个内置的topic中，该topic为__consumer_offsets**



## 6 Kafka高效读写数据原因

- 顺序写磁盘
- 零拷贝技术

### 6.1 顺序读写磁盘

Kafka的producer生产数据，要写入到log文件中，写的过程是一直追加到文件末端,为顺序写。官网有数据表明，同样的磁盘，顺序写能到600M/s，而随机写只有100K/s。这与磁盘的机械机构有关，顺序写之所以快，是因为其省去了大量磁头寻址的时间。

### 6.2 零拷贝技术 

![](http://dist415.oss-cn-beijing.aliyuncs.com/kafkazero.png)

## 7 Zookeeper在Kafka中的作用

**Kafka 集群中有一个 broker 会被选举为 Controller，负责管理集群 broker 的上下线，所有 topic 的分区副本分配和 leader 选举等工作。**
**Controller 的管理工作都是依赖于 Zookeeper 的。**

![](http://dist415.oss-cn-beijing.aliyuncs.com/kafkazk.png)



## 8 Kafka事务

Kafka 从 0.11 版本开始引入了事务支持。事务可以保证 Kafka 在 Exactly Once 语义的基
础上，**生产和消费可以跨分区和会话，要么全部成功，要么全部失败**

### 8.1 Producer事务

为了实现跨分区跨会话的事务，需要引入一个全局唯一的 Transaction ID，并将 Producer
获得的PID 和Transaction ID 绑定。这样当Producer 重启后就可以通过正在进行的 Transaction
ID 获得原来的 PID。
为了管理 Transaction，Kafka 引入了一个新的组件 **Transaction Coordinator**。Producer 就
是通过和 Transaction Coordinator 交互获得 Transaction ID 对应的任务状态。Transaction
Coordinator 还负责将事务所有写入 Kafka 的一个内部 Topic，这样即使整个服务重启，由于
事务状态得到保存，进行中的事务状态可以得到恢复，从而继续进行。

### 8.2 Consumer事务

上述事务机制主要是从 Producer 方面考虑，对于 Consumer 而言，事务的保证就会相对
较弱，尤其时无法保证 Commit 的信息被精确消费。**这是由于 Consumer 可以通过 offset 访**
**问任意信息，而且不同的 Segment File 生命周期不同，同一事务的消息可能会出现重启后被**
**删除的情况**












