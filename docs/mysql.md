# MySQL

## InnoDB Locking

- 共享/排它锁 （Shared and Exclusive Locks）
- 意向锁 （Intention Locks）
- 记录锁 （Record Locks）
- 间隙锁 （Gap Locks）
- 临键锁 （Next-Key Locks）
- 插入意向锁 （Insert Intention Locks）
- 自增锁 （AUTO-INC Locks）
- 空间索引谓词锁 （Predicate Locks for Spatial Indexes）

### 共享/排它锁

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

### 意向锁

InnoDB 支持多粒度锁，允许行锁和表锁共存。例如：`LOCK TABLES ... WRITE`之类的语句在指定表上采用排他(X)锁。为了使多粒度级别的锁定变得实用，InnoDB 使用意图锁。
意向锁是表级锁，它指示事务稍后对表中的行需要哪种类型的锁（共享或独占）。有两种类型的意图锁：

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

## Metadata Locking

## ACID

是指数据库管理系统（DBMS）在写入或更新资料的过程中，为保证事务（Transaction）是正确可靠的，所必须具备的四个特性：

- 原子性（Atomicity）
- 一致性（Consistency）
- 隔离性（Isolation）
- 持久性（Durability）
