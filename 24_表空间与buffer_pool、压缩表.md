# InnoDB 存储引擎缓冲池

缓冲池就是把数据页缓存到内存里，下一次再查数据的话，如果缓存里已经有了的话就会直接命中缓存里的内容

* InnoDB Buffer Pool 
  * 主要存放的是热点数据
  * Like Oracle SGA
  * --innodb_buffer_pool_size 通常来说越大越好
  * the more ... the more 服务器物理内存的60%~70%
  * 生产环境64G、96G

每张表都有一个表空间，每张表空间都是由一个一个页组成的，每个页里面有一行一行的记录，页是最小的 IO 单位

比如找 id=1 的这条记录的话，首先通过 id 的索引把 id 所在的这个页读到内存里面，读到内存里面之后通过 row offset array 做一个二分查找法定位到某一条具体的记录，那怎么样在内存里标记这些页呢？

在 MySQL 中用一个叫 HASH->(space, page_number) 的方式标记每一个页，其中 sapce 表示的是表空间 id，系统表空间是 0，page_number 代表偏移量表示的是这个表空间的第几个页

查看 space：

    (root@localhost) [information_schema]> select name, space from INNODB_SYS_TABLESPACES;
    +---------------------------------+-------+
    | name                            | space |
    +---------------------------------+-------+
    | mysql/plugin                    |     2 |
    | mysql/servers                   |     3 |
    | mysql/help_topic                |     4 |
    | mysql/help_category             |     5 |
    | mysql/help_relation             |     6 |
    | mysql/help_keyword              |     7 |
    | mysql/time_zone_name            |     8 |
    | mysql/time_zone                 |     9 |
    | mysql/time_zone_transition      |    10 |
    | mysql/time_zone_transition_type |    11 |
    | mysql/time_zone_leap_second     |    12 |
    | mysql/innodb_table_stats        |    13 |
    | mysql/innodb_index_stats        |    14 |
    | ...                             |   ... |
    | test/z                          |   132 |
    +---------------------------------+-------+
    58 rows in set (0.00 sec)

查看偏移量，缓冲池里有哪些页：

    (root@localhost) [information_schema]> select space,page_number,page_type from INNODB_BUFFER_PAGE limit 10;
    +-------+-------------+-------------+
    | space | page_number | page_type   |
    +-------+-------------+-------------+
    |     0 |           7 | SYSTEM      |
    |     0 |           3 | SYSTEM      |
    |     0 |           2 | INODE       |
    |     0 |           4 | IBUF_INDEX  |
    |     0 |          11 | INDEX       |
    |     0 |           1 | IBUF_BITMAP |
    |     0 |           5 | TRX_SYSTEM  |
    |     0 |           6 | SYSTEM      |
    |     0 |          45 | SYSTEM      |
    |     0 |          46 | SYSTEM      |
    +-------+-------------+-------------+
    10 rows in set (0.06 sec)

在线上不要取查这张表，数据量很大查询会非常慢


查看 InnoDB 数据引擎信息：

    (root@localhost) [information_schema]> show engine innodb status\G
    *************************** 1. row ***************************
      Type: InnoDB
      Name:
    Status:
    =====================================
    2022-06-18 21:40:36 0x7f6b8c196700 INNODB MONITOR OUTPUT
    =====================================
    Per second averages calculated from the last 8 seconds
    -----------------
    BACKGROUND THREAD
    -----------------
    srv_master_thread loops: 6 srv_active, 0 srv_shutdown, 28118 srv_idle
    srv_master_thread log flush and writes: 28124
    ----------
    SEMAPHORES
    ----------
    OS WAIT ARRAY INFO: reservation count 14
    OS WAIT ARRAY INFO: signal count 14
    RW-shared spins 0, rounds 4, OS waits 2
    RW-excl spins 0, rounds 0, OS waits 0
    RW-sx spins 0, rounds 0, OS waits 0
    Spin rounds per wait: 4.00 RW-shared, 0.00 RW-excl, 0.00 RW-sx
    ------------
    TRANSACTIONS
    ------------
    Trx id counter 663300
    Purge done for trx's n:o < 0 undo n:o < 0 state: running but idle
    History list length 0
    LIST OF TRANSACTIONS FOR EACH SESSION:
    ---TRANSACTION 421574878472016, not started
    0 lock struct(s), heap size 1136, 0 row lock(s)
    --------
    FILE I/O
    --------
    I/O thread 0 state: waiting for completed aio requests (insert buffer thread)
    I/O thread 1 state: waiting for completed aio requests (log thread)
    I/O thread 2 state: waiting for completed aio requests (read thread)
    I/O thread 3 state: waiting for completed aio requests (read thread)
    I/O thread 4 state: waiting for completed aio requests (read thread)
    I/O thread 5 state: waiting for completed aio requests (read thread)
    I/O thread 6 state: waiting for completed aio requests (write thread)
    I/O thread 7 state: waiting for completed aio requests (write thread)
    I/O thread 8 state: waiting for completed aio requests (write thread)
    I/O thread 9 state: waiting for completed aio requests (write thread)
    Pending normal aio reads: [0, 0, 0, 0] , aio writes: [0, 0, 0, 0] ,
    ibuf aio reads:, log i/o's:, sync i/o's:
    Pending flushes (fsync) log: 0; buffer pool: 0
    4042 OS file reads, 2145 OS file writes, 7 OS fsyncs
    0.00 reads/s, 0 avg bytes/read, 0.00 writes/s, 0.00 fsyncs/s
    -------------------------------------
    INSERT BUFFER AND ADAPTIVE HASH INDEX
    -------------------------------------
    Ibuf: size 1, free list len 3093, seg size 3095, 0 merges
    merged operations:
    insert 0, delete mark 0, delete 0
    discarded operations:
    insert 0, delete mark 0, delete 0
    Hash table size 276707, node heap has 0 buffer(s)
    Hash table size 276707, node heap has 0 buffer(s)
    Hash table size 276707, node heap has 0 buffer(s)
    Hash table size 276707, node heap has 0 buffer(s)
    Hash table size 276707, node heap has 0 buffer(s)
    Hash table size 276707, node heap has 0 buffer(s)
    Hash table size 276707, node heap has 0 buffer(s)
    Hash table size 276707, node heap has 0 buffer(s)
    0.00 hash searches/s, 0.00 non-hash searches/s
    ---
    LOG
    ---
    Log sequence number 8604583883
    Log flushed up to   8604583883
    Pages flushed up to 8604583883
    Last checkpoint at  8604583874
    0 pending log flushes, 0 pending chkp writes
    10 log i/o's done, 0.00 log i/o's/second
    ----------------------
    BUFFER POOL AND MEMORY  <--内存状态信息 
    ----------------------
    Total large memory allocated 1099431936
    Dictionary memory allocated 132117
    Buffer pool size   65536
    Free buffers       60752
    Database pages     4784
    Old database pages 1514
    Modified db pages  0
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
    ---BUFFER POOL 0
    Buffer pool size   8192
    Free buffers       7628
    Database pages     564
    Old database pages 209
    Modified db pages  0
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
    Buffer pool size   8192
    Free buffers       7558
    Database pages     634
    Old database pages 220
    Modified db pages  0
    Pending reads      0
    Pending writes: LRU 0, flush list 0, single page 0
    Pages made young 0, not young 0
    0.00 youngs/s, 0.00 non-youngs/s
    Pages read 507, created 127, written 324
    0.00 reads/s, 0.00 creates/s, 0.00 writes/s
    No buffer pool page gets since the last printout
    Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
    LRU len: 634, unzip_LRU len: 0
    I/O sum[0]:cur[0], unzip sum[0]:cur[0]
    ---BUFFER POOL 2
    Buffer pool size   8192
    Free buffers       7643
    Database pages     549
    Old database pages 209
    Modified db pages  0
    Pending reads      0
    Pending writes: LRU 0, flush list 0, single page 0
    Pages made young 0, not young 0
    0.00 youngs/s, 0.00 non-youngs/s
    Pages read 481, created 68, written 204
    0.00 reads/s, 0.00 creates/s, 0.00 writes/s
    No buffer pool page gets since the last printout
    Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
    LRU len: 549, unzip_LRU len: 0
    I/O sum[0]:cur[0], unzip sum[0]:cur[0]
    ---BUFFER POOL 3
    Buffer pool size   8192
    Free buffers       7529
    Database pages     663
    Old database pages 224
    Modified db pages  0
    Pending reads      0
    Pending writes: LRU 0, flush list 0, single page 0
    Pages made young 0, not young 0
    0.00 youngs/s, 0.00 non-youngs/s
    Pages read 535, created 128, written 320
    0.00 reads/s, 0.00 creates/s, 0.00 writes/s
    No buffer pool page gets since the last printout
    Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
    LRU len: 663, unzip_LRU len: 0
    I/O sum[0]:cur[0], unzip sum[0]:cur[0]
    ---BUFFER POOL 4
    Buffer pool size   8192
    Free buffers       7572
    Database pages     620
    Old database pages 209
    Modified db pages  0
    Pending reads      0
    Pending writes: LRU 0, flush list 0, single page 0
    Pages made young 0, not young 0
    0.00 youngs/s, 0.00 non-youngs/s
    Pages read 492, created 128, written 319
    0.00 reads/s, 0.00 creates/s, 0.00 writes/s
    No buffer pool page gets since the last printout
    Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
    LRU len: 620, unzip_LRU len: 0
    I/O sum[0]:cur[0], unzip sum[0]:cur[0]
    ---BUFFER POOL 5
    Buffer pool size   8192
    Free buffers       7521
    Database pages     671
    Old database pages 227
    Modified db pages  0
    Pending reads      0
    Pending writes: LRU 0, flush list 0, single page 0
    Pages made young 0, not young 0
    0.00 youngs/s, 0.00 non-youngs/s
    Pages read 543, created 128, written 320
    0.00 reads/s, 0.00 creates/s, 0.00 writes/s
    No buffer pool page gets since the last printout
    Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
    LRU len: 671, unzip_LRU len: 0
    I/O sum[0]:cur[0], unzip sum[0]:cur[0]
    ---BUFFER POOL 6
    Buffer pool size   8192
    Free buffers       7705
    Database pages     487
    Old database pages 0
    Modified db pages  0
    Pending reads      0
    Pending writes: LRU 0, flush list 0, single page 0
    Pages made young 0, not young 0
    0.00 youngs/s, 0.00 non-youngs/s
    Pages read 423, created 64, written 192
    0.00 reads/s, 0.00 creates/s, 0.00 writes/s
    No buffer pool page gets since the last printout
    Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
    LRU len: 487, unzip_LRU len: 0
    I/O sum[0]:cur[0], unzip sum[0]:cur[0]
    ---BUFFER POOL 7
    Buffer pool size   8192
    Free buffers       7596
    Database pages     596
    Old database pages 216
    Modified db pages  0
    Pending reads      0
    Pending writes: LRU 0, flush list 0, single page 0
    Pages made young 0, not young 0
    0.00 youngs/s, 0.00 non-youngs/s
    Pages read 532, created 64, written 192
    0.00 reads/s, 0.00 creates/s, 0.00 writes/s
    No buffer pool page gets since the last printout
    Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
    LRU len: 596, unzip_LRU len: 0
    I/O sum[0]:cur[0], unzip sum[0]:cur[0]
    --------------
    ROW OPERATIONS
    --------------
    0 queries inside InnoDB, 0 queries in queue
    0 read views open inside InnoDB
    Process ID=3524, Main thread ID=140098522285824, state: sleeping
    Number of rows inserted 196618, updated 0, deleted 0, read 65565
    0.00 inserts/s, 0.00 updates/s, 0.00 deletes/s, 0.00 reads/s
    ----------------------------
    END OF INNODB MONITOR OUTPUT
    ============================

    1 row in set (0.00 sec)

## 缓冲池存放的内容

* 数据页
* 索引页
* change buffer 
* 自适应哈希
* 锁 重要概念！MySQL的锁是存在内存里的

## 压缩的数据在 BP(buffer pool) 里的表示

MySQL 的压缩所有的出发点都是围绕着**性能**展开

对于一个压缩的页来说，在 BP 里很有可能会出现两个页，一个压缩的页和一个非压缩的页，当第一次这个页被读到的时候它读到的大小是 8K，第二次进行读取的时候就不需要再进行解压了，这时候读到的页是 16K 的

在 BP 里的一般都是热点数据，对热点数据是不可能每次都去进行解压的，那样对性能是有损耗的，所以说这里的解压只解压一次

对于 DML 操作，会先把 DML 操作的重做日志写入到压缩页的剩余空间，然后在解压的页上执行 DML 操作。如果这时候写日志写压缩的页写满了，会将解压的页重新压缩看是否弄压缩到原压缩页的大小，如果压缩不到的时候就会再次申请一个压缩页的大小，并且重新解压一个页

也就是说对压缩的表来说，首先它的读取只有第一次在磁盘读到内存的时候才做一次解压。写入的时候也并不是每条记录的写入都是需要压缩的，它是先写日志的，页写满的时候再做一次压缩，不够的话再做拆分

另外一点读取和写入它的 IO 的大小从 16K 变成了 8K 或者 4K，压缩表除了对空间会有一个减小以外对性能来说的也会有帮助，因为非常重要的一点是 IO 变小了，但是 CPU 使用率也会变高，但是 IO 的提升是要远远大于 CPU 的提升的，所以一个真正好的压缩算法应该是对性能有帮助的

日志记录在这个页里面的作用是不需要每次都对记录进行压缩，缺点是在内存里会有一个压缩页和非压缩页，内存使用率会降低

当然压缩不是说一定性能就会变快，还是需要进行一定的衡量的，但是默认开启压缩还是一个大方向

OLAP 的系统由于牵扯到 JOIN 的问题，不建议开启压缩