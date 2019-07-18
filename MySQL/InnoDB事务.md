# InnoDB事务

事务的基本属性

* 原子性 Atomicity
* 一致性 Consistency
* 隔离性 Isolation
* 持久性 Durability

隔离级别：读未提交、读提交、可重复读、串行化

两阶段锁协议：InnoDB事务中，**行锁**是在需要的时候才加上，但需要等到事务提交才释放。

<img src="https://static001.geekbang.org/resource/image/82/d6/823acf76e53c0bdba7beab45e72e90d6.png" width=500/>

`begin/start transaction`命令并不是一个事务的起点，在执行到他们之后的**第一个操作InnoDB表的语句，事务才真正启动**，创建**一致性视图**。如果想要马上启动一个事务，可以使用`start transaction with consistent snapshot`命令，立即**创建一致性视图**（可重复读隔离级别下）。

MySQL中有两种视图：

* view 
* consistent read view   InnoDB在实现**MVCC**时用到的一致性读视图，用于支持RC(Read Committed,读提交)和RR(Repeatable Read,可重复读)隔离级别的实现。

### *查询和更新在视图方面的区别*

InnoDB中每个事务在开始的时候会向InnoDB事务系统申请**唯一**的事务ID，**transaction id**，**按顺序严格递增**。

每行数据也都有多个版本，每次事务更新数据的时候会生成一个新的数据版本，并且把transaction id赋值给这个数据版本，记作row trx_id。同时旧的数据版本要保留，并且在新的数据版本中，能够有信息可以直接拿到他。

<img src="https://static001.geekbang.org/resource/image/68/ed/68d08d277a6f7926a41cc5541d3dfced.png" width="500"/>

###### 																															一条记录被多个事务连续更新后的状态

上图中三个虚线箭头，就是**undo log**。V1,V2,V3并不是物理上真实存在，而是每次需要的时候根据当前版本和undo log计算出来的。

####快照

**可重复读隔离级别下，事务在启动的时候就基于整个库生成了快照**

快照的生成，就是根据上边说到的row trx_id以及undo log而生成的

按照可重复读的定义，一个事物启动的时候，能够看到所有已经提交的事务结果，在该事务执行期间其他事务的更新对它不可见。



在实现上，InnoDB为每个事务构造了一个数组，用来保存事务启动瞬间，*当前启动了但**没有提交**的所有transaction id*。数组里transaction id的最小值记为**低水位**，当前**系统里已经创建过的**transaction id的最大值加1记为**高水位**。

这个视图数组和高水位就组成了当前事务的一致性视图。

<img src="https://static001.geekbang.org/resource/image/88/5e/882114aaf55861832b4270d44507695e.png" width="500"/>

如上图，对于当前事务的启动瞬间来说，**一个数据的版本的row trx_id**,有如下几种情况：

1.如果落在绿色部分，表示这个数据的版本是已提交的事务或者是当前事务自己生成的，可见

2.如果落在红色部分，表示这个数据的版本是有将来启动的事务生成的，肯定不可见

3.如果落在黄色部分

* 若row trx_id在数组中，则产生这个数据版本的事务还未提交，不可见
* 若row trx_id不在数组中，表示这个版本是已经提交了的事务生成的，可见



#### 更新逻辑

更新数据时是先读后写，读则只能读当前值，称为“当前读”

即当要更新数据的时候，不能在历史版本上更新，更新时需要通过**当前读**读到最新的值在其结果上更新。

除了update语句外，select语句如果加锁，也是当前读。

```mysql
select k from t where id=1 lock in share mode;  #读锁(共享锁)
select k from t where id=1 for update;  #写锁(排他锁)
```





