# 事务

* A Atomicity 原子性
  * redo
* C consistency 一致性
  * undo 
* I isolation 隔离性
  * lock
* D Durable 持久性
  * redo & undo

原子性代表操作是原子的，要么做要么都不做

一致性表示状态前后是一致的

隔离性和事务的隔离级别是两个概念，隔离性表示的是一个事务所作的修改对其他是事务是不可见的

持久性就是一旦写了数据就不会丢失了

## 事务的使用方法

有下面的表：

    (root@localhost) [test]> show create table z\G
    *************************** 1. row ***************************
          Table: z
    Create Table: CREATE TABLE `z` (
      `b` int(11) DEFAULT NULL,
      UNIQUE KEY `idx_b` (`b`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8
    1 row in set (0.00 sec)

开启事务和提交：

    (root@localhost) [test]> select * from z;
    +------+
    | b    |
    +------+
    | NULL |
    |    1 |
    |    2 |
    |    3 |
    |   10 |
    |   20 |
    |   30 |
    |   40 |
    +------+
    8 rows in set (0.00 sec)

    (root@localhost) [test]> begin;   # 开始事务
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> insert into z values(50);  # 插入数据
    Query OK, 1 row affected (0.00 sec)

    (root@localhost) [test]> insert into z values(60);  # 插入数据
    Query OK, 1 row affected (0.00 sec)

    (root@localhost) [test]> insert into z values(70);  # 插入数据
    Query OK, 1 row affected (0.01 sec)

    (root@localhost) [test]> commit;    # 提交事务
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> select * from z;
    +------+
    | b    |
    +------+
    | NULL |
    |    1 |
    |    2 |
    |    3 |
    |   10 |
    |   20 |
    |   30 |
    |   40 |
    |   50 |
    |   60 |
    |   70 |
    +------+
    11 rows in set (0.00 sec)

开启事务和回滚：

    (root@localhost) [test]> start transaction;   # 开始事务
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> insert into z values(80);  # 插入数据
    Query OK, 1 row affected (0.00 sec)

    (root@localhost) [test]> insert into z values(90);  # 插入数据
    Query OK, 1 row affected (0.00 sec)

    (root@localhost) [test]> insert into z values(100); # 插入数据
    Query OK, 1 row affected (0.00 sec)

    (root@localhost) [test]> rollback;    # 回滚事务
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> select * from z;
    +------+
    | b    |
    +------+
    | NULL |
    |    1 |
    |    2 |
    |    3 |
    |   10 |
    |   20 |
    |   30 |
    |   40 |
    |   50 |
    |   60 |
    |   70 |
    +------+
    11 rows in set (0.00 sec)

如果不用 begin 的话，SQL 语句就会自动提交 autocommit，不能进行 rollback，这个特性可以通过参数关闭，不过一般开启就行：

    (root@localhost) [test]> show variables like 'autocommit';
    +---------------+-------+
    | Variable_name | Value |
    +---------------+-------+
    | autocommit    | ON    |
    +---------------+-------+
    1 row in set (0.01 sec)

保存点 savepoint：

    (root@localhost) [test]> begin;   # 事务开始
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> insert into z values(120); # 插入数据
    Query OK, 1 row affected (0.00 sec)

    (root@localhost) [test]> savepoint s1;  # 建立保存点
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> insert into z values(140); # 插入数据
    Query OK, 1 row affected (0.00 sec)

    (root@localhost) [test]> insert into z values(160); # 插入数据
    Query OK, 1 row affected (0.00 sec)

    (root@localhost) [test]> rollback to s1;    # 回滚到保存点s1，但是此时整个事务并没有回滚也没有提交
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> # 可以接着继续操作，继续插入数据或者commit

小技巧，在线上开发人员经常会写错代码，基本上就是没要处理对一些错误的逻辑导致有事务没提交：

    (root@localhost) [test]> begin;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> insert into z values(200);
    Query OK, 1 row affected (0.00 sec)

    (root@localhost) [test]> insert into z values(120);
    ERROR 1062 (23000): Duplicate entry '120' for key 'idx_b'

出错后并没有进行提交或者回滚，一直处于这个状态，导致这个事务对应的所有锁（行锁）占用的资源都没有释放，如果其他的线程用到了这里对应的资源的话，会导致这些线程都没有办法继续进行提交

通过 show processlist 命令可以看到一些 sleep 状态的线程，就是疑似有问题的：

    (root@localhost) [(none)]> show processlist;
    +----+------+----------------+-----------+---------+------+----------+------------------+
    | Id | User | Host           | db        | Command | Time | State    | Info             |
    +----+------+----------------+-----------+---------+------+----------+------------------+
    |  2 | root | localhost      | test      | Sleep   |  247 |          | NULL             | <--这个就是疑似事务没提交的
    |  5 | root | localhost      | NULL      | Query   |    0 | starting | show processlist |
    +----+------+----------------+-----------+---------+------+----------+------------------+
    4 rows in set (0.00 sec)

通过 kill 命令回滚掉：

    (root@localhost) [(none)]> kill 2;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [(none)]> show processlist;
    +----+------+----------------+-----------+---------+------+----------+------------------+
    | Id | User | Host           | db        | Command | Time | State    | Info             |
    +----+------+----------------+-----------+---------+------+----------+------------------+
    |  5 | root | localhost      | NULL      | Query   |    0 | starting | show processlist |
    +----+------+----------------+-----------+---------+------+----------+------------------+
    3 rows in set (0.00 sec)

## type

* Flat Transactions                 平事务，begin 开头 commit 结束的，大部分事务都是平事务
* Flat Transactions with Savepoint 
* Chanied Transactions              链事务，MySQL中用不太到
* Nested Transactions               嵌套事务，大部分数据库都不支持
* Distributed Transactions          分布式事务
* Long Transactions                 长事务，大事务

## 事务隔离级别

    (root@localhost) [(none)]> show variables like 'tx_isolation';
    +---------------+-----------------+
    | Variable_name | Value           |
    +---------------+-----------------+
    | tx_isolation  | REPEATABLE-READ |
    +---------------+-----------------+
    1 row in set (0.00 sec)

事务隔离要解决三个问题：

* 脏读
* 不可重复读
* 幻读

### 设置事务隔离级别

    (root@localhost) [test]> set tx_isolation = 'read-committed'; #对当前会话有效
    Query OK, 0 rows affected, 1 warning (0.00 sec)

    (root@localhost) [test]> set global tx_isolation = 'read-committed'; #对之后创建的连接有效
    Query OK, 0 rows affected, 1 warning (0.00 sec)

### 查看当前各个线程的事务隔离级别

先查看各个连接的事务隔离级别：

    (root@localhost) [performance_schema]> select * from variables_by_thread where variable_name = 'tx_isolation' limit 10;
    +-----------+---------------+-----------------+
    | THREAD_ID | VARIABLE_NAME | VARIABLE_VALUE  |
    +-----------+---------------+-----------------+
    |        39 | tx_isolation  | REPEATABLE-READ |
    |        40 | tx_isolation  | READ-COMMITTED  |
    +-----------+---------------+-----------------+
    2 rows in set (0.00 sec)

然后去 threads 查找 PROCESSLIST_ID：

    (root@localhost) [performance_schema]> select * from threads where thread_id in(39,40)\G
    *************************** 1. row ***************************
              THREAD_ID: 39
                  NAME: thread/sql/one_connection
                  TYPE: FOREGROUND
        PROCESSLIST_ID: 2
      PROCESSLIST_USER: root
      PROCESSLIST_HOST: localhost
        PROCESSLIST_DB: performance_schema
    PROCESSLIST_COMMAND: Query
      PROCESSLIST_TIME: 0
      PROCESSLIST_STATE: Sending data
      PROCESSLIST_INFO: select * from threads where thread_id in(39,40)
      PARENT_THREAD_ID: 1
                  ROLE: NULL
          INSTRUMENTED: YES
                HISTORY: YES
        CONNECTION_TYPE: Socket
          THREAD_OS_ID: 26096
    *************************** 2. row ***************************
              THREAD_ID: 40
                  NAME: thread/sql/one_connection
                  TYPE: FOREGROUND
        PROCESSLIST_ID: 3
      PROCESSLIST_USER: root
      PROCESSLIST_HOST: localhost
        PROCESSLIST_DB: NULL
    PROCESSLIST_COMMAND: Sleep
      PROCESSLIST_TIME: 42
      PROCESSLIST_STATE: NULL
      PROCESSLIST_INFO: NULL
      PARENT_THREAD_ID: 1
                  ROLE: NULL
          INSTRUMENTED: YES
                HISTORY: YES
        CONNECTION_TYPE: Socket
          THREAD_OS_ID: 27696
    2 rows in set (0.00 sec)

最后使用 show processlist 对应以下就知道对应的线程 ID 了：

    (root@localhost) [performance_schema]> show processlist;
    +----+------+-----------+--------------------+---------+------+----------+------------------+
    | Id | User | Host      | db                 | Command | Time | State    | Info             |
    +----+------+-----------+--------------------+---------+------+----------+------------------+
    |  2 | root | localhost | performance_schema | Query   |    0 | starting | show processlist |
    |  3 | root | localhost | NULL               | Sleep   |  176 |          | NULL             |
    +----+------+-----------+--------------------+---------+------+----------+------------------+
    2 rows in set (0.00 sec)

### read-uncommitted 隔离级别，基本不用

能够读到没有提交的事务，就是如果某个事务插入了一些数据但是还没有提交，这个时候去进行查询时可以看到插入操作的数据的，也就是可以读到没有提交的数据，这个就叫**脏读**

    (root@localhost) [test]> set tx_isolation = 'read-uncommitted';
    Query OK, 0 rows affected, 1 warning (0.00 sec)

    (root@localhost) [test]> begin;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> insert into z values(200);
    Query OK, 1 row affected (0.00 sec)

    (root@localhost) [test]> select * from z;
    +------+
    | b    |
    +------+
    | NULL |
    |    1 |
    |    2 |
    |    3 |
    |   10 |
    |   20 |
    |   30 |
    |   40 |
    |   50 |
    |   60 |
    |   70 |
    |  120 |
    |  200 |   <--注意看这里
    +------+
    13 rows in set (0.00 sec)

### read-committed 读到提交的数据

read-committed 会碰到不可重复读的问题

也就是本来不会读到没有提交的数据，但是 RC 事务隔离级别就是能够看到其他事务提交后的数据，举个例子

客户端连接 1 中执行下列操作：

    (root@localhost) [test]> show variables like 'tx_isolation';
    +---------------+-----------------+
    | Variable_name | Value           |
    +---------------+-----------------+
    | tx_isolation  | REPEATABLE-READ |   <--RR事务隔离级别
    +---------------+-----------------+
    1 row in set (0.00 sec)

    (root@localhost) [test]> begin;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> insert into z values(200);
    Query OK, 1 row affected (0.00 sec)

    (root@localhost) [test]> select * from z;
    +------+
    | b    |
    +------+
    | NULL |
    |    1 |
    |    2 |
    |    3 |
    |   10 |
    |   20 |
    |   30 |
    |   40 |
    |   50 |
    |   60 |
    |   70 |
    |  120 |
    |  200 |
    +------+
    13 rows in set (0.00 sec)

这时候在客户端连接 2 中查看没有 200 这条数据：

    (root@localhost) [test]> show variables like 'tx_isolation';
    +---------------+----------------+
    | Variable_name | Value          |
    +---------------+----------------+
    | tx_isolation  | READ-COMMITTED |  <--RC事务隔离级别
    +---------------+----------------+
    1 row in set (0.00 sec)

    (root@localhost) [test]> begin;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> select * from z where b = 200;
    Query OK, 0 rows affected (0.00 sec)

然后把连接 1 中的事务提交：

    (root@localhost) [test]> commit;
    Query OK, 0 rows affected (0.00 sec)

然后在连接 2 中可以查看到 200 这条数据了：

    (root@localhost) [test]> select * from z where b = 200;
    +------+
    | b    |
    +------+
    |  200 |
    +------+
    1 row in set (0.00 sec)

这就是问题所在了，这种情况两次读到的数据可能会不一样，这就破坏了隔离性的要求，事务隔离级别应该是互相不可见的串行原则

RC 事务隔离级别下，除了唯一性的约束检查和外键约束的检查需要 gap lock，InnoDB 存储引擎不会使用 gap lock 的锁算法

### repeatable-read

用来解决不可重复读的问题

    (会话2@root@localhost) [test]> set tx_isolation = 'repeatable-read';
    Query OK, 0 rows affected, 1 warning (0.00 sec)

    (连接1@root@localhost) [test]> begin;
    Query OK, 0 rows affected (0.00 sec)

    (连接1@root@localhost) [test]> insert into z values(300);
    Query OK, 1 row affected (0.00 sec)

    (会话2@root@localhost) [test]> begin;
    Query OK, 0 rows affected (0.00 sec)

    (会话2@root@localhost) [test]> select * from z where b = 300;
    Empty set (0.00 sec)

    (连接1@root@localhost) [test]> commit;
    Query OK, 0 rows affected (0.01 sec)

    (会话2@root@localhost) [test]> select * from z where b = 300;
    Empty set (0.00 sec)

可以看到在 RR 事务隔离级别下，连接 2 事务中的两次查询都不会读到 300 这条记录，这就叫重复读，重复读两次读到的数据都是一样的

注意是在事务读不到，回滚事务或者提交事务之后，在事务外是可以正常读到的，所以叫才叫**事务隔离**

重复读还是有一个问题，就是**幻读**，幻读的表示的意思是读到了之前不存在的记录，repeatable-read 事务隔离级别就能解决幻读问题

并发操作解决 3 个问题的话就能达到真正的隔离性的要求，一个叫做**脏读**一个叫做**不可重复读**一个叫做**幻读**

repeatable-read 虽然解决了脏读、不可重复读、幻读的问题，但也只满足了 99.99% 的隔离性要求而不是真正的隔离性要求

是因为如果执行了 for update 加锁操作就能读到 300 的记录：

    (会话2@root@localhost) [test]> select * from z where b = 300 for update;
    +------+
    | b    |
    +------+
    |  300 | <--幻读，不符合隔离性要求的现象
    +------+
    1 row in set (0.00 sec)

### serializable

表示的意思是对于所有的读和写每行上面都有锁，所有操作都要加锁

    (root@localhost) [test]> set tx_isolation = 'serializable';
    Query OK, 0 rows affected, 1 warning (0.00 sec)

### 配置文件默认建议设置为RC事务隔离级别，这点非常重要

对于大部分场景，简单来说 RC 的性能会比 RR 好非常多，RC 的锁理解起了也比较简单

如果设置成 RC，binlog_format 必须设置成 row

my.cnf：

    [mysqld]
    ...
    transaction_isolation = read-committed #new
    binlog_format = row #new

只有如果你的应用是偏读的，业务类型里面一个事务有很多的 SELECT，那么这个时候 RR 会比 RC 的性能来的好

# REDO 

* redo组成
  * redo log buffer
  * --innodb_log_buffer
  * 通常8M已经足够使用
* redo log file
  * --innodb_log_file_size
  * --innodb_log_files_in_group
  * --innodb_log_group_home_dir
    * 和数据文件分开
    * 选择更快的磁盘

在数据目录下面的 ib_logfile0、ib_logfile1 就是 redo 重做日志文件：

    $ ll /mdata/mysql_test_data/
    total 347M
    ...
    -rw-r----- 1 mysql mysql 128M Jun 21 14:51 ib_logfile0
    -rw-r----- 1 mysql mysql 128M Jun 12 10:55 ib_logfile1
    ...

重做日志写入到重做日志文件的时候，会先写入到 redo log buffer 内存里面，然后再定期的刷新到磁盘里面：

    (root@localhost) [(none)]> show variables like 'innodb%log%';
    +----------------------------------+------------+
    | Variable_name                    | Value      |
    +----------------------------------+------------+
    ...
    | innodb_log_buffer_size           | 16777216   |
    ...
    | innodb_log_file_size             | 134217728  |
    +----------------------------------+------------+
    15 rows in set (0.00 sec)

innodb_log_buffer_size 并不需要很多的内存，因为它会定期的刷新到磁盘

重做日志表示的是记录了对每个页的修改操作，也叫做物理逻辑日志，意思是记录每个页操作的逻辑内容，而不是完全物理的二进制差异值

* redo log buffer 
  * 由log block组成
  * 每个log block 512字节
    * no need doublewrite

* redo log buffer 刷新条件
  * master thread 每秒进行刷新
  * redo log buffer 使用大于1/2进行刷新
  * 事务提交时进行刷新
    * innodb_flush_log_at_trx_commit={0|1|2}

innodb_flush_log_at_trx_commit 这个参数非常重要，意思是当一个事务进行提交了，即使其他条件没有满足，也会将 innodb_log_buffer 内存中的重做日志写入到磁盘，这个参数默认值的 1 就代表这种效果，不需要修改

0 表示事务提交的时候不刷新到磁盘，也就是依赖 master thread 来刷新，就有可能最多丢 1 秒的数据

2 表示事务提交时写入到操作系统缓存，就是当 MySQL 重启操作系统没有重启的话数据是不会丢失的

## 组提交

* 一次fsync刷新多个事务
  * 性能提高10~100+倍
  * sysbench update_non_index.lua
* InnoDB存储引擎原生支持

innodb_flush_log_at_trx_commit = 1 参数设置的意思就是事务提交的时候确保日志要写盘

所谓的写盘调用的是 fsync API 接口写磁盘，fsync 是一个 IO 操作

一个非常有意思的问题，假设一个 HDD 硬盘的 IOPS 只有 100，这意味着它每秒只能执行 100 次 fsync，对应的增删改 max(QPS) 就是 100，这边指的 QSP 是每个是事务里面每做一个增删改操作就提交

这样的话数据库性能就太差了，比如批量导数据的时候不要一条一条记录插，可以先开启一个 begin，十条记录十条记录的插，也就是 insert 十条记录后再 commit

这样批量做的话 QPS 就变成了 100*10 了，性能就提升了，不过这样的性能提升需要应用端改写代码配合才能实现

组提交相关参数：

    (root@localhost) [(none)]> show variables like '%group%';
    +-----------------------------------------+-------+
    | Variable_name                           | Value |
    +-----------------------------------------+-------+
    | binlog_group_commit_sync_delay          | 0     |
    | binlog_group_commit_sync_no_delay_count | 0     |
    +-----------------------------------------+-------+
    6 rows in set (0.02 sec)

binlog_group_commit_sync_delay 表示组提交一定要等待多少微秒，这个参数设大后一次组提交能提交的事务变多了，本来设为 0 的话提交的时候有多少就提交多少，如果等待个 100 毫秒累积的事务量变多提交后 fsync 的次数又变少了

binlog_group_commit_sync_no_delay_count 参数表示一定累积到多少个后提交

因为这两个值的调优很难，所以不建议去调整这两个参数，默认认为数据库组提交已经完成的足够好就可以了

## sync_binlog

    (root@localhost) [(none)]> show variables like 'sync_binlog';
    +---------------+-------+
    | Variable_name | Value |
    +---------------+-------+
    | sync_binlog   | 1     |
    +---------------+-------+
    1 row in set (0.01 sec)


默认值是 1，表示事务提交的时候写二进制日志

设成 0 的话表示写操作系统缓存

二进制日志是用来做同步的

设置成 0 有数据丢失的风险