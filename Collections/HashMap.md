# HashMap

#### HashMap结构

* jdk1.7  哈希冲突时通过链表串起来
* jdk1.8  当大于TREEIFY_THRESHOLD时，将链表转为红黑树



<img src="https://img-blog.csdn.net/20180905105402336?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2NTIwMjM1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70" />



##### *为什么线程不安全？*

1.7 resize()时会链表会死循环

1.8 树化后，resize()时也可能导致数据丢失

##### *为什么容量是2的幂次方？*

为了方便通过位操作**定位元素的索引**

`h = hash(key)`

HashMap里对h进行索引，h存储的位置就是 `h % length`，发现当length是2的幂次方时，

`h & length - 1 == h % length  `



