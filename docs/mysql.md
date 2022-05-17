# MySQL

```text
分布式
    cap
    一致性
    可用性
    分区容忍性

数据库
    acid
    原子性
    一致性
    隔离性
    持久性

Shared and Exclusive Locks 共享/排它锁 
Intention Locks 意向锁
Record Locks 记录锁
Gap Locks 间隙锁
Next-Key Locks 临键锁
Insert Intention Locks 插入意向锁
AUTO-INC Locks 自增锁
Predicate Locks for Spatial Indexes 空间索引谓词锁

Metadata Locks 元数据锁

drop table test;
create table `test`
(
    `id`  int(11) primary key auto_increment,
    `xid` int,
    key `xid` (`xid`)
) engine = InnoDB;

# (-∞, 1]  (1, 3]  (3, 5]  (5, 8]  (8, 11]  (11, +∞)
insert into test(id, xid)
values (1, 1),
       (2, 3),
       (3, 5),
       (4, 8),
       (5, 11);

start transaction;

select *
from test
where xid = 8 for
update;
# (5, 8] ∪ (8, 11]

commit;

# 预测 非堵塞, 实际 堵塞
insert into test( xid) values (5);
# 预测 堵塞, 实际 非堵塞
insert into test( xid) values (11);


CREATE TABLE child (id int(11) NOT NULL, PRIMARY KEY(id)) ENGINE=InnoDB;
INSERT INTO child (id) values (90),(102);

START TRANSACTION;
SELECT * FROM child WHERE id > 100 FOR UPDATE;

START TRANSACTION;
INSERT INTO child (id) VALUES (101);


原则 1：加锁的基本单位是 next-key lock。next-key lock 是前开后闭区间。
原则 2：只有访问到的对象才会加锁。
优化 1：索引上的等值查询，
    命中唯一索，退化为行锁。
    命中普通索引，左右两边的GAP Lock + Record Lock。
优化 2：
    索引上的等值查询，未命中，所在的Net-Key Lock，退化为GAP Lock 。
索引在范围查询： 
    1.等值和范围分开判断。
    2.索引在范围查询的时候 都会访问到所在区间不满足条件的第一个值为止。
    3.如果使用了倒叙排序，按照倒叙排序后，
    检索范围的右边多加一个GAP。
    哪个方向还有命中的等值判断，再向同方向拓展外开里闭的区间。
```