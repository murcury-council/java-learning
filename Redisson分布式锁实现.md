# Redisson分布式锁实现

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
* 锁被其他线程持有，则当前线程通过Redis的PUBSUB订阅Channel监听队列(每把锁都有一个channel)，然后通过CountDownLatch阻塞
  * 若超时
    * future无法被取消，则添加listener回调，在future完成后取消unsubscribe
    * 获取失败acquireFailed
* 循环获取锁，同时检测是否过期

##### RedissonLock#tryAcquire

###### RedissonLock#tryLockInnerAsync

执行一段Lua脚本，Redis将整个脚本作为一个整体执行，不会被其他命令插入

lua脚本内容：

检查锁名称是否存在

* 不存在，设置线程级别锁名称，设置过期时间
* 锁名称和线程级别锁都存在，则自增1，记录重入的次数，更新过期时间
* 锁名称存在，则返回锁的剩余过期时间

