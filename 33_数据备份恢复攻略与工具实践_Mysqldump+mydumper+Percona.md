# 恢复 

## 一般的恢复方法

    $ vim bbb.sql
    create database bbb;

    $ mysql < bbb.sql   # 恢复

## 对压缩的文件进行恢复

    $ ls -lh tpcc1000.backup.tgz
    -rw-r--r-- 1 root root 124M Jun 26 13:40 tpcc1000.backup.tgz

    $ gunzip < tpcc1000.backup.tgz | mysql

现在这种恢复都是比较简单的，但是在线上的话可能还会遇到一些出错的情况

mysqldump 虽然不错但是有一个很大的缺陷，那就是它是单线程备份，恢复速度相对来说就会比较慢，如果备份了一个 100G 的数据库，但是只想恢复其中的一张表就必须恢复这整个 100G 的库

## mydumper 工具 (包含myloader) 推荐使用

https://github.com/mydumper/mydumper

安装：

    $ yum install https://github.com/mydumper/mydumper/releases/download/v0.12.3-2/mydumper-0.12.3-2.stream8.x86_64.rpm

    $ mydumper --version
    mydumper 0.12.3-2, built against MySQL 5.7.37-40

这个工具的备份的时候它是并行的，另外一点重要的是它的并行是基于行的，也就是说即使是基于一张表也能并行备份

这个工具的恢复也是基于并行的，并且恢复的时候可以只恢复指定的表

这看上去才是一个完美的逻辑备份的解决方案，小小的缺点就是这是一个第三方的工具

一些常用参数说明：

* -G, --triggers          备份触发器
* -E, --event             备份定时器
* -R, --routines          备份存储过程
* --trx-consistency-only  事务一致性，在一个事务中实现
* -t, --threads           要开多少个线程进行备份，默认值是 4
* -o, --outputdir         备份到指定的文件夹目录下
* -x, --regex             通过正则匹配备份内容
* -c, --compress          压缩备份文件
* -r, --rows              基于每张表每个备份文件备份的行数，到了指定的行数就 split 出一个备份文件
* -F, --chunk-filesize    基于每张表每个备份文件备份的数据大小，到指定大小了就 split 出一个备份文件，单位MB
* -B, --database          备份指定的数据库
* -T, --tables-list       备份指定的表

备份命令示例：

    $ mydumper -G -E -R --trx-consistency-only -t 4 -c -B tpcc1000 -o backup_20210626

    ** (mydumper:391317): WARNING **: 22:14:46.482: Using trx_consistency_only, binlog coordinates will not be accurate if you are writing to non transactional tables.

    $ ll backup_20210626
    total 124M
    -rw-r--r-- 1 root root  136 Jun 26 22:14 metadata   <--自动记录当前二进制日志的位置，不需要再通过master-data参数去指定
        ($ cat backup_20210626/metadata
        Started dump at: 2021-06-26 22:14:46
        SHOW MASTER STATUS:
                Log: bin.000013
                Pos: 582329381
                GTID:

        Finished dump at: 2021-06-26 22:14:51)
    -rw-r--r-- 1 root root  23M Jun 26 22:14 tpcc1000.customer.00000.sql.gz   <--记录每一条的insert语句
    -rw-r--r-- 1 root root  468 Jun 26 22:14 tpcc1000.customer-schema.sql.gz  <--另外还有schema备份每张表的表结构
    -rw-r--r-- 1 root root 1.6K Jun 26 22:14 tpcc1000.district.00000.sql.gz
    -rw-r--r-- 1 root root  353 Jun 26 22:14 tpcc1000.district-schema.sql.gz
    -rw-r--r-- 1 root root 2.0M Jun 26 22:14 tpcc1000.history.00000.sql.gz
    -rw-r--r-- 1 root root  357 Jun 26 22:14 tpcc1000.history-schema.sql.gz
    -rw-r--r-- 1 root root 5.3M Jun 26 22:14 tpcc1000.item.00000.sql.gz
    -rw-r--r-- 1 root root  248 Jun 26 22:14 tpcc1000.item-schema.sql.gz
    -rw-r--r-- 1 root root  35K Jun 26 22:14 tpcc1000.new_orders.00000.sql.gz
    -rw-r--r-- 1 root root  280 Jun 26 22:14 tpcc1000.new_orders-schema.sql.gz
    -rw-r--r-- 1 root root  48M Jun 26 22:14 tpcc1000.order_line.00000.sql.gz
    -rw-r--r-- 1 root root  407 Jun 26 22:14 tpcc1000.order_line-schema.sql.gz
    -rw-r--r-- 1 root root 1.3M Jun 26 22:14 tpcc1000.orders.00000.sql.gz
    -rw-r--r-- 1 root root  343 Jun 26 22:14 tpcc1000.orders-schema.sql.gz
    -rw-r--r-- 1 root root  109 Jun 26 22:14 tpcc1000-schema-create.sql.gz
    -rw-r--r-- 1 root root  45M Jun 26 22:14 tpcc1000.stock.00000.sql.gz
    -rw-r--r-- 1 root root  382 Jun 26 22:14 tpcc1000.stock-schema.sql.gz
    -rw-r--r-- 1 root root  297 Jun 26 22:14 tpcc1000.warehouse.00000.sql.gz
    -rw-r--r-- 1 root root  285 Jun 26 22:14 tpcc1000.warehouse-schema.sql.gz

## myloader 推荐使用

一些常用参数说明：

* -d, --directory     需要恢复的备份数据源目录
* -t, --threads       使用的线程数，默认值是4
* -B, --database      可以恢复到另一个数据库名字的库里

命令示例：

    $ myloader -d backup_20210626 -t 4

    $ myloader -d backup_20210626 -t 4 -B tpcc2000

这两个工具能做到并行和单表复制的原理就是 ftwrl 后同时开多个事务去进行备份，每个备份线程都加上 ftwrl 锁是为了个保证每个线程读到的数据都是一直，因为 ftwrl 这个锁会把当前整个数据库实例锁成只读

首先有一个主线程去执行一个 flush tables with read lock，然后其他的线程切换到 RR 的事务隔离级别并且 start transaction with consistent snapshot，这个时候主线程就可以 unload tables，接着其他的线程去做备份相关的操作就可以了

|     | 主线程                       | 其他线程                                    | 其他线程                                    |
| --- | ---------------------------- | ------------------------------------------- | ------------------------------------------- |
|     | flush tables with read lock; |                                             |                                             |
|     |                              | start transaction with consistent snapshot; | start transaction with consistent snapshot; |
|     | unlock tables;               |                                             |                                             |
|     |                              | 备份操作                                    | 备份操作                                    |

如果对一张表也需要做并行备份，可以去检查每张表是否有唯一索引，有唯一索引的话就去进行分片


## mysqlpump 5.7版本以上支持的官方工具

在 MySQL 5.7 以上版本用来取代 msyqldump 的工具

这个工具的命令行参数和 msyqldump 基本上是一样的

这个工具也是单线程的，也没有类似 master-data 的参数，所以这个工具没啥实用性

## 物理备份工具

以上介绍的都是一些逻辑备份工具，MySQL 还有一些物理备份工具可以使用

## percona-xtrabackup

https://github.com/percona/percona-xtrabackup

xtrabackup 现在分了 2 个大版本，一个是 8.0.xx 支持支 MySQL8.0+ ，一个是 2.4.xx 支持 MySQL5.6+ 和 MySQL5.7+

2.4.xx 下载页面：https://www.percona.com/downloads/Percona-XtraBackup-2.4/LATEST/

安装：

    $ wget https://downloads.percona.com/downloads/Percona-XtraBackup-2.4/Percona-XtraBackup-2.4.26/binary/tarball/percona-xtrabackup-2.4.26-Linux-x86_64.glibc2.12.tar.gz

    $ tar xvf percona-xtrabackup-2.4.26-Linux-x86_64.glibc2.12.tar.gz

    $ mv percona-xtrabackup-2.4.26-Linux-x86_64.glibc2.12 /usr/local

添加到环境变量，.zshrc：

    PATH=/usr/local/percona-xtrabackup-2.4.26-Linux-x86_64.glibc2.12/bin:$PATH:$HOME/bin

    $ zsh

### 备份

    $ innobackupex --compress --compress-threads=8 --stream=xbstream --user=root --parallel=4 ./ > backup.xbstream

    # --compress            压缩备份文件
    # --compress-threads=8  压缩使用到的线程是8
    # -stream=xbstream      使用xbstream的流方式
    # --parallel            备份使用的线程数是4

xtrabackup 可以备份指定的库和指定的表，但是不是很推荐这种方式的，因为 xtrabackup 的备份原理是备份所有的表空间，如果不备份所有的完整的表空间的话，以后可能会遇到各种问题，比如只备份了 sbstest 这个数据库没有备份 tpcc1000 这个库，那么如果以后要去创建 tpcc1000 库下面的某张表的时候，如果这张表已经存在和原来的表有冲突的话就不能建，还有各种莫名其妙的问题

所以建议使用 xtrabackup 工具的时候备份完整的数据库

xtrabackup 的备份是非常简单和直接的，就是备份 idb 表空间文件，另外在备份的过程中会频繁的打印一些 log scanned up to (xxxxxxxx)，这些信息的意思是有一个单独的线程会一直不断的去备份重做日志，然后另外其他的线程去备份表空间文件 Compressing and streaming，最终就达到了一个物理备份的效果

另外 xtrabackup 备份的时候一定要看最后一行，只有最后一行出现 completed OK! 才表示备份成功了，否则这次备份就是失败的

备份后的数据大小对比：

    $ du -sh backup.xbstream
    1.6G    backup.xbstream

    $ du -sh /mdata/mysql_test_data
    4.5G    /mdata/mysql_test_data

xtrabackup 还有一个参数 --throttle=# 也是比较有用的，表示的是备份的时候用到的 IOPS，这个参数在云环境下面的时候或许会比较有用，因为可以根据服务器配置来限制备份速度，可以防止备份的时候对业务影响过大

### 恢复 

对 xtrabackup 备份出来的数据进行恢复是一个稍显麻烦的事情，需要先停止数据库：

    $ /etc/init.d/mysql.server stop
    Shutting down MySQL.. SUCCESS!

    $ mv /mdata/mysql_test_data /mdata/mysql_test_data.old

    $ cd /mdata

    $ mkdir backup

    $ mv ~/backup.xbstream backup

    $ cd backup

    $ xbstream -x < backup.xbstream   # 从备份文件里解压出个各个文件

    $ ll
    total 1.6G
    -rw-r----- 1 root root  474 Jun 27 10:14 backup-my.cnf.qp
    -rw-r--r-- 1 root root 1.6G Jun 27 09:58 backup.xbstream
    drwxr-x--- 2 root root 4.0K Jun 27 10:14 bbb
    drwxr-x--- 2 root root 4.0K Jun 27 10:14 dbt3
    drwxr-x--- 2 root root 4.0K Jun 27 10:14 employees
    -rw-r----- 1 root root  675 Jun 27 10:14 ib_buffer_pool.qp
    -rw-r----- 1 root root  31M Jun 27 10:14 ibdata1.qp
    drwxr-x--- 2 root root 4.0K Jun 27 10:14 inaction
    drwxr-x--- 2 root root 4.0K Jun 27 10:14 mysql
    drwxr-x--- 2 root root 4.0K Jun 27 10:14 performance_schema
    drwxr-x--- 2 root root  12K Jun 27 10:14 sys
    drwxr-x--- 2 root root 4.0K Jun 27 10:14 test
    drwxr-x--- 2 root root 4.0K Jun 27 10:14 tpcc1000
    -rw-r----- 1 root root  102 Jun 27 10:14 xtrabackup_binlog_info.qp
    -rw-r----- 1 root root  143 Jun 27 10:14 xtrabackup_checkpoints
    -rw-r----- 1 root root  492 Jun 27 10:14 xtrabackup_info.qp
    -rw-r----- 1 root root  552 Jun 27 10:14 xtrabackup_logfile.qp

可以看到解压完后还有一些 qp 结尾的文件，这些文件也是被压缩过的文件没有解压，所以 xbstream 命令只是用来解压 stream 文件成很多相应的表空间文件，这时候需要另外一个命令解压：

    $ yum install https://repo.percona.com/yum/percona-release-latest.noarch.rpm -y

    $ yum install qpress -y

    $ for f in `find ./ -iname "*\.qp"`; do qpress -dT4 $f $(dirname $f) && rm -f $f; done
    # for f in `find ./ -iname "*\.qp"` 查找当前目录下所有以qb结尾的文件
    # do qpress -dT4 $f $(dirname $f)   通过qpress命令解压，T4表示用4个线程进行解压
    # rm -f $f; done                    删除当前目录下的qp文件

    $ ll
    total 1.7G
    -rw-r--r-- 1 root root  488 Jun 27 10:41 backup-my.cnf    <--还帮你备份了配置文件的一部分
    -rw-r--r-- 1 root root 1.6G Jun 27 09:58 backup.xbstream
    drwxr-x--- 2 root root 4.0K Jun 27 10:41 bbb
    drwxr-x--- 2 root root 4.0K Jun 27 10:41 dbt3
    drwxr-x--- 2 root root 4.0K Jun 27 10:41 employees
    -rw-r--r-- 1 root root  907 Jun 27 10:41 ib_buffer_pool
    -rw-r--r-- 1 root root 140M Jun 27 10:41 ibdata1
    drwxr-x--- 2 root root 4.0K Jun 27 10:41 inaction
    drwxr-x--- 2 root root 4.0K Jun 27 10:41 mysql
    drwxr-x--- 2 root root  12K Jun 27 10:41 performance_schema
    drwxr-x--- 2 root root  12K Jun 27 10:41 sys
    drwxr-x--- 2 root root 4.0K Jun 27 10:41 test
    drwxr-x--- 2 root root 4.0K Jun 27 10:41 tpcc1000
    -rw-r--r-- 1 root root   15 Jun 27 10:41 xtrabackup_binlog_info   <--自动记录下了二进制日志的位置
    -rw-r----- 1 root root  143 Jun 27 10:14 xtrabackup_checkpoints
    -rw-r--r-- 1 root root  519 Jun 27 10:41 xtrabackup_info
    -rw-r--r-- 1 root root 2.5K Jun 27 10:41 xtrabackup_logfile

恢复：

    $ innobackupex --apply-log ./
    # 将当前的日志应用到备份文件，产生新的重做日志

看到 completed OK! 就表示恢复完成了，并且目录下也产生了 ib_logfile0 和 ib_logfile1 文件

最后重命名 backup 目录为数据目录然后启动数据库就可以了：

    $ cd ..

    $ mv backup mysql_test_data

    $ chown -R mysql:mysql mysql_test_data    # 记得改数据目录权限

    $ /etc/init.d/mysql.server start
    Starting MySQL.Logging to '/mdata/mysql_test_data/error.log'.
    . SUCCESS!

### 原理

1. 首先备份表空间 ibd 文件
2. 开启线程备份重做日志，
3. 当所有的表空间都备份完成之后，通过 flush tables with read lock 把整个数据库锁成只读的
4. show master status，把当前二进制日志的位置保存下来
5. 最后 unlock tables

xtrabackup 备份的是备份结束时间点的数据，备份和恢复的速度都很快

xtrabackup 做增量备份性能比较差，不推荐使用