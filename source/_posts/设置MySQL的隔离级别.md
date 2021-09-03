---
title: 设置MySQL的隔离级别
date: 2021-08-25 23:00:28
index_img: /img/mysql/mysql.png
banner_img: /img/mysql/isolation.jpg
categories:
  - 存储
tags:
  - mysql
---


## 怎么查当前的隔离级别？

<p class="note note-success">MySQL数据库默认的存储引擎是支持事务的innodb,所以自然的就有默认的隔离级别--可重复读</p>
<p class="note note-warning">只有支持ACID的存储引擎才有对应的各种隔离级别</p>

在我们设置当前事务的隔离级别的时候我们首先要会查询我们的MySQL的隔离级别是什么

MySQL查询隔离级别的语句是：

```bash
mysql> select @@global.tx_isolation,@@tx_isolation;
+-----------------------+-----------------+
| @@global.tx_isolation | @@tx_isolation  |
+-----------------------+-----------------+
| REPEATABLE-READ       | REPEATABLE-READ |
+-----------------------+-----------------+
1 row in set (0.00 sec)
```

## 如何设置自己的隔离级别？

设置innodb的事务级别方法是：

set 作用域 transaction isolation level 事务隔离级别，例如~

> SET [SESSION | GLOBAL] TRANSACTION ISOLATION LEVEL {READ UNCOMMITTED | READ COMMITTED | REPEATABLE READ | SERIALIZABLE}

```bash
// 全局的
mysql> set global transaction isolation level read committed;
// 当前会话
mysql> set session transaction isolation level read committed;
```

## 各个隔离级别都是啥意思？

SQL标准定义了4类隔离级别，包括了一些具体规则，用来限定事务内外的哪些改变是可见的，哪些是不可见的。低级别的隔离级一般支持更高的并发处理，并拥有更低的系统开销。

但是读提交和串行化一般很少使用

- **Read Uncommitted（读取未提交内容）**
在该隔离级别，所有事务都可以看到其他未提交事务的执行结果。本隔离级别很少用于实际应用，因为它的性能也不比其他级别好多少。读取未提交的数据，也被称之为脏读（Dirty Read）。

- **Read Committed（读取提交内容）**
这是大多数数据库系统的默认隔离级别（但不是MySQL默认的）。它满足了隔离的简单定义：一个事务只能看见已经提交事务所做的改变。这种隔离级别 也支持所谓的不可重复读（Nonrepeatable Read），因为同一事务的其他实例在该实例处理其间可能会有新的commit，所以同一select可能返回不同结果。

- **Repeatable Read（可重读）**
这是MySQL的默认事务隔离级别，它确保同一事务的多个实例在并发读取数据时，会看到同样的数据行。不过理论上，这会导致另一个棘手的问题：幻读 （Phantom Read）。简单的说，幻读指当用户读取某一范围的数据行时，另一个事务又在该范围内插入了新行，当用户再读取该范围的数据行时，会发现有新的“幻影” 行。InnoDB和Falcon存储引擎通过多版本并发控制（MVCC，Multiversion Concurrency Control）机制解决了该问题。

- **Serializable（可串行化）**
这是最高的隔离级别，它通过强制事务排序，使之不可能相互冲突，从而解决幻读问题。简言之，它是在每个读的数据行上加上共享锁。在这个级别，可能导致大量的超时现象和锁竞争。

## 脏读、不可重复读、幻读

1. 脏读 ：脏读就是指当一个事务正在访问数据，并且对数据进行了修改，而这种修改还没有提交到数据库中，这时，另外一个事务也访问 这个数据，然后使用了这个数据。
 
2. 不可重复读 ：是指在一个事务内，多次读同一数据。在这个事务还没有结束时，另外一个事务也访问该同一数据。那么，在第一个事务中的两 次读数据之间，由于第二个事务的修改，那么第一个事务两次读到的的数据可能是不一样的。这样就发生了在一个事务内两次读到的数据是不一样的，因此称为是不 可重复读。例如，一个编辑人员两次读取同一文档，但在两次读取之间，作者重写了该文档。当编辑人员第二次读取文档时，文档已更改。原始读取不可重复。如果 只有在作者全部完成编写后编辑人员才可以读取文档，则可以避免该问题。
 
3. 幻读 : 是指当事务不是独立执行时发生的一种现象，例如第一个事务对一个表中的数据进行了修改，这种修改涉及到表中的全部数据行。 同时，第二个事务也修改这个表中的数据，这种修改是向表中插入一行新数据。那么，以后就会发生操作第一个事务的用户发现表中还有没有修改的数据行，就好象 发生了幻觉一样。例如，一个编辑人员更改作者提交的文档，但当生产部门将其更改内容合并到该文档的主复本时，发现作者已将未编辑的新材料添加到该文档中。 如果在编辑人员和生产部门完成对原始文档的处理之前，任何人都不能将新材料添加到文档中，则可以避免该问题。

<p class="note note-primary">这里只有幻读理解起来有些绕 简单来讲就是同一个事物中连续执行两次同样的sql语句，可能导致不同的结果问题，第二次sql语句可能返回之前不存在的行</p>

## 不同隔离级别对应可能出现问题

|      隔离级别      |  脏读  | 不可重复读 | 幻读  |
|       ----        | ----  | ---      | ---   |
| Read Uncommitted  | Y     | Y        | Y     |
| Read Committed    | N     | Y        | Y     |
| Repeatable Read   | N     | N        | Y     |
| Serializable      | N     | N        | N     |

