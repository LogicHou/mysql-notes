## 例子 1

    (root@localhost) [test]> set tx_isolation='repeatable-read';
    Query OK, 0 rows affected, 1 warning (0.00 sec)

    (root@localhost) [test]> select @@tx_isolation;
    +-----------------+
    | @@tx_isolation  |
    +-----------------+
    | REPEATABLE-READ |
    +-----------------+
    1 row in set, 1 warning (0.00 sec)
    
    (root@localhost) [test]> select * from l;
    +---+------+------+------+
    | a | b    | c    | d    |
    +---+------+------+------+
    | 2 |    4 |    6 |    8 |
    | 4 |    6 |    8 |   10 |
    | 6 |    8 |   10 |   12 |
    | 8 |   10 |   12 |   14 |
    +---+------+------+------+
    5 rows in set (0.00 sec)

| 时间 | 会话A                                   | 会话B                                            |
| ---- | --------------------------------------- | ------------------------------------------------ |
| 1    | begin;                                  |                                                  |
| 2    | select * from l where b = 6 for update; |                                                  |
| 3    |                                         | insert into l values(3,4,100,200);<br># 阻塞住了 |
| 4    |                                         | insert into l values(1,4,200,400);<br># 可以插入 |
| 5    | rollback;                               |                                                  |
| 6    |                                         | rollback;                                        |

这个例子时间 3 中被阻塞住是因为 4 位于被锁住的 (4,6] 内，所以无法插入

那么时间 4 为什么又可以插入了呢，是因为时间 2 中真正被锁住的范围是 ((4,2<-主键), (6,4<-主键)]，其中包含主键值，而插入的 values(1,4,200,400) 转换一下其实就是插入 (4,1) 在 (4,2) 之前所以是可以插入的

## 例子 2

还是 RR 的事务隔离级别下，如果锁住的是一条不存在的记录

    (root@localhost) [test]> select * from l where b = 12 for update;
    Empty set (0.00 sec)

这时在 show engine innodb status\G 可以看到这样的记录

    RECORD LOCKS space id 185 page no 5 n bits 72 index b of table `test`.`l` trx id 680718 lock_mode X
    Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
    0: len 8; hex 73757072656d756d; asc supremum;;

可以看到这里是将锁加到了 heap no 1 上，而 heap no 1 表示记录里的 max 伪列，所以这里的加锁范围是 (10,正无穷)，表示插入任何大于 10 的记录都是不允许的

## 例子 3

还是 RR 的事务隔离级别下，在会话 A 中锁住 7，在会话 B 中锁住 8 的情况

会话A：

    (root@localhost) [test]> begin;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> select * from l where b = 7 for update;
    Empty set (0.00 sec)

锁住了 (6,8) 不包含 8 的范围，这里是一把 gap 锁：

    TABLE LOCK table `test`.`l` trx id 680719 lock mode IX
    RECORD LOCKS space id 185 page no 5 n bits 72 index b of table `test`.`l` trx id 680719 lock_mode X locks gap before rec
    Record lock, heap no 4 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
    0: len 4; hex 80000008; asc     ;;
    1: len 4; hex 80000006; asc     ;;

会话B：

    (root@localhost) [test]> begin;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> select * from l where b = 8 for update;
    Empty set (0.00 sec)

会锁住 (6,8] 包含 8 这条记录，注意这里这不是 gap 锁了，而是 next key lock 锁：

    MySQL thread id 3, OS thread handle 140599303280384, query id 125 localhost root
    TABLE LOCK table `test`.`l` trx id 680720 lock mode IX
    RECORD LOCKS space id 185 page no 5 n bits 72 index b of table `test`.`l` trx id 680720 lock_mode X <--RR 模式下像这种没有额外提示内容的，只有类似 lock_mode X 的就表示是 next key lock 锁
    Record lock, heap no 4 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
    0: len 4; hex 80000008; asc     ;;
    1: len 4; hex 80000006; asc     ;;

会发现在 8 这条记录上面是加了 2 把锁，一个是 where b = 7 产生的 gap 锁，一个是 where b = 8 产生的 next key lock 锁

所以这两把锁是互相兼容，这样的设计很巧妙

## 例子 4

现在换成 RC 事务隔离接再重试下例子 2 中 where b = 12 的条件：

    (root@localhost) [test]> set tx_isolation = 'read-committed';
    Query OK, 0 rows affected, 1 warning (0.01 sec)

    (root@localhost) [test]> begin;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> select * from l where b = 12 for update;
    Empty set (0.00 sec)

会发现没有锁，因为 RC 事务隔离接不需要去避免幻读的问题，所以自然也就没有锁


## 例子 5

    (root@localhost) [test]> select @@tx_isolation;
    +----------------+
    | @@tx_isolation |
    +----------------+
    | READ-COMMITTED |
    +----------------+
    1 row in set, 1 warning (0.00 sec)

    (root@localhost) [test]> select * from l where d = 10 for update;
    +---+------+------+------+
    | a | b    | c    | d    |
    +---+------+------+------+
    | 4 |    6 |    8 |   10 |
    +---+------+------+------+
    1 row in set (0.00 sec)

RC 事务隔离级别下，对没有索引的列进行加锁，只会对 10 对应的主键 4 进行加锁，锁的算法是 rec(record) but not gap 就是一个记录锁

    TABLE LOCK table `test`.`l` trx id 680722 lock mode IX
    RECORD LOCKS space id 185 page no 3 n bits 72 index PRIMARY of table `test`.`l` trx id 680722 lock_mode X locks rec but not gap

## 例子 6

将例子 5 改成 RR 隔离级别的效果

    (root@localhost) [test]> set tx_isolation = 'repeatable-read';
    Query OK, 0 rows affected, 1 warning (0.00 sec)

    (root@localhost) [test]> begin;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> select * from l where d = 10 for update;
    +---+------+------+------+
    | a | b    | c    | d    |
    +---+------+------+------+
    | 4 |    6 |    8 |   10 |
    +---+------+------+------+
    1 row in set (0.00 sec)

这时候会在主键索引上锁住 (负无穷,2]，(2,4]，(4,6]，(6,8]，(8,正无穷)，变现上虽然看起来像表锁，但是这里还是记录锁

    TABLE LOCK table `test`.`l` trx id 680723 lock mode IX
    RECORD LOCKS space id 185 page no 3 n bits 72 index PRIMARY of table `test`.`l` trx id 680723 lock_mode X
    Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
    0: len 8; hex 73757072656d756d; asc supremum;; ；<--注意看 heap no 1，所以这里是锁住(...,正无穷)

    Record lock, heap no 2 PHYSICAL RECORD: n_fields 6; compact format; info bits 0 <--(2,4]
    0: len 4; hex 80000002; asc     ;;
    1: len 6; hex 0000000a5b26; asc     [&;;
    2: len 7; hex bf000010020110; asc        ;;
    3: len 4; hex 80000004; asc     ;;
    4: len 4; hex 80000006; asc     ;;
    5: len 4; hex 80000008; asc     ;;

    Record lock, heap no 3 PHYSICAL RECORD: n_fields 6; compact format; info bits 0 <--(4,6]
    0: len 4; hex 80000004; asc     ;;
    1: len 6; hex 0000000a5b27; asc     [';;
    2: len 7; hex c000000fc00110; asc        ;;
    3: len 4; hex 80000006; asc     ;;
    4: len 4; hex 80000008; asc     ;;
    5: len 4; hex 8000000a; asc     ;;

    Record lock, heap no 4 PHYSICAL RECORD: n_fields 6; compact format; info bits 0 <--(6,8]
    0: len 4; hex 80000006; asc     ;;
    1: len 6; hex 0000000a5b2c; asc     [,;;
    2: len 7; hex a300000fc40110; asc        ;;
    3: len 4; hex 80000008; asc     ;;
    4: len 4; hex 8000000a; asc     ;;
    5: len 4; hex 8000000c; asc     ;;

    Record lock, heap no 5 PHYSICAL RECORD: n_fields 6; compact format; info bits 0 <--(8,...)
    0: len 4; hex 80000008; asc     ;;
    1: len 6; hex 0000000a5b2d; asc     [-;;
    2: len 7; hex a400000fc50110; asc        ;;
    3: len 4; hex 8000000a; asc     ;;
    4: len 4; hex 8000000c; asc     ;;
    5: len 4; hex 8000000e; asc     ;;

这里要锁住所有记录(负无穷到正无穷)的原因是 d 列上没有索引，所以后面插入的任何一条记录都可以是 10，所以说任何一条记录都需要被锁住

这样锁的代价可能会很大，假如这张表有 100 万行记录，由于没有锁升级就会把扫到的记录都加锁，就会产生 100 万的 next key lock

## 例子 7 

幻读可能会带来的问题

假设有张表的记录为 a: 1 3 4 7，执行下表中的操作

| 时间 | 会话A        | 会话B     |
| ---- | ------------ | --------- |
| 1    | begin;       |           |
| 2    | delete <= 7; |           |
| 3    |              | begin;    |
| 4    |              | insert 2; |
| 5    |              | commit;   |
| 6    | commit;      |           |

这种情况在 RC 事务隔离情况下是允许的，delete <= 7 只会加一个记录锁不会加范围锁，所以会话 B 的 insert 是允许的，此时剩下的记录就是 2 这一条记录

此时如果 binlog_format 的设置不是 row 的话，日志里的记录是：

    log：
        insert 2     <--事务中先提交的语句先记录到日志
        delete <=7

这时候数据库发生了同步，则会先在 slave 库上插入 2 这条记录，然后再执行 delete <=7，这样一来在 slave 库上会连同 2 这条记录也一起删除了，这显然是有问题的

将 binlog_format 设置成为 row 后，日志里记录的删除是每一行的记录：

    log：
        insert 2
        delete 1     <--这就是binlog_format设置成row的意义所在
        delete 3
        delete 5
        delete 7

需要注意的是 binlog_format = row 这个设置 MySQL5.1 才开始支持

## 如何知道事务中的记录是否可见

每一条记录都有一个事务 id txid，它是全局自增的，存放在共享表中的某个位置，没开启一个事务的时候都会分配一个事务 id

当前事务会有一个活跃事务列表，其中记录了当前事务正在执行的没有提交的，事务提交了就不在这个列表里面了

开启一个事务后，只要执行了一条语句就会产生 read_view 对象，现在在 MySQL 中完全看不到这个东西的具体信息，只能通过 show engine innodb status\G 命令看到有多少哥 read_view 对象

read_view 表示的是开启事务时它的活跃列表中的事务是什么样的，它会把活跃事务列表去拷贝一份下来，就会拿到很多个事务 id，只要记录对应的事务 id 是在这个列表里面的，这条记录就是不可见的，因为这条记录在开启的时候它还没有提交

RR 事务隔离级别中 read_view 对象只产生一次

RC 事务隔离级别中 read_view 对象每执行一条 SQL 语句就创建 read_view 对象

不管 RR 还是 RC read_view 都是在第一次 select 执行的时候创建的

## 插入意向锁 insertion intention lock 

* gap lock 
* 提高并发插入性能
* intention_insert_lock

### 例子 1

    (root@localhost) [test]> set tx_isolation='read-committed';
    Query OK, 0 rows affected, 1 warning (0.00 sec)

    (root@localhost) [test]> select * from l;
    +---+------+------+------+
    | a | b    | c    | d    |
    +---+------+------+------+
    | 2 |    4 |    6 |    8 |
    | 4 |    6 |    8 |   10 |
    | 6 |    8 |   10 |   12 |
    | 8 |   10 |   12 |   14 |
    +---+------+------+------+
    4 rows in set (0.01 sec)

会话 A：

    (root@localhost) [test]> begin;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> insert into l values(16,18,20,22);
    Query OK, 1 row affected (0.00 sec)

    # 这时候 status 中只能看到有一把意向锁
    1 lock struct(s), heap size 1136, 2 row lock(s)
    MySQL thread id 4, OS thread handle 140287396792064, query id 23 localhost root
    TABLE LOCK table `test`.`l` trx id 681221 lock mode IX

会话 B：

    (root@localhost) [test]> begin;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> select * from l where a = 16 for update;
    | # 被阻塞

    # status 中显示在等待16的那把锁
    RECORD LOCKS space id 185 page no 3 n bits 72 index PRIMARY of table `test`.`l` trx id 681223 lock_mode X locks rec but not gap
    Record lock, heap no 6 PHYSICAL RECORD: n_fields 6; compact format; info bits 0
    0: len 4; hex 80000010; asc     ;;
    1: len 6; hex 0000000a6507; asc     e ;;
    2: len 7; hex a700000fc80110; asc        ;;
    3: len 4; hex 80000012; asc     ;;
    4: len 4; hex 80000014; asc     ;;
    5: len 4; hex 80000016; asc     ;;

产生这种情况的原因是 InnoDB 引擎做了优化，这个锁叫做隐式锁，insert into l values(16,18,20,22) 这条记录不需要加锁就知道上面有锁了，因为这条记录对应的事务还在事务活跃列表中，就代表上面肯定有锁

对于 insert 操作来说一开始是不创建锁对象，只有当发生等待的时候它才会转化成显示的锁，select * from l where a = 16 for update 的时候会查到 a = 16 这条记录已经在活跃事务列表中，说明没有提交这时候再去创建锁对象，也就是延迟去创建锁的对象，如果在延迟的过程中没有对这条记录加锁的话，就不用创建锁的对象了，这样就节省了内存

### 例子 2

    (root@localhost) [test]> insert into l values(10,12,14,16);
    (root@localhost) [test]> insert into l values(12,14,16,18);
    Query OK, 1 row affected (0.01 sec)

    (root@localhost) [test]> begin;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> select * from l where a <= 10 for update;
    +----+------+------+------+
    | a  | b    | c    | d    |
    +----+------+------+------+
    |  2 |    4 |    6 |    8 |
    |  4 |    6 |    8 |   10 |
    |  6 |    8 |   10 |   12 |
    |  8 |   10 |   12 |   14 |
    | 10 |   12 |   14 |   16 |
    +----+------+------+------+
    5 rows in set (0.00 sec)

会话 B：

    (root@localhost) [test]> set tx_isolation = 'repeatable-read';

    (root@localhost) [test]> begin;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> select * from l where a = 10 for update;
    Empty set (0.00 sec)

会话 A：

    (root@localhost) [test]> begin;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> insert into l values(9,11,13,15);
    | # 阻塞住了

    # 这时候可以在 status 中看到带 insert intention 关键字的锁信息
    ------- TRX HAS BEEN WAITING 11 SEC FOR THIS LOCK TO BE GRANTED:
    RECORD LOCKS space id 185 page no 3 n bits 72 index PRIMARY of table `test`.`l` trx id 681240 lock_mode X locks gap before rec insert intention waiting
    Record lock, heap no 6 PHYSICAL RECORD: n_fields 6; compact format; info bits 0
    0: len 4; hex 8000000a; asc     ;;
    1: len 6; hex 0000000a6516; asc     e ;;
    2: len 7; hex b100000ff60110; asc        ;;
    3: len 4; hex 8000000c; asc     ;;
    4: len 4; hex 8000000e; asc     ;;
    5: len 4; hex 80000010; asc     
    
insert intention 表示阻塞当前的这条记录，但是不阻塞 insert 操作

### 例子 3

会话 B：

    (root@localhost) [test]> insert into l values(20,22,24,26);
    Query OK, 1 row affected (0.00 sec)

    (root@localhost) [test]> select * from l;
    +----+------+------+------+
    | a  | b    | c    | d    |
    +----+------+------+------+
    |  2 |    4 |    6 |    8 |
    |  4 |    6 |    8 |   10 |
    |  6 |    8 |   10 |   12 |
    |  8 |   10 |   12 |   14 |
    | 10 |   12 |   14 |   16 |
    | 12 |   14 |   16 |   18 |
    | 20 |   22 |   24 |   26 |
    +----+------+------+------+
    7 rows in set (0.00 sec)

    (root@localhost) [test]> begin;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> select * from l where a <= 20 for update;
    +----+------+------+------+
    | a  | b    | c    | d    |
    +----+------+------+------+
    |  2 |    4 |    6 |    8 |
    |  4 |    6 |    8 |   10 |
    |  6 |    8 |   10 |   12 |
    |  8 |   10 |   12 |   14 |
    | 10 |   12 |   14 |   16 |
    | 12 |   14 |   16 |   18 |
    | 20 |   22 |   24 |   26 |
    +----+------+------+------+
    7 rows in set (0.00 sec)

会话 A：

    (root@localhost) [test]> begin;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> insert into l values(14,16,18,20);

    # status 中可以看到有插入意向锁
    RECORD LOCKS space id 185 page no 3 n bits 80 index PRIMARY of table `test`.`l` trx id 681250 lock_mode X locks gap before rec insert intention waiting
    Record lock, heap no 8 PHYSICAL RECORD: n_fields 6; compact format; info bits 0
    0: len 4; hex 80000014; asc     ;;
    1: len 6; hex 0000000a6520; asc     e ;;
    2: len 7; hex b900000fe00110; asc        ;;
    3: len 4; hex 80000016; asc     ;;
    4: len 4; hex 80000018; asc     ;;
    5: len 4; hex 8000001a; asc     ;;

会话 B：

    (root@localhost) [test]> commit;
    Query OK, 0 rows affected (0.00 sec)

会话 A：

    (root@localhost) [test]> insert into l values(14,16,18,20);
    Query OK, 1 row affected (4.21 sec)

会话B提交后，会话A中的这条记录马上就被插入了，这时候 status 上的锁信息是这样的：

    RECORD LOCKS space id 185 page no 3 n bits 80 index PRIMARY of table `test`.`l` trx id 681250 lock_mode X locks gap before rec insert intention <--此时对20这条记录有insert intention锁
    Record lock, heap no 8 PHYSICAL RECORD: n_fields 6; compact format; info bits 0
    0: len 4; hex 80000014; asc     ;;
    1: len 6; hex 0000000a6520; asc     e ;;
    2: len 7; hex b900000fe00110; asc        ;;
    3: len 4; hex 80000016; asc     ;;
    4: len 4; hex 80000018; asc     ;;
    5: len 4; hex 8000001a; asc     ;;

这个例子的流程是，会话 B 先在 20 上面加了一把 X 锁，锁住的是 (12,20] 的 next key lock 锁，这时候会话 A 要插入 14 这条记录，会对 20 这条记录加上一个 (12,20) 的 gap 锁，但是这个 gap 锁比较特殊，会有一个叫 inset intention 的属性

当第会话 B 的事务提交了，事务 A 就会持有一把 (14,20) insert intention 的 gap 锁，这时候在会话 B 中下面的插入语句是可以执行的：

    (root@localhost) [test]> insert into l values(15,17,19,21);
    Query OK, 1 row affected (0.00 sec)

    (root@localhost) [test]> select * from l;
    +----+------+------+------+
    | a  | b    | c    | d    |
    +----+------+------+------+
    |  2 |    4 |    6 |    8 |
    |  4 |    6 |    8 |   10 |
    |  6 |    8 |   10 |   12 |
    |  8 |   10 |   12 |   14 |
    | 10 |   12 |   14 |   16 |
    | 12 |   14 |   16 |   18 |
    | 15 |   17 |   19 |   21 |
    | 20 |   22 |   24 |   26 |
    +----+------+------+------+
    8 rows in set (0.00 sec)

按照之前的理解是有了 (14,20) 这把 gap 锁， 15 这条记录应该是不能插入的，但是 insert intention gap 锁的插入是允许的，它的意义在于提升了插入的性能

## 禁用 next-key locking

把事务隔离级别设置成 READ-COMMITTED 即可，不过设置成 RC 只不过是在用户层面没有了 gap 锁，用户层面加的都是记录锁，但是 gap 锁依然有可能广泛的存在

# 死锁

* 两个或两个以上的事务在执行过程中
* 因争夺锁资源而造成的一种互相等待的现象
* A等待B - B等待A

## 解决死锁

* 超时
  * --innodb_lock_timeout
* wait-for graph
  * 自动死锁检测
  * 锁的信息链表
  * 锁的等待链表

死锁和锁超时其实是两种不同的概念，用超时解决死锁这种机制在数据库里不用的，数据库不通过超时来解决死锁问题，数据库的超时就是锁等待超时

数据库通过构建等待图来实现自动死锁检测的机制，通过锁的信息链表和锁的等待链表构造出等待图，如果发现等待图中的节点有互相等待的回路情况就视为死锁

发现死锁的时候，数据库会选择其中的一个事务来进行回滚，选择哪个事务回滚的原则是使用重做成本 undo 量较小的哪个

## 死锁的例子

    (root@localhost) [test]> select * from z;
    +------+
    | b    |
    +------+
    | NULL |
    |    1 |
    |    2 |
    |    3 |
    |   10 |
    |   20 |
    |   30 |
    |   40 |
    |   50 |
    |   60 |
    |   70 |
    |  100 |
    |  120 |
    |  180 |
    |  200 |
    |  300 |
    +------+
    16 rows in set (0.00 sec)

会话 A：

    (root@localhost) [test]> begin;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> select * from z where b = 1 for update;
    +------+
    | b    |
    +------+
    |    1 |
    +------+
    1 row in set (0.00 sec)

会话 B：

    (root@localhost) [test]> begin;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> select * from l where a = 10 for update;
    +----+------+------+------+
    | a  | b    | c    | d    |
    +----+------+------+------+
    | 10 |   12 |   14 |   16 |
    +----+------+------+------+
    1 row in set (0.00 sec)

会话 A：

    (root@localhost) [test]> select * from l where a = 10 for update;
    | # 阻塞等待超时

会话 B：

    (root@localhost) [test]> select * from z where b = 1 for update;
    ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
    # 造成了死锁

这就是 AB-BA 的互相交叉等待死锁

可以在 show innodb engine status 中查看到最近发生的意思死锁的情况：

    ------------------------
    LATEST DETECTED DEADLOCK
    ------------------------
    2022-06-25 14:44:01 0x7f973473c700
    *** (1) TRANSACTION:
    TRANSACTION 681279, ACTIVE 158 sec starting index read
    mysql tables in use 1, locked 1
    LOCK WAIT 5 lock struct(s), heap size 1136, 3 row lock(s)
    MySQL thread id 2, OS thread handle 140288597595904, query id 121 localhost root statistics
    select * from l where a = 10 for update
    *** (1) WAITING FOR THIS LOCK TO BE GRANTED:
    RECORD LOCKS space id 185 page no 3 n bits 80 index PRIMARY of table `test`.`l` trx id 681279 lock_mode X locks rec but not gap waiting
    Record lock, heap no 6 PHYSICAL RECORD: n_fields 6; compact format; info bits 0
    0: len 4; hex 8000000a; asc     ;;
    1: len 6; hex 0000000a6516; asc     e ;;
    2: len 7; hex b100000ff60110; asc        ;;
    3: len 4; hex 8000000c; asc     ;;
    4: len 4; hex 8000000e; asc     ;;
    5: len 4; hex 80000010; asc     ;;

    *** (2) TRANSACTION:
    TRANSACTION 681281, ACTIVE 90 sec starting index read
    mysql tables in use 1, locked 1
    4 lock struct(s), heap size 1136, 2 row lock(s)
    MySQL thread id 4, OS thread handle 140287396792064, query id 122 localhost root statistics
    select * from z where b = 1 for update
    *** (2) HOLDS THE LOCK(S):
    RECORD LOCKS space id 185 page no 3 n bits 80 index PRIMARY of table `test`.`l` trx id 681281 lock_mode X locks rec but not gap
    Record lock, heap no 6 PHYSICAL RECORD: n_fields 6; compact format; info bits 0
    0: len 4; hex 8000000a; asc     ;;
    1: len 6; hex 0000000a6516; asc     e ;;
    2: len 7; hex b100000ff60110; asc        ;;
    3: len 4; hex 8000000c; asc     ;;
    4: len 4; hex 8000000e; asc     ;;
    5: len 4; hex 80000010; asc     ;;

    *** (2) WAITING FOR THIS LOCK TO BE GRANTED:
    RECORD LOCKS space id 170 page no 4 n bits 88 index idx_b of table `test`.`z` trx id 681281 lock_mode X locks rec but not gap waiting
    Record lock, heap no 3 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
    0: len 4; hex 80000001; asc     ;;
    1: len 6; hex 000000029402; asc       ;;

如果想要把所有的死锁信息都记录下来的话，可以通过 innodb_print_all_deadlocks 参数进行设置

    (root@localhost) [test]> show variables like 'innodb%dead%';
    +----------------------------+-------+
    | Variable_name              | Value |
    +----------------------------+-------+
    | innodb_deadlock_detect     | ON    |
    | innodb_print_all_deadlocks | OFF   |
    +----------------------------+-------+
    2 rows in set (0.00 sec)

这个参数建议设置为 1，my.cnf：

    # innodb
    ...
    innodb_print_all_deadlocks = 1

MySQL5.7 中的 innodb_deadlock_detect 参数还可以控制是否检测死锁，在秒杀的场景之下把这个参数设置为 0 是有意义的，性能会有一点点的提升并没有太大的帮助，所以这个参数默认 1 就性了

## MySQL 8.0 锁的新语法

nowait 语法不等待锁超时，等待就报错：

    (root@localhost) [test]> begin;

    (root@localhost) [test]> select * from stock where skuId = 1 for update nowait;
    ERROR 3572 (HY000)：Do not wait for lock.

skip locked 跳过锁语法，表示对于已经加锁的记录跳过并返回空的结果集：

    (root@localhost) [test]> select * from stock where skuId = 1 for update skip locked;
    Empty set (0.00 sec)

如果业务能用好这两个语法的话，其实是会有非常大的帮助的，比如秒杀这类业务

## 购物车死锁

购物车简单来说就是把商品添加到购物车然后一下子去结算，这容易引发一个叫购物车死锁的问题，这个问题引申开来讲还是一个秒杀的问题

比如把选中的很多个商品都放入到购物车然后结算，结算完之后就是提交订单了，会发现在提交订单的时候在数据库层面会做一个减库存的操作：

    begin;

    update stock set count=count-1 where skuId = 1;
    update stock set count=count-1 where skuId = 2;
    update stock set count=count-1 where skuId = 30;

    commit;

这时如果有另外一个线程做的事情是和前面一样的，不过减库存的 skuId 顺序可能不一样：

    begin;

    update stock set count=count-1 where skuId = 30;
    update stock set count=count-1 where skuId = 1;
    update stock set count=count-1 where skuId = 2;

    commit;

这时候如果在并发的场景之下，就会遇到死锁了，这个问题不是数据库层能解决的，这个情况在业务层每个用户在商品详情页点击添加购物车的时候它的顺序可能都不一样，不能规定任何人点击的商品都是同一个顺序来下单的，所以出现这样的场景是非常正常的

死锁并不是问题，它是数据库的一个正常现象，只有当你的死锁影响到你业务情况时，这时候才认为 DBA 需要介入去处理死锁，否则如果只是线上偶发的一个死锁，不对业务产生很大的影响的话，不是很建议去处理可以忽略掉

### 解决购物车死锁问题

如果需要解决购物车死锁问题，从数据库层面是解决不了的，需要业务层面进行配合

可以在提交订单之前，将商品数据按照 skuId 进行排序，这样就可以解决购物车锁的问题

但是这样会产生另一个锁等待的问题，就算将 InnoDB 锁等待时间参数调小，在并发量很大的情况下又会遇到性能不是特别好的问题，这时候应该引入消息队列的机制，如果要用数据库层面解决可以用 MySQL 的线程池

## 唯一索引引起的死锁

案例情景：

    (root@localhost) [test]> select * from l;
    +----+------+------+------+
    | a  | b    | c    | d    |
    +----+------+------+------+
    |  2 |    4 |    6 |    8 |
    |  4 |    6 |    8 |   10 |
    |  8 |   10 |   12 |   14 |
    | 10 |   12 |   14 |   16 |
    | 12 |   14 |   16 |   18 |
    | 20 |   22 |   24 |   26 |
    +----+------+------+------+
    7 rows in set (0.00 sec)

    (root@localhost) [test]> show create table l\G
    *************************** 1. row ***************************
          Table: l
    Create Table: CREATE TABLE `l` (
      `a` int(11) NOT NULL,
      `b` int(11) DEFAULT NULL,
      `c` int(11) DEFAULT NULL,
      `d` int(11) DEFAULT NULL,
      PRIMARY KEY (`a`),
      UNIQUE KEY `idx_c` (`c`),  <--唯一索引
      KEY `b` (`b`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8
    1 row in set (0.00 sec)

    (root@localhost) [test]> select @@tx_isolation;
    +----------------+
    | @@tx_isolation |
    +----------------+
    | READ-COMMITTED |
    +----------------+
    1 row in set, 1 warning (0.00 sec)

会话 A：

    (root@localhost) [test]> begin;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> delete from l where c = 12;
    Query OK, 1 row affected (0.00 sec)

会话 B：

    (root@localhost) [test]> insert into l values(40,40,12,60);
    | # 阻塞+超时

    # status 中显示在等待10这把锁
    insert into l values(40,40,12,60)
    ------- TRX HAS BEEN WAITING 3 SEC FOR THIS LOCK TO BE GRANTED:
    RECORD LOCKS space id 185 page no 4 n bits 80 index idx_c of table `test`.`l` trx id 682269 lock mode S waiting
    Record lock, heap no 5 PHYSICAL RECORD: n_fields 2; compact format; info bits 32
    0: len 4; hex 8000000c; asc     ;;
    1: len 4; hex 80000008; asc     ;;

如果这张表上面没有唯一索引的话，在这种情况下所有的插入在 RC 的事务隔离级别之下应该都是并行的，不会被阻塞，因为 RC 隔离级别下是没有 gap 锁的只有 record lock 只锁住记录，不锁住范围的话插入肯定就是可以并行插入的不会被阻塞

但是如果这张表中除了自增主键以外还有唯一索引的话，插入就会发生等待

假设有 1 3 4 7 四条记录，现在要插入一条 3 的记录，这时候的插入逻辑是：

1. 对于插入来说首先是找大于 3(这里是插入的那个3，不是原有的那个3) 的第一条记录，这条大于 3 的第一条记录叫做 next_rec
2. 看此记录上是否有 gap 锁，如果有的话就不能插，没有的话才可以插，inset intention 的话也可以插
3. 如果插入的 3 是唯一索引的列，对于唯一索引显然只有上面两个步骤的是不行的，现在会去看 next_rec 记录的前面一条记录 prev_rec，如果 prev_rec = rec(当前要插入的记录) 的话就意味着冲突了，这样的话就是用来检查它的唯一性
4. 如果 prev_rec 上面是有 lock 的话，这时候插入的 3 这条记录，需要对这一条记录加上一个 S lock，这就是 S lock 产生的一个原因

这里对唯一索引的插入需要额外再做一次检查

## 唯一索引造成的死锁问题

案例情景：

    (root@localhost) [test]> create table ll( a int primary key );
    Query OK, 0 rows affected (0.01 sec)

    (root@localhost) [test]> insert into ll values (1);
    Query OK, 1 row affected (0.00 sec)

    (root@localhost) [test]> select * from ll;
    +---+
    | a |
    +---+
    | 1 |
    +---+
    1 row in set (0.00 sec)

会话 A：

    (root@localhost) [test]> begin;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> delete from ll where a = 1;  # 加 X lock
    Query OK, 1 row affected (0.00 sec)

会话 B：

    (root@localhost) [test]> begin;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> insert into ll values(1);    # 因为上面有 X lock，所以这里持有 S lock 并 wait
    | # 阻塞

会话 C：

    (root@localhost) [test]> begin;
    Query OK, 0 rows affected (0.00 sec)

    (root@localhost) [test]> insert into ll values(1);    # 因为上面有 X lock，所以这里持有  S lock 并 wait
    | # 阻塞

会话 A：

    (root@localhost) [test]> commit;      # 释放 X 锁
    Query OK, 0 rows affected (0.00 sec)

会话 B：

    (root@localhost) [test]> insert into ll values(1);  # 由于上面释放了先前的 X 锁，这里要插入数据了就需要加上一个 X lock，但是这个 X lock 在等待会话 C 持有的 S lock
    Query OK, 1 row affected (14.35 sec) # 会话A提交之后插入了1

会话 C：

    (root@localhost) [test]> insert into ll values(1);  # 这里也要加 X lock，也在等待会话 B 中的 S lock，所以产生了死锁
    ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction  <--提示了死锁

如果在线上遇到了死锁的话，建议先看一下它的索引是否是唯一的，如果是唯一索引的话可能就是这样的一个问题