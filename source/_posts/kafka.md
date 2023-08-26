## Kafka

### Overall

Kafka 特点：

    可靠性：具有副本及容错机制。
    可扩展性：kafka无需停机即可扩展节点及节点上线。
    持久性：数据存储到磁盘上，持久性保存。
    性能：kafka具有高吞吐量。达到TB级的数据，也有非常稳定的性能。
    速度快：顺序写入和零拷贝技术使得kafka延迟控制在毫秒级。

### Kafka 底层原理

先看下 Kafka 系统的架构

![](https://ask.qcloudimg.com/http-save/2039230/lw4iwwg233.png)

kafka 支持消息持久化，消费端是主动拉取数据，消费状态和订阅关系由客户端负责维护，消息消费完后，不会立即删除，会保留历史消息。因此支持多订阅时，消息只会存储一份就可以。

    broker：kafka集群中包含一个或者多个服务实例（节点），这种服务实例被称为broker（一个broker就是一个节点/一个服务器）；
    topic：每条发布到kafka集群的消息都属于某个类别，这个类别就叫做topic；
    partition：partition是一个物理上的概念，每个topic包含一个或者多个partition；
    segment：一个partition当中存在多个segment文件段，每个segment分为两部分，.log文件和 .index 文件，其中 .index 文件是索引文件，主要用于快速查询， .log 文件当中数据的偏移量位置；
    producer：消息的生产者，负责发布消息到 kafka 的 broker 中；
    consumer：消息的消费者，向 kafka 的 broker 中读取消息的客户端；
    consumer group：消费者组，每一个 consumer 属于一个特定的 consumer group（可以为每个consumer指定 groupName）；
    .log：存放数据文件；
    .index：存放.log文件的索引数据。

### Kafka 主要组件

1. producer（生产者）

producer 主要是用于生产消息，是 kafka 当中的消息生产者，生产的消息通过 topic 进行归类，保存到 kafka 的 broker 里面去。

2. topic（主题）

   kafka 将消息以 topic 为单位进行归类；
   topic 特指 kafka 处理的消息源（feeds of messages）的不同分类；
   topic 是一种分类或者发布的一些列记录的名义上的名字。kafka 主题始终是支持多用户订阅的；也就是说，一 个主题可以有零个，一个或者多个消费者订阅写入的数据；
   在 kafka 集群中，可以有无数的主题；
   生产者和消费者消费数据一般以主题为单位。更细粒度可以到分区级别。

3. partition（分区）

kafka 当中，topic 是消息的归类，一个 topic 可以有多个分区（partition），每个分区保存部分 topic 的数据，所有的 partition 当中的数据全部合并起来，就是一个 topic 当中的所有的数据。

一个 broker 服务下，可以创建多个分区，broker 数与分区数没有关系；

在 kafka 中，每一个分区会有一个编号：编号从 0 开始。

每一个分区内的数据是有序的，但全局的数据不能保证是有序的。（有序是指生产什么样顺序，消费时也是什么样的顺序）

4. consumer（消费者）

consumer 是 kafka 当中的消费者，主要用于消费 kafka 当中的数据，消费者一定是归属于某个消费组中的。

5. consumer group（消费者组）

消费者组由一个或者多个消费者组成，同一个组中的消费者对于同一条消息只消费一次。

每个消费者都属于某个消费者组，如果不指定，那么所有的消费者都属于默认的组。

每个消费者组都有一个 ID，即 group ID。组内的所有消费者协调在一起来消费一个订阅主题( topic)的所有分区(partition)。当然，每个分区只能由同一个消费组内的一个消费者(consumer)来消费，可以由不同的消费组来消费.

![](https://ask.qcloudimg.com/http-save/2039230/ev6txob5p8.png)

![](https://ask.qcloudimg.com/http-save/2039230/j1r2rsufsw.png)

> 总结下 kafka 中分区与消费组的关系：

    消费组： 由一个或者多个消费者组成，同一个组中的消费者对于同一条消息只消费一次。
    某一个主题下的分区数，对于消费该主题的同一个消费组下的消费者数量，应该小于等于该主题下的分区数。

如：某一个主题有 4 个分区，那么消费组中的消费者应该小于等于 4，而且最好与分区数成整数倍 1 2 4 这样。同一个分区下的数据，在同一时刻，不能同一个消费组的不同消费者消费。

> 总结：分区数越多，同一时间可以有越多的消费者来进行消费，消费数据的速度就会越快，提高消费的性能。

#### kafka 分区副本

副本数（replication-factor）：控制消息保存在几个 broker（服务器）上，一般情况下副本数等于 broker 的个数。

一个 broker 服务下，不可以创建多个副本因子。创建主题时，副本因子应该小于等于可用的 broker 数。

副本因子操作以分区为单位的。每个分区都有各自的主副本和从副本；

主副本叫做 leader，从副本叫做 follower（在有多个副本的情况下，kafka 会为同一个分区下的所有分区，设定角色关系：一个 leader 和 N 个 follower），处于同步状态的副本叫做 in-sync-replicas(ISR);

follower 通过拉的方式从 leader 同步数据。
消费者和生产者都是从 leader 读写数据，不与 follower 交互。

副本因子的作用：让 kafka 读取数据和写入数据时的可靠性。

副本因子是包含本身，同一个副本因子不能放在同一个 broker 中。

如果某一个分区有三个副本因子，就算其中一个挂掉，那么只会剩下的两个中，选择一个 leader，但不会在其他的 broker 中，另启动一个副本（因为在另一台启动的话，存在数据传递，只要在机器之间有数据传递，就会长时间占用网络 IO，kafka 是一个高吞吐量的消息系统，这个情况不允许发生）所以不会在另一个 broker 中启动。

如果所有的副本都挂了，生产者如果生产数据到指定分区的话，将写入不成功。

lsr 表示：当前可用的副本。

#### segment 文件

一个 partition 当中由多个 segment 文件组成，每个 segment 文件，包含两部分，一个是 .log 文件，另外一个是 .index 文件，其中 .log 文件包含了我们发送的数据存储，.index 文件，记录的是我们.log 文件的数据索引值，以便于我们加快数据的查询速度。

索引文件与数据文件的关系

既然它们是一一对应成对出现，必然有关系。索引文件中元数据指向对应数据文件中 message 的物理偏移地址。

比如索引文件中 3,497 代表：数据文件中的第三个 message，它的偏移地址为 497。

再来看数据文件中，Message 368772 表示：在全局 partiton 中是第 368772 个 message。

     注：segment index file 采取稀疏索引存储方式，减少索引文件大小，通过mmap（内存映射）可以直接内存操作，稀疏索引为数据文件的每个对应message设置一个元数据指针，它比稠密索引节省了更多的存储空间，但查找起来需要消耗更多的时间。

.index 与 .log 对应关系如下：

![](https://ask.qcloudimg.com/http-save/2039230/002wqqdvqv.png)

上图左半部分是索引文件，里面存储的是一对一对的 key-value，其中 key 是消息在数据文件（对应的 log 文件）中的编号，比如“1,3,6,8……”，
分别表示在 log 文件中的第 1 条消息、第 3 条消息、第 6 条消息、第 8 条消息……

那么为什么在 index 文件中这些编号不是连续的呢？
这是因为 index 文件中并没有为数据文件中的每条消息都建立索引，而是采用了稀疏存储的方式，每隔一定字节的数据建立一条索引。
这样避免了索引文件占用过多的空间，从而可以将索引文件保留在内存中。
但缺点是没有建立索引的 Message 也不能一次定位到其在数据文件的位置，从而需要做一次顺序扫描，但是这次顺序扫描的范围就很小了。

value 代表的是在全局 partiton 中的第几个消息。

以索引文件中元数据 3,497 为例，其中 3 代表在右边 log 数据文件中从上到下第 3 个消息，
497 表示该消息的物理偏移地址（位置）为 497(也表示在全局 partiton 表示第 497 个消息-顺序写入特性)。

#### log 日志目录及组成

kafka 在我们指定的 log.dir 目录下，会创建一些文件夹；名字是 （主题名字-分区名） 所组成的文件夹。 在（主题名字-分区名）的目录下，会有两个文件存在，如下所示：

```
#索引文件
00000000000000000000.index
#日志内容
00000000000000000000.log
```

在目录下的文件，会根据 log 日志的大小进行切分，.log 文件的大小为 1G 的时候，就会进行切分文件；如下：

```
-rw-r--r--. 1 root root 389k  1月  17  18:03   00000000000000000000.index
-rw-r--r--. 1 root root 1.0G  1月  17  18:03   00000000000000000000.log
-rw-r--r--. 1 root root  10M  1月  17  18:03   00000000000000077894.index
-rw-r--r--. 1 root root 127M  1月  17  18:03   00000000000000077894.log
```

在 kafka 的设计中，将 offset 值作为了文件名的一部分。

segment 文件命名规则：partion 全局的第一个 segment 从 0 开始，后续每个 segment 文件名为上一个全局 partion 的最大 offset（偏移 message 数）。数值最大为 64 位 long 大小，20 位数字字符长度，没有数字就用 0 填充。

通过索引信息可以快速定位到 message。通过 index 元数据全部映射到内存，可以避免 segment File 的 IO 磁盘操作；

通过索引文件稀疏存储，可以大幅降低 index 文件元数据占用空间大小。

稀疏索引：为了数据创建索引，但范围并不是为每一条创建，而是为某一个区间创建；
好处：就是可以减少索引值的数量。
不好的地方：找到索引区间之后，要得进行第二次处理。

#### message 的物理结构

生产者发送到 kafka 的每条消息，都被 kafka 包装成了一个 message

message 的物理结构如下图所示：

![](https://ask.qcloudimg.com/http-save/2039230/5q1z5tmizh.png)

所以生产者发送给 kafka 的消息并不是直接存储起来，而是经过 kafka 的包装，每条消息都是上图这个结构，只有最后一个字段才是真正生产者发送的消息数据。

#### kafka 中的数据不丢失机制

1. 生产者生产数据不丢失

   发送消息方式

生产者发送给 kafka 数据，可以采用同步方式或异步方式

- 同步方式：

发送一批数据给 kafka 后，等待 kafka 返回结果：

    生产者等待10s，如果broker没有给出ack响应，就认为失败。
    生产者重试3次，如果还没有响应，就报错.

- 异步方式：

发送一批数据给 kafka，只是提供一个回调函数：

    先将数据保存在生产者端的buffer中。buffer大小是2万条 。
    满足数据阈值或者数量阈值其中的一个条件就可以发送数据。
    发送一批数据的大小是500条。

> 注：如果 broker 迟迟不给 ack，而 buffer 又满了，开发者可以设置是否直接清空 buffer 中的数据。

2. ack 机制（确认机制）

生产者数据发送出去，需要服务端返回一个确认码，即 ack 响应码；ack 的响应有三个状态值 0,1，-1

0：生产者只负责发送数据，不关心数据是否丢失，丢失的数据，需要再次发送

1：partition 的 leader 收到数据，不管 follow 是否同步完数据，响应的状态码为 1

-1：所有的从节点都收到数据，响应的状态码为-1

> 如果 broker 端一直不返回 ack 状态，producer 永远不知道是否成功；producer 可以设置一个超时时间 10s，超过时间认为失败。

2. broker 中数据不丢失

在 broker 中，保证数据不丢失主要是通过副本因子（冗余），防止数据丢失。

3. 消费者消费数据不丢失

在消费者消费数据的时候，只要每个消费者记录好 offset 值即可，就能保证数据不丢失。也就是需要我们自己维护偏移量(offset)，可保存在 Redis 中。

### 选举

Kafka 选举分为： leader-follower 选举, broker 选举 和 consumer group 选举

1. broker 选举

> Kafka 核心总控制器 Controller

    在Kafka集群中会有一个或者多个broker，其中有一个broker会被选举为控制器（Kafka Controller），它负责管理整个集群中所有分区和副本的状态。
    当某个分区的leader副本出现故障时，由控制器负责为该分区选举新的leader副本。
    当检测到某个分区的ISR集合发生变化时，由控制器负责通知所有broker更新其元数据信息。
    当使用kafka-topics.sh脚本为某个topic增加分区数量时，同样还是由控制器负责分区的重新分配。

    Kafka控制器的作用是管理和协调Kafka集群，具体如下：

- 主题管理：创建、删除 Topic，以及增加 Topic 分区等操作都是由控制器执行。
- 分区重分配：执行 Kafka 的 reassign 脚本对 Topic 分区重分配的操作，也是由控制器实现。
- Preferred leader 选举。

> 因为在 Kafka 集群长时间运行中，broker 的宕机或崩溃是不可避免的，leader 就会发生转移，即使 broker 重新回来，也不会是 leader 了。在众多 leader 的转移过程中，就会产生 leader 不均衡现象，可能一小部分 broker 上有大量的 leader，影响了整个集群的性能，所以就需要把 leader 调整回最初的 broker 上，这就需要 Preferred leader 选举。

- 集群成员管理：控制器能够监控新 broker 的增加，broker 的主动关闭与被动宕机，进而做其他工作。这也是利用 Zookeeper 的 ZNode 模型和 Watcher 机制，控制器会监听 Zookeeper 中/brokers/ids 下临时节点的变化。

- 数据服务：控制器上保存了最全的集群元数据信息，其他所有 broker 会定期接收控制器发来的元数据更新请求，从而更新其内存中的缓存数据。

- Controller 选举机制

1. 在 kafka 集群启动的时候，会自动选举一台 broker 作为 controller 来管理整个集群，选举的过程是集群中每个 broker 都会尝试在 zookeeper 上创建一个 /controller 临时节点，zookeeper 会保证有且仅有一个 broker 能创建成功，这个 broker 就会成为集群的总控器 controller。

   > 集群中第一个启动的 Broker 会通过在 Zookeeper 中创建临时节点/controller 来让自己成为控制器，其他 Broker 启动时也会在 zookeeper 中创建临时节点，但是发现节点已经存在，所以它们会收到一个异常，意识到控制器已经存在，那么就会在 Zookeeper 中创建 watch 对象，便于它们收到控制器变更的通知。

2. 当这个 controller 角色的 broker 宕机了，此时 zookeeper 临时节点会消失，集群里其他 broker 会一直监听这个临时节点，发现临时节点消失了，就竞争再次创建临时节点，就是我们上面说的选举机制，zookeeper 又会保证有一个 broker 成为新的 controller。

   那么如果控制器由于网络原因与 Zookeeper 断开连接或者异常退出，那么其他 broker 通过 watch 收到控制器变更的通知，就会去尝试创建临时节点/controller，如果有一个 Broker 创建成功，那么其他 broker 就会收到创建异常通知，也就意味着集群中已经有了控制器，其他 Broker 只需创建 watch 对象即可。

   如果集群中有一个 Broker 发生异常退出了，那么控制器就会检查这个 broker 是否有分区的副本 leader，如果有那么这个分区就需要一个新的 leader，此时控制器就会去遍历其他副本，决定哪一个成为新的 leader，同时更新分区的 ISR 集合。

   如果有一个 Broker 加入集群中，那么控制器就会通过 Broker ID 去判断新加入的 Broker 中是否含有现有分区的副本，如果有，就会从分区副本中去同步数据。

具备控制器身份的 broker 需要比其他普通的 broker 多一份职责，具体细节如下：

    1. 监听broker相关的变化。为Zookeeper中的/brokers/ids/节点添加BrokerChangeListener，用来处理broker增减的变化。
    2. 监听topic相关的变化。为Zookeeper中的/brokers/topics节点添加TopicChangeListener，用来处理topic增减的变化；为Zookeeper中的/admin/delete_topics节点添加TopicDeletionListener，用来处理删除topic的动作。
    3. 从Zookeeper中读取获取当前所有与topic、partition以及broker有关的信息并进行相应的管理。对于所有topic所对应的Zookeeper中的/brokers/topics/[topic]节点添加PartitionModificationsListener，用来监听topic中的分区分配变化。
    4. 更新集群的元数据信息，同步到其他普通的broker节点中。

#### 防止控制器脑裂 Split Brain

如果控制器所在 broker 挂掉了或者 Full GC 停顿时间太长超过 zookeepersession timeout 出现假死，Kafka 集群必须选举出新的控制器，但如果之前被取代的控制器又恢复正常了，它依旧是控制器身份，这样集群就会出现两个控制器，这就是控制器脑裂问题。

解决方法：

为了解决 Controller 脑裂问题，ZooKeeper 中还有一个与 Controller 有关的持久节点/controller_epoch，存放的是一个整形值的 epoch number（纪元编号，也称为隔离令牌），集群中每选举一次控制器，就会通过 Zookeeper 创建一个数值更大的 epoch number，如果有 broker 收到比这个 epoch 数值小的数据，就会忽略消息。

- leader follower 选举

当 Leader 服务器出现网络中断、崩溃退出与重启等异常情况时，就会进入 Leader 选举过程，这个过程会选举产生新的 Leader 服务器。

ZooKeeper 服务器的四种状态：

    looking：寻找Leader的状态，当前集群没有leader
    leading：成为一个leader节点。
    following：成为一个follower节点。
    observing：当前服务角色是一个observer。

1. ZooKeeper 集群启动时的选举

假如有三个节点(s1,s2,s3)组成的集群。在集群启动过程中，当有一台 ZooKeeper 节点 s1 启动完成后，此时集群中只有一个节点无法进行 leader 的选举。当第二个节点 s2 启动成功后，此时两个节点可以正常通信，进入 leader 的选举过程，具体如下：

1）每一个节点都是自私的，各自都投自己 1 票。每次投票都会包含选举服务器的 myid 和 zxid，投票结果使用(myid,zxid)表示，此时 s1 的投票为(1,0)，s2 为(2,0)。然后各自投票给集群中其他节点。

2）集群中的每个节点都接收其他节点的投票，s1 将(1,0)发给 s2，s2 将(2,0)发给 s1。

3）PK 投票结果，对于每一次投票，服务器都会将自己的投票结果和其他节点做比较：

> 首先检查 zxid，zxid 大的优先作为 leader。如果 zxid 相同，则比较 myid，myid 大的优先作为 leader。

对于 s1，它的投票是（1,0），接收到 s2 的投票（2,0），首先它会比较自己的 zxid 和 s2 的 zxid，此时相等，然后比较 myid，s2 的 myid 大，s1 更新自己的投票结果为(2,0)，然后重新投票，对于 s2 收到 s1 的投票(1,0)，显然 s2 胜出，无需更新自己的投票结果。
4）统计投票。每次投票结束后，服务器都会统计投票信息，判断是否已经有过半机器接收到相同的投票信息，对于 s1，s2，都统计出集群中已经有两台机器是（2,0）的投票信息，此时认为 leader 已经产生。

5）改变服务器状态。一旦确定了 leader，每个节点都会更改自己的状态，胜出的 leader 将状态改为 leading，败者 follower 更改自己的状态为 following。

2. Zookeeper 集群运行中 leader 选举

假如有三台服务器(s1,s2,s3)组成的集群，s2 是 leader。在集群运行中时，只有当集群中的 leader 宕机才会触发 leader 的重新选举，集群中 follower 宕机或者新节点的加入并不影响 leader 的地位。

选举过程如下：

1）状态更改。leader 宕机之后，各节点将自己的状态更改为 looking，然后进入 leader 的选举。

2）每个节点都会发出投票，运行期间每个节点的 zxid 可能不同，假设 s1 的 zxid 为 122，s3 的 zxid 也为 122，在第一轮投票中 s1，s2 都会投自己，产生投票 s1(1,122),s2(3,122)，然后各自将投票发送给其他节点。
3）接收来自其他节点的投票。

4）处理投票。PK 投票，与启动时相同，此时 s3 胜出。

5）统计投票。s1，s3 的投票都为(3,122)，已经有过半机器是(3,122)，s3 成为 leader。

6）更改状态。s1，s3 更改自己的状态，s1 修改为 following，s3 修改为 leading

### ZooKeeper 集群为什么最好奇数台？

ZooKeeper 集群在宕掉几个 ZooKeeper 服务器之后，如果剩下的 ZooKeeper 服务器个数大于宕掉的个数的话整个 ZooKeeper 才依然可用。假如我们的集群中有 n 台 ZooKeeper 服务器，那么也就是剩下的服务数必须大于 n/2。

比如假如我们有 3 台，那么最大允许宕掉 1 台 ZooKeeper 服务器，如果我们有 4 台的的时候也同样只允许宕掉 1 台。 假如我们有 5 台，那么最大允许宕掉 2 台 ZooKeeper 服务器，如果我们有 6 台的的时候也同样只允许宕掉 2 台。

### ZooKeeper 选举的过半机制防止脑裂

#### 集群 split brain

对于一个集群，通常多台机器会部署在不同机房，来提高这个集群的可用性。保证可用性的同时，会发生一种机房间网络线路故障，导致机房间网络不通，而集群被割裂成几个小集群。这时候子集群各自选主导致“脑裂”的情况。

举例说明：比如现在有一个由 6 台服务器所组成的一个集群，部署在了 2 个机房，每个机房 3 台。正常情况下只有 1 个 leader，但是当两个机房中间网络断开的时候，每个机房的 3 台服务器都会认为另一个机房的 3 台服务器下线，而选出自己的 leader 并对外提供服务。若没有过半机制，当网络恢复的时候会发现有 2 个 leader。仿佛是 1 个大脑（leader）分散成了 2 个大脑，这就发生了脑裂现象。脑裂期间 2 个大脑都可能对外提供了服务，这将会带来数据一致性等问题。

#### 过半机制是如何防止脑裂现象产生的？

ZooKeeper 的过半机制导致不可能产生 2 个 leader，因为少于等于一半是不可能产生 leader 的，这就使得不论机房的机器如何分配都不可能发生脑裂。

- consume group 选举

1. 消费组选主

在 Kafka 的消费端，会有一个消费者协调器以及消费组，组协调器（Group Coordinator）需要为消费组内的消费者选举出一个消费组的 leader。

如果消费组内还没有 leader，那么第一个加入消费组的消费者即为消费组的 leader，如果某一个时刻 leader 消费者由于某些原因退出了消费组，那么就会重新选举 leader，选举方式如下：

```
private val members = new mutable.HashMap[String, MemberMetadata]
leaderId = members.keys.headOption
```

在组协调器中消费者的信息是以 HashMap 的形式存储的，其中 key 为消费者的 member_id，而 value 是消费者相关的元数据信息。而 leader 的取值为 HashMap 中的第一个键值对的 key **（等同于随机）**.

    消费组的Leader和Coordinator没有关联。消费组的leader负责Rebalance过程中消费分配方案的制定。

2. 消费端 Rebalance 机制

就 Kafka 消费端而言，有一个难以避免的问题就是消费者的重平衡即 Rebalance。Rebalance 是让一个消费组的所有消费者就如何消费订阅 topic 的所有分区达成共识的过程，在 Rebalance 过程中，所有 Consumer 实例都会停止消费，等待 Rebalance 的完成。因为要停止消费等待重平衡完成，因此 Rebalance 会严重影响消费端的 TPS，是应当尽量避免的。
触发 Rebalance 的时机

Rebalance 的触发条件有 3 个。

    消费组成员个数发生变化。例如有新的Consumer实例加入或离开该消费组。
    订阅的 Topic 个数发生变化。
    订阅 Topic 的分区数发生变化。

Rebalance 发生时，Group 下所有 Consumer 实例都会协调在一起共同参与，kafka 能够保证尽量达到最公平的分配。但是 Rebalance 过程对 consumer group 会造成比较严重的影响。在 Rebalance 的过程中 consumer group 下的所有消费者实例都会停止工作，等待 Rebalance 过程完成。

3. Rebalance 过程

Rebalance 过程分为两步：Join 和 Sync。

    Join。所有成员都向Group Coordinator发送JoinGroup请求，请求加入消费组。一旦所有成员都发送了JoinGroup请求，Coordinator会从中选择一个Consumer担任leader的角色，并把组成员信息以及订阅信息发给leader——注意leader和coordinator不是一个概念。leader负责消费分配方案的制定。

![](https://ask.qcloudimg.com/http-save/yehe-5086501/50792077f8ac74f0b8b6442a33b9275f.png)

    Sync。这一步leader开始分配消费方案，即哪个consumer负责消费哪些topic的哪些partition。一旦完成分配，leader会将这个方案封装进SyncGroup请求中发给coordinator，非leader也会发SyncGroup请求，只是内容为空。coordinator接收到分配方案之后会把方案塞进SyncGroup的response中发给各个consumer。这样组内的所有成员就都知道自己应该消费哪些分区了。

![](https://ask.qcloudimg.com/http-save/yehe-5086501/16db83b75ade4388e1cfb4ef07544263.png)

4. 避免不必要的 Rebalance

前面说过 Rebalance 发生的时机有三个，后两个时机是可以人为避免的。发生 Rebalance 最常见的原因是消费组成员个数发生变化。

这其中消费者成员正常的添加和停掉导致 Rebalance，也是无法避免。但是在某些情况下，Consumer 实例会被 Coordinator 错误地认为已停止从而被踢出 Group。从而导致 rebalance。

```
这种情况可以通过Consumer端的参数session.timeout.ms和max.poll.interval.ms进行配置。
```

```
除了这个参数，Consumer还提供了控制发送心跳请求频率的参数，就是heartbeat.interval.ms。这个值设置得越小，Consumer实例发送心跳请求的频率就越高。频繁地发送心跳请求会额外消耗带宽资源，但好处是能够更快地知道是否开启Rebalance，因为Coordinator通知各个Consumer实例是否开启Rebalance就是将REBALANCE_NEEDED标志封装进心跳请求的响应体中。
```

总之，要为业务处理逻辑留下充足的时间使 Consumer 不会因为处理这些消息的时间太长而引发 Rebalance，但也不能时间设置过长导致 Consumer 宕机但迟迟没有被踢出 Group。

### Kafka 舍弃 ZooKeeper 的理由

Kafka 目前强依赖于 ZooKeeper：ZooKeeper 为 Kafka 提供了元数据的管理，例如一些 Broker 的信息、主题数据、分区数据等等，还有一些选举、扩容等机制也都依赖 ZooKeeper。

1. 运维复杂度

运维 Kafka 的同时需要保证一个高可用的 Zookeeper 集群，增加了运维和故障排查的复杂度。

2.  性能差

在一些大公司，Kafka 集群比较大，分区数很多的时候，ZooKeeper 存储的元数据就会很多，性能就会变差。
ZooKeeper 需要选举，选举的过程中是无法提供服务的。
Zookeeper 节点如果频繁发生 Full Gc，与客户端的会话将超时，由于无法响应客户端的心跳请求，从而与会话相关联的临时节点也会被删除。
