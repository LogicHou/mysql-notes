# 透明页压缩

MySQL 默认的压缩算法是使用 zlip 的

    (root@localhost) [test]> create table a (a int primary key) compression='zlip';
    Query OK, 0 rows affected (0.01 sec)

可以改成使用透明页压缩 lz4 的

    (root@localhost) [test]> drop table a;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> create table a (a int primary key) compression='lz4';
    Query OK, 0 rows affected (0.01 sec)

    (root@localhost) [test]> show create table a\G
    *************************** 1. row ***************************
          Table: a
    Create Table: CREATE TABLE `a` (
      `a` int(11) NOT NULL,
      PRIMARY KEY (`a`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMPRESSION='lz4'
    1 row in set (0.00 sec)

lz4 这个压缩算法更快，zlip 则压缩比更高，但是通常来说选择 lz4，因为他的压缩速度够快，虽然他的压缩比率比 zlip 小，但通常来说已经足够使用了

参考文档：https://dev.mysql.com/blog-archive/innodb-transparent-page-compression/

## Hole Punch Size 空洞特性

假设一个文件只使用了 12K 的大小，那么后面会全部填入 0 直到文件大小到达了 16K 大小

在 Linux 文件系统中，通过 fopen(f, O_DIRECT|O_PUNCHHOLE)，fwrite(f, page) 写入

## TPC 透明页压缩过程

在一个页要进行写入的时候，先对这 16K 的页进行压缩，压缩完之后前面的部分是压缩后的数据，后面的部分则全部填入很多个 0，填完 0 之后调用 fwrite 写入，如果这个表空间一开始支持 Punch Hole 的话，这时候在磁盘上就进行了压缩

压缩的大小是根据文件系统的块的大小对齐的，默认的大小是 4K 的，16K 的页之能压缩成 4K 的倍数，如果压缩后是 6K 的依然会占用 8K 的空间

透明压缩有一个好处是在 buffer pool 中只会有一个 16K 的页，既不区分压缩页和解压后的页，只有第一次读取 (sapce, page_num) 的时候会读这个页，读完之后解压后也是 16K 的大小，在 buffer pool 中只占用了一个空间，像之前的算法既有压缩的页也有非压缩的页，写入和更新很麻烦既要更新压缩的页又要更新非压缩的页

透明页压缩是一个非常不错的特性，甚至是应该默认开启的特性

## 扩展性参数

以前 MySQL 假设这台服务器是 100 核的 然后可能 200 个逻辑核，去跑测试的话 200 个核心应该都是跑满的，跑不满的话并发就是有瓶颈的，不过现在的 MySQL 已经没有这个问题了

buffer pool 对这种线性增长的扩展性是非常重要的，因为 buffer pool 是一个非常非常热的热点，所有热点的页都在其中

重要的参数 innodb_buffer_pool_instances，建议设置为 cpu 的逻辑核数量：

    (root@localhost) [(none)]> show variables like 'innodb_buffer_pool%';
    +-------------------------------------+----------------+
    | Variable_name                       | Value          |
    +-------------------------------------+----------------+
    | ...                                 | ...            |
    | innodb_buffer_pool_instances        | 8              |
    | ...                                 | ...            |
    +-------------------------------------+----------------+
    10 rows in set (0.00 sec)

也可以配置到 my.cnf 配置文件里：

    # innodb
    30 innodb_buffer_pool_size = 1G
    31 innodb_buffer_pool_instances = 8 <--
    32 innodb_log_file_size = 128M

通过这样的设置，这里就可以产生 8 个 128MB(1G/8) 的 PB，这样 latch 就从 1 个变成了 8 个，这是做并发最简单的设计，就是把 latch 这把锁去做分片

这个参数非常重要，设于不设性能差距可能有 30% 之多，可以检查下线上服务器是否设成了 CPU 的逻辑核数量

# buffer pool 的管理

* Buffer pool
  * Free List
  * LRU List
    * LRU
    * unzip_LRU
  * Flush List
    * 根据oldest_lsn进行排序

buffer pool 的管理是根据页的单位来进行管理的，是一个页一个页来进行管理的

不是 Free List + LRU List + Flush List 等于所有 buffer pool 的大小，真正的大小是 Free List + LRU List，LRU List 包含了 Flush List

假设现在一开始有 8 个 buffer pool 的页，一开始数据库刚刚启动这时所有的这 8 个页都是在 Free List 里面的

如果现在磁盘上某一个页需要被读到 buffer pool 里面，就会问 Free List 去要一个空闲的页，然后将磁盘上的内容读到缓冲池里，这时候从 Free List 移动到 LRU List 需要上面说的 latch 做并发控制

如果现在有另外一个页被读到了，会继续放到 LRU List 中，假设现在被读到的这个页又更新了，这个页就被称为脏页，这个脏页的指针就会被放入 Flush List 当中去

对于 LRU List 存放的是所有已经使用的页，不管这页是干净的页还是脏页，里面既有没有修改过的页也有已经发生修改过的页

Flush List 只包含脏，是只包含脏页的指针，所以 BP 的大小实际就是 Free List + LRU List

## 常用命令 show engine innodb status\G

    (root@localhost) [information_schema]> show engine innodb status\G
    *************************** 1. row ***************************
      Type: InnoDB
      Name:
    Status:
    ...
    ----------------------
    BUFFER POOL AND MEMORY  <--内存状态信息 
    ----------------------
    Total large memory allocated 1099431936
    Dictionary memory allocated 132117
    Buffer pool size   65536    <--总BP大小
    Free buffers       60752    <--空闲页列表
    Database pages     4784     <--在LRU List中的数量
    Old database pages 1514
    Modified db pages  0        <--脏页数量
    Pending reads      0
    Pending writes: LRU 0, flush list 0, single page 0
    Pages made young 0, not young 0
    0.00 youngs/s, 0.00 non-youngs/s
    Pages read 4013, created 771, written 2065
    0.00 reads/s, 0.00 creates/s, 0.00 writes/s
    No buffer pool page gets since the last printout
    Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
    LRU len: 4784, unzip_LRU len: 0
    I/O sum[0]:cur[0], unzip sum[0]:cur[0]
    ----------------------
    INDIVIDUAL BUFFER POOL INFO
    ----------------------
    ---BUFFER POOL 0          <--第几个BUFFER POOL，这里是第0个
    Buffer pool size   8192   <--每个BP的大小 8192*8个BP=一共有多少页*16K/1024=一共多少M的BP大小
    Free buffers       7628   <--空闲页列表
    Database pages     564    <--在LRU List中的数量
    Old database pages 209
    Modified db pages  0      <--脏页数量
    Pending reads      0
    Pending writes: LRU 0, flush list 0, single page 0
    Pages made young 0, not young 0
    0.00 youngs/s, 0.00 non-youngs/s
    Pages read 500, created 64, written 194
    0.00 reads/s, 0.00 creates/s, 0.00 writes/s
    No buffer pool page gets since the last printout
    Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
    LRU len: 564, unzip_LRU len: 0
    I/O sum[0]:cur[0], unzip sum[0]:cur[0]
    ---BUFFER POOL 1
    ...

    1 row in set (0.00 sec)

## LRU list

* LRU list
  * 最近最少使用算法
  * midpoint LRU 
    * 3/8
    * --innodb_old_blocks_pct={3/8}
    * --innodb_old_blocks_time
      * 避免扫描语句污染LRU

和朴素的 LRU 算法不同，MySQL 的只会将最新访问到的页放到中间 3/8 处，这么做是为了保证最热的数据都排在前面

从空闲列表读到 LRU 列表中的时候第一次会放到 midpoint 的点不是放到最前面，这个页如果第二次被读到的话，就会被放到最前面

这时候可以设置一个参数 innodb_old_blocks_time=1000 将最新访问的页定在 midpoint 多少时间，此时不是第二次读到就会马上被放到最前面，一定要等 1 秒钟

在 select * from table; 扫描 scan 的情况下是希望可以去设置 innodb_old_blocks_time 这个参数的，比如一个页里面有 10 条记录的话这个页会被读 10 次而不是 0 次 

数据库是一个并发的系统，如果这个页有 10 条记录一次读的话就意味着读这 10 条记录的时候，这个页要被锁住锁成只读的，这就以为这其他线程的插入就不被允许了，如果把一个操作放大成这样那数据库就做不到并发了

所以数据库是每读一条记录去读一个页然后马上释放的，然后把读到的位置(游标)保存下来，下次再要用的话打开游标继续读，但是读的时候位置可能发生变化，所以可能需要重新再去读这个页再继续读记录，这时候可能会有一些不一样，所以在 scan 的时候可能会遇到这样的问题

比如不设置 innodb_old_blocks_time 的时候，突然 scan 了一堆数据到 BP 里，但是这一堆 scan 的可能就用了一次就再也没用到过，那么 LRU 里就塞满了这类数据，这时就产生了扫描语句污染 LRU 的问题

所以像遇到 scan 操作的时候就可以设置这个参数，scan 完了之后再设回 0

    (root@localhost) [test]> set global innodb_old_block_time=1000;
    (root@localhost) [test]> select * from table;
    (root@localhost) [test]> set global innodb_old_block_time=0;

## 预热

* 预热
  * 使得数据库快速恢复到运行状态
  * 数据库重启
  * 数据库服务宕机
* 例子
  * 64G BP，10M/s读取，~100分钟
* 预热策略
  * 将LRU链表dump出来
  * 通过较顺序的方式预热50M~200M/s 
* 预热方法
  * SELECT COUNT(1) FROM table FORCE INDEX(PRIMARY)
  * SELECT COUNT(1) FROM index 
  * \>=MySQL5.6
    * innodb_buffer_pool_dump_now={ON|OFF}
    * innodb_buffer_pool_load_now={ON|OFF}
    * innodb_buffer_pool_dump_at_shutdown={ON|OFF}
    * innodb_buffer_pool_load_at_startup={ON|OFF}
    * innodb_buffer_pool_filename={ib_buffer_pool}
    * innodb_buffer_pool_load_abort={ON|OFF}

举个例子一个 300G 的 BP，在数据库刚启动的时候磁盘有大量的数据要 load 到 BP 中去，这时候需要的时间是非常长的，如果 load 的速度是 10M/s 的话，一共需要 300*100=30000秒=10个小时，这么长时间一般是没办法接受的，所以需要预热

SELECT COUNT(1) FROM table FORCE INDEX(PRIMARY) 和 SELECT COUNT(1) FROM index 这两种方法的缺点是并没有预热真正的热点数据，只不过把数据读进来了，100G当中真正的热点数据可能也就10G、20G的样子，这时一种力度非常粗的预热基本上没什么用

MySQL5.6 开始 

数据库关闭的时候可以做一个 dump 的操作 innodb_buffer_pool_dump_at_shutdown={ON|OFF}

数据库启动的时候可以把上面 dump 出来的东西 load 进来 innodb_buffer_pool_load_now={ON|OFF}

相关参数：

    (root@localhost) [test]> show variables like 'innodb_buffer_pool%';
    +-------------------------------------+----------------+
    | Variable_name                       | Value          |
    +-------------------------------------+----------------+
    | innodb_buffer_pool_chunk_size       | 134217728      |
    | innodb_buffer_pool_dump_at_shutdown | ON             |
    | innodb_buffer_pool_dump_now         | OFF            |
    | innodb_buffer_pool_dump_pct         | 25             |
    | innodb_buffer_pool_filename         | ib_buffer_pool |  <--默认dump出来的内容的文件名
    | innodb_buffer_pool_instances        | 8              |
    | innodb_buffer_pool_load_abort       | OFF            |
    | innodb_buffer_pool_load_at_startup  | ON             |
    | innodb_buffer_pool_load_now         | OFF            |
    | innodb_buffer_pool_size             | 1073741824     |
    +-------------------------------------+----------------+
    10 rows in set (0.00 sec)

dump 的是 {space, page_num}，而不是整个内存的东西，直接 dump 内存里的东西话太大了

ib_buffer_pool 实际上就是一个文本文件，里面存放的就是 LRU List 中的无序的 {space, page_num}，下一次装在的时候把 {space, page_num} 里面的页再去读一次，这时候才算是一个比较好的预热策略

在预热的时候还会做一个操作，根据 {space, page_num} 进行一个排序然后再进行读取

查看 ib_buffer_pool 内容前几行：

    $ head -n 10 ib_buffer_pool
    0,9
    0,4122
    16,2
    16,1
    16,3
    33,21952
    8,2
    8,1
    8,3
    33,21503

MySQL5.7 可以 dump LRU 的前 25%(默认)，也可以通过 innodb_buffer_pool_dump_pct 参数调整数值，25% 可能小了点，调成 40% 左右

在 my.cnf 中配置这个参数

    # innodb
    innodb_buffer_pool_size = 1G
    innodb_buffer_pool_instances = 8
    innodb_log_file_size = 128M
    innodb_online_alter_log_max_size = 256M # 配置好可以调大至1G
    innodb_buffer_pool_dump_pct=40 # new

# 后台线程

通过 performance_schema 库查看后台有哪些线程：

    (root@localhost) [performance_schema]> select name from threads where name like 'thread/innodb%';
    +----------------------------------------+
    | name                                   |
    +----------------------------------------+
    | thread/innodb/io_ibuf_thread           |  <--上面介绍过dump数据导入进来的线程
    | thread/innodb/io_log_thread            |  <--写log日志的线程
    | thread/innodb/io_read_thread           |  <--包含read的是异步读的线程
    | thread/innodb/io_read_thread           |
    | thread/innodb/io_read_thread           |
    | thread/innodb/io_read_thread           |
    | thread/innodb/io_write_thread          |  <--包含write的是异步写的线程
    | thread/innodb/io_write_thread          |
    | thread/innodb/io_write_thread          |
    | thread/innodb/io_write_thread          |
    | thread/innodb/page_cleaner_thread      |  <--刷新脏页的线程数量
    | thread/innodb/srv_lock_timeout_thread  |  <--监控锁超时的线程
    | thread/innodb/srv_error_monitor_thread |  <--错误信息监控线程
    | thread/innodb/srv_monitor_thread       |
    | thread/innodb/srv_master_thread        |  <--刷新重做日志的线程
    | thread/innodb/srv_purge_thread         |
    | thread/innodb/srv_worker_thread        |
    | thread/innodb/srv_worker_thread        |
    | thread/innodb/srv_worker_thread        |
    | thread/innodb/buf_dump_thread          |
    | thread/innodb/dict_stats_thread        |  <--元数据线程
    +----------------------------------------+
    21 rows in set (0.00 sec)

异步读和异步写的线程可以由参数进行控制，对于 io_write_thread 需要调大一些特别是 SSD 的话

page_cleaner_thread 也建议设得尽可能大，和 io_write_thread 一样大就差不多了

脏页的刷新是通过 innodb_io_capacity 参数来进行控制的，每秒会刷新 innodb_io_capacity 设置的页数，如果是 4000 每秒就会刷新 4000 个页，每 10 秒还会进行一次全量的刷新

my.cnf：

    # innodb
    innodb_buffer_pool_size = 1G
    innodb_buffer_pool_instances = 8
    innodb_log_file_size = 128M
    innodb_online_alter_log_max_size = 256M # 配置好可以调大至1G
    innodb_buffer_pool_dump_pct=40
    innodb_write_io_threads=16 # new
    innodb_page_cleaners=16    # new
    innodb_io_capacity=4000    # new

### 计算哪个线程 MySQL 的使用率是最高的（可用于故障诊断）

因为 MySQL 是个单进程多线程的架构，所以使用 top 命令之能看到一个 myqld 进程

如果还需要查看线程需要使用 top -H 命令，再通过 performance_schema 库中的 THREAD_OS_ID 表可以和 top 命令里的 PID 对应起来

    (root@localhost) [performance_schema]> select name,type,thread_os_id from threads;
    +----------------------------------------+------------+--------------+
    | name                                   | type       | thread_os_id |
    +----------------------------------------+------------+--------------+
    | thread/sql/main                        | BACKGROUND |       351084 |
    | thread/sql/thread_timer_notifier       | BACKGROUND |       351085 |
    | thread/innodb/io_ibuf_thread           | BACKGROUND |       351086 |
    | thread/innodb/io_log_thread            | BACKGROUND |       351087 |
    | thread/innodb/io_read_thread           | BACKGROUND |       351088 |
    | thread/innodb/io_read_thread           | BACKGROUND |       351089 |
    | thread/innodb/io_read_thread           | BACKGROUND |       351091 |
    | thread/innodb/io_read_thread           | BACKGROUND |       351090 |
    | thread/innodb/io_write_thread          | BACKGROUND |       351092 |
    | thread/innodb/io_write_thread          | BACKGROUND |       351094 |
    | thread/innodb/io_write_thread          | BACKGROUND |       351093 |
    | thread/innodb/io_write_thread          | BACKGROUND |       351095 |
    | thread/innodb/io_write_thread          | BACKGROUND |       351096 |
    | thread/innodb/io_write_thread          | BACKGROUND |       351097 |
    | thread/innodb/io_write_thread          | BACKGROUND |       351099 |
    | thread/innodb/io_write_thread          | BACKGROUND |       351098 |
    | thread/innodb/io_write_thread          | BACKGROUND |       351100 |
    | thread/innodb/io_write_thread          | BACKGROUND |       351101 |
    | thread/innodb/io_write_thread          | BACKGROUND |       351102 |
    | thread/innodb/io_write_thread          | BACKGROUND |       351104 |
    | thread/innodb/io_write_thread          | BACKGROUND |       351103 |
    | thread/innodb/io_write_thread          | BACKGROUND |       351105 |
    | thread/innodb/io_write_thread          | BACKGROUND |       351106 |
    | thread/innodb/io_write_thread          | BACKGROUND |       351107 |
    | thread/innodb/page_cleaner_thread      | BACKGROUND |       351108 |
    | thread/innodb/srv_lock_timeout_thread  | BACKGROUND |       351117 |
    | thread/innodb/srv_error_monitor_thread | BACKGROUND |       351118 |
    | thread/innodb/srv_monitor_thread       | BACKGROUND |       351119 |
    | thread/innodb/srv_master_thread        | BACKGROUND |       351120 |
    | thread/innodb/srv_purge_thread         | BACKGROUND |       351121 |
    | thread/innodb/srv_worker_thread        | BACKGROUND |       351122 |
    | thread/innodb/srv_worker_thread        | BACKGROUND |       351123 |
    | thread/innodb/srv_worker_thread        | BACKGROUND |       351124 |
    | thread/innodb/buf_dump_thread          | BACKGROUND |       351125 |
    | thread/innodb/dict_stats_thread        | BACKGROUND |       351126 |
    | thread/sql/signal_handler              | BACKGROUND |       351129 |
    | thread/sql/compress_gtid_table         | FOREGROUND |       351130 |
    | thread/sql/one_connection              | FOREGROUND |       365399 |  <--这个表示的用户连接
    | thread/sql/one_connection              | FOREGROUND |       375312 |
    +----------------------------------------+------------+--------------+
    39 rows in set (0.00 sec)

通过系统线程查看用户线程在干什么：

    (root@localhost) [performance_schema]> select name,type,processlist_id,thread_os_id from threads where thread_os_id
    = 375312;
    +---------------------------+------------+----------------+--------------+
    | name                      | type       | processlist_id | thread_os_id |
    +---------------------------+------------+----------------+--------------+
    | thread/sql/one_connection | FOREGROUND |              4 |       375312 |
    +---------------------------+------------+----------------+--------------+
    1 row in set (0.00 sec)

    (root@localhost) [performance_schema]> show processlist;
    +----+------+-----------+--------------------+---------+------+----------+------------------+
    | Id | User | Host      | db                 | Command | Time | State    | Info             |
    +----+------+-----------+--------------------+---------+------+----------+------------------+
    |  3 | root | localhost | performance_schema | Query   |    0 | starting | show processlist |
    |  4 | root | localhost | performance_schema | Sleep   |  145 |          | NULL             |   <--对应上面的processlist_id
    +----+------+-----------+--------------------+---------+------+----------+------------------+

show processlist 也可以通过表来进行查询：

    (root@localhost) [information_schema]> select * from processlist;
    +----+------+-----------+--------------------+---------+------+-----------+---------------------------+
    | ID | USER | HOST      | DB                 | COMMAND | TIME | STATE     | INFO                      |
    +----+------+-----------+--------------------+---------+------+-----------+---------------------------+
    |  3 | root | localhost | information_schema | Query   |    0 | executing | select * from processlist |
    |  4 | root | localhost | performance_schema | Sleep   |  444 |           | NULL                      |
    +----+------+-----------+--------------------+---------+------+-----------+---------------------------+
    2 rows in set (0.00 sec)

### 查看 IO 使用率最高的线程

    $ iotop -p 365445 # 根据进程pid
    $ iotop -u mysql  # 根据进程名