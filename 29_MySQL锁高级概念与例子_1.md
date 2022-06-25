# 锁的概念

* 什么是锁？
  * 对共享资源进行并发访问
  * 提供数据的完整性和一致性
* 每个数据库的锁实现完全不同
  * MyISAM表锁
  * InnoDB行锁
    * Like Oracle？？？
  * Microsoft SQL Server 行级锁with锁升级

* latch
  * mutex
  * rw-lock

## lock 与 latch 的区别

这两个是完全不一样的东西，应用的层面很不一样，lock 是保护数据库的对象，latch 是保护内存的对象

|          | lock                                                  | latch                                                                                |
| -------- | ----------------------------------------------------- | ------------------------------------------------------------------------------------ |
| 对象     | 事务                                                  | 线程                                                                                 |
| 保护     | 数据库的内容，row                                     | 内存数据结构                                                                         |
| 持续时间 | 整个事务过程                                          | 临界资源                                                                             |
| 模式     | 行锁、表锁、意向锁                                    | 读写锁、互斥量                                                                       |
| 死锁     | 通过waits-for graph、time out等机制进行死锁检测与处理 | 无死锁检测与处理机制。仅通过应用程序加锁的顺序（latch leveling）保证无死锁的情况发生 |
| 存在于   | Lock Manager的哈希表中                                | 每个数据结构的对象中                                                                 |

## latch 的查看

    (root@localhost) [test]> show engine innodb mutex;
    +--------+------------------------+---------+
    | Type   | Name                   | Status  |
    +--------+------------------------+---------+
    | InnoDB | rwlock: log0log.cc:846 | waits=8 |
    +--------+------------------------+---------+
    1 row in set (0.12 sec)
 
通过这个命令的数据查看数据库存在哪些故障的意义不大

## InnoDB 存储引擎中的锁

* S 行级共享锁
* X 行级排它锁
* IS 意向共享锁
* IX 意向排它锁
* A 自增锁

共享锁，允许事务读一行记录，共享锁允许其他人读取资源，但是禁止其他人删除，修改资源

排它锁，允许事务删除或更新一行数据，就是对记录进行更改（增删改）的时候就会加上一把排它锁

示例库创建：

    (root@localhost) [test]> create table l( a int primary key, b int, c int ,d int, key(b), unique key idx_c(c));
    Query OK, 0 rows affected (0.03 sec)
    
    (root@localhost) [test]> show create table l\G
    *************************** 1. row ***************************
          Table: l
    Create Table: CREATE TABLE `l` (
      `a` int(11) NOT NULL,
      `b` int(11) DEFAULT NULL,
      `c` int(11) DEFAULT NULL,
      `d` int(11) DEFAULT NULL,
      PRIMARY KEY (`a`),
      UNIQUE KEY `idx_c` (`c`),
      KEY `b` (`b`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8
    1 row in set (0.00 sec)

    (root@localhost) [test]> insert into l values(2,4,6,8);
    Query OK, 1 row affected (0.00 sec)

    (root@localhost) [test]> insert into l values(4,6,8,10);
    Query OK, 1 row affected (0.00 sec)

    (root@localhost) [test]> insert into l values(6,8,10,12);
    Query OK, 1 row affected (0.01 sec)

    (root@localhost) [test]> insert into l values(8,10,12,14);
    Query OK, 1 row affected (0.01 sec)

    (root@localhost) [test]> select * from l;
    +---+------+------+------+
    | a | b    | c    | d    |
    +---+------+------+------+
    | 2 |    4 |    6 |    8 |
    | 4 |    6 |    8 |   10 |
    | 6 |    8 |   10 |   12 |
    | 8 |   10 |   12 |   14 |
    +---+------+------+------+
    4 rows in set (0.00 sec)

加锁操作：

    (root@localhost) [test]> begin;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> delete from l where a = 2;  # 现在这条记录被加锁了
    Query OK, 1 row affected (0.00 sec)

接着执行一个 update 操作：

    (root@localhost) [test]> update l set b = b+1 where a = 4;  # 这里现在也被加上了一把锁
    Query OK, 1 row affected (0.00 sec)
    Rows matched: 1  Changed: 1  Warnings: 0

    (root@localhost) [test]> select * from l;
    +---+------+------+------+
    | a | b    | c    | d    |
    +---+------+------+------+
    | 4 |    7 |    8 |   10 |
    | 6 |    8 |   10 |   12 |
    | 8 |   10 |   12 |   14 |
    +---+------+------+------+
    3 rows in set (0.00 sec)

由于支持 MVCC 功能，现在在另外一个线程中还是原来的数据：

    (会话2@root@localhost) [test]> select * from l;
    +---+------+------+------+
    | a | b    | c    | d    |
    +---+------+------+------+
    | 2 |    4 |    6 |    8 |
    | 4 |    6 |    8 |   10 |
    | 6 |    8 |   10 |   12 |
    | 8 |   10 |   12 |   14 |
    +---+------+------+------+
    4 rows in set (0.00 sec)

然后通过 show engine innodb status 命令查看锁状态：

    (root@localhost) [test]> pager less  #取消是 PAGER set to stdout
    PAGER set to 'less'
    (root@localhost) [test]> show engine innodb status\G
    1 row in set (0.00 sec)
    ------------
    TRANSACTIONS
    ------------
    Trx id counter 678711
    Purge done for trx's n:o < 678711 undo n:o < 0 state: running but idle
    History list length 0
    LIST OF TRANSACTIONS FOR EACH SESSION:
    ---TRANSACTION 421262136963584, not started
    0 lock struct(s), heap size 1136, 0 row lock(s)
    ---TRANSACTION 421262136962672, not started
    0 lock struct(s), heap size 1136, 0 row lock(s)
    ---TRANSACTION 421262136961760, not started
    0 lock struct(s), heap size 1136, 0 row lock(s)
    ---TRANSACTION 678706, ACTIVE 543 sec
    2 lock struct(s), heap size 1136, 2 row lock(s), undo log entries 2  <--这里看到现在有2把记录2 row lock(s)锁和2个undo log entries，2 lock struct(s)则表示有1个记录锁的结构和1个表锁的结构
    MySQL thread id 2, OS thread handle 139785930331904, query id 154 localhost root
    #对应的thread id为2

select 也可以有一个记录锁，然后上面的会变成 3 row lock(s)，不过由于 MVCC 机制这条记录即使被锁住了还是可以读到的：

    (root@localhost) [test]> select * from l where a = 8 for update;
    +---+------+------+------+
    | a | b    | c    | d    |
    +---+------+------+------+
    | 8 |   10 |   12 |   14 |
    +---+------+------+------+
    1 row in set (0.00 sec)

这时候如果在另一个连接里进行同样的操作就会有排它锁的情况发生：

    (会话2@root@localhost) [test]> select * from l where a = 8 for update;
    ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
    (会话2@root@localhost) [test]> select * from l where a = 8;
    +---+------+------+------+
    | a | b    | c    | d    |
    +---+------+------+------+
    | 8 |   10 |   12 |   14 |
    +---+------+------+------+
    1 row in set (0.00 sec)

lock in share mode 共享锁语法，表示对 where a = 8 这条记录加上一个 S 共享锁，共享锁又称读锁，是读取操作创建的锁：

    (root@localhost) [test]> select * from l where a = 8 lock in share mode;
    ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction

ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction 是锁等待超时的意思，不要和死锁概念搞混了，可以通过参数设置这个超时时间：

    (root@localhost) [test]> show variables like 'innodb%lock%';
    +--------------------------------+-------+
    | Variable_name                  | Value |
    +--------------------------------+-------+
    | innodb_lock_wait_timeout       | 50    |
    +--------------------------------+-------+
    10 rows in set (0.00 sec)

建议设为 2 秒或者 3 秒

    (root@localhost) [test]> set global innodb_lock_wait_timeout = 3;
    Query OK, 0 rows affected (0.00 sec)

my.cnf：

    # innodb
    ...
    innodb_flush_neighbors = 0
    innodb_lock_wait_timeout = 3 # new

## 意向锁

* 揭示下一层级请求的锁类型
* IS：事务想要获得一张表中某几行的共享锁
* IX：事务想要获得一张表中某几行的排它锁
* InnoDB存储引擎中意向锁都是**表锁**

| 锁兼容性 | S   | X   |
| -------- | --- | --- |
| S        | ✔   | ❌   |
| X        | ❌   | ❌   |

| 意向锁兼容性 | IS  | IX  |
| ------------ | --- | --- |
| IS           | ✔   | ✔   |
| IX           | ✔   | ✔   |

数据是支持在多个粒度级别加锁的，对于 MySQL 来说支持表级别的锁和行级别的锁

意向锁是用来实现多粒度级别的锁

意向锁对行进行加锁的话是从上往下进行进行加锁，可以加在数据库、表、页、、记录上，是多粒度级别的

对一个对象进行加锁，哪怕这个对象是最低级别的记录锁，它也是从上往下一层一层进行加锁的，S 和 X 可以在多个粒度级别上进行加，意向锁表示的下一层级要加什么锁，对当前层级大家都是互相兼容的

在 MySQL 中没有数据库级别的锁和页级别的锁，只有表锁和记录级别锁两个层次，这就意味着所有的意向锁最上层都是加在表上面的，所以 MySQL 文档里才说 InnoDB 存储引擎中的意向锁都是表锁

## 通过 innodb_status_output_locks 参数看到锁等待状态

(root@localhost) [test]> show variables like 'innodb%lock%';
+--------------------------------+-------+
| Variable_name                  | Value |
+--------------------------------+-------+
| innodb_status_output_locks     | OFF   |
+--------------------------------+-------+
10 rows in set (0.00 sec)

打开这个参数可以输出锁的信息：

    (root@localhost) [test]> set global innodb_status_output_locks=1;
    Query OK, 0 rows affected (0.00 sec)

这时候相对之前输出了更多的锁信息：

    (root@localhost) [test]> show engine innodb status\G
    2 lock struct(s), heap size 1136, 3 row lock(s), undo log entries 2
    MySQL thread id 3, OS thread handle 139908548077312, query id 10 localhost root
    TABLE LOCK table `test`.`l` trx id 679172 lock mode IX  <--这里表示有一个test库l表的IX表锁
    RECORD LOCKS space id 185 page no 3 n bits 72 index PRIMARY of table `test`.`l` trx id 679172 lock_mode X locks rec but not gap   <--这里表示有一个记录锁，space id 185 page no 3 表示的是表空间的第4个页，index PRIMARY表示是在主键的聚集索引上加了一把锁，锁住的锁模式是X
    ##---- X锁是对这下面这三条记录加了X锁
    Record lock, heap no 2 PHYSICAL RECORD: n_fields 6; compact format; info bits 32
    0: len 4; hex 80000002; asc     ;;          <--主键列，如果没有主键就会用rowid代替，对应第一行的2那条记录
    1: len 6; hex 0000000a5d04; asc     ] ;;    <--隐藏列，事务id列
    2: len 7; hex 2400000f702757; asc $   p'W;; <--隐藏列，回滚指针
    3: len 4; hex 80000004; asc     ;;          <--对应第一行的4那条记录
    4: len 4; hex 80000006; asc     ;;          <--对应第一行的6那条记录
    5: len 4; hex 80000008; asc     ;;          <--对应第一行的8那条记录

    Record lock, heap no 3 PHYSICAL RECORD: n_fields 6; compact format; info bits 0
    0: len 4; hex 80000004; asc     ;;
    1: len 6; hex 0000000a5d04; asc     ] ;;
    2: len 7; hex 2400000f702786; asc $   p' ;;
    3: len 4; hex 80000007; asc     ;;
    4: len 4; hex 80000008; asc     ;;
    5: len 4; hex 8000000a; asc     ;;

    Record lock, heap no 5 PHYSICAL RECORD: n_fields 6; compact format; info bits 0
    0: len 4; hex 80000008; asc     ;;
    1: len 6; hex 0000000a5b2d; asc     [-;;
    2: len 7; hex a400000fc50110; asc        ;;
    3: len 4; hex 8000000a; asc     ;;
    4: len 4; hex 8000000c; asc     ;;
    5: len 4; hex 8000000e; asc     ;;

如果在其他连接中进行了加锁，这个命令也会显示出相关阻塞的信息：

    (会话2@root@localhost) [test]> select * from l where a = 2 for update;

    (root@localhost) [test]> show engine innodb status\G
    ...
    ---TRANSACTION 679178, ACTIVE 2 sec starting index read   <--如果有事务在等待会产生这样的信息
    mysql tables in use 1, locked 1           
    LOCK WAIT 2 lock struct(s), heap size 1136, 1 row lock(s) <--这里表示锁等待，2个锁结构，有一条记录锁
    MySQL thread id 5, OS thread handle 139908294240000, query id 107 localhost root statistics
    select * from l where a = 2 for update                    <--等待事务的语句
    ------- TRX HAS BEEN WAITING 2 SEC FOR THIS LOCK TO BE GRANTED:  <--WAITING 2 SEC 表示这个事务已经等待了2秒了，等待的是下面这把锁被受理
    RECORD LOCKS space id 185 page no 3 n bits 72 index PRIMARY of table `test`.`l` trx id 679178 lock_mode X locks rec but not gap waiting
    Record lock, heap no 2 PHYSICAL RECORD: n_fields 6; compact format; info bits 32
    0: len 4; hex 80000002; asc     ;;      <--等待主键值为2的记录的处理
    1: len 6; hex 0000000a5d04; asc     ] ;;
    2: len 7; hex 2400000f702757; asc $   p'W;;
    3: len 4; hex 80000004; asc     ;;
    4: len 4; hex 80000006; asc     ;;
    5: len 4; hex 80000008; asc     ;;

    ------------------
    TABLE LOCK table `test`.`l` trx id 679178 lock mode IX
    RECORD LOCKS space id 185 page no 3 n bits 72 index PRIMARY of table `test`.`l` trx id 679178 lock_mode X locks rec but not gap waiting
    Record lock, heap no 2 PHYSICAL RECORD: n_fields 6; compact format; info bits 32
    0: len 4; hex 80000002; asc     ;;
    1: len 6; hex 0000000a5d04; asc     ] ;;
    2: len 7; hex 2400000f702757; asc $   p'W;;
    3: len 4; hex 80000004; asc     ;;
    4: len 4; hex 80000006; asc     ;;
    5: len 4; hex 80000008; asc     ;;

    ---TRANSACTION 679172, ACTIVE 15507 sec
    2 lock struct(s), heap size 1136, 3 row lock(s), undo log entries 2
    MySQL thread id 3, OS thread handle 139908548077312, query id 10 localhost root   <--造成阻塞的主键2的操作是被thread id 3这个事务所获得的
    TABLE LOCK table `test`.`l` trx id 679172 lock mode IX
    RECORD LOCKS space id 185 page no 3 n bits 72 index PRIMARY of table `test`.`l` trx id 679172 lock_mode X locks rec but not gap
    Record lock, heap no 2 PHYSICAL RECORD: n_fields 6; compact format; info bits 32
    0: len 4; hex 80000002; asc     ;;
    1: len 6; hex 0000000a5d04; asc     ] ;;
    2: len 7; hex 2400000f702757; asc $   p'W;;
    3: len 4; hex 80000004; asc     ;;
    4: len 4; hex 80000006; asc     ;;
    5: len 4; hex 80000008; asc     ;;
    ...

通过 show processlist 命令可以看到 thread id 3 的实例：

    (root@localhost) [test]> show processlist;
    +----+------+----------------+-----------+---------+-------+----------+------------------+
    | Id | User | Host           | db        | Command | Time  | State    | Info             |
    +----+------+----------------+-----------+---------+-------+----------+------------------+
    |  3 | root | localhost      | test      | Sleep   | 16817 |          | NULL             |
    |  4 | root | localhost      | test      | Sleep   |  1316 |          | NULL             |
    |  5 | root | localhost      | test      | Query   |     0 | starting | show processlist |
    |  6 | neo  | _gateway:54442 | employees | Sleep   |   387 |          | NULL             |
    |  7 | neo  | _gateway:54448 | employees | Sleep   |   387 |          | NULL             |
    +----+------+----------------+-----------+---------+-------+----------+------------------+
    5 rows in set (0.00 sec)

需要注意的是在 treads 表里对应的是 PROCESSLIST_ID，而不是 THREAD_ID：

    (root@localhost) [performance_schema]> desc threads;
    +---------------------+---------------------+------+-----+---------+-------+
    | Field               | Type                | Null | Key | Default | Extra |
    +---------------------+---------------------+------+-----+---------+-------+
    | THREAD_ID           | bigint(20) unsigned | NO   |     | NULL    |       |  <--这张表的内部自增ID
    | ...                                                                      |
    | PROCESSLIST_ID      | bigint(20) unsigned | YES  |     | NULL    |       |  <--对应show processlist中的Id
    | ...                                                                      |
    | THREAD_OS_ID        | bigint(20) unsigned | YES  |     | NULL    |       |  <--系统进程号
    +---------------------+---------------------+------+-----+---------+-------+
    17 rows in set (0.00 sec)

意向锁之间是互相兼容的，意向锁和表锁和 S、X 锁之间是冲突的，因为如果要加表锁的话需要加一个 S lock，这时候就有意向锁去等待排它的 S 锁

## MySQL5.7 中查看锁等待状态

MySQL5.7 中就方便很多了，使用 sys 库下面的 innodb_lock_waits 表即可查询相关数据

    (root@localhost) [sys]> show tables like '%lock%';
    +---------------------------+
    | Tables_in_sys (%lock%)    |
    +---------------------------+
    | innodb_lock_waits         |
    | schema_table_lock_waits   |
    | x$innodb_lock_waits       |
    | x$schema_table_lock_waits |
    +---------------------------+
    4 rows in set (0.00 sec)

如果没有锁等待这张表是看不到数据的，为了演示方便把锁等待时间先调大一点：

    (root@localhost) [test]> set innodb_lock_wait_timeout = 50;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [sys]> select * from innodb_lock_waits\G
    *************************** 1. row ***************************
                    wait_started: 2022-06-22 21:18:18   <--等待开始时间
                        wait_age: 00:00:17
                  wait_age_secs: 17
                    locked_table: `test`.`l`
                    locked_index: PRIMARY
                    locked_type: RECORD
                  waiting_trx_id: 679179
            waiting_trx_started: 2022-06-22 21:12:11
                waiting_trx_age: 00:06:24
        waiting_trx_rows_locked: 2
      waiting_trx_rows_modified: 0
                    waiting_pid: 4
                  waiting_query: select * from test.l for update  <--等待的语句
                waiting_lock_id: 679179:185:3:2
              waiting_lock_mode: X
                blocking_trx_id: 679172   <--表示谁阻塞它
                    blocking_pid: 3
                  blocking_query: NULL
                blocking_lock_id: 679172:185:3:2
              blocking_lock_mode: X
            blocking_trx_started: 2022-06-22 15:48:50
                blocking_trx_age: 05:29:45
        blocking_trx_rows_locked: 3
      blocking_trx_rows_modified: 2
        sql_kill_blocking_query: KILL QUERY 3   <--可以KILL掉阻塞的查询
    sql_kill_blocking_connection: KILL 3        <--这个KILL会把阻塞的连接杀掉，需要小心
    1 row in set, 3 warnings (0.03 sec)

blocking_query 为 NULL 的原因是在执行这条语句前，造成阻塞的语句已经执行过了，所以这里它并不知道是哪些语句造成了阻塞。不过这里有时候也能看到，就是刚好阻塞的语句还没执行结束的时候

### heap no

heap no 表示的是记录插入的顺序，表示的是一个页中的记录是什么时候被插入的，记录之间是逻辑有序的，但并不是通过 heap no 来表示的

    (root@localhost) [test]> rollback;
    Query OK, 0 rows affected (0.01 sec)

    (root@localhost) [test]> select * from l;
    +---+------+------+------+
    | a | b    | c    | d    |
    +---+------+------+------+
    | 2 |    4 |    6 |    8 |
    | 4 |    6 |    8 |   10 |
    | 6 |    8 |   10 |   12 |
    | 8 |   10 |   12 |   14 |
    +---+------+------+------+
    4 rows in set (0.00 sec)

    (root@localhost) [test]> begin;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> delete from l where a = 2;
    Query OK, 1 row affected (0.00 sec)

    (root@localhost) [test]> delete from l where a = 4;
    Query OK, 1 row affected (0.00 sec)

    (root@localhost) [test]> delete from l where a = 6;
    Query OK, 1 row affected (0.00 sec)

在另外一个连接里查看：

    (root@localhost) [test]> show engine innodb status\G
    1 row in set (0.00 sec)

    ---TRANSACTION 679184, ACTIVE 83 sec
    2 lock struct(s), heap size 1136, 3 row lock(s), undo log entries 3
    MySQL thread id 3, OS thread handle 139908548077312, query id 177 localhost root
    TABLE LOCK table `test`.`l` trx id 679184 lock mode IX
    RECORD LOCKS space id 185 page no 3 n bits 72 index PRIMARY of table `test`.`l` trx id 679184 lock_mode X locks rec but not gap
    Record lock, heap no 2 PHYSICAL RECORD: n_fields 6; compact format; info bits 32
    0: len 4; hex 80000002; asc     ;;
    1: len 6; hex 0000000a5d10; asc     ] ;;
    2: len 7; hex 2c0000105a200c; asc ,   Z  ;;
    3: len 4; hex 80000004; asc     ;;
    4: len 4; hex 80000006; asc     ;;
    5: len 4; hex 80000008; asc     ;;

    Record lock, heap no 3 PHYSICAL RECORD: n_fields 6; compact format; info bits 32
    0: len 4; hex 80000004; asc     ;;
    1: len 6; hex 0000000a5d10; asc     ] ;;
    2: len 7; hex 2c0000105a203b; asc ,   Z ;;;
    3: len 4; hex 80000006; asc     ;;
    4: len 4; hex 80000008; asc     ;;
    5: len 4; hex 8000000a; asc     ;;

    Record lock, heap no 4 PHYSICAL RECORD: n_fields 6; compact format; info bits 32
    0: len 4; hex 80000006; asc     ;;
    1: len 6; hex 0000000a5d10; asc     ] ;;
    2: len 7; hex 2c0000105a206a; asc ,   Z j;;
    3: len 4; hex 80000008; asc     ;;
    4: len 4; hex 8000000a; asc     ;;
    5: len 4; hex 8000000c; asc     ;;

对于 heap no 2 这条记录进行加锁，是通过 (sapce, heap no) 来进行定位的，表示对哪一条记录进行加锁

一个页如果一条记录都没有的话，InnoDB 默认会生成两条记录，一个叫最小记录一个叫最大记录，这两个记录是虚拟的伪记录，不管页里有没有记录每个页都存在这两个记录

最小记录的 heap no 值是 0，最大记录的 heap no 值是 1，所以用户插入的记录的 heap no 值都是从 2 开始的

# InnoDB锁的算法

* **每个事务每个页**一个锁对象
  * 30~100字节
* 通过位图存放锁信息
  * 内存使用率小
* 没有锁升级
  * like Oracle

每个页会有一个锁对象而不是每条记录有一个锁对象，是根据每个页来进行管理的，页里面哪条记录进行加锁了，就把它 heap no 对应的值设置为 1，表示就进行加锁了

所以 InnoDB 的锁是占用内存的

