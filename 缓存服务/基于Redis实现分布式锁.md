> **文章已收录Github精选，欢迎Star**：[https://github.com/yehongzhi](https://github.com/yehongzhi/learningSummary)

# 前言

如果在一个分布式系统中，我们从数据库中读取一个数据，然后修改保存，这种情况很容易遇到并发问题。因为读取和更新保存不是一个原子操作，在并发时就会导致数据的不正确。这种场景其实并不少见，比如电商秒杀活动，库存数量的更新就会遇到。如果是单机应用，直接使用本地锁就可以避免。如果是分布式应用，本地锁派不上用场，这时就需要引入分布式锁来解决。

由此可见分布式锁的目的其实很简单，就是**为了保证多台服务器在执行某一段代码时保证只有一台服务器执行**。

为了保证分布式锁的可用性，至少要确保锁的实现要同时满足以下几点：

- 互斥性。在任何时刻，保证只有一个客户端持有锁。
- 不能出现死锁。如果在一个客户端持有锁的期间，这个客户端崩溃了，也要保证后续的其他客户端可以上锁。
- 保证上锁和解锁都是同一个客户端。

一般来说，实现分布式锁的方式有以下几种：

- 使用MySQL，基于唯一索引。
- 使用ZooKeeper，基于临时有序节点。
- **使用Redis，基于setnx命令**。

本篇文章主要讲解Redis的实现方式。

# 实现思路

Redis实现分布式锁主要利用Redis的`setnx`命令。`setnx`是`SET if not exists`(如果不存在，则 SET)的简写。

```shell
127.0.0.1:6379> setnx lock value1 #在键lock不存在的情况下，将键key的值设置为value1
(integer) 1
127.0.0.1:6379> setnx lock value2 #试图覆盖lock的值，返回0表示失败
(integer) 0
127.0.0.1:6379> get lock #获取lock的值，验证没有被覆盖
"value1"
127.0.0.1:6379> del lock #删除lock的值，删除成功
(integer) 1
127.0.0.1:6379> setnx lock value2 #再使用setnx命令设置，返回0表示成功
(integer) 1
127.0.0.1:6379> get lock #获取lock的值，验证设置成功
"value2"
```

上面这几个命令就是最基本的用来完成分布式锁的命令。

加锁：使用`setnx key value`命令，如果key不存在，设置value(加锁成功)。如果已经存在lock(也就是有客户端持有锁了)，则设置失败(加锁失败)。

解锁：使用`del`命令，通过删除键值释放锁。释放锁之后，其他客户端可以通过`setnx`命令进行加锁。

key的值可以根据业务设置，比如是用户中心使用的，可以命令为`USER_REDIS_LOCK`，value可以使用uuid保证唯一，用于标识加锁的客户端。保证加锁和解锁都是同一个客户端。

那么接下来就可以写一段很简单的加锁代码：

```java
private static Jedis jedis = new Jedis("127.0.0.1");

private static final Long SUCCESS = 1L;

/**
  * 加锁
  */
public boolean tryLock(String key, String requestId) {
    //使用setnx命令。
    //不存在则保存返回1，加锁成功。如果已经存在则返回0，加锁失败。
    return SUCCESS.equals(jedis.setnx(key, requestId));
}

//删除key的lua脚本，先比较requestId是否相等，相等则删除
private static final String DEL_SCRIPT = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";

/**
  * 解锁
  */
public boolean unLock(String key, String requestId) {
    //删除成功表示解锁成功
    Long result = (Long) jedis.eval(DEL_SCRIPT, Collections.singletonList(key), Collections.singletonList(requestId));
    return SUCCESS.equals(result);
}
```

![](https://static.lovebilibili.com/redis_lock_1.png)

## 问题一

这仅仅满足上述的第一个条件和第三个条件，保证上锁和解锁都是同一个客户端，也保证只有一个客户端持有锁。

但是第二点没法保证，因为如果一个客户端持有锁的期间突然崩溃了，就会导致无法解锁，最后导致出现死锁的现象。

![](https://static.lovebilibili.com/redis_lock_2.png)

所以要有个超时的机制，在设置key的值时，需要加上有效时间，如果有效时间过期了，就会自动失效，就不会出现死锁。然后加锁的代码就会变成这样。

```java
public boolean tryLock(String key, String requestId, int expireTime) {
    //使用jedis的api，保证原子性
    //NX 不存在则操作 EX 设置有效期，单位是秒
    String result = jedis.set(key, requestId, "NX", "EX", expireTime);
    //返回OK则表示加锁成功
    return "OK".equals(result);
}
```

![](https://static.lovebilibili.com/redis_lock_3.png)

但是聪明的同学肯定会问，有效时间设置多长，假如我的业务操作比有效时间长，我的业务代码还没执行完就自动给我解锁了，不就完蛋了吗。

这个问题就有点棘手了，在网上也有很多讨论，第一种解决方法就是靠程序员自己去把握，预估一下业务代码需要执行的时间，然后设置有效期时间比执行时间长一些，保证不会因为自动解锁影响到客户端业务代码的执行。

但是这并不是万全之策，比如网络抖动这种情况是无法预测的，也有可能导致业务代码执行的时间变长，所以并不安全。

有一种方法比较靠谱一点，就是给锁续期。在Redisson框架实现分布式锁的思路，就使用watchDog机制实现锁的续期。当加锁成功后，同时开启守护线程，默认有效期是30秒，每隔10秒就会给锁续期到30秒，只要持有锁的客户端没有宕机，就能保证一直持有锁，直到业务代码执行完毕由客户端自己解锁，如果宕机了自然就在有效期失效后自动解锁。

![](https://static.lovebilibili.com/redis_lock_4.png)

## 问题二

但是聪明的同学可能又会问，你这个锁只能加一次，不可重入。可重入锁意思是在外层使用锁之后，内层仍然可以使用，那么可重入锁的实现思路又是怎么样的呢？

在Redisson实现可重入锁的思路，使用Redis的哈希表存储可重入次数，当加锁成功后，使用`hset`命令，value(重入次数)则是1。

```java
"if (redis.call('exists', KEYS[1]) == 0) then " +
"redis.call('hset', KEYS[1], ARGV[2], 1); " +
"redis.call('pexpire', KEYS[1], ARGV[1]); " +
"return nil; " +
"end; "
```

如果同一个客户端再次加锁成功，则使用`hincrby`自增加一。

```java
"if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
"redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
"redis.call('pexpire', KEYS[1], ARGV[1]); " +
"return nil; " +
"end; " +
"return redis.call('pttl', KEYS[1]);"
```

![](https://static.lovebilibili.com/redis_lock_6.png)

解锁时，先判断可重复次数是否大于0，大于0则减一，否则删除键值，释放锁资源。

```java
protected RFuture<Boolean> unlockInnerAsync(long threadId) {
    return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
"if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then " +
"return nil;" +
"end; " +
"local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); " +
"if (counter > 0) then " +
"redis.call('pexpire', KEYS[1], ARGV[2]); " +
"return 0; " +
"else " +
"redis.call('del', KEYS[1]); " +
"redis.call('publish', KEYS[2], ARGV[1]); " +
"return 1; "+
"end; " +
"return nil;",
Arrays.<Object>asList(getName(), getChannelName()), LockPubSub.UNLOCK_MESSAGE, internalLockLeaseTime, getLockName(threadId));
}
```

![](https://static.lovebilibili.com/redis_lock_7.png)

为了保证操作原子性，加锁和解锁操作都是使用lua脚本执行。

## 问题三

上面的加锁方法是加锁后立即返回加锁结果，如果加锁失败的情况下，总不可能一直轮询尝试加锁，直到加锁成功为止，这样太过耗费性能。所以需要利用发布订阅的机制进行优化。

步骤如下：

当加锁失败后，订阅锁释放的消息，自身进入阻塞状态。

当持有锁的客户端释放锁的时候，发布锁释放的消息。

当进入阻塞等待的其他客户端收到锁释放的消息后，解除阻塞等待状态，再次尝试加锁。

![](https://static.lovebilibili.com/redis_lock_5.png)

# 总结

以上的实现思路仅仅考虑在单机版Redis上，如果是集群版Redis需要考虑的问题还要再多一点。Redis由于他的高性能读写能力，所以在并发高的场景下使用Redis分布式锁会多一点。

问题一，二，三其实就是redis分布式锁不断改良发展的过程，第一个问题是设置有效期防止死锁，并且引入守护线程给锁续期，第二个问题是支持可重入锁，第三个问题是加锁失败后阻塞等待，等锁释放后再次尝试加锁。Redisson框架解决这三个问题的思路也非常值得学习。

这篇文章就写到这里了，非常感谢大家的阅读，希望看完之后能得到一些启发和收获。

**觉得有用就点个赞吧，你的点赞是我创作的最大动力**~

**我是一个努力让大家记住的程序员。我们下期再见！！！**

![img](https://static.lovebilibili.com/dashacha.png)

> 能力有限，如果有什么错误或者不当之处，请大家批评指正，一起学习交流！