## Java中AtomicReference操作的ABA问题

问题:

线程T1将atomic操作的值从A变为B, 立即又从B变回成A, 这时候在T2看起来预期值仍是A, 故操作可以成功. 但是流程可能出现问题.

一般情况下用于数值累加问题不大, 但是涉及到交换有状态的对象时会出问题.



AtomicReference中的关键操作:

* `get()`获取当前值
* `compareAndSet(expect, update)`如果ar的值和expect相等(是==比对), 则更新为预期值.

JDK中描述:

<big>`Atomically sets the value to the given updated value if the current value `==` the expected value.`</big>

很显然, 2个操作中间是有空隙的, 某线程很可能在get后让出时间片, 稍后恢复. 及时是在AR中提供的类似`AtomicInteger`的`getAndIncrement()`的`getAndAccumulate()`也不例外:

```java
public final V getAndAccumulate(V x, BinaryOperator<V> accumulatorFunction) {
    V prev, next;
    do {
        prev = get();
        // 这里有空隙, 可能产生ABA问题
        next = accumulatorFunction.apply(prev, x);
    } while (!compareAndSet(prev, next));
    return prev;
}
```



解决的办法是使用AtomicStampedReference(), 因为ASR内部集成了一个Pair类, 类里有一个int作为版本号.

ASR的CAS操作成功条件如下:

```java
// 期待交换的引用和当前引用是同一个
expectedReference == current.reference &&
// 期待交换的的引用的版本和当前的版本相同
expectedStamp == current.stamp &&
(
  // 目标对象的引用和当前对象的引用是一个, 而且版本号还相等
  (newReference == current.reference && newStamp == current.stamp) ||
  // 或者能够成功的直接比较交换Pair.
  casPair(current, Pair.of(newReference, newStamp))
)
```

也就是说, 一系列操作ABA变成了A1, B2, A3

不过ASR并没有`getAndAccumulate()`这样的方法.