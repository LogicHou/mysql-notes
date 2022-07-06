# MySQL 复制技术 基础

复制（Replication）

数据从一台 MySQL 复制到一台或者多台 MySQL 的服务器上面，这就是主从，分发的那台服务器叫 master，接收的服务器就叫 slave，比如一主两从的高可用架构，复制可能会有一些小小的延迟，但是这个集群里的数据最终应该是一致的

|      | MySQL                                                                | Oracle Data Gurad    | SQL Server Mirroring |
| ---- | -------------------------------------------------------------------- | -------------------- | -------------------- |
| 类型 | 逻辑复制                                                             | 物理逻辑复制         | 物理逻辑复制         |
| 有点 | 灵活                                                                 | 复制速度快           | 复制速度快           |
| 缺点 | 配置不当易出错，速度比较慢<br>事务提交的时候才同步，和事务大小有关系 | 要求物理数据严格一致 | 要求物理数据严格一致 |

## 复制类型

* 逻辑复制
  * 记录每次逻辑操作
  * 主从数据要求可以不一致
* 物理逻辑复制
  * 记录每次对于数据页的操作
  * 主从数据物理严格一致
  * 基于重做日志

## 二进制日志 -- 逻辑复制的基础

* 二进制日志（binary log）
  * 记录每次数据库的逻辑操作
* 二进制日志类型
  * Statement
  * Row         <--推荐使用Row格式的二进制日志，默认不打开通过参数打开 [mysqld] log_bin=1, binlog_format=row
  * Mixed

|          | Statement           | Row                       | Mixed            |
| -------- | ------------------- | ------------------------- | ---------------- |
| 说明     | 记录操作的SQL语句   | 记录操作的每一行数据      | 混合模式         |
| 优点     | 易于理解            | 数据一致性高、可flashback | 结合上述两种模式 |
| 缺点     | 不支持不确定SQL语句 | 每张表一定要有主键        | 之前版本bug较多  |
| 线上使用 | 不推荐              | 推荐                      | 不推荐           |

相关参数添加到 my.cnf中：

    log_bin=bin                                 # 打开binlog，日志文件都是以bin开头的，比如bin.000001
    binlog_format=row                           # 模式设置成row类型

二进制日志内容由各种类型的event组成，并且通过 (filename,pos) 表示位置信息

    (root@localhost) [bbb]> show tables;
    +---------------+
    | Tables_in_bbb |
    +---------------+
    | bba           |
    +---------------+
    1 row in set (0.00 sec)

    (root@localhost) [bbb]> desc bba;
    +-------+---------+------+-----+---------+-------+
    | Field | Type    | Null | Key | Default | Extra |
    +-------+---------+------+-----+---------+-------+
    | a     | int(11) | YES  |     | NULL    |       |
    +-------+---------+------+-----+---------+-------+
    1 row in set (0.00 sec)

    (root@localhost) [bbb]> show variables like 'binlog_format';
    +---------------+-------+
    | Variable_name | Value |
    +---------------+-------+
    | binlog_format | ROW   |
    +---------------+-------+
    1 row in set (0.00 sec)

    (root@localhost) [bbb]> flush binary logs;    # 新产生一个二进制日志文件
    Query OK, 0 rows affected (0.01 sec)

    (root@localhost) [bbb]> show master status;   # 查看当前二进制日志的情况
    +------------+----------+--------------+------------------+-------------------+
    | File       | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
    +------------+----------+--------------+------------------+-------------------+
    | bin.000013 |      154 |              |                  |                   |
    +------------+----------+--------------+------------------+-------------------+
    1 row in set (0.00 sec)

    (root@localhost) [bbb]> flush binary logs;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [bbb]> show master status;   # File从bin.000013变成bin.000014了
    +------------+----------+--------------+------------------+-------------------+
    | File       | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
    +------------+----------+--------------+------------------+-------------------+
    | bin.000014 |      154 |              |                  |                   |
    +------------+----------+--------------+------------------+-------------------+
    1 row in set (0.00 sec)

    (root@localhost) [bbb]> show binlog events in 'bin.000013';   # 查看二进制文件内容
    +------------+-----+----------------+-----------+-------------+---------------------------------------+
    | Log_name   | Pos | Event_type     | Server_id | End_log_pos | Info                                  |
    +------------+-----+----------------+-----------+-------------+---------------------------------------+
    | bin.000013 |   4 | Format_desc    |         1 |         123 | Server ver: 5.7.37-log, Binlog ver: 4 | <--开头都由这2个event组成
    | bin.000013 | 123 | Previous_gtids |         1 |         154 |                                       | <--开头都由这2个event组成
    | bin.000013 | 154 | Rotate         |         1 |         195 | bin.000014;pos=4                      | <--如果这个文件结束的话(flush操作)，就由Rotate事件组成，表示指向下一个event的起始的位置是什么
    +------------+-----+----------------+-----------+-------------+---------------------------------------+
    3 rows in set (0.00 sec)

    (root@localhost) [bbb]> show binlog events in 'bin.000014';
    +------------+-----+----------------+-----------+-------------+---------------------------------------+
    | Log_name   | Pos | Event_type     | Server_id | End_log_pos | Info                                  |
    +------------+-----+----------------+-----------+-------------+---------------------------------------+
    | bin.000014 |   4 | Format_desc    |         1 |         123 | Server ver: 5.7.37-log, Binlog ver: 4 |
    | bin.000014 | 123 | Previous_gtids |         1 |         154 |                                       |
    +------------+-----+----------------+-----------+-------------+---------------------------------------+
    2 rows in set (0.01 sec)

插入数据后的 binlog 情况：

    (root@localhost) [bbb]> begin;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [bbb]> insert into bba values (100);
    Query OK, 1 row affected (0.01 sec)

    (root@localhost) [bbb]> insert into bba values (200);
    Query OK, 1 row affected (0.00 sec)

    (root@localhost) [bbb]> commit;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [bbb]> show binlog events in 'bin.000014';
    +------------+-----+----------------+-----------+-------------+---------------------------------------+
    | Log_name   | Pos | Event_type     | Server_id | End_log_pos | Info                                  |
    +------------+-----+----------------+-----------+-------------+---------------------------------------+
    | bin.000014 |   4 | Format_desc    |         1 |         123 | Server ver: 5.7.37-log, Binlog ver: 4 |
    | bin.000014 | 123 | Previous_gtids |         1 |         154 |                                       |
    | bin.000014 | 154 | Anonymous_Gtid |         1 |         219 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'  |
    | bin.000014 | 219 | Query          |         1 |         290 | BEGIN                                 | <--表示执行了SQL语句
    | bin.000014 | 290 | Table_map      |         1 |         335 | table_id: 109 (bbb.bba)               | <--插入数据的表信息
    | bin.000014 | 335 | Write_rows     |         1 |         375 | table_id: 109 flags: STMT_END_F       | <--插入了一条数据
    | bin.000014 | 375 | Table_map      |         1 |         420 | table_id: 109 (bbb.bba)               |
    | bin.000014 | 420 | Write_rows     |         1 |         460 | table_id: 109 flags: STMT_END_F       |
    | bin.000014 | 460 | Xid            |         1 |         491 | COMMIT /* xid=24 */                   | <--表示提交了事务已经事务id是多少
    +------------+-----+----------------+-----------+-------------+---------------------------------------+
    9 rows in set (0.00 sec)

解析二进制日志的工具，其中的日志可以和 show binlog events 的输出内容对应起来看，并且附带了更多的信息：

    $ mysqlbinlog bin.000014

    # Write_rows在日志中类似下面的内容就是那一行的经过base64转化过的内容，方便传输
    BINLOG '
    y7K5YhMBAAAALQAAAE8BAAAAAG0AAAAAAAEAA2JiYgADYmJhAAEDAAFFO/Mq
    y7K5Yh4BAAAAKAAAAHcBAAAAAG0AAAAAAAEAAgAB//5kAAAAxUF4FA==
    '/*!*/;

想输出实际的内容可以附带一个 --base64-output 参数：

    $ mysqlbinlog --base64-output=decode-rows bin.000014
    # decode-rows就是解析每行的数据，但是可能还是看不到，需要再加上-v参数，更支持-vv参数可以显示更详细的列信息，通常来说-v就够看了

    # "###"表示的是注释
    $ mysqlbinlog --base64-output=decode-rows -v bin.000014
    ...
    BEGIN
    /*!*/;
    # at 290
    #220627 21:38:19 server id 1  end_log_pos 335 CRC32 0x2af33b45  Table_map: `bbb`.`bba` mapped to number 109
    # at 335
    #220627 21:38:19 server id 1  end_log_pos 375 CRC32 0x147841c5  Write_rows: table id 109 flags: STMT_END_F
    ### INSERT INTO `bbb`.`bba`   <--这些其实不是SQL语句，显示INSERT是因为这时一个Write_rows类型
    ### SET
    ###   @1=100                  <--@1表示列1，列1的记录是100，就算使用了replace into这里也依旧是INSERT INTO，它不管你SQL语句是什么只，管你这个一行记录是什么，Write_rows类型都会是INSERT INTO
    # at 375
    #220627 21:38:23 server id 1  end_log_pos 420 CRC32 0xb200f0ec  Table_map: `bbb`.`bba` mapped to number 109
    # at 420
    #220627 21:38:23 server id 1  end_log_pos 460 CRC32 0x55ca7d40  Write_rows: table id 109 flags: STMT_END_F
    ### INSERT INTO `bbb`.`bba`
    ### SET
    ###   @1=200
    ...

两个列产生的日志内容：

    (root@localhost) [bbb]> alter table bba add column b int default 100;
    Query OK, 0 rows affected (0.05 sec)
    Records: 0  Duplicates: 0  Warnings: 0

    (root@localhost) [bbb]> flush binary logs;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [bbb]> insert into bba value(400,400);
    Query OK, 1 row affected (0.00 sec)

    $ mysqlbinlog --base64-output=decode-rows -v bin.000015
    ...
    #220627 22:12:06 server id 1  end_log_pos 380 CRC32 0xb4af0977  Write_rows: table id 110 flags: STMT_END_F
    ### INSERT INTO `bbb`.`bba`
    ### SET
    ###   @1=400    <--列1和它的数据
    ###   @2=400    <--列2和它的数据
    # at 380
    ...

DELETE 产生的日志内容：

    (root@localhost) [bbb]> delete from bba where a < 400;
    Query OK, 2 rows affected (0.00 sec)

    $ mysqlbinlog --base64-output=decode-rows -v bin.000015
    ...
    #220627 22:14:06 server id 1  end_log_pos 547 CRC32 0xf12da754  Query   thread_id=2     exec_time=0     error_code=0    <--Query类型
    SET TIMESTAMP=1656339246/*!*/;
    BEGIN
    ...
    ### DELETE FROM `bbb`.`bba`
    ### WHERE
    ###   @1=100
    ###   @2=100
    ### DELETE FROM `bbb`.`bba`
    ### WHERE
    ###   @1=200
    ###   @2=100
    # at 646
    ...

INSERT 和 DELETE 都记录整行的记录，只不过一个标记为插入一个标记为删除，但是 UPDATE 产生的日志内容有所不同：

    (root@localhost) [bbb]> update bba set b = 500 where b =4;
    Query OK, 0 rows affected (0.00 sec)
    Rows matched: 0  Changed: 0  Warnings: 0

    $ mysqlbinlog --base64-output=decode-rows -v bin.000015
    ...
    BEGIN
    /*!*/;
    # at 813
    #220627 22:20:20 server id 1  end_log_pos 859 CRC32 0x6cafb4cf  Table_map: `bbb`.`bba` mapped to number 110
    # at 859
    #220627 22:20:20 server id 1  end_log_pos 913 CRC32 0xfe8e56cc  Update_rows: table id 110 flags: STMT_END_F
    ### UPDATE `bbb`.`bba`  <--UPDATE记录的前项和后项，所以UPDATE日志相对来说会大一点
    ### WHERE               <--UPDATE记录的前项
    ###   @1=400
    ###   @2=400
    ### SET                 <--UPDATE记录的后项
    ###   @1=400
    ###   @2=500
    # at 913
    ...

二进制日志的优点是记录的是每一行的操作，所以可以确保主从的日志是严格一致的，缺点是大，UPDATE全表和DELETE全表的时候日志文件会比较大性能也会非常差，所以在 MySQL 中不建议 UPDATE 全表这类操作

row 格式下可以通过打开另外一个参数记录执行的原样的 SQL 语句：

    (root@localhost) [bbb]> show variables like '%binlog_rows_query_log_events%';
    +------------------------------+-------+
    | Variable_name                | Value |
    +------------------------------+-------+
    | binlog_rows_query_log_events | OFF   |
    +------------------------------+-------+
    1 row in set (0.00 sec)

    (root@localhost) [bbb]> set binlog_rows_query_log_events=1;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [bbb]> begin;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [bbb]> insert into bba values(100,100);
    Query OK, 1 row affected (0.00 sec)

    (root@localhost) [bbb]> insert into bba values(200,200);
    Query OK, 1 row affected (0.00 sec)

    (root@localhost) [bbb]> update bba set b = 400;
    Query OK, 3 rows affected (0.00 sec)
    Rows matched: 3  Changed: 3  Warnings: 0

    (root@localhost) [bbb]> commit;
    Query OK, 0 rows affected (0.01 sec)

这个功能是比较有用的，可以考虑加入到 my.cnf 中在线上打开

还有一些有用的参数：

* --start-datetime=name   # 从哪个时间点开始解析
* --start-position=#      # 从哪个二进制的偏移量开始解析
* --stop-position=#       # 从哪个二进制的偏移量停止解析，和上面的参数配合就是一个范围

如果在一条 SQL 语句插入了三行记录，在 binlog 会记录 3 次操作而不是 1 次，比如：

    mysql> insert into a values(1,2),(2,3),(3,4);

    # 这条SQL语句会对应3个write_rows
    write_rows    (1,2)
    write_rows    (2,3)
    write_rows    (3,4)

    mysql> delete from a;   # 回放的时候先根据主键进行回放，如果没有主键就找一个索引，如果一个索引都没有就scan全表扫
                            # 这时候如果有10万条记录一删，就会扫描10万次
    delete_rows    (1,2)
    delete_rows    (2,3)
    delete_rows    (3,4)

为了解决这个问题 MySQL 推出了一个扫描算法的参数控制回放速度：

    (root@localhost) [test]> show variables like '%slave_rows_search_algorithms%';
    +------------------------------+-----------------------+
    | Variable_name                | Value                 |
    +------------------------------+-----------------------+
    | slave_rows_search_algorithms | TABLE_SCAN,INDEX_SCAN |
    +------------------------------+-----------------------+
    1 row in set (0.00 sec)

## 二进制日志增量备份

增量备份其实就是备份二进制日志就好了，知道了每次操作都会产生二进制日志，一旦你由一个全备了接着去备份增量对应的二进制日志就可以了

但是主从这种架构下每台服务器都有一致的数据，那也就没有必要备份了，也并不需要强调增量备份，除非你的 MySQL 是单实例的

## binlog 文件的恢复

单个二进制文件的恢复：

    $ mysqlbinlog bin.000014 | mysql -uroot -p -f   # -f 可以忽略错误
    # 恢复的时候可以在mysql中通过showprocess看到恢复的线程

多个二进制文件的恢复：

    $ mysqlbinlog bin.[0-9]* | mysql -uroot -p

更多用法可以参考官方文档 https://dev.mysql.com/doc/refman/5.7/en/mysqlbinlog.html

binlog 的恢复非常简单，日常通常备份好 binlog 文件，用上面的命令恢复即可

## flashback 

注意 MySQL5.7 8.0 官方的 mysqlbinlog 命令还不支持

https://github.com/danfengcao/binlog2sql/blob/master/example/mysql-flashback-priciple-and-practice.md

https://juejin.cn/post/6950641397695250439

假设现在有一张如下的表，并且误操作不小心把表里的数据全删了：

    (root@localhost) [test]> select * from aa;
    +------+------+------+
    | a    | b    | c    |
    +------+------+------+
    |    1 |    2 |    3 |
    |    2 |    4 |    6 |
    |    3 |    5 |    7 |
    |    4 |    6 |    8 |
    +------+------+------+
    4 rows in set (0.00 sec)

    (root@localhost) [test]> delete from aa;
    Query OK, 4 rows affected (0.00 sec)

    (root@localhost) [test]> select * from aa;
    Empty set (0.00 sec)

避免这种误操作的方法是，在 DML 语句前面先使用 begin，但即使打了 begin 也会有误操作的时候，比如看错了然后进行了提交

这个时候可以通过 flashback 进行恢复，首先查看一下 binlog 日志状态：

    (root@localhost) [test]> show master status;
    +------------+----------+--------------+------------------+-------------------+
    | File       | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
    +------------+----------+--------------+------------------+-------------------+
    | bin.000016 |     2372 |              |                  |                   |
    +------------+----------+--------------+------------------+-------------------+
    1 row in set (0.00 sec)

    $ cd /mdata/mysql_test_data

    $ mysqlbinlog bin.000016 -vv
    ...
    # at 2135       <--查看到删除数据以前的pos位置
    #220628 12:58:27 server id 1  end_log_pos 2207 CRC32 0x2009e2fc         Query   thread_id=4     exec_time=0     error_code=0
    SET TIMESTAMP=1656392307/*!*/;
    BEGIN
    /*!*/;
    # at 2207
    #220628 12:58:27 server id 1  end_log_pos 2254 CRC32 0x50d870a0         Table_map: `test`.`aa` mapped to number 110
    # at 2254
    #220628 12:58:27 server id 1  end_log_pos 2341 CRC32 0x6eb6142d         Delete_rows: table id 110 flags: STMT_END_F

    BINLOG '
    c4q6YhMBAAAALwAAAM4IAAAAAG4AAAAAAAEABHRlc3QAAmFhAAMDAwMAB6Bw2FA=
    c4q6YiABAAAAVwAAACUJAAAAAG4AAAAAAAEAAgAD//gBAAAAAgAAAAMAAAD4AgAAAAQAAAAGAAAA
    +AMAAAAFAAAABwAAAPgEAAAABgAAAAgAAAAtFLZu
    '/*!*/;
    ### DELETE FROM `test`.`aa`
    ### WHERE
    ###   @1=1 /* INT meta=0 nullable=1 is_null=0 */
    ###   @2=2 /* INT meta=0 nullable=1 is_null=0 */
    ###   @3=3 /* INT meta=0 nullable=1 is_null=0 */
    ### DELETE FROM `test`.`aa`
    ### WHERE
    ###   @1=2 /* INT meta=0 nullable=1 is_null=0 */
    ###   @2=4 /* INT meta=0 nullable=1 is_null=0 */
    ###   @3=6 /* INT meta=0 nullable=1 is_null=0 */
    ### DELETE FROM `test`.`aa`
    ### WHERE
    ###   @1=3 /* INT meta=0 nullable=1 is_null=0 */
    ###   @2=5 /* INT meta=0 nullable=1 is_null=0 */
    ###   @3=7 /* INT meta=0 nullable=1 is_null=0 */
    ### DELETE FROM `test`.`aa`
    ### WHERE
    ###   @1=4 /* INT meta=0 nullable=1 is_null=0 */
    ###   @2=6 /* INT meta=0 nullable=1 is_null=0 */
    ###   @3=8 /* INT meta=0 nullable=1 is_null=0 */
    # at 2341
    ...

根据找到的位置恢复数据：

    $ mysqlbinlog -vv bin.000016 --start-position=2135 -B

会发现输出的内容从 DELETE 变成了 INSERT，这就是所谓的 flashback

最后进行恢复：

    $ mysqlbinlog -vv bin.000016 --start-position=2135 -B | mysql -uroot -p

## 搭建复制

搭建复制最重要的基础是二进制日志

MySQL 的 commit 分成三个阶段，一个是 prepare 阶段，一个是写二进制日志的阶段，一个是最后的一次阶段，第一次和第三次是在 InnoDB 层，第二次是写二进制日志，并且所有这三个阶段都是 group commit 组提交的，所以性能并不会非常差

当 commit 写入到二进制日志文件之后，如果在 MySQL 中搭建了复制就会有一个所谓的 dump 线程，dump 线程会将你的日志以 event 为最小单位发送到远程的服务器，远程服务器的 I/O 线程会负责接收二进制日志 event，还有个线程叫 SQL 线程用来回放，MySQL 5.6 开始可以有多个 SQL 线程可以用来进行并行回放，相对来说可能会比较快

### 创建复制关系

1. 主机上备份数据
2. 从机上恢复数据
3. 主机上授权一个复制用户权限
4. 从机 CHANGE MASTER TO

CHANGE MASTER TO 的作用是，现在有一个备份对应的二进制日志的文件名是 (filename,pos)，用 mysqldump 进行备份的时候会加上一个 master-data 参数，这个参数加着就会包含 (filename,pos)，二进制日志对应的文件名和偏移量

所谓的 CHANGE MASTER 就是迁移到这个位置，这个位置之后的数据都通过 binlog 来进行同步，所以它是一个最终一致性的模型，主机不断的将日志发送到从机来进行恢复

### 环境准备

my.cnf 中配置多实例：

    [mysqld1]
    server-id = 11                 # 主从之间的server-id一定要不一样
    innodb_buffer_pool_size = 32M
    port = 3307
    datadir = /mdata/data1
    socket = /tmp/mysql.sock1

    [mysqld2]
    server-id = 12                 # 主从之间的server-id一定要不一样
    innodb_buffer_pool_size = 32M
    port = 3308
    datadir = /mdata/data2
    socket = /tmp/mysql.sock2

启动 mysqld1 和 mysqld2：

    $ mysqld_multi start 1    # 对应3307端口的MySQL

    $ mysqld_multi start 2    # 对应3308端口的MySQL

    $ mysqld_multi report
    Reporting MySQL servers
    MySQL server from group: mysqld1 is running
    MySQL server from group: mysqld2 is running
    MySQL server from group: mysqld80 is not running

封装启动的快捷方式：

    $ vim m3307
    1 mysql -h 127.0.0.1 -P 3307

    $ vim m3308
    1 mysql -h 127.0.0.1 -P 3308

    $ chmod u+x m3307 m3308

    $ mv m3307 m3308 /usr/local/bin

并假设 m3307 为主机，m3308 为从机，主从之间的server-id一定要不一样

### 主机上备份数据

首先对 3307 进行备份，用 mysqldump 和 xtrabackup 都可以，xtrabackup 自动保存二进制备份日志的位置，这里用 mysqldump 演示

    $ mysqldump --single-transaction --master-data=1 -A -S /tmp/mysql.sock1 > fullbackup.sql

### 从机上恢复数据

    $ m3308 < fullbackup.sql
    
fullbackup.sql 文件中有一行 CHANGE MASTER TO MASTER_LOG_FILE='bin.000002', MASTER_LOG_POS=154; 表示这个备份对应的位置点

### 创建一个复制用户

在 m3307 主机上授权一个复制用户权限：

    # 创建用户
    (root@127.0.0.1:3307) [(none)]> create user rpl@'%' identified by 'Abc123__';
    Query OK, 0 rows affected (0.00 sec)

    # 授予复制权限
    (root@127.0.0.1:3307) [(none)]> grant replication slave on *.* to rpl@'%';
    Query OK, 0 rows affected (0.00 sec)

### 找一下备份文件里对应二进制日志的位置

    $ head -n 24 fullbackup.sql
    -- MySQL dump 10.13  Distrib 5.7.37, for linux-glibc2.12 (x86_64)
    --
    -- Host: localhost    Database:
    -- ------------------------------------------------------
    -- Server version       5.7.37-log

    /*!40101 SET @OLD_CHARACTER_SET_CLIENT=@@CHARACTER_SET_CLIENT */;
    /*!40101 SET @OLD_CHARACTER_SET_RESULTS=@@CHARACTER_SET_RESULTS */;
    /*!40101 SET @OLD_COLLATION_CONNECTION=@@COLLATION_CONNECTION */;
    /*!40101 SET NAMES utf8 */;
    /*!40103 SET @OLD_TIME_ZONE=@@TIME_ZONE */;
    /*!40103 SET TIME_ZONE='+00:00' */;
    /*!40014 SET @OLD_UNIQUE_CHECKS=@@UNIQUE_CHECKS, UNIQUE_CHECKS=0 */;
    /*!40014 SET @OLD_FOREIGN_KEY_CHECKS=@@FOREIGN_KEY_CHECKS, FOREIGN_KEY_CHECKS=0 */;
    /*!40101 SET @OLD_SQL_MODE=@@SQL_MODE, SQL_MODE='NO_AUTO_VALUE_ON_ZERO' */;
    /*!40111 SET @OLD_SQL_NOTES=@@SQL_NOTES, SQL_NOTES=0 */;

    --
    -- Position to start replication or point-in-time recovery from
    --

    CHANGE MASTER TO MASTER_LOG_FILE='bin.000002', MASTER_LOG_POS=154; <--

### 从机上进行 CHANGE MASTER 操作

把这个位置告诉从机，并在从机上执行 CHANGE MASTER 操作

    (root@127.0.0.1:3308) [(none)]> CHANGE MASTER TO MASTER_LOG_FILE='bin.000001', MASTER_LOG_POS=154,MASTER_HOST='127.0.0.1',MASTER_PORT=3307,MASTER_USER='rpl',MASTER_PASSWORD='Abc123__';
    Query OK, 0 rows affected, 2 warnings (0.01 sec)

    # MASTER_LOG_FILE   从主库的哪一个日志文件开始
    # MASTER_LOG_POS    从主库的哪一个日志点开始
    # MASTER_HOST       需要同步的主数据库 IP 
    # MASTER_PORT       需要同步的主数据库端口
    # MASTER_USER       主库上创建的那个用于同步的用户名
    # MASTER_PASSWORD   主库上创建的那个用于同步的用户的密码

查看同步状态：

    (root@127.0.0.1:3308) [(none)]> show slave status\G
    *************************** 1. row ***************************
                  Slave_IO_State:
                      Master_Host: 127.0.0.1
                      Master_User: rpl
                      Master_Port: 3307
                    Connect_Retry: 60
                  Master_Log_File: bin.000001
              Read_Master_Log_Pos: 154
                  Relay_Log_File: 9fc028750734-relay-bin.000001
                    Relay_Log_Pos: 4
            Relay_Master_Log_File: bin.000001
                Slave_IO_Running: No    <--No表示还没开始同步
                Slave_SQL_Running: No   <--No表示还没开始同步
    ...

开始同步：

    (root@127.0.0.1:3308) [(none)]> start slave;
    Query OK, 0 rows affected (0.00 sec)

    (root@127.0.0.1:3308) [(none)]> show slave status\G
    *************************** 1. row ***************************
                  Slave_IO_State: Connecting to master
                      Master_Host: 127.0.0.1
                      Master_User: rpl
                      Master_Port: 3307
                    Connect_Retry: 60
                  Master_Log_File: bin.000001
              Read_Master_Log_Pos: 154
                  Relay_Log_File: 9fc028750734-relay-bin.000001
                    Relay_Log_Pos: 4
            Relay_Master_Log_File: bin.000001
                Slave_IO_Running: Connecting  <--开始同步
                Slave_SQL_Running: Yes        <--开始同步

测试一下，在主库上创建一个库：

    (root@127.0.0.1:3307) [(none)]> create database a;
    Query OK, 1 row affected (0.00 sec)

然后在从库上可以看到 a 这个库被复制过来了：

    (root@127.0.0.1:3308) [(none)]> show databases;
    +--------------------+
    | Database           |
    +--------------------+
    | information_schema |
    | a                  |
    | mysql              |
    | performance_schema |
    | sys                |
    +--------------------+
    5 rows in set (0.00 sec)

主机查看连过来的从机有哪些：

    (root@127.0.0.1:3307) [(none)]> show slave hosts;
    +-----------+------+------+-----------+--------------------------------------+
    | Server_id | Host | Port | Master_id | Slave_UUID                           |
    +-----------+------+------+-----------+--------------------------------------+
    |        12 |      | 3308 |        11 | 50b88b6d-e635-11ec-8832-0242ac110002 |
    +-----------+------+------+-----------+--------------------------------------+
    1 row in set (0.00 sec)

从机最好是只读的，不然你在从机上的操作和主机上的操作重复了的话，同步就会报错

比如现在从机上创建一个 b 库，然后在主机上创建一个 b 库，同步就会报错：

    (root@127.0.0.1:3308) [(none)]> create database b;
    Query OK, 1 row affected (0.01 sec)

    (root@127.0.0.1:3307) [(none)]> create database b;
    Query OK, 1 row affected (0.01 sec)

    (root@127.0.0.1:3308) [(none)]> show slave status\G
    *************************** 1. row ***************************
                  Slave_IO_State: Waiting for master to send event
                      Master_Host: 127.0.0.1
                      Master_User: rpl
                      Master_Port: 3307
                    Connect_Retry: 60
                  Master_Log_File: bin.000002
              Read_Master_Log_Pos: 1187
                  Relay_Log_File: 9fc028750734-relay-bin.000003
                    Relay_Log_Pos: 1238
            Relay_Master_Log_File: bin.000002
                Slave_IO_Running: Yes
                Slave_SQL_Running: No   <--这里出现了No
                  Replicate_Do_DB:
              Replicate_Ignore_DB:
              Replicate_Do_Table:
          Replicate_Ignore_Table:
          Replicate_Wild_Do_Table:
      Replicate_Wild_Ignore_Table:
                      Last_Errno: 1007
                      Last_Error: Error 'Can't create database 'b'; database exists' on query. Default database: 'b'. Query: 'create database b' <--错误信息

这时候在主机上创建新的 c 库也不会同步过来了，可以通过参数配置跳过当前执行的 event：

    (root@127.0.0.1:3308) [(none)]> set global sql_slave_skip_counter=1;  # 1表示跳过几个，100就是跳过100个
    Query OK, 0 rows affected (0.01 sec)

    (root@127.0.0.1:3308) [(none)]> start slave;
    Query OK, 0 rows affected (0.00 sec)

所以复制的时候从库最好不要有写入操作，一旦有写入操作可能就会导致主从数据不一致，所以强烈建议在从机上面执行一个命令将从库设置成只读：

    (root@127.0.0.1:3308) [(none)]> set global read_only = 1;
    Query OK, 0 rows affected (0.00 sec)

    # 即使root用户也无法进行写入操作，MySQL5.7以上支持
    (root@127.0.0.1:3308) [(none)]> set global super_read_only = 1;   
    Query OK, 0 rows affected (0.00 sec)

### 停止同步

从机上执行 stop slave 命令即可：

    (root@127.0.0.1:3308) [(none)]> stop slave;

### 最快搭建从机的姿势

可以使用 xtrabackup 做备份，然后重建恢复

https://cloud.tencent.com/developer/article/1508794