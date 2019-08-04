## LockSupport分析

LockSupport(LP)是实现阻塞的一种工具, 很多工具的底层都是使用LP实现的

* #### 回顾线程的阻塞状态

  共3类, 不同的操作会让线程到达不同的状态, 但效果看起来都是阻塞的.

  * WAITING

    通过`Object.wait()`, `Thread.join()`, `LockSupport.park()`到达, 线程会在waitingQueue中等待.

  * BLOCKED

    等待进入sychronize代码块或者等待进入sychronize方法, 线程会在LockPool中等待

  * TIMED_WAITING

    通过`Thead.sleep()`,带时间的`wait()`, 带时间的`parkNanos()`等方法

* LP的基本结构和使用

  1. LP无法被实例化;

  2. 通过UNSAFE的park和unpark实现

     ```java
     public native void park(boolean var1, long var2);
     public native void unpark(Object var1);
     ```
     其中park的boolean参数表示是否绝对时间,  long值表示时间值如果为0表示无限等待.  unpark的参数表示需要唤醒的线程.

  3. 调用park()或park(Object blocker)实现阻塞:

     二者的区别是前者没有blocker, 后者有. 代码如下

     ```java
     public static void park(Object blocker) {
       // 阻塞的是当前线程
       Thread t = Thread.currentThread();
       // 设定blocker
       setBlocker(t, blocker);
       // park当前线程, 到此当前线程将处于waiting状态
       UNSAFE.park(false, 0L);
       // 被唤醒后, 将blocker设置为空
       setBlocker(t, null);
     }
     ```

  4. 调用unpark(Thread thread)唤起阻塞的线程

     ```java
     public static void unpark(Thread thread) {
       if (thread != null) // 线程为不空
         UNSAFE.unpark(thread); // 释放该线程许可
     }
     ```

* 使用LP的好处

  * 不像wait()和notify()必须有严格的顺序. 观察如下代码:

    ```java
    public static void main(String[] args) {
      String parkOn = "ParkOn";
      final Thread mainThread = Thread.currentThread();
      LockSupport.unpark(mainThread);
      LockSupport.park(parkOn);
      System.out.println("exit");
    }
    ```

    虽然先调用了unpark在调用park, 但是程序还是会正常退出. 据JDK描述

    <big>**``Makes available the permit for the given thread, if it was not already available. If the thread was blocked on park then it will unblock. Otherwise, its next call to park is guaranteed not to block. This operation is not guaranteed to have any effect at all if the given thread has not been started.``**</big>

    也就是说, 即使调用了unpark, 如果线程已经start了, 也会让下一次park立即返回.

  * 可以相应中断. 如果调用`t.interrupt()`, 会让park的线程退出waiting状态.