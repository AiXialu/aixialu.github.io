## Message Queue

### 你们为什么使用 mq？具体的使用场景是什么？

> 削峰，异步，解耦

mq 的作用很简单，削峰填谷。以电商交易下单的场景来说，正向交易的过程可能涉及到创建订单、扣减库存、扣减活动预算、扣减积分等等。每个接口的耗时如果是 100ms，那么理论上整个下单的链路就需要耗费 400ms，这个时间显然是太长了。

![](https://pic1.zhimg.com/80/v2-9be1ef7a125254dcc46926a474a593b0_720w.webp)

如果这些操作全部同步处理的话，首先调用链路太长影响接口性能，其次分布式事务的问题很难处理，这时候像扣减预算和积分这种对实时一致性要求没有那么高的请求，完全就可以通过 mq 异步的方式去处理了。同时，考虑到异步带来的不一致的问题，我们可以通过 job 去重试保证接口调用成功，而且一般公司都会有核对的平台，比如下单成功但是未扣减积分的这种问题可以通过核对作为兜底的处理方案。

![](https://pic2.zhimg.com/80/v2-97a2a9d6573f4647e3a850eb1c87f405_720w.webp)

使用 mq 之后我们的链路变简单了，同时异步发送消息我们的整个系统的抗压能力也上升了。

### 那你们使用什么 mq？基于什么做的选型？

我们主要调研了几个主流的 mq，kafka、rabbitmq、rocketmq、activemq，选型我们主要基于以下几个点去考虑：

由于我们系统的 qps 压力比较大，所以性能是首要考虑的要素。
开发语言，由于我们的开发语言是 java，主要是为了方便二次开发。
对于高并发的业务场景是必须的，所以需要支持分布式架构的设计。
功能全面，由于不同的业务场景，可能会用到顺序消息、事务消息等。

![](https://pic4.zhimg.com/80/v2-ca4923be2e937a3ef9b5b0d98b97ffd7_720w.webp)

### 你上面提到异步发送，那消息可靠性怎么保证？

消息丢失可能发生在生产者发送消息、MQ 本身丢失消息、消费者丢失消息 3 个方面。

#### 生产者丢失

生产者丢失消息的可能点在于程序发送失败抛异常了没有重试处理，或者发送的过程成功但是过程中网络闪断 MQ 没收到，消息就丢失了。

由于同步发送的一般不会出现这样使用方式，所以我们就不考虑同步发送的问题，我们基于异步发送的场景来说。

异步发送分为两个方式：异步有回调和异步无回调，无回调的方式，生产者发送完后不管结果可能就会造成消息丢失，而通过异步发送+回调通知+本地消息表的形式我们就可以做出一个解决方案。以下单的场景举例。

下单后先保存本地数据和 MQ 消息表，这时候消息的状态是发送中，如果本地事务失败，那么下单失败，事务回滚。
下单成功，直接返回客户端成功，异步发送 MQ 消息
MQ 回调通知消息发送结果，对应更新数据库 MQ 发送状态
JOB 轮询超过一定时间（时间根据业务配置）还未发送成功的消息去重试
在监控平台配置或者 JOB 程序处理超过一定次数一直发送不成功的消息，告警，人工介入。

![](https://pic4.zhimg.com/80/v2-afce501e1819bc2e693e9c5f058722fb_720w.webp)

#### MQ 丢失

如果生产者保证消息发送到 MQ，而 MQ 收到消息后还在内存中，这时候宕机了又没来得及同步给从节点，就有可能导致消息丢失。

```
比如RocketMQ：

RocketMQ分为同步刷盘和异步刷盘两种方式，默认的是异步刷盘，就有可能导致消息还未刷到硬盘上就丢失了，可以通过设置为同步刷盘的方式来保证消息可靠性，这样即使MQ挂了，恢复的时候也可以从磁盘中去恢复消息。
```

```
比如Kafka也可以通过配置做到：

acks=all 只有参与复制的所有节点全部收到消息，才返回生产者成功。这样的话除非所有的节点都挂了，消息才会丢失。
replication.factor=N,设置大于1的数，这会要求每个partion至少有2个副本
min.insync.replicas=N，设置大于1的数，这会要求leader至少感知到一个follower还保持着连接
retries=N，设置一个非常大的值，让生产者发送失败一直重试

```

#### 消费者丢失

消费者丢失消息的场景：消费者刚收到消息，此时服务器宕机，MQ 认为消费者已经消费，不会重复发送消息，消息丢失。

RocketMQ 默认是需要消费者回复 ack 确认，而 kafka 需要手动开启配置关闭自动 offset。

消费方不返回 ack 确认，重发的机制根据 MQ 类型的不同发送时间间隔、次数都不尽相同，如果重试超过次数之后会进入死信队列，需要手工来处理了。（Kafka 没有这些）

![](https://pic2.zhimg.com/80/v2-d8b454a2dd281bfb631fdce70fde5b99_720w.webp)

#### 你说到消费者消费失败的问题，那么如果一直消费失败导致消息积压怎么处理？

因为考虑到时消费者消费一直出错的问题，那么我们可以从以下几个角度来考虑：

    消费者出错，肯定是程序或者其他问题导致的，

    如果容易修复，先把问题修复，让consumer恢复正常消费如果时间来不及处理很麻烦，做转发处理，写一个临时的consumer消费方案，先把消息消费，然后再转发到一个新的topic和MQ资源，

    这个新的topic的机器资源单独申请，要能承载住当前积压的消息处理完积压数据后,修复consumer，去消费新的MQ和现有的MQ数据，新MQ消费完成后恢复原状

![](https://pic3.zhimg.com/80/v2-4cd603c8f3b96d8889f62ac18eda01b2_720w.webp)

#### 那如果消息积压达到磁盘上限，消息被删除了怎么办

最初，我们发送的消息记录是落库保存了的，而转发发送的数据也保存了，那么我们就可以通过这部分数据来找到丢失的那部分数据，再单独跑个脚本重发就可以了。如果转发的程序没有落库，那就和消费方的记录去做对比，只是过程会更艰难一点。
