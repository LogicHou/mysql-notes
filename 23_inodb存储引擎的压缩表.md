# InnoDB 存储引擎特性

## InnoDB 存储引擎特点

* Fully ACID
* Row-level locking
* Multi-version concurrency control (MVCC)
* Foreign key support 
* Automatic deadlock detection
* High performance, High scalability, High availability
  
## InnoDB 存储引擎文件

表空间结构：

* 独立表空间结构
* 全局表空间结构
* undo 表空间结构（from MySQL5.6）

重做日志文件：

* 物理逻辑日志
* 没有 Oracle 的归档重做日志

### 表空间 

* 内部由多个段对象组成
* 每个段由区组成
* 每个区由页组成
* 每个页里保存数据

### 表空间 -- 区

* 区是最小的空间申请单位
* 区的大小固定为1M
  * 16K 64个页
  * 8K  128个页
  * 4K  256个页
* 通常一次申请4个区的大小

页的大小由 innodb_page_size 参数控制

    (root@localhost) [(none)]> show variables like 'innodb_page_size';
    +------------------+-------+
    | Variable_name    | Value |
    +------------------+-------+
    | innodb_page_size | 16384 |
    +------------------+-------+
    1 row in set (0.00 sec)

### 共享表空间和系统表空间

是由这个参数所控制的，默认是使用系统表空间：

    (root@localhost) [(none)]> show variables like 'innodb_file_per_table';
    +-----------------------+-------+
    | Variable_name         | Value |
    +-----------------------+-------+
    | innodb_file_per_table | ON    |    <--每个文件一个表空间，MySQL5.1版本后支持
    +-----------------------+-------+
    1 row in set (0.00 sec)

如果设置为 0(OFF)，新建表的时候数据目录下只会生成 frm 表定义文件，而不会再生成 ibd 数据存放文件，此时的数据会存放在数据目录下的 ibdata1 文件中

**两种方式在速度方面是一样快的，并没有区别**，因为即使是共享表空间也是按照 1M 大小连续申请空间的，这 1M 的空间可以认为是连续的，这样的话是不会有任何的区别的，它们的分配和管理机制都是一样的

引入独立表空间主要是为了管理方便，使用独立表结构每个表都会有对应表名的 frm 和 ibd 文件，如果存在 ibdata1 文件中就根本无法区分开来了

还有就是如果需要删除文件的话，独立表空间的删除会很简单，而且删除后空间是可以被释放的，但是对于 ibdata1 来说它是一个整体的文件，它之能增长不能删除。所以如果是共享表空间的话，去删除一张表的时候只是把这张表对应的空间标记为可用，但是空间并不能释放

另外不要在数据目录下改动和删除任何文件！！！

### 表空间 -- 页 

* 页 
  * 最小的I/O操作单位
* 普通用户表
  * 默认每个页16K
  * --innodb_page_size（from5.6）
* 压缩表
  * 基于页的压缩
  * 每个表的页大小可以不同

可以通过下面的命令压缩表：

    ALTER TABLE XXX
    ENGINE=InnoDB
    ROW_FORMAT=compressed,KEY_BLOCK_SIZE=8;  # 8表示用几K的页大小进行压缩

注意并不是页设置的越小压缩率就越小，因为这个压缩算法是这样的，如果 16K 可以压缩到 8K 那么就可以压缩到 8K，如果 16K 只能压缩到 12K，那么会压缩出两个 8K，这种情况下压缩率基本是变化不大的

压缩成功率方面，16K 压缩到 8K 的成功率大致在 80% 左右，但是从 16K 压缩到 4K 成功率不会那么高

#### 设置成 16K 的意义

    ALTER TABLE XXX
    ENGINE=InnoDB
    ROW_FORMAT=compressed,KEY_BLOCK_SIZE=16;

虽然设置的和默认一样是 16K，但是里面的内容是经过压缩的，简单里说就是压缩过的 16K 的页比非压缩的 16K 的页存放的数据要更多

### 表空间 -- 记录

* 用户记录保存在数据页中
* 记录格式由ROW_FORMAT选项决定
  * REDUDENT：兼容老版本InnoDB
  * COMPACT: 默认格式         <-- 5.6开始默认使用的
  * COMPRESSED：支持压缩
  * DYNAMIC：大对象记录优化     <-- 5.7 开始默认使用的

        (root@localhost) [dbt3]> show variables like 'innodb%row%';
        +----------------------------+---------+
        | Variable_name              | Value   |
        +----------------------------+---------+
        | innodb_api_disable_rowlock | OFF     |
        | innodb_default_row_format  | dynamic | <--默认和COMPACT一样，只有存大对象的时候会优化
        +----------------------------+---------+
        2 rows in set (0.01 sec)

一个页由页头Page header、页尾Page trailer、Row记录、row offset array 构成

其中页头、页尾用来保证这个页的写入是完整的，页头页尾都有一个 LSN 校验值，如果页头的 LSN 和页尾的 LSN 相等的话这个页的写入就是完整的

页头 Page header、页尾 Page trailer 会占用200个字节，所以每个页能存放的数据实际是 16K - 200字节

B+树索引只能快速定位到**记录所在的页**，页内的数据存放是无序的，row offset array 就是对页内的记录进行排序的，和数据与数据之间的 next 指针不同，row offset array可以用来快速定位到页内的某一条记录

### 表空间 -- 记录2

* InnoDB 存储引擎是索引组织表
  * Index Organized Table 
  * 与ORacle IOT表类似
  * 适合OLTP应用
* 叶子节点存放所有数据
  * 索引即数据
  * 数据即索引
* 聚集索引记录存在以下系统列
  * rowid：6字节int B+树索引键值
  * trx id： 6字节 事务id
  * roll pointer：7字节 undo指针列

假设有一张如下的表

| a 主键 | b 普通索引 | c   | d   |
| ------ | ---------- | --- | --- |
| ...    | ...        | ... | ... |

这时候这张表有两棵B+树索引，一个是主键的聚集索引，一个是普通索引

主键索引叶子节点存放的是一整行完整的记录，在主键索引中的每一条记录是有隐藏列的

    | rowid | trx id | roll pointer | a | b | c | d |