# 	简介

![](image\kafka拓扑图.png)

1. **Producer** ：消息生产者，就是向kafka broker 发消息的客户端；
2. **Consumer** ：消息消费者，向kafka broker 取消息的客户端；
3. **Consumer Group** （CG）：消费者组，由多个consumer 组成。消费者组内每个消费者负
   责消费不同分区的数据，一个分区只能由一个组内消费者消费；消费者组之间互不影响。所
   有的消费者都属于某个消费者组，即消费者组是逻辑上的一个订阅者。
4. **Broker** ：一台kafka 服务器就是一个broker。一个集群由多个broker 组成。一个broker
   可以容纳多个topic。
5. **Topic** ：可以理解为一个队列，生产者和消费者面向的都是一个topic；
6. **Partition**：为了实现扩展性，一个非常大的topic 可以分布到多个broker（即服务器）上，
   一个topic 可以分为多个partition，每个partition 是一个有序的队列；
7. **Replica**：副本，为保证集群中的某个节点发生故障时，该节点上的partition 数据不丢失，且
   	kafka仍然能够继续工作 kafka提供了副本机制，一个 topic的每个分区都有若干个副本，
   一个 leader和若干个 follower。
8.  **leader** 每个分区多个副本的“主”，生产者发送数据的对象，以及消费者消费数据的对
   象都是 leader。
9.  **follower** 每个分区多个副本中的“从”，实时从 leader中同步数据，保持和 leader数据
   的同步。 leader发生故障时，某个 follower会成为新的 follower。

# Kafka 架构深入

### Kafka 工作流程及文件存储机制

![image-20210306200903654](.\image\kafka工作流程.jpeg)

Kafka 中消息是以 topic 进行分类的，生产者生产消息，消费者消费消息，都是面向 topic 的。 topic 是逻辑上的概念，而 partition 是物理上的概念，每个 partition 对应于一个 log 文 件，该 log 文件中存储的就是 producer 生产的数据。Producer 生产的数据会被不断追加到该 log 文件末端，且每条数据都有自己的 offset。消费者组中的每个消费者，都会实时记录自己 消费到了哪个 offset，以便出错恢复时，从上次的位置继续消费。

![image-20210306201205744](.\image\kafka文件存储机制.png)

由于生产者生产的消息会不断追加到 log 文件末尾，为防止 log 文件过大导致数据定位 效率低下，Kafka 采取了分片和索引机制，将每个 partition 分为多个 segment。每个 segment 对应三个文件——2个“.index”文件和“.log”文件。这些文件位于一个文件夹下，该文件夹的命名 规则为：topic 名称+分区序号。例如，first 这个 topic 有三个分区，则其对应的文件夹为 first0,first-1,first-2。

```dockerfile
00000000000000000000.index
00000000000000000000.log
00000000000000170410.index
00000000000000170410.log
00000000000000239430.index
00000000000000239430.log
```

index 和 log 文件以当前 segment 的第一条消息的 offset 命名。下图为 index 文件和 log 文件的结构示意图。

![image-20210306201647491](image\kafka-index文件和log文件详解.png)

“.index”文件存储大量的索引信息，“.log”文件存储大量的数据，索引文件中的元 数据指向对应数据文件中 message 的物理偏移地址。

# Kafka 生产者 

### 生产者客户端整体架构

![](image\Kafka生产者客户端整体架构.png)

​		整个生产者客户端主要有两个线程，主线程以及Sender线程。Producer在主线程中产生消息，然后通过拦截器，序列化器，分区器之后缓存到消息累加器RecordAccumulator中。Sender线程从RecordAccumulator中获取消息并发送到kafka中

​	RecordAccumulator主要用来缓存消息，这样发送的时候进行批量发送以便减少相应的网络传输。RecordAccumulator缓存的大小可以通过配置参数buffer.memory配置，默认是**32M**。如果创建消息的速度过快，超过sender发送给kafka服务器的速度，会导致缓存空间不足，这个时候sender线程可能会阻塞或者抛出异常，max.block.ms配置决定阻塞的最大时间。

​	RecordAccumulator中为每个分区维护了一个双端队列，队列中的内容是ProducerBatch，即Deque<ProduderBatch>,创建消息写入到尾部，发送消息从头部读取。ProducerBatch是消息发送的一个批次，里面包含了一个或多个ProducerRecord。

​		Sender从RecordAccumulator中获取到缓存的消息，会将消息进行分区，Deque<ProducerBatch>>
转换为<Node,List<ProruderBatch>>,Node表示的是kafka集群的broker节点，生产者客户端与具体broker节点建立的连接。也就是向具体的broker节点发送消息而不关心具体分区。

​		转换为<Node,List<ProruderBatch>>后，sender还会进一步封装转换成<Node,Request>形式，将请求发送给各个Node。
请求在发送给Kafka之前还会保存到InFlightRequests中，形式为： Map<NodeId,Dequeue<Request>>
主要作用是缓存了已经发出去但是还未收到响应的请求。InFlightRequests通过配置参数max.flight.requests.per.connection可以限制每个链接最多缓存数量，默认值为5，即每个链接最多只能缓存5个未响应的请求，超过该参数之后就不能继续像这个连接发送请求。

### 消息的发送

发送消息主要有3种模式：发后即忘，同步，异步

KafkaProducer中一般有两种异常：可重试的异常和不可重试的异常。对于可重试的异常会根据retries参数进行重试，retries默认为0

```java
NetworkException、LeaderNotAvailableException、UnknownTopicOrPartitionException、NotEnoughReplicasException、
NotCoordinatorException
```

### 重要的生产者参数

1. acks指定需要多少个副本收到这条消息之后生产者才会认为消息写入成功    `properties.put("acks", "-1");`
2. max.request.size 限制生产者能发送消息的最大值，默认是1MB
3. retries    retry.backoff.ms  重试次数与重试的间隔时间
4. connections.max.idle.ms 指定多久之后关闭闲置的连接，默认是9分钟

### 分区策略 

​	**所谓分区策略是决定生产者将消息发送到哪个分区的算法。**Kafka 为我们提供了默认的分区策略，同时它也支持你自定义分区策略

#### **轮询策略**

​		也称 Round-robin 策略，即顺序分配。比如一个主题下有 3 个分区，那么第一条消息被发送到分区 0，第二条被发送到分区 1，第三条被发送到分区 2，以此类推。当生产第 4 条消息时又会重新开始，即将其分配到分区 0，就像下面这张图展示的那样。

![](image\kafaka轮询策略.png)

​		这就是所谓的轮询策略。轮询策略是 Kafka Java 生产者 API 默认提供的分区策略。如果你未指定`partitioner.class`参数，那么你的生产者程序会按照轮询的方式在主题的所有分区间均匀地“码放”消息。

​		**轮询策略有非常优秀的负载均衡表现，它总是能保证消息最大限度地被平均分配到所有分区上，故默认情况下它是最合理的分区策略，也是我们最常用的分区策略之一。**

#### **随机策略**

也称 Randomness 策略。所谓随机就是我们随意地将消息放置到任意一个分区上，如下面这张图所示。

![](image\kafka随机策略.png)

如果要实现随机策略版的 partition 方法，很简单，只需要两行代码即可：

```
List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
return ThreadLocalRandom.current().nextInt(partitions.size());
```

​		先计算出该主题总的分区数，然后随机地返回一个小于它的正整数。

​		本质上看随机策略也是力求将数据均匀地打散到各个分区，但从实际表现来看，它要逊于轮询策略，所以**如果追求数据的均匀分布，还是使用轮询策略比较好**。事实上，随机策略是老版本生产者使用的分区策略，在新版本中已经改为轮询了。

#### **按消息键保序策略**

​		也称 Key-ordering 策略。有点尴尬的是，这个名词是我自己编的，Kafka 官网上并无这样的提法。

​		Kafka 允许为每条消息定义消息键，简称为 Key。这个 Key 的作用非常大，它可以是一个有着明确业务含义的字符串，比如客户代码、部门编号或是业务 ID 等；也可以用来表征消息元数据。特别是在 Kafka 不支持时间戳的年代，在一些场景中，工程师们都是直接将消息创建时间封装进 Key 里面的。一旦消息被定义了 Key，那么你就可以保证同一个 Key 的所有消息都进入到相同的分区里面，由于每个分区下的消息处理都是有顺序的，故这个策略被称为按消息键保序策略，如下图所示。

![](image\kafka按消息键保序策略.png)

#### 黏性分区

​		既没有 partition 值又没有 key 值的情况下， [kafka](https://so.csdn.net/so/search?q=kafka&spm=1001.2101.3001.7020)采用Sticky Partition(黏性分区器)，会随机选择一个分区，并尽可能一直使用该分区，待该分区的batch已满或者已完成，kafka再随机一个分区进行使用.(以前是一条条的轮询，现在是一批次的轮询)

### 分区的原因 

1. 方便在集群中扩展，每个 Partition 可以通过调整以适应它所在的机器，而一个 topic 又可以有多个 Partition 组成，因此整个集群就可以适应任意大小的数据了； 

2. 可以提高并发，因为可以以 Partition 为单位读写了。

   

   ### 分区的原则

   我们需要将 producer 发送的数据封装成一个 ProducerRecord 对象。

   1. 指明 partition 的情况下，直接将指明的值直接作为 partiton 值；
   2. 没有指明 partition 值但有 key 的情况下，将 key 的 hash 值与 topic 的 partition 数进行取余得到 partition 值；
   3. 既没有 partition 值又没有 key 值的情况下，第一次调用时随机生成一个整数（后 面每次调用在这个整数上自增），将这个值与 topic 可用的 partition 总数取余得到 partition 值，也就是常说的 round-robin 算法。

   ### 数据可靠性保证

   **为保证 producer 发送的数据，能可靠的发送到指定的 topic，topic 的每个 partition 收到 producer 发送的数据后，都需要向 producer 发送 ack（acknowledgement 确认收到），如果 producer 收到 ack，就会进行下一轮的发送，否则重新发送数据。**

   ![image-20210306202400438](image\kafka生产者消息发送.png)

#### 副本数据同步策略

| 方案                        | 优点                                           | 缺点                                            |
| --------------------------- | ---------------------------------------------- | ----------------------------------------------- |
| 半数以上完成同步，就发送ACK | 延迟低                                         | 选举Leader时，容忍n台节点的故障，需要2n+1个副本 |
| 全部完成同步，才发送ACK     | 选举Leader时，容忍n台节点的故障，需要n+1个副本 | 延迟高                                          |

**说明：** Kafka选择第二种方案，原因如下：

- 第一种方案：需要2n+1 个副本，而Kafka的每个分区都有大量的数据，造成大量的冗余
- 第二种方案：需要n+1个副本，，虽然网络延迟高点，但对于Kafka的影响较小　　

#### ISR 

​		采用第二种方案之后，设想以下情景：leader 收到数据，所有 follower 都开始同步数据， 但有一个 follower，因为某种故障，迟迟不能与 leader 进行同步，那 leader 就要一直等下去， 直到它完成同步，才能发送 ack。这个问题怎么解决呢？ Leader 维护了一个动态的 in-sync replica set (ISR)，意为和 leader 保持同步的 follower 集 合。当 ISR 中的 follower 完成数据的同步之后，leader 就会给 follower 发送 ack。如果follower长时间未向leader同步 数据 ，则该follower将被踢出ISR ，该时间 阈 值 由**replica.lag.time.max.ms** (默认是10000)参数设定。Leader 发生故障之后，就会从 ISR 中选举新的 leader。

ISR集合发生变化的两个条件

1. 上一次ISR集合发送变化超过5S
2. 上一次写入zookeeper的时间距离超过60S

#### ack 应答机制 

​		对于某些不太重要的数据，对数据的可靠性要求不是很高，能够容忍数据的少量丢失， 所以没必要等 ISR 中的 follower 全部接收成功。 所以 Kafka 为用户提供了三种可靠性级别，用户根据对可靠性和延迟的要求进行权衡， 选择以下的配置。 acks 参数配置： acks： 

- **0**：producer 不等待 broker 的 ack，这一操作提供了一个最低的延迟，broker 一接收到还 没有写入磁盘就已经返回，当 broker 故障时有可能丢失数据； 
- **1**：producer 等待 broker 的 ack，partition 的 leader 落盘成功后返回 ack，如果在 follower 同步成功之前 leader 故障，那么将会丢失数据；

![image-20210307102029613](image\kafka数据丢失.png)

- **-1**（all）：producer 等待 broker 的 ack，partition 的 leader 和 follower 全部落盘成功后才 返回 ack。但是如果在 follower 同步完成后，broker 发送 ack 之前，leader 发生故障，那么会 造成数据重复

![](image\kafaka ack生产者数据重复.png)



### 故障处理细节 

#### follower 故障 

​		follower 发生故障后会被临时踢出 ISR，待该 follower 恢复后，follower 会读取本地磁盘 记录的上次的 HW，并将 log 文件高于 HW 的部分截取掉，从 HW 开始向 leader 进行同步。 等该 follower 的 LEO 大于等于该 Partition 的 HW，即 follower 追上 leader 之后，就可以重 新加入 ISR 了。

#### leader 故障 

​		leader 发生故障之后，会从 ISR 中选出一个新的 leader，之后，为保证多个副本之间的数据一致性，其余的 follower 会先将各自的 log 文件高于 HW 的部分截掉，然后从新的 leader 同步数据。 

**注意：这只能保证副本之间的数据一致性，并不能保证数据不丢失或者不重复。**

### 消息传输级别

#### At Most Once 

至多一次。消息有可能丢失，但是绝不会重复传输

#### At Least Once

至少传输一次。消息绝不会丢失，但是有可能会重复传输

#### Exactly Once

恰好一次，每条消息肯定会被传输一次，且只传输一次。

​		将服务器的 ACK 级别设置为-1，可以保证 Producer 到 Server 之间不会丢失数据，即 At Least Once 语义。相对的，将服务器 ACK 级别设置为 0，可以保证生产者每条消息只会被 发送一次，即 At Most Once 语义。

​		At Least Once 可以保证数据不丢失，但是不能保证数据不重复；相对的，At Most Once 可以保证数据不重复，但是不能保证数据不丢失。但是，对于一些非常重要的信息，比如说 交易数据，下游数据消费者要求数据既不重复也不丢失，即 Exactly Once 语义。在 0.11 版 本以前的 Kafka，对此是无能为力的，只能保证数据不丢失，再在下游消费者对数据做全局 去重。对于多个下游应用的情况，每个都需要单独做全局去重，这就对性能造成了很大影响。 0.11 版本的 Kafka，引入了一项重大特性：幂等性。所谓的幂等性就是指 Producer 不论 向 Server 发送多少次重复数据，Server 端都只会持久化一条。幂等性结合 At Least Once 语 义，就构成了 Kafka 的 Exactly Once 语义。即：

​										**At Least Once + 幂等性 = Exactly Once**

​		要启用幂等性，只需要将 Producer 的参数中 **enable.idompotence** 设置为 true 即可。Kafka 的幂等性实现其实就是将原来下游需要做的去重放在了数据上游。开启幂等性的 Producer 在 初始化的时候会被分配一个 PID，发往同一 Partition 的消息会附带 Sequence Number。而 Broker 端会对做缓存，当具有相同主键的消息提交时，Broker 只 会持久化一条。

​		但是 PID 重启就会变化，同时不同的 Partition 也具有不同主键，所以幂等性无法保证跨 分区跨会话的 Exactly Once。

#### Kafka 幂等性

​		在说 Kafka 的事务之前，先要说一下 Kafka 中幂等（Idempotent）的实现。幂等和事务是 Kafka 0.11.0.0 版本引入的两个特性，以此来实现 EOS 语义。

Kafka 幂等性是 Producer 端的特性，为了实现生产端幂等性，Kafka 引入了 Producer ID（即PID）和 Sequence Number。

PID：每个新的 Producer 在初始化的时候会被分配一个唯一的 PID，这个PID 对用户完全是透明的。

Sequence Numbler：对于每个 PID，该 Producer 发送到每个 Partition 的数据都有对应的序列号，这些序列号是从0开始单调递增的。

Broker 端在缓存中保存了这 Sequence Numbler，对于接收的每条消息，如果其序号比 Broker 缓存中序号大于1则接受它，否则将其丢弃。这样就可以实现了消息重复提交了。幂等涉及的参数是 enable.idempotence，默认为 false，开启需要设置为 ture。

但是，这种只能保证单个 Producer 对于单会话单 Partition 的 Exactly Once 语义。不能保证同一个 Producer 一个 topic 不同的 Partition 幂等

### producer 的写入流程

1. producer 先从 zookeeper 的 "/brokers/.../state" 节点找到该 partition 的 leader 
2. producer 将消息发送给该 leader 
3. leader 将消息写入本地 log 
4. followers 从 leader pull 消息，写入本地 log 后 leader 发送 ACK 
5. leader 收到所有 ISR 中的 replica 的 ACK 后，增加 HW（high watermark，最后 commit 的 offset） 并向 producer 发送 ACK

# Kafka 消费者

**每个分区只能被一个消费者组中的一个消费者消费到**

### 消费方式

​		consumer 采用 **pull**（拉）模式从 broker 中读取数据。

​		push（推）模式很难适应消费速率不同的消费者，因为消息发送速率是由 broker 决定的。 它的目标是尽可能以最快速度传递消息，但是这样很容易造成 consumer 来不及处理消息，典型的表现就是拒绝服务以及网络拥塞。而 pull 模式则可以根据 consumer 的消费能力以适 当的速率消费消息。

​		**pull 模式不足之处是，如果 kafka 没有数据，消费者可能会陷入循环中**，一直返回空数 据。针对这一点，Kafka 的消费者在消费数据时会传入一个时长参数 **timeout**，如果当前没有 数据可供消费，consumer 会等待一段时间之后再返回，这段时长即为 **timeout**。

### 分区分配策略

​		一个 consumer group 中有多个 consumer，一个 topic 有多个 partition，所以必然会涉及 到 partition 的分配问题，即确定那个 partition 由哪个 consumer 来消费。在Kafka中会由coordinator进行协调进行管理

- **RoundRobin**   按照消费者总数与分区总数进行整除运算来获得一个跨度，然后根据跨度进行分配，尽可能均匀的分配给所有消费者若出现不够平均分配，则字典序靠前的会多分配一个去

  缺点：有可能出现部分消费者过载

  ```
  消费者c0: t0p0,t0p1,t1p0,t1p1
  消费者c1: t0p2,t1p2
  ```

- **Range**  将消费者按照分区进行排序，之后通过轮询的方式依次逐个进行分配

  缺点：同一分区消息被不同消费者消费，没有分区有序

- **StickAssignor**   保留上一次分区分配结果   (最优解)

### consumer的位移提交（enable.auto.commit）

1. 自动提交：消费者每隔5秒会将每隔分区钟拉取到的最大的消息位移进行提交。自动提交的动作是在poll的逻辑里面完成的，每次发起拉取请求之前都是先检查一遍是否可以进行位移提交，如果可以，就会提交上一次轮询的位移。
2. 手动提交：
   1. commotSync()，其中无参方法是提交本次拉取的最新位移进行提交的
   2. commitAsync()，有3个重载函数。

### 关于consumer的offset的维护

​	旧版的是维护在ZK当中的，新版的是维护在**_consumer_offset**的内部主题中进行持久化。消费者在消费完消息之后，需要执行消费者位移的提交。由于 consumer 在消费过程中可能会出现断电宕机等故障，consumer 恢复后，需要从故 障前的位置的继续消费，所以 consumer 需要实时记录自己消费到了哪个 offset，以便故障恢 复后继续消费。consumer在维护offset的过程中，是根据**groupId跟partition**来维护offset的，即这样可以一定程度的避免rebalance过程中，避免同一消息被重复消费。

lastComsumedOffset: 当前消费到的位置

position:下一次拉取消息的位置

committedOffset:已经提交的消费位移

### 必要的参数设置

1. bootstrap.servers:kafa集群所需的brocker地址清单
2. group.id：消费者组的ID，默认为""。如果设置为空，则会报错

# KAFKA 客户端(broker)

## Zookeeper 在 Kafka 中的作用

​		Kafka 集群中有一个 broker 会被选举为 Controller，负责管理集群 broker 的上下线，所 有 topic 的分区副本分配和 leader 选举等工作。

​	Controller 的管理工作都是依赖于 Zookeeper 的。

​		以下为 partition 的 leader 选举过程：

![image-20210307110326373](image\kafka分区leader选举.png)

## Kafka的分区有序性

1. kafka的顺序消息仅仅是通过partitionKey，将某类消息写入同一个partition，同一个partition是在同一个broker上面的

2. 在kafka的server端，在selector.select 的mute  与unmute  来控制客户端与server的io的读取，控制secket的chanel的read事件

   ```scala
   processCompletedReceives()   -> selector.mute(connectionId); //读取事件后mute
   
   processCompletedSends（）    -> tryUnmuteChannel(send.destinationId)  //处理完send之后,unmute
   ```

3. 除了发送消息需要指定partitionKey外，producer和consumer实例化无区别。

4. kafka broker宕机，kafka会有自选择，所以宕机不会减少partition数量，也就不会影响partitionKey的sharding

## Partition中的HW和LEO

- LEO(Log End Offest)：指的是当前副本最大的 **offset + 1**； 

- HW(High Watermark)：指的是消费者能见到的最大的 **offset + 1**，ISR 队列中最小的 LEO。

  其中follower是通过自身循环线程，去向leader拉取消息的，在拉取的过程中，会把本follower的LEO带给leader，从而让leader更新HW。因为拉取是一直在的，这样处理可以减少响应的交互次数。其中如果leader中的LEO跟follwer的LEO相等的话,会有一个delay

整个消息追加的过程可以概括为

1. 生产者客户端发送消息至leader副本
2. 消息被追加到leader副本的本地日志中，并且更新日志的偏移量
3. follwer副本向leader请求副本同步数据，并且带上自己的HW
4. leader副本所在服务器读取本地日志，并更新对应拉取follwer的副本信息
5. leader副本将所在服务器将拉取结果返回follwer副本
6. follwer副本收到返回结果，将消息追加到本地日志，并更新偏移量

## Kfaka的Segment的形成因素(LogSegment)

1. segment文件创建时间是否大于配置的日志清除时间
2. 当前消息 + segment已有大小 > 配置文件segment文件大小
3. offsetIndex.isFull    10MB
4. timeIndex.isFull      10MB
5. !canConvertToRelativeOffset(rollParams.maxOffsetInMessages)    

## Kafka选举机制

### 控制器（Controller）选举

> ​		当broker启动的时候，都会创建KafkaController对象，但是集群中只能有一个leader对外提供服务，这些每个节点上的KafkaController会在指定的zookeeper路径下创建临时节点，只有第一个成功创建的节点的KafkaController才可以成为leader，其余的都是follower。当leader故障后，所有的follower会收到通知，再次竞争在该路径下创建节点从而选举新的leader
>
> ​	如果有一个broker加入集群中，那么控制器就会通过Broker ID去判断新加入的broker中是否含有现有分区的副本，如果有，就会从分区副本中去同步数据。
>
> ​		所谓控制器就是一个Borker，在一个kafka集群中，有多个broker节点，但是它们之间需要选举出一个leader，其他的broker充当follower角色。broker启动的时候会先去获取controller节点信息,如果不存在,会通过在zookeeper中创建临时节点/controller来让自己成为控制器，第一个创建成功的会成为controller。其他broker启动时也会在zookeeper中创建临时节点，但是发现节点已经存在，所以它们会收到一个异常，意识到控制器已经存在，那么就会在zookeeper中创建watch对象，便于它们收到控制器变更的通知。

### Controller职责

- 监听partition相关的变化。为Zookeeper中的/admin/reassign_partitions节点注册PartitionReassignmentListener，用来处理分区重分配的动作。为Zookeeper中的/isr_change_notification节点注册IsrChangeNotificetionListener，用来处理ISR集合变更的动作。为Zookeeper中的/admin/preferred-replica-election节点添加PreferredReplicaElectionListener，用来处理优先副本的选举动作。
- 监听topic相关的变化。为Zookeeper中的/brokers/topics节点添加TopicChangeListener，用来处理topic增减的变化；为Zookeeper中的/admin/delete_topics节点添加TopicDeletionListener，用来处理删除topic的动作。
- 监听broker相关的变化。为Zookeeper中的/brokers/ids/节点添加BrokerChangeListener，用来处理broker增减的变化。
- 从Zookeeper中读取获取当前所有与topic、partition以及broker有关的信息并进行相应的管理。对于所有topic所对应的Zookeeper中的/brokers/topics/[topic]节点添加PartitionModificationsListener，用来监听topic中的分区分配变化。
- 启动并管理分区状态机和副本状态机。
- 更新集群的元数据信息。
- 如果参数auto.leader.rebalance.enable设置为true，则还会开启一个名为“auto-leader-rebalance-task”的定时任务来负责维护分区的优先副本的均衡。

### 分区副本（partition）

 由controller leader执行

- 从Zookeeper中读取当前分区的所有ISR(in-sync replicas)集合
- 调用配置的分区选择算法选择分区的leader

> 如何处理所有Replica都不工作？
>
> 在ISR中至少有一个follower时，Kafka可以确保已经commit的数据不丢失，但如果某个Partition的所有Replica都宕机了，就无法保证数据不丢失了。这种情况下有两种可行的方案：
>
> 1. 等待ISR中的任一个Replica“活”过来，并且选它作为Leader
> 2. 选择第一个“活”过来的Replica（不一定是ISR中的）作为Leader
>
>   这就需要在可用性和一致性当中作出一个简单的折衷。如果一定要等待ISR中的Replica“活”过来，那不可用的时间就可能会相对较长。而且如果ISR中的所有Replica都无法“活”过来了，或者数据都丢失了，这个Partition将永远不可用。选择第一个“活”过来的Replica作为Leader，而这个Replica不是ISR中的Replica，那即使它并不保证已经包含了所有已commit的消息，它也会成为Leader而作为consumer的数据源（前文有说明，所有读写都由Leader完成）。Kafka0.8.*使用了第二种方式。根据Kafka的文档，在以后的版本中，Kafka支持用户通过配置选择这两种方式中的一种，从而根据不同的使用场景选择高可用性还是强一致性。 unclean.leader.election.enable 参数决定使用哪种方案，默认是true，采用第二种方案

### 消费组选主

在kafka的消费端，会有一个消费者协调器以及消费组，组协调器GroupCoordinator需要为消费组内的消费者选举出一个消费组的leader，那么如何选举的呢？

如果消费组内还没有leader，那么第一个加入消费组的消费者即为消费组的leader，如果某一个时刻leader消费者由于某些原因退出了消费组，那么就会重新选举leader，member是一个hashmap的数据结构，key为消费者的`member_id`，value是元数据信息，那么它会将leaderId选举为Hashmap中的第一个键值对，它和随机基本没啥区别

## Kafka 高效读写数据

- **顺序写磁盘**

  ​	Kafka 的 producer 生产数据，要写入到 log 文件中，写的过程是一直追加到文件末端， 为顺序写。官网有数据表明，同样的磁盘，顺序写能到 600M/s，而随机写只有 100K/s。这 与磁盘的机械机构有关，顺序写之所以快，是因为其省去了大量磁头寻址的时间。

- **零复制技术 Sendfile**

  sendfile：直接从内核空间缓冲区中将数据读取到网卡

------

## Kafka为什么不支持读写分离

### 读写分离的缺点	

- 数据一致性问题。数据从主节点到从节点，必然会存在一定的时间延时的时间窗口，这个时间窗口会导致主从节点之间的数据不一致的问题
- 延时问题。

读写分离主要是为了均摊负载，而Kafka本身的分区的设计就能很好的进行负载均衡，leader可以很好的分配到不同的broker中。

## Kafka 事务

​		Kafka 从 0.11 版本开始引入了事务支持。事务可以保证 Kafka 在 Exactly Once 语义的基 础上，生产和消费可以跨分区和会话，要么全部成功，要么全部失败。

### Producer 事务

​		为了实现跨分区跨会话的事务，需要引入一个全局唯一的 Transaction ID，并将 Producer 获得的PID 和Transaction ID 绑定。这样当Producer 重启后就可以通过正在进行的 Transaction ID 获得原来的 PID。Transaction ID与PID一一对应，不同的是，Transaction ID是由用户显式指定，而PID是由KAFKA内部进行分配的。

​		为了管理 Transaction，Kafka 引入了一个新的组件 Transaction Coordinator。Producer 就 是通过和 Transaction Coordinator 交互获得同一个Transaction ID 下的**epoch** 对应的任务状态。Transaction Coordinator 还负责将事务所有写入 Kafka 的一个内部 Topic，这样即使整个服务重启，由于 事务状态得到保存，进行中的事务状态可以得到恢复，从而继续进行。

### Consumer 事务

​		上述事务机制主要是从 Producer 方面考虑，对于 Consumer 而言，事务的保证就会相对 较弱，尤其时无法保证 Commit 的信息被精确消费。这是由于 Consumer 可以通过 offset 访 问任意信息，而且不同的 Segment File 生命周期不同，同一事务的消息可能会出现重启后被 删除的情况。

### 压缩

我们上面已经知道了Kafka支持以集合（batch）为单位发送消息，在此基础上，Kafka还支持对消息集合进行压缩，Producer端可以通过 GZIP或Snappy格式对消息集合进行压缩。Producer端进行压缩之后，在Consumer端需进行解压。压缩的好处就是减少传输的数据量，减 轻对网络传输的压力，在对大数据处理上，瓶颈往往体现在网络上而不是CPU（压缩和解压会耗掉部分CPU资源）。
那么如何区分消息是压缩的还是未压缩的呢，Kafka在消息头部添加了一个描述压缩属性字节，这个字节的后两位表示消息的压缩采用的编码，如果后两位为0，则表示消息未被压缩。

## 消息可靠性

在消息系统中，保证消息在生产和消费过程中的可靠性是十分重要的，在实际消息传递过程中，可能会出现如下三中情况：

- 一个消息发送失败
- 一个消息被发送多次
- 最理想的情况：exactly-once ,一个消息发送成功且仅发送了一次

有 许多系统声称它们实现了exactly-once，但是它们其实忽略了生产者或消费者在生产和消费过程中有可能失败的情况。比如虽然一个Producer 成功发送一个消息，但是消息在发送途中丢失，或者成功发送到broker，也被consumer成功取走，但是这个consumer在处理取过来的消息时 失败了。
从Producer端看：Kafka是这么处理的，当一个消息被发送后，Producer会等待broker成功接收到消息的反馈（可 通过参数控制等待时间），如果消息在途中丢失或是其中一个broker挂掉，Producer会重新发送（我们知道Kafka有备份机制，可以通过参数控 制是否等待所有备份节点都收到消息）。
从Consumer端看：前面讲到过partition，broker端记录了partition中的一 个offset值，这个值指向Consumer下一个即将消费message。当Consumer收到了消息，但却在处理过程中挂掉，此时 Consumer可以通过这个offset值重新找到上一个消息再进行处理。Consumer还有权限控制这个offset值，对持久化到broker端 的消息做任意处理。

## 备份机制

备 份机制是Kafka0.8版本的新特性，备份机制的出现大大提高了Kafka集群的可靠性、稳定性。有了备份机制后，Kafka允许集群中的节点挂掉后而 不影响整个集群工作。一个备份数量为n的集群允许n-1个节点失败。在所有备份节点中，有一个节点作为lead节点，这个节点保存了其它备份节点列表，并 维持各个备份间的状体同步。下面这幅图解释了Kafka的备份机制:

![](image\kafka备份机制.jpg)

## Kafka高效性相关设计

### 消息的持久化

​		Kafka高度依赖文件系统来存储和缓存消息，一般的人认为磁盘是缓慢的，这导致人们对持久化结构具有竞争性持怀疑态度。其实，磁盘远比你想象的要快或者慢，这决定于我们如何使用磁盘。
一 个和磁盘性能有关的关键事实是：磁盘驱动器的吞吐量跟寻到延迟是相背离的，也就是所，线性写的速度远远大于随机写。比如：在一个6 7200rpm SATA RAID-5 的磁盘阵列上线性写的速度大概是600M/秒，但是随机写的速度只有100K/秒，两者相差将近6000倍。线性读写在大多数应用场景下是可以预测的，因 此，操作系统利用read-ahead和write-behind技术来从大的数据块中预取数据，或者将多个逻辑上的写操作组合成一个大写物理写操作中。 更多的讨论可以在[ACMQueueArtical](https://link.zhihu.com/?target=http%3A//queue.acm.org/detail.cfm%3Fid%3D1563874)中找到，他们发现，对磁盘的线性读在有些情况下可以比内存的随机访问要快一些。
为了补偿这个性能上的分歧，现代操作系统都会把空闲的内存用作磁盘缓存，尽管在内存回收的时候会有一点性能上的代价。所有的磁盘读写操作会在这个统一的缓存上进行。
此外，如果我们是在JVM的基础上构建的，熟悉java内存应用管理的人应该清楚以下两件事情：

1. 一个对象的内存消耗是非常高的，经常是所存数据的两倍或者更多。
2. 随着堆内数据的增多，Java的垃圾回收会变得非常昂贵。

基于这些事实，利用文件系统并且依靠页缓存比维护一个内存缓存或者其他结构要好——我们至少要使得可用的缓存加倍，通过自动访问可用内存，并且通过存储更紧 凑的字节结构而不是一个对象，这将有可能再次加倍。这么做的结果就是在一台32GB的机器上，如果不考虑GC惩罚，将最多有28-30GB的缓存。此外， 这些缓存将会一直存在即使服务重启，然而进程内缓存需要在内存中重构（10GB缓存需要花费10分钟）或者它需要一个完全冷缓存启动（非常差的初始化性 能）。它同时也简化了代码，因为现在所有的维护缓存和文件系统之间内聚的逻辑都在操作系统内部了，这使得这样做比one-off in-process attempts更加高效与准确。如果你的磁盘应用更加倾向于顺序读取，那么read-ahead在每次磁盘读取中实际上获取到这人缓存中的有用数据。
以 上这些建议了一个简单的设计：不同于维护尽可能多的内存缓存并且在需要的时候刷新到文件系统中，我们换一种思路。所有的数据不需要调用刷新程序，而是立刻 将它写到一个持久化的日志中。事实上，这仅仅意味着，数据将被传输到内核页缓存中并稍后被刷新。我们可以增加一个配置项以让系统的用户来控制数据在什么时 候被刷新到物理硬盘上。

### 常数时间性能保证

​		消息系统中持久化数据结构的设计通常是维护者一个和消费队列有关的B树或者其它能够随机存取结构的元数据信息。B树是一个很好的结构，可以用在事务型与非事 务型的语义中。但是它需要一个很高的花费，尽管B树的操作需要O(logN)。通常情况下，这被认为与常数时间等价，但这对磁盘操作来说是不对的。磁盘寻 道一次需要10ms，并且一次只能寻一个，因此并行化是受限的。
直觉上来讲，一个持久化的队列可以构建在对一个文件的读和追加上，就像一般情况 下的日志解决方案。尽管和B树相比，这种结构不能支持丰富的语义，但是它有一个优点，所有的操作都是常数时间，并且读写之间不会相互阻塞。这种设计具有极 大的性能优势：最终系统性能和数据大小完全无关，服务器可以充分利用廉价的硬盘来提供高效的消息服务。
事实上还有一点，磁盘空间的无限增大而不影响性能这点，意味着我们可以提供一般消息系统无法提供的特性。比如说，消息被消费后不是立马被删除，我们可以将这些消息保留一段相对比较长的时间（比如一个星期）。

#### 进一步提高效率

​	我们已经为效率做了非常多的努力。但是有一种非常主要的应用场景是：处理Web活动数据，它的特点是数据量非常大，每一次的网页浏览都会产生大量的写操作。[更进一步](https://link.zhihu.com/?target=https%3A//www.baidu.com/s%3Fwd%3D%E6%9B%B4%E8%BF%9B%E4%B8%80%E6%AD%A5%26tn%3D24004469_oem_dg%26rsv_dl%3Dgh_pl_sl_csd)，我们假设每一个被发布的消息都会被至少一个consumer消费，因此我们更要怒路让消费变得更廉价。
通过上面的介绍，我们已经解决了磁盘方面的效率问题，除此之外，在此类系统中还有两类比较低效的场景：

- 太多小的I/O操作
- 过多的字节拷贝

为 了减少大量小I/O操作的问题，kafka的协议是围绕消息集合构建的。Producer一次网络请求可以发送一个消息集合，而不是每一次只发一条消息。 在server端是以消息块的形式追加消息到log中的，consumer在查询的时候也是一次查询大量的线性数据块。消息集合即MessageSet， 实现本身是一个非常简单的API，它将一个字节数组或者文件进行打包。所以对消息的处理，这里没有分开的序列化和反序列化的上步骤，消息的字段可以按需反 序列化（如果没有需要，可以不用反序列化）。
另一个影响效率的问题就是字节拷贝。为了解决字节拷贝的问题，kafka设计了一种“标准字节消 息”，Producer、Broker、Consumer共享这一种消息格式。Kakfa的message log在broker端就是一些目录文件，这些日志文件都是MessageSet按照这种“标准字节消息”格式写入到磁盘的。
维持这种通用的格式对这些操作的优化尤为重要：持久化log 块的网络传输。流行的[unix操作系统](https://link.zhihu.com/?target=https%3A//www.baidu.com/s%3Fwd%3Dunix%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%26tn%3D24004469_oem_dg%26rsv_dl%3Dgh_pl_sl_csd)提供了一种非常高效的途径来实现页面缓存和socket之间的数据传递。在[Linux操作系统](https://link.zhihu.com/?target=https%3A//www.baidu.com/s%3Fwd%3DLinux%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%26tn%3D24004469_oem_dg%26rsv_dl%3Dgh_pl_sl_csd)中，这种方式被称作：sendfile system call（Java提供了访问这个系统调用的方法：FileChannel.transferTo api）。

为了理解sendfile的影响，需要理解一般的将数据从文件传到socket的路径：

1. 操作系统将数据从磁盘读到内核空间的页缓存中
2. 应用将数据从内核空间读到用户空间的缓存中
3. 应用将数据写回内核空间的socket缓存中
4. 操作系统将数据从socket缓存写到网卡缓存中，以便将数据经网络发出

这种操作方式明显是非常低效的，这里有四次拷贝，两次系统调用。如果使用sendfile，就可以避免两次拷贝：操作系统将数据直接从页缓存发送到网络上。所以在这个优化的路径中，只有最后一步将数据拷贝到网卡缓存中是需要的。
我 们期望一个主题上有多个消费者是一种常见的应用场景。利用上述的zero-copy，数据只被拷贝到页缓存一次，然后就可以在每次消费时被重得利用，而不 需要将数据存在内存中，然后在每次读的时候拷贝到内核空间中。这使得消息消费速度可以达到网络连接的速度。这样以来，通过页面缓存和sendfile的结 合使用，整个kafka集群几乎都已以缓存的方式提供服务，而且即使下游的consumer很多，也不会对整个集群服务造成压力。







# 面试题

1、分区消费模型和组消费模型对比：

分区消费模型更加灵活但是：
（1）需要自己处理各种异常情况；
（2）需要自己管理offset（以实现消息传递的其他语义）；
组消费模型更加简单，但是不灵活：
（1）不需要自己处理异常情况，不需要自己管理offset；
（2）只能实现kafka默认的最少一次消息传递语义；

两种生产模型对比：
1、同步生产模型：
（1）低消息丢失率；
（2）高消息重复率（由于网络原因，回复确认未收到）；
（3）高延迟；
2、异步生产模型：
（1）低延迟；
（2）高发送性能；
（3）高消息丢失率（无确认机制，发送端队列满），1打包发送的时候无网络，会丢掉全部队列消息，2发送端队列满的时候无网络，会丢掉发送到队列的消息。

模型选择：
（1）一条消息都不能丢的，低吞吐量，选择同步生产模型；
（2）可以容忍丢消息（日志），高吞吐量，选择异步生产模型；

2、Kafka 中的 ISR(InSyncRepli)、OSR(OutSyncRepli)、AR(AllRepli)代表什么？

ISR：同步副本列表
AR：所有副本列表
OSR：与Leader副本滞后过多的副本
AR=ISR+OSR

3、Kafka 中的 HW、LEO 等分别代表什么？

LEO（Log end offset）：每个分区最后一条消息的偏移量，分区的每个副本都维护着自己的LEO。
HW（Hight Watermark）：每个分区对应的ISR中最小的LEO作为HW，HW之前的消息表示已提交的消息，对消费者是可见的。HW之后的消息表示还没被 Follower 副本同步完成。

4、Kafka 中是怎么体现消息顺序性的？
每个分区内，每条消息都有一个offset，所以只能保证一个分区内消息有序。

5、Kafka 中的分区器、序列化器、拦截器是否了解？它们之间的处理顺序是什么？
分区器、序列化器、拦截器、消息累加器详解

拦截器->序列化器->分区器

6、Kafka生产者客户端中使用了几个线程来处理？分别是什么？
2个线程，主线程和sender线程。主线程负责创建消息，然后通过拦截器、序列化器、分区器缓存到消息累加器 RecordAccumulator 中。sender线程负责将 RecordAccumulator 中的消息发送到kafka 中。

7、“消费组中的消费者个数如果超过 topic 的分区，那么就会有消费者消费不到数据”这句话是否正确？
正确

8、消费者提交消费位移时提交的是当前消费到的最新消息的 offset 还是 offset+1？
offset +1

9、有哪些情形会造成重复消费？
消费者消费了消息没有来得及提交偏移量（宕机以及其他问题）

10、那些情景会造成消息漏消费？
消费者还没有处理完消息就自动提交了offset（异常结束）

11、简述消费者与消费组之间的关系
消费者从属于消费者组，消费偏移以消费者组为单位。每个消费者组可以独立消费主题的所有数据，一条消息只能被同一个消费组的某一个消费者消费。

12、当你使用kafka-topics.sh创建（删除）了一个topic之后，Kafka背后会执行什么逻辑？
创建：在ZK上/brokers/topics/下节点 kafkabroker会监听节点变化创建主题。
删除：调用脚本删除topic会在ZK上将topic设置待删除标志，Kafka后台有定时的线程会扫描所有需要删除的topic进行删除。

13、topic的分区数可不可以增加？如果可以怎么增加？如果不可以，那又是为什么？
可以

14、topic的分区数可不可以减少？如果可以怎么减少？如果不可以，那又是为什么？
不可以，分区数据难以处理。

15、创建topic时如何选择合适的分区数？
根据集群的机器数量以及需要的吞吐量来决定分区数。

16、Kafka目前有那些内部topic，它们都有什么特征？各自的作用又是什么？
__consumer_offsets 以下划线开头，保存消费组的偏移。

17、优先副本是什么？它有什么特殊的作用？
优先副本 会是默认的leader副本 发生leader变化时重选举会优先选择优先副本作为leader

18、简述Kafka的日志目录结构
每个partition一个文件夹，包含四类文件.index .log .timeindex leader-epoch-checkpoint
.index .log .timeindex 三个文件成对出现 前缀为上一个segment的最后一个消息的偏移 log文件中保存了所有的消息 index文件中保存了稀疏的相对偏移的索引 timeindex保存的则是时间索引
leader-epoch-checkpoint中保存了每一任leader开始写入消息时的offset，会定时更新
follower被选为leader时会根据这个确定哪些消息可用

19、如果我指定了一个offset，Kafka怎么查找到对应的消息？
（1）通过文件名前缀数字x找到该绝对offset 对应消息所在文件
（2）offset-x为在文件中的相对偏移
（3）通过index文件中记录的索引找到最近的消息的位置
（4）从最近位置开始逐条寻找
