## Java内存模型

### 分成5个组成部分

#### 1. 虚拟机栈

线程独享, 连续的空间

组成: 由栈帧组成, 1次方法调用对应1个帧

栈帧里保存的内容:

1. **原始类型**的局部变量
2. 对象引用
3. 返回地址
4. 操作数

综上所述

1. 栈顶=活动的栈帧=当前执行的方法
2. 调用方法=入栈, 方法结束=出栈

过大的栈(调用深度)导致StackOverFlow, 如没有退出条件的递归调用. 通过Xss设定大小.



#### 2. 堆

所有线程共享的一块区域, 存储new出来的**对象**和**数组**

包括Eden, 2个survivor, old



#### 3. 方法区

1. 存放**类的信息**, 包括类的字节码, 类的结构
2.  java8以后的版本不再有字符串常量池, 静态变量

java8以前放在堆空间上, java8以后叫Metaspace, 使用本地内存, 不再占用jvm空间.



#### 4. 程序计数器

线程独享, 如果是jvm方法, 保存当前指令执行的地址, 即线程切换时的记录点. 生命周期和线程相同.



#### 5. 本地方法栈

类似虚拟机栈, 为java执行native方法服务.会抛出Stack Overflow和OutOfMemory



## Java垃圾回收

**程序计数器**, **虚拟机栈**, **本地方法栈**由于其结构确定, 因此方法结束, 内存即回收.

堆空间和方法区因为共享, 因此需要GC算法进行回收

#### 两种对象是否在使用的判断方法:

##### 1. 引用计数: 

过时的方法, 解决不了循环引用

##### 2. 可到达性分析:

如果一个对象到GC Root没有任何引用链相连, 即为不可到达的对象. 也就是说, 如果A, B相互引用, 但是都没有从GC Root指向他们的引用链, 那也一样是不可到达的对象.

可以被使用为GC Root的对象

* 栈帧中的本地变量表引用的对象
* 方法区中类静态属性引用的对象
* 方法区中常量引用的对象

具体流程

1. 根据跟搜索算法, 如果直接可达, 则判定为可达到;
2. 如果不可达, 判断是否需要执行finalize方法;
3. 如果没有覆盖finalize方法或者已经被调用过, 则判定为不可到达;
4. 放入f-queue中, 等待执行finalize方法;
5. 执行后存在一引用指向某对象, 则重新判定为可到达;
6. 若最终仍然没有引用指向, 判定为不可到达;

##### 3. 引用

1. 强: 

   直接赋值的引用, 如 `Object o=new Object();`

2. 软:

   通过SoftReference实现, 发生oom之前会检查, 进行二次回收, 如果还不够空间才会报oom

3. 弱

   通过WeakReference实现, 存活到下一次垃圾回收之前, 不论当前内存是否充足都会回收

4. 幻影

   一个对象是否有虚引用的存在，完全不会对其生存时间构成影响，也无法通过虚引用来取得一个对象实例。为一个对象设置虚引用关联的唯一目的就是希望能在这个对象被收集器回收时收到一个系统通知。

##### 4. 垃圾回收算法

1. 标记-清除

   2个阶段, 标记, 清除. 先标记, 然后统一回收. 2个阶段效率都不高, 同时产生大量空间碎片.

2. 复制

   针对新生代的算法, 将内存区域分成2部分, 每次使用其中的1个部分, 发生回收时, 将存活的对象集中复制到另一块, 然后直接删除刚才使用的完整区域. HotSpot中是以默认8:1:1的比率分配eden和2个survivor. 因为绝大部分eden的对象都是短命的. 不过如果存活的对象超过survivor的大小, 会进老年代.

3. 标记-整理

   类似标记-清除, 只不过标记后不立即清理, 而是**移动存活对象到一端**, 然后整体清理边界外的 所有区域.

   现代虚拟机主要是用"滑动整理法", 不改变对象的相对顺序, 不影响赋值器的空间局部性, 还可以将其和其父节点, 兄弟节点放的更近, 提高空间局部性.

4. 分代

   根据对象的代数, 在生命周期的不同阶段, 使用不同的回收算法, 如新生代使用复制, 老年代使用标记整理.

##### 5. 垃圾回收器

分为新生代和老年代

* 新生代回收器

  * **Serial**

    高效, 串行, StopTheWorld

  * **ParNew**

    Serial的多线程版本, 算法是复制算法, 可以和CMS配合

  * **Parallel Scavenge**

    使用复制算法, 关注点是吞吐量(执行用户代码的时间/总耗时)

* 老年代回收器

  * **Serial Old**

    Serial的老年代版本, 使用标记-整理

  * **Parallel Old**

    Parallel Scavenge的老年代版本, 

  * **CMS**
  
    并发**标记-清除**, 目标是停顿时间最短. 
  
    大致运行步骤
  
    1. 初始标记: StopTheWorld, 标记GC Root能直接关联的对象, 速度快
    2. **并发**标记: 执行GC Root Tracing过程(从GCRoot往下找)
    3. 重新标记: StopTheWorld, 重新标记执行2时候发生变动的对象的标记记录
    4. 清除
  
    -XX:+UseCMSCompactAtFullCollection 默认开启，用于CMS在进行FullGC的时候开启内存碎片的合并整理
  
    -XX:CMSFullGCsBeforeCompaction 默认值0，用于设置执行多少次的不压缩FullGC后，来一次带压缩的
  
  * **G1**
  
    算法: 标记-整理
  
    将新老堆空间划分为固定大小的独立区域(Region), 根据垃圾的堆积程度, 维护一个队列, 每次根据允许的收集时间, 优先回收垃圾最多的区域.
  
    

### 内存屏障

volatile基于内存屏障实现

volatile的2个特性

1. 禁止指令重排序

   一个典型的指令重拍的例子

   ```java
   public class OutofOrderExecution {
     private static int x = 0, y = 0;
     private static int a = 0, b = 0;
   
     public static void main(String[] args) throws InterruptedException {
       Thread t1 = new Thread(new Runnable() {
         public void run() {
           a = 1;
           x = b;
         }
       });
       Thread t2 = new Thread(new Runnable() {
         public void run() {
           b = 1;
           y = a;
         }
       });
     t1.start();
     t2.start();
     t1.join();
     t2.join();
     System.out.println(“(” + x + “,” + y + “)”);
     }
   }
   ```

   看起来结果是(1,0), (1,1), (0,1), 但是也可能是(0,0), 因为**单线程互不影响的程序可乱序**

   

2. 可见性

   可见性的定义常见于各种并发场景中，以多线程为例：当一个线程修改了线程共享变量的值，其它线程能够立即得知这个修改。

##### 内存屏障有2个指令

* load指令

  将内存存储的数据读取到处理器缓存

* store指令

  将处理器缓存中的数据刷入内存

共有4种类的屏障

| 屏障类型   | 指令例子                 | 说明                                                         |
| ---------- | ------------------------ | ------------------------------------------------------------ |
| LoadLoad   | load1, LoadLoad,load2    | load1的数据装载, 先于load2和后面所有的load指令               |
| StoreStore | store1,StoreStore,store2 | store1立即刷入内存的数据, 先于store2和后面所有的store指令    |
| LoadStore  | Load1;LoadStore;Store2   | Load1的数据装载先于Store2及其后所有的存储指令刷新数据到内存的操作 |
| StoreLoad  | Store1;StoreLoad;Load2   | Store1立刻刷新数据到内存的操作先于Load2及其后所有装载装载指令的操作。它会使该屏障之前的所有内存访问指令(存储指令和访问指令)完成之后, 才执行该屏障之后的内存访问指令 |

其中StoreLoad是万能的. 指令是mfence指令, 不过x86没实现, x86只有lfence和sfence

* sfence

  相当于StoreStore, 强制所有在sfence指令之前的store指令，都在该sfence指令执行之前被执行，发送缓存失效信号，并把store buffer中的数据刷出到CPU的L1 Cache中；所有在sfence指令之后的store指令，都在该sfence指令执行之后被执行。即，禁止对sfence指令前后store指令的重排序跨越sfence指令，使所有Store Barrier之前发生的内存更新都是可见的。

* lfence

  lfence指令实现了Load Barrier，相当于LoadLoad Barriers。强制所有在lfence指令之后的load指令，都在该lfence指令执行之后被执行，并且一直等到load buffer被该CPU读完才能执行之后的load指令（发现缓存失效后发起的刷入）。即，禁止对lfence指令前后load指令的重排序跨越lfence指令，配合Store Barrier，使所有Store Barrier之前发生的内存更新，对Load Barrier之后的load操作都是可见的。

  