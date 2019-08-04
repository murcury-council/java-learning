# synchronized底层实现

> Hotspot虚拟机对象头分为两部分，Mark Word（对象哈希码、GC分代年龄等自身运行时数据）、指向方法区中的对象类型数据的指针，如果是数组对象，还会有额外的部分用于存储数组长度

### 偏斜锁（-XX:+/-UseBiasedLocking）

当锁对象第一次被线程获取的时候，虚拟机通过CAS将线程id记录到对象头的Mark Word中

当有另一个线程尝试获取这个锁的时候，偏向模式宣告结束

### 轻量级锁

进入同步块时，如果对象没有被锁定，则在当前线程的栈帧中创建Lock Record,用于拷贝对象的Mark Word,称为Displaced Mark Word。

然后虚拟机使用CAS尝试将对象的Mark Word更新为指向Lock Record的指针。

* 更新成功则当前线程拥有了该对象的锁
* 更新失败则需要检查对象的Mark Word是否指向了当前线程的Lock Record,如果指向了，依然获取成功，否则膨胀为重量级锁。