# Kafka消息模型

- Kafka采用发布-订阅模型，将生产者发布的消息发送到 **Topic（主题）** 中，需要这些消息的消费者可以订阅这些 **Topic（主题）**，如下图所示：

![](https://blogpicture2022.oss-cn-hangzhou.aliyuncs.com/202206301643555.png)

其中：

- **Producer（生产者）** : 产生消息的一方。

- **Consumer（消费者）** : 消费消息的一方。
- **Consumer Group** ：消费者组，由**多个 Consumer 组成**。消费者**组内每个消费者负责消费不同分区的数据**，**一个分区只能由一个组内消费者消费**；消费者组之间互不影响。所有的消费者都属于某个消费者组，即消费者组是逻辑上的一个订阅者。

- **Broker（代理）** :可以看做一个独立的 **Kafka 服务节点或 Kafka 服务实例**。如果一台服务器上只部署了一个 Kafka 实例，那么我们也可以将 Broker 看做一台 Kafka 服务器。

同时，每个 Broker 中又包含以下几个重要的概念：

- **Topic（主题）** :一个逻辑上的概念，包含很多 Partition，**同一个 Topic 下的 Partiton 的消息内容是不相同的**
- **Partition（分区）** : 为了实现扩展性，一个非常大的 topic **可以分布到多个 broker 上，一个 topic 可以分为多个 partition**，每个 partition 是一个有序的队列。
- **Replica** ：副本，**同一分区的不同副本保存的是相同的消息**，为保证集群中的某个节点发生故障时，该节点上的 partition 数据不丢失，且 kafka 仍然能够继续工作，kafka 提供了副本机制，一个 topic 的每个分区都有若干个副本，一个 leader 和若干个 follower。
- **Leader** ：每个分区的多个副本中的"主副本"，**生产者以及消费者只与 Leader 交互**。
- **Follower** ：每个分区的多个副本中的"从副本"，**负责实时从 Leader 中同步数据，保持和 Leader 数据的同步**。Leader 发生故障时，从 Follower 副本中重新选举新的 Leader 副本对外提供服务。

# Replica副本管理

![](https://blogpicture2022.oss-cn-hangzhou.aliyuncs.com/202206301707764.png)

- AR:分区中的**所有 Replica 统称为 AR**
- ISR:所有与 Leader 副本**保持一定程度同步**的Replica(包括 Leader 副本在内)组成 ISR
- OSR:与 Leader 副本**同步滞后过多的** Replica 组成了 OSR

Leader 负责维护和跟踪 ISR 集合中所有 Follower 副本的滞后状态，当 Follower 副本落后过多时，就会将其放入 OSR 集合，当 Follower 副本追上了 Leader 的进度时，就会将其放入 ISR 集合。

默认情况下，**只有 ISR 中的副本才有资格晋升为 Leader**。

# 消息发送

## 发送模式（避免消息丢失在发送端）

- 发后即忘（fire-and-forget）：它只管往 Kafka 里面发送消息，但是**不关心消息是否正确到达**，这种方式的**效率最高**，但是**可靠性也最差**，比如当发生某些不可充实异常的时候会造成消息的丢失

- 同步（sync）：producer.send()返回一个Future对象，调用get()方法变回进行**同步等待**，就知道消息是否发送成功，**发送一条消息需要等上个消息发送成功后才可以继续发送**

- 异步（async）：Kafka支持 producer.send() 传入一个回调函数，消息**不管成功或者失败都会调用这个回调函数**，这样就算是异步发送，我们也知道消息的发送情况，然后再回调函数中**选择记录日志还是重试都取决于调用方**

## 发送消息的分区策略（消费顺序的保证）

![](https://blogpicture2022.oss-cn-hangzhou.aliyuncs.com/202206301845900.png)

- **轮询**：**依次**将消息发送该topic下的所有分区，如果在创建消息的时候 key 为 null，Kafka 默认采用这种策略。
- **key 指定分区（消费顺序有序的保证）**：在创建消息是 key 不为空，并且使用默认分区器，Kafka 会将 key 进行 hash，然后**根据hash值映射到指定的分区上**。这样的好处是 key 相同的消息会在一个分区下，Kafka 并不能保证全局有序，但是**在每个分区下的消息是有序的**，按照顺序存储，按照顺序消费。在保证同一个 key 的消息是有序的，这样基本能满足消息的顺序性的需求。但是**如果 partation 数量发生变化，那就很难保证 key 与分区之间的映射关系了**。

- 自定义策略：实现 Partitioner 接口就能自定义分区策略。
- 指定 Partiton 发送

# 消息接收

