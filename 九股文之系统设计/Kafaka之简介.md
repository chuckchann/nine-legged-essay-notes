# Kafaka简介

------

## 1. 简介

 Kafka 是一个分布式的基于发布/订阅模式的消息队列（Message Queue），主要应用与大数据实时处理领域。其主要设计目标如下：

1. 以时间复杂度为O(1)的方式提供消息持久化能力，即使对TB级以上数据也能保证常数时间的访问性能。
2. 高吞吐率。即使在非常廉价的机器上也能做到单机支持每秒100K条消息的传输。
3. 支持消息分区及分布式消费，同时保证每个partition内的消息顺序传输，同时支持离线数据处理和实时数据处理

## 2. 组件

1. Broker：一台 Kafka 机器就是一个 Broker。一个集群是由多个 Broker 组成的且一个 Broker 可以容纳多个 Topic。
2. Topic：可以简单理解为队列，Topic 将消息分类，生产者和消费者面向的都是同一个 Topic。
3. Producer：即消息生产者，向 Kafka Broker 发消息的客户端。
4. Consumer Group：即消费者组，消费者组内每个消费者负责**消费不同分区的数据，以提高消费能力**。**一个分区只能由组内一个消费者消费，不同消费者组之间互不影响**。
5. Partition：为了实现Topic扩展性，提高并发能力，一个非常大的 Topic 可以分布到多个 Broker 上，一个 Topic 可以分为多个 Partition 进行存储，每个 Partition 是一个有序的队列。
6. Replica：即副本，为实现数据备份的功能，保证集群中的某个节点发生故障时，该节点上的 Partition 数据不丢失，且 Kafka 仍然能够继续工作，为此Kafka提供了副本机制，一个 Topic 的每个 Partition 都有若干个副本，一个 Leader 副本和若干个 Follower 副本。我们的 **Producer 端在发送数据的时候，只能发送到Leader Partition里面 ，然后Follower Partition会去Leader那自行同步数据, Consumer 消费数据的时候，也只能从 Leader 副本那去消费数据的**。
7. Leader：即每个分区多个副本的主副本，**生产者发送数据的对象，以及消费者消费数据的对象，都是 Leader**。
8. Offset：消费者消费的位置信息，监控数据消费到什么位置，当消费者挂掉再重新恢复的时候，可以从消费位置继续消费。
9. ZooKeeper服务：Kafka 集群能够正常工作，需要依赖于 ZooKeeper，ZooKeeper 帮助 Kafka 存储和管理集群元数据信息。在最新版本中, 已经慢慢要脱离 ZooKeeper。
10. Controller: 其实就是一个 Kafka 集群中一台 Broker。它除了具有普通Broker 的消息发送、消费、同步功能之外，还需承担一些额外的工作。Kafka 使用公平竞选的方式来确定 Controller ，最先在 ZooKeeper 成功创建临时节点 /controller 的Broker会成为 Controller ，一般而言，Kafka集群中第一台启动的 Broker 会成为Controller，并将自身 Broker 编号等信息写入ZooKeeper临时节点/controller。

## 3. 集群架构

![640 (6)](image/640 (6).png)

## 3. kafaka之高可用

### a. *<u>选举机制</u>*

Kafka 中的选举大致分为三大类: 控制器的选举, Leader 的选举, 消费者的选举。

- 控制器的选举：Kafka 控制器其实是一个Broker。它除了具有一般 Broker 的功能外, 还具有选举分区Leader节点的功能, 在启动 Kafka 系统时候, 其中一个 Broker 会被选举为控制器, 负责管理主题分区和副本的状态, 还会执行重分配的任务。 选举控制器的核心思路是：各个节点公平竞争抢占 Zookeeper 系统中创建 /controller临时节点，最先创建成功的节点会成为控制器，
- Leader Partition的选举：上一步选出控制器后，控制器（Broker）所在的Partition即为Leader Partition（TODO）

### b. *<u>副本机制</u>*

 副本机制简单来说就是备份机制，就是在分布式集群中保存着相同的数据备份。那么副本机制的好处就是提供数据冗余,  副本机制是kafka确保系统高可用和高持久的重要基石。 为了保证高可用，kafka 的分区是多副本的，如果其中一个副本丢失了，那么还可以从其他副本中获取分区数据(要求对应副本的数据必须是完整的)。这是 Kafka 数据一致性的基础。

 Kafka 使用 Zookeeper 来维护集群 Brokers 的信息，每个 Broker 都有一个唯一的标识**`broker.id`**，用于标识自己在集群中的身份。Brokers 会通过 Zookeeper 选举出一个叫**`Controller Broker`**节点，它除了具备其它Brokers的功能外，还**负责管理主题分区及其副本的状态**。 Kafka 使用 Zookeeper 来维护集群 Brokers 的信息，每个 Broker 都有一个唯一的标识**`broker.id`**，用于标识自己在集群中的身份。Brokers 会通过 Zookeeper 选举出一个叫**`Controller Broker`**节点，它除了具备其它Brokers的功能外，还**负责管理主题分区及其副本的状态**。

 在 Kafka 中 Topic 被分为多个分区（Partition），分区是 Kafka ***最基本的存储单位***。在创建主题的时候可使用**`replication-factor`**参数指定分区的副本个数。分区副本总会有一个 Leader 副本，所有的消息都直接发送给Leader 副本，其它副本都需要通过复制 Leader 中的数据来保证数据一致。当 Leader 副本不可用时，其中一个 Follower 将会被选举并成为新的 Leader。

### c. *<u>ISR机制</u>*

前面说过Kafka 中 Topic 被分为多个分区（Partition），每个分区又有若干个副本，其中又有包含一个Leader Partition和Flollower Partition。生产者发送过来的消息，会先存到Leader Partition里面，然后再把消息复制到Follower Partition，这样设计的好处就是一旦Leader Partition所在的节点挂了，可以重新从剩余的Partition副本里面选举出新的Leader，然后消费者可以继续从新的Leader Partition里面获取未消费的数据。

为了尽可能的保证新选举出来的Leader Partition里面的数据是最新的，Kafka设计了ISR这样一个方案。ISR全称是 in-sync replica，它是一个集合列表，里面保存的是和Leader Parition节点数据最接近的Follower Partition。如果某个Follower Partition里面的数据落后Leader太多，就会被剔除ISR列表。当Follower Partition 与 Leader Parition相差较小的时候，又再把它加回ISR列表与简单来说，ISR列表里面的节点，同步的数据一定是最新的，所以后续的Leader选举，只需要从ISR列表里面筛选就行了。

<img src="image/640 (12).png" alt="640 (12)" style="zoom:67%;" />

## 4.kafaka之高性能

### a. *<u>使用基于Reactor模型的多路复用模式（Java的NIO实现）</u>*

<img src="image/640 (7).png" alt="640 (7)" style="zoom:67%;" />

### b. *<u>顺序写 + os cache</u>*

Kafka 为了保证磁盘写入性能，通过基于操作系统的页缓存来实现文件写入的。操作系统本身有一层缓存，叫做 page cache，是在内存里的缓存，我们也可以称之为 os cache，意思就是操作系统自己管理的缓存。那么在写磁盘文件的时候，就可以先直接写入 os cache 中，也就是仅仅写入内存中，接下来由操作系统自己决定什么时候把 os cache 里的数据真的刷入到磁盘中, 这样大大提高写入效率和性能。

<img src="image/640 (8).png" alt="640 (8)" style="zoom:67%;" />

另外还有一个非常关键的操作,就是 kafka 在写数据的时候是以**磁盘顺序写**的方式来进行落盘的, 即将数据追加到文件的末尾, 而不是在文件的随机位置来修改数据, 对于普通机械磁盘, 如果是随机写的话, 涉及到磁盘寻址的问题,导致性能确实极低, 但是如果只是按照顺序的方式追加文件末尾的话, 这种磁盘顺序写的性能基本可以跟写内存的性能差不多的。

### c. *<u>零拷贝技术(zero-copy)</u>*

kafaka在的consumer在消费数据的时候，会从kafaka磁盘文件读取数据然后发送给消费者，kafaka在这个过程中引入了零拷贝的技术。即让操作系统的 os cache 中的数据**直接发送到**网卡后传出给下游的消费者，中间跳过了两次拷贝数据的步骤，从而减少拷贝的 CPU 开销, 减少用户态内核态的上下文切换次数, 从而优化数据传输的性能, **而Socket缓存中仅仅会拷贝一个描述符过去，不会拷贝数据到Socket缓存。**

<img src="image/640 (9).png" alt="640 (9)" style="zoom:67%;" />

### d. *<u>压缩传输</u>*

默认情况下, 在 Kafka 生产者中不启用压缩。 压缩不仅可以更快地从生产者传输到代理, 还可以在复制过程中进行更快的传输。压缩有助于提高吞吐量, 降低延迟并提高磁盘利用率。在 Kafka 中, 压缩可能会发生在两个地方: 生产者端和Broker端, 一句话总结下压缩和解压缩, 即 **Producer 端压缩, Broker 端保持, Consumer 端解压缩**。

### e. *<u>服务端内存池</u>*

在kafaka服务端会创建一个32MB大小的内存池，内存池分为两个部分, 一个部分是内存队列, 队列里面有一个个**内存块(16K)**, 另外一部分是可用内存,  一条消息过来后会向内存池申请内存块, 申请完后封装批次并写入数据, sender线程就会发送并响应, 然后清空内存放回内存池里面进行反复使用, 这样就大大减少了GC的频率, 保证了生产者的稳定和高效, 性能会大大提高 。

<img src="image/640 (10).png" alt="640 (10)" style="zoom:67%;" />

## 4.kafaka之高并发

整体网络模型

![640 (11)](image/640 (11).png)



## 5.kafaka存储设计

### a. *<u>背景</u>*

各种存储介质的速度入下图所示。

![640 (/Users/chuckchen/Study/open_source/nine-legged-essay-notes/九股文之系统设计/image/640 (13).png)](image/640 (13).png)

虽然磁盘的读写速度比内存要慢，但假如我们读写磁盘是顺序的，而不是随机的，那么磁盘顺序I/O的性能是要强于内存随机I/O的。（由下图可知，磁盘顺序I/O的性能指标53.2M values/s，而内存的随机I/O性能指标是36.7M values/s）

![640 (/Users/chuckchen/Study/open_source/nine-legged-essay-notes/九股文之系统设计/image/640 (14).png)](image/640 (14).png)

### b. *<u>Kafaka存储方案</u>*

对于 Kafka 来说， 它主要用来处理海量数据流，这个场景的特点主要包括：

-  写操作：写并发要求非常高，基本得达到百万级 TPS，顺序追加写日志即可，无需考虑更新操作
-  读操作：相对写操作来说，比较简单，只要能按照一定规则高效查询即可（offset或者时间戳）

根据上面两点分析，对于写操作来说，直接采用**顺序追加写日志**的方式就可以满足 Kafka 对于百万TPS写入效率要求。但是如何解决高效查询这些日志呢？ 

- MySQL的B+树索引：那么每次写都要维护索引，还需要有额外空间来存储索引、更会出现关系型数据库中经常出现的“数据页分裂”等操作，所以对Kafaka这种高并发写的系统来说并不适合。
- 哈希索引：读写都是O(1)的时间复杂度，但是哈希索引需要常驻内存，对于Kafka 每秒写入几百万消息数据来说容易造成oom，所以也不适合。

Kafka最终的存储方案是**基于顺序追加写日志 + 稀疏哈希索引。**前面说过Kafka的日志是顺序写的，且日志只有追加写，没有删除改动的情况。所以Kafka直接将消息划分成若干个块，**对于每个块，我们只需要索引当前块的第一条消息的 Offset，这种只索引块的索引就叫稀疏索引。当消费者拿着具体的Offset来消费的时候，可以先通过二分查找找到Offset对应的块，然后在块中顺序查找。（稀疏索引存储于磁盘中，但通过mmap映射到虚拟内存上）。 **

<img src="image/640 (15).png" alt="640 (15)" style="zoom:67%;" />

