# B+TREE数据结构与索引基本原理

## Explain 命令

### 最简单的一种场景

Explain 用来查看具体的执行计划

    (root@localhost) [dbt3]> show create table orders\G
    *************************** 1. row ***************************
          Table: orders
    Create Table: CREATE TABLE `orders` (
      `o_orderkey` int(11) NOT NULL,
      `o_custkey` int(11) DEFAULT NULL,
      `o_orderstatus` char(1) DEFAULT NULL,
      `o_totalprice` double DEFAULT NULL,
      `o_orderDATE` date DEFAULT NULL,
      `o_orderpriority` char(15) DEFAULT NULL,
      `o_clerk` char(15) DEFAULT NULL,
      `o_shippriority` int(11) DEFAULT NULL,
      `o_comment` varchar(79) DEFAULT NULL,
      PRIMARY KEY (`o_orderkey`),              <--主键索引
      KEY `i_o_orderdate` (`o_orderDATE`),     <--普通索引
      KEY `i_o_custkey` (`o_custkey`)          <--普通索引
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
    1 row in set (0.00 sec)

查看语句的执行计划使用了哪一个索引：

    (root@localhost) [dbt3]> select * from orders where o_orderkey = 1\G
    *************************** 1. row ***************************
        o_orderkey: 1
          o_custkey: 36901
      o_orderstatus: O
      o_totalprice: 173665.47
        o_orderDATE: 1996-01-02
    o_orderpriority: 5-LOW
            o_clerk: Clerk#000000951
    o_shippriority: 0
          o_comment: blithely final dolphins solve-- blithely blithe packages nag blith
    1 row in set (0.00 sec)

    (root@localhost) [dbt3]> explain select * from orders where o_orderkey = 1\G
    *************************** 1. row ***************************
              id: 1
      select_type: SIMPLE
            table: orders
      partitions: NULL
            type: const
    possible_keys: PRIMARY
              key: PRIMARY         <--这里用到了主键索引
          key_len: 4
              ref: const
            rows: 1
        filtered: 100.00
            Extra: NULL
    1 row in set, 1 warning (0.00 sec)

普通索引查询：

    (root@localhost) [dbt3]> explain select * from orders where o_orderdate = '1996-01-02'\G
    *************************** 1. row ***************************
              id: 1
      select_type: SIMPLE
            table: orders
      partitions: NULL
            type: ref
    possible_keys: i_o_orderdate
              key: i_o_orderdate    <--用到了普通二级索引，相对主键索引查询是要比较慢的，因为还要往后进行扫描
          key_len: 4
              ref: const
            rows: 637
        filtered: 100.00
            Extra: NULL
    1 row in set, 1 warning (0.00 sec)

    (root@localhost) [dbt3]> select * from orders where o_orderdate = '1996-01-02';
    ...
    ...
    637 rows in set (0.03 sec)

没有索引时候的表现：

    # 先删除索引，删除索引是秒级别的，不需要使用 pt-online-schema 这样的工具去搞
    (root@localhost) [dbt3]> alter table orders drop index i_o_orderdate;
    Query OK, 0 rows affected (0.01 sec)
    Records: 0  Duplicates: 0  Warnings: 0

    # 再次查看执行计划
    (root@localhost) [dbt3]> explain select * from orders where o_orderdate = '1996-01-02'\G
    *************************** 1. row ***************************
              id: 1
      select_type: SIMPLE
            table: orders
      partitions: NULL
            type: ALL
    possible_keys: NULL
              key: NULL         <--没用用到索引
          key_len: NULL
              ref: NULL
            rows: 1488824       <--扫描的行数多了不止一点点
        filtered: 10.00
            Extra: Using where
    1 row in set, 1 warning (0.00 sec)

    (root@localhost) [dbt3]> select * from orders where o_orderdate = '1996-01-02';
    ...
    ...
    637 rows in set (1.53 sec) <--慢了很多 1.53/0.03 倍

由于没用使用到索引，这条SQL语句会被记录到慢查询日志当中：

    # Time: 2022-03-14T17:47:45.903876+08:00
    # User@Host: root[root] @ localhost []  Id:     3
    # Query_time: 1.539316  Lock_time: 0.004656 Rows_sent: 637 <--返回了 637 条记录  Rows_examined: 1500000           <--共检查了 150 万行记录
    use dbt3;
    SET timestamp=1647251265;
    select * from orders where o_orderdate = '1996-01-02';

再次提醒，线上大部分调优的工作，都是看 slow.log，调优的过程基本就是把 slow.log 里的慢查询 SQL 语句找出来去 MySQL 命令行中加上 explain 去查看，然后根据情况进行调优

把索引加回来：

    (root@localhost) [dbt3]> alter table orders add index i_o_orderdate(o_orderdate);
    Query OK, 0 rows affected (4.37 sec)  <--150 万行记录的表用了 4.37 秒，可以记住来当作一个指标，根据服务器环境不同会有所上下浮动
    Records: 0  Duplicates: 0  Warnings: 0

## mysqldumpslow 命令

把对表的一些操作给格式化，这个格式化就是说在日志文件中会有很多只是查询条件内容不同，其他部分都相同的 SQL 语句，把它们查询条件内容部分格式化成某个字符，比如：

    select * from orders where o_orderdate = '1996-01-02';
    select * from orders where o_orderdate = '1996-02-02';
    select * from orders where o_orderdate = '1996-03-02';

    # 会格式化归类为一条语句
    select * from orders where o_orderdate = 'S'

又比如将 SQL 语句中的 LIMIT 10 变成 LIMIT N

    $ mysqldumpslow slow.log

    Count: 5  Time=4.00s (19s)  Lock=0.00s (0s)  Rows=4000.0 (20000), root[root]@localhost
      SELECT
      *
      FROM
      lineitem
      ORDER BY l_discount
      LIMIT N             <--这里进行了格式化

这个命令默认是根据执行时间 Query time 来进行排序的，可以修改，具体命令可以查看此命令的帮助：

    $ mysqldumpslow --help

这个命令对于解析大的(几个G) slow.log 依然是很慢的，这时候可以用到采样

只要使用了慢查询日志，记录到里面的所有查询都是重复的，既然是重复的话只要用 [tail](https://www.runoob.com/linux/linux-comm-tail.html) 命令采样靠后一些的内容来分析就可以了

    $ tail -n 100000 slow.log > analytics.log
    $ mysqldumpslow analytics.log

处理完慢日志查询后还要做一件事情，就是清理日志，以便确认慢查询会不会再次被记录进来，日志不推荐直接删除，如何清理日志前面有提及(mv slow.log 备份文件然后在 MySQL 中 flush slow logs)

接着如果调优基本上都完成的话，slow.log 应该是增长的非常慢的或者说基本上是不增长，如果再有增长的话就去 slow.log 里看

**这一套下来基本上可以解决线上百分之八九十的问题！**

## MySQL 5.7 中的 sys 库介绍 

### statement_analysis 表

这个库中的 statement_analysis 表提供的数据比 slow.log 要更加直观，因为可以对很多维度来进行重新的排序，比如：

* 想知道哪条 SQL 语句查询的行数是最多的
* 哪条 SQL 语句被锁住的时间是最多的
* 哪条 SQL 语句最大执行时间是多少

示例：

    (root@localhost) [(none)]> use sys;
    Database changed
    (root@localhost) [sys]> select * from statement_analysis limit 3\G
    *************************** 1. row ***************************
    ......
    *************************** 2. row ***************************
                query: SELECT * FROM `orders` WHERE `o_orderdate` = ? <--把之前的语句记录下来并且进行了格式化
                  db: dbt3
            full_scan: *
          exec_count: 5          <--执行的次数是 5 次
            err_count: 0
          warn_count: 0
        total_latency: 5.35 s    <--总执行时间
          max_latency: 1.53 s    <--最大执行时间
          avg_latency: 1.07 s
        lock_latency: 4.07 s
            rows_sent: 637       <--返回的行数
        rows_sent_avg: 637
        rows_examined: 1500000
    rows_examined_avg: 1500000
        rows_affected: 0
    rows_affected_avg: 0
          tmp_tables: 0
      tmp_disk_tables: 0
          rows_sorted: 0
    sort_merge_passes: 0
              digest: 81a3f642e19c6f2defd29d51a9fdb11d
          first_seen: 2022-03-14 17:47:46
            last_seen: 2022-03-14 17:47:46
    *************************** 3. row ***************************
    ......

有参数可以控制这张表一共可以保存多少行的记录，默认基本就够用了

x$ 开头的 statement_analysis 表内数据除了 SQL 语句其他的不提供格式化，这样做就可以对这张表每秒钟进行采集，对这些数据做差值，可以计算出一个波段的增长量，可以每过一段时间做一个增量的采集

    (root@localhost) [sys]> select * from x$statement_analysis\G
    *************************** 11. row ***************************
                query: SELECT SYSTEM_USER ( )
                  db: NULL
            full_scan:
          exec_count: 4
            err_count: 0
          warn_count: 0
        total_latency: 257761000 <--并没有格式化
          max_latency: 88826000  <--并没有格式化
          avg_latency: 64440000  <--并没有格式化
        lock_latency: 0
            rows_sent: 4
        rows_sent_avg: 1
        rows_examined: 0
    rows_examined_avg: 0
        rows_affected: 0
    rows_affected_avg: 0
          tmp_tables: 0
      tmp_disk_tables: 0
          rows_sorted: 0
    sort_merge_passes: 0
              digest: 80b5982aee134a094ef74dffb2986f27
          first_seen: 2022-03-14 17:47:39
            last_seen: 2022-03-14 18:24:17
    11 rows in set (0.00 sec)

statement_analysis 这张表其实是一张视图，从 performance_schema 里面的 events_statements_summary_by_digest 这张表中根据总的等待时间从大到小排序然后取得的

总体来说，sys 库下面的所有表可以认为都是视图，是用来方便进行统计的

基于其他维度统计的一些表：

* statements_with_errors_or_warnings 报 error 和 warning 的 SQL 语句有哪些
* statements_with_full_table_scans   全表扫面或者没走索引的
* statements_with_sorting            带有排序的
* statements_with_temp_tables        带有临时表的

以前需要通过 slow.log 进行优化的手段，现在都可以通过 sys 库中的这几张表获得一些汇总信息，而 slow.log 则是一条条的记录

MySQL5.6，5.7 开始 slow.log 有没有那种重要，或许还不一定。如果是找线上哪些是平均慢了的可以找 sys 库，想找某个时间点可以找 slow.log

MySQL5.6 并没有 sys 库，但是都是通过 performance_schema 中提取出来的，可以在一下链接中找到 sys_56.sql 文件生成视图

https://github.com/mysql/mysql-sys

### schema_index_statistics 表

用来查看每个索引的使用情况，通过查看能知道哪张表哪个索引是比较活跃的

    (root@localhost) [sys]> select * from schema_index_statistics limit 10\G
    *************************** 3. row ***************************
    ......
    *************************** 4. row ***************************
      table_schema: dbt3          <--这个库
        table_name: orders        <--这张表
        index_name: i_o_orderdate <--对应的这个索引
    rows_selected: 0              <--执行的次数是多少
    select_latency: 0 ps          <--查询的时间是多少
    rows_inserted: 0              <--往下看，增删查改都有
    insert_latency: 0 ps          <--增
      rows_updated: 0
    update_latency: 0 ps
      rows_deleted: 0
    delete_latency: 0 ps
    4 rows in set (0.00 sec)

## 一些例行巡检脚本的实现

### 通过一条 SQL 语句把当前线上所有的表都查一遍，然后找出没有主键的表

通过对 information_schema 库中的 STATISTICS 和 TABLES 表进行关联查询可以实现：

    (root@localhost) [information_schema]> desc STATISTICS;
    +---------------+---------------+------+-----+---------+-------+
    | Field         | Type          | Null | Key | Default | Extra |
    +---------------+---------------+------+-----+---------+-------+
    | TABLE_CATALOG | varchar(512)  | NO   |     |         |       |
    | TABLE_SCHEMA  | varchar(64)   | NO   |     |         |       |   <--通过这3个配合实现，库名
    | TABLE_NAME    | varchar(64)   | NO   |     |         |       |   <--通过这3个配合实现，表名
    | NON_UNIQUE    | bigint(1)     | NO   |     | 0       |       |
    | INDEX_SCHEMA  | varchar(64)   | NO   |     |         |       |
    | INDEX_NAME    | varchar(64)   | NO   |     |         |       |   <--通过这3个配合实现，索引名称
    | SEQ_IN_INDEX  | bigint(2)     | NO   |     | 0       |       |
    | COLUMN_NAME   | varchar(64)   | NO   |     |         |       |
    | COLLATION     | varchar(1)    | YES  |     | NULL    |       |
    | CARDINALITY   | bigint(21)    | YES  |     | NULL    |       |
    | SUB_PART      | bigint(3)     | YES  |     | NULL    |       |
    | PACKED        | varchar(10)   | YES  |     | NULL    |       |
    | NULLABLE      | varchar(3)    | NO   |     |         |       |
    | INDEX_TYPE    | varchar(16)   | NO   |     |         |       |
    | COMMENT       | varchar(16)   | YES  |     | NULL    |       |
    | INDEX_COMMENT | varchar(1024) | NO   |     |         |       |
    +---------------+---------------+------+-----+---------+-------+
    16 rows in set (0.00 sec)

    (root@localhost) [information_schema]> SELECT 
        *
    FROM
        information_schema.TABLES t
            LEFT JOIN
        information_schema.STATISTICS s ON t.table_schema = s.table_schema
            AND t.table_name = s.table_name
            AND s.index_name = 'PRIMARY'
    WHERE
        t.table_schema NOT IN ('mysql' , 'performance_schema',
            'information_schema',
            'sys')
            AND t.table_type = 'BASE TABLE'
            AND s.index_name IS NULL;

### 找出索引区分度小于 10% 的 SQL

    (root@localhost) [information_schema]> SELECT 
        CONCAT(t.TABLE_SCHEMA, '.', t.TABLE_NAME) table_name,
        INDEX_NAME,
        CARDINALITY,
        TABLE_ROWS,
        CARDINALITY / TABLE_ROWS AS SELECTIVITY
    FROM
        information_schema.TABLES t,
        (SELECT 
            table_schema, table_name, index_name, cardinality
        FROM
            information_schema.STATISTICS
        WHERE
            (table_schema , table_name, index_name, seq_in_index) IN (SELECT 
                    table_schema, table_name, index_name, seq_in_index
                FROM
                    information_schema.STATISTICS
                WHERE
                    seq_in_index = 1)) s
    WHERE
        t.table_schema = s.table_schema
            AND t.table_name = s.table_name
            AND t.table_rows != 0
            AND t.table_schema NOT IN ('mysql' , 'performance_schema',
            'information_schema',
            'sys')
    ORDER BY SELECTIVITY;

### 查询从来没有被使用过的索引

    (root@localhost) [sys]> desc schema_unused_indexes;
    +---------------+-------------+------+-----+---------+-------+
    | Field         | Type        | Null | Key | Default | Extra |
    +---------------+-------------+------+-----+---------+-------+
    | object_schema | varchar(64) | YES  |     | NULL    |       |
    | object_name   | varchar(64) | YES  |     | NULL    |       |
    | index_name    | varchar(64) | YES  |     | NULL    |       |
    +---------------+-------------+------+-----+---------+-------+
    3 rows in set (0.00 sec)
    
    (root@localhost) [sys]> select * from schema_unused_indexes;

## 索引的第一个作用可以用来快速定位，第二个作用可以用来加速 order by

如果要排序的话可以把 sort_buffer_size 参数值调大

    (root@localhost) [dbt3]> select * from orders order by o_totalprice desc limit 10;
    ...
    ...
    ...
    10 rows in set (0.89 sec)

    通过 explain 查看执行步骤：

    (root@localhost) [dbt3]> explain select * from orders order by o_totalprice desc limit 10\G
    *************************** 1. row ***************************
              id: 1
      select_type: SIMPLE
            table: orders
      partitions: NULL
            type: ALL
    possible_keys: NULL
              key: NULL                <--没有使用到任何索引
          key_len: NULL
              ref: NULL
            rows: 1493376
        filtered: 100.00
            Extra: Using filesort      <--发现一个Using filesort，表示需要排序，此时通过调大sort_buffer_size参数是有帮助的
    1 row in set, 1 warning (0.00 sec)

如果对这个列创建了一个索引可以加速 order by 操作：

    (root@localhost) [dbt3]> alter table orders add index idx_o_totalprice(o_totalprice);
    Query OK, 0 rows affected (4.70 sec)
    Records: 0  Duplicates: 0  Warnings: 0

    (root@localhost) [dbt3]> explain select * from orders order by o_totalprice desc limit 10\G
    *************************** 1. row ***************************
              id: 1
      select_type: SIMPLE
            table: orders
      partitions: NULL
            type: index
    possible_keys: NULL
              key: idx_o_totalprice     <--添加索引之后order by操作现在可以使用到创建的索引
          key_len: 9
              ref: NULL
            rows: 10
        filtered: 100.00
            Extra: NULL                 <--并且在Extra这边就没有了Using filesort，此时不会使用到排序的内存了，就不需要再去调大sort_buffer_size参数了
    1 row in set, 1 warning (0.00 sec)

    (root@localhost) [dbt3]> select * from orders order by o_totalprice desc limit 10;
    ...
    ...
    ...
    10 rows in set (0.00 sec)