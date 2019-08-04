## AbstractQueuedSychronizer分析

### AQS原理

如果被请求的共享资源空闲（单身），则将当前请求资源的线程设置为占有该资源的线程（现男友），并给该共享资源加锁。如果被请求的资源被占用（有男朋友），则需要一套线程阻塞和唤醒时锁分配的机制（选择下一任男友的机制），这个机制AQS通过CLH队列实现，将暂时获取不到锁的线程（备胎）加入到队列中

###### CLH队列

虚拟的双向队列，不存在队列实例，仅存在节点之间的关联关系

### 继承关系

* 绝大部分并发组件基于AbstractQueuedSychronizer实现, 一般是在内部有一个实现类``sync``. 后续统一用``sync``表示某个具体的实现类
* 使用者可以自己实现AQS达到自己同步的功能
* aqs继承了``AbstractOwnableSynchronizer``(aos),  aos只有1个成员变量``exclusiveOwnerThread``, 顾名思义, 用于保存在排他模式下, 当前持有sync对象的线程.

### aqs的结构

1. 内部类``Node``

   链表结构

   Node有一些重要的成员变量

   1. volatile的``waitStatus``: 等待状态, 取值有一些固定值, 分别是:
      * 0: 初始状态
      * CANCELLED(1): 等待的线程已经被取消或中断
      * SIGNAL(-1): 当前线程释放锁后, 后继等待的线程需要被唤醒(unpark)
      * CONDITION(-2): 表明等待的线程正在条件上等待
      * PROPAGATE(-3): 下一个``acquireShared``需要无条件的传播

2. 成员变量

   1. volatile的state: 核心变量, 用于表述当前的sync是否被某线程占用, 以及被同一个线程重入的次数
   2. volatile的head: 等待队列的头结点, lazy初始化, 通过``setHead``方法设定, 头结点的``waitStatus``不能是``CANCELLED``
   3. volatile的tail: 等待队列的尾节点, lazy初始化, 增加新的等待的节点时, 通过``enq``方法设定.

3. 需要覆盖方法

   共有5个有默认实现的方法, 子类需要根据情况覆盖, 这5个方法默认的实现都会抛出``UnsupportedOperationException``

   * tryAcquire
   * tryRelease
   * tryAcquireShared
   * tryReleaseShared
   * isHeldExclusively



#### ReentrantLock

独占锁，fair /nonfair

当一个线程尝试获取锁时，如果锁未被持有，`compareAndSetState(0,1)`返回`true`，则获取成功

否则这个锁已经被持有，检查持有者是否是当前线程，如果是当前线程，重入成功，否则获取失败，入队



###### 公平锁

```java
		/**
     * Sync object for fair locks
     */
    static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;

        final void lock() {
            acquire(1);
        }

        /**
         * Fair version of tryAcquire.  Don't grant access unless
         * recursive call or no waiters or is first.
         */
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                // 1.判断当前线程前面是否有线程在等待
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
    }
```

**当前线程位于队列第一个时**，才会通过`compareAndSetState(0,acquires)`修改同步状态变量

###### 非公平锁

```java
		/**
     * Sync object for non-fair locks
     */
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        /**
         * Performs lock.  Try immediate barge, backing up to normal
         * acquire on failure.
         */
        final void lock() {
            // 2. 直接CAS
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }
				/**
         * Performs non-fair tryLock.  tryAcquire is implemented in
         * subclasses, but both need nonfair try for trylock method.
         */
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                // 3.直接CAS，没有判断当前线程是否是队列第一个
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```

非公平锁的获取过程，进入`lock()`后直接CAS判断锁是否被占用（抢锁），成功则返回，不成功则进入`acquire(1)`，再一次判断锁是否被占用，如果这个时候锁恰巧被释放了，则通过CAS再次抢锁，如过锁依然被占用，则判断占用线程是不是当前线程。获取锁失败，则返回false。

###### Condition

当获取锁的线程需要等待某个条件时(`Condition.await()`)，线程会进入condition queue并挂起。当Condition条件满足时，线程会从condition queue 转移到 sync queue，参与锁的竞争。线程转义过程waitStatus变化过程：Condition => 0 => Signal

#### Semaphore

共享锁

#### CountDownLatch

#### ReentrantReadWriteLock

读锁采用共享锁，写锁采用独占锁



