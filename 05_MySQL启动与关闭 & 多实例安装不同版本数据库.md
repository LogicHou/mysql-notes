# 多实例安装

| MySQL1                  | MySQL2                  | MySQL3                  | MySQL3                  |
| :---------------------- | :---------------------- | :---------------------- | :---------------------- |
| port:3306               | port:3307               | port:3308               | port:3308               |
| datadir:/data1          | datadir:/data2          | datadir:/data3          | datadir:/data3          |
| socket:/tmp/mysql.sock1 | socket:/tmp/mysql.sock2 | socket:/tmp/mysql.sock3 | socket:/tmp/mysql.sock3 |

* 一台服务器上安装多个MySQL实例
* 充分利用硬件资源
* 通过mysqld_multi程序即可

## 安装单机多实例

修改配置文件：

    shell> vim /etc/my.cnf
    # 加入
    [mysqld1]
    port = 3307
    datadir = /mdata/data1
    socket = /tmp/mysql.sock1

初始化数据目录

    shell> mysqld --initialize --datadir=/mdata/data1

配置文件中配置mysqld_multi参数，告诉MySQL启动的时候、关闭的时候调用的程序是哪些，日志是哪些

    [mysqld_multi]
    mysqld=/usr/local/mysql/bin/mysqld_safe
    mysqladmin=/usr/local/mysql/bin/mysqladmin
    log=/usr/local/mysql/mysqld_multi.log

此时使用mysqld_multi命令，它会去读取配置文件，会发现有一个叫mysqld1的标签未运行

    shell> mysqld_multi report
    Reporting MySQL servers
    MySQL server from group: mysqld1 is not running

启动mysql1

    shell> mysqld_multi start 1
    shell> mysqld_multi report
    Reporting MySQL servers
    MySQL server from group: mysqld1 is running <-- 表示mysqld1已经开开始运行

    # 查看端口占用情况
    shell> netstat -ntl
    Active Internet connections (only servers)
    Proto Recv-Q Send-Q Local Address           Foreign Address         State
    tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN
    tcp6       0      0 :::3306                 :::*                    LISTEN
    tcp6       0      0 :::3307                 :::*                    LISTEN


通过socket连接mysql1

    查看密码
    shell> cat /mdata/data1/error.log | grep temp
    2020-01-02T06:20:38.237184Z 1 [Note] A temporary password is generated for root@localhost: jiRJgtvf%2to
    2020-01-02T06:21:10.241856Z 0 [Note] InnoDB: Creating shared tablespace for temporary tables
    shell> mysql -S /tmp/mysql.sock1 -p"jiRJgtvf%2to"

    # 重设密码
    (root@localhost) [(none)]> set password = 'Abc123__';
    Query OK, 0 rows affected (0.00 sec)

配置第二个实例

    # my.cnf中添加
    [mysqld2]
    port = 3308
    datadir = /mdata/data2
    socket = /tmp/mysql.sock2

    # 初始化数据目录
    shell> mysqld --initialize --datadir=/mdata/data2
    shell> mysqld_multi report
    Reporting MySQL servers
    MySQL server from group: mysqld1 is running
    MySQL server from group: mysqld2 is not running
    shell> mysqld_multi start 2
    shell> mysqld_multi report
    Reporting MySQL servers
    MySQL server from group: mysqld1 is running
    MySQL server from group: mysqld2 is running

    # 重设密码
    shell> cat /mdata/data2/error.log | grep temp
    2020-01-02T06:20:38.237184Z 1 [Note] A temporary password is generated for root@localhost: ,p!)yKsBi3ek
    2020-01-02T06:21:10.241856Z 0 [Note] InnoDB: Creating shared tablespace for temporary tables
    shell> mysql -S /tmp/mysql.sock2 -p",p\!)yKsBi3ek"
    
    (root@localhost) [(none)]> set password = 'Abc123__';
    Query OK, 0 rows affected (0.00 sec)

如果mysqld_multi stop 命令无效，可以尝试使用下面的命令：

    shell> mysqladmin -h127.0.0.1 -P3307 -uroot -p shutdown
    # 密码为各个子实例的密码

    导致这个问题的原因是如果在[client]里配置了用户和密码，实例会调用[client]里的这个用户名和密码，所以需要保持每个实例的密码需要和[client]里的密码保持保持一致

    另外用户名和密码也可以在[mysqld_multi]下配置，但是参数名称为user和pass，而在[client]中密码参数名则为password这点需要注意，一般推荐是在[client]中配置用户名密码即可

    https://dev.mysql.com/doc/mysql-startstop-excerpt/5.7/en/mysqld-multi.html
    Make sure that the MySQL account used for stopping the mysqld servers (with the mysqladmin program) has the same user name and password for each server.

注意：实例的参数配置会自动继承[mysqld]下面的参数，如果实例配置中有相同的配置项就会进行替换

添加mysqld_multi.server服务

    shell> cd /usr/local/mysql
    shell> cp support-files/mysqld_multi.server /etc/init.d

    # 通过mysqld_multi.server停止
    shell> /etc/init.d/mysqld_multi.server stop
    shell> mysqld_multi report
    Reporting MySQL servers
    MySQL server from group: mysqld1 is not running
    MySQL server from group: mysqld2 is not running
    # 同样可以
    shell> /etc/init.d/mysqld_multi.server start 1
    shell> /etc/init.d/mysqld_multi.server start 2

## 当前配置文件

    [client]
    user = root
    password = Abc123__
    socket = /tmp/mysql.sock

    [mysql]
    prompt = (\\u@\\h) [\\d]>\\_

    [mysqld]
    port = 3306
    user = mysql
    datadir = /mdata/mysql_test_data
    log_error = error.log
    plugin-load=validate_password.so

    [mysqld_multi]
    mysqld=/usr/local/mysql/bin/mysqld_safe
    mysqladmin=/usr/local/mysql/bin/mysqladmin
    log=/usr/local/mysql/mysqld_multi.log

    [mysqld100]
    port = 3306
    datadir = /mdata/mysql_test_data
    socket = /tmp/mysql.sock

    [mysqld1]
    port = 3307
    datadir = /mdata/data1
    socket = /tmp/mysql.sock1

    [mysqld2]
    port = 3308
    datadir = /mdata/data2
    socket = /tmp/mysql.sock2

# 多实例安装不同版本数据库

## MySQL的启动

mysqld 和mysqld_safe启动的区别

其中mysqld是个二进制文件，而mysqld_safe是一个shell脚本

mysqld_safe是一个守护进程，会定期检测mysqld的进程是否存在，如果不存在的话就会把它给拉起来

## 忘记密码的处理方式

在my.cnf中配置skip-grant-tables参数：

    [mysqld]
    ......
    ......
    ......
    skip-grant-tables
    # 意为启动msyql但是忽略授权表

为忘记的账户修改密码：

    (root@localhost) [(none)]> use mysql
    (root@localhost) [mysql]> select user,host,authentication_string from user;
    +---------------+-----------+-------------------------------------------+
    | user          | host      | authentication_string                     |
    +---------------+-----------+-------------------------------------------+
    | root          | localhost | *F16C278E7B959471C8164A38C9C94DDC7C15D40F |
    | mysql.session | localhost | *THISISNOTAVALIDPASSWORDTHATCANBEUSEDHERE |
    | mysql.sys     | localhost | *THISISNOTAVALIDPASSWORDTHATCANBEUSEDHERE |
    | hou           | %         | *F16C278E7B959471C8164A38C9C94DDC7C15D40F |
    | amy           | %         |                                           |
    +---------------+-----------+-------------------------------------------+
    5 rows in set (0.00 sec)

    (root@localhost) [mysql]> update user set authentication_string = password('Abc123__') where user = 'root' and host = 'localhost';
    Query OK, 0 rows affected, 1 warning (0.00 sec)
    Rows matched: 1  Changed: 0  Warnings: 1
    # 注意这里必须使用password函数生成密码而不是写入密码

    # 刷新权限
    (root@localhost) [mysql]> flush privileges;
    Query OK, 0 rows affected (0.00 sec)

    # 从my.cnf中去除或者注释掉skip-grant-tables参数
    # 重启MySQL
    shell> /etc/init.d/mysql.server restart
    Shutting down MySQL..                                      [  OK  ]
    Starting MySQL.                                            [  OK  ]

## 安装多个版本不同的数据库

添加mysql4配置项：

    [mysqld4]
    server-id = 14 <-- 暂时无用，后面讲复制内容的时候会用到
    innodb_buffer_pool_size = 32M
    basedir = /usr/local/mysql56/ <-- MySQL安装路径
    datadir = /mdata/data4
    socket = /tmp/mysql.sock4
    port = 3356

初始化mysql56数据目录

    shell> cd /usr/local
    shell> tar zxvf /path/to/mysql-VERSION-OS.tar.gz
    shell> ln -s full-path-to-mysql-5.6.x-OS mysql56
    shell> cd /usr/local/mysql56
    shell> scripts/mysql_install_db --user=mysql --datadir=/mdata/data4
    shell> mysqld_multi start 4
    shell> mysql -S /tmp/mysql.sock4 -p
    shell> set password = password('Abc123__');

    !! 如果安装后无法用mysqld_multi启动，尝试删除数据目录，然后禁用my.cnf中的
    #plugin-load=validate_password.so
    #default_password_lifetime = 0
    然后重新初始化

添加mysql80配置项：

    [mysqld80]
    server-id = 18 <-- 暂时无用，后面讲复制内容的时候会用到
    innodb_buffer_pool_size = 32M
    basedir = /usr/local/mysql80/ <-- MySQL路径
    datadir = /mdata/data80
    socket = /tmp/mysql.sock80
    port = 3380

初始化mysql80数据目录

    shell> cd /usr/local
    shell> tar xvf /path/to/mysql-VERSION-OS.tar.xz
    shell> ln -s full-path-to-mysql-8.0.x-OS mysql80
    shell> cd /usr/local/mysql80
    shell> bin/mysqld --initialize --datadir=/mdata/data80
    shell> mysqld_multi start 80
    shell> cat /mdata/data80/error.log | grep temp
    shell> mysql -S /tmp/mysql.sock80 -p'******'
    shell> set password = 'Abc123__';

由此可以证实5.7版本的mysqld_safe可以成功启动5.6，5.7，8.0的版本

    shell> mysqld_multi report
    Reporting MySQL servers
    MySQL server from group: mysqld1 is running
    MySQL server from group: mysqld2 is running
    MySQL server from group: mysqld4 is running
    MySQL server from group: mysqld80 is running


密码过期功能（MySQL5.6版本开始）

    (root@localhost) [(none)]> show variables like 'default%';
    +-------------------------------+-----------------------+
    | Variable_name                 | Value                 |
    +-------------------------------+-----------------------+
    | default_authentication_plugin | mysql_native_password |
    | default_password_lifetime     | 0                     |
    | default_storage_engine        | InnoDB                |
    | default_tmp_storage_engine    | InnoDB                |
    | default_week_format           | 0                     |
    +-------------------------------+-----------------------+
    5 rows in set (0.00 sec)

    这里default_password_lifetime如果不为0，时间到期后密码就会过期失效，可能对线上运行的程序造成影响。注意在某些MySQL5.7版本中这个值默认为360推荐改为0，可以在配置文件中就直接将这个值设为0，避免5.7密码有坑的问题

    shell> vim /etc/my.cnf

    [mysqld]
    ......
    ......
    ......
    default_password_lifetime = 0

## MySQL8.0 用户角色权限的使用

    shell> mysqld_multi start 80
    shell> mysql -S /tmp/mysql.sock80 -p'Abc123__'

相关命令

    -- 创建Role
    create role senior_dba, app_dev;
    grant all on *.* to senior_dba with grant option;
    grant select,insert,update,delete on wp.* to app_dev;

    -- 用户与角色绑定
    create user 'tom'@'192.168.1.%' identified by 'Abc123__';
    grant senior_dba to tom@'192.168.1.%';

    -- 显示用户权限
    show grants for 'tom'@'192.168.1.%';
    show grants for 'tom'@'192.168.1.%' using senior_dba;
    show grants for 'tom'@'192.168.1.%' using senior_dba\G

    -- 删除角色
    drop role senior_dba;

## 当前配置文件

    [client]
    user = root
    password = Abc123__
    socket = /tmp/mysql.sock

    [mysql]
    prompt = (\\u@\\h) [\\d]>\\_

    [mysqld]
    server-id = 1
    port = 3306
    user = mysql
    datadir = /mdata/mysql_test_data
    log_error = error.log
    plugin-load=validate_password.so
    default_password_lifetime = 0
    #skip-grant-tables

    [mysqld_multi]
    mysqld=/usr/local/mysql/bin/mysqld_safe
    mysqladmin=/usr/local/mysql/bin/mysqladmin
    log=/usr/local/mysql/mysqld_multi.log

    [mysqld1]
    server-id = 11
    innodb_buffer_pool_size = 32M
    port = 3307
    datadir = /mdata/data1
    socket = /tmp/mysql.sock1

    [mysqld2]
    server-id = 12
    innodb_buffer_pool_size = 32M
    port = 3308
    basedir = /usr/local/mysql/
    datadir = /mdata/data2
    socket = /tmp/mysql.sock2

    [mysqld4]
    server-id = 14
    innodb_buffer_pool_size = 32M
    basedir = /usr/local/mysql56/
    datadir = /mdata/data4
    socket = /tmp/mysql.sock4
    port = 3356

    [mysqld80]
    server-id = 18
    innodb_buffer_pool_size = 32M
    basedir = /usr/local/mysql80/
    datadir = /mdata/data80
    socket = /tmp/mysql.sock80
    port = 3380