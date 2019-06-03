
`join()`是`Thread`类的方法
```
public final void join() throws InterruptedException {
        join(0);
}
```
`t.join()`方法的作用：阻塞**调用**此方法的线程，即在什么线程内调用，什么线程阻塞。**阻塞到t线程内的任务执行完**，被阻塞的线程才能继续执行。

*为什么会阻塞？通过什么阻塞的？*
```
public final synchronized void join(long millis)
    throws InterruptedException {
        long base = System.currentTimeMillis();
        long now = 0;

        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (millis == 0) {
            while (isAlive()) {
                wait(0);
            }
        } else {
            while (isAlive()) {
                long delay = millis - now;
                if (delay <= 0) {
                    break;
                }
                wait(delay);
                now = System.currentTimeMillis() - base;
            }
        }
    }
```
该方法是`synchronized`的，说明是`synchronized(this)`。

重点查看这个分支
```
if (millis == 0) {
    while (isAlive()) {
        wait(0); // 需要拿到锁，才能wait
    }
}
```
`wait()`方法是`Object`类中的方法。
调用该方法的条件是调用的线程需要持有对象的锁,释放该对象的锁并等待到其他线程调用`notify()`或`notifyAll()`唤醒阻塞在该对象上的线程，重新获得该对象的锁，并继续执行。下面是`wait()`方法的注释。
```
The current thread must own this object's monitor. The thread
releases ownership of this monitor and waits until another thread notifies threads waiting on this object's monitor to wake up either through a call to the {@code notify} method or the
{@code notifyAll} method. The thread then waits until it can
re-obtain ownership of the monitor and resumes execution.
```
*那么该`synchronized`方法中的`this`是谁？*
`this`就是当前类的对象。如果在主线程内调用了`t.join()`,那么`this`就是`t`,主线程持有了`t`这个对象的锁，调用`join()`时将锁释放，阻塞在`t`线程这个对象上。

*调用`join()`的线程被`wait()`阻塞了，那什么时候，谁调用了`notify()`把它唤醒了？*
jvm源码中有这样一段
```
// 位于/hotspot/src/share/vm/runtime/thread.cpp中
void JavaThread::exit(bool destroy_vm, ExitType exit_type) {
    // ...

    // Notify waiters on thread object. This has to be done after exit() is called
    // on the thread (if the thread is the last thread in a daemon ThreadGroup the
    // group should have the destroyed bit set before waiters are notified).
    // 有一个贼不起眼的一行代码，就是这行
    ensure_join(this);

    // ...
}


static void ensure_join(JavaThread* thread) {
    // We do not need to grap the Threads_lock, since we are operating on ourself.
    Handle threadObj(thread, thread->threadObj());
    assert(threadObj.not_null(), "java thread object must exist");
    ObjectLocker lock(threadObj, thread);
    // Ignore pending exception (ThreadDeath), since we are exiting anyway
    thread->clear_pending_exception();
    // Thread is exiting. So set thread_status field in  java.lang.Thread class to TERMINATED.
    java_lang_Thread::set_thread_status(threadObj(), java_lang_Thread::TERMINATED);
    // Clear the native thread instance - this makes isAlive return false and allows the join()
    // to complete once we've done the notify_all below
    java_lang_Thread::set_thread(threadObj(), NULL);

    // 同志们看到了没，别的不用看，就看这一句
    // thread就是当前线程，是啥？就是刚才说的t线程对象。
    lock.notify_all(thread);
    // Ignore pending exception (ThreadDeath), since we are exiting anyway
    thread->clear_pending_exception();
```
当子线程t执行完毕后，JVM会自动唤醒阻塞在t这个对象上的线程。是一种安全机制。那么现在，主线程就能继续执行下去了。



