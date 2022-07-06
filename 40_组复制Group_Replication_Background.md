# 组复制 Group Replication Background

能够保证集群间的数据不丢失

无损复制也能保证数据不丢失但是实现的方式是有一些不一样的

复制还是通过一个叫做主从的机制来进行保证，但是主从这种机制本身不能解决两个问题

MySQL 5.7 中的半同步复制是无损的，数据是一致的

MySQL 5.7 解决了一致性问题但是没有解决高可用的问题，因为复制只是用来做日志的传输，集群虽然搭建完了但是如果主机宕机了，并不会进行选主，所以复制只是一个部分的高可用，故障转移不能通过集群自己去解决的，需要通过外部的依赖的软件去解决，semi-sync 没有解决集群内高可用的问题

但是这些外部工具都存在一个问题就是脑裂，一旦存在脑裂的话可能各个节点都可以进行数据的写入，这时候就是个灾难了

MySQL 无损复制是单写的，集群的 slave 可以有很多个，但是同一时间写入的节点只能是 master

|        | semi | MGR |
| ------ | ---- | --- |
| 一致性 | Yes  | Yes |
| 高可用 | No   | Yes |
| 多写   | No   | Yes |

MGR 通过 Paxos 实现一致性

MGR 支持多写，默认的模式是单主(singal master)的，还有另外一种模式是多主的，多主模式下所有其他的节点都可以进行写入，这时候 GR 是通过原子广播解决写入冲突的问题的

所谓原子广播就是每次更新事务提交的时候会有一个叫 writeset （其实就是binlog），如果是单写的话会蒋 writeset 发送到各个节点上面，如果是多写的话原子广播会通过一定的顺序广播到所有的 master 节点上

## 搭建 MGR 环境

### 基本环境

|            | mysql1                                 | mysql2                                 | mysql3                                 |
| ---------- | -------------------------------------- | -------------------------------------- | -------------------------------------- |
| MySQL版本  | 5.7.37                                 | 5.7.37                                 | 5.7.37                                 |
| 操作系统   | Docker Centos 8.2.2004                 | Docker Centos 8.2.2004                 | Docker Centos 8.2.2004                 |
| 服务器IP   | 172.20.0.10                            | 172.20.0.11                            | 172.20.0.12                            |
| 端口       | 对外服务端口-3006<br>MGR通讯端口-33061 | 对外服务端口-3006<br>MGR通讯端口-33061 | 对外服务端口-3006<br>MGR通讯端口-33061 |
| 服务器配置 | ...                                    | ...                                    | ...                                    |

### 安装配置

1. 安装插件

    --安装mgr插件
    INSTALL PLUGIN group_replication SONAME 'group_replication.so';
    --检查
    show plugins

也可以在配置文件 my.cnf 中加入：

    plugin-dir=/usr/local/mysql/lib/plugin/
    plugin_load_add='group_replication.so'

2. 配置 hosts： 

也可以直接用 IP，直接用 IP 的话跳过这步即可

    vim /etc/hosts
    172.20.0.10 mysql1
    172.20.0.11 mysql2
    172.20.0.12 mysql3

3. 修改 auto.cnf：

如果三台 MySQL 目录的 server-uuid 一致，需进行修改

    vim /mysql/data/auto.cnf
    --主库的server-uuid的末尾建议设置成0001，依次类推，这样方便识别
    [auto]
    server-uuid=76ee3e3d-f99d-11ec-be51-0242ac110001

4. 配置 my.cnf：

    #Group Replication
    # install plugin
    plugin-dir=/usr/local/mysql/lib/plugin/
    plugin_load_add='group_replication.so'

    binlog_checksum = NONE
    transaction_write_set_extraction = XXHASH64
    slave_preserve_commit_order = true
    loose-group_replication_group_name = 'aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa'
    loose-group_replication_start_on_boot = off
    loose-group_replication_local_address = '172.20.0.10:33061'
    loose-group_replication_group_seeds ='172.20.0.10:33061, 172.20.0.11:33061, 172.20.0.12:33061'
    loose-group_replication_bootstrap_group = off
    # single primary
    loose-group_replication_single_primary_mode = on
    # multi primary
    #loose-group_replication_single_primary_mode = off
    #loose-group_replication_enforce_update_everywhere_checks=true

参数说明

    binlog_checksum                           这个参数必须设置为NONE，不做校验
    transaction_write_set_extraction          writeset 算法
    loose-group_replication_group_name        组名字，可以通过 select uuid(); 自己生成，然后复制过来
    loose-group_replication_start_on_boot     启动MySQL的时候是否执行 START GROUP_REPLICATION 命令，第一次配置的时候设成off
    loose-group_replication_local_address     除了3307端口以外对应的GR的端口，从机上记得改成从机的IP+端口
    loose-group_replication_group_seeds       GR里面的通信节点+端口有哪些，端口配的是GR的通信端口
    loose-group_replication_bootstrap_group   一定要设置为off

5. 启动MGR-主库

    --配置复制用户
    set sql_log_bin=off;  # 操作不记录到二进制日志
    GRANT REPLICATION SLAVE ON *.* TO 'repl_user'@'%' IDENTIFIED BY '123456';
    set sql_log_bin=on;
    
    --建立channel
    CHANGE MASTER TO MASTER_USER='repl_user', MASTER_PASSWORD='123456' FOR CHANNEL 'group_replication_recovery';

    --第一个节点启动时，需要设置bootstrap_group
    --第一个节点一定要把这里设成ON再去START GROUP_REPLICATION，添加后续的节点则不需要执行这个语句
    SET GLOBAL group_replication_bootstrap_group = ON;

    --启动MGR
    START GROUP_REPLICATION;

    --取消bootstrap_group
    SET GLOBAL group_replication_bootstrap_group = OFF;

    --查看当前MGR成员信息
    SELECT * FROM performance_schema.replication_group_members;
    SELECT * FROM performance_schema.replication_group_member_stats;
    SELECT * FROM performance_schema.replication_connection_status;
    SELECT * FROM performance_schema.replication_applier_status;

6. 启动MGR-从库

    --配置复制用户，每一个从库上都需要创建
    set sql_log_bin=off;
    GRANT REPLICATION SLAVE ON *.* TO 'repl_user'@'%' IDENTIFIED BY '123456';
    set sql_log_bin=on;
    
    --建立channel
    CHANGE MASTER TO MASTER_USER='repl_user', MASTER_PASSWORD='123456' FOR CHANNEL 'group_replication_recovery';

    --启动MGR
    START GROUP_REPLICATION;

    -- 查看GTID执行到的位置
    SELECT @@global.gtid_executed\G

7. 查看MGR

    --查看当前MGR成员信息
    SELECT * FROM performance_schema.replication_group_members;
    --查看只读参数
    show variables like '%read_only%';

        (root@localhost:mysql.sock) [test]> SELECT * FROM performance_schema.replication_group_members;
        +---------------------------+--------------------------------------+-------------+-------------+--------------+
        | CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST | MEMBER_PORT | MEMBER_STATE |
        +---------------------------+--------------------------------------+-------------+-------------+--------------+
        | group_replication_applier | 76ee3e3d-f99d-11ec-be51-0242ac110001 | mysql1      |        3306 | ONLINE       |
        | group_replication_applier | 76ee3e3d-f99d-11ec-be51-0242ac110002 | mysql2      |        3306 | ONLINE       |
        | group_replication_applier | 76ee3e3d-f99d-11ec-be51-0242ac110003 | mysql3      |        3306 | ONLINE       |
        +---------------------------+--------------------------------------+-------------+-------------+--------------+
        3 rows in set (0.00 sec)

### 基础维护

1. 启动MGR

START GROUP_REPLICATION;

2. 停止MGR

STOP GROUP_REPLICATION;

### 注意事项

开启 GR 后，表必须带上主键，否则插入数据会报错：

    (root@localhost:mysql.sock) [test]> create table x(a int, b int, c int);
    Query OK, 0 rows affected (0.02 sec)

    (root@localhost:mysql.sock) [test]> create table y(a int primary key, b int, c int);
    Query OK, 0 rows affected (0.02 sec)

    (root@localhost:mysql.sock) [test]> insert into x values(1,2,3);
    ERROR 3098 (HY000): The table does not comply with the requirements by an external plugin.
    (root@localhost:mysql.sock) [test]> select * from x;
    Empty set (0.00 sec)

    (root@localhost:mysql.sock) [test]> insert into y values(1,2,3);
    Query OK, 1 row affected (0.01 sec)

如果是多写的模式，提交事务的时候如果检测到冲突或者脑裂就可能会提交失败，所以业务层面的代码要把 commit 部分写到 try 里面

### 其他参数

group_replication_flow_control_mode = DISABLED      # 关闭流控制

loose_group_replication_compression_threshold= 10   # 用于压缩的线程数