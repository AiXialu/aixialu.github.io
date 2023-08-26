---
title: redis-lock
date: 2020-06-12 11:32:57
tags:
---

## 分布式锁

在多线程的环境下，为了保证一个代码块在同一时间只能由一个线程访问 --> 分布式锁，它是控制分布式系统之间互斥访问共享资源的一种方式。

<!-- More -->

比如说在一个分布式系统中，多台机器上部署了多个服务，当客户端一个用户发起一个数据插入请求时，如果没有分布式锁机制保证，那么那多台机器上的多个服务可能进行并发插入操作，导致数据重复插入，对于某些不允许有多余数据的业务来说，这就会造成问题。而分布式锁机制就是为了解决类似这类问题，保证多个服务之间互斥的访问共享资源，如果一个服务抢占了分布式锁，其他服务没获取到锁，就不进行后续操作。大致意思如下图所示（不一定准确）：

![](https://user-gold-cdn.xitu.io/2019/4/25/16a53749547937bb?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

分布式锁的特点
分布式锁一般有如下的特点：

- 互斥性： 同一时刻只能有一个线程持有锁
- 可重入性： 同一节点上的同一个线程如果获取了锁之后能够再次获取锁
- 锁超时：和 J.U.C 中的锁一样支持锁超时，防止死锁
- 高性能和高可用： 加锁和解锁需要高效，同时也需要保证高可用，防止分布式锁失效
- 具备阻塞和非阻塞性：能够及时从阻塞状态中被唤醒

分布式锁的实现方式
我们一般实现分布式锁有以下几种方式：

- 基于数据库
- 基于 Redis
- 基于 zookeeper

### Redis Lock:

### 上锁

使用 set key value [EX seconds][PX milliseconds][NX|XX] 命令 (正确做法)
Redis 在 2.6.12 版本开始，为 SET 命令增加一系列选项：

```
SET key value[EX seconds][PX milliseconds][NX|XX]
```

- EX seconds: 设定过期时间，单位为秒
- PX milliseconds: 设定过期时间，单位为毫秒
- NX: 仅当 key 不存在时设置值
- XX: 仅当 key 存在时设置值

value 必须要具有唯一性，我们可以用 UUID 来做，设置随机字符串保证唯一性，至于为什么要保证唯一性？假如 value 不是随机字符串，而是一个固定值，那么就可能存在下面的问题：

1.客户端 1 获取锁成功 2.客户端 1 在某个操作上阻塞了太长时间 3.设置的 key 过期了，锁自动释放了 4.客户端 2 获取到了对应同一个资源的锁 5.客户端 1 从阻塞中恢复过来，因为 value 值一样，所以执行释放锁操作时就会释放掉客户端 2 持有的锁，这样就会造成问题

所以通常来说，在释放锁时，我们需要对 value 进行验证

Set Lock:

Golang:

```
func (c *Client) SetLock(key string, value string) error {
	redisClient, _ := c.GetClient()
	err := redisClient.Set(key, value, 0).Err()
	if err != nil {
		return err
	}
	return nil
}
```

Java: (Jedis)

```
public boolean tryLock_with_set(String key, String UniqueId, int seconds) {
    return "OK".equals(jedis.set(key, UniqueId, "NX", "EX", seconds));
}
```

### 释放锁的实现

释放锁时需要验证 value 值，也就是说我们在获取锁的时候需要设置一个 value，不能直接用 del key 这种粗暴的方式，因为直接 del key 任何客户端都可以进行解锁了，所以解锁时，我们需要判断锁是否是自己的，基于 value 值来判断，

Golang:

```
func (c *Client) Expire(key string, expiration time.Duration) error {
	redisClient, _ := c.GetClient()
	err := redisClient.Expire(key, expiration).Err()
	if err != nil {
		return err
	}
	return nil
}
```

Java:

```
public boolean releaseLock_with_lua(String key,String value) {
    String luaScript = "if redis.call('get',KEYS[1]) == ARGV[1] then " +
            "return redis.call('del',KEYS[1]) else return 0 end";
    return jedis.eval(luaScript, Collections.singletonList(key), Collections.singletonList(value)).equals(1L);
}
```

但是，
使用 set key value [EX seconds][PX milliseconds][NX|XX] 命令 看上去很 OK，实际上在 Redis 集群的时候也会出现问题，比如说 A 客户端在 Redis 的 master 节点上拿到了锁，但是这个加锁的 key 还没有同步到 slave 节点，master 故障，发生故障转移，一个 slave 节点升级为 master 节点，B 客户端也可以获取同个 key 的锁，但客户端 A 也已经拿到锁了，这就导致多个客户端都拿到锁。
所以针对 Redis 集群这种情况，还有其他方案

### Redlock

- 在 Redis 的分布式环境中，我们假设有 N 个 Redis master。这些节点完全互相独立，不存在主从复制或者其他集群协调机制。我们确保将在 N 个实例上使用与在 Redis 单实例下相同方法获取和释放锁。现在我们假设有 5 个 Redis master 节点，同时我们需要在 5 台服务器上面运行这些 Redis 实例，这样保证他们不会同时都宕掉。

为了取到锁，客户端应该执行以下操作:

- 获取当前 Unix 时间，以毫秒为单位。

- 依次尝试从 5 个实例，使用相同的 key 和具有唯一性的 value（例如 UUID）获取锁。当向 Redis 请求获取锁时，客户端应该设置一个网络连接和响应超时时间，这个超时时间应该小于锁的失效时间。例如你的锁自动失效时间为 10 秒，则超时时间应该在 5-50 毫秒之间。这样可以避免服务器端 Redis 已经挂掉的情况下，客户端还在死死地等待响应结果。如果服务器端没有在规定时间内响应，客户端应该尽快尝试去另外一个 Redis 实例请求获取锁。

- 客户端使用当前时间减去开始获取锁时间（步骤 1 记录的时间）就得到获取锁使用的时间。当且仅当从大多数（N/2+1，这里是 3 个节点）的 Redis 节点都取到锁，并且使用的时间小于锁失效时间时，锁才算获取成功。

- 如果取到了锁，key 的真正有效时间等于有效时间减去获取锁所使用的时间（步骤 3 计算的结果）。

- 如果因为某些原因，获取锁失败（没有在至少 N/2+1 个 Redis 实例取到锁或者取锁时间已经超过了有效时间），客户端应该在所有的 Redis 实例上进行解锁（即便某些 Redis 实例根本就没有加锁成功，防止某些节点获取到锁但是客户端没有得到响应而导致接下来的一段时间不能被重新获取锁）

---

## 总结

1. Redis 记得解锁
2. Redis 只解自己的锁
3. Redisson 自动续约
4. RedisLock 解决 Master-slave 的问题

### Redisson 自动续约

![](https://img-blog.csdnimg.cn/img_convert/e4b903c186f40d7137e730ada3ccef25.png)

> in redisson, there is a watchdog, which will renew the lock every 30 seconds.

![](https://img-blog.csdnimg.cn/img_convert/83d66f4af1820e979bf701304c60a36d.png)

![](https://img-blog.csdnimg.cn/img_convert/01dd95e61a9d8d9069bdf85cc6ea415e.png)

所以说 每一个 30-10=20s redisson 将会续 30s 的 lock，这样 lock 将一直在哪里，如果 system 退出了，lock 就不会续，那么在 30s 之后就会解锁。
