# B+TREE 高级索引

## CARDINALITY

这个值就是代表什么时候创建索引是比较有效的

* 唯一记录不重复数据的数量，数值越大重复数据越多
* 高选择性的
* B+ 树是从大量的不同数据里面去找出一小部分数据
  * select * from student where sex = 'M'
    * 低选择性，性别这种属性重复的非常多
  * select * from student where category = 'fruit'
    * 低选择性，只根据类别去查询的场景很少，如果和另一个字段组成符合索引进行查询，那么这时候可以为分类创建索引
  * select * from student where name = 'Neo'
    * 高选择性，姓名这种属性重复的概率小

在某个列上创建索引的原则就是它是高选择性的，也就是 CARDINALITY 的值是比较高的

通过元数据表查看索引选择度：

    (root@localhost) [information_schema]> desc STATISTICS;
    +---------------+---------------+------+-----+---------+-------+
    | Field         | Type          | Null | Key | Default | Extra |
    +---------------+---------------+------+-----+---------+-------+
    | ...           | varchar(1)    | YES  |     | NULL    |       |
    | CARDINALITY   | bigint(21)    | YES  |     | NULL    |       |  <--
    | ...           | varchar(1024) | NO   |     |         |       |
    +---------------+---------------+------+-----+---------+-------+
    16 rows in set (0.00 sec)

    (root@localhost) [information_schema]> select * from STATISTICS where table_schema='dbt3' limit 1\G
    *************************** 1. row ***************************
    TABLE_CATALOG: def
    TABLE_SCHEMA: dbt3
      TABLE_NAME: customer
      NON_UNIQUE: 0
    INDEX_SCHEMA: dbt3
      INDEX_NAME: PRIMARY        <--主键索引
    SEQ_IN_INDEX: 1
      COLUMN_NAME: c_custkey
        COLLATION: A
      CARDINALITY: 148296        <--CARDINALITY值
        SUB_PART: NULL
          PACKED: NULL
        NULLABLE:
      INDEX_TYPE: BTREE
          COMMENT:
    INDEX_COMMENT:
    1 row in set (0.00 sec)

通过 TABLES 表查看这张表一共有多少记录：

    (root@localhost) [information_schema]> select * from tables where table_schema='dbt3' and table_name='customer' limit 1\G
    *************************** 1. row ***************************
      TABLE_CATALOG: def
      TABLE_SCHEMA: dbt3
        TABLE_NAME: customer
        TABLE_TYPE: BASE TABLE
            ENGINE: InnoDB
            VERSION: 10
        ROW_FORMAT: Dynamic
        TABLE_ROWS: 148296       <--预估行数不是精确的值，这里因为是主键索引，所以和上面的CARDINALITY值相等
    AVG_ROW_LENGTH: 201
        DATA_LENGTH: 29949952
    MAX_DATA_LENGTH: 0
      INDEX_LENGTH: 3686400
          DATA_FREE: 0
    AUTO_INCREMENT: NULL
        CREATE_TIME: 2022-05-16 18:26:08
        UPDATE_TIME: NULL
        CHECK_TIME: NULL
    TABLE_COLLATION: latin1_swedish_ci
          CHECKSUM: NULL
    CREATE_OPTIONS:
      TABLE_COMMENT:
    1 row in set (0.00 sec)

通过这 2 张表就可以通过 STATISTICS 表中的 CARDINALITY 值去除以 TABLES 表中的 TABLE_ROWS，如果结果是比较大的就是比较接近于 1 的话，那么就是高选择性的，反之则是低选择性的

通过这个特性又可以实现一种例行巡检脚本，找出可能创建的索引是有问题的索引，也就是说选择度比较低的索引（CARDINALITY/TABLE_ROWS 小于 0.1 的），这个也可以叫做索引区分度

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
    -- HAVING SELECTIVITY < 0.1 -- 去掉注释可以删选出小于0.1的数据
    ORDER BY SELECTIVITY;

## 复合索引

复合索引就是对多个列组合起来创建索引，原理和单索引的原理一样，只不过存放的 Key 变成了由多个索引组成的一个 Key

对于 (a,b) 组成的符合索引，其中 a 是排序过的，根据 (a,b) 也已经是排序过的，但是对 (a,b) 排序了不一定代表对 b 也进行了排序

对于复合索引可以用于这些 SQL 语句：

    SELECT * FROM t WHERE a = ?           因为复合索引中的 a 经过排序
    SELECT * FROM t WHERE a = ? and b= ?  因为复合索引中的 ab 经过排序

不能用于的 SQL 语句：
    
    SELECT * FROM t WHERE b = ?           因为复合索引中的 b 没有经过排序

使 Sort(a,b) 不依赖 Using filesort：

    SELECT * FROM t WHERE a=? order by b 
    
WHERE a=? order by b 的用法是复合索引用得比较多的场景，比如查询用户叫 neo 的订单信息，并且根据下单信息列出最近下单的 N 条记录，如果只对 a 做索引，b 依旧会用到受 sort_buffer_size 参数影响的 Using filesort ，Using filesort 对 cpu 的开销会比较大

对这样的语句线上很可能就只对 a 创建了索引，所以这里也可以根据这个做一个例行巡检脚本，通过 sys 库中的 statements_with_sorting 表中的 exec_count 字段可以发现这个问题

    (root@localhost) [sys]> select * from statements_with_sorting limit 3\G
    *************************** 1. row ***************************
                query: SELECT * FROM `orders` ORDER BY `o_totalprice` DESC LIMIT ?
                  db: dbt3
          exec_count: 3                     <--排序次数
        total_latency: 8.03 s               <--执行时间
    sort_merge_passes: 0
      avg_sort_merges: 0
    sorts_using_scans: 2
    sort_using_range: 0
          rows_sorted: 20
      avg_rows_sorted: 7
          first_seen: 2022-05-24 15:43:23
            last_seen: 2022-05-24 15:47:43
              digest: 8e1b189b82809ad7b6b48706f99d84d2
    *************************** 2. row ***************************
                query: SELECT `test` . `Orders` . `pr ... n_extract` ( `data` , ? ) ) )
                  db: information_schema
          exec_count: 5                     <--排序次数
        total_latency: 124.62 ms            <--执行时间
    sort_merge_passes: 0
      avg_sort_merges: 0
    sorts_using_scans: 5
    sort_using_range: 0
          rows_sorted: 162
      avg_rows_sorted: 32
          first_seen: 2022-05-24 16:29:20
            last_seen: 2022-05-24 16:32:49
              digest: 6affb5ed8ab97d800d60cadbb0e03ede
    2 rows in set (0.01 sec)

### 复合索引创建规则

对于 (a,b,c) 这样的复合索引，也只是对 a 进行了排序，那么如果碰到 3 个列组成的复合索引该怎么做呢

比如说现在有 3 个列 a b c 组成的索引，对于这种典型的 SQL 查询条件：

    a=? and b=? order by c
    或者是
    a=? and b=? and c=?

这时候有很多种创建复合索引的排列组合，比如 (abc)，(acb)，(bac) 等，那么这时该如何创建复合索引

创建复合索引的时候，有一个规则就是要把选择度高的放在前面，比如 a 的选择度(CARDINALITY值)是 80，b 的是 20，c 的是 70，那么这个复合索引的排列从高到低就应该是 (a,c,b)

需要注意的是，要不要在类型这种字段上去创建索引呢，可能是不需要要的，但是如果说前面有高选择都的字段给它组合成了一个复合索引的话，这时候对类型字段来说或许是适合的

#### 查看复合索引是否是高效的

@todo 找出设计不怎么合理的索引，也就是CARDINALITY值比较低的索引，注意abc abc的CARDINALITY值加起来其实都是相等的

可以通过 information_schema.STATISTICS 这张表查看 (a,b,c) 组成的索引是否是高效的

对于有 2 个列所组成的复合索引，会找出有 2 个列：

    (root@localhost) [information_schema]> select * from STATISTICS where INDEX_NAME = 'i_l_orderkey_quantity'\G
    *************************** 1. row ***************************
    TABLE_CATALOG: def
    TABLE_SCHEMA: dbt3
      TABLE_NAME: lineitem
      NON_UNIQUE: 1
    INDEX_SCHEMA: dbt3
      INDEX_NAME: i_l_orderkey_quantity
    SEQ_IN_INDEX: 1                 <--表示索引的第1个列
      COLUMN_NAME: l_orderkey
        COLLATION: A
      CARDINALITY: 1380678          <--第1个列的CARDINALITY值
        SUB_PART: NULL
          PACKED: NULL
        NULLABLE:
      INDEX_TYPE: BTREE
          COMMENT:
    INDEX_COMMENT:
    *************************** 2. row ***************************
    TABLE_CATALOG: def
    TABLE_SCHEMA: dbt3
      TABLE_NAME: lineitem
      NON_UNIQUE: 1
    INDEX_SCHEMA: dbt3
      INDEX_NAME: i_l_orderkey_quantity
    SEQ_IN_INDEX: 2                 <--表示索引的第2个列
      COLUMN_NAME: l_quantity
        COLLATION: A
      CARDINALITY: 5319436          <--第1个列和第2个列组合起来的CARDINALITY值
        SUB_PART: NULL
          PACKED: NULL
        NULLABLE: YES
      INDEX_TYPE: BTREE
          COMMENT:
    INDEX_COMMENT:
    2 rows in set (0.00 sec)

### 索引覆盖

通常来说复合索引都是二级索引，对于二级索引的叶子节点保存的是键值和主键值，对于 a b c d 四个列的索引，假设 d 是用的主键索引，那么由 a b 创建的复合索引，它的叶子节点中就会保存 (a b d)，其中 d 是主键值，并且 (a b d) 是排序的

索引覆盖的意思是如果二级复合索引中包含了主键值，对于下面的这些查询不需要回表：

* (primary key1, primary key2, ..., key1, key2, ...)
  * SELECT key2 FROM table WHERE key1=xxxx;
  * SELECT primary key2,key2 FROM table WHERE key1=xxx;
  * SELECT primary key1,key2 FROM table WHERE key1=xxx;
  * SELECT primary key1,primary key2, key2 FROM table WHERE key1=xxx;

虽然这种查询中会带有一些额外的列，但这些额外的列都是主键值

假设有一个复合索引由 (userid, buy_date) 组成，下面示例只用到了 buy_date，相当于上面的 (a,b) 只用到了 b 去进行查询的：

    SELECT count(1) FROM buy_log WHERE buy_date>='xxxx-xx-xx' AND buy_date<'xxxx-xx-xx';

按照上面所说的按道理是不能这样(where b=?)使用的，但是在执行计划里可以看到使用了 Using where; Using index

**Using index** 的意思是使用到了索引覆盖，通过二级索引就直接能找到数据而不需要回表，但是这条语句的最优做法是在 buy_date 创建单独的索引

索引覆盖指的面其实很广，只要是不回表取得数据的都可以叫做索引覆盖，索引覆盖也不一定就是效率高的

注意如果是 (a,b) 这样的复合索引，如果需要根据 a 进行查询，那么需不需要单独创建 a 的索引来提高效率呢，个人觉得是不需要的，这两个索引从高度上来说也是差不多的，即使数据量很大效率方面也不会有太大的变化，但是创建两个索引对 DML 操作维护方面的代价就大多了，所以这时候并不需要去创建 a 的索引

对于创建了 (a,b) 复合索引，又创建了 a 索引的，将 a 叫做**冗余索引**，通过 sys 库的 schema_redundant_indexes 表可以找出冗余索引，又是可以作为例行巡检的

    (root@localhost) [sys]> select * from schema_redundant_indexes limit 1\G
    *************************** 1. row ***************************
                  table_schema: dbt3
                    table_name: lineitem
          redundant_index_name: i_l_orderkey          <--冗余的索引名
      redundant_index_columns: l_orderkey             <--冗余的索引列
    redundant_index_non_unique: 1
          dominant_index_name: i_l_orderkey_quantity  <--占据主要的索引，根据orderkey和quantity列所组成的复合索引
        dominant_index_columns: l_orderkey,l_quantity <--占据主要的索引列
    dominant_index_non_unique: 1
                subpart_exists: 0
                sql_drop_index: ALTER TABLE `dbt3`.`lineitem` DROP INDEX `i_l_orderkey` <--修复语句
    1 row in set (0.01 sec)

### 索引不可见 (MySQL8.0 适用)

通过 sys 库中的 schema_unused_indexes 查看到一些没有使用到的索引

    (root@localhost) [sys]> select * from schema_unused_indexes;
    +---------------+--------------+-----------------------+
    | object_schema | object_name  | index_name            |
    +---------------+--------------+-----------------------+
    | dbt3          | customer     | i_c_nationkey         |
    | dbt3          | lineitem     | i_l_commitdate        |
    | dbt3          | lineitem     | i_l_partkey           |
    | dbt3          | lineitem     | i_l_orderkey_quantity |
    | dbt3          | lineitem     | i_l_suppkey_partkey   |
    | dbt3          | lineitem     | i_l_orderkey          |
    | dbt3          | lineitem     | i_l_shipdate          |
    | dbt3          | lineitem     | i_l_receiptdate       |
    | dbt3          | lineitem     | i_l_suppkey           |
    | dbt3          | nation       | i_n_regionkey         |
    | dbt3          | orders       | i_o_orderdate         |
    | dbt3          | orders       | i_o_custkey           |
    | dbt3          | partsupp     | i_ps_partkey          |
    | dbt3          | partsupp     | i_ps_suppkey          |
    | dbt3          | supplier     | i_s_nationkey         |
    | employees     | dept_emp     | dept_no               |
    | employees     | dept_manager | dept_no               |
    | employees     | employees    | idx_birth_date        |
    | test          | a            | idx_a                 |
    | test          | UserJson     | idx_name              |
    +---------------+--------------+-----------------------+
    20 rows in set (0.01 sec)

是否应该删除这些索引呢？虽然这些索引暂时没有用到，但是也保不准可能将来会用到

在 8.0 中可以将这些索引设置成不可见的，优化器就不会使用不可见索引，跑一段时间再去观察一下到底对业务有没影响，如果没有影响的话再把这些索引删掉

    ALTER TABLE xxx ALTER INDEX index_name INVISIBLE; -- 不可见
    ALTER TABLE xxx ALTER INDEX index_name VISIBLE;   -- 可见

这个功能可能还是蛮有用的，但是不是硬需求

### 降序锁 非常有用 (MySQL8.0 适用)

MySQL 目前存在的一个问题

    (root@localhost) [dbt3]> EXPLAIN SELECT * FROM orders WHERE o_custkey = 1 ORDER BY o_orderDate,o_orderStatus\G
    *************************** 1. row ***************************
      ...
            Extra: Using index condition; Using filesort <--在使用Using filesort，所以可以为这张表创建多列索引
    1 row in set, 1 warning (0.00 sec)

创建多列索引：

    (root@localhost) [dbt3]> alter table orders add index idx_a_b_c(o_custkey,o_orderDate,o_orderStatus);
    Query OK, 0 rows affected (4.82 sec)
    Records: 0  Duplicates: 0  Warnings: 0

    (root@localhost) [dbt3]> EXPLAIN SELECT * FROM orders WHERE o_custkey = 1 ORDER BY o_orderDate,o_orderStatus\G
    *************************** 1. row ***************************
      ...
            Extra: Using index condition <--现在Using filesort没有了
    1 row in set, 1 warning (0.00 sec)

使用降序排列遇到的问题：

    (root@localhost) [dbt3]> EXPLAIN SELECT * FROM orders WHERE o_custkey = 1 ORDER BY o_orderDate DESC,o_orderStatus\G
    *************************** 1. row ***************************
              id: 1
      select_type: SIMPLE
            table: orders
      partitions: NULL
            type: ref
    possible_keys: i_o_custkey,idx_a_b_c,idx_cust_date_status
              key: i_o_custkey
          key_len: 5
              ref: const
            rows: 6
        filtered: 100.00
            Extra: Using index condition; Using filesort   <--改成o_orderDate DESC条件后，再次出现了Using 
    1 row in set, 1 warning (0.00 sec)

在 5.7 中创建 (A,B,C) 这样的复合索引它的排序规则都是升序的 (A asc,B asc,C asc)，如果这时 B desc (A asc,B desc,C asc) 就不能使用到这个复合索引就又要再一次进行排序(Using filesort)了，所以这是 5.7 中一个比较大的问题

但是下面这条语句是可以的，因为 B C 可以同时找到最大的值，同时逆序就没有问题：

    (root@localhost) [dbt3]> EXPLAIN SELECT * FROM orders WHERE o_custkey = 1 ORDER BY o_orderDate DESC,o_orderStatus DESC\G
    *************************** 1. row ***************************
      ...
            Extra: Using where
    1 row in set, 1 warning (0.00 sec)

简单来说只要排序的不是一个方向就都需要 Using filesort

对于这种在复合索引中排序有降序又有升序的需求，MySQL8.0 中可以创建这样的索引(o_orderDate DESC)：

    (root@localhost) [dbt3]> ALTER TABLE orders ADD INDEX idx_cust_date_status_80(o_custkey, o_orderDate DESC, o_orderStatus);
    Query OK, 0 rows affected, 1 warning (4.83 sec)
    Records: 0  Duplicates: 0  Warnings: 1

    # 在 5.7 中会报 Warnings
    (root@localhost) [dbt3]> show warnings;
    +---------+------+----------------------------------------------------------------------------------------------------------------------------------------------+
    | Level   | Code | Message
        |
    +---------+------+----------------------------------------------------------------------------------------------------------------------------------------------+
    | Warning | 1831 | Duplicate index 'idx_cust_date_status_80' defined on the table 'dbt3.orders'. This is deprecated and will be disallowed in a future release. |
    +---------+------+----------------------------------------------------------------------------------------------------------------------------------------------+
    1 row in set (0.00 sec)

### 函数索引(虚拟列)

在 5.7 中要解决降序索引的问题需要用到函数索引(虚拟列)

    ALTER TABLE tablename ADD COLUMN AS (function(column)) VIRTUAL/STORED; -- 默认VIRTUAL

每次访问虚拟列需要额外的一次计算，也可以定义成 STORED 会在磁盘上占据空间，通常来说都是用 VIRTUAL

虚拟的列添加完之后是不占用任何存储空间的，**每次访问虚拟列需要额外的一次计算得到**，但是可以在这个列上添加索引，这个添加的索引是占用空间的，索引不是虚拟的是实际存在的，这个索引叫做函数索引，因为它是一个在函数计算出来的虚拟列上创建的索引

#### 通过虚拟列为 5.7 实现降序索引

现在为 orders 表添加一个虚拟的列：

    (root@localhost) [dbt3]> ALTER TABLE orders ADD column o_orderdate2 INT AS (DATEDIFF('2099-01-01',o_orderdate)) VIRTUAL;
    Query OK, 0 rows affected (2.42 sec)
    Records: 0  Duplicates: 0  Warnings: 0

    (root@localhost) [dbt3]> desc orders;
    +-----------------+-------------+------+-----+---------+-------------------+
    | Field           | Type        | Null | Key | Default | Extra             |
    +-----------------+-------------+------+-----+---------+-------------------+
    | o_orderkey      | int(11)     | NO   | PRI | NULL    |                   |
    | o_custkey       | int(11)     | YES  | MUL | NULL    |                   |
    | o_orderstatus   | char(1)     | YES  |     | NULL    |                   |
    | o_totalprice    | double      | YES  | MUL | NULL    |                   |
    | o_orderDATE     | date        | YES  | MUL | NULL    |                   |
    | o_orderpriority | char(15)    | YES  |     | NULL    |                   |
    | o_clerk         | char(15)    | YES  |     | NULL    |                   |
    | o_shippriority  | int(11)     | YES  |     | NULL    |                   |
    | o_comment       | varchar(79) | YES  |     | NULL    |                   |
    | o_orderdate2    | int(11)     | YES  |     | NULL    | VIRTUAL GENERATED |
    +-----------------+-------------+------+-----+---------+-------------------+
    10 rows in set (0.00 sec)

新添加了一个名为 o_orderdate2， 类型为 INT，并且 AS 后跟一个函数表达式的虚拟列，这个虚拟列是通过使用函数表达式 (DATEDIFF('2099-01-01',o_orderdate)) 计算日期差得到的

这里使用日期差是因为现在需要降序的排列数据，但是现在在原来数据复合索引上是不支持降序的，那么就要想办法将降序的数据转换成升序的，这么说可能还有点不太理解，看个例子：

    # 原来的日期数据
    2000-01-01  --> DATEDIFF('2099-01-01', '2000-01-01') --> 36160
    2000-02-01  --> DATEDIFF('2099-01-01', '2000-02-01') --> 36129
    2000-03-01  --> DATEDIFF('2099-01-01', '2000-03-01') --> 36100
    2000-04-01  --> DATEDIFF('2099-01-01', '2000-04-01') --> 36069
    2000-05-01  --> DATEDIFF('2099-01-01', '2000-05-01') --> 36039

这样根据 o_orderdate2 虚拟列升序的排序其实就是原来数据降序的排列

接着对为虚拟列创建索引：

    (root@localhost) [dbt3]> ALTER TABLE orders ADD INDEX idx_cust_date_status(o_custkey,o_orderdate2,o_orderStatus);
    Query OK, 0 rows affected (6.02 sec)
    Records: 0  Duplicates: 0  Warnings: 0

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
      `o_orderdate2` int(11) GENERATED ALWAYS AS ((to_days('2099-01-01') - to_days(`o_orderDATE`))) VIRTUAL,  <--虚拟列
      PRIMARY KEY (`o_orderkey`),
      KEY `i_o_custkey` (`o_custkey`),
      KEY `idx_o_totalprice` (`o_totalprice`),
      KEY `idx_a_b_c` (`o_custkey`,`o_orderDATE`,`o_orderstatus`),
      KEY `i_o_orderdate` (`o_orderDATE`),
      KEY `idx_cust_date_status` (`o_custkey`,`o_orderdate2`,`o_orderstatus`)  <--虚拟列的索引
    ) ENGINE=InnoDB DEFAULT CHARSET=latin1
    1 row in set (0.00 sec)

将那条逆序查询的 SQL 语句：

    (root@localhost) [dbt3]> EXPLAIN SELECT * FROM orders WHERE o_custkey = 1 ORDER BY o_orderDate DESC,o_orderStatus\G

改写为：

    (root@localhost) [dbt3]> EXPLAIN SELECT * FROM orders WHERE o_custkey = 1 ORDER BY o_orderDate2,o_orderStatus\G
    *************************** 1. row ***************************
              id: 1
      select_type: SIMPLE
            table: orders
      partitions: NULL
            type: ref
    possible_keys: i_o_custkey,idx_a_b_c,idx_cust_date_status
              key: idx_cust_date_status   <--使用到了虚拟列的索引
          key_len: 5
              ref: const
            rows: 6
        filtered: 100.00
            Extra: Using where            <--Using filesort没有了，使用到了虚拟列的索引
    1 row in set, 1 warning (0.00 sec)

这样就变相实现了这个需求，这个方法只支持 5.7，5.6 并不支持函数索引