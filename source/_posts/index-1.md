---
title: index-1
date: 2020-06-29 15:15:40
tags:
---

## 聚簇索引 (clustered index) VS 非聚簇索引 (secondary index)

聚簇索引并不是一种索引类型，只是一种存储方式。当表有了聚簇索引的时候，表的数据行都存放在索引树的叶子页中。无法把数据行放到两个不同的地方，所以一张表只允许有一个聚簇索引

<!-- More -->

我们熟悉的 Myisam 和 innodb 两大引擎，innodb 的默认数据结构是聚簇索引，而 Myisam 是非聚簇索引

Innodb 是通过主键来聚集数据的，就是“被索引的列就是主键”。如果一张表没有主键，那就会通过某一唯一列来聚集数据，没有唯一列的时候，就会隐式的生成一个 id，通过这个 id 来聚集数据。

## 速度区别

1. 在 Myisam 引擎索引和数据是分开存储的，而 Innodb 是索引和数据是一起以 idb 文件的形式进行存储的

   在访问速度上，聚簇索引比非聚簇索引快。非聚簇索引需要先查询一遍索引文件，得到索引，跟据索引获取数据。而聚簇索引的索引树的叶子节点的直接指向要查找的数据行

2. 对于采用聚簇索引的 innodb 引擎的主键索引 B+Tree 和 Myisam 的主键索引树还有 Myisam 的二级索引 B+Tree 都是采用这样的结构

![](https://img-blog.csdn.net/2018080917254946?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1dlaUppRmVuZ18=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

    Innodb的二级索引B+Tree却是这样的:

![](https://img-blog.csdn.net/2018080917301364?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1dlaUppRmVuZ18=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

在使用二级索引进行查询的时候，Innodb 首先通过二级索引 B+Tree 得到数据行的主键索引，然后再通过主键索引树查询数据。所以在二级索引，Innodb 的性能消耗比较大

于是 这就有另一个问题

1. auto-increment 更好，主键采用自增主键会保证每次都在最后面增加叶子节点。不会带来新数据的插入，导致前面的数据变动，而产生页分裂，随之带来空间碎片和时间的消耗
2. 并发的时候， innodb 的 Auto_increment

### innodb 的 Auto_increment

```
Tx1: INSERT INTO t1 (c2) SELECT 1000 rows from another table ...
Tx2: INSERT INTO t1 (c2) VALUES ('xxx');
```

InnoDB 不能预先得知有多少行会从 TX1 的 select 部分获取到,所以在事务进行过程中,InnoDB 一次只会为 AUTO_INCREMENT 列分配一个值.
通过一个表级锁的控制,保证了在同一时刻只有一个引用表 t1 的 INSERT 语句可以执行,直到整个 INSERT 语句结束,并且由不同语句生成自动递增数不会交错

在连续锁定模式下，InnoDB 可以避免为“Simple inserts”语句使用表级 AUTO-INC 锁，其中行数是预先已知的，并且仍然保留基于语句的复制的确定性执行和安全性。

1. tradition(innodb_autoinc_lock_mode=0)

   - 它提供了一个向后兼容的能力

   - 在这一模式下，所有的 insert 语句("insert like") 都要在语句开始的时候得到一个表级的 auto_inc 锁，在语句结束的时候才释放这把锁，注意呀，这里说的是语句级而不是事务级的，一个事务可能包涵有一个或多个语句。

   - 它能保证值分配的可预见性，与连续性，可重复性，这个也就保证了 insert 语句在复制到 slave 的时候还能生成和 master 那边一样的值(它保证了基于语句复制的安全)。

   - 由于在这种模式下 auto_inc 锁一直要保持到语句的结束，所以这个就影响到了并发的插入。

1. innodb_autoinc_lock_mode = 1 (“consecutive” lock mode)

   这是默认的锁定模式.在这个模式下,“bulk inserts”仍然使用 AUTO-INC 表级锁,并保持到语句结束.这适用于所有 INSERT ... SELECT，REPLACE ... SELECT 和 LOAD DATA 语句。同一时刻只有一个语句可以持有 AUTO-INC 锁.

   “Simple inserts”（要插入的行数事先已知）通过在 mutex（轻量锁）的控制下获得所需数量的自动递增值来避免表级 AUTO-INC 锁， 它只在分配过程的持续时间内保持，而不是直到语句完成。 不使用表级 AUTO-INC 锁，除非 AUTO-INC 锁由另一个事务保持。 如果另一个事务保持 AUTO-INC 锁，则“简单插入”等待 AUTO-INC 锁，如同它是一个“批量插入”。

1. innodb_autoinc_lock_mode = 2 (“interleaved” lock mode)

   在这种锁定模式下,所有类 INSERT(“INSERT-like” )语句都不会使用表级 AUTO-INC lock,并且可以同时执行多个语句。这是最快和最可扩展的锁定模式，但是当使用基于语句的复制或恢复方案时，从二进制日志重播 SQL 语句时，这是不安全的。

   在此锁定模式下，自动递增值保证在所有并发执行的“类 INSERT”语句中是唯一且单调递增的。但是，由于多个语句可以同时生成数字（即，跨语句交叉编号），为任何给定语句插入的行生成的值可能不是连续的。

---

### DB Isolation Level:

**Read Uncommitted** Read Uncommitted is the lowest isolation level. In this level, one transaction may read not yet committed changes made by other transaction, thereby allowing dirty reads. In this level, transactions are not isolated from each other.

--> Dirty Read

**Read committed** is an isolation level that guarantees that any data read was committed at the moment is read. It simply restricts the reader from seeing any intermediate, uncommitted, 'dirty' read. It makes no promise whatsoever that if the transaction re-issues the read, will find the Same data, data is free to change after it was read.

--> Non Repeatable Read

**Repeatable read** is a higher isolation level, that in addition to the guarantees of the read committed level, it also guarantees that any data read cannot change, if the transaction reads the same data again, it will find the previously read data in place, unchanged, and available to read.

c--> Phantom Read

Because the repeatable read make the read result as the same as before, even through the new data has been insert. So you can update this data. Then read again. It will appear.

**serializable**, makes an even stronger guarantee: in addition to everything repeatable read guarantees, it also guarantees that no new data can be seen by a subsequent read.
