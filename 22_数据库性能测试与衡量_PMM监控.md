# 数据库性能测试与衡量 PMM监控

## for update 语法加锁操作

这个语法最重要的作用就是用来对数据进行加锁


(root@localhost) [tpcc1000]> use tpcc1000;
Database changed
(root@localhost) [tpcc1000]> select * from stock where s_w_id = 1 and s_i_id = 1 limit 1\G  # 库存表
*************************** 1. row ***************************
      s_i_id: 1
      s_w_id: 1
  s_quantity: 69
   s_dist_01: BnRQhk1sORwbUb8aB8Vn1zI0
   s_dist_02: 8X8R6Yarx0WfzO3RJevukU5t
   s_dist_03: tsw9svIQ2jUYKLWIrBIDDhWp
   s_dist_04: Oa4SRzJJuSBniZpQ3OiMu0B3
   s_dist_05: NGxt6YGgai7a1OSRXFA75Y8R
   s_dist_06: TJGeMV1dSth09NbrSBqobHLq
   s_dist_07: StxZKD2bSeyxeZPXGQebzT18
   s_dist_08: pjaAZ0bRxAj9ALboHWSqvx4J
   s_dist_09: dI2cQLR7wZfuA6rIp0S8ZFP5
   s_dist_10: D1bFIAsZDSxYt5SRkMZoCsdC
       s_ytd: 0
 s_order_cnt: 0
s_remote_cnt: 0
      s_data: mkV9pBNomc7wTg7vbgng6CQRlEdbqnuaXPhNY49Kf
1 row in set (0.00 sec)

既然是库存如果不锁住就更新，可能会造成好几个用户同时更新库存的情况，这时候可以用 for update 先锁住记录，然后就可以做减扣除的操作了：

(root@localhost) [tpcc1000]> select * from stock where s_w_id = 1 and s_i_id = 1 for update;

## mysqlslap 命令

使用这个命令可以使用自己编写的脚本针对某个业务场景进行测试，脚本通过 --query 参数调用即可：

    $ mysqlslap --query=stock.sql -c 1024 --number-of-queries=1000000000
    # --query 所要使用的脚本
    # -c      并发数
    # --number-of-queries 要执行的SQL的次数

mysqlslap 有一个问题就是运行后并没有任何的输出，可以配合 mysqladmin 观察：

    $ mysqladmin extended-status -i 1 -r | grep -i Questions

或者通过监控平台开监控

## mysql 线程池

(root@localhost) [(none)]> show variables like 'thread_hand%';
+-----------------+---------------------------+
| Variable_name   | Value                     |
+-----------------+---------------------------+
| thread_handling | one-thread-per-connection |
+-----------------+---------------------------+
1 row in set (0.00 sec)

MySQL 默认用的是一个叫 one-thread-per-connection 的值，表示一个连接连过来就分配给它一个线程，如果有 1000 个连接此时就会对应 1000 个线程，用这种模式对库存进行更新竞争就会很大，因为 update 操作此时都是串行的

### 配置成 pool-of-threads 

MySQL5.7 用不了这个参数，需要更换为 Percona 数据库

不管多少个连接连过来，最多只分配池子里最大数量的线程，比如如果池子最多只有 16 个线程，就算有 1024 个连接连过来，同时在运行的最多也只有 16 个线程

## 整理测试结果数据

    # 存入文件
    $ sysbench ............ > ret.txt
    # 抓取想要的数据
    $ cat ret.txt | grep reads | aws {'print $8'} | tr -d ,

整理出来的数据可以用 Excel 或者在线可视化工具，画一个折线图或者其他可视化图表

## 一个简单的集成测试脚本

    mkdir -p /mdata/testsysbench_date +Y'
    mysql -e "show variables " > /mdata/testsysbench_date +Y'/variables.log 

    # cpu.sh 
    sar -P ALL 1 /mdata/testsysbench_date +Y'/cpu.log 

    # status.sh 
    mysqladmin extended-status -r -i 1 /mdata/testsysbench_date +Y'/status.log

    # sysbench.sh 
    sysbench --test=/root/sysbench/tests/include/oltp_legacy/select.lua --oltp-tables-count=4 --oltp-table-size=1000000 --oltp-dist-res=95 --mysql-host=127.0.0.1 --mysql-user=root --mysql-password=Abc123__ --threads=$1 --max-requests=0 --time=$2 --report-interval=3 run > /mdata/testsysbench_date +Y'/sysbench.log 

    sleep $2 
    p1='ps -ef | grep sar | grep -v "grep sar" | awk '{print $2}'
    ki11-9 $p1 
    p2='ps -ef | grep mysqladmin | grep -v "grep mysqladmin" | awk '{print $2}'
    ki11-9 $p2 

# PMM监控

## PMM 安装

参考文档 https://docs.percona.com/percona-monitoring-and-management/setting-up/index.html

### 安装服务端

    $ docker pull percona/pmm-server:2
    $ docker create --volume /srv --name pmm-data percona/pmm-server:2 /bin/true
    $ docker run --detach --restart always --publish 443:443 --publish 80:80 --volumes-from pmm-data --name pmm-server percona/pmm-server:2
    # 修改管理密码
    $ docker exec -t pmm-server bash -c 'grafana-cli --homepath /usr/share/grafana --configOverrides cfg:default.paths.data=/srv/grafana admin reset-admin-password newpass'

安装好后访问 https://localhost:443

如果不想使用 80 端口，容器启动后登录进去 shell ，去改 ngxin 的配置文件或许也是可以的
    
### 安装客户端

通过 yum 安装 client：

    $ yum install -y https://repo.percona.com/yum/percona-release-latest.noarch.rpm
    $ yum install -y pmm2-client
    $ pmm-admin --version
    ProjectName: pmm-admin
    Version: 2.28.0
    PMMVersion: 2.28.0
    Timestamp: 2022-05-10 16:55:12 (UTC)
    FullCommit: ec40520c9db85e9aa5c0d4feaa35990ebca74fb9

如果监控节点使用的是 doker 容器，则需要通过容器内部 IP 地址进行配置，那先首先获取 PMM server 容器的 ip 地址：

    ❯ docker exec -it pmm-server ip addr
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
          valid_lft forever preferred_lft forever
    2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
        link/ipip 0.0.0.0 brd 0.0.0.0
    3: sit0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
        link/sit 0.0.0.0 brd 0.0.0.0
    12: eth0@if13: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
        link/ether 02:42:ac:11:00:03 brd ff:ff:ff:ff:ff:ff
        inet 172.17.0.3/16 brd 172.17.255.255 scope global eth0  <--这里的172.17.0.3就是容器内部IP
          valid_lft forever preferred_lft forever

注册监控节点：

    $ pmm-admin config --server-insecure-tls --server-url=https://admin:Abc123__@172.17.0.3:443

为 PMM 新建一个数据库账户

On MySQL 8.0：

    CREATE USER 'pmm'@'127.0.0.1' IDENTIFIED BY 'Abc123__' WITH MAX_USER_CONNECTIONS 10;
    GRANT SELECT, PROCESS, REPLICATION CLIENT, RELOAD, BACKUP_ADMIN ON *.* TO 'pmm'@'127.0.0.1';


On MySQL 5.7：

    CREATE USER 'pmm'@'127.0.0.1' IDENTIFIED BY 'Abc123__' WITH MAX_USER_CONNECTIONS 10;
    GRANT SELECT, PROCESS, REPLICATION CLIENT, RELOAD ON *.* TO 'pmm'@'127.0.0.1';

添加服务，这里使用的是 perfschema：

    $ pmm-admin add mysql --query-source=perfschema --username=pmm --password=Abc123__ --service-name=MYSQL_NODE --host=127.0.0.1 --port=3306

查看服务：

    $ pmm-admin list
    Service type        Service name        Address and port        Service ID
    MySQL               MYSQL_NODE          127.0.0.1:3306          /service_id/cf20d83e-1d6d-4790-94cb-9342b8400571

    Agent type                    Status           Metrics Mode        Agent ID                                              Service ID
    pmm_agent                     Connected                            /agent_id/96fae83a-29ae-4189-897a-44008d2db60f
    node_exporter                 Waiting          push                /agent_id/35e55e01-97c5-41f2-9917-fee336cdfd23
    mysqld_exporter               Running          push                /agent_id/2b727f47-6425-422a-ae15-69083960e5b9        /service_id/cf20d83e-1d6d-4790-94cb-9342b8400571
    mysql_perfschema_agent        Running                              /agent_id/c06c4730-b81e-4bdb-8635-0c2812fb56f9        /service_id/cf20d83e-1d6d-4790-94cb-9342b8400571
    vmagent                       Starting         push                /agent_id/c74af69c-b17d-42db-b7ec-1bb88d0954b2

配置完后后可以使用链接 https://127.0.0.1/graph/login 进行登录，默认账号密码都是 amdin，然后就可以看到图表上开始有数据在跑了

如果配置过程中遇到问题可以去 /var/log/ 目录下查看 pmm- 开头的日志文件