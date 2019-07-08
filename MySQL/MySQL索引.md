# InnoDB索引

InnoDB如果没有定义主键索引，会选择一个**唯一**的**非空**索引代替。如果没有这样的索引，会隐式的定义一个主键来作为聚簇索引。

![avator](https://static001.geekbang.org/resource/image/dc/8d/dcda101051f28502bd5c4402b292e38d.png)

* 主键索引的叶子节点存的是整行的数据。在InnoDB中，主键索引也被称为聚簇索引。
* 非主键索引的叶子节点内容是主键的值。在InnoDB中，非主键索引也被称为二级索引。

### 索引维护

InnoDB存储引擎中，聚簇索引是一种数据存储方式，叶子节点是实际的数据行

B+树为了维护索引的有序性，在插入新值的时候需要做必要的维护。以上图为例，如果插入新的行ID值为700，则只需要在R5的记录后面插入一个新的记录。如果新插入的ID为400，就需要挪动后面的数据，空出位置。而更糟的是，如果R5所在的数据页已经满了，则需要申请一个新的数据页，然后将数据挪动部分过去，这个过程称为页分裂。

### 覆盖索引

如果二级索引的信息满足了查询需要，不需要回表，即此次查询只使用了二级索引B+树，则称为覆盖索引

### 联合索引

##### 最左前缀原则

![avator](https://static001.geekbang.org/resource/image/89/70/89f74c631110cfbc83298ef27dcd6370.jpg)

(name,age)联合索引

##### 索引下推

无索引下推

<img src="https://static001.geekbang.org/resource/image/b3/ac/b32aa8b1f75611e0759e52f5915539ac.jpg" width="400"/>

有​索引下推

<img src="https://static001.geekbang.org/resource/image/76/1b/76e385f3df5a694cc4238c7b65acfe1b.jpg" width="400px"/>

`like 'jiang%' and age >10`检索，MySQL5.6版本之前会对匹配的数据进行回表查询。之后的版本会查看age字段的数据，过滤掉age<=10的数据再进行回表查询

`Explain select * from T where city like "%杭州% order by name limit 100"`