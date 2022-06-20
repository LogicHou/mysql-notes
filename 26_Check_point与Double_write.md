# CHECKPOINT 检查点技术

* Checkpoint
  * 缩短数据库的恢复时间
  * 缓冲池不够用时，将脏页刷新到磁盘
  * 重做日志不可用时，**刷新脏页**

在某一个时间点，一个页在 BP 和磁盘上可能是不一致的，但是会保证最终一致

如果没有 Checkpoint 通过日志恢复数据会很困难，因为当日志增长到非常大后恢复数据会耗费非常多非常多的时间，几乎是一种很难操作的手段

每一个页都会有一个 LSN 号，当这个页发生更新这个页的 LSN 号就会发生变化。LSN 存在三个地方，一个是每个页有一个 LSN，每个数据库实例也有 LSN（检查点），日志里面也有 LSN

Checkpoint 机制会定期的对磁盘进行刷新，比如说现在 BP 中的页已经刷新到 LSN200 这个位置，DISK 上面的页只刷新到 LSN100，那么在宕机后做恢复的时候只要恢复 100~200 之间的数据即可

## LSN 回放过程示例

BP 中的一个页刚开始被都进去的时候 LSN 是100，如果这个页发生了修改就会把它的 LSN 修改成 130，如果这个页修个了它对应的这个事务提交了，那么会把这个页的修改记录到 redolog 里面，这时候 MySQL 全局的 LSN 就变成了 130

如果这个时候又有另外一个页读进来的时候它的 LSN 是 50，现在又对它进了修改，那么它的 LSN 变成了 140，然后写 redolog，这时候 MySQL 全局的 LSN 变成了 140

假设这时候 BP 中 LSN130 的这个页刷新到磁盘了，这时候磁盘上这个页的老的页还是旧的 LSN，如果这个时候宕机了，这个时候 130 的这个页已经刷回去了没有问题，然后通过 redolog 从 130 开始把 140 回放进去，数据就恢复过去了

## 查看 LSN：

    (root@localhost) [(none)]> show engine innodb status\G
    ...
    ---
    LOG
    ---
    Log sequence number 8604591333    <--当前内存里最大的LSN位置
    Log flushed up to   8604591333    <--当先刷新代词日志磁盘位置的LSN位置
    Pages flushed up to 8604591333    <--当前最大的页的LSN位置
    Last checkpoint at  8604591324    <--全局LSN位置
    0 pending log flushes, 0 pending chkp writes
    10 log i/o's done, 0.00 log i/o's/second
    ----------------------
    BUFFER POOL AND MEMORY
    ----------------------
    ...

注意到 Last checkpoint 的值其实是总是小于等于 Pages flushed up 的，只有每个页都只改一次的时候才会相等

# DOUBLE WRITE 

* 目的：数据写入的可靠性
* partial write
  * 16K的页只写入4K，6K，8K，12K的情况
  * 不可通过redo log进行恢复
  * Like Oracle media corrupt

如果 page header 更新了但是 page trailer 没有更新，就判断这不是一个完整的页，不是一个干净的页是不能够通过 redo log 来进行恢复的

* 解决方法
  * 引入 doublewrite
* doublewrite
  * 存在一个段对象doublewrite
  * 2M大小（both file and memory）
  * 页在刷新时首先顺序地写入到doublewrite
  * 然后再刷新回磁盘

页在更新的时候，并不是直接先写到对应的表空间文件，是先写入到一个叫 doublewrite 的对象以后再写入到磁盘，所以 DOUBLE WRITE 的意思就是一次写入到 doublewrite 队列，一次写入到磁盘数据文件

需要注意的不是每个页都会每次去 doublewrite，而是先把 128 个页也就是 2M，先写 doublewrite 的共享表空间文件，然后再写磁盘表空间文件

如果发生 partial write 问题的话，因为 doublewrite 里已经有一个干净的页的副本就可以把这个页恢复过去，如果 doublewrite 页发生了partial write 问题，数据文件里的页还是干净的并没有被破坏，可以通过 redo log 来进行恢复，doublewrite 和 数据文件里总有一份干净的页，这就是 doublewrite

* 性能开销
  * doublewrite写入是顺序的
  * 性能开销取决于写入量
  * 通常5% ~ 25%
  * slave服务器可考虑是否关闭

doublewrite 默认是打开的，可以通过 innodb_doublewrite 参数关闭：

    (root@localhost) [(none)]> show variables like '%double%';
    +--------------------+-------+
    | Variable_name      | Value |
    +--------------------+-------+
    | innodb_doublewrite | ON    |
    +--------------------+-------+
    1 row in set (0.02 sec)

在原子写入的硬件/文件系统上可以关闭这个特性

* atomic write
  * 磁盘
    * Fusion-IO
  * 文件系统
    * ZFS
    * btrfs
  * --skip-doublewrite

总之不管在哪建议不要关闭这个特性

# Insert Buffer 

* 提高辅助索引的插入性能
* none unique secondary index

提升二级索引的增删改的性能，二级索引有一个很大的特性就是它是随机的，当然也有时间类型这种比较顺序的数据类型

## Insert Buffer  工作原理

* 先判断插入的非聚集索引页是否在缓冲池中，若在，则直接插入
* 若不在，则先放入到一个Insert Buffer对象中
  * Insert Buffer也是一颗B+树 
  * 每次最多缓存2K的记录
* 当读取辅助索引页到缓冲池，将Insert Buffer中该页的记录合并到辅助索引页

## 潜在问题

* 最大可以使用1/2缓冲吃内存
  * --innodb-change-buffer-max-size={25}
* shutdown不进行insert buffer记录的合并
* insert buffer开始进行合并时插入性能开始下降

# APAPTIVE HASH INDEX 自适应哈希索引

* 搜索时间复杂度
  * B+树 O(T_height)
  * 哈希表 O (1)
* 根据B+树的访问模式构建哈希
  * 猜测基本没有开销
* 仅对热点页中的记录创建哈希索引
* 非持久化
* 仅支持点查询（point select）

|                | B+树索引                 | 自适应哈希索引   |
| -------------- | ------------------------ | ---------------- |
| 查询时间复杂度 | O（树的高度）            | O（1）           |
| 是否持久化     | 是，并通过日志保证完整性 | 否，仅在内存     |
| 索引对象       | 页                       | 热点页所在的记录 |

## 创建过程

* 索引是否被访问了17次
* 索引中某个页已经被访问了至少100次
* 对索引中的页访问的模式都需要相同
  * idx_a_b(a,b)
  * WHERE a=xxx 
  * WHERE a=xxx and b=xxx
  * 如果既通过a访问又通过 ab 访问，就不会对这个页创建自适应哈希索引

这个特性通常来说没什么太大的作用，建议关闭：

    (root@localhost) [(none)]> show variables like '%innodb%ash%';
    +----------------------------------+-------+
    | Variable_name                    | Value |
    +----------------------------------+-------+
    | innodb_adaptive_hash_index       | ON    |
    | innodb_adaptive_hash_index_parts | 8     |
    +----------------------------------+-------+
    2 rows in set (0.00 sec)

my.cnf:

    # innodb
    ...
    innodb_io_capacity=4000
    innodb_adaptive_hash_index=0 #new

# FLUSH NEIGHTBOR PAGE 

* Flush Neightbor Page（FNP）
  * 试图刷新页所在区中的所有脏页
* 传统机械磁盘有效
* SSD关闭此功能
* MySQL 5.6
  * --innodb_flush_neighbors={1|0}

这个功能的出发点是因为一个区里的数据是连续的，所以刷新这一个区的脏页是比较顺序的，刷新后性能就会提升

但是这样的一个刷新太狠了，刷一个页是 16K，而这个要刷 1M。这个功能的出发点是好的，问题是这个页刷干净了过一会又变脏了，因为一个页可以反复的被更新本来只需要刷 10 次，然后现在可能要刷 20 次、30 次

在 SSD 这样的设备上随机读写性能已经足够好了不需要做这样的一种优化

my.cnf 中关闭:

    # innodb
    ...
    innodb_adaptive_hash_index=0
    innodb_flush_neighbors=0 #new