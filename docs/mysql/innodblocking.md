# InnoDB Locking

- 共享/排它锁 （Shared and Exclusive Locks）
- 意向锁 （Intention Locks）
- 记录锁 （Record Locks）
- 间隙锁 （Gap Locks）
- 临键锁 （Next-Key Locks）
- 插入意向锁 （Insert Intention Locks）
- 自增锁 （AUTO-INC Locks）
- 空间索引谓词锁 （Predicate Locks for Spatial Indexes）

## 共享/排它锁

InnoDB实现标准的行级锁定，其中有两种类型的锁：

- 共享锁（S）：允许持有该锁的事务读取一行。
- 排它锁（X）：允许持有该锁的事务更新或删除一行。

如果一个事务`T1`持有行`r`上的一个共享(S)锁，那么来自不同事务T2的请求对行`r`上的一个锁处理如下:

- 可以立即授予`T2`对(S)锁的请求，因此，`T1`和`T2`都对行`r`保持(S)锁定。
- 不能立即授予`T2`对(X)锁的请求。

如果事务`T1`在行`r`上持有排他(X)锁，则无法立即授予来自某个不同事务`T2`对`r`上任一类型的锁的请求。相反，事务`T2`必须等待事务`T1`释放其对行`r`的锁定。

|     |  X   |  S   |
|:---:|:----:|:----:|
|  X  |  冲突  |  冲突  |
|  S  |  冲突  |  兼容  |

## 意向锁

InnoDB 支持多粒度锁，允许行锁和表锁共存。例如：`LOCK TABLES ... WRITE`之类的语句在指定表上采用排他(X)锁。为了使多粒度级别的锁定变得实用，InnoDB 使用意图锁。意向锁是表级锁，它指示事务稍后对表中的行需要哪种类型的锁（共享或独占）。有两种类型的意图锁：

- 意向共享锁（IS）：事务想要获得一张表中某几行的共享锁。
- 意向排它锁（IX）：事务想要获得一张表中某几行的排它锁。

意向锁规则如下：

在事务可以获取表中行的共享(S)锁之前，它必须首先获取表上的(IS)锁或更重的锁。

在事务可以获取表中行的排他(S)锁之前，它必须首先获取表上的(IX)锁。

除了全表请求（例如：`LOCK TABLES ... WRITE`）之外，意图锁不会阻塞任何东西。意图锁的主要目的是表明有人正在锁定一行，或者要锁定表中的一行。

设置(IS)锁：`SELECT ... FOR SHARE`

设置(IX)锁：`SELECT ... FOR UPDATE`

注意，`FOR SHARE`是`LOCK IN SHARE MODE`的替代品，但`LOCK IN SHARE MODE`仍可用于向后兼容。

|     |  X   |  IX  |  S   |  IS  |
|:---:|:----:|:----:|:----:|:----:|
|  X  |  冲突  |  冲突  |  冲突  |  冲突  |
| IX  |  冲突  |  兼容  |  冲突  |  兼容  |
|  S  |  冲突  |  冲突  |  兼容  |  兼容  |
| IS  |  冲突  |  兼容  |  兼容  |  兼容  |

## 记录锁

记录锁是对索引记录的锁。例如：`SELECT c1 FROM t WHERE c1 = 10 FOR UPDATE`，防止任何其他事务插入、更新或删除`t.c1`值为`10`的行。

## 间隙锁

间隙锁是在索引记录之间的间隙上的锁，或在第一条索引记录之前或最后一条索引记录之后的间隙上的锁。例如：`SELECT c1 FROM t WHERE c1 BETWEEN 10 and 20 FOR UPDATE`，防止其他事务将值`15`插入`t.c1`列，无论该列中是否已经存在任何此类值，因为该范围内所有现有值之间的间隙都已锁定。

间隙可能跨越单个索引值、多个索引值，甚至是空的。

InnoDB 中的间隙锁是“纯粹的抑制性”，这意味着它们的唯一目的是防止其他事务插入到间隙中。间隙锁可以共存。一个事务采用的间隙锁不会阻止另一个事务在同一间隙上采用间隙锁。共享和独占间隙锁之间没有区别。它们彼此不冲突，并且执行相同的功能。

间隙锁定可以被显式禁用。如果您将事务隔离级别更改为`READ COMMITTED`，则会发生这种情况。在这种情况下，间隙锁定对搜索和索引扫描禁用，仅用于外键约束检查和重复键检查。

## 临键锁

临键锁是记录锁和间隙锁的组合。

假设存在以下表及数据：

```mysql
create table `test`
(
    `id` int primary key auto_increment,
    `v`  int,
    key `v` (`v`)
) engine = InnoDB;
```

其中`id`列为自增主键，并为`v`列创建了索引。

| id  | v   |
|:----|:----|
| 10  | 1   |
| 11  | 3   | 
| 12  | 5   | 
| 13  | 8   | 
| 14  | 11  | 
| 15  | 13  | 

该索引可能的临键锁涵盖以下区间：

```text
(-∞, 1]
(1, 3]
(3, 5]
(5, 8]
(8, 11]
(11, 13]
(13, +∞)
```

当我们开启事务`T1`执行以下`SQL`时，保持事务不提交，会锁住`(5, 8]`和`(8, 11]`区间。

```mysql
start transaction;
select * from test where v = 8 for update;
```

所以我们可以预测在新的事务`T2`中分别执行以下`SQL`的结果：

```mysql
insert into test(v) values(1);  # 区间外 预测：non-blocking 实际：non-blocking 符合预期
insert into test(v) values(4);  # 区间外 预测：non-blocking 实际：non-blocking 符合预期
insert into test(v) values(5);  # 区间外 预测：non-blocking 实际：blocking     不符合预期
insert into test(v) values(9);  # 区间内 预测：blocking     实际：blocking     符合预期
insert into test(v) values(11); # 区间内 预测：blocking     实际：non-blocking 不符合预期
insert into test(v) values(12); # 区间外 预测：non-blocking 实际：non-blocking 符合预期
```

根据临键锁的区间，我们预测在`T2`事务中分别插入值为`9`和`11`的数据会被`T1`事务阻塞。 但是实际上在插入值为`5`和`9`时才会被阻塞，结果并不符合预期，似乎违背了临键锁的设计，这就要从索引结构（B+tree）说起了。

是由于 InnoDB 的叶子节点都是按顺序的插入的，并且`id`为自增主键，因此我们插入值为`5`的记录时，会将数据追加到原来值为`5`数据的后面，于是就落到了`T1`事务临键锁涵盖的区间内。同理可知插入值为`11`的记录时，实际的叶子节点在区间外。

![mysql_nextkeylocks_btree](../images/mysql_nextkeylocks_btree.png)

## 插入意向锁

插入意向锁是由`INSERT`操作设置的一种特殊的间隙锁，而并非是上述提到的意向锁，意向锁是表级锁，而插入意向锁是行级锁。

此锁表示插入的意图，即如果插入到同一索引间隙中的多个事务未插入到间隙内的同一位置，则它们无需相互等待。 假设有值为 4 和 7 的索引记录。分别尝试插入值 5 和 6 的单独事务，在获得插入行的排他锁之前，每个使用插入意图锁锁定 4 和 7 之间的间隙，但不要相互阻塞，因为行是不冲突的。

## 自增锁

自增锁是一种特殊的表级锁，由插入到具有`AUTO_INCREMENT`列的表中的事务使用。 在最简单的情况下，如果一个事务正在向表中插入值，则任何其他事务都必须等待在该表中执行自己的插入操作，以便第一个事务插入的行接收连续的主键值。

## 空间索引谓词锁

TODO

## 测试题

假设存在如下表：

```mysql
create table test
(
    id bigint primary key auto_increment,
    k  int,
    v  int,
    constraint uk_k unique (k)
);
```

|     |                Session 1                |                Session 2                |                Session 3                | 
|:---:|:---------------------------------------:|:---------------------------------------:|:---------------------------------------:|
|  1  | `insert into test (k, v) values (1, 1)` |                                         |                                         |
|  2  |                                         | `insert into test (k, v) values (1, 1)` |                                         | 
|  3  |                                         |                                         | `insert into test (k, v) values (1, 1)` | 
|  4  |                rollback                 |                                         |                                         |  
|  5  |                                         |                                         |                deadlock                 |  
|  6  |                                         |                 commit                  |                                         |  

为什么 Session 1 回滚会导致 Session 3 死锁，而 Session 2 可以插入成功呢？

原因是 Session 1 插入成功后未提交事务，此时持有排它锁（X）， Session 2 和 Session 3 需要排队获取共享锁（S）检查唯一索引。
Session 1 插入成功后 rollback ，于是 Session 2 和 Session 3 同时获得了共享锁（S），发现不存在冲突的唯一索引，所以它们此时都想升级排它锁（X），
于是双方都在等对方释放共享锁（S）而发生死锁，最后只能 abort 其中一个，而另一个成功升级排它锁（X）之后插入成功。