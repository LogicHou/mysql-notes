# GROUP BY、分页优化

## GROUP BY

模拟一张Orders表

| userid | price | date |
| ------ | ----- | ---- |
| 1      | 100   |      |
| 1      | 200   |      |
| 1      | 50    |      |
| 2      | 40    |      |
| 2      | 80    |      |
| 3      | 500   |      |

GROUP BY 的意思表示的是做分组，分组后面可以加字段列表（可以是一个字段也可以是多个字段）

比如一个需求，每个用户产生的订单总额，其中的**每**就代表 GROUP BY（GROUP BY userid FROM orders）

**然后 SELECT 中必须出现 GROUP BY 的那个字段 (userid) 和聚合函数**

这就是分组，也就是说根据某个字段来进行分组，分组完之后要对其他出现的字段进行一个聚合操作（max，min，count，sum）

通常来说 GROUP BY 分组之后是要有一个聚合函数，既然是分组了肯定是要来进行计算的，并且这个计算一定是一个聚合的计算，这就是 GROUP BY

如果是每个用户每个月这样的需求，GROUP BY 就需要两个字段

演示：每个雇员每个月产生的订单的数量，订单的总额，平均订单的价格是多少

    (root@localhost) [dbt3]> SELECT 
        DATE_FORMAT(o_orderDATE, '%Y-%m'),
        o_clerk,
        COUNT(1),
        SUM(o_totalprice),
        AVG(o_totalprice)
    FROM
        orders
    GROUP BY DATE_FORMAT(o_orderDATE, '%Y-%m') , o_clerk;

这条SQL语句还可以进行调优

做分组 GROUP BY 操作的时候会产生一个临时表，用 explain 命令看下：

    (root@localhost) [dbt3]> EXPLAIN SELECT
                 DATE_FORMAT(o_orderDATE, '%Y-%m'),
                 o_clerk,
                 COUNT(1),
                 SUM(o_totalprice),
                 AVG(o_totalprice)
             FROM
                 orders
             GROUP BY DATE_FORMAT(o_orderDATE, '%Y-%m') , o_clerk;
    +----+-------------+--------+------------+------+---------------+------+---------+------+---------+----------+---------------------------------+
    | id | select_type | table  | partitions | type | possible_keys | key  | key_len | ref  | rows    | filtered | Extra                           |
    +----+-------------+--------+------------+------+---------------+------+---------+------+---------+----------+---------------------------------+
    |  1 | SIMPLE      | orders | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 1486685 |   100.00 | Using temporary; Using filesort |
    +----+-------------+--------+------------+------+---------------+------+---------+------+---------+----------+---------------------------------+
    1 row in set, 1 warning (0.00 sec)

    # Using temporary 就是表示产生了一个临时表

对优化的方法可以使用 tmp_table_size 参数，这个参数默认是 16M 大小

    (root@localhost) [dbt3]> show variables like 'tmp_table_size';
    +----------------+----------+
    | Variable_name  | Value    |
    +----------------+----------+
    | tmp_table_size | 16777216 |
    +----------------+----------+
    1 row in set (0.00 sec)

如果你的排序的表非常大的话，可以考虑调大这个参数，比如如果把这个参数调小上面的 GROUP BY 查询语句就会变慢：

    (root@localhost) [dbt3]> set tmp_table_size = 256*1024;

怎么看 tmp_table_size 是否足够：

    (root@localhost) [dbt3]> show status like '%tmp%';
    +-------------------------+-------+
    | Variable_name           | Value |
    +-------------------------+-------+
    | Created_tmp_disk_tables | 3     |  <--产生基于磁盘的临时表的次数
    | Created_tmp_files       | 7     |
    | Created_tmp_tables      | 8     |
    +-------------------------+-------+
    3 rows in set (0.00 sec)

通过 flush status 清除后再运行 GROUP BY 语句如果 Created_tmp_disk_tables 越小则性能提升越多

my.cnf 中推荐设置成 32M

    [mysqld]
    ...
    ...
    ...
    # session menory
    sort_buffer_size = 32M
    tmp_table_size = 32M

查看全局的(可以监控此参数)：

    (root@localhost) [(none)]> show global status like '%tmp%';
    +-------------------------+-------+
    | Variable_name           | Value |
    +-------------------------+-------+
    | Created_tmp_disk_tables | 2599  |
    | Created_tmp_files       | 2     |
    | Created_tmp_tables      | 21877 |
    +-------------------------+-------+
    3 rows in set (0.00 sec)

## 知识点补充 TIPS

### count(1) 和 count(*) 的区别

    (root@localhost) [test]> create table a(a int);
    Query OK, 0 rows affected (0.02 sec)

    (root@localhost) [test]> insert into a values(1);
    Query OK, 1 row affected (0.01 sec)

    (root@localhost) [test]> insert into a values(2);
    Query OK, 1 row affected (0.00 sec)

    (root@localhost) [test]> insert into a values(NULL);
    Query OK, 1 row affected (0.01 sec)

    (root@localhost) [test]> select * from a;
    +------+
    | a    |
    +------+
    |    1 |
    |    2 |
    | NULL |
    +------+
    3 rows in set (0.00 sec)

    (root@localhost) [test]> select count(1), count(a), count(*) from a;
    +----------+----------+----------+
    | count(1) | count(a) | count(*) |
    +----------+----------+----------+
    |        3 |        2 |        3 |
    +----------+----------+----------+
    1 row in set (0.00 sec)

    # count(1) 返回表里的记录数
    # count(a) 返回表里列a不为NULL值的记录数
    # count(*) 和 count(1) 一模一样没有区别，其实括号里的数字随便多少都可以比如 count(2), count(3), count(100), 都是一样的

### HAVING WHERE DONDITION

和 WHERE 过滤是不一样的是，HAVING 的意思是对聚合之后的表达式进行过滤，看下例子：

    (root@localhost) [dbt3]> SELECT 
        DATE_FORMAT(o_orderDATE, '%Y-%m') month,
        COUNT(*) count,
        SUM(o_totalprice) sum
    FROM
        orders
    WHERE
        o_orderDATE >= '1991-01-01'
            AND o_orderDATE < '1992-01-01'
    GROUP BY DATE_FORMAT(o_orderDATE, '%Y-%m')
    HAVING count > 19000;
    Empty set (0.00 sec)

如果没有使用 where 进行过滤就使用 HAVING，可能就会扫描全表，如果使用 WHERE 之后再使用 HAVING 则是在过滤之后的结果中再进行过滤，查询性能就会好很多，这一点**非常重要**

像下面这么写查询性能就会比较差：

    (root@localhost) [dbt3]> SELECT 
        DATE_FORMAT(o_orderDATE, '%Y-%m') month,
        COUNT(*) count,
        SUM(o_totalprice) sum
    FROM
        orders
    GROUP BY DATE_FORMAT(o_orderDATE, '%Y-%m')
    HAVING count > 19000 AND month >= '1991-01' AND month < '1992-01';
    Empty set (1.53 sec) <--虽然结果也是Empty的，但是运行时间明显变长了，而上面那个例子则是秒出的

### GROUP BY 中出现非分组字段列表和聚合函数的问题

    (root@localhost) [test]> drop table a;
    Query OK, 0 rows affected (0.02 sec)

    (root@localhost) [test]> create table a (userid int, price decimal, date datetime);
    Query OK, 0 rows affected (0.02 sec)

    (root@localhost) [test]> insert into a values(1, 100, '2016-01-01');
    Query OK, 1 row affected (0.01 sec)
    (root@localhost) [test]> insert into a values(2, 50, '2016-02-01');
    Query OK, 1 row affected (0.01 sec)
    (root@localhost) [test]> insert into a values(1, 50, '2016-02-01');
    Query OK, 1 row affected (0.00 sec)
    (root@localhost) [test]> insert into a values(1, 200, '2016-03-01');
    Query OK, 1 row affected (0.01 sec)
    (root@localhost) [test]> insert into a values(2, 100, '2016-07-01');
    Query OK, 1 row affected (0.00 sec)
    (root@localhost) [test]> insert into a values(3, 500, '2016-09-01');
    Query OK, 1 row affected (0.01 sec)

    (root@localhost) [test]> select * from a;
    +--------+-------+---------------------+
    | userid | price | date                |
    +--------+-------+---------------------+
    |      1 |   100 | 2016-01-01 00:00:00 |
    |      2 |    50 | 2016-02-01 00:00:00 |
    |      1 |    50 | 2016-02-01 00:00:00 |
    |      1 |   200 | 2016-03-01 00:00:00 |
    |      2 |   100 | 2016-07-01 00:00:00 |
    |      3 |   500 | 2016-09-01 00:00:00 |
    +--------+-------+---------------------+
    6 rows in set (0.00 sec)

    (root@localhost) [test]> select userid, sum(price) from a group by userid;
    +--------+------------+
    | userid | sum(price) |
    +--------+------------+
    |      1 |        350 |
    |      2 |        150 |
    |      3 |        500 |
    +--------+------------+
    3 rows in set (0.00 sec)
    # 这里正常

    (root@localhost) [test]> select userid, sum(price), date from a group by userid;
    ERROR 1055 (42000): Expression #3 of SELECT list is not in GROUP BY clause and contains nonaggregated column 'test.a.date' which is not functionally dependent on columns in GROUP BY clause; this is incompatible with sql_mode=only_full_group_by
    # mysql 5.7 版本中如果group by了某个字段，那么在select中它只能出现那个分组的字段列表和聚合函数，绝对不能出现另外一个单独的列(这里是date列)
    # 在mysql 5.6 版本中不会报错，date会出现任何任何记录当中随机的一条记录

一个很重要的参数 sql model，比 5.6 多了 ONLY_FULL_GROUP_BY，所以 5.7 中变得不一样

    (root@localhost) [(none)]> show variables like 'sql_mode';
    +---------------+-------------------------------------------------------------------------------------------------------------------------------------------+
    | Variable_name | Value                                                                                                                                     |
    +---------------+-------------------------------------------------------------------------------------------------------------------------------------------+
    | sql_mode      | ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION |
    +---------------+-------------------------------------------------------------------------------------------------------------------------------------------+
    1 row in set (0.00 sec)

5.6 也可以改成 5.7 这样的模式

    set sql_mode = 'ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION';
    Query OK, 0 rows affected (0.00 sec)

推荐设置成这样的严格模式 my.cnf，但是有个问题如果将已经插入非严格数据的 5.6、5.5 版本的数据库突然改成了严格模式，那么应用程序那边可能就完蛋了。特别是 5.5、5.6 升级到 5.7 的时候就出错了

所以老业务推荐想清楚了再改，新业务则强烈推荐开启严格模式 my.cnf


    [mysqld]
    ...
    ...
    ...
    #skip-grant-tables
    sql_mode = 'ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION'

### GROUP BY 的 group_concat 函数

对一个用户进行分组，然后用逗号（默认）把它们连接起来

    (root@localhost) [test]> select userid, group_concat(price) from a group by userid;
    +--------+---------------------+
    | userid | group_concat(price) |
    +--------+---------------------+
    |      1 | 100,50,200          |
    |      2 | 50,100              |
    |      3 | 500                 |
    +--------+---------------------+
    3 rows in set (0.00 sec)

可以指定符号

    (root@localhost) [test]> select userid, group_concat(price separator ':') from a group by userid;
    +--------+-----------------------------------+
    | userid | group_concat(price separator ':') |
    +--------+-----------------------------------+
    |      1 | 100:50:200                        |
    |      2 | 50:100                            |
    |      3 | 500                               |
    +--------+-----------------------------------+
    3 rows in set (0.00 sec)

对结果排序

    (root@localhost) [test]> select userid, group_concat(price order by price separator ':') from a group by userid;
    +--------+--------------------------------------------------+
    | userid | group_concat(price order by price separator ':') |
    +--------+--------------------------------------------------+
    |      1 | 50:100:200                                       |
    |      2 | 50:100                                           |
    |      3 | 500                                              |
    +--------+--------------------------------------------------+
    3 rows in set (0.01 sec)

逆序

    (root@localhost) [test]> select userid, group_concat(price order by date desc separator ':') from a group by userid;
    +--------+------------------------------------------------------+
    | userid | group_concat(price order by date desc separator ':') |
    +--------+------------------------------------------------------+
    |      1 | 200:50:100                                           |
    |      2 | 100:50                                               |
    |      3 | 500                                                  |
    +--------+------------------------------------------------------+
    3 rows in set (0.00 sec)

优化前面的 utf8mb4 轮询语句

    (root@localhost) [mysql]> SELECT 
        CONCAT(TABLE_SCHEMA, '.', table_name) AS name,
        character_set_name,
        GROUP_CONCAT(COLUMN_NAME
            SEPARATOR ':') AS COLUMN_LIST
    FROM
        information_schema.COLUMNS
    WHERE
        data_type IN ('varchar' , 'longtext', 'text', 'mediumtext', 'char')
            AND character_set_name <> 'utf8mb4'
            AND table_schema NOT IN ('mysql' , 'performance_schema', 'information_schema', 'sys')
    GROUP BY name , character_set_name;

## SELECT 的 distinct 去重

    (root@localhost) [test]> select distinct userid from a;
    +--------+
    | userid |
    +--------+
    |      1 |
    |      2 |
    |      3 |
    +--------+
    3 rows in set (0.00 sec)

## 子查询 单表子查询

### 子查询的使用

* operand comparison_operator ANY (subquery)
* operand IN (subquery)
* operand comparison_operator SOME (subquery)
* operand comparison_operator ALL (subquery)

其实子查询中 ANY、SOME、ALL 用得都比较少，用得最多的是 IN，建议大部分情况下都写成 IN

##### ANY

* ANY 关键词的意思是“对于在子查询返回的列中的任一数值，如果比较结果为 TRUE 的话，则返回 TRUE”
        
      SELECT s1 FROM t1 WHERE s1 > ANY (SELECT s1 FROM t2); # 也就是s1 t1中的所有值必须大于any后面筛选出来的值，这个where条件才成立

* SOME = ANY
* IN equals = ANY 表示等于 ANY 中的任何一个值就可以了，这是用得最多的的类型，大部分情况下推荐使用这个

      SELECT s1 FROM t1 WHERE s1 = ANY (SELECT s1 FROM t2);
      SELECT s1 FROM t1 WHERE s1 IN (SELECT s1 FROM t2);
* ALL
  * 对于子查询返回的列中的所有值，如果比较结果为 TRUE，则返回 TRUE
* NOT IN equals <> ALL
  
      SELECT s1 FROM t1 WHERE s1 > ALL (SELECT s1 FROM t2)

## 子查询的分类

* 独立子查询
  * self-contained subquery
* 相关子查询
  * correlated subquery
  * dependent subquery
* 独立子查询
  * 不依赖外部查询而运行的子查询
  * 返回每个美国员工至少为其处理过一个订单的所有客户

        SELECT customerid
            FROM orders
        WHERE employeeid IN (1,2,3,4,8)
            GROUP BY customerid
        HAVING COUNT(DISTINCT employeeid) = 5
      
* 相关子查询
  * 引用了外部查询列的子查询

独立子查询示例：

    (root@localhost) [test]> select * from a where userid in (1,2);
    +--------+-------+---------------------+
    | userid | price | date                |
    +--------+-------+---------------------+
    |      1 |   100 | 2016-01-01 00:00:00 |
    |      2 |    50 | 2016-02-01 00:00:00 |
    |      1 |    50 | 2016-02-01 00:00:00 |
    |      1 |   200 | 2016-03-01 00:00:00 |
    |      2 |   100 | 2016-07-01 00:00:00 |
    +--------+-------+---------------------+
    5 rows in set (0.00 sec)

    # 等价于：
    (root@localhost) [test]> select * from a where userid = 1 or userid = 2;
    或者
    (root@localhost) [test]> select * from a where userid = 1 union select * from a where userid = 2;

### EXISTS谓词

* EXISTS 谓词
  * 仅返回 TRUE、FALSE
  * UNKNOWN 返回为 FALSE
* 查询返回来自 Spain 且发生过订单的消费者
  
      SELECT 
          customerid, companyname
      FROM
          customers AS A
      WHERE
          country = 'Spain'
              AND EXISTS( SELECT 
                  *
              FROM
                  orders AS B
              WHERE
                  A.customerid = B.customerid);  # 有两个表关联条件的都可以视为相关子查询

IN 写法的示例，这里属于独立子查询

    (root@localhost) [employees]> SELECT 
        *
    FROM
        employees
    WHERE
        emp_no IN (SELECT       # IN的写法是某个列IN某个子查询
                emp_no
            FROM
                dept_emp
            WHERE
                dept_no = 'd005')
    LIMIT 10;

EXISTS 写法的示例，这里属于相关子查询

    (root@localhost) [employees]>  SELECT 
        *
    FROM
        employees e
    WHERE
        EXISTS( SELECT       # EXISTS的写法就是直接带一个子查询
                *
            FROM
                dept_emp de
            WHERE
                dept_no = 'd005' AND e.emp_no = de.emp_no) # 这里和外部表进行了关联，以此作为是相关子查询的判断依据
    LIMIT 10;

### 子查询的优化

每月最后实际订单日期发生的订单(每个月最后一笔订单的情况)，这里的 GROUP BY 只执行了一次

      (root@localhost) [dbt3]> SELECT 
          *
      FROM
          orders
      WHERE
          o_orderdate IN (SELECT 
                  MAX(o_orderdate)
              FROM
                  orders
              GROUP BY (DATE_FORMAT(o_orderdate, '%Y%M')));

改写成 EXISTS，这里的 GROUP BY 执行了 150W 次，明显上面的写法性能更好，同时也体现了独立子查询和相关子查询的区别

      (root@localhost) [dbt3]> SELECT 
          *
      FROM
          orders a
      WHERE
          EXISTS( SELECT 
                  MAX(o_orderdate)
              FROM
                  orders b
              GROUP BY (DATE_FORMAT(o_orderdate, '%Y%M'))
              HAVING MAX(b.o_orderdate) = a.o_orderdate);  #每个a.o_orderdate都会和b.o_orderdate进行一次关联查询，所以性能很差

MySQL 对于所有 IN 的子查询在 5.6 以前的版本中都是重写成了 EXISTS 相关子查询，执行效率非常非常差，最重要的原因是相关子查询会执行 N 多次，因为需要额外表中的每一行数据来进行关联，外部表有多少行记录子查询就需要执行多少次

从 5.6 开始 IN 不会重写成 EXISTS，对 IN 有一些独特的优化

### 什么时候 EXISTS 的效率会更高?

### NOT EXISTS

* NOT EXISTS => NOT IN
  * 注意 NULL 值的过滤
* 包含 NULL 的 NOT IN
  * FALSE
  * NOT NULL

在某些情况下 IN 和 EXISTS 有很大的不同，这种情况叫 NULL 值

来看下 IN 会有什么样的问题

    (root@localhost) [test]> select 'a' in ('a','b','c');
    +----------------------+
    | 'a' in ('a','b','c') |
    +----------------------+
    |                    1 |
    +----------------------+
    1 row in set (0.00 sec)

    (root@localhost) [test]> select 'a' not in ('a','b','c');
    +--------------------------+
    | 'a' not in ('a','b','c') |
    +--------------------------+
    |                        0 |
    +--------------------------+
    1 row in set (0.00 sec)

    (root@localhost) [test]> SELECT 'c' NOT IN ('a','b', NULL);
    +----------------------------+
    | 'c' NOT IN ('a','b', NULL) |
    +----------------------------+
    |                       NULL |
    +----------------------------+
    1 row in set (0.00 sec)

    (root@localhost) [test]> SELECT 'c' IN ('a','b', NULL);
    +------------------------+
    | 'c' IN ('a','b', NULL) |
    +------------------------+
    |                   NULL |
    +------------------------+
    1 row in set (0.00 sec)

    (root@localhost) [test]> SELECT 'a' NOT IN ('a','b', NULL);
    +----------------------------+
    | 'a' NOT IN ('a','b', NULL) |
    +----------------------------+
    |                          0 |
    +----------------------------+
    1 row in set (0.00 sec)

在和有 NULL 值的数据集进行比较的时候，如果条件不成立都会返回 NULL 值

这就会遇到一个问题，来看下这个例子：

    (root@localhost) [test]> select * from a;
    +--------+-------+---------------------+
    | userid | price | date                |
    +--------+-------+---------------------+
    |      1 |   100 | 2016-01-01 00:00:00 |
    |      2 |    50 | 2016-02-01 00:00:00 |
    |      1 |    50 | 2016-02-01 00:00:00 |
    |      1 |   200 | 2016-03-01 00:00:00 |
    |      2 |   100 | 2016-07-01 00:00:00 |
    |      3 |   500 | 2016-09-01 00:00:00 |
    +--------+-------+---------------------+
    6 rows in set (0.00 sec)

    (root@localhost) [test]> create table b ( y int );
    Query OK, 0 rows affected (0.01 sec)

    (root@localhost) [test]> insert into b value(1);
    Query OK, 1 row affected (0.01 sec)
    (root@localhost) [test]> insert into b value(2);
    Query OK, 1 row affected (0.00 sec)
    (root@localhost) [test]> insert into b value(NULL);
    Query OK, 1 row affected (0.00 sec)

    (root@localhost) [test]> select * from b;
    +------+
    | y    |
    +------+
    |    1 |
    |    2 |
    | NULL |
    +------+
    3 rows in set (0.00 sec)

    (root@localhost) [test]> select * from a where userid not in (select y from b);
    Empty set (0.00 sec)
    # 按道理应该返回 userid 为 3 的数据，但是返回的确是空值，空值的原因就在于前面例子中的 NOT IN 对于带有 NULL 值的只会返回 0 和 NULL 值，where 中 0 和 NULL 值代表的就是 false，所以永远取不到想要的结果
    # 这一点要非常注意
    # 如果遇到类似的问题，第一反应应该就是子查询的数据集里是否带了 NULL 值

#### 改写，先把 NULL 值过滤掉

    (root@localhost) [test]> select * from a where userid not in (select y from b where y is not null);
    +--------+-------+---------------------+
    | userid | price | date                |
    +--------+-------+---------------------+
    |      3 |   500 | 2016-09-01 00:00:00 |
    +--------+-------+---------------------+
    1 row in set (0.00 sec)

既然有 not in 那应该也有 not exists

    (root@localhost) [test]> select * from a where not exists (select y from b where y = a.userid);
    +--------+-------+---------------------+
    | userid | price | date                |
    +--------+-------+---------------------+
    |      3 |   500 | 2016-09-01 00:00:00 |
    +--------+-------+---------------------+
    1 row in set (0.01 sec)
    # 用 NOT EXISTS 时 y = a.userid 条件不成立那么 3 这条记录会被取出来，也就是 not exists 条件成立

NULL 值的确是一个坑，很多时候要去避免这个坑，要想到很多时候问题是由 NULL 值引起的

到 5.6 版本还多关于 IN 的优化都不需要了，MySQL 对于 IN 的优化已经做得足够好了

# 分页的优化

对于分页的问题，越分到后面性能就越差

    (root@localhost) [employees]> SELECT * FROM employees ORDER BY birth_date LIMIT 30;
    30 rows in set (0.25 sec)

    (root@localhost) [employees]> SELECT * FROM employees ORDER BY birth_date LIMIT 100000, 30;
    30 rows in set (0.25 sec) # 性能越来越差

### 一种优化方式

首先对排序的列 birth_date 创建一个索引，但是这个索引一定要和一个唯一索引绑定，这里带上了 emp_no 唯一列
      
    (root@localhost) [emp]> ALTER TABLE employees ADD INDEX idx_birth_date(birth_date,emp_no);
    
    # 然后用下面的方式进行查询
    (root@localhost) [emp]> SELECT * FROM employees ORDER BY birth_date, emp_no LIMIT 30;
    +--------+------------+------------+--------------+--------+------------+
    | emp_no | birth_date | first_name | last_name    | gender | hire_date  |
    +--------+------------+------------+--------------+--------+------------+
    |  65308 | 1952-02-01 | Jouni      | Pocchiola    | M      | 1985-03-10 |
    |  ..... | .......... | ....       | ............ | .      | .......... |
    | 217446 | 1952-02-02 | Mayuri     | Barriga      | F      | 1996-06-26 |
    +--------+------------+------------+--------------+--------+------------+
    30 rows in set (0.00 sec)

    SELECT * FROM employees 
    WHERE (birth_date, emp_no) > ('1952-02-02', 217446) # 可以理解为两个列组成的一个子查询
    ORDER BY birth_date LIMIT 30;

这种做法性能都是平均的，不会越翻越慢，其中的('1952-02-02',217446)从程序里或者页面里取就可以

缺点是分页的时候只能下一页和上一页，不能做跳转到第几页，同时这么做对应用程序的改动太大比较难用

* 首先取 1000 条数据，前端进行分页
* 找到一个唯一索引，通过排序列和唯一索引进行分页

### select * 和 select count(1) 的优化问题

其实这是没什么意义的事情，select count(1) 你看到的数据永远不是很准确的，你看到的只是你在 9 点钟按下回车那个时候的数据，但是如果这条 sql 语句执行了 10 秒钟，这 10 秒钟里面这张表可能少了数据也有可能多了数据，除非这张表是不变化的，否则 count 不是很准确的

既然 count 不是一个精确的值就有很多种方法去优化它，首先第一点 count 这个值可以写在缓存里，之后每次有发生更新的话先更新缓存，定期再把数据更新到数据库

另外一点既然他是不准的，那么可以定期去执行，比如去 slave 执行，不要实时的去执行，不建议 count 这个操作实时的在数据库里进行操作

需要准确的 count 话就先加锁，for update

    (root@localhost) [dbt3]> select count(1) from orders;
    +----------+
    | count(1) |
    +----------+
    |  1500000 |
    +----------+
    1 row in set (0.29 sec)

    (root@localhost) [dbt3]> select count(1) from orders for update;
    +----------+
    | count(1) |
    +----------+
    |  1500000 |
    +----------+
    1 row in set (3.40 sec)