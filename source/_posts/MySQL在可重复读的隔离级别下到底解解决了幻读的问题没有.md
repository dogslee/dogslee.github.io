---
title: MySQL在可重复读的隔离级别下到底解解决了幻读的问题没有?
date: 2021-11-10 16:18:07
index_img: /img/mysql/mysql.png
banner_img: 
categories:
- 存储
tags:
- MySQL
- MVCC
- lock
- RR
---

## RR幻读到底被解决了没？

做业务朋友对数据库的ACID多少都了解一些，这里面又尤其隔离性是讨论最多且最复杂的，说到隔离性就不得不说由于隔离产生的问题：

1. 脏读
2. 不可重复读
3. 幻读

这里面争议最大的就数MySQL默认隔离级别RR是否解决了幻读的问题， 这篇文章就是以实操的角度来看这个问题，所谓光说不练假把式，透过现象看到事情的本质才是我们要做的

> 先说结论:  
> MySQL在可重复读隔离级别下并没有解决幻读的问题。
> MySQL读取数据有两种模式，分为快照读和当前读，在当前读下MySQL通过引入间隙锁（gap lock） 能避免幻读的出现。
> 当使用快照读的时候仍可能存在幻读的出现。

可能了解一些MVCC的人就会说，你这说的不对，都是快照读了怎么可能读取到幻读的数据。其实这个问题在我不清楚MVCC细节的时候也有一样的疑问。 我们先看下面的实例。

## 出现幻读的实例

数据库表结构：

```sql
CREATE TABLE `test` (
  `id` int(11) NOT NULL,
  `name` varchar(32) NOT NULL DEFAULT '',
  `age` int(11) NOT NULL DEFAULT '0',
  `sex` tinyint(2) NOT NULL DEFAULT '0',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

-- ----------------------------
-- Records of test
-- ----------------------------
INSERT INTO `test` VALUES ('1', 'bob', '18', '1');
```

事务时序图：

| 时间 | 事务A | 事务B |
| ---    | ---     | ---      |
| T1 | begin; |          |
| T2 | select * from test where id >= 1;  | begin; |
| T3 |          | insert into test(id, name, age, sex) values(2, 'lisa', 18, 0); |
| T4 |          | commit; |
| T5 | update test set age = 19 where id >=1; |           |
| T6 | select * from test where id >= 1; |           |
| T7 | commit; |       |

T2时刻获取结果：

```sh
mysql> select * from test where id >=1;
+----+------+-----+-----+
| id | name | age | sex |
+----+------+-----+-----+
|  1 | bob  |  18 |   1 |
+----+------+-----+-----+
1 row in set (0.00 sec)
```

T6时刻获取结果：

```sh
mysql> select * from test where id >= 1;
+----+------+-----+-----+
| id | name | age | sex |
+----+------+-----+-----+
|  1 | bob  |  19 |   1 |
|  2 | lisa |  19 |   0 |
+----+------+-----+-----+
2 rows in set (0.00 sec)
```

从上图中可以看出我们使用**快照读**在同一个事务的前后两次操作中查到了其他事务新插入的数据，满足幻读的概念。

那么既然使用快照读了，为什呢还会出现这个问题嘞？

### 按照一般的隔离性分析
  
理想条件下，事务A获取一条数据然后更新这条数据，最后再次查询这条数据，都应该是对一条数据进行操作。

### 然而这并不是理想条件

实际上事务A的T6时刻的查询，获得了事务B中新插入的数据，仔细看发现新插入的数据也被事务A中的update语句更新了。

### 造成幻读原因

1. 更新语句是当前读，T5时刻的 update 语句执行之前会使用当前读获取最新的全部满足条件的数据进行操作，这就会导致更新语句查询到事务B在T3时刻新插入满足更新条件的数据。
2. 在可重复读隔离级别下使用MVCC无锁实现快照读，T6时刻由于MVCC的原理使得用户可以获取到当前事务更新的数据，也就是自己在T5时刻更新的新插入的数据，所以可以展示出第一次快照读中没有的数据。

### 尝试用当前读执行同样的事务

事务时序图：

| 时间 | 事务A | 事务B |
| ---    | ---     | ---      |
| T1 | begin; |          |
| T2 | select * from test where id >= 1 for update ;  | begin; |
| T3 |          | insert into test(id, name, age, sex) values(2, 'lisa', 18, 0); ***(blocking)*** |
| T4 | update test set age = 19 where id >=1; |           |
| T5 | select * from test where id >= 1 for update; |           |
| T6 | commit; |       |
| T7 |           | commit; |

对比快照读T3时刻插入数据将被阻塞， 这是因为RR下当前读引入了间隙锁的概念。

查询条件 where id >= 1 不光锁住了表中存在的数据id=1的哪一行，同时也锁住了 (1, +∞) 这个间隙，所以事务B在T3时刻语句将被阻塞，直到事务A提交事务之前都将一直阻塞。

所以插要插入的数据根本不能出现在事务A的查询语句中，也就没有可能出现幻读。

### 总结

1. 当前读下由于有行锁和间隙锁，所以不存在幻读的情况发生。
2. 在快照读的情况下，由于事务中可能存在更新语句覆盖了其他事务中不可见的提交。导致一致性视图发生改变可以读到别的事务提交的语句内容，最终导致出现幻读。

### 思考🤔

mysql知识点都是细碎的，但分析问题的时候往往需要将全部的知识脉络串联起来，正如上面的这个例子，涉及到了当前读，快照读，mvcc原理，行锁和间隙锁这些问题。

当我们不清楚的时候最好是实际的动手实验下，仅凭网络上的文章根本无法清楚问题的本质，甚至有些文章上的例子都是有问题的。

所以学习的最好方式就是实践，真的动手尝试才能发现问题的本质。

## 相关概念

概念来自网络，有问题望指正。

### 快照读

一个正常的select…语句就是快照读。
快照读，使得在RR（repeatable read）级别下一个普通select...语句也能做到可重复读。即利用一致性视图来做到（当前事务只能读到该事物开启以前已经提交的数据）。

### 当前读

insert语句、update语句、delete语句、显示加锁的select语句（select… LOCK IN SHARE MODE、select… FOR UPDATE）是当前读。

### mvcc原理

多版本并发控制（MVCC） 是实现一直性视图（consistent read view）的关键 其中主要原理如下：

#### read view 创建时机

可重复读隔离级别：这个视图是在事务启动时创建的，整个事务存在期间都用这个视图。

读提交隔离级别：这个视图是在每个 SQL 语句开始执行的时候创建的。

> 需要注意的是，“读未提交”隔离级别下直接返回记录上的最新值，没有视图概念；而“串行
> 化”隔离级别下直接用加锁的方式来避免并行访问。

#### read view 的主要内容

ReadView所解决的问题是使用READ COMMITTED和REPEATABLE READ隔离级别的事务中，不能读到未提交的记录。通过判断一下版本链中的哪个版本是当前事务可见的。

ReadView中主要包含4个比较重要的内容：

1. m_ids：表示在生成ReadView时当前系统中活跃的读写事务的事务id列表。
2. min_trx_id：表示在生成ReadView时当前系统中活跃的读写事务中最小的事务id，也就是m_ids中的最小值。
3. max_trx_id：表示生成ReadView时系统中应该分配给下一个事务的id值。
4. creator_trx_id：表示生成该ReadView的事务的事务id。

InnoDB 为每个事务构造了一个数组，用来保存这个事务启动瞬间，当前正在“活跃”的所有事务 ID。

> “活跃”指的就是，启动了但还没提交。

数组里面事务 ID 的最小值记为低水位，当前系统里面已经创建过的事务 ID 的最大值加 1记为高水位。这个视图数组和高水位，就组成了当前事务的一致性视图(图-1)

![图-1](/img/mysql/read_view.png)

#### 如何工作

按照下边的步骤判断记录的某个版本是否可见：

如果被访问版本的trx_id属性值与ReadView中的creator_trx_id值相同，意味着当前事务在访问它自己修改过的记录，所以该版本可以被当前事务访问。
如果被访问版本的trx_id属性值小于ReadView中的min_trx_id值，表明生成该版本的事务在当前事务生成ReadView前已经提交，所以该版本可以被当前事务访问。
如果被访问版本的trx_id属性值大于ReadView中的max_trx_id值，表明生成该版本的事务在当前事务生成ReadView后才开启，所以该版本不可以被当前事务访问。
如果被访问版本的trx_id属性值在ReadView的min_trx_id和max_trx_id之间，那就需要判断一下trx_id属性值是不是在m_ids列表中，如果在，说明创建ReadView时生成该版本的事务还是活跃的，该版本不可以被访问；如果不在，说明创建ReadView时生成该版本的事务已经被提交，该版本可以被访问。

### next-key lock

行锁和间隙锁合称next-key lock。Next-Key Lock 只发生在 RR（REPEATABLE-READ） 隔离级别下。

简单来讲是就是说枷锁的时候不单单加在满足条件的行上，同时也要加在不存在数据且满足条件的间隙上。由于不同版本的mysql对枷锁的规则上会有不同，这里不做展开。

## Refrence

[MySQL实战45讲](https://time.geekbang.org/column/intro/100020801?tab=catalog)
[一篇文章带你掌握mysql的一致性视图（MVCC）](https://juejin.cn/post/6844903908146413576)
[MySQL官方文档](https://dev.mysql.com/doc/refman/5.7/en/innodb-transaction-isolation-levels.html)
[什么是间隙锁？到底锁了什么？](https://www.jianshu.com/p/42e60848b3a6)