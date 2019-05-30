# Java并发技巧

### 使用jstack统计线程信息

使用jstack可以dump出某java进程的线程栈信息, 过滤出县城状态, 排序, 分组计数

`` jstack ${pid} | grep java.lang.Thread.State | awk '{print $2$3$4$5}' | sort | uniq -c``

例如:

```
57 RUNNABLE
 2 TIMED_WAITING(onobjectmonitor)
12 TIMED_WAITING(parking)
 4 TIMED_WAITING(sleeping)
 2 WAITING(onobjectmonitor)
64 WAITING(parking)
```

大量的``WAITING(parking)``表示闲置的线程太多了

在tomcat中, 修改了tomcat的maxThread参数后发现并没有任何变化, 观察具体jstack日志, 发现是某些自己启动的线程太多了, 但是工作有太少, 导致大量的parking(LockSupport.park())

### Java的对象内存布局

使用jol工具可以查看某对象在内存中的布局, 有时候可能需要sudo执行

```java
package com.qyer.test.concurrent;

import org.openjdk.jol.info.ClassLayout;
import org.openjdk.jol.vm.VM;
import sun.misc.Contended;

public class RacingObject {

  @Contended
  private volatile int a0;
  private volatile int a1;
  @Contended
  private volatile long a2;
  private volatile long a3;

  public static void main(String[] args) throws Exception {
    System.out.println(VM.current().details());
    System.out.println(ClassLayout.parseClass(RacingObject.class).toPrintable());
  }
}
```

执行:

```bash
/usr/java/latest/bin/java \
-cp .:/home/zijing.wu.R/test/lib/jol-core-0.9.jar \
com.qyer.test.concurrent.RacingObject
```

效果:

```
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 0x0000000800000000 base address and 0-bit shift.
# Objects are 8 bytes aligned.
# Field sizes by type: 4, 1, 1, 2, 2, 4, 4, 8, 8 [bytes]
# Array element sizes: 4, 1, 1, 2, 2, 4, 4, 8, 8 [bytes]

com.qyer.test.concurrent.RacingObject object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0    12        (object header)                           N/A
     12     4    int RacingObject.a0                           N/A
     16     8   long RacingObject.a2                           N/A
     24     8   long RacingObject.a3                           N/A
     32     4    int RacingObject.a1                           N/A
     36     4        (loss due to the next object alignment)
Instance size: 40 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

### Java的FalseShare伪共享问题

在对称多处理器 (SMP) 系统中，每个处理器均有一个本地高速缓存。 内存系统必须保证高速缓存的一致性。 当不同处理器上的线程修改驻留在同一高速`缓存行`中的变量时就会发生假共享， 结果导致高速缓存行无效，并强制执行更新，进而影响系统性能。

上述`缓存行`, 指的是CPU操作数据时, 从主存里总是取出一定大小的数据(哪怕只要1个byte). 当代64bit处理器一般取出64 bytes. 因此如果某个class的内存布局不合理(如逻辑上相关但是总是频繁修改的field, 如队列的head和tail引用), 结合上述jol布局查看工具, 很可能会总是压入一个cache-line读取, 导致cache-line失效, 速度降低.

解决办法:
在早期的JDK里, 需要人工在某一个field前后手工制造一些完全没用的field, 叫做padding. 在JDK1.8里, 只需要将一个field标记为注解
`@sun.misc.Contended`即可. 不过需要在启动java进程的时候加上
`-XX:-RestrictContended`参数才能使其生效
上面的RacingObject经过上述操作后, 观察对象内存布局:

```bash
/usr/java/latest/bin/java \
-XX:-RestrictContended
-cp .:/home/zijing.wu.R/test/lib/jol-core-0.9.jar \
com.qyer.test.concurrent.RacingObject
```

效果:

```
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 0x0000000800000000 base address and 0-bit shift.
# Objects are 8 bytes aligned.
# Field sizes by type: 4, 1, 1, 2, 2, 4, 4, 8, 8 [bytes]
# Array element sizes: 4, 1, 1, 2, 2, 4, 4, 8, 8 [bytes]

com.qyer.test.concurrent.RacingObject object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0    12        (object header)                           N/A
     12     4    int RacingObject.a1                           N/A
     16     8   long RacingObject.a3                           N/A
     24   128        (alignment/padding gap)                  
    152     4    int RacingObject.a0                           N/A
    156   132        (alignment/padding gap)                  
    288     8   long RacingObject.a2                           N/A
Instance size: 296 bytes
Space losses: 260 bytes internal + 0 bytes external = 260 bytes total
```

可见, 首先field的顺序经过调整, 带有Contentded的field会移动到最后, 同时会在field前增加足够多(128bytes)的padding, 让某field独享缓存行

多说一句, 这里的
`Objects are 8 bytes aligned.`
指的是地址对齐, 64bits的机器上采用8字节对齐, 可以加快cpu寻址速度.



### Java的Synchronize锁的晋升

1. 偏向锁

   简单来说, 只需测试锁对象的MarkWord(对象头前4个字)存储的线程id是否是当前线程, 以及偏向锁标记是否是偏向锁, 这样同一个线程获取锁的代价就降低了. 发生条件:

   * 同步代码块
   * 实质无竞争, 即总是1个线程获取锁

2. 轻量锁

   竞争出现时, 先尝试自旋(不断尝试获取锁, java8里自旋次数和时间不再由用户控制, `自适应自旋`), 不阻塞线程.

3. 重量锁

   由轻量锁升级, 自旋失败, 调用系统挂起线程, 等待唤醒

   