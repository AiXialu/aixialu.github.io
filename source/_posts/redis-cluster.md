---
title: redis-cluster
date: 2020-10-28 15:09:12
tags:
---

## Redis 集群

### 故障检测与修复

#### 故障恢复总体介绍

Master-Slave:

1. 下线 Master 节点的 Slave 节点来成为新的 Master 节点，接管旧 Master 负责的所有 slots 向外提供服务
2. 当集群中的某个 Master 节点没有 Slave 节点时（称之为 Orphaned Master），其他有富余 Slave 节点的主节点会向该节点迁移一个 Slave 节点以防该节点下线之后没有子节点来替换从而导致整个集群下线(Replica Migration)

- 故障发现： PFAIL -- FAIL

  PFAIL 就是主观下线，比如节点 1 判定节点 3 下线，那么他会标记节点 3 的状态为 PFAIL。但是如果绝大部分节点都判定节点 3 为 PFAIL，那么我们就可以断定节点 3 故障下线，其状态判定为 FAIL 状态

  Redis 的每个节点会不停的向其他节点发送 PING 消息来与其他节点同步信息的同时检测其他节点是否可达

  1. Ping 不同 --》 重新连接
  2. 连接超时 --》 PFAIL
  3. 然后向其他随机节点发送 PFAIL 信息，然后收到信息的节点在进行判断
  4. 当超过一半的节点 发现 PFAIL 的时候， -》 FAIL 并且广播出去

- 故障迁移

1. 资格检查

   Slave 节点会不停的与 Master 节点通信来复制 Master 节点的数据，如果一个 Slave 节点长时间不与 Master 节点通信，那么很可能意味着该 Slave 节点上的数据已经落后 Master 节点过多（因为 Master 节点再不停的更新数据但是 Slave 节点并没有随之更新）

   - 原理：

   Redis 用了和 Raft 算法 term（任期）类似的的概念，在 Redis 中叫作 epoch（纪元），epoch 是一个无符号的 64 整数，一个节点的 epoch 从 0 开始。

   如果一个节点接收到的 epoch 比自己的大，则将自已的 epoch 更新接收到的 epoch（假定为信任网络，无拜占庭将军问题）。

   每个 master 都会在 ping 和 pong 消息中广播自己的 epoch 和所负责的 slots 位图，slave 发起选举时，创建一个新的 epoch（增一），epoch 的值会持久化到文件 nodes.conf 中，如（最新 epoch 值为 27，最近一次投票给了 27）：

   ```
   vars currentEpoch 27 lastVoteEpoch 27
   ```

2. 休眠：Raft

   所有参与选举的节点首先随机休眠一段时间，每个节点一旦唤醒就立刻向所有的投票节点发起拉票请求。对于投票节点来说，每一轮选举中只能投出一票，投票的规则就是先到先得。所以一般情况下，都是休眠时间最短的节点容易获得大部分投票

   - 一部分为固定的 500ms 时间，这 500ms 主要是为了等待集群状态同步。上面我们讲到节点 2 会向集群所有节点广播消息，那么这 500ms 就是等待确保集群的所有节点都收到了消息并更新了状态。
   - 另一部分主要是一个随机的时间加上由该 Slave 节点的排名决定的附加时间。我们在之前主从复制的文章中讲过，每个 slave 都会记录自己从主节点同步数据的复制偏移量。复制偏移量越大，说明该节点与主节点数据保持的越一致。那么显然我们选举的时候肯定是想选状态更新最近的子节点，所以我们按照更新状态的排序来确定休眠时间的附加部分。状态更新最近的节点 SLAVE_RANK 排名为 1，那么其休眠的时间相应的也最短，也就意味着该节点最有可能获得大部分选票。

3. 发起拉票&选举投票

   主节点才能投票

   1. 对一个 epoch，只投票一次；

   2. 会拒绝所有更小 epoch 的投票请求；

   3. 不会给小于 lastVoteEpoch 的 epoch 投票；

   4. master 只给 master 状态为 fail 的 slave 投票；

   5. 如果 slave 请求的 currentEpoch 小于 master 的 currentEpoch，则 master 忽略该请求，但下列情况例外：

   一个 slave 发起选举的条件：

   1. 它的 master 为 fail 状态（非 pfail 状态）；

   2. 它的 master 至少负责了一个 slot；

   3. slave 和 master 的复制连接断开时间不超过给定的值（值可配置，目的是确保 slave 上的数据足够完整，所以运维时不能任由一个 slave 长时间不可用，需要通过监控将异常的 slave 及时恢复）。

   发起选举前，slave 先给自己的 epoch（即 currentEpoch）增一，然后请求其它 master 给自己投票。slave 是通过广播 FAILOVER_AUTH_REQUEST 包给集中的每一个 masters。

   slave 发起投票后，会等待至少两倍 NODE_TIMEOUT 时长接收投票结果，不管 NODE_TIMEOUT 何值，也至少会等待 2 秒。

   master 接收投票后给 slave 响应 FAILOVER_AUTH_ACK，并且在（NODE_TIMEOUT\*2）时间内不会给同一 master 的其它 slave 投票。

   如果 slave 收到 FAILOVER_AUTH_ACK 响应的 epoch 值小于自己的 epoch，则会直接丢弃。一旦 slave 收到多数 master 的 FAILOVER_AUTH_ACK，则声明自己赢得了选举。

   如果 slave 在两倍的 NODE_TIMEOUT 时间内（至少 2 秒）未赢得选举，则放弃本次选举，然后在四倍 NODE_TIMEOUT 时间（至少 4 秒）后重新发起选举。

   > slave 的 SLAVE_RANK 是一个与 master 复制数有关的值，具有最新复制时 SLAVE_RANK 值为 0，第二则为 1，以此类推。这样可让具有最全数据的 slave 优先发起选举。当具有更高 SLAVE_RANK 值的 slave 如果没有当选，则其它 slaves 会很快发起选举（至少 4 秒后）。

4. 替换节点

   S1 替换节点 3 的过程比较清晰易懂。即首先标记自己为主节点，然后将原来由节点 3 负责的 slots 标记为由自己负责，最后向整个集群广播现在自己是 Master 同时负责旧 Master 所有 slots 的信息。其他节点接收到该信息后会更新自己维护的 S1 的状态并标记 S1 为主节点，将节点 3 负责的 slots 的负责节点设置为 S1 节点。

5. 集群配置更新

   那么最后我们需要解决的是，当 S1 成为了新的 Master 之后，S2 和节点 3 该如何处理？显然并不是篡位之后就杀掉 hh。实际上我们是让 S2 和节点 3 成为新的主节点 S1 的 Slave 节点，去备份 S1 节点的数据。那这个过程是如何进行的呢？这是在各个节点信息更新的时候自动实现的。当节点 3 故障恢复重新上线后，发现原先本该由自己负责的 slot 被 S1 负责了，那么他就知道自己被替代了，会自动成为 S1 节点的子节点，当 S2 节点发现原先应该由其 Master 节点 3 负责的 slot 被 S1 负责了，那么他就知道自己的 Master 被替代了，就会成为 S1 的 Slave 节点。
