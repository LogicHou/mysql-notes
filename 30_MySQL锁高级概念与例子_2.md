## 自增锁

* 一个表一个自增列
* AUTO_INCREMENT
* SELECT MAX(auto_inc_col) FROM t FOR UPDATE; 自增回溯问题根源
* 在事务提交前释放
  * 其他锁在事务提交时才释放
* 思考
  * INSERT...SELECT...

### InnoDB自增列必须被定义为一个KEY，并且是键值的开始部分
  
以下这两种方法定义都会报错：

    (root@localhost) [test]> create table ai ( a int auto_increment, b int, key(b, a));
    ERROR 1075 (42000): Incorrect table definition; there can be only one auto column and it must be defined as a key
    (root@localhost) [test]> create table ai ( a int auto_increment, b int);
    ERROR 1075 (42000): Incorrect table definition; there can be only one auto column and it must be defined as a key

MySQL5.7 的自增没有持久化，MySQL 每次启动的时候都会执行 SELECT MAX(auto_inc_col) FROM t FOR UPDATE; 来重新计算最大的自增值，如果没有设置自增值就相当于会全表扫描了，就算是定义成包含自增列的复合索引，SELECT MAX(auto_inc_col) 又要去扫整个索引，所以 InnoDB 有这样的一个限制避免重启的时候扫描效率太低

### 事务提交前释放

这个特性主要是为了保证自增值的连续性

    (root@localhost) [test]> begin;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> insert into ai(a,b) select null,id from sbtest.sbtest1 limit 1000;  <--自增所在这条语句执行完就释放了了，不用等commit后再释放，然如果插入的过程是一个大事务的话依然会阻塞


### innodb_autoinc_lock_mode 参数 

    (root@localhost) [test]> show variables like 'innodb%auto%';
    +-----------------------------+-------+
    | Variable_name               | Value |
    +-----------------------------+-------+
    | innodb_autoextend_increment | 64    |
    | innodb_autoinc_lock_mode    | 1     |
    | innodb_stats_auto_recalc    | ON    |
    +-----------------------------+-------+
    3 rows in set (0.00 sec)

默认值 1 的话则是 select 语句执行完之后再释放，就会导致一定程度的阻塞

innodb_autoinc_lock_mode 将这个参数改成 2，可以将行为调整为每自增一次就释放一次，这样就可以并发的进行插入，自增所的粒度就会变得非常小，但是它的一个缺点是连续性不能保证

还有在 2 的情况下，binlog 格式必须设置为 row

这个值推荐设置为 2，因为一般业务场景中满足单调递增就行了，虽然可能不连续，但是依旧是自增唯一的

myc.cnf：

    # innodb
    ...
    innodb_lock_wait_timeout = 3
    innodb_autoinc_lock_mode = 2 # new 

## InnoDB 锁的算法

* Record Lock 
  * 单个行记录上的锁
* Gap Lock 
  * 锁定一个范围，但不包含记录本身
* Next-key Lock 
  * Gap Lock + Record Lock，锁定一个范围，并且锁定记录本身
* **锁住的是索引**

### 主键加锁状态分析

假设现在对 test.l 表中的 a(主键聚集索引) = 2 的记录进行了加锁：

    (root@localhost) [test]> begin;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> select * from l where a = 2 for update;
    +---+------+------+------+
    | a | b    | c    | d    |
    +---+------+------+------+
    | 2 |    4 |    6 |    8 |
    +---+------+------+------+
    1 row in set (0.00 sec)

    (root@localhost) [test]> select * from l;
    +---+------+------+------+
    | a | b    | c    | d    |
    +---+------+------+------+
    | 2 |    4 |    6 |    8 |
    | 4 |    6 |    8 |   10 |
    | 6 |    8 |   10 |   12 |
    | 8 |   10 |   12 |   14 |
    +---+------+------+------+
    4 rows in set (0.00 sec)

如果对记录 a 进行了了加锁，如果加的是 record lock 的话，就表示对记录 2 本身加了一个 X 锁，这就是大家通常意义上理解的所谓的记录锁

gap lock 表示的是如果在 2 上面加了一把 gap 锁，表示对负无穷到 2 加了一把 gap 锁，锁的模式是X的 (-x,2)X

这时候 gap lock 没有对  2 这条记录加锁，如果现在有另外一个线程去执行了 a=2 for update 更新也好删除也好这时候是可以执行的

举个例子如果线程 1 持有并 hold 了 2 这个排它锁，但是他是 gap 的话，那么线程 2 如果希望 hold 记录等于 2 X锁然后是一个 record lock 的话，这两者是兼容的

如果说这时候有一个线程 3，insert 1 这条记录就会发现因为 1 在负无穷到 2 的范围里面，这个时候就会发生等待

next key lock 锁则会锁住负无穷和记录本身 (-x,2]X，record lock 和 gap lock 是兼容的，next key lock 则和前面两者都不兼容

RR 事务隔离级别表示的是所有的对某一条记录进行加锁，用的锁的算法都是 next key lock，用 RR 锁存在的一个问题是 insert 性能可能会有问题

RC 事务隔离级别的锁都是 record lock

### 普通索引加锁状态分析

TODO