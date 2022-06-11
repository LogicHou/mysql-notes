# 多表连接SQL语法

## SQL JOIN -- 语法(内连接)

    SELECT...FROM           SELECT...FROM a           SELECT...FROM a
    a,b                     INNER JOIN b              JOIN b
    WHERE a.x = b.x         on a.x = b.x              on a.x = b.x

* 上述这些语法没有任何区别
* 性能方面也没有任何区别，如果非要比一个出来那应该是第三个最好，因为字节数最少
* 那为什么需要不同的语法？
  * ANSI SQL89、ANSI SQL92 语法标准
  * ANSI 92 标准开始支持 OUTER JOIN 
  * INNER JOIN 可以省略 INNER 关键字

示例：

    (root@localhost) [test]> create table x ( a int, b int );
    Query OK, 0 rows affected (0.01 sec)

    (root@localhost) [test]> insert into x value (1, 1);
    Query OK, 1 row affected (0.01 sec)
    (root@localhost) [test]> insert into x value (2, 20);
    Query OK, 1 row affected (0.01 sec)
    (root@localhost) [test]> insert into x value (3, 30);
    Query OK, 1 row affected (0.00 sec)

    (root@localhost) [test]> select * from x;
    +------+------+
    | a    | b    |
    +------+------+
    |    1 |    1 |
    |    2 |   20 |
    |    3 |   30 |
    +------+------+
    3 rows in set (0.00 sec)

    (root@localhost) [test]> create table y ( a int, b datetime );
    Query OK, 0 rows affected (0.02 sec)

    (root@localhost) [test]> insert into y value(1, '2019-03-14 20:29:49');
    Query OK, 1 row affected (0.00 sec)
    (root@localhost) [test]> insert into y value(2, '2019-03-14 20:29:46');
    Query OK, 1 row affected (0.01 sec)
    (root@localhost) [test]> insert into y value(2, '2019-03-14 20:40:11');
    Query OK, 1 row affected (0.00 sec)

    (root@localhost) [test]> select * from y;
    +------+---------------------+
    | a    | b                   |
    +------+---------------------+
    |    1 | 2019-03-14 20:29:49 |
    |    2 | 2019-03-14 20:29:46 |
    |    2 | 2019-03-14 20:40:11 |
    +------+---------------------+
    3 rows in set (0.01 sec)

    (root@localhost) [test]> select * from x,y where x.a = y.a;
    (root@localhost) [test]> select * from x inner join y on x.a = y.a;
    (root@localhost) [test]> select * from x cross join y on x.a = y.a;
    (root@localhost) [test]> select * from x join y on x.a = y.a;
    +------+------+------+---------------------+ <--这3条语句都是等价的，出来的结果都一样，性能也完全一样
    | a    | b    | a    | b                   |
    +------+------+------+---------------------+
    |    1 |    1 |    1 | 2019-03-14 20:29:49 |
    |    2 |   20 |    2 | 2019-03-14 20:29:46 |
    |    2 |   20 |    2 | 2019-03-14 20:40:11 |
    +------+------+------+---------------------+
    3 rows in set (0.00 sec)

和 IN 语句的区别，上面的称为连接 JOIN，而下面的则是子查询也可以称为半连接

    (root@localhost) [test]> select * from x where a in( select a from y );
    +------+------+
    | a    | b    |
    +------+------+
    |    1 |    1 |
    |    2 |   20 |
    +------+------+
    2 rows in set (0.00 sec)

这里会对重复的数据进行去重，只出现 x 表中出现的数据，不会重复出现，而 JOIN 关联出来的数据可以是一对多的情况

JOIN 和子查询其实是会有一些些关联的，但又不完全一样，IN 只会出现 x 表中的数据，INNER JOIN 的话只要关联条件满足的话就会把数据组合起来，会出现多条数据

将上面的 IN 写法改写成 JOIN 但是依旧保留原来的去重效果

    (root@localhost) [test]> select distinct x.* from x join y on x.a = y.a;
    +------+------+
    | a    | b    |
    +------+------+
    |    1 |    1 |
    |    2 |   20 |
    +------+------+
    2 rows in set (0.00 sec)

JOIN 的性能问题(经常会听到说 JOIN 的性能很不好)？ TODO

### 笛卡尔积 CROSS JOIN

去两张表里面取数据，如果没有关联条件，那么出来就是一个称之为笛卡尔积的结果，下面的例子就是 3*3 出来了 9 条数据

    (root@localhost) [test]> select * from x,y;
    +------+------+------+---------------------+
    | a    | b    | a    | b                   |
    +------+------+------+---------------------+
    |    1 |    1 |    1 | 2019-03-14 20:29:49 | <--符合x.a=y.a关联条件的数据
    |    2 |   20 |    1 | 2019-03-14 20:29:49 |
    |    3 |   30 |    1 | 2019-03-14 20:29:49 |
    |    1 |    1 |    2 | 2019-03-14 20:29:46 |
    |    2 |   20 |    2 | 2019-03-14 20:29:46 | <--符合x.a=y.a关联条件的数据
    |    3 |   30 |    2 | 2019-03-14 20:29:46 |
    |    1 |    1 |    2 | 2019-03-14 20:40:11 |
    |    2 |   20 |    2 | 2019-03-14 20:40:11 | <--符合x.a=y.a关联条件的数据
    |    3 |   30 |    2 | 2019-03-14 20:40:11 |
    +------+------+------+---------------------+
    9 rows in set (0.00 sec)

可能会经常用到的语句，增加了 x.a = 1 的判断

    (root@localhost) [test]> select * from x,y where x.a = y.a and x.a = 1;
    +------+------+------+---------------------+
    | a    | b    | a    | b                   |
    +------+------+------+---------------------+
    |    1 |    1 |    1 | 2019-03-14 20:29:49 |
    +------+------+------+---------------------+
    1 row in set (0.01 sec)

有人会觉得上面几条语句的执行顺序是这样依次向下的：

    1. select * from x,y; 先两张表做笛卡尔积
    2. select * from x,y where x.a = y.a; 然后根据 x.a = y.a 进行过滤出3条数据
    3. select * from x,y where x.a = y.a and x.a = 1; 最后过滤出 x.a = 1 的这条数据

但是执行过程并不出这样的，数据库的执行过程是先取出 x.a = 1 的这条数据

    (root@localhost) [test]> select * from x where a = 1;
    +------+------+ <--先取出这张表，再去和b表进行关联
    | a    | b    |
    +------+------+
    |    1 |    1 |
    +------+------+
    1 row in set (0.00 sec)

所以即使 x,y 两张表很大，但是在 a=1 这个列上是有索引的话，其实 JOIN 并不会产生笛卡尔积

只有你没有关联条件，然后又有 a=1 这样的过滤条件的话，才会先产生笛卡尔积

有人觉得这条SQL语句不能写

    select * from x,y where x.a = y.a and x.a = 1;

要分两条语句写，一个是先写成：

    select * from x where a = 1;

取出结果之后再去y表中进行操作：

    select * from y where a = 1;

分两条语句写性能会更好？其实是很没有道理的

### 示例：employees 数据库下有 employees 这张表，现在要求出这个员工所在的部门的名称是什么

首先使用这样的语句进行查询

    (root@localhost) [employees]> SELECT 
        e.emp_no, t.title
    FROM
        employees e,
        titles t
    WHERE
        e.emp_no = t.emp_no;
    +--------+--------------+
    | emp_no | title        |
    +--------+--------------+
    |  65308 | Senior Staff |
    |  65308 | Staff        |
    |  ..... | .....        |
    |  ..... | .....        |
    +--------+--------------+

如果是这样取的话会发现一个员工会有多个结果，这样的话是否满足查询条件就要看你的需求了，因为这张表是一对多的关系，一个员工对应 titles 表中是有多条记录的，因为 titles 表中有个叫 from_date 的字段表示从什么时间开始员工的 title 是什么样的

所以这样进行关联的话取出的是员工所有曾经的 title 信息，将 t 表中的 t.from_date, t.to_date 取出来会看得更清晰一点

    (root@localhost) [employees]> SELECT 
        e.emp_no, t.title, t.from_date, t.to_date
    FROM
        employees e,
        titles t
    WHERE
        e.emp_no = t.emp_no;
    +--------+--------------+------------+------------+
    | emp_no | title        | from_date  | to_date    |
    +--------+--------------+------------+------------+
    |  65308 | Senior Staff | 1998-01-24 | 9999-01-01 | <--这时候是一个高级职员
    |  65308 | Staff        | 1993-01-24 | 1998-01-24 | <--这时候是普通员工
    |  ..... | .....        | .....      | .....      |
    |  ..... | .....        | .....      | .....      |
    +--------+--------------+------------+------------+

一对多的情况下取 JOIN 会将这种多的关系给显示出来

#### 取出每个员工最新的 title 信息：

独立子查询写法：

    (root@localhost) [employees]> SELECT
        emp_no, title
    FROM
        titles
    WHERE
        (emp_no, to_date) IN (SELECT 
                emp_no, MAX(to_date)
            FROM
                titles
            GROUP BY emp_no);
    300033 rows in set (1.62 sec)

改写关联语句达到同样效果(通过派生表的方式，也就是以上面查询语句的结果集进行关联)：

    (root@localhost) [employees]> SELECT 
        e.emp_no, t.title
    FROM
        employees e,
        (SELECT 
            emp_no, title
        FROM
            titles
        WHERE
            (emp_no , to_date) IN (SELECT 
                    emp_no, MAX(to_date)
                FROM
                    titles
                GROUP BY emp_no)) t
    WHERE
        e.emp_no = t.emp_no;
    300033 rows in set (2.56 sec)  <--改写后性能下降了，因为这里是相关子查询

使用这种方法输出的结果虽然一样，但是需要获得更多 employees 表相关的字段时就得这么写

## 外连接（LEFT JOIN, RIGHT JOIN）， outer 关键词可以省略

LEFT JOIN 左连接将 x 作为左表，x 作为保留表，保留表中的所有行**都要出现**，之后再和 y 表进行关联

如果关联条件成立就将数据关联出来，如果关联条件不成立就用 NULL 值进行填充，但是左表中的数据**都要出现**

RIGHT OUTER JOIN 右连接就是反过来右边那个表是保留表，原理和左连接也是一样的

    (root@localhost) [test]> select * from x left outer join y on x.a = y.a;
    +------+------+------+---------------------+
    | a    | b    | a    | b                   |
    +------+------+------+---------------------+
    |    1 |    1 |    1 | 2019-03-14 20:29:49 |
    |    2 |   20 |    2 | 2019-03-14 20:29:46 |
    |    2 |   20 |    2 | 2019-03-14 20:40:11 |
    |    3 |   30 | NULL | NULL                | <--关联条件不成立用NULL值进行了填充
    +------+------+------+---------------------+
    4 rows in set (0.00 sec)

对于 LEFT JOIN 经常使用的一个方法是，取出在左表中但是不在右表中的数据(差集)

    (root@localhost) [test]> select * from x left outer join y on x.a = y.a where y.a is NULL;
    +------+------+------+------+
    | a    | b    | a    | b    |
    +------+------+------+------+
    |    3 |   30 | NULL | NULL |
    +------+------+------+------+
    1 row in set (0.00 sec)

    #用子查询也可以写，而且子查询看起来比较好理解
    (root@localhost) [test]> select * from x where a not in (select a from y);
    +------+------+
    | a    | b    |
    +------+------+
    |    3 |   30 |
    +------+------+
    1 row in set (0.00 sec)

### where 和 on 过滤条件两者到底是怎样来进行过滤的呢？

先来看下 inner join，此时将 where 条件写在 on 里是可以的：

    (root@localhost) [test]> select * from x inner join y on x.a = y.a where x.a = 1;
    +------+------+------+---------------------+
    | a    | b    | a    | b                   |
    +------+------+------+---------------------+
    |    1 |    1 |    1 | 2019-03-14 20:29:49 |
    +------+------+------+---------------------+
    1 row in set (0.00 sec)

改成下面的写法 inner join 也是可以的( where 改成了 and )

    (root@localhost) [test]> select * from x inner join y on x.a = y.a and x.a = 1;
    +------+------+------+---------------------+
    | a    | b    | a    | b                   |
    +------+------+------+---------------------+
    |    1 |    1 |    1 | 2019-03-14 20:29:49 |
    +------+------+------+---------------------+
    1 row in set (0.00 sec)

如果写成 where y.a is NULL 就是取出在左表中但是不在右表中的数据

    (root@localhost) [test]> select * from x left outer join y on x.a = y.a where y.a is NULL;
    +------+------+------+------+
    | a    | b    | a    | b    |
    +------+------+------+------+
    |    3 |   30 | NULL | NULL |
    +------+------+------+------+
    1 row in set (0.00 sec)

如果改成 on x.a = y.a and y.a is NULL 右边就会都是 NULL 值

    (root@localhost) [test]> select * from x left outer join y on x.a = y.a and y.a is NULL;
    +------+------+------+------+
    | a    | b    | a    | b    |
    +------+------+------+------+
    |    1 |    1 | NULL | NULL |
    |    2 |   20 | NULL | NULL |
    |    3 |   30 | NULL | NULL |
    +------+------+------+------+
    3 rows in set (0.00 sec)

这是因为首先 x 表的所有都要保留，接着和 y 表进行关联，但是 y 表的关联条件不仅是 x.a = y.a，而且要 y.a 是 NULL 值，也就是说这两张表的关联条件是这整个条件(on x.a = y.a and y.a is NULL)，显然这里 y.a 是 1，2 的时候都不满足因为此时 y.a 都不是 NULL 值，所以这里出现的都是 NULL

对 inner join 来说 where 的过滤条件是可以写在 on 里面。但是对外连接的话 on 是做表之间的关联，where 是用来做其他条件的过滤，千万不要把其他条件的过滤写到 on 里面，否则出来的结果可能是错误的

### 例子 - 返回客户是加拿大的，但是 1997 年没有产生订单的客户

    -- 订单表
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

    -- 订单表 国家表
    (root@localhost) [dbt3]> desc nation;
    +-------------+--------------+------+-----+---------+-------+
    | Field       | Type         | Null | Key | Default | Extra |
    +-------------+--------------+------+-----+---------+-------+
    | n_nationkey | int(11)      | NO   | PRI | NULL    |       |
    | n_name      | char(25)     | YES  |     | NULL    |       |
    | n_regionkey | int(11)      | YES  | MUL | NULL    |       |
    | n_comment   | varchar(152) | YES  |     | NULL    |       |
    +-------------+--------------+------+-----+---------+-------+
    4 rows in set (0.00 sec)

    -- 客户表
    (root@localhost) [dbt3]> desc customer;
    +--------------+--------------+------+-----+---------+-------+
    | Field        | Type         | Null | Key | Default | Extra |
    +--------------+--------------+------+-----+---------+-------+
    | c_custkey    | int(11)      | NO   | PRI | NULL    |       |
    | c_name       | varchar(25)  | YES  |     | NULL    |       |
    | c_address    | varchar(40)  | YES  |     | NULL    |       |
    | c_nationkey  | int(11)      | YES  | MUL | NULL    |       |
    | c_phone      | char(15)     | YES  |     | NULL    |       |
    | c_acctbal    | double       | YES  |     | NULL    |       |
    | c_mktsegment | char(10)     | YES  |     | NULL    |       |
    | c_comment    | varchar(117) | YES  |     | NULL    |       |
    +--------------+--------------+------+-----+---------+-------+
    8 rows in set (0.00 sec)

#### 很容易写错的写法：

    (root@localhost) [dbt3]> SELECT 
        c.c_name, c.c_phone, c.c_address
    FROM
        customer c
            LEFT JOIN
        orders o ON c.c_custkey = o.o_custkey
            LEFT JOIN
        nation n ON c.c_nationkey = n_nationkey
    WHERE
        o.o_orderkey IS NULL
            AND o.o_orderDATE >= '1997-01-01' # 问题出现在这里，其实符合条件的没有下过订单的用户这里都是NULL值，所以用这个字段来进行判断本身就是互斥的是错误的
            AND o.o_orderDATE < '1998-01-01'
            AND n.n_name = 'CANADA';

在写 left join 的时候，要判断一下这里的 where is null 进行过滤的时候和我其他的过滤条件之间是什么样的一个关系，如果这个关系之间是互斥的话，那么建议要把这个关系写在派生表去进行过滤，然后再做 left join

#### 正确的两种写法：

不要直接和 order 表进行关联，先把 orders 表中 97 年的订单数据取出来，然后再和 customer 表中客户表去进行关联

第一种写法：

    (root@localhost) [dbt3]> SELECT 
        c.c_name, c.c_phone, c.c_address
    FROM
        customer c
            LEFT JOIN
        (SELECT 
            *
        FROM
            orders
        WHERE
            o_orderDATE >= '1997-01-01'
                AND o_orderDATE < '1998-01-01') o ON c.c_custkey = o.o_custkey
            LEFT JOIN
        nation n ON c.c_nationkey = n_nationkey
    WHERE
        o.o_orderkey IS NULL
            AND n.n_name = 'CANADA';

也可以写成这样，但是这么写可能不太好理解，上面的写法更符合直觉，而且某些判断条件下也不能这么写，所以还是推荐写成上面那样：

    (root@localhost) [dbt3]> SELECT 
        c.c_name, c.c_phone, c.c_address
    FROM
        customer c
            LEFT JOIN
        orders o ON c.c_custkey = o.o_custkey
            AND o.o_orderDATE >= '1997-01-01'
            AND o.o_orderDATE < '1998-01-01'
            LEFT JOIN
        nation n ON c.c_nationkey = n_nationkey
    WHERE
        o.o_orderkey IS NULL
            AND n.n_name = 'CANADA';

### SQL JOIN -- 问题1

* 行号问题

从表里取数据希望每条记录前面有一个行号，但是如果想要取出来的数据没有一个自增的行号，在 oracal 数据库中有一个叫 row number 的函数，会自动帮你加一个行号的列，但是 MySQL 并没有这样的一个功能

    (oracal适用) [employees]> select * from employees order by emp_no limit 10;
    +--------+------------+------------+-----------+--------+------------+
    | emp_no | birth_date | first_name | last_name | gender | hire_date  |
    +--------+------------+------------+-----------+--------+------------+
    |  10001 | 1953-09-02 | Georgi     | Facello   | M      | 1986-06-26 |
    |  10002 | 1964-06-02 | Bezalel    | Simmel    | F      | 1985-11-21 |
    |  10003 | 1959-12-03 | Parto      | Bamford   | M      | 1986-08-28 |
    |  10004 | 1954-05-01 | Chirstian  | Koblick   | M      | 1986-12-01 |
    |  10005 | 1955-01-21 | Kyoichi    | Maliniak  | M      | 1989-09-12 |
    |  10006 | 1953-04-20 | Anneke     | Preusig   | F      | 1989-06-02 |
    |  10007 | 1957-05-23 | Tzvetan    | Zielinski | F      | 1989-02-10 |
    |  10008 | 1958-02-19 | Saniya     | Kalloufi  | M      | 1994-09-15 |
    |  10009 | 1952-04-19 | Sumant     | Peac      | F      | 1985-02-18 |
    |  10010 | 1963-06-01 | Duangkaew  | Piveteau  | F      | 1989-08-24 |
    +--------+------------+------------+-----------+--------+------------+
    10 rows in set (0.00 sec)

可以用自定义变量实现，首先去 set 一个变量：

    (root@localhost) [employees]> set @a:=0;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [employees]> select @a;
    +------+
    | @a   |
    +------+
    |    0 |
    +------+
    1 row in set (0.00 sec)

    (root@localhost) [employees]> select @a:=@a+1 as rownum,emp_no,birth_date,first_name,last_name,gender,hire_date from employees limit 10;
    +--------+--------+------------+------------+-----------+--------+------------+
    | rownum | emp_no | birth_date | first_name | last_name | gender | hire_date  |
    +--------+--------+------------+------------+-----------+--------+------------+
    |      1 |  10001 | 1953-09-02 | Georgi     | Facello   | M      | 1986-06-26 |
    |      2 |  10002 | 1964-06-02 | Bezalel    | Simmel    | F      | 1985-11-21 |
    |      3 |  10003 | 1959-12-03 | Parto      | Bamford   | M      | 1986-08-28 |
    |      4 |  10004 | 1954-05-01 | Chirstian  | Koblick   | M      | 1986-12-01 |
    |      5 |  10005 | 1955-01-21 | Kyoichi    | Maliniak  | M      | 1989-09-12 |
    |      6 |  10006 | 1953-04-20 | Anneke     | Preusig   | F      | 1989-06-02 |
    |      7 |  10007 | 1957-05-23 | Tzvetan    | Zielinski | F      | 1989-02-10 |
    |      8 |  10008 | 1958-02-19 | Saniya     | Kalloufi  | M      | 1994-09-15 |
    |      9 |  10009 | 1952-04-19 | Sumant     | Peac      | F      | 1985-02-18 |
    |     10 |  10010 | 1963-06-01 | Duangkaew  | Piveteau  | F      | 1989-08-24 |
    +--------+--------+------------+------------+-----------+--------+------------+
    10 rows in set (0.01 sec)

这么写有两个小问题

* 要分两次 sql 语句才能提交
* 第二次执行的时候行号从11开始了，变量需要重新初始化

其实还有这么一种写法，好处是一条 sql 就可以完成，会自动初始化自定义变量

    (root@localhost) [employees]> 
    SELECT 
        @a:=@a + 1 AS rownum,
        emp_no,
        birth_date,
        first_name,
        last_name,
        gender,
        hire_date
    FROM
        employees,
        (SELECT @a:=0) a
    LIMIT 10;

原理是两表关联产生了笛卡尔积，首先select @a:=0；产生了只有一个列的一张表：

    (root@localhost) [employees]> select @a:=0;
    +-------+
    | @a:=0 |
    +-------+
    |     0 |
    +-------+
    1 row in set (0.00 sec)

employees 和一行一列这个表进行关联，出来的还是 N*1 行数据，然后MySQL中允许某个变量在每一行进行变化 (@a:=@a+1)

这就是 MySQL 中求行号问题的一个标准语法

TIPS: limit 取数据不加排序是随机取数据的，因为集合这个东西就是随机的

### SQL JOIN -- 问题2

* 子查询求行号
* 行号通过 count(1) 进行计算，也就是每次计算和第一条数据的数据总数来进行计算行号
* 但是这么做就形成了一个相关子查询，相关子查询的计算的代价是非常大的，每一条数据都要做 count(1)

        (root@localhost) [employees]> SELECT 
            emp_no,
            (SELECT 
                    COUNT(1)
                FROM
                    employees t2
                WHERE
                    t2.emp_no <= t1.emp_no) AS row_num
        FROM
            employees t1
        ORDER BY emp_no
        LIMIT 10;

### ON AND WHERE 区别

on 是表之间的关联的过滤

where 是其他列的过滤条件

所以要清楚的是要关联的条件到底是什么，通常来说 on 后面就是跟关联的字段就可以了不要把其他的字段带进去

TIPS：

倒库的时候可以 buffer_pool 调大一点

    vim /etc/my.cnf

    [mysqld]
    # innodb
    innodb_buffer_pool_size = 1G
    innodb_log_file_size = 128M

## 一些例子 

首先发现有一个员工的数据存在了两天 to_date 一样的数据，这样用 group by 来进行分组的话就会出错

    (root@localhost) [employees]>
    SELECT 
        *
    FROM
        salaries
    WHERE
        (emp_no , to_date) IN (SELECT 
                emp_no, to_date
            FROM
                salaries
            GROUP BY emp_no , to_date
            HAVING COUNT(1) > 1);
    +--------+--------+------------+------------+
    | emp_no | salary | from_date  | to_date    |
    +--------+--------+------------+------------+
    |  10867 |  46330 | 2001-03-22 | 2002-03-22 | <-to_date一样1
    |  10867 |  48248 | 2002-03-22 | 2002-03-22 | <-to_date一样2
    | ...... |  ..... | .......... | .......... |
    | 498586 |  48994 | 1996-07-24 | 1996-07-24 |
    +--------+--------+------------+------------+
    312 rows in set (14.83 sec)
  
那么先筛选掉重复的数据，即一对多表示的情况下求最新的员工的信息：

    (root@localhost) [employees]> 
    SELECT 
        emp_no, salary
    FROM
        salaries
    WHERE
        (emp_no , from_date, to_date) IN (SELECT 
                emp_no, MAX(from_date), MAX(to_date)
            FROM
                salaries
            GROUP BY emp_no);

这样需要扫 2 次才能查询完毕，第一次是派生表的扫描，第二次是外面 select 的扫描

一次扫完的写法(先使用 GROUP_CONCAT，然后再 SUBSTRING_INDEX 取第一个字符串，最后转成整形)：

    (root@localhost) [employees]> 
    SELECT 
        emp_no,
        CAST(SUBSTRING_INDEX(GROUP_CONCAT(salary
                        ORDER BY to_date DESC , from_date DESC),
                    ',',
                    1)
            AS UNSIGNED) AS salary
    FROM
        salaries
    GROUP BY emp_no;