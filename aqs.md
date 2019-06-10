## AbstractQueuedSychronizer分析

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
      * 1: 等待的线程已经被取消或中断
      * -1: 当前线程释放锁后, 后继等待的线程需要被唤醒(unpark)
      * -2: 表明等待的线程正在条件上等待
      * -3: 下一个``acquireShared``需要无条件的传播

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





