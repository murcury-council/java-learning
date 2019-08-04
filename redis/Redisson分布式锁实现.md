# Redisson分布式锁实现

RedissonLock: Implements a non-fair locking

RedissonFairLock: Implements a fair locking

### RLock#tryLock(waitTIme,leaseTime,unit)

>Returns <code>true</code> as soon as the lock is acquired.
>
>If the lock is currently held by another thread in this or any other process in the distributed system this method keeps trying to acquire the lock for up to <code>waitTime</code> before giving up and returning <code>false</code>. 
>
>If the lock is acquired, it is held until <code>unlock</code> is invoked, or until <code>leaseTime</code> have passed since the lock was granted - whichever comes first.
##### RedissonLock#tryLock

使用当前线程id，尝试获取锁`tryAcquire(leaseTime,timeUnit,threadId)`,返回**过期时间**

* 没有过期时间，未上锁，获取成功
* 如果当前时间已超时，获取失败
* 锁被其他线程持有，则当前线程通过Redis的**PUBSUB**订阅Channel监听队列(每把锁都有一个channel)，通过CountDownLatch阻塞**等待订阅结果**
  * 若超时
    * future无法被取消，则添加listener回调，在future完成后取消unsubscribe
    * 获取失败acquireFailed
* 循环获取锁，同时检测是否过期，未过期则通过Semaphore(0)，挂起，等待释放锁的通知

##### RedissonLock#tryAcquire

###### RedissonLock#tryLockInnerAsync

执行一段Lua脚本，Redis将整个脚本作为一个整体执行，不会被其他命令插入

*lua脚本内容：*

```java
<T> RFuture<T> tryLockInnerAsync(long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {
        internalLockLeaseTime = unit.toMillis(leaseTime);

        return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, command,
                  "if (redis.call('exists', KEYS[1]) == 0) then " +
                      "redis.call('hset', KEYS[1], ARGV[2], 1); " +
                      "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                      "return nil; " +
                  "end; " +
                  "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                      "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
                      "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                      "return nil; " +
                  "end; " +
                  "return redis.call('pttl', KEYS[1]);",
                    Collections.<Object>singletonList(getName()), internalLockLeaseTime, getLockName(threadId));
    }
```



检查锁名称是否存在

* 不存在，设置线程级别锁名称，设置过期时间
* 锁名称和线程级别锁都存在，则自增1，记录重入的次数，更新过期时间
* 锁名称存在，持有锁的线程不是当前线程，则返回锁的剩余过期时间



### RedissonLock#unlock



##### RedissonLock#unlockInnerAsync

```java
protected RFuture<Boolean> unlockInnerAsync(long threadId) {
        return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
                // key不存在，说明锁已释放，执行publish发布释放锁的消息
                "if (redis.call('exists', KEYS[1]) == 0) then " +
                    "redis.call('publish', KEYS[2], ARGV[1]); " +
                    "return 1; " +
                "end;" +
                // 锁存在，但是field不存在，说明持有锁的线程不是当前线程，无权释放，返回nil
                "if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then " +
                    "return nil;" +
                "end; " +
                // 对锁减一
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
                Arrays.<Object>asList(getName(), getChannelName()), LockPubSub.unlockMessage, internalLockLeaseTime, getLockName(threadId));

    }
```



释放锁时通过`publish`进行通知，发布`LockPubSub.unlockMessage`的消息

###### LockPubSub

```java
protected void onMessage(RedissonLockEntry value, Long message) {
        if (message.equals(unlockMessage)) {
            Runnable runnableToExecute = value.getListeners().poll();
            if (runnableToExecute != null) {
                runnableToExecute.run();
            }
						// Semaphore state + 1
            value.getLatch().release();
        } else if (message.equals(readUnlockMessage)) {
            while (true) {
                Runnable runnableToExecute = value.getListeners().poll();
                if (runnableToExecute == null) {
                    break;
                }
                runnableToExecute.run();
            }

            value.getLatch().release(value.getLatch().getQueueLength());
        }
    }
```

当收到`LockPubSub.unlockMessage`消息时，通过释放`Semaphore`资源，阻塞的线程就可以被唤醒了