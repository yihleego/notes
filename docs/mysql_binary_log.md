# Binary Log

Binary Log 是 MySQL 服务层产生的日志，常用来进行数据恢复、数据库复制，常见的 MySQL 主从架构，就是采用 slave 同步 master 的 Binary Log 实现的。
另外通过解析 Binary Log 能够实现 MySQL 到其他数据源的数据复制，如：Elasticsearch、Redis 等。

## Binary Log 格式

Binary Log 中记录的事件格式支持三种类型：基于行的日志记录（row-based logging）、基于语句的日志记录（statement-based logging）和混合的日志记录（mixed-based logging）。

### 基于行的日志记录（默认）

将数据源事件写入 Binary Log，表示单个表的行是如何受到影响。

- 优点：不受存储过程、函数、触发器等影响，节约`IO`，提⾼性能。
- 缺点：会产⽣⼤量的⽇志，尤其是`ALTER TABLE`操作会使⽇志暴涨。

### 基于语句的日志记录

将原始 SQL 语句将写入 Binary Log。

- 优点：相比基于行的日志记录，减少了日志量，
- 缺点：基于语句的复制，复制非确定性语句可能会出现问题。

对于基于语句的复制，复制非确定性语句可能会出现问题。
在确定给定语句对于基于语句的复制是否安全时，MySQL 确定它是否可以保证可以使用基于语句的日志记录来复制该语句。
如果 MySQL 不能做出这种保证，它会将语句标记为可能不可靠并发出警告，Statement may not be safe to log in statement 格式。

### 混合的日志记录

对于混合日志，默认使用基于语句的日志，但在某些情况下，日志模式会自动切换为基于行的日志。

服务器会在以下情况下自动从基于语句的日志切换到基于行的日志记录：

- 存在`UUID()`函数时
- 存在一个或多个具有`AUTO_INCREMENT`列的表被更新，且调用了触发器或存储函数时。与所有其他不安全语句一样，如果`binlog_format = STATEMENT`，则会生成警告
- 涉及对可加载函数的调用时
- 使用`FOUND_ROWS()`或`ROW_COUNT()`时
- 使用`USER()`、`CURRENT_USER()`或`CURRENT_USER`时
- 涉及的表之一是 MYSQL 数据库中的日志表时
- 使用`LOAD_FILE()`函数时
- 一个语句引用一个或多个系统变量时：
    - auto_increment_increment
    - auto_increment_offset
    - character_set_client
    - character_set_connection
    - character_set_database
    - character_set_server
    - collation_connection
    - collation_database
    - collation_server
    - foreign_key_checks
    - identity
    - last_insert_id
    - lc_time_names
    - pseudo_thread_id
    - sql_auto_is_null
    - time_zone
    - timestamp
    - unique_checks

## 设置 Binary Log 格式

启动 MySQL 服务器时可以通过使用`--binlog-format=type`来显式选择二进制日志记录格式。

- `ROW`：使日志记录是基于行的，这是默认设置。
- `STATEMENT`：使日志记录是基于语句的。
- `MIXED`：使日志记录使用混合格式。

在启用 Binary Log 格式前需要保证服务器启用了 Binary Log 记录，即系统变量`--log-bin=ON`时。
从 MySQL 8.0 开始，Binary Log 默认启用，只有在启动时指定`--skip-log-bin`或`--disable-log-bin`选项时才会禁用。

日志格式也可以在运行时切换，设置`binlog_format`系统变量的全局值以指定更改后连接的客户端的格式：

```sql
mysql> SET GLOBAL binlog_format = 'STATEMENT';
mysql> SET GLOBAL binlog_format = 'ROW';
mysql> SET GLOBAL binlog_format = 'MIXED';
```
