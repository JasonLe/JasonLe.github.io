---
title: MySQL——MVCC原理讲解&当前读、快照读
subtitle: MySQL——MVCC原理讲解&当前读、快照读
image: https://img-blog.csdnimg.cn/20210124225711901.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTM1MjM3NzU=
alt: MySQL
date: 2021-01-24 16:58:04

caption:
  title: MySQL——MVCC原理讲解&当前读、快照读
  subtitle: MySQL——MVCC原理讲解&当前读、快照读
  thumbnail: https://wimg.ruan8.com/uploadimg/image/20190402/20190402163328_31815.jpg
---

# MVCC

## MVCC的作用
可重复读隔离级别的时候，通过MVCC解决幻读问题
只在可重复读和读已提交两个隔离级别下工作
因为读未提交总是读取最新的数据，而不是读取当前事务版本的数据行，而可串行化则会对所有读取的行加锁


## MVCC的基本原理
MVCC的实现，通过保存数据在某个时间点的快照来实现的。也就是一个版本链，相当于保存了事务操作的一个历史纪录。

<br><br>

# 版本链
对于使用InnoDB存储引擎的表，其聚簇索引记录中包含了两个重要的隐藏列：

+ trx_id：每当事务对聚簇索引中的记录进行修改时，都会把当前事务的事务id记录到trx_id中。
+ roll_pointer：每当事务对聚簇索引中的记录进行修改时，都会把该记录的旧版本记录到undo日志中，通过roll_pointer这个指针可以用来获取该记录旧版本的信息。

如果在一个事务中多次对记录进行修改，则每次修改都会生成undo日志，并且这些undo日志通过roll_pointer指针串联成一个版本链，版本链的头结点是该记录最新的值，尾结点是事务开始时的初始值。

例如，我们在表book中做以下修改：
```sql
BEGIN;  // 开启Transaction 10

UPDATE book SET stock = 200 WHERE id = 1;

UPDATE book SET stock = 300 WHERE id = 1;
```
![id=1的记录此时的版本链](https://img-blog.csdnimg.cn/20210124224617170.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTM1MjM3NzU=,size_16,color_FFFFFF,t_70#pic_center)

<br><br>

# ReadView
针对**可重复读和读已提交**两种隔离级别提出，因为这两种隔离级别都需要读取已经提交的事务所修改的数据，所以需要确定在**可重复读和读已提交**两种隔离级别下，版本链中哪个版本是能被当前事务读取的。于是ReadView的概念被提出以解决这个问题。

>首先我们需要知道的一个事实是：事务id是递增分配的。ReadView的机制就是在生成ReadView时确定了以下几种信息：

+ m_ids：表示在生成ReadView时当前系统中活跃的读写事务的事务id列表。
+ min_trx_id：表示在生成ReadView时当前系统中活跃的读写事务中最小的事务id，也就是m_ids中的最小值。
+ max_trx_id：表示生成ReadView时系统中将要分配给下一个事务的id值。
+ creator_trx_id：表示生成该ReadView的事务的事务id。

>这样事务id就可以分成3个区间：

+ 区间(0, min_trx_id)：事务id在这个范围内的事务在生成此ReadView时已经提交，因此这些事务修改的版本记录都是被当前事务可以读取的；
+ 区间[min_trx_id, max_trx_id): 事务id在这个范围内的事务可能是活跃的，也有可能是已经提交的，而事务id存在于m_ids中的事务都是活跃事务，否则就是已提交事务。
+ 区间[max_trx_id, +∞)：事务id在这个范围内的事务都是在生成ReadView之后创建的。

>下面我们根据ReadView提供的条件信息，顺着版本链从头结点开始查找最新的可被读取的版本记录：

1. 首先判断版本记录的trx_id与ReadView中的creator_trx_id是否相等。如果相等，那就说明该版本的记录是在当前事务中生成的，自然也就能够被当前事务读取；否则进行第2步。

2. 根据版本记录的trx_id以及上述3个区间信息，判断生成该版本记录的事务是否是已提交事务，进而确定该版本记录是否可被当前事务读取。

如果某个版本记录经过以上步骤判断确定其可被当前事务读取，则查询结果返回此版本记录；否则读取下一个版本记录继续按照上述步骤进行判断，直到版本链的尾结点。如果遍历完版本链没有找到可读取的版本，则说明该记录对当前事务不可见，查询结果为空。

>在MySQL中，**可重复读和读已提交**两种隔离级别下的区别就是它们生成ReadView的时机不同。

读已提交：事务每次读取数据就生成一个ReadView
可重复读：事务首次读取数据时候生成一个ReadView，之后就一直沿用着一个


# 读已提交(Read Committed)隔离级别下的MVCC工作原理
假设在读已提交隔离级别下，有如下事务在执行，事务id为10：
```sql
BEGIN; // 开启Transaction 10

UPDATE book SET stock = 200 WHERE id = 2;

UPDATE book SET stock = 300 WHERE id = 2;
```

此时该事务尚未提交，id为2的记录版本链如下图所示：
![id为2的记录版本链](https://img-blog.csdnimg.cn/20210124224759482.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTM1MjM3NzU=,size_16,color_FFFFFF,t_70#pic_center)


然后我们开启一个事务Transaction 11对id为2的记录进行查询：
```sql
BEGIN; // 开启Transaction 11
SELECT * FROM book WHERE id = 2; 
```
当执行SELECT语句时会生成一个ReadView，该ReadView中的m_ids为[10]，min_trx_id为10，max_trx_id为11，creator_trx_id为0（因为事务中当执行写操作时才会分配一个单独的事务id，否则事务id为0）。按照我们之前所述ReadView的工作原理，当前数据如下：

```
+----------+-----------+-------+
| book_id  | book_name | stock |
+----------+-----------+-------+
| 2        | Java编程  |  100  |
+----------+-----------+-------+
```
然后我们将事务id为10的事务提交：

```sql
BEGIN; // 开启Transaction 10

UPDATE book SET stock = 200 WHERE id = 2;

UPDATE book SET stock = 300 WHERE id = 2;

COMMIT;
```

同时开启执行另一事务id为11的事务，但不提交：
```sql
BEGIN; // 开启Transaction 11

UPDATE book SET stock = 400 WHERE id = 2;
```
此时id为2的记录版本链如下图所示：

![id为2_2的记录版本链](https://img-blog.csdnimg.cn/2021012422483266.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTM1MjM3NzU=,size_16,color_FFFFFF,t_70#pic_center)

然后我们回到刚才的查询事务中再次查询id为2的记录：

```sql
BEGIN;

SELECT * FROM book WHERE id = 2; // 此时Transaction 10 未提交

SELECT * FROM book WHERE id = 2; // 此时Transaction 10 已提交
```

当第二次执行SELECT语句时会再次生成一个ReadView，该ReadView中的m_ids为[11]，min_trx_id为11，max_trx_id为12，creator_trx_id为0。按照ReadView的工作原理进行分析，我们查询到的版本记录为，**因为Transaction 11还未提交，所以并不会读到最新记录**，即我们查询到的版本记录为：
```
+----------+-----------+-------+
| book_id  | book_name | stock |
+----------+-----------+-------+
| 2        | Java编程  |  300  |
+----------+-----------+-------+
```
从上述分析可以发现，因为每次执行查询语句都会生成新的ReadView，所以在**读已提交**隔离级别下的事务读取到的是版本链中已提交事务修改之后的数据。

# 可重复读(Repeatable Read)隔离级别下MVCC工作原理

我们在Repeatable Read隔离级别下重复上面的事务操作：
```sql
BEGIN; // 开启Transaction 20

UPDATE book SET stock = 200 WHERE id = 2;

UPDATE book SET stock = 300 WHERE id = 2;
```
此时该事务尚未提交，然后我们开启一个查询事务对id为2的记录进行查询：
```sql
BEGIN;

SELECT * FROM book WHERE id = 2;
```
当事务第一次执行SELECT语句时会生成一个ReadView，该ReadView中的m_ids为[20]，min_trx_id为20，max_trx_id为21，creator_trx_id为0。根据ReadView的工作原理，我们查询到的版本记录为

```
+----------+-----------+-------+
| book_id  | book_name | stock |
+----------+-----------+-------+
| 2        | Java编程  |  100  |
+----------+-----------+-------+
```

然后我们将事务id为20的事务提交：
```sql
BEGIN; // 开启Transaction 20

UPDATE book SET stock = 200 WHERE id = 2;

UPDATE book SET stock = 300 WHERE id = 2;

COMMIT;
```
同时开启执行另一事务id为21的事务，但不提交：
```sql
BEGIN; // 开启Transaction 21

UPDATE book SET stock = 400 WHERE id = 2;
```

然后我们回到刚才的查询事务中再次查询id为2的记录：
```sql
BEGIN;

SELECT * FROM book WHERE id = 2; // 此时Transaction 10 未提交

SELECT * FROM book WHERE id = 2; // 此时Transaction 10 已提交
```

当第二次执行SELECT语句时不会生成新的ReadView，依然会使用第一次查询时生成ReadView。因此我们查询到的版本记录跟第一次查询到的结果是一样的：
```
+----------+-----------+-------+
| book_id  | book_name | stock |
+----------+-----------+-------+
| 2        | Java编程  |  100  |
+----------+-----------+-------+
```
从上述分析可以发现，因为在Repeatable Read隔离级别下的事务只会在第一次执行查询时生成ReadView，该事务中后续的查询操作都会沿用这个ReadView，因此此隔离级别下一个事务中多次执行同样的查询，其结果都是一样的，这样就实现了可重复读。

<br><br>

# 快照读和当前读

## 快照读

在**读已提交和可重复读**隔离级别下，普通的SELECT查询都是读取MVCC版本链中的一个版本，相当于读取一个快照，因此称为快照读。这种读取方式不会加锁，因此读操作时非阻塞的，因此也叫非阻塞读。

在标准的**可重复读**隔离级别下读操作会加S锁，直到事务结束，因此可以阻止其他事务的写操作；但在MySQL的**可重复读**隔离级别下读操作没有加锁，不会阻止其他事务对相同记录的写操作，因此在后续进行写操作时就有可能写入基于版本链中的旧数据计算得到的结果，这就导致了提交覆盖的问题。想要避免此问题，就需要另外加锁来实现。

## 当前读
之前提到MySQL有两种锁定读的方式：
```sql
SELECT ... LOCK IN SHARE MODE; // 读取时对记录加读锁，直到事务结束

SELECT ... FOR UPDATE; // 读取时对记录加写锁，直到事务结束
```
这种读取方式读取的是记录的当前最新版本，称为当前读。另外对于DELETE、UPDATE操作，也是需要先读取记录，获取记录的写锁，这个过程也是一个当前读。由于需要对记录进行加锁，会阻塞其他事务的写操作，因此也叫加锁读或阻塞读。

当前读不仅会对当前记录加行记录锁，还会对查询范围空间的数据加间隙锁（GAP LOCK），因此可以阻止幻读问题的出现。



<br><br>
> 部分内容借鉴至: https://juejin.cn/post/6844904096378404872
