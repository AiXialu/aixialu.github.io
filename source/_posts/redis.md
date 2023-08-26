---
title: redis
date: 2020-05-27 17:00:12
tags:
disqusId: aitudou-blog
---

### Redis 数据库（非关系型）的几个框架

---

#### Redis 支持字符串、哈希表、列表、集合、有序集合五种数据类型的存储

#### 字符串（string）--> 会采用预分配冗余空间的方式来减少内存的频繁分配

<!-- More -->

![Redis string](https://www.ibm.com/developerworks/cn/java/know-redis-and-use-it-in-springboot-projects/image001.png)

> 常用命令: set,get,decr,incr,mget 等。

#### 列表（list）--> 链表和 ziplist(在列表元素较少的情况下会使用一块连续的内存存储) 结合起来组成了 quicklist (双向列表) 就是说 Redis 利用连续内存存一部分数据 然后利用双向指针 存上下内存的地址

![Redis list](https://www.ibm.com/developerworks/cn/java/know-redis-and-use-it-in-springboot-projects/image002.png)

> 常用命令: lpush,rpush,lpop,rpop,lrange 等

#### 哈希表（hash）--> 数组 + LinkList 组成的

![Redis Hash](https://pic3.zhimg.com/80/v2-c9c3fc05a707ae1bb4cb46c8d29aaa7a_720w.jpg)

数组是用来确定桶的位置，利用元素的 key 的 hash 值对数组长度取模得到.

链表是用来解决 hash 冲突问题，当出现 hash 值一样的情形，就在数组上的对应位置形成一条链表。

hash 冲突 -->

- 开放定址法 (找到另一个 可以用的地址)
- 链地址法 (就是上面用链表的原因)
- 再哈希法
- 公共溢出区域法

https://www.jianshu.com/p/379680144004

https://zhuanlan.zhihu.com/p/79505299

> 常用命令： hget,hset,hgetall 等。

#### 集合（set）--> HashSet 的内部实现使用的是 HashMap，只不过所有的 value 都指向同一个对象。Redis 的 Set 结构也是一样，它的内部也使用 Hash 结构，所有的 value 都指向同一个内部值。

> 常用命令： sadd,spop,smembers,sunion 等

#### 有序集合（sorted set）

有时也被称作 ZSet，是 Redis 中一个比较特别的数据结构，在有序集合中我们会给每个元素赋予一个权重，其内部元素会按照权重进行排序，我们可以通过命令查询某个范围权重内的元素，这个特性在我们做一个排行榜的功能时可以说非常实用了。其底层的实现使用了两个数据结构， hash 和跳跃列表，hash 的作用就是关联元素 value 和权重 score，保障元素 value 的唯一性，可以通过元素 value 找到相应的 score 值。跳跃列表的目的在于给元素 value 排序，根据 score 的范围获取元素列表。

> 常用命令： zadd,zrange,zrem,zcard 等

---

### Spring Boot 集成：

https://www.ibm.com/developerworks/cn/java/know-redis-and-use-it-in-springboot-projects/index.html

---

## 经典问题：

1. 缓存与数据库一致性问题

对于既有数据库操作又有缓存操作的接口，一般分为两种执行顺序。

- 先操作数据库，再操作缓存。这种情况下如果数据库操作成功，缓存操作失败就会导致缓存和数据库不一致。
- 第二种情况就是先操作缓存再操作数据库，这种情况下如果缓存操作成功，数据库操作失败也会导致数据库和缓存不一致。

大部分情况下，我们的缓存理论上都是需要可以从数据库恢复出来的，所以基本上采取第一种顺序都是不会有问题的。针对那些必须保证数据库和缓存一致的情况，通常是不建议使用缓存的，如果必须使用的话，可以参考这篇文章来解决这个缓解缓存与数据库的一致性问题。

2. 缓存击穿问题 hotspot invalidation

缓存击穿表示恶意用户频繁的模拟请求缓存中不存在的数据，以致这些请求短时间内直接落在了数据库上，导致数据库性能急剧下降，最终影响服务整体的性能。这个在实际项目很容易遇到，如抢购活动、秒杀活动的接口 API 被大量的恶意用户刷，导致短时间内数据库宕机。对于缓存击穿的问题，有以下几种解决方案，这里只做简要说明。

- 使用互斥锁排队。当从缓存中获取数据失败时，给当前接口加上锁，从数据库中加载完数据并写入后再释放锁。若其它线程获取锁失败，则等待一段时间后重试。

- 最简单：即使是空，也去存 redis

- 使用布隆过滤器。将所有可能存在的数据缓存放到布隆过滤器中，当黑客访问不存在的缓存时迅速返回避免缓存及 DB 挂掉。

https://zhuanlan.zhihu.com/p/43263751

3. 缓存雪崩 cache avalanche

redis 在同一时间一起过期。

4. 缓存穿透 cache penetration

缓存 hotkey 过期

3 和 4 的解决方法：

- lock： 同一时间只能一个 request
- 均匀过期时间
- 读写分离

> 读数据的时候需要检查数据是不是快要过期，如果快过期了，通过 MQ 让另一个系统更新 redis

3. 缓存并发问题

这里的并发指的是多个 Redis 的客户端同时 set 值引起的并发问题。比较有效的解决方案就是把 set 操作放在队列中使其串行化，必须得一个一个执行。

---

## 总结：

1. Overall

![](https://user-gold-cdn.xitu.io/2018/4/18/162d7773080d4570?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

2. 清缓存

- volatile-lru：从已设置过期时间的数据集（server.db[i].expires）中挑选最近最少使用的数据淘汰
- volatile-ttl：从已设置过期时间的数据集（server.db[i].expires）中挑选将要过期的数据淘汰
- volatile-random：从已设置过期时间的数据集（server.db[i].expires）中任意选择数据淘汰
- allkeys-lru：从数据集（server.db[i].dict）中挑选最近最少使用的数据淘汰
- allkeys-random：从数据集（server.db[i].dict）中任意选择数据淘汰
- no-enviction（驱逐）：禁止驱逐数据

3. Redis 的并发竞争问题如何解决 -- 上锁

4. Redis 回收进程如何工作的 -- LRU/LFU/FIFO
5. Redis 大量数据插入 -- Pipelining
6. Redis 持久化数据和缓存怎么做扩容 -- RDB + AOF

7. Redis 常见性能问题和解决方案:

- Master 最好不要做任何持久化工作，如 RDB 内存快照和 AOF 日志文件
- 如果数据比较重要，某个 Slave 开启 AOF 备份数据，策略设置为每秒同步一次
- 为了主从复制的速度和连接的稳定性，Master 和 Slave 最好在同一个局域网内
- 尽量避免在压力很大的主库上增加从库

8. Redis 尽量不用 pubsub 用 mq 工具
9. Redis 三大问题：

   - 缓存穿透： 是指查询一个一定不存在的数据，将导致这个不存在的数据每次请求都要去查数据库，造成缓存穿透。

     - 将空对象也缓存起来
     - 采用布隆过滤器，将所有可能的数据 hash 到一个足够大的 bitmap 中，一个一定不存在的数据肯定会被布隆过滤器过滤掉，避免查询数据库。

   - 缓存雪崩： 缓存在集中一段时间内失效，发送大量的缓存穿透，都落在数据库上，造成缓存雪崩。

     - 尽量让失效的时间点，不分布在一个时间点上

   - 缓存击穿: 一个 key 非常热点，在不停的扛着大并发，key 在失效的瞬间，大并发就穿破缓存，直接请求数据库。

     - 设置 key 不要过期

10. Redis 为什么这么快：

- 数据读写都在内存中完成。
- 单线程请求处理

  多线程并行对数据读取的确能带来好处，但是同样带来了数据写入时锁的开销以及线程切换的开销。再大量的写入情况下，过多的锁带来的时间消耗比多线程带来的多核利用优势更大。

- I/O 多路复用技术： socket 是否可读写的状态监控交给了操作系统

11. 淘汰策略：不淘汰/随机/LRU/LFU

12. 持久化： RBD/AOF

- RBD: 快照
- AOF: BIN-log 但是只有 update，insert，delete 的 log

13. Redis 注册：

![](https://pic1.zhimg.com/80/v2-34146d4550f3057858ca45de87c1c73c_720w.jpg)
