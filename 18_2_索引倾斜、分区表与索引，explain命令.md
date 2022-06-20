# 索引倾斜、分区表与索引、explain 命令

有个别同学回去尝试建立一个只有几条数据的表，然后几十条数据去建立索引

走不走索引不是绝对的，MySQL是很聪明的，MySQL回去分析这个SQL语句，然后去分析分别走什么索引？或者不走索引进行全表扫描看一下时间，最后对比一下哪个快

## 索引倾斜

很多时候我们喜欢去 status 这类字段上创建索引，status 通常来说就几种有限的状态比如 0，1，2，其中大部分的订单状态都是已完成的状态，这就会导致一个叫做索引倾斜的问题

我们去查询的时候大部分都是查未完成的订单，而这种未完成的状态却又是很少量的，比如说 1000w 的一张表可能未完成的订单只有 10w，如果去取未完成的订单的话就符合 B+ 树索引的一个条件，就是去大量的数据中去取少量的数据，这就是索引倾斜

0 1 2 三个状态，假设其中 0 这种状态只占 1% 的数据，如果要查的就是这 1% 的数据，这时候去 status 上创建索引就是有意义的，因为你是取小部分的数据

但是这时候如果去做一些比较复杂的查询的话，explain 优化器不能感知到倾斜的问题可能会出错，它并不知道你的索引是倾斜的从而导致它在选择上可能会有一些问题。如果在日常开发中优化器的执行计划出现了一些小小的偏差，那么可以去看一下它使用的索引是哪个索引，然后是否存在倾斜的情况

### 强制使用某个索引

    (root@localhost) [dbt3]> explain select * from lineitem where l_orderkey = 1\G
    *************************** 1. row ***************************
              id: 1
      select_type: SIMPLE
            table: lineitem
      partitions: NULL
            type: ref
    possible_keys: PRIMARY,i_l_orderkey,i_l_orderkey_quantity  <--表示可能用到的3个索引
              key: PRIMARY                                     <--优化器选择了使用主键作为索引
          key_len: 4
              ref: const
            rows: 6
        filtered: 100.00
            Extra: NULL
    1 row in set, 1 warning (1.01 sec)

这个例子比较简单通常来说不会出错，但如果是一些包含 group by， join 语句的比较复杂的例子可能就会出错，这时候可以使用 **force index** 去强制使用一个指定的索引

    (root@localhost) [dbt3]> explain select * from lineitem force index(i_l_orderkey) where l_orderkey = 1\G
    *************************** 1. row ***************************
              id: 1
      select_type: SIMPLE
            table: lineitem
      partitions: NULL
            type: ref
    possible_keys: i_l_orderkey
              key: i_l_orderkey     <--优化器此时强制去使用了i_l_orderkey作为索引
          key_len: 4
              ref: const
            rows: 6
        filtered: 100.00
            Extra: NULL
    1 row in set, 1 warning (0.00 sec)

如果真的碰到索引倾斜，确认所有的查询都是走另外的索引会更快的情况下，的确就可以去 force index 一下，但是大部分的场景不需要使用到这个 force index(hint)，而且要去强制使用索引还是因为索引本身是存在倾斜的本身数据里有些错误，也不能说是优化器出错

建议这种提示性 hint 写得越少越好

## SQL JOIN 算法

对于单条语句，用 explain 看使用到的 key 也都能分析得差不多了

如果是有多个条件的话加复合索引，多个条件再去 order by 就把 order by 的列加入复合索引，这样 Extra 这个选项就不会出现 filesort 排序的提示

这些都是比较基本的，线上还会出现一些 JOIN 复杂的查询，这时就需要了解 JOIN 算法的问题

### nested_loop join 嵌套查询

#### simple nested_loop join 简单(朴素表)嵌套查询

算法：

    For each row r in R do
              For each row r in S do
                        if r s satisfy the join condition
                                Then output the tuple <r,s>

就是写两个 for 循环，for R 然后 for S 这样的一个集合，如果它们关联 Join 的列是相等的话，那么就输出结果

这种算法的扫描成本是 O(Rn X Sn)，非常低效

**MySQL 中永远不会使用这种算法**

#### index nested_loop join 基于索引的嵌套查询

最推荐的嵌套查询 JOIN 算法

算法：

    For each row r in R do
              lookup r in S index
                        if found s == r
                                 Then output the tuple <r,s>

基于索引的嵌套查询，对于 R 表中的每一条记录并不是逐条去比较的，而是到 S 表中列上的索引去进行查询，通过索引去找数据的速度是非常快的

R 表可以称为驱动表（外表），S 表可以称为被驱动表（内表）

这种算法的扫描成本是 O(Rn)，也就是外表的行数的扫描次数

那么如果是 A 和 B 表进行关联，那么 A 表是 R 表，还是 B 表是 R 表

对于 inner join 来说，A join B 和 B join A 其实结果都是一样的，通常来说只要你的驱动表越小优化器就倾向于使用小表作为驱动表

比如说一张 10w 的表和一张 100w 的表进行 join，优化器倾向于使用 10w 的小表来作为外表，然后为 100w 大表上需要去 join 的列创建索引。如果索引高度为3，这时候扫描次数是 10w * 3 = 30万次，但是如果将 100w 作为外表，那么扫描次数就变成了100w * 3 = 300万次，明显比 10w 作为外表大多了，这就是为什么优化器倾向于使用小表作为驱动表

对于 MySQL 来说只要两张表进行关联的话，非常建议两张表上面一定要去创建索引，**而且此时索引应该创建在比较大的那张表也就是内表 S 上**

需要注意的是在线上的话不会是直接两张表去关联而不带其他附加过滤条件的，优化器会把 where 后面的过滤条件也考虑进去，然后看经过这些条件过滤条件之后，再判断哪张表作为驱动表，那张表作为内表，但是索引有倾斜的话在索引顺序的选择上还是很容易出错

#### block nested_loop join 基于块的嵌套查询

* 两张表上没有索引的时候触发
* 优化 simple nested_loop join 
* 减少内部表的扫描次数

    For each tuple r in R do
              store used columns as p from R in join buffer
              For each tuple s in S do
                        if p and s satisfy the join condition
                                  Then output the tuple <p,s>

原理是通过加入内存 join_buffer 用空间换时间，系统变量 join_buffer_size 决定了 join_buffer 的大小

join_buffer 可被用于连接是 All，index，range 的类型

join_buffer 只存储需要进行查询操作的相关列数据，**而不是整行的记录**，整体来说还是比较高效的

join_buffer_size 建议最大设置到 1G 就差不多了

对于外表要关联的这些列并不直接去一行一行进行比较，现在是把 R 中的关联列(不是整行记录)全部放到 join_buffer 里面（如果能全部放下的话就一次全放进去），然后一次性的和内表 S 中的列数据进行比较，这样就能减少内表的扫描次数

如果 join_buffer_size 设置的足够大能够把驱动表中的所有列都能 cache 起来的话，那么只需要扫描 1 次内表

如果 join 已经使用了索引，那么再加大 join_buffer_size 也是没有意义的，网上搜到的某些的为了提高 join 性能而盲目的加大 join_buffer_size 显然不完全是对的

**MySQL 中建议两张表关联的话一定要加索引，如果分不清哪张作为驱动表，哪张作为内表的时候，就干脆简单点两张表都加索引**

#### SNLP BNLP 表扫描次数比较

数据模型(假设 join_buffer 值足够大)：

    A表   B表
    1     1
    2     2
    3     3
          4

|              | SNLP | BNLP |
| ------------ | ---- | ---- |
| 外表扫描次数 | 1    | 1    |
| 内表扫描次数 | 3    | 1    |
| 比较次数     | 12   | 12   |

注意 BNLP 算法中，是 join_buffer 的数据和 B 表中的每个数据进行比较，比如(123)和1、(123)和2比较以此类推，所以还是 12 次 

比较次数并没有省下来，但是内表扫描次数缩小到了 1 次，数据量小的时候可能还没有什么感觉，但假设 A B 表都是百万级别的这时候 BNLP 的性能就会好非常多了。百万级别没有索引的话通过 BNLP 算法 MySQL 还是勉强能跑出来的，除了加索引优化的方法是加大 join_buffer_size，由于只是把 R 中的关联列(不是整行记录)放入 join_buffer，所以整体来说还是比较高效的

#### batched key access join

bka join 是用来解决如果关联的列是二级索引的时候，会对回表的列进行一个 MRR 排序操作，它会先 cache 起来再进行回表，此时的回表就会变得比较顺序性能就会变得比较好

因为 bka 调用的是 MRR 的接口，所以 join 算法默认是不会启用的，如果要启用的话需要写相关的 [hint](https://dev.mysql.com/doc/refman/5.7/en/optimizer-hints.html#optimizer-hints-table-level) ：

    SELECT /*+ NO_BKA(t1, t2) */ t1.* FROM t1 INNER JOIN t2 INNER JOIN t3;
    SELECT /*+ NO_BNL() BKA(t1) */ t1.* FROM t1 INNER JOIN t2 INNER JOIN t3;

这也是 MySQL 现在比较大的问题，那为什么还有这么多人用 MySQL 呢？主要是因为在线业务(OLTP)都是一些比较简单的查询，OLTP 的业务也不推荐使用很多的 join 操作

MySQL 也不是很适合做 OLAP 的业务，所以不推荐用 MySQL 来做 OLAP 的业务

## 一个可能用不到索引的例子

下面这个例子中并没有使用到索引，就会对整张表进行扫描，然后通过 where 条件来进行过滤而不是通过索引来进行定位，所以在 Extra 那里会看到有 Using where

    (root@localhost) [dbt3]> explain select * from orders where o_orderdate > '1998-01-01'\G
    *************************** 1. row ***************************
              id: 1
      select_type: SIMPLE
            table: orders
      partitions: NULL
            type: ALL
    possible_keys: i_o_orderdate        <--对于orderdate实际上是有索引的
              key: NULL                 <--但是优化器并没有选择使用索引定位，而是强制扫所有的数据，然后通过where来进行过滤
          key_len: NULL
              ref: NULL
            rows: 1493376
        filtered: 18.52
            Extra: Using where          <--只通过where进行了过滤，由于i_o_orderdate是一个二级索引，走索引需要回表带来额外的开销，所以优化器这里选择不走索引而是选择了扫描全表(会用到主键)
    1 row in set, 1 warning (0.00 sec)

查询条件改成 o_orderdate > '1999-01-01 会发现优化器会去使用索引

    (root@localhost) [dbt3]> explain select * from orders where o_orderdate > '1999-01-01'\G
    *************************** 1. row ***************************
              id: 1
      select_type: SIMPLE
            table: orders
      partitions: NULL
            type: range
    possible_keys: i_o_orderdate
              key: i_o_orderdate        <--这里用到了索引，这里数据量小使用二级索引代价更小，所以优化器选择了使用索引，但是如果数据量大的情况下，使用索引产生大量的回表操作就不划算了
          key_len: 4
              ref: NULL
            rows: 1                     <--只有1条数据需要扫描
        filtered: 100.00
            Extra: Using index condition
    1 row in set, 1 warning (0.00 sec)

这样一下用到索引一下用不到索引并不代表 MySQL 优化器不稳定，恰恰相反的是这说明 MySQL 优化器很聪明

导致这个现象的原因是因为这里的 o_orderdate 其实是一个二级索引 KEY \`i_o_orderdate\` (\`o_orderDATE\`) 并且是 select *，所以 o_orderdate > '1998-01-01' 例子中需要回表

通过一个数据模型来分析一下这个问题产生的原因，假设对订单表的某个查询条件匹配的是 150w 个记录且每行的记录大小是 100 个字节，现在通过 order_date 来过滤的话优化器预估出来的行数是 50 万行，意味着就要回表 50w 次 IO。这还不一定准确，因为如果此时的 B+树高度为 3，那么就需要回表 150w 次 IO

如果不走 order_date 索引，直接去扫描 150w 行的主键，那么只需要 150w 除以每个叶能存放的记录 (16k / 100字节) 次数的 IO，这样肯定是小于 150w 次

而且主键扫描基本上可以认为是一个顺序 IO，而回表 IO 则是随机的，随机 IO 比顺序 IO 慢可能至少有 10 倍

所以 '1999-01-01' 这个例子使用主键扫描肯定会比较快，MySQL 优化器会自动根据这样的 cost 成本判断使用哪种方法去进行扫描

简单来说就是这个模型下面，走主键扫描会比走索引回表扫描的数据量更小也就更快更效率，这就是这个问题产生的原因

### 通过 MRR 进行优化

对于随机 IO 的回表扫描语句，可以通过 MRR 进行优化：

    (root@localhost) [dbt3]> explain select /*+ MRR(orders) */ * from orders where o_orderdate > '1998-01-01'\G
    *************************** 1. row ***************************
              id: 1
      select_type: SIMPLE
            table: orders
      partitions: NULL
            type: range
    possible_keys: i_o_orderdate
              key: i_o_orderdate
          key_len: 4
              ref: NULL
            rows: 276576
        filtered: 100.00
            Extra: Using index condition; Using MRR    <--提示用到了MRR
    1 row in set, 1 warning (0.00 sec)

MRR 也是通过空间换时间，会新开一块内存用来存放每次回表的一个主键值，等放不下了就会去进行一次 sort 操作，排序完之后再去进行回表，这个时候去进行回表操作它的 IO 就会变得比较顺序了，然是依然是需要很多的 IO 的这个避免不了。这样就是通过随机转顺序的方式来做了一个优化。

    (root@localhost) [dbt3]> select /*+ MRR(orders) */ * from orders where o_orderdate > '1998-01-01'\G
    ...
    132988 rows in set (1.73 sec)

    # 这时去掉 MRR关键词也没有问题，因为所有的记录都在内存里了，虽然也涉及到 IO 但是在内存里会很快，基本感觉不到区别
    (root@localhost) [dbt3]> select * from orders where o_orderdate > '1998-01-01'\G

MRR 有个比较遗憾的点就是优化器永远不会主动去选择 MRR，至少在 5.7 版本里是这样的

## join 和索引的关系

有索引的话千万级别的 join 其实也是可以的不至于不行，但速度的确会很慢

导致缓慢的原因是 join 的时候 join 的列如果是**二级索引**就需要回表，用下面的 SQL 语句举个例子：

    (root@localhost) [dbt3]> SELECT 
        MAX(l_extendedprice)
    FROM
        orders,
        lineitem
    WHERE
        o_orderdate BETWEEN '1995-01-01' AND '1995-01-31'
            AND l_orderkey = o_orderkey;
            
    set optimizer_switch='batched_key_access=on,mrr_cost_based=off';

订单表 orders 和 订单明细表 lineitem 进行关联，关联的条件是订单ID l_orderkey = o_orderkey，由于用的 l_orderkey 是二级索引，也就是说 lineitem 这张表是内表，内表上有这样的一个索引来进行关联，但是这个索引是二级索引，用二级索引关联比较是没有问题，比较完通过索引来快速定位之后，要查 l_extendedprice 还是需要回表

关联主键的话不需要回表，代价就小多了

## hash join (MySQL5.+现在是不支持的)

hash join 也是先将外表中的数据先放到内存里去，放进去之后并不直接通过 join buffer 和内表去直接进行比较，而是对 join buffer 中的列去创建了一张哈希表，然后在扫描内表，内表中的每条记录去探测这张哈希表，这就是 hash join 最基本的一个原理

因为哈希的这个算法，通常假设某一条记录从哈希表中进行寻找的话它的成本只需要一次

    A表   B表
    1     1
    2     2
    3     3
          4

|              | hash join |
| ------------ | --------- |
| 外表扫描次数 | 1         |
| 内表扫描次数 | 3         |
| 比较次数     | 4         |

再加上 hash join 的一个很重要的特点和区别就是它不需要创建索引也不需要回表，所以 hash join 的优势就会比较大

但是 hash join 有个缺点就是只支持等值的比较 A.x = B.y，大于小于等于这种是不行的

## Explain 命令

* 显示 SQL 语句的执行计划
* 5.6 版本支持 DML 语句 
* 5.6 版本开始支持 JSON 格式的输出

### query_cost 查询成本

query_cost 这个值没有单位，数值越大就表示开销越大

通过 JSON 格式输出：

    (root@localhost) [dbt3]> explain format=json select * from orders where o_orderdate > '1998-01-01'\G
    *************************** 1. row ***************************
    EXPLAIN: {
      "query_block": {
        "select_id": 1,
        "cost_info": {
          "query_cost": "310880.20"      <--查询成本，会选择成本最小的作为它的一个执行计划
        },
        "table": {
          "table_name": "orders",
          "access_type": "ALL",
          "possible_keys": [
            "i_o_orderdate"
          ],
          "rows_examined_per_scan": 1493376,
          "rows_produced_per_join": 276576,
          "filtered": "18.52",
          "cost_info": {
            "read_cost": "255565.00",
            "eval_cost": "55315.20",
            "prefix_cost": "310880.20",
            "data_read_per_join": "35M"
          },
          "used_columns": [
            "o_orderkey",
            "o_custkey",
            "o_orderstatus",
            "o_totalprice",
            "o_orderDATE",
            "o_orderpriority",
            "o_clerk",
            "o_shippriority",
            "o_comment"
          ],
          "attached_condition": "(`dbt3`.`orders`.`o_orderDATE` > '1998-01-01')"
        }
      }
    }
    1 row in set, 1 warning (0.68 sec)

加 force index 后的效果：

    (root@localhost) [dbt3]> explain format=json select * from orders force index(i_o_orderdate) where o_orderdate > '1998-01-01'\G
    *************************** 1. row ***************************
    EXPLAIN: {
      "query_block": {
        "select_id": 1,
        "cost_info": {
          "query_cost": "387207.41"      <--force index后查询成本变成了38w，所以优化器会优先使用上面那种执行计划
        },
        "table": {
          "table_name": "orders",
          "access_type": "range",
          "possible_keys": [
            "i_o_orderdate"
          ],
          "key": "i_o_orderdate",        <--使用到了i_o_orderdate索引
          "used_key_parts": [
            "o_orderDATE"
          ],
          "key_length": "4",
          "rows_examined_per_scan": 276576,
          "rows_produced_per_join": 276576,
          "filtered": "100.00",
          "index_condition": "(`dbt3`.`orders`.`o_orderDATE` > '1998-01-01')",
          "cost_info": {
            "read_cost": "331892.21",
            "eval_cost": "55315.20",
            "prefix_cost": "387207.41",
            "data_read_per_join": "35M"
          },
          "used_columns": [
            "o_orderkey",
            "o_custkey",
            "o_orderstatus",
            "o_totalprice",
            "o_orderDATE",
            "o_orderpriority",
            "o_clerk",
            "o_shippriority",
            "o_comment"
          ]
        }
      }
    }
    1 row in set, 1 warning (0.00 sec)

加 MRR 后的效果：

    (root@localhost) [dbt3]> explain format=json select /*+ MRR(orders) */ * from orders force index(i_o_orderdate) where o_orderdate > '1998-01-01'\G
    *************************** 1. row ***************************
    EXPLAIN: {
      "query_block": {
        "select_id": 1,
        "cost_info": {
          "query_cost": "311602.90"        <--也是31w多，但是还是比第一种例子种的扫描主键方式代价要大一点点，所以优化器还是会选择第一个例子中的方式
        },
        "table": {
          "table_name": "orders",
          "access_type": "range",
          "possible_keys": [
            "i_o_orderdate"
          ],
          "key": "i_o_orderdate",
          "used_key_parts": [
            "o_orderDATE"
          ],
          "key_length": "4",
          "rows_examined_per_scan": 276576,
          "rows_produced_per_join": 276576,
          "filtered": "100.00",
          "index_condition": "(`dbt3`.`orders`.`o_orderDATE` > '1998-01-01')",
          "using_MRR": true,
          "cost_info": {
            "read_cost": "256287.70",
            "eval_cost": "55315.20",
            "prefix_cost": "311602.90",
            "data_read_per_join": "35M"
          },
          "used_columns": [
            "o_orderkey",
            "o_custkey",
            "o_orderstatus",
            "o_totalprice",
            "o_orderDATE",
            "o_orderpriority",
            "o_clerk",
            "o_shippriority",
            "o_comment"
          ]
        }
      }
    }
    1 row in set, 1 warning (0.00 sec)

### Explain 输出

| 列            | 含义                                   |
| ------------- | -------------------------------------- |
| id            | 执行计划的 id 标志                     |
| select_type   | SELECT的类型                           |
| table         | 输出记录的表                           |
| partitions    | 符合的分区，[PARTITIONS]               |
| type          | JOIN的类型                             |
| possible_keys | 优化器可能使用到的索引                 |
| key           | 优化器实际选择的索引                   |
| key_len       | 使用索引的字节长度                     |
| ref           | 进行比较的索引列                       |
| rows          | 优化器预估的记录数量                   |
| filtered      | 根据条件过滤得到记录的百分比[EXTENDED] |
| Extra         | 额外的显示选项                         |

#### select_type

    (root@localhost) [dbt3]> explain SELECT 
        *
    FROM
        part
    WHERE
        p_partkey IN (SELECT 
                l_partkey
            FROM
                lineitem
            WHERE
                l_shipdate BETWEEN '1997-01-01' AND '1997-02-01')
    ORDER BY p_retailprice DESC
    LIMIT 10;
    +----+--------------+-------------+------------+--------+----------------------------------------------+--------------+---------+---------------------+--------+----------+-----------------------------+
    | id | select_type  | table       | partitions | type   | possible_keys                                | key          | key_len | ref                 | rows   | filtered | Extra                       |
    +----+--------------+-------------+------------+--------+----------------------------------------------+--------------+---------+---------------------+--------+----------+-----------------------------+
    |  1 | SIMPLE       | part        | NULL       | ALL    | PRIMARY                                      | NULL         | NULL    | NULL                | 198303 |   100.00 | Using where; Using filesort |
    |  1 | SIMPLE       | <subquery2> | NULL       | eq_ref | <auto_key>                                   | <auto_key>   | 5       | dbt3.part.p_partkey |      1 |   100.00 | NULL                        |
    |  2 | MATERIALIZED | lineitem    | NULL       | range  | i_l_shipdate,i_l_suppkey_partkey,i_l_partkey | i_l_shipdate | 4       | NULL                | 140106 |   100.00 | Using index condition       |
    +----+--------------+-------------+------------+--------+----------------------------------------------+--------------+---------+---------------------+--------+----------+-----------------------------+
    3 rows in set, 1 warning (0.00 sec)

执行出来的结果首先要看 id ，MySQL 中比较讨厌的一个问题是，执行计划出来的 id 是会相等的，并且对于不同的 id 应该怎么来看呢？

可以记住一个口诀，id 相等的从上往下看，id 不同的从下往上看，比如上面的例子中首先执行的是 id 为 2 的，执行完之后再去执行 id 为 1 的第一个，然后再执行 id 为 1 的第二个，也就是上面例子中其实是按照第 3 1 2 行的顺序来执行的

而且这个口诀只对 80% 的场景是适用的，还有一些场景是不适用的

上面执行计划的解释，按照①②③的顺序看：

    (root@localhost) [dbt3]> ...\G
    *************************** 1. row ***************************
              id: 1                       <--②：②和③的id相等，根据经验来说id相等的话通常来说表示join关联，顺位第一的表示的是外表
      select_type: SIMPLE
            table: part                   <--part表和③中的<subquery2>的所有记录进行了关联
      partitions: NULL
            type: ALL
    possible_keys: PRIMARY
              key: NULL
          key_len: NULL
              ref: NULL
            rows: 198303                  <-- 关联的数据一共是19w行记录
        filtered: 100.00
            Extra: Using where; Using filesort  <--用到了order by所以这里还是有Using filesort
    *************************** 2. row ***************************
              id: 1                       <--③：②和③的id相等，根据经验来说id相等的话通常来说表示join关联，顺位第二的表示的是内表
      select_type: SIMPLE
            table: <subquery2>            <--<subquery2>就是①子查询中产生的那张14w多记录的表
      partitions: NULL
            type: eq_ref                  <--表示通过唯一索引来进行关联的
    possible_keys: <auto_key>
              key: <auto_key>             <--因为这张表示内表，所以在内表上要去创建索引，优化器自动把①子查询里面取出来的数据去添加了一个唯一索引，所以①的select_type为MATERIALIZED，表示产生了一张实际的表并且去为l_partkey添加了唯一索引，用唯一索引的原因是这里是where in，in里面是去重的
          key_len: 5
              ref: dbt3.part.p_partkey    <--表示关联的列是哪一个，这里关联的列就是②中的p_partkey（和①中子查询中的l_partkey进行了关联）
            rows: 1
        filtered: 100.00
            Extra: NULL
    *************************** 3. row ***************************
              id: 2                       <--①：这条执行计划是从这里先开始的，所以先从这里开始看
      select_type: MATERIALIZED           <--表示产生了一张固化的表
            table: lineitem               <--先通过子查询先查了lineitem这张表
      partitions: NULL
            type: range                   <--是一个范围的查询
    possible_keys: i_l_shipdate,i_l_suppkey_partkey,i_l_partkey <--可能使用到的索引
              key: i_l_shipdate           <--最后使用了i_l_shipdate索引
          key_len: 4                      <--i_l_shipdate是date类型的，所以这里是占用了4个字节
              ref: NULL
            rows: 140106                  <--优化器预估出来的有多少行记录
        filtered: 100.00                  <--过滤100%，就是说过滤出来就是140106的记录
            Extra: Using index condition
    3 rows in set, 1 warning (0.00 sec)

##### 物化的解释

通常来说查询的结果是不需要物化的

就比如说A表和B表进行关联，B表是通过子查询选出来的(比如1231四条记录)，本来这些记录都是存放在内存里面的，内存里面的记录不是物化的无法为其建立索引，所以这时和A表进行关联B表一定是外表

如果使用的是 in 来进行关联，in 出来的结果是要做去重的，在 MySQL 中去重的话会建一张表然后对这张表上加一个唯一键，那上面B表的结果就从(1231)变成了(123)。这就可以认为此时B表子查询出来的是物化的，并且这张表上还有一个索引并不是光一个结果集，这张表试试上面例子中出现过的 <subquery2\>

在 MySQL 中如果是一个 in 的子查询，这个 in 的子查询 MySQL 优化器会帮你重写成 Join ，并且优化器会非常聪明的帮你选择这个子查询是作为外表还是内表

##### 将这条 in 的子查询改写成 join 的写法

    (root@localhost) [dbt3]> SELECT 
        a.*
    FROM
        part a,
        (SELECT DISTINCT
            l_partkey
        FROM
            lineitem
        WHERE
            l_shipdate BETWEEN '1997-01-01' AND '1997-02-01') b
    WHERE
        a.p_partkey = b.l_partkey
    ORDER BY a.p_retailprice DESC
    LIMIT 10;

重写完后会存在一个问题，现在 a 表和 b 表进行 join 的话，这时 b 表永远是外表，因为这样的写法只是去产生了一张临时派生表，并且没有对派生表创建索引的功能，如果这时候派生表产生的结果集特别大的话，这时候性能就不如用 in 了，因为 in 会帮你作为内表

##### 示例

这个执行计划 l_orderkey 上是有索引的，但是它最后使用的是 PRIMARY KEY 而没有使用索引

    (root@localhost) [dbt3]> explain format=json SELECT
            MAX(l_extendedprice)
        FROM
            orders,
            lineitem
        WHERE
            o_orderdate BETWEEN '1995-01-01' AND '1995-01-31'
                AND l_orderkey = o_orderkey;
    ************************* 1. row ***************************
              id: 1                         <--id都是1，所以从上往下看
      select_type: SIMPLE
            table: orders                   <--这里orders表作为外表，是首先根据过滤条件把数据过滤出来之后再作为外表
      partitions: NULL
            type: range                     <--范围查询
    possible_keys: PRIMARY,i_o_orderdate
              key: i_o_orderdate
          key_len: 4
              ref: NULL
            rows: 36758
        filtered: 100.00
            Extra: Using where; Using index
    *************************** 2. row ***************************
              id: 1
      select_type: SIMPLE
            table: lineitem                 <--然后和lineitem这张表进行关联
      partitions: NULL
            type: ref
    possible_keys: PRIMARY,i_l_orderkey,i_l_orderkey_quantity
              key: PRIMARY                  <--关联的时候用得是lineitem的主键
          key_len: 4
              ref: dbt3.orders.o_orderkey   <--关联的列是orders.o_orderkey
            rows: 4
        filtered: 100.00
            Extra: NULL
    2 rows in set, 1 warning (0.00 sec)

如果强制使用二级索引 i_l_orderkey ，会发现查询成本是会非常大

    (root@localhost) [dbt3]> explain format = json SELECT 
        MAX(l_extendedprice)
    FROM
        orders,
        lineitem FORCE INDEX (i_l_orderkey)
    WHERE
        o_orderdate BETWEEN '1995-01-01' AND '1995-01-31'
            AND l_orderkey = o_orderkey\G
    *************************** 1. row ***************************
    EXPLAIN: {
      "query_block": {
        "select_id": 1,
        "cost_info": {
          "query_cost": "191184.09"  <--这里是18w，而上面的例子只有8w多，比上面多多了因为这里需要回表
        },
    ...
    }
    1 row in set, 1 warning (0.00 sec)

##### 合并两个结果集的示例

    (root@localhost) [dbt3]> explain SELECT
              *
            FROM
                lineitem
            WHERE
                l_shipdate <= '1992-12-31'
            UNION SELECT    -- UNION表示如果有重复的记录就需要去重
                *
            FROM
                lineitem
            WHERE
                l_shipdate >= '1997-01-01';
    +----+--------------+------------+------------+------+---------------+------+---------+------+---------+----------+-----------------+
    | id | select_type  | table      | partitions | type | possible_keys | key  | key_len | ref  | rows    | filtered | Extra           |
    +----+--------------+------------+------------+------+---------------+------+---------+------+---------+----------+-----------------+
    |  1 | PRIMARY      | lineitem   | NULL       | ALL  | i_l_shipdate  | NULL | NULL    | NULL | 5756863 |    31.44 | Using where     |
    |  2 | UNION        | lineitem   | NULL       | ALL  | i_l_shipdate  | NULL | NULL    | NULL | 5756863 |    50.00 | Using where     |
    | NULL | UNION RESULT | <union1,2> | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    NULL |     NULL | Using temporary |
    +----+--------------+------------+------------+------+---------------+------+---------+------+---------+----------+-----------------+
    3 rows in set, 1 warning (0.00 sec)

输出结果的第三行，也是 id 是空值 NULL 的那行的 select_type 为 UNION RESULT，UNION RESULT 的意思就是把两个结果集合并起来，合并起来的时候会使用到临时表，所以这里 Extra 的值为 Using temporary，这里会使用到临时表的原因是因为 UNION 表示需要去重然后在上面加了唯一索引

从这个例子可以看到 MySQL 一个 SQL 语句里也是可以用到两个索引的，这里上半部分使用了一个索引，下半部分使用了一个索引

##### 求行号问题性能差的原因分析

    (root@localhost) [employees]> explain SELECT -- 先执行这个外部的查询
        emp_no,
        dept_no,
        (SELECT 
                COUNT(1)
            FROM
                dept_emp t2
            WHERE
                t1.emp_no <= t2.emp_no) AS row_num
    FROM
        dept_emp t1;
    +----+--------------------+-------+------------+-------+---------------+---------+---------+------+--------+----------+------------------------------------------------+
    | id | select_type        | table | partitions | type  | possible_keys | key     | key_len | ref  | rows   | filtered | Extra                                          |
    +----+--------------------+-------+------------+-------+---------------+---------+---------+------+--------+----------+------------------------------------------------+
    |  1 | PRIMARY            | t1    | NULL       | index | NULL          | dept_no | 16      | NULL | 331143 |   100.00 | Using index                                    |
    |  2 | DEPENDENT SUBQUERY | t2    | NULL       | ALL   | PRIMARY       | NULL    | NULL    | NULL | 331143 |    33.33 | Range checked for each record (index map: 0x1) |
    +----+--------------------+-------+------------+-------+---------------+---------+---------+------+--------+----------+------------------------------------------------+
    2 rows in set, 2 warnings (0.00 sec)

这个执行计划由于查询类型叫做**依赖子查询 DEPENDENT SUBQUERY**，依赖子查询肯定是依赖外部表的，所以对于这个查询是先执行外部的查询，所以这里 id 的执行顺序是 1 2 而不是按照口诀说的 2 1

另外这张表每次要关联 331143 * (331143 / 33) 次，也就是 33 万条外部记录乘以过滤后的 DEPENDENT SUBQUERY 数据，所以这条 SQL 语句的代价会非常大，每条外部记录都要去关联扫描10 万多条记录

#### type 类型

按照从上到下，越往后代价可能越大

* system 只有一行记录的系统表
* const 最多只有一行返回记录，如主键查询
* eq_ref 通过唯一键进行 Join
* ref 使用普通索引进行查询
* fulltext 使用全文索引进行查询
* ref_or_null 和ref类似，使用普通索引进行查询，但要查询NULL值
* index_merge or查询会使用到的类型，一条SQL语句可能使用到了2个索引来进行merge，通常来说比较少见
* unique_subquery 子查询的列是唯一索引
* index_subquery 子查询的列是普通索引
* range 范围扫描
* index 索引扫描，不是表示使用了索引，而是表示使用索引在进行扫描
* ALL   全表扫描

#### Extra

| Extra常见值              | 说明                                                     | 调优方法             |
| ------------------------ | -------------------------------------------------------- | -------------------- |
| Using filesort           | 需要使用额外的排序得到结果                               | 调大sort_buffer_size |
| Using index              | 优化器只需要索引就能得到结果，也就是索引覆盖             | 不需要太关注         |
| Using index condition    | 优化器使用Index Condition Pushdown优化，使用二级索引出现 |                      |
| Using index for group by | 优化器只需要使用索引就能处理group by或distinct语句       |                      |
| Using join buffer        | 优化器需要使用 join buffer，join_buffer_size             | 调大join_buffer_size |
| Using MRR                | 优化器使用MRR优化                                        | 调大read_cache_size  |
| Using temporary          | 优化器需要使用临时表                                     | 调大temporary_size   |
| Using where              | 优化器使用where过滤                                      |                      |

### 很难的一个 Explain

    (root@localhost) [dbt3]> explain SELECT 
        s_name, s_address
    FROM
        supplier,
        nation
    WHERE
        s_suppkey IN (SELECT DISTINCT
                (ps_suppkey)
            FROM
                partsupp,
                part
            WHERE
                ps_partkey = p_partkey
                    AND p_name LIKE 'orchid%'
                    AND ps_availqty > (SELECT 
                        0.5 * SUM(l_quantity)
                    FROM
                        lineitem
                    WHERE
                        l_partkey = ps_partkey
                            AND l_suppkey = ps_suppkey
                            AND l_shipdate >= '1996-01-01'
                            AND l_shipdate < DATE_ADD('1996-01-01', INTERVAL 1 YEAR)))
            AND s_nationkey = n_nationkey
            AND n_name = 'ALGERIA'
    ORDER BY s_name;
    +----+--------------------+-------------+------------+--------+----------------------------------------------------------+---------------------+---------+---------------------------------------------------+--------+----------+----------------------------------------------+
    | id | select_type        | table       | partitions | type   | possible_keys                                            | key                 | key_len | ref                                               | rows   | filtered | Extra                                        |
    +----+--------------------+-------------+------------+--------+----------------------------------------------------------+---------------------+---------+---------------------------------------------------+--------+----------+----------------------------------------------+
    |  1 | PRIMARY            | nation      | NULL       | ALL    | PRIMARY                                                  | NULL                | NULL    | NULL                                              |     25 |    10.00 | Using where; Using temporary; Using filesort |
    |  1 | PRIMARY            | supplier    | NULL       | ref    | PRIMARY,i_s_nationkey                                    | i_s_nationkey       | 5       | dbt3.nation.n_nationkey                           |    399 |   100.00 | Using index condition                        |
    |  1 | PRIMARY            | <subquery2> | NULL       | eq_ref | <auto_key>                                               | <auto_key>          | 4       | dbt3.supplier.s_suppkey                           |      1 |   100.00 | NULL                                         |
    |  2 | MATERIALIZED       | part        | NULL       | ALL    | PRIMARY                                                  | NULL                | NULL    | NULL                                              | 198303 |    11.11 | Using where                                  |
    |  2 | MATERIALIZED       | partsupp    | NULL       | ref    | PRIMARY,i_ps_partkey,i_ps_suppkey                        | PRIMARY             | 4       | dbt3.part.p_partkey                               |      3 |   100.00 | Using where                                  |
    |  3 | DEPENDENT SUBQUERY | lineitem    | NULL       | ref    | i_l_shipdate,i_l_suppkey_partkey,i_l_partkey,i_l_suppkey | i_l_suppkey_partkey | 10      | dbt3.partsupp.ps_partkey,dbt3.partsupp.ps_suppkey |      7 |    28.80 | Using where                                  |
    +----+--------------------+-------------+------------+--------+----------------------------------------------------------+---------------------+---------+---------------------------------------------------+--------+----------+----------------------------------------------+
    6 rows in set, 3 warnings (0.00 sec)