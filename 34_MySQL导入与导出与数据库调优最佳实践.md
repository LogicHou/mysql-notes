# MySQL导入与导出

    (root@localhost) [(none)]> show variables like 'secure%';
    +------------------+-------+
    | Variable_name    | Value |
    +------------------+-------+
    | secure_auth      | ON    |
    | secure_file_priv | NULL  |
    +------------------+-------+
    2 rows in set (0.00 sec)

导入导出的这个 secure_file_priv 参数有 3 个值

* 一个叫 NULL 值表示不允许导入导出
* set global secure_file_priv='' 设置为快空值的话就是允许导入导出，而且导入导出可以是任何路径
* set global secure_file_priv='/tmp/' 如果设置具体路径的就只能在设置的位置进行导入导出

my.cnf 里可以设置：

    [mysqld]
    ...
    binlog_format = row
    secure_file_priv=/tmp   # new

## 导出

    (root@localhost) [tpcc1000]> select * into outfile '/tmp/tpcc1000.dat' from orders where o_id < 3000;
    Query OK, 59980 rows affected (0.07 sec)

    $ cd /tmp
    
    $ ls -lh tpcc1000.dat
    -rw-rw-rw- 1 mysql mysql 2.3M Jun 27 12:43 tpcc1000.dat

    $ head tpcc1000.dat   # 导出的是每个列的数据，每列的数据通过 tab 来进行分隔
    1       1       1       1743    2022-06-07 11:13:15     9       10      1
    2       1       1       45      2022-06-07 11:13:15     9       8       1
    3       1       1       2401    2022-06-07 11:13:15     9       9       1
    4       1       1       1381    2022-06-07 11:13:15     7       6       1
    5       1       1       145     2022-06-07 11:13:15     3       14      1
    6       1       1       142     2022-06-07 11:13:15     6       12      1
    7       1       1       646     2022-06-07 11:13:15     1       5       1
    8       1       1       184     2022-06-07 11:13:15     8       9       1
    9       1       1       567     2022-06-07 11:13:15     2       6       1
    10      1       1       1779    2022-06-07 11:13:15     8       15      1

导出的时候也可以加参数设置分隔符

## 导入

    (root@localhost) [tpcc1000]> load data infile '/tmp/tpcc1000.dat' into table orders2;
    Query OK, 59980 rows affected (0.54 sec)
    Records: 59980  Deleted: 0  Skipped: 0  Warnings: 0

现在导入的数据表的列是一一对应的，想不对应的话，也是可以的，比如：

    (root@localhost) [tpcc1000]> load data infile '/tmp/xxx.dat' into table b fields terminated by ',' (a,b) set c = a+b;

load data 比 mysqldump 要快得比较多，因为没有解析的开销

load data 的缺点是它是一个文本文件，文本文件其实是有限制的，字符本身如果有些麻烦的字符，比如分隔符和导出分隔符一样就比较麻烦了，比如像导博客这样的数据类型就很麻烦

使用 hex() 尝试解决这个问题：

    # 导出
    (root@localhost) [tpcc1000]> select c_id,concat('0x',hex(c_street_1)),c_balance from customer limit 1;
    +------+--------------------------------------------+-----------+
    | c_id | concat('0x',hex(c_street_1))               | c_balance |
    +------+--------------------------------------------+-----------+
    |    1 | 0x4755684E4939636B324F72544151543943633072 |   4101.29 |
    +------+--------------------------------------------+-----------+
    1 row in set (0.00 sec)

    (root@localhost) [tpcc1000]> select c_id,concat('0x',hex(c_street_1)),c_balance into outfile '/tmp/tpcc1000.customer1.dat' from customer limit 1;
    Query OK, 1 row affected (0.00 sec)

    $ head /tmp/tpcc1000.customer1.dat
    1       0x4755684E4939636B324F72544151543943633072      4101.29

    # 导入

有点问题，先挂个TODO

## 独立表空间的导入导出

导入导出更多的使用场景是在一个异构数据库的情况下，比如 Oracle 导入 MySQL 或者 mongodb 导入到 MySQL，如果一张表很大有几 G 或者几十 G 的数据导入就会很慢

MySQL5.6 开始推出了一个独立表空间的导入和导出功能，比较类似 xtrabackup 的备份机制，它备份的是表空间

如果有一张表想直接导到另外一个 MySQL 数据库的实例上面，可以用独立表空间的导入导出，在文档里面叫做透明表空间传输，这样的好处就是不需要把表的数据导入导出了，缺点是数据量可能会比较大

两张表的表结构必须是一样的：

    (root@localhost) [tpcc1000]> create table orders3 like orders2;
    Query OK, 0 rows affected (0.01 sec)

在源服务器中先删除新建表的表空间文件：

    (root@localhost) [tpcc1000]> alter table orders3 discard tablespace;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [tpcc1000]> select * from orders3;
    ERROR 1814 (HY000): Tablespace has been discarded for table 'orders3'

在源服务器中将源表锁成只读：

    (root@localhost) [tpcc1000]> flush tables orders2 for export;
    Query OK, 0 rows affected (0.00 sec)
    # show process 中可以查看这把锁

然后在目的服务器中拷贝源数据的表空间文件：

    $ cd /mdata/mysql_test_data/tpcc1000

    $ cp orders2.ibd orders3.ibd
    # 没有导入导出，所以这个速度是非常快的

    $ chown -R mysql:mysql orders3.ibd
    # 记得修改权限

在源服务器中解锁表：

    (root@localhost) [tpcc1000]> unlock tables;
    Query OK, 0 rows affected (0.00 sec)

在目的服务器中导入表空间：

    (root@localhost) [tpcc1000]> alter table orders3 import tablespace;
    Query OK, 0 rows affected, 1 warning (0.05 sec)

最后就可以在目的服务器查看到数据了：

    (root@localhost) [tpcc1000]> select * from orders3 limit 1;
    +------+--------+--------+--------+---------------------+--------------+----------+-------------+
    | o_id | o_d_id | o_w_id | o_c_id | o_entry_d           | o_carrier_id | o_ol_cnt | o_all_local |
    +------+--------+--------+--------+---------------------+--------------+----------+-------------+
    |    1 |      1 |      1 |   1743 | 2022-06-07 11:13:15 |            9 |       10 |           1 |
    +------+--------+--------+--------+---------------------+--------------+----------+-------------+

## MySQL5.7 基于分区表的透明表空间传输 用的比较多

https://dev.mysql.com/doc/refman/5.7/en/alter-table-partition-operations.html

将分区表拷贝到另外一个实例

用法和上面的类似，就是 alter table 这块的语法需要修改下：

    mysql> ALTER TABLE t1 DISCARD PARTITION p2, p3 TABLESPACE;

import 的时候：

    mysql> ALTER TABLE t1 IMPORT PARTITION p2, p3 TABLESPACE;

也就是说可以拷贝远程的某几个分区然后再加载过来，这个或许是有一些用的

比如快递或者订单业务 orders 表保留三个月的数据，还有一张 orders_archive 保留的是半年的数据，当这些数据都变得非常老的时候，就可以传输到另外一个地方做归档，这些老的归档数据可能一时半会也用不上但是一定是需要它存放着的

这时候可能有另外一台配置可能也不是特别高的服务器就是存个数据，上面也有一个存放所有历史记录的 orders_archive 表，这张表会非常非常大，这时候有透明表传输的话就会相对来说比较简单一点一些了

通过 scp 到源数据服务器拿数据文件，然后通过 IMPORT PARTITION p201701 TABLESPACE 某个比如 2020 年 1 月份的数据，历史做归档的时候可以这么来搞

# 数据库调优最佳实践

## 数据库配置

* InnoDB的缓冲池用来管理所有数据库对象
* 写文件操作通过O_DIRECT选项来避免两次缓存
* InnoDB缓冲池越大性能越好
  * 通常设置成内存的60%~80%

调优参数：

    innodb_buffer_pool_size=100G      # 内存的60%~80%
    innodb_buffer_pool_instances=16   # CPU数量的一半或者CPU数量
    innodb_page_size=4096             # 千万不要设置成4K，4K性能是非常差的，可以设置成16K
    innodb_flush_method=O_DIRECT      # 不要要两次缓存效果

    # MySQL 5.7 online resize buffer pool   # 可以在线调整buffer_pool
    mysql> set global innodb_disable_resize_buffer_pool_debug=off;

    mysql> set global innodb_buffer_pool_size=256*1024*1024

* FUZZY CHECKPOINT
  * 刷新部分脏页，对系统影响较小
  * 5.6：独立的刷新线程
  * 5.7：并行刷新线程
  * innodb_io_capacity
* SHARP CHECKPOINT
  * 刷新全部脏页，系统hang住
  * innodb_fast_shutdown
* Neighbor Page flush
  * innodb_flush_neighbors

调优参数：

    innodb_io_capacity=1000/4000/8000   # 读写性能的一半
    innodb_page_cleaners=1/4            # CPU的数量或者一半
    innodb_fast_shutdown=0/1            
    innodb_flush_neighbors=0/1/2        # SSD直接设置成0就行了

* 重做日志
  * 记录**页**操作的日志
  * 与二进制日志完全不同
  * 循环覆盖写
  * 默认没有类似PG或者Oracle的归档
* 重做日志大小限制
  * before 5.6：max 4G
  * start from 5.6：max 512G

调优参数：

    innodb_log_file_size=1900M/4G       # 至少4G，性能好设大点和刷新是有关系的
    innodb_log_buffer_size=8M           # 
    innodb_log_file_in_group=2/3
    innodb_log_group_home_dir=/redolog/

* undo段
  * 实现回滚
  * 实现MVCC功能
* undo段数量
  * before MySQL 5.5：1024
  * start from MySQL 5.5：128*1024
* undo回收
  * purge
* 不需要怎么调整

调优参数：

    innodb_undo_directory=/undolog/
    innodb_undo_logs=128
    innodb_undo_tablespace=3

    innodb_undo_log_truncate=1
    innodb_undo_undo_log_size=1G
    innodb_purge_rseg_truncate_frequency=128

    innodb_purge_batch_size=300
    innodb_purge_threads=4/8

* 线程池
  * 保障高并发下的性能稳定
  * MariaDB线程池没有优先级队列
  * 推荐MySQL/InnoDB/Percona线程池
  * **推荐默认开启线程池**
  * **MySQL社区版支持线程池**

调优参数：

    thread_handling=pool-of-threads 
    thread_pool_size=32         #CPU 
    thread_pool_oversubscribe=3
    extra_port=3333             #额外的端口


## SQL优化

不要急着想去升级硬件，先看看自己的慢查询日志都处理完没，再去考虑升级硬件的问题

* 子查询
  * before 5.6：
    * lazy：rewrite to exists
    * poor performance exists都是相关子查询，性能差
  * from 5.6：（MariaDB 5.3）
    * semi-join

## 软硬件设置

* 内存
* 网卡 
* RAID卡
* SSD 

### NUMA

* NUMA
  * Non-Uniform Memory Access
  * 非一致性存储访问结构

NUMA的内存分配策略有四种：

1. default：总是在本地节点分配
2. bind：强制分配到指定节点上
3. interleave：在所有节点或者指定的节点上交织分配
4. preferred：在指定节点上分配，失败则在其他节点上分配

用 MySQL 的时候 NUMA 需要设置成 interleave，MySQL 5.7 后期的版本才支持这个参数

    (root@localhost) [(none)]> show variables like '%innodb%numa%';
    +------------------------+-------+
    | Variable_name          | Value |
    +------------------------+-------+
    | innodb_numa_interleave | OFF   |
    +------------------------+-------+
    1 row in set (0.01 sec)

在 my.cnf 中设置：

    [mysqld]
    ...
    secure_file_priv=/tmp
    innodb_numa_interleave=1    # new

### 网卡软中断

MySQL QPS 非常高的时候可能会遇到这个问题，网卡可能会成为瓶颈

软中断的意思的是 TOP 命令查看 CPU 信息的时候，发现某一个 CPU 的 %soft 列非常高，而且就只有这一个核的 CPU 使用率非常高其他的会非常低，这就叫软中断

要解决这个问题就要一定要打开网卡多队列的功能，现在的服务器不是太老的基本都支持网卡多队列

* 启动网卡多队列
  * set_irq_affinity.sh 
    * ./set_irq_affinity.sh 0 eth0
    * ./set_irq_affinity.sh 8 eth1
  * service irqbalance stop   # 关闭操作系统本身提供的中断平衡服务

### RAID卡

写缓存打开，打开和不打开性能会差很多

### SSD与数据库优化

* 磁盘调度算法设置为：deadline或者noop
* InnoDB存储引擎参数设置
  * innodb_flush_neighbors=0
  * innodb_log_file_size=4G   # 越大越好

### 文件系统与操作系统

* 文件系统 
  * 推荐xfs/ext4
  * noatime
  * nobarrier
* 操作系统
  * 推荐Linux操作系统
  * 关闭swap
  * 磁盘调度算法
    * mount -o noatime,nobarrier /dev/sdb1 /data