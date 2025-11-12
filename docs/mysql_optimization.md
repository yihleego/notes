# 优化

## 分析SQL语句执行慢的思路

### 一、确认问题特征

1. 慢查询范围
    - 是所有 SQL 都慢（可能是系统或磁盘问题）。
    - 还是部分 SQL 慢（重点关注语句和索引）。
2. 时间维度
    - 是否间歇性变慢（资源竞争或锁等待）。
    - 是否长期慢（语句设计或索引问题）。

### 二、环境与系统层面检查

1. 服务器资源
    - CPU 使用率、内存使用率、IO 等待（top, vmstat, iostat）。
    - 确认磁盘、网络未成为瓶颈。
2. 数据库连接与负载
    - 当前连接数：SHOW PROCESSLIST;
    - 最大连接数配置：show variables like 'max_connections';
    - 查看活跃会话是否过多、锁是否积压。

### 三、数据库层配置优化

1. 参数检查
    - innodb_buffer_pool_size（建议占物理内存的 60%~80%）
    - query_cache_size（MySQL 5.7 之后通常关闭）
    - innodb_log_file_size、tmp_table_size 等影响性能的参数。

2. 慢查询日志

- 打开慢查询日志：
    ```sql
    SET global slow_query_log = 1;
    SET global long_query_time = 1;  -- 超过1秒记录
    ```
- 分析日志文件：mysqldumpslow 或 pt-query-digest。

### 四、语句层面分析

1. 使用 EXPLAIN 查看执行计划
    ```sql
    EXPLAIN SELECT ...;
    ```
   重点关注：
    - type（连接类型，越靠近 ALL 越差）
    - key（使用的索引）
    - rows（扫描行数）
    - Extra（是否出现 “Using filesort” 或 “Using temporary”）
2. 索引问题
    - 是否缺乏合适索引。
    - 是否出现 索引未命中（如函数运算、类型不匹配、隐式转换）。
    - 是否存在 多列索引顺序不匹配（索引最左匹配原则）。
3. SQL 编写问题
    - 使用 SELECT * 导致多余字段读取。
    - OR、IN 过多时可能导致全表扫描。
    - ORDER BY、GROUP BY 未命中索引会触发排序和临时表。
    - 子查询可考虑改写为 JOIN 或 EXISTS。

### 五、运行时与锁等待分析

1. 事务与锁
    - 查看锁等待：
    ```sql
    SHOW ENGINE INNODB STATUS\G;
    ```
    - 若有锁竞争，可分析事务隔离级别、长事务、死锁等。
2. 实时监控
    - SHOW FULL PROCESSLIST; 查看当前 SQL 执行状态。
    - performance_schema 或 sys schema 分析等待事件。

### 六、综合工具与优化建议

1. 分析工具
    - pt-query-digest（慢日志分析）
    - EXPLAIN ANALYZE（MySQL 8.0+，查看真实执行时间）
    - MySQLTuner（自动检测配置建议）
2. 优化方向
    - 加索引 / 改写 SQL
    - 优化缓存与内存配置
    - 拆分大表 / 增加分区
    - 读写分离 / 水平扩展

## 执行计划 EXPLAIN

EXPLAIN 是 MySQL 优化查询性能的核心工具之一。它能展示优化器打算如何执行 SQL 查询（或实际执行情况，取决于用法）。

### 一、EXPLAIN 的基本用法

语法：

```sql
EXPLAIN SELECT * FROM table_name WHERE ...
```

或在 MySQL 8.0 中，也可以：

```sql
EXPLAIN ANALYZE SELECT ...
```

后者不仅展示计划，还会实际执行并给出真实耗时（更精确）。

### 二、输出字段详解

执行 EXPLAIN 后会得到类似这样的结果：

| id | select_type | table | partitions | type | possible_keys | key | key_len | ref | rows | filtered | Extra | 
|----|-------------|-------|------------|------|---------------|-----|---------|-----|------|----------|-------|

下面是重点字段说明：

| 字段                | 含义                                        | 性能意义             |
|-------------------|-------------------------------------------|------------------|
| **id**            | 查询中每个子查询的标识符，数字越大越先执行                     | 一般一个查询只有一个 id    |
| **select_type**   | 查询类型，如 SIMPLE、PRIMARY、SUBQUERY、DERIVED    | 用来区分主查询/子查询      |
| **table**         | 当前行对应的表名或别名                               | —                |
| **type**          | 连接类型（最重要）                                 | **越靠近 ALL 性能越差** |
| **possible_keys** | 优化器认为可用的索引                                | —                |
| **key**           | 实际使用的索引                                   | NULL 表示未使用索引     |
| **key_len**       | 使用索引的字节长度                                 | 反映是否用到全部联合索引列    |
| **ref**           | 与索引比较的列或常量                                | —                |
| **rows**          | MySQL 预计扫描的行数                             | 越大说明代价越高         |
| **filtered**      | 按条件过滤后剩余的比例                               | —                |
| **Extra**         | 额外信息，如 "Using filesort"、"Using temporary" | 重要的性能信号          |

### 三、type（连接类型）优劣排序

从好到坏的大致顺序如下：

| type   | 含义                       | 性能 |
|--------|--------------------------|----|
| system | 表只有一行                    | 最优 |
| const  | 通过主键或唯一索引一次定位            | 很好 |
| eq_ref | 联接时使用主键/唯一索引             | 很好 |
| ref    | 使用普通索引匹配多行               | 一般 |
| range  | 使用索引范围扫描 (BETWEEN, >, <) | 一般 |
| index  | 全索引扫描                    | 较差 |
| ALL    | 全表扫描                     | 最差 |

重点：如果出现 ALL，几乎总是性能问题，需要检查索引或 WHERE 条件。

做 ALL 扫描时，它读取的是“聚簇索引的叶子页”，目的是读数据，而不是为了使用主键索引进行查找。
做 index 全索引扫描时，扫描二级索引（整个索引树）。

### 四、Extra 常见提示含义

| Extra 内容          | 含义                         | 性能建议           |
|-------------------|----------------------------|----------------|
| Using where       | 有条件过滤                      | 正常             |
| Using index       | 覆盖索引（无需回表）                 | 👍 优化良好        |
| Using temporary   | 使用临时表（如 GROUP BY、DISTINCT） | ⚠️ 有优化空间       |
| Using filesort    | 需要额外排序（ORDER BY）           | ⚠️ 尝试建索引       |
| Using join buffer | 联接未用索引                     | ⚠️ 需优化 JOIN 条件 |
| Impossible WHERE  | WHERE 条件恒为 false           | 查询可优化掉         |

### 五、分析性能瓶颈的思路

1. 查看 type 是否为 ALL
   → 没有索引或索引未命中。
2. rows 是否很大
   → 表明扫描行过多，应加索引或优化条件。
3. Extra 是否有 Using filesort / temporary
   → 表明 ORDER BY / GROUP BY 无法用索引排序。
4. key 是否为 NULL
   → 没有使用索引。
5. key_len 是否小于索引总长度
   → 联合索引未完全利用。

EXPLAIN ANALYZE 查看实际耗时和差距，识别估算不准的地方。

### 六、优化方向示例

- WHERE + ORDER BY 的列可以建立复合索引。
- 使用覆盖索引（SELECT 中只取索引列）。
- 避免在索引列上使用函数或类型转换。
- 用 LIMIT 限制返回数据量。
- 尽量减少子查询、使用 JOIN 替代。

## 8.0 EXPLAIN ANALYZE

### 一、什么是 EXPLAIN ANALYZE

- 从版本 8.0.18 开始，MySQL 支持 EXPLAIN ANALYZE。
- 与传统 EXPLAIN（只是显示优化器预估的执行计划）不同，EXPLAIN ANALYZE 会*实际执行*查询，同时对执行过程中每一个迭代器（iterator）进行时间统计、行数统计。
- 输出采用 FORMAT=TREE 树形格式（嵌套结构）来显示各个执行步骤，并在每个步骤中显示：
    - 预估的成本 (cost) 和预估行数 (rows)
    - 实际执行的时间 (actual time)
    - 实际读取行数 (rows)
    - 循环次数 (loops)
- 由于会 执行查询，所以如果查询本身更新数据、或你不想真正运行查询，那么需慎用 EXPLAIN ANALYZE。

### 二、基本语法与支持范围

#### 语法

可以指定其他语句，如 UPDATE、DELETE、TABLE 等（在支持版本中）:

```sql
EXPLAIN ANALYZE
SELECT …;
```

格式默认 FORMAT=TREE，也可显式指定：

```sql
EXPLAIN FORMAT=TREE ANALYZE SELECT …;
```

注意：传统的 FORMAT=JSON、FORMAT=TRADITIONAL 虽然 EXPLAIN 支持，但 EXPLAIN ANALYZE 强制／默认使用 TREE 格式。

#### 支持范围

- 支持的语句：SELECT、多表 UPDATE、多表 DELETE、TABLE 语句等。
- 注意事项：某些语句或特性可能不支持被 EXPLAIN ANALYZE 执行（例如存储过程中的语句、某些执行路径）——这在实践中遇到过。

### 三、输出字段详解

```sql
-> Index range scan on sbtest1 using idx3 (cost=98896.53 rows=493204) (actual time=0.021..96.488 rows=625262 loops=1)
```

| 项目                                       | 含义                                              |
|------------------------------------------|-------------------------------------------------|
| `Index range scan on sbtest1 using idx3` | 操作类型：在表 `sbtest1` 上使用索引 `idx3` 进行范围扫描           |
| `cost=98896.53`                          | 优化器预估的执行成本（抽象单位）                                |
| `rows=493204`                            | 优化器预估此操作将返回／处理约 493,204 行                       |
| `actual time=0.021..96.488`              | 实际执行时间：从初始化到返回第一行大约 0.021 毫秒，到全部行返回大约 96.488 毫秒 |
| `rows=625262`                            | 实际处理的行数：625,262 行                               |
| `loops=1`                                | 此迭代器被调用 (Init + Read) 的次数：1 次                   |

此外，顶层会有整体信息，比如：

```sql
EXPLAIN:
-> Aggregate: count(0) (actual time=178.225..178.225 rows=1 loops=1)
...
```

可以看出：聚合操作的实际时间、返回行数、循环次数。

要点说明

- actual time 的第一个数字是“初始化＋第一行处理时间”；第二个数字是“处理完全部行”的时间。
- rows 和 loops 给出实际行数和此步骤被调用次数；如果 loops>1，说明这个迭代器被反复用了（如嵌套循环 join 等情境）
- 对比 cost / rows（预估）和实际 rows / actual time 是关键：若差异大，说明优化器对该步骤估计不准确，可能是性能瓶颈所在。

### 四、使用场景与实战建议

#### 场景

- 当一个查询执行缓慢、你希望定位是哪一步耗时最多、或者优化器估计严重失真时，用 EXPLAIN ANALYZE 是非常有帮助的。
- 想检查索引是否真的被利用、扫描量是否如预估、是否出现大量循环、多次读取、JOIN 方式是否合理。
- 与传统 EXPLAIN 配合使用：先用 EXPLAIN 看预估计划，再用 EXPLAIN ANALYZE 看实际执行。

#### 实践建议

- 在生产库上使用需谨慎：因为它真正执行查询，会对系统负载有额外开销，并且会返回实际数据操作（如果查询有副作用的话）。
- 建议在测试环境或复制库上用 EXPLAIN ANALYZE 来调优。
- 在 EXPLAIN ANALYZE 结果中重点观察：
    - 预估行数 vs 实际行数差异大吗？差别大说明统计信息可能不准确或数据分布有偏差。
    - 某步骤实际耗时特别长，可能是扫描／排序／嵌套循环瓶颈。
    - 是否出现 loops > 1 且每次循环耗时又高？说明有重复读取或子查询被重复评估。
    - 索引是否被使用、访问类型是否合理（比如避免全表扫描 unless 必要）。
- 在调优之后，再次用 EXPLAIN ANALYZE 复查效果。
- 注意： EXPLAIN ANALYZE 本身会稍微比纯执行查询慢一点，因为有额外的时间测量逻辑开销。

### 五、限制与注意事项

- 因为 EXPLAIN ANALYZE 会执行查询，如果查询修改数据（如 UPDATE、DELETE）或者返回大量结果、或在高负载环境，可能造成问题。
- 并非所有查询路径均可被 “迭代器执行模式” 支持，因此某些查询可能会报告 “not executable by iterator executor” 或输出较少信息。
- 输出中 cost 是“预估”值，不能直接当作实际耗时时间。实际时间（actual time）才是关键。
- 要理解树形结构中多个层级之间的关系，需要有一定执行计划和数据库执行机制的理解。比如：一个 join 会有子-迭代器、再上层聚合、排序等。
- 使用版本需 ≥ 8.0.18 才能用 EXPLAIN ANALYZE。如果版本低于该版本，将报语法错误。

## 查询优化器 Optimizer Trace

MySQL Optimizer Trace 是 MySQL 提供的一种诊断工具，用来深入了解查询优化器（optimizer）是如何选择执行计划的。它能显示优化器在分析、重写和生成执行计划过程中的详细步骤，非常适合调试慢查询或研究优化器行为。

### 一、开启 Optimizer Trace

Optimizer Trace 默认是关闭的。你可以在会话级别开启：

```sql
SET optimizer_trace="enabled=on";
```

若只想开启一次分析，不影响其他会话：

```sql
SET optimizer_trace="enabled=on", optimizer_trace_max_mem_size=1000000;
```

optimizer_trace_max_mem_size 控制内存大小（字节数），若 trace 太长可适当调大。

### 二、执行你要分析的查询

举例：

```sql
SELECT * FROM orders WHERE customer_id = 5 AND order_date > '2025-01-01';
```

### 三、查看 Trace 结果

执行：

SELECT * FROM INFORMATION_SCHEMA.OPTIMIZER_TRACE;

该表包含四个主要字段：

| 字段名                               | 说明                   |
|:----------------------------------|:---------------------|
| QUERY                             | 你刚执行的 SQL            |
| TRACE                             | 优化器的详细 JSON 格式 trace |
| MISSING_BYTES_BEYOND_MAX_MEM_SIZE | 若非 0，表示 trace 被截断    |
| INSUFFICIENT_PRIVILEGES           | 若非空，表示无权限            |

### 四、阅读 TRACE 内容

TRACE 是 JSON 格式的字符串，里面包含多个阶段：

示例（简化）：

```json
{
  "steps": [
    {
      "join_preparation": {
        "select_id": 1,
        "steps": [
          { "expanded_query": "SELECT * FROM orders WHERE customer_id=5 AND order_date>'2025-01-01'" }
        ]
      }
    },
    {
      "join_optimization": {
        "rows_estimation": [
          { "table": "orders", "records": 1000000, "records_after_filter": 1000 }
        ],
        "chosen_plan": {
          "best_access_path": { "table": "orders", "index": "idx_customer_date", "cost": 120.5 }
        }
      }
    },
    { "join_execution": {} }
  ]
}
```

重点部分：

- join_preparation：查询重写阶段，包含子查询展开、视图展开。
- join_optimization：最关键部分，显示索引选择、成本估算、join 顺序评估。
- rows_estimation：估算行数信息。
- chosen_plan：最终选择的执行计划及成本。
- join_execution：执行阶段（一般为空）。

### 五、常见用途

1. 验证优化器索引选择是否合理
   看 chosen_plan 使用了哪个索引。
2. 分析为什么没用索引
   查看 best_access_path 或 considered_access_paths 中的成本比较。
3. 研究复杂 join 顺序
   查看 join_optimization 的 considered_execution_plans。
4. 辅助调优
   搭配 EXPLAIN FORMAT=JSON 对照查看。

### 六、关闭 Trace

用完后关闭（避免占用内存）：

```
SET optimizer_trace="enabled=off";
```

### 七、实用建议

- 与 EXPLAIN 一起使用：先用 EXPLAIN 看计划，再用 trace 看“为什么”。
- 仅在测试环境使用：生产环境开启 trace 可能有性能影响。
- 可以保存 JSON 到文件，用 JSON viewer 分析结构。

## `EXISTS` vs `IN`

### `IN`的执行方式

`IN`子句会先执行子查询，把结果集保存下来，然后将外层查询的每一行拿来与这个结果集进行比对。

```sql
SELECT * FROM users
WHERE id IN (SELECT user_id FROM orders);
```

执行逻辑大致如下：

1. 执行子查询 (SELECT user_id FROM orders)；
2. 将结果集（可能是上千行甚至更多）放入内存；
3. 外层查询扫描 users 表的每一行，判断 id 是否存在于结果集中。

⚠️如果子查询结果集很大，IN 的判断过程会变得非常慢。
尤其在 MySQL 5.x 早期版本中，IN 子查询甚至可能被物化成临时表。

### `EXISTS`的执行方式

`EXISTS`子句则是一种布尔判断，它不关心子查询返回什么，只关心是否存在匹配的记录。

```sql
SELECT * FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);
```

执行逻辑：

1. 外层表 users 每扫描一行；
2. 立刻执行子查询，用当前 users.id 去 orders 表里查；
3. 一旦找到匹配行（WHERE o.user_id = u.id），立即返回 TRUE，不再继续扫描；
4. 如果找不到，则返回 FALSE。

✅ 因此，EXISTS 可以利用索引高效查找，并且只要找到第一条匹配记录就停止，非常节省资源。

| 特性        | `IN`          | `EXISTS` |
|-----------|---------------|----------|
| 是否缓存子查询结果 | 是（可能会物化）      | 否        |
| 是否逐行判断    | 否（一次性比对）      | 是（逐行匹配）  |
| 能否利用索引    | 不一定（取决于子查询优化） | 通常能利用索引  |
| 大结果集性能    | 较差            | 较优       |
| 小结果集性能    | 差异不大          | 差异不大     |

## MySQL 修改表结构

### 不同操作的具体影响

| 操作类型                                         | 是否锁表                    | 是否重建表  | 影响分析                                                         |
|----------------------------------------------|-------------------------|--------|--------------------------------------------------------------|
| **新增字段（ALTER TABLE ADD COLUMN）**             | 一般锁表短时间（Online DDL 可减少） | ✅ 可能重建 | 如果新增列放在表末尾，MySQL 8.0 的 Online DDL 可避免重建；但如果放在中间或指定位置，需要重建整表。 |
| **修改字段类型（ALTER TABLE MODIFY COLUMN）**        | ✅ 锁表时间较长                | ✅ 必须重建 | 数据需转换并重写，I/O 压力大。对大表非常慢。                                     |
| **删除字段（ALTER TABLE DROP COLUMN）**            | ✅ 可能重建                  | ✅ 重建   | 删除列会导致行格式变化，需复制整表。                                           |
| **调整字段顺序（ALTER TABLE MODIFY ... AFTER ...）** | ✅                       | ✅      | 表结构重新定义，完全重建表，成本最高。                                          |

| 操作类型               | 是否重建表       | 锁表风险 | 推荐做法                                 |
|--------------------|-------------|------|--------------------------------------|
| `VARCHAR` 扩大长度     | 否（In-place） | 低    | 直接执行或指定 `ALGORITHM=INPLACE`          |
| `VARCHAR` 缩小长度     | 是           | 高    | 使用 pt-online-schema-change  或 gh-ost |
| `DECIMAL` 修改精度/小数位 | 是           | 高    | 使用 pt-online-schema-change 或 gh-ost  |
| 多列联合修改             | 可能是         | 中等   | 分步修改、先确认执行计划                         |

### 推荐方案

| 环境规模                     | 推荐方案                             | 说明       |
|--------------------------|----------------------------------|----------|
| 中小表（< 1000 万）            | Online DDL                       | 简单、安全    |
| 大表（上亿数据）                 | pt-online-schema-change 或 gh-ost | 生产首选     |
| 云数据库（如 AWS RDS, 阿里云 RDS） | 官方 Online DDL / gh-ost           | 云厂商一般已优化 |
| 低峰维护窗口                   | 暂停写入，直接 ALTER                    | 可接受短暂停机  |

### 总结：

| 操作       | 是否安全  | 推荐方式       | 备注          |
|----------|-------|------------|-------------|
| 新增字段（末尾） | ✅ 安全  | Online DDL | 快速且影响小      |
| 修改字段类型   | ⚠️ 慎重 | gh-ost     | 必然重建表       |
| 删除字段     | ⚠️ 慎重 | gh-ost     | 必然重建表       |
| 调整字段顺序   | ❌ 禁止  | 重新建表       | 成本高且无实际性能收益 |


## `NOT IN`

`NOT IN`不一定会全表扫描，但很多情况下确实会触发全表扫描，这要看查询条件、索引设计、以及`NOT IN`的值集来源。下面详细说明。
