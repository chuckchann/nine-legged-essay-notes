# kafaka之消费数据

## 消费方式

 我们知道消息队列一般有两种实现方式,(1)Push(推模式) (2)Pull(拉模式)，那么 Kafka Consumer 究竟采用哪种方式进行消费的呢？**其实 Kafka Consumer 采用的是主动拉取 Broker 数据进行消费的即 Pull 模式**。

- 推模式： Push 模式最大缺点就是 Broker 不清楚 Consumer 的消费速度，且推送速率是 Broker 进行控制的， 这样很容易造成消息堆积，如果 Consumer 中执行的任务操作是比较耗时的，那么 Consumer 就会处理的很慢， 严重情况可能会导致系统 Crash。
- 拉模式：如果选择 Pull 模式，这时 Consumer 可以根据自己的情况和状态来拉取数据, 也可以进行延迟处理。但是如果 Kafka Broker 没有消息，这时每次 Consumer 拉取的都是空数据, 可能会一直循环返回空数据。 针对这个问题，Consumer 在每次调用 Poll() 消费数据的时候，顺带一个 timeout 参数，当返回空数据的时候，会在 Long Polling 中进行阻塞，等待 timeout 再去消费，直到数据到达。 

## 消费者组

**为什么 Kafka 要设计 Consumer Group, 只有 Consumer 不可以吗？** 我们知道 Kafka 是一款高吞吐量，低延迟，高并发, 高可扩展性的消息队列产品， 那么如果某个 Topic 拥有数百万到数千万的数据量， 仅仅依靠 Consumer 进程消费， 消费速度可想而知， 所以需要一个扩展性较好的机制来保障消费进度， 这个时候 Consumer Group 应运而生， **Consumer Group 是 Kafka 提供的可扩展且具有容错性的消费者机制。**  **Kafka Consumer Group 特点如下:**

1. 每个Consumer Group拥有一个或者多个Consumer
2. 每个Consumer Group拥有一个公共且唯一的Group ID
3. Consumer Group 在消费 Topic 的时候，Topic 的每个 Partition 只能分配给组内的某个 Consumer，只要被任何 Consumer 消费一次, 那么这条数据就可以认为被当前 Consumer Group 消费成功

## 分区与消费者的数量关系

- 分区数量等于消费者数量：最理想，每个消费者都能负责一个分区。
- 分区数量小于消费者数量：多出的消费者数量会空跑，浪费了消费者，但任然能满足消费需求。
- 分区数量大于消费者数量：一个分区不能同时被两个消费者消费，kafka在它设计的时候就是要保证分区下的消息顺序，那么要做到这一点就首先要保证消息是由消费者主动拉取的（pull），其次还要保证一个分区只能由一个消费者负责。倘若，两个消费者负责同一个分区，那么就意味着两个消费者同时读取分区的消息，由于消费者自己可以控制读取消息的offset，就有可能C1才读到2，而C1读到1，C1还没处理完，C2已经读到3了，则会造成很多浪费，因为这就相当于多线程读取同一个消息，会造成消息处理的重复，且不能保证消息的顺序，这就跟主动推送（push）无异。

## 分区的分配策略

我们知道一个 Consumer Group 中有多个 Consumer，一个 Topic 也有多个 Partition，所以必然会涉及到 Partition 的分配问题: 确定哪个 Partition 由哪个 Consumer 来消费的问题。**Kafka 客户端提供了3 种分区分配策略：RangeAssignor、RoundRobinAssignor 和 StickyAssignor**

### *a. RangeAssignor*

**RangeAssignor 是 Kafka 默认的分区分配算法，它是按照 Topic 的维度进行分配的**，对于每个 Topic，首先对 Partition 按照分区ID进行排序，然后对订阅这个 Topic 的 Consumer Group 的 Consumer 再进行排序，之后尽量均衡的按照范围区段将分区分配给 Consumer。此时可能会造成先分配分区的 Consumer 进程的任务过重（分区数无法被消费者数量整除）。

### *b. **RoundRobinAssignor***

 **RoundRobinAssignor 的分区分配策略是将 Consumer Group 内订阅的所有 Topic 的 Partition 及所有 Consumer 进行排序后按照顺序尽量均衡的一个一个进行分配**。如果 Consumer Group 内，每个 Consumer 订阅都订阅了相同的Topic，那么分配结果是均衡的。如果订阅 Topic 是不同的，那么分配结果是不保证“尽量均衡”的，因为某些 Consumer 可能不参与一些 Topic 的分配。

### *c. **StickyAssignor***

**StickyAssignor 分区分配算法**是 Kafka Java 客户端提供的分配策略中最复杂的一种，可以通过 partition.assignment.strategy 参数去设置，从 0.11 版本开始引入，目的就是在执行新分配时，尽量在上一次分配结果上少做调整，其主要实现了以下2个目标：

  **1)、Topic Partition 的分配要尽量均衡。**

  **2)、当 Rebalance(重分配，后面会详细分析) 发生时，尽量与上一次分配结果保持一致。**

## 消费者组重新分配

对于 Consumer Group 来说，可能随时都会有 Consumer 加入或退出，那么 Consumer 列表的变化必定会引起 Partition 的重新分配。我们将这个分配过程叫做 **Consumer Rebalance，但是这个分配过程需要借助 Broker 端的 Coordinator 协调者组件，在 Coordinator 的帮助下完成整个消费者组的分区重分配**，**也是通过监听ZooKeeper 的 /admin/reassign_partitions 节点触发的。**

**Rebalance 的触发条件有三种:**

1. 当 Consumer Group 组成员数量发生变化(主动加入或者主动离组，故障下线等)
2. 当订阅主题数量发生变化
3. 当订阅主题的分区数发生变化

## 消费者位移提交机制

这里首先需要区分**位移**与**消费者位移**之间的区别：

- 位移：通常所说的位移是指 Topic Partition 在 Broker 端的存储偏移量
- 消费者位移：指某个 Consumer Group 在不同 Topic Partition 上面的消费偏移量（也可以理解为消费进度），**它记录了 Consumer 要消费的下一条消息的位移**。

 **Consumer 需要向 Kafka 上报自己的位移数据信息，我们将这个上报过程叫做提交位移（Committing Offsets）**。它是为了保证 Consumer的消费进度正常，当 Consumer 发生故障重启后， 可以直接从之前提交的 Offset 位置开始进行消费而不用重头再来一遍（Kafka 认为小于提交的 Offset 的消息都已经成功消费了），Kafka 设计了这个机制来保障消费进度。**我们知道 Consumer 可以同时去消费多个分区的数据，所以位移提交是按照分区的粒度进行上报的，也就是说** **Consumer 需要为分配给它的每个分区提交各自的位移数据。**

Kafka Consumer 提供了多种提交方式，**从用户角度来说：位移提交可以分为自动提交和手动提交，但从 Consumer 的角度来说，位移提交可以分为同步提交和异步提交**， 接下来我们就来聊聊自动提交和手动提交方式：

### *自动提交*

 **自动提交是指 Kafka Consumer 在后台默默地帮我们提交位移，用户不需要关心这个事情。**启用自动提交位移，在 初始化 KafkaConsumer 的时候，通过设置参数 **enable.auto.commit = true** (默认为true)，开启之后还需要另外一个参数进行配合即 **auto.commit.interval.ms**，这个参数表示 Kafka Consumer 每隔 X 秒自动提交一次位移，这个值默认是5秒。

自动提交看起来是挺美好的, 那么**自动提交会不会出现消费数据丢失的情况呢？**在设置了 **enable.auto.commit = true** 的时候，Kafka 会保证在开始调用 Poll() 方法时，提交上一批消息的位移，再处理下一批消息, 因此它能保证不出现消费丢失的情况。但**自动提交位移也有设计缺陷，那就是它可能会出现重复消费**。就是在自动提交间隔之间发生 Rebalance 的时候，此时 Offset 还未提交，待 Rebalance 完成后， 所有 Consumer 需要将发生 Rebalance 前的消息进行重新消费一次。

### *手动提交*

与自动提交相对应的就是手动提交了。开启手动提交位移的方法就是在初始化KafkaConsumer 的时候设置参数 **enable.auto.commit = false，**但是只设置为 false 还不行，它只是告诉 Kafka Consumer 不用自动提交位移了，你还需要在处理完消息之后调用相应的 Consumer API 手动进行提交位移，**对于手动提交位移，又分为同步提交和异步提交。**

- 同步提交： **KafkaConsumer#commitSync()，**该方法会提交由 KafkaConsumer#poll() 方法返回的最新位移值，它是一个同步操作，会一直阻塞等待直到位移被成功提交才返回，如果提交的过程中出现异常，该方法会将异常抛出。这里我们知道在调用 **commitSync() 方法的时机**是在处理完 Poll() 方法返回所有消息之后进行提交，如果过早的提交了位移就会出现消费数据丢失的情况。
- 异步提交： **KafkaConsumer#commitAsync()，**该方法是异步方式进行提交的，调用 commitAsync() 之后，它会立即返回，并不会阻塞，因此不会影响 Consumer 的 TPS。另外 Kafka 针对它提供了callback，方便我们来实现提交之后的逻辑，比如记录日志或异常处理等等。由于它是一个异步操作， 假如出现问题是不会进行重试的，这时候重试位移值可能已不是最新值，所以重试无意义。

**从上面分析可以得出 commitSync 和 commitAsync 都有自己的缺陷，**我们需要将 commitSync 和 commitAsync 组合使用才能到达最理想的效果，既不影响 Consumer TPS，又能利用 commitSync 的自动重试功能来避免一些瞬时错误（网络抖动，GC，Rebalance 问题)，**在生产环境中建议大家使用混合提交模式来提高 Consumer的健壮性**。
