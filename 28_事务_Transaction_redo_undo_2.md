# Redo

## Redo 日志的分类

* 物理日志：记录整个页的变化（diff）
* 逻辑日志：Like SQL语句
* 物理逻辑日志：根据页进行记录，内容逻辑

每种不同的重做日志类型，可能有的重做日志内在的格式是不一样的

## redo 和 binlog 的区别

* InnoDB vs MySQL
* 物理逻辑日志 vs 逻辑日志
* 写入时间点

binlog 默认是关闭的，这个参数是一定要开的，因为 MySQL 做复制是基于这个日志来做的

    (root@localhost) [(none)]> show variables like 'log_bin';
    +---------------+-------+
    | Variable_name | Value |
    +---------------+-------+
    | log_bin       | OFF   |
    +---------------+-------+
    1 row in set (0.01 sec)

my.cnf 中设置默认开启:

    # log
    ...
    general_log_file = general.log
    log_bin=bin   # new

然后这就意味着 MySQL 事务的一次提交要写入两个日志，一个是重做日志一个是二进制日志，重做日志是偏物理逻辑日志（space，page），binlog 是逻辑日志（相当于是SQL语句）

binlog 写入的顺序就是一个事务提交的顺序

    | T1 | T4 | T3 | T2 | T8 | T6 | T7 | T5 |

redo log 写入的顺序并不是按照事务提交顺序来的，标星的表示最后提交的顺序，这里也就是 *T2 -> *T3 -> *T1，产生这种情况的原因可以举个例子，比如一条 update 语句可能更新了很多页相关的数据，那么就需要并发的去写这些页，所以页可能有很多的重做日志需要写

    | T1 | T2 | T1 | *T2 | T3 | T1 | *T3 | *T1 |

在重做日志里面一个事务的日志可以有很多，但是二进制日志却只有一个

两个日志写入的时间点是不一样的，重做日志在事务提交的过程中就开始写了，而二进制日志在事务提交的最后那个时间才开始写

MySQL 内部通过分布式事务保证 redolog 的写入和 binlog 的写入是原子的，MySQL 在最后进行 commit 的过程中共分了三步去处理：

1. 先在 InnoDB 引擎层面去写一个 prepare redo log(->fsync)
2. 然后再去写 binlog(->fsync)
3. 确保都刷新到磁盘后再去 InnoDB 的 commit redo log 这次是可以不需要 fsync 的

假设 1 成功了，2 失败了，那么 3 也就失败了，如果这个时候宕机了由于二进制日志还没写，就需要 rollback

如果 1 成功了，2 也成功了，3 失败了，这个事务也做 commit

如果 3 都成功，当然也就是 commit

恢复的时候先去扫描 binglog，然后从 binlog 的事务 ID 列表把所有的事务 ID 拿出来建立一张 hash table，然后再去三年内 InnoDb 重做日志中 checkpoint 到后面的日志，这时候也会产生事务 ID 的列表，然后再去搜索前面的 hash table，如果这个事务 ID 在 hash table 中，则表示 1 和 2 都是成功的就会选择去 commit，如果不在里面的话就会选择去 rollback

## redo log file

* 与redo log buffer中的内容一致
* 追加写入方式（append write）
  * 全顺序？
* 循环使用（round-robin）

    Redo Log File1
    | Log File Header | CP1 |      | CP2 | Log Block | 每个512字节 | Log Block | Log Block | ... | Log Block |

    Redo Log File2
    |      |      |      |      | Log Block | Log Block | Log Block | ... | Log Block |

* LSN
  * log sequenct number
  * 重做日志写入的字节量
* LSN存在于：
  * page
  * redo log block 
  * checkpoint

# UNDO 

* InnoDB undo 对象
  * rollback segment
  * undo log segment
  * undo page
  * undo log 
  * undo log record

undo 日志记录的内容是逻辑的是每一条记录的日志，而 redo log 记录的内容是物理的是基于每个页的修改

所以 rollback 是一个逻辑过程而不是一个物理过程

* undo log
  * undo log header 
  * undo log records
    * insert undo log record 
    * update undo log record 
    * 逻辑记录

            UNDO LOG
            | UNDO LOG HEADER | UNDO LOG RECORD | UNDO LOG RECORD | ... |

insert undo log record 和 update undo log record 的处理方式是不一样的而且也是分开存放在不同的段中的，有一个非常重要的概念叫 undo 的回收，对于 insert undo 来说只要插入了就可以被回滚，而 update undo 即使事务提交了也不能马上被回收

如果一个事务提交之后，其他的线程还在引用它的 undo，undo 就不能马上被回收，undo 真正的回收是在一个 purge 线程中

* purge
  * 真正删除记录
  * 删除undo log 

使用 DLETE 语句进行删除的时候，在页上面不是马上真正被删掉了，即使一个事务提交了这条记录也只是标记为删除而没有被真正的删除，最后时通过 purge 线程进行删除的

purge 线程有一个参数，SSD 磁盘可以设置得稍微大点，4 或者 8：

    (root@localhost) [(none)]> show variables like 'innodb%purge%';
    +--------------------------------------+-------+
    | Variable_name                        | Value |
    +--------------------------------------+-------+
    | innodb_purge_threads                 | 4     |
    +--------------------------------------+-------+
    5 rows in set (0.03 sec)

* purge效率不高
  * 系统空间不断增大
  * 存在长事务
    * 将长事务拆成小事务
  * **索引没有添加**
    * 检查slow log

# 分布式事务

* MySQL XA事务并不完美
  * client退出导致PREPARE成功事务丢失
  * MySQL Server宕机导致binlog丢失
  * **外部XA prepare成功不写日志**

用法：

    (root@localhost) [test]> xa start 'a';
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> insert into z values (2000);
    Query OK, 1 row affected (0.01 sec)

    (root@localhost) [test]> insert into z values (3000);
    Query OK, 1 row affected (0.00 sec)

    (root@localhost) [test]> xa end 'a';
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> xa prepare 'a';
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> xa recover;
    +----------+--------------+--------------+------+
    | formatID | gtrid_length | bqual_length | data |
    +----------+--------------+--------------+------+
    |        1 |            1 |            0 | a    |
    +----------+--------------+--------------+------+
    1 row in set (0.00 sec)

    (root@localhost) [test]> xa rollback 'a';
    Query OK, 0 rows affected (0.01 sec)

    或者提交
    (root@localhost) [test]> xa commit 'a';
    Query OK, 0 rows affected (0.01 sec)

不过这是在单实例的 MySQL 命令行中执行意义不大

在应用中使用的伪代码差不多是这样：

    conn1 = "数据库连接1地址"
    conn2 = "数据库连接2地址"

    ds1 = getDataSource(connAddress1, "账号", "密码")
    ds2 = getDataSource(connAddress2, "账号", "密码")

    xaRes 1 = ds1.getXAConnection();
    stmt1 = conn1.createStatment();
    xaRes 2 = ds2.getXAConnection();
    stmt2 = conn2.createStatment();

    xaRes1.start(...)
    stmt1.execute("
        update account set money = money-10000 where user='limei'"
    )
    xa.xaRes1.end(...)

    xaRes2.start(...)
    stmt2.execute("
        update account set money = money+10000 where user='lucy'"
    )
    xa.xaRes2.end(...)

    int ret2 = xaRes2.prepare(...)
    int ret1 = xaRes1.prepare(...)

    if(ret1 == XAResource.XA_OK && ret2 == XAResource.XA_OK){
      xaRes1.commit(...)
      xaRes2.commit(...)
    }

MySQL 分布式事务的性能是非常差的，不建议使用

# 事务编程

## 不好的事务提交习惯

* 在循环中提交
* 使用自动提交
* 使用自动回滚


## 在循环中提交

    CREATE PROCEDURE load1 (count int unsigned)
    BEGIN
      DECLARE s INT UNSIGNED DEFAULT 1;
      DECLARE c CHAR(80) DEFAULT REPEAT('a',80);
        WHILE s <= count DO
          INSERT INTO t1 SELECT NULL,c;
          SET s = s + 1;
        END WHILE;
    END;

如果调用这个存储过程 1000 次 call load1(1000); 由于 MySQL 是 autocommit 的，所以这里 fsync 就执行了 1000 次

而且这样直接调用还有一个问题，就是万一失败了，想回滚都回滚不了，因为前面的都是即时提交的做不到一个原子性

正确的调用方法应该是这样的：

    begin;
    call load1(1000);
    commit;

## 长事务

拆成小事务

对于利息统计这种需求千万不要用一个大事务去做，这对 MySQL 的影响是非常大的

    UPDATE account SET account_total = account_total + (1 + interest_rate)

拆成小事务批量的去执行，因为写 binlog 成本很大主从延时也会增大，同时也避免产生较大的 undo

# 锁的概念

next