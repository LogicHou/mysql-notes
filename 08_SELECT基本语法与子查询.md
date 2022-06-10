# SELECT基本语法与子查询

## MySQL--SELECT语法

https://dev.mysql.com/doc/refman/5.7/en/select.html

    SELECT
        [ALL | DISTINCT | DISTINCTROW ]
        [HIGH_PRIORITY]
        [STRAIGHT_JOIN]
        [SQL_SMALL_RESULT] [SQL_BIG_RESULT] [SQL_BUFFER_RESULT]
        [SQL_CACHE | SQL_NO_CACHE] [SQL_CALC_FOUND_ROWS]
        select_expr [, select_expr] ...
        [into_option]
        [FROM table_references
          [PARTITION partition_list]]
        [WHERE where_condition]
        [GROUP BY {col_name | expr | position}
          [ASC | DESC], ... [WITH ROLLUP]]
        [HAVING where_condition]
        [ORDER BY {col_name | expr | position}
          [ASC | DESC], ...]
        [LIMIT {[offset,] row_count | row_count OFFSET offset}]
        [PROCEDURE procedure_name(argument_list)]
        [into_option]
        [FOR UPDATE | LOCK IN SHARE MODE]

    into_option: {
        INTO OUTFILE 'file_name'
            [CHARACTER SET charset_name]
            export_options
      | INTO DUMPFILE 'file_name'
      | INTO var_name [, var_name] ...
    }

基本用法：

    (root@localhost) [dbt3]> SELECT 
      o_orderkey, o_orderDATE, LENGTH(o_orderpriority)
    FROM
        orders
    WHERE
        o_orderDATE >= '2009-02-01'
            AND o_orderDATE < '2009-03-01'
    LIMIT 10;

**SELECT 语法中最重要的三个知识点分别是 ORDER BY，GROUP BY 和 LIMIT**

### ORDER BY

ORDER BY 默认是从小往大 (ASC) 进行排序的：

    (root@localhost) [dbt3]> SELECT 
        *
    FROM
        lineitem
    ORDER BY l_discount
    LIMIT 10000;
    .....
    10000 rows in set (30.563 sec)

需要注意的是加上 order by 后查询会比不加要慢很多，这是因为 order by 是需要额外的性能开销的，去掉 order by 会发现查询效率变得快多了：

    (root@localhost) [dbt3]> SELECT 
        *
    FROM
        lineitem
    LIMIT 10000;
    .....
    10000 rows in set (0.06 sec)

一个关于 order by 的参数，叫排序会用到的内存，默认是 256K 的大小，这个参数可以在会话级别进行修改

    (root@localhost) [(none)]> show variables like 'sort_buffer_size';
    +------------------+--------+
    | Variable_name    | Value  |
    +------------------+--------+
    | sort_buffer_size | 262144 |
    +------------------+--------+
    1 row in set (0.00 sec)

显然256k有点小，现在改成256m试一下

    (root@localhost) [(none)]> set sort_buffer_size = 256*1024*1024;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [(none)]> show variables like 'sort_buffer_size';
    +------------------+-----------+
    | Variable_name    | Value     |
    +------------------+-----------+
    | sort_buffer_size | 268435456 |
    +------------------+-----------+
    1 row in set (0.00 sec)
    
再试一下上面买的select语句，性能提升接近一倍

    SELECT 
        *
    FROM
        lineitem
    ORDER BY l_discount
    LIMIT 10000;
    .....
    10000 rows in set (17.54 sec)  <--注意注意这里的时间少了很多

所以如果你的查询中有大量的 order by 并且是真正的需要去排序的**同时没有索引可以利用**的话，这时候一定要注意要去调大 sort_buffer_size

my.cnf 中稍微调大一点 sort_buffer_size

    [mysqld]
    ...
    ...
    # session menory
    sort_buffer_size = 32M

    # innodb
    innodb_buffer_pool_size = 1G

其实也不用看修改参数前后用 select 语句去查询速度对比，可以用下面命令来查看当前会话排序的一个状态：

    (root@localhost) [dbt3]> show status like 'sort%';
    +-------------------+-------+
    | Variable_name     | Value |
    +-------------------+-------+
    | Sort_merge_passes | 488   | <--如果排序的记录在内存放不下，就会多次归并放到磁盘
    | Sort_range        | 0     |
    | Sort_rows         | 10000 | <--排序了一万条记录
    | Sort_scan         | 1     | <--进行了一次scan
    +-------------------+-------+
    4 rows in set (0.00 sec)

    # 可以使用 flush 命令重置状态
    (root@localhost) [dbt3]> flush status;
    Query OK, 0 rows affected (0.00 sec)

在生产环境全局查看当前的 sort_buffer_size 设置的是否 OK，如果 Sort_merge_passes 比较大的话就需要调大 sort_buffer_size 这个参数：

    (root@localhost) [dbt3]> show global status like 'sort%';
    +-------------------+--------+
    | Variable_name     | Value  |
    +-------------------+--------+
    | Sort_merge_passes | 1476   |
    | Sort_range        | 138944 |
    | Sort_rows         | 1957824|
    | Sort_scan         | 32     |
    +-------------------+--------+

需要注意的是 sort_buffer_size 是相对于每个会话的，如果这时有 100 个线程连着都在做 sort 操作的话就会有 100 * sort_buffer_size 的开销，所以 sort_buffer_size 可以设得相对来说大一点但是也别设得太大，因为这个变量是基于每个会话可以使用的内存，如果并发量大的话加起来对内存的开销可能是会比较大的。不建议设得很多，因为有些查询可以去建个索引来避免排序的

order by 的另一种写法：

    (root@localhost) [dbt3]> SELECT
        o_orderkey,o_orderDATE,o_orderpriority
    FROM
        orders
    ORDER BY 3 # 表示对第三个列进行排序
    LIMIT 10000;

#### LIMIT 语法--取前 N 条记录

实现分页的效果：

    (root@localhost) [dbt3]> SELECT
        o_orderkey,o_orderDATE,o_orderpriority
    FROM
        orders
    LIMIT 0,10; # 取前10条

    (root@localhost) [dbt3]> SELECT
        o_orderkey,o_orderDATE,o_orderpriority
    FROM
        orders
    LIMIT 10,10; # 从第10条开始再取10条

LIMIT 语法有一个非常严重的问题，如果 limit 值很大速度就会非常慢

    SELECT 
        o_orderkey,o_orderDATE,o_orderpriority
    FROM
        orders
    LIMIT 1000000,10; # 表示先读一百万十条数据，然后再读10条数据给你，前面先要读一百万条数据所以就慢了

### GROUP BY

对于订单表有这么一个需求，根据 orders 表中的 o_totalprice，o_orderDATE 求出每个月产生的订单的总价，这种方式就需要用到 GROUP BY 了

    (root@localhost) [dbt3]> desc orders;
    +-----------------+-------------+------+-----+---------+-------+
    | Field           | Type        | Null | Key | Default | Extra |
    +-----------------+-------------+------+-----+---------+-------+
    | o_orderkey      | int(11)     | NO   | PRI | NULL    |       |
    | o_custkey       | int(11)     | YES  | MUL | NULL    |       |
    | o_orderstatus   | char(1)     | YES  |     | NULL    |       |
    | o_totalprice    | double      | YES  |     | NULL    |       |
    | o_orderDATE     | date        | YES  | MUL | NULL    |       |
    | o_orderpriority | char(15)    | YES  |     | NULL    |       |
    | o_clerk         | char(15)    | YES  |     | NULL    |       |
    | o_shippriority  | int(11)     | YES  |     | NULL    |       |
    | o_comment       | varchar(79) | YES  |     | NULL    |       |
    +-----------------+-------------+------+-----+---------+-------+
    9 rows in set (0.00 sec)

通过 date_format 函数先求出月份时间

    (root@localhost) [dbt3]> SELECT
            o_orderDATE,date_format(o_orderDATE,'%Y%m'), o_totalprice
        FROM
            orders limit 10;
    +-------------+---------------------------------+--------------+
    | o_orderDATE | date_format(o_orderDATE,'%Y%m') | o_totalprice |
    +-------------+---------------------------------+--------------+
    | 1996-01-02  | 199601                          |    173665.47 | <-这里的数据尚未归类，只是其中的某条数据
    | 1996-12-01  | 199612                          |     46929.18 |
    | 1993-10-14  | 199310                          |    193846.25 |
    | 1995-10-11  | 199510                          |     32151.78 |
    | 1994-07-30  | 199407                          |     144659.2 |
    | 1992-02-21  | 199202                          |     58749.59 |
    | 1996-01-10  | 199601                          |    252004.18 |
    | 1995-07-16  | 199507                          |    208660.75 |
    | 1993-10-27  | 199310                          |    163243.98 |
    | 1998-07-21  | 199807                          |     58949.67 |
    +-------------+---------------------------------+--------------+
    10 rows in set (0.00 sec)

然后使用通过使用 GROUP BY 归类

    (root@localhost) [dbt3]> SELECT
          DATE_FORMAT(o_orderDATE, '%Y%m'), sum(o_totalprice)
      FROM
          orders
      GROUP BY DATE_FORMAT(o_orderDATE, '%Y%m');
    +----------------------------------+--------------------+
    | DATE_FORMAT(o_orderDATE, '%Y%m') | sum(o_totalprice)  |
    +----------------------------------+--------------------+
    | 199201                           | 2924475612.3100023 |
    | 199202                           |  2722431986.229988 |
    | ...                              |                    |
    | 199807                           | 2935593508.9800105 |
    | 199808                           | 181794522.69999963 |
    +----------------------------------+--------------------+
    80 rows in set (1.52 sec)