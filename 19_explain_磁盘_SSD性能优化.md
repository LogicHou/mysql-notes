# explain 磁盘 SSD性能优化

## 全文索引

### 概述

* 搜索引擎的实现核心技术，搜索类似 WHERE col LIKE '%XXX%';
* 首先需要通过分词进行各词的提取
* 支持在 VARCHAR、CHAR、TEXT等类型上创建全文索引
* MySQL 5.6 版本仅 MyISAM 支持全文索引
* MySQL 5.6 版本 InnoDB引擎支持全文索引
* MySQL 5.7 版本支持中文、日文的全文索引（真正生产环境可用）
* 目前一张表之能有一个全文索引
* 添加全文索引时，表是只读的，不可写入更新

用法：

    # 在title，body列上创建全文索引
    ALTER TABLE XXX ADD FULLTEXT INDEX IDX_XXX(title, body)

### 全文索引 SQL 查询 

* 不能使用 LIKE 进行
* 需要使用全文索引的语法
* 全文索引的搜索支持三种模式
  * IN NATURAL LANGUAGE MODE 自然语言模式
  * IN BOOLEAN MODE 布尔模式
  * QUERY EXPANSION 语言扩展查询

用法：

    mysql> SELECT * FROM articles
        WHERE MATCH (title,body)
        AGAINST ('database' IN NATURAL LANGUAGE MODE);
    +----+-------------------+------------------------------------------+
    | id | title             | body                                     |
    +----+-------------------+------------------------------------------+
    |  1 | MySQL Tutorial    | DBMS stands for DataBase ...             |
    |  5 | MySQL vs. YourSQL | In the following database comparison ... |
    +----+-------------------+------------------------------------------+
    2 rows in set (0.00 sec)

    # 查看相关性
    mysql> SELECT id, body, MATCH (title,body) AGAINST
        ('Security implications of running MySQL as root'
        IN NATURAL LANGUAGE MODE) AS score
        FROM articles WHERE MATCH (title,body) AGAINST
        ('Security implications of running MySQL as root'
        IN NATURAL LANGUAGE MODE);
    +----+-------------------------------------+-----------------+
    | id | body                                | score           |
    +----+-------------------------------------+-----------------+
    |  4 | 1. Never run mysqld as root. 2. ... | 1.5219271183014 |   <--score数字越大相关性越高
    |  6 | When configured properly, MySQL ... | 1.3114095926285 |
    +----+-------------------------------------+-----------------+
    2 rows in set (0.00 sec)

    mysql> SELECT * FROM articles WHERE MATCH (title,body)
        AGAINST ('+MySQL -YourSQL' IN BOOLEAN MODE);  -- 包含MySQL但是不包含YourSQL关键字的
    +----+-----------------------+-------------------------------------+
    | id | title                 | body                                |
    +----+-----------------------+-------------------------------------+
    |  1 | MySQL Tutorial        | DBMS stands for DataBase ...        |
    |  2 | How To Use MySQL Well | After you went through a ...        |
    |  3 | Optimizing MySQL      | In this tutorial we will show ...   |
    |  4 | 1001 MySQL Tricks     | 1. Never run mysqld as root. 2. ... |
    |  6 | MySQL Security        | When configured properly, MySQL ... |
    +----+-----------------------+-------------------------------------+

    # 通常不要使用 WITH QUERY EXPANSION
    mysql> SELECT * FROM articles
        WHERE MATCH (title,body)
        AGAINST ('database' WITH QUERY EXPANSION);        <--会把与database相关的（比如MySQL、Oracle数据库）所有的单词搜索出来
    +----+-----------------------+------------------------------------------+
    | id | title                 | body                                     |
    +----+-----------------------+------------------------------------------+
    |  5 | MySQL vs. YourSQL     | In the following database comparison ... |
    |  1 | MySQL Tutorial        | DBMS stands for DataBase ...             |
    |  3 | Optimizing MySQL      | In this tutorial we will show ...        |
    |  6 | MySQL Security        | When configured properly, MySQL ...      |
    |  2 | How To Use MySQL Well | After you went through a ...             |
    |  4 | 1001 MySQL Tricks     | 1. Never run mysqld as root. 2. ...      |
    +----+-----------------------+------------------------------------------+
    6 rows in set (0.00 sec)

## 地理空间索引

### 概述

* MySQL 5.6 版本仅 MyISAM 引擎支持地理空间索引
* MySQL 5.7 版本 InnoDB 引擎支持地理空间索引
* 比较推荐用 Redis 或者 MongoDB 搞

## 磁盘类型与 IOPS 性能指标

### 磁盘的访问模式

* 顺序访问
  * 顺序地访问磁盘上的块
* 随机访问
  * 随机地访问磁盘上的块

### 磁盘的分类

HDD 机械磁盘

SSD 固态硬盘，用 MySQL 推荐要用 SSD

* 传统磁盘
* 数据库的优化都是根据机械磁盘特性
  * 随机转顺序
* 通过磁头旋转进行数据定位
* 常见转速
  * 笔记本硬盘：5400转/分钟
  * 桌面硬盘：7200转/分钟
  * 服务器硬盘：10000转/分钟，15000转/分钟

机械磁盘性能

* 顺序访问性能较好
  * 100M/s 
  * 磁盘吞吐率
* 随机访问性能较差
  * 需要磁头旋转定位
  * IOPS 较低

IOPS 

* IO per second
* 表示每秒随机访问 IO 的能力
* 机械硬盘
  * 100 ~ 200
* 固态硬盘
  * 50000+

提升 IOPS 性能手段

* 通过 RAID 技术
  * 功耗非常高
* 通过购买共享存储设备
  * 加个非常昂贵
* 然而提升都是非常有限的
  * IOPS 以千为单位

SSD

* 纯电设备
* 由 Flash Memory 组成
* 没有读写磁头
* IOPS 高
  * 50000+ IOPS 
  * 读写速度非对称
* 性能指标
  * RPM（rotations per minute）
    * 5400
    * 7200
    * 10000
    * 15000
* SATA
  * 120 ~ 150 IOPS
* SAS 
  * 150 ~ 200 IOPS
* 性能下降，早期 SSD 随着使用时间变长，速度会越来越慢
* 现在服务器级别的 SSD 寿命都不用太担心，5 年内报废都是一个可以接受的时间点
* 莫名的性能波动，也不用太担心
* SSD 与数据库优化
  * 磁盘调度算法设置为：deadline 或者 noop
  * InnoDB 存储引擎参数设置
    * innodb_flush_neighbors=0 刷新效率更高
    * innodb_log_file_size=4G  日志大小，IO更高的高设备保可以改成 8G 或者 16G
  * 结论
    * 性能更平稳
    * 可以有大约 15% 的性能提升

查看当前磁盘调度算法的命令：

    $ cat /sys/block/sda/queue/scheduler
    [mq-deadline] kyber bfq none

    # 如果不是deadling，通过下面命令修改
    $ echo deadline > /sys/block/sda/queue/scheduler

服务器上线标准中这个就应该为 deadline

SSD 选择

* PCIE or SATA/SAS
* SATA/SAS 易于安装与升级
* SATA/SAS 与 PCIE 的性能差距逐渐缩小
* PCIE 的性能很少有应用可以完全使用
* 优先选择 SATA/SAS 接口的 SSD

SSD 品牌推荐

* Intel
* FusionIO
* 宝存
* 选购时注意 “寿命” 指标
  * 每天全量写10次，保证5年

## 文件系统与操作系统

文件系统

* 推荐 xfs/ext4
* noatime
* nobarrier

操作系统

* 推荐 Linux 操作系统
* 关闭 swap
* 磁盘调度算法设置为 deadline

    $ mount -o noatime,nobarrier /dev/sda /data
    $ echo deadline > /sys/block/sda/queue/scheduler