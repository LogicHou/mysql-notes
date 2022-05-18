# Prepared SQL语法

* 优势
  * Less overhead for parsing the statement each time it is executed. 解析的开销更小，性能更好
  * Protection against SQL injection attacks. 防范SQL注入
  * dynamic search condition 动态查询条件

https://dev.mysql.com/doc/refman/5.7/en/sql-prepared-statements.html

例子：

    SET @s = 'SELECT * FROM employees WHERE emp_no = ?'; -- ? 号表示预替换的值
    SET @a = 100080; -- 设置值
    PREPARE stmt FROM @s; -- 准备语句
    EXECUTE stmt USING @a; -- 执行语句
    DEALLOCATE PREPARE stmt; -- 释放

什么是SQL注入：

    SELECT * FROM employees WHERE emp_no = 10080 OR 1=1;
    -- 这样( 1=1 表示 true )会取出所有数据而不是 10080 号员工的数据，这时如果你的管理员用户在同一张表，那么可能就会泄露了

改写：

    SET @s = 'SELECT * FROM employees WHERE emp_no = ?'; -- emp_no是INT类型
    SET @a = '100080 or 1=1'; -- 所以这里的输入转成了INT类型
    PREPARE stmt FROM @s;
    EXECUTE stmt USING @a;
    DEALLOCATE PREPARE stmt; -- 最后会只出现一条数据

## 实现动态查询

    (root@localhost) [emp]> desc employees;
    +------------+---------------+------+-----+---------+-------+
    | Field      | Type          | Null | Key | Default | Extra |
    +------------+---------------+------+-----+---------+-------+
    | emp_no     | int(11)       | NO   | PRI | NULL    |       |
    | birth_date | date          | NO   |     | NULL    |       |
    | first_name | varchar(14)   | NO   |     | NULL    |       |
    | last_name  | varchar(16)   | NO   |     | NULL    |       |
    | gender     | enum('M','F') | NO   |     | NULL    |       |
    | hire_date  | date          | NO   |     | NULL    |       |
    +------------+---------------+------+-----+---------+-------+
    6 rows in set (0.00 sec)

### 示例1

    SET @s = 'SELECT * FROM employees WHERE 1=1'; -- 写1=1是为了方便where继续拼接上其他条件，而不需要先拼接上一个实际的条件
    SET @s = CONCAT(@s, ' AND gender = "m"');
    SET @s = CONCAT(@s, ' AND birth_date >= "1960-01-01"'); -- 条件可以动态的添加删除

    PREPARE stmt FROM @s;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;

或者

    SET @s = 'SELECT * FROM employees WHERE 1=1'; 
    SET @s = CONCAT(@s, ' AND gender = ?');
    SET @s = CONCAT(@s, ' AND birth_date >= ?');
    SET @a = "m"; -- 设置值
    SET @b = "1960-01-01"; -- 设置值

    PREPARE stmt FROM @s;
    EXECUTE stmt USING @a, @b;
    DEALLOCATE PREPARE stmt;

### 示例2，带上了分页

    SET @s = 'SELECT * FROM employees WHERE 1=1';
    SET @s = CONCAT(@s, ' AND gender = "m"');
    SET @s = CONCAT(@s, ' AND birth_date >= "1960-01-01"');
    SET @s = CONCAT(@s, ' ORDER BY emp_no LIMIT ?,?');
    SET @page_no = 0;
    SET @page_count = 10;

    PREPARE stmt FROM @s;
    EXECUTE stmt USING @page_count, @page_count;
    DEALLOCATE PREPARE stmt;

# DML语法

## INSERT 基本语法

### 一次插入多个值

    (root@localhost) [test]> insert into x values(4, 40);
    Query OK, 1 row affected (0.00 sec)

    (root@localhost) [test]> insert into x values(5,40),(6,60),(7,70);
    Query OK, 3 rows affected (0.00 sec)
    Records: 3  Duplicates: 0  Warnings: 0

    (root@localhost) [test]> select * from x;
    +------+------+
    | a    | b    |
    +------+------+
    |    1 |    1 |
    |    2 |   20 |
    |    3 |   30 |
    |    4 |   40 |
    |    5 |   40 |
    |    6 |   60 |
    |    7 |   70 |
    +------+------+
    7 rows in set (0.00 sec)

### INSERT …… SET 语法

    (root@localhost) [test]> insert into x set a=8, b=80;
    Query OK, 1 row affected (0.01 sec)

    (root@localhost) [test]> select * from x;
    +------+------+
    | a    | b    |
    +------+------+
    |    1 |    1 |
    |    2 |   20 |
    |    3 |   30 |
    |    4 |   40 |
    |    5 |   40 |
    |    6 |   60 |
    |    7 |   70 |
    |    8 |   80 |
    +------+------+
    8 rows in set (0.00 sec)

### INSERT …… SELECT 语法

    (root@localhost) [test]> insert into x select * from y;

## DELETE 语法

### 示例1

    (root@localhost) [test]> delete from x where a = 6;
    Query OK, 1 row affected (0.01 sec)

    (root@localhost) [test]> delete from x where b = 40;
    Query OK, 2 rows affected (0.00 sec)

    (root@localhost) [test]> select * from x;
    +------+------+
    | a    | b    |
    +------+------+
    |    1 |    1 |
    |    2 |   20 |
    |    3 |   30 |
    |    7 |   70 |
    |    8 |   80 |
    +------+------+
    5 rows in set (0.00 sec)

### 示例2

    (root@localhost) [test]> delete from x where a in (7,8);
    Query OK, 2 rows affected (0.00 sec)

    (root@localhost) [test]> select * from x;
    +------+------+
    | a    | b    |
    +------+------+
    |    1 |    1 |
    |    2 |   20 |
    |    3 |   30 |
    +------+------+
    3 rows in set (0.00 sec)

### 示例3 把 a 字段在 x 中但不在 y 中的数据删除

    (root@localhost) [test]> delete from x where a not in (select a from y); -- 第一种写法

    (root@localhost) [test]> delete x,y from x left join y on x.a = y.a where y.a is NULL; -- 第二种写法
    Query OK, 1 row affected (0.00 sec)

    (root@localhost) [test]> select * from x;
    +------+------+
    | a    | b    |
    +------+------+
    |    1 |   10 |
    |    2 |   20 |
    +------+------+
    2 rows in set (0.00 sec)

## UPDATE语法

### 示例

    (root@localhost) [test]> insert into x values(3,40);
    Query OK, 1 row affected (0.00 sec)
    (root@localhost) [test]> update x set b = b + 10 where a = 3;

### 存在的小问题

    (root@localhost) [test]> select * from x;
    +------+------+
    | a    | b    |
    +------+------+
    |    1 |   10 |
    |    2 |   20 |
    |    3 |   40 |
    +------+------+
    3 rows in set (0.00 sec)

    (root@localhost) [test]> update x set a = a + 1, b = a + 10 where a = 2;
    Query OK, 1 row affected (0.00 sec)
    Rows matched: 1  Changed: 1  Warnings: 0

    (root@localhost) [test]> select * from x;
    +------+------+
    | a    | b    |
    +------+------+
    |    1 |   10 |
    |    3 |   13 | -- 这里是 13 而不是 12 是因为 a=a+1 已经将 a 计算成了 3，然后再在 b=a+10 中进行了计算
    |    3 |   50 |
    +------+------+
    3 rows in set (0.00 sec)

## MySQL对语法的一些扩展

先插入一些示例数据：

    (root@localhost) [test]> drop table z;
    Query OK, 0 rows affected (0.01 sec)

    (root@localhost) [test]> create table z ( a int primary key );
    Query OK, 0 rows affected (0.01 sec)

    (root@localhost) [test]> insert into z value(1);
    Query OK, 1 row affected (0.00 sec)

    (root@localhost) [test]> insert into z select 2; -- select(2)表示from于一张空表的意思
    Query OK, 1 row affected (0.00 sec)
    Records: 1  Duplicates: 0  Warnings: 0

    (root@localhost) [test]> insert into z select 3 from dual; -- 同上 这里from于一张dual空表的
    Query OK, 1 row affected (0.01 sec)
    Records: 1  Duplicates: 0  Warnings: 0

    (root@localhost) [test]> select * from z;
    +---+
    | a |
    +---+
    | 1 |
    | 2 |
    | 3 |
    +---+
    3 rows in set (0.00 sec)

### on duplicate key 语法，如果有存在的项则更新，不建议使用

    (root@localhost) [test]> begin;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> insert into z values (2);
    ERROR 1062 (23000): Duplicate entry '2' for key 'PRIMARY'
    (root@localhost) [test]> insert into z values (2) on duplicate key update a = a + 10;
    Query OK, 2 rows affected (0.00 sec)

    (root@localhost) [test]> select * from z;
    +----+
    | a  |
    +----+
    |  1 |
    |  3 |
    | 12 |
    +----+
    3 rows in set (0.00 sec)

    (root@localhost) [test]> rollback;
    Query OK, 0 rows affected (0.00 sec)

### replace 语法，和 insert 一样，但是遇到重复项会先进行删除再插入

    (root@localhost) [test]> replace into z values (2);
    Query OK, 1 row affected (0.01 sec)

    (root@localhost) [test]> select * from z; --这里发现没有变化
    +---+
    | a |
    +---+
    | 1 |
    | 2 |
    | 3 |
    +---+
    3 rows in set (0.00 sec)

    (root@localhost) [test]> alter table z add column b int not null default 1;
    Query OK, 0 rows affected (0.07 sec)
    Records: 0  Duplicates: 0  Warnings: 0

    (root@localhost) [test]> select * from z;
    +---+---+
    | a | b |
    +---+---+
    | 1 | 1 |
    | 2 | 1 |
    | 3 | 1 |
    +---+---+
    3 rows in set (0.00 sec)

    (root@localhost) [test]> update z set b = 2 where a = 2;
    Query OK, 1 row affected (0.00 sec)
    Rows matched: 1  Changed: 1  Warnings: 0

    (root@localhost) [test]> update z set b = 3 where a = 1;
    Query OK, 1 row affected (0.00 sec)
    Rows matched: 1  Changed: 1  Warnings: 0

    (root@localhost) [test]> select * from z;
    +---+---+
    | a | b |
    +---+---+
    | 1 | 3 |
    | 2 | 2 |
    | 3 | 1 |
    +---+---+
    3 rows in set (0.00 sec)

    (root@localhost) [test]> alter table z add unique index idx_b(b);
    Query OK, 0 rows affected (0.02 sec)
    Records: 0  Duplicates: 0  Warnings: 0

    (root@localhost) [test]> show create table z \G
    *************************** 1. row ***************************
          Table: z
    Create Table: CREATE TABLE `z` (
      `a` int(11) NOT NULL,
      `b` int(11) NOT NULL DEFAULT '1',
      PRIMARY KEY (`a`),
      UNIQUE KEY `idx_b` (`b`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
    1 row in set (0.00 sec)

    (root@localhost) [test]> select * from z;
    +---+---+
    | a | b |
    +---+---+
    | 3 | 1 |
    | 2 | 2 |
    | 1 | 3 |
    +---+---+
    3 rows in set (0.00 sec)

    (root@localhost) [test]> begin; replace into z values (2, 3);
    Query OK, 0 rows affected (0.00 sec)

    Query OK, 3 rows affected (0.00 sec) -- 发现这里影响了3行记录，为什么不是2行？

    (root@localhost) [test]>  select * from z;
    +---+---+
    | a | b |
    +---+---+
    | 3 | 1 |
    | 2 | 3 |
    +---+---+
    2 rows in set (0.00 sec)

意思是 replace into z values (2, 3) 先插入 2，发现 2 这条记录是存在的所以 replace 做的是 delete 操作，把重复的 a=2 的这条记录删除了，删完之后重新再插 (2, 3) 这条记录，这时候又发现 b=3 这条记录也存在于是又把 b=3 也删除了，删除这 2 条记录之后接着再插入 (2, 3)

所以影响的 3 行依次是：

    delete | 2 | 2 |
    delete | 1 | 3 |
    insert | 2 | 3 |

replace 会将所有重复的数据先删掉再插入，是肯定可以执行成功的

使用 on duplicate key 则不行：

    (root@localhost) [test]> insert into z values(2,3) on duplicate key update a = a + 10, b = 3;
    ERROR 1062 (23000): Duplicate entry '3' for key 'idx_b' -- 再次遇到有唯一索引的3停止了，on duplicate key 执行的只是 update 操作，并不会执行 delet e和 insert
    (root@localhost) [test]> rollback;
    Query OK, 0 rows affected (0.00 sec)

replace 是一个很好的语法，可以用来实现[幂等](https://zh.wikipedia.org/wiki/%E5%86%AA%E7%AD%89)

可以将所有的 insert、delete、update 语句都转化成 replace 语句，只要他没有唯一索引

在做跨机房的复制的时候，可以考虑用 replace 迁移数据，将语句翻译成 replace 再去同步，这样做的好处是操作是幂等的，随便怎么执行执行多少次，出来的结果都是一样的，这是一个非常好的特性

# 临时表

    (root@localhost) [test]> drop tables a;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> create temporary table a ( a int );
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> show tables; -- 这时看不到a表
    +----------------+
    | Tables_in_test |
    +----------------+
    | UserJson       |
    | b              |
    | x              |
    | y              |
    | z              |
    +----------------+
    5 rows in set (0.00 sec)

    (root@localhost) [test]> insert into a values (1); -- 但是数据依旧可以插入
    Query OK, 1 row affected (0.00 sec)

    (root@localhost) [test]> select * from a;
    +------+
    | a    |
    +------+
    |    1 |
    +------+
    1 row in set (0.00 sec)

临时表是基于会话的，如果退出了会话临时表就不存在了

所以临时表在多个会话中可以创建同名的临时表，但是他们是互相独立的，维护的数据也不同

临时表默认是 Innodb 的

    (root@localhost) [test]> show variables like '%tmp%';
    +----------------------------------+----------+
    | Variable_name                    | Value    |
    +----------------------------------+----------+
    | default_tmp_storage_engine       | InnoDB   |
    | innodb_tmpdir                    |          |
    | internal_tmp_disk_storage_engine | InnoDB   |
    | max_tmp_tables                   | 32       |
    | slave_load_tmpdir                | /tmp     |
    | tmp_table_size                   | 33554432 |
    | tmpdir                           | /tmp     |
    +----------------------------------+----------+
    7 rows in set (0.01 sec)

在 MySQL 的数据文件目录下，有一个数据表文件 **ibtmp1**，叫临时表空间，在临时表中插入数据后这个数据表文件就会慢慢变大，数据库重启后这个文件会被删除重建

同时创建临时表后，会在系统的 /tmp 目录下生成对应的临时表表结构文件，文件名类似 #sqlxxxxx.frm，而临时表的内容还是放在 MySQL 数据文件目录 ibtmp1 文件中

在 5.6 版本中有些不同，5.6 中没有 ibtmp1 数据表文件，而会在 /tmp 下同时生成 #sqlxxxxx.frm 和 #sqlxxxxx.ibd文件，所有的内容都存放在 /tmp 目录下，如果 /tmp 目录不够大则会报错，而通常 /tmp 目录也不会很大要注意这个问题

5.6 建议将 tmpdir 配置到数据目录下

    vim /etc/my.cnf

    ...
    general_log_file = general.log
    
    [mysqld-5.6]
    #tmpdir = /mdata/tmp

    ...

存储过程中会产生很多临时的数据，这些数据有时会存放到临时表中，然后 select 一下返回给应用程序

临时表在存储过程中会用得比较多

# 存储过程和自定义函数

## 存储过程 -- 概述

* 存储在数据库端的一组 SQL 语句集
* 用户可以通过存储过程名和传参多次调用的程序模块
* 存储过程的特点：
  * 使用灵活，可以使用流控制语句、自定义变量等完成复杂的业务逻辑
  * 提高数据安全性，屏蔽应用程序直接对表的操作，易于进行审计
  * 减少网络传输
  * 提高代码维护的复杂度，实际使用中要评估场景是否适合

应用端基本不用

MySQL 的存储过程性能比较差