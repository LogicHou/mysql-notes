# 安装MySQL5.7

## 官方安装文档链接

https://dev.mysql.com/doc/refman/5.7/en/binary-installation.html

MySQL依赖于libaio库，需要先安装，基于Yum的通过以下命令安装：

    $> yum search libaio  # search for info
    $> yum install libaio # install library
    $> yum install openssl # install openssl

基于APT的使用以下命令安装：

    $> apt-cache search libaio # search for info
    $> apt-get install libaio1 # install library

MySQL 5.7.19以上还需要安装libnuma库：

    yum install numactl

首先确认系统上是否存在/etc/mysql/my.cnf文件，如果有将其重命名**mv /etc/mysql/my.cnf /etc/mysql/my.cnf.old**，然后将如下配置文件my.cnf放置到/etc/my.cnf

    [mysqld]
    port = 3306
    user = mysql
    datadir = /mdata/mysql_test_data
    log_error = error.log

二进制安装MySQL 5.7：

    $> groupadd mysql
    $> useradd -r -g mysql -s /bin/false mysql
    $> cd /usr/local
    $> tar zxvf /path/to/mysql-VERSION-OS.tar.gz
    $> ln -s full-path-to-mysql-VERSION-OS mysql
    $> cd mysql
    $> mkdir -p /mdata/mysql_test_data
    $> chown mysql:mysql /mdata/mysql_test_data
    $> chmod 750 /mdata/mysql_test_data
    $> bin/mysqld --initialize --user=mysql
    $> bin/mysql_ssl_rsa_setup
    $> bin/mysqld_safe --user=mysql &
    # Next command is optional
    $> cp support-files/mysql.server /etc/init.d/mysql.server
    $> chown -R root .
    # 如果执行这步就是为了安全，启动的时候用的是mysql用户，如果mysql用户对目录有权限的话还是比较危险的，比如上传一些文件到bin下面并做了一些篡改

如果碰到**bin/mysql: error while loading shared libraries: libncurses.so.5: cannot open shared object file: No such file or directory**错误，运行一下命令：

    yum install ncurses-compat-libs

执行 **bin/mysqld --initialize --user=mysql** 初始化命令后会打开/mdata/mysql_test_data/error.log文件查看密码：

    [Note] A temporary password is generated for root@localhost: Nsf/7N+wQ%<m

其中的 **Nsf/7N+wQ%<m** 就是初始化后自动分配的密码，在初始化命令时改用 **--initialize-insecure** 表示不创建密码

可以使用以下命令修改密码：

    $> bin/mysql -uroot -p
    Enter password: Nsf/7N+wQ%<m
    SET PASSWORD FOR 'root'@'localhost' = '123';

额外的配置：

    # 添加环境变量
    vim ~/.bash_profile or vim ~/.zshrc
    PATH=/usr/local/mysql/bin:$PATH:$HOME/bin
    # 修改命令行格式
    export MYSQL_PS1="(\u@\h:\p)[\d]>"
    source ~/.bash_profile

添加自动开机启动命令：

    # 添加
    $> chkconfig --add mysql.server
    # 查看
    $> chkconfig --list
    
设置免密登录(可选)： 可能会有安全问题

    #编辑/etc/my.cnf文件 加入
    [client]
    user = root
    password = 123

优化MySQL提示符显示：

    #编辑/etc/my.cnf文件 加入 u代表user h代表host d表示database
    [mysql]
    prompt = (\\u@\\h) [\\d]>\\_

## 常用命令 

    # 查看数据库
    (root@localhost) [(none)]> show databases;
    +--------------------+
    | Database           |
    +--------------------+
    | information_schema |
    | mysql              |
    | performance_schema |
    | sys                |
    +--------------------+
    4 rows in set (0.00 sec)

    # 使用某个数据库
    (root@localhost) [(none)]> use mysql;
    Reading table information for completion of table and column names
    You can turn off this feature to get a quicker startup with -A

    Database changed
    (root@localhost) [mysql]> #这里[(none)]变成了[mysql]，表示当前使用的数据库

    # 查看当前数据库下面有哪些表
    (root@localhost) [mysql]> show tables;
    +---------------------------+
    | Tables_in_mysql           |
    +---------------------------+
    | columns_priv              |
    | ...........               |
    | ...........               |
    | ...........               |
    | user                      |
    +---------------------------+
    31 rows in set (0.00 sec)

    # 查看某张表中的所有数据(显示结果的样式不太友好)
    (root@localhost) [mysql]> select * from user;
    # 只看一条(显示结果的样式不太友好)
    (root@localhost) [mysql]> select * from user limit 1;
    # 在末尾加\G可以将结果竖起来看，方便查看
    (root@localhost) [mysql]> select * from user limit 1\G

基础库简单介绍：

    (root@localhost) [mysql]> show databases;
    +--------------------+
    | Database           |
    +--------------------+
    | information_schema | <-- 信息架构表，元数据表，记录了所有元数据信息
    | mysql              | <-- 非常重要，记录了用户登录的一些信息
    | performance_schema | <-- 性能库，查看各种各样的性能，比较专业普通用户比较难看懂
    | sys                | <-- 5.7版本新加的一个库，对performance_schema库中的数据归类创建视图，使其更容易理解更容易看懂
    +--------------------+
    4 rows in set (0.00 sec)

## 安装MySQL5.6

暂停已经安装并已经运行的5.7

    $> /etc/init.d/mysql.server stop

将5.7产生的数据目录删除

    $> cd /mdata
    $> rm -rf mysql_test_data

取消5.7的link

    $> cd /usr/local/
    $> unlink mysql

安装MySQL5.6

    $> yum install perl-Data-Dumper.x86_64
    $> tar zxvf /path/to/mysql-VERSION-OS.tar.gz
    $> ln -s full-path-to-mysql-VERSION-OS mysql
    $> cd mysql
    $> scripts/mysql_install_db --user=mysql # <--------与mysql5.7不同的初始化命令
    $> bin/mysqld_safe --user=mysql &
    # 5.6安装完成默认没有密码，记得修改密码，与mysql5.7不同
    # Next command is optional
    $> cp support-files/mysql.server /etc/init.d/mysql.server

    # 修改密码 与5.7不同
    set password = password('123');

在某些版本安装时，安装完成需要将/usr/local/mysql目录权限重新设置为root，因为mysql启动的时候是用mysql用户启动的，如果此时mysql安装目录为mysql用户权限可能会有安全问题

查看默认配置参数默认值的命令：

    查看全部 mysqld --help --verbose
    查看单个 mysqld --help --verbose | grep less

## MySQL的升级，以5.6升级到5.7为例

    $> cd /usr/local/
    $> /etc/init.d/mysql.server stop
    $> unlink mysql
    $> ln -s mysql-5.7.xx-linux-glibxxxxx-x86PATH mysql
    $> /etc/init.d/mysql.server start
    $> mysql
    Welcome to the MySQL monitor.  Commands end with ; or \g.
    Your MySQL connection id is 2
    Server version: 5.7.24 MySQL Community Server (GPL) #此时查看版本号，已经发现升级到5.7版本了

    Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

    Oracle is a registered trademark of Oracle Corporation and/or its
    affiliates. Other names may be trademarks of their respective
    owners.

    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

    (root@localhost) [(none)]>

简单说一下就是先停机，改个软链，重新启动就升级完成了，非常简单

但是这个时候查看error.log文件，发现其中有很多如下的ERROR提示：

    2018-12-06T07:44:21.415502Z 0 [ERROR] Column count of performance_schema.threads is wrong. Expected 17, found 14. Created with MySQL 50642, now running 50724. Please use mysql_upgrade to fix this error.

日志中说得很清楚了**Created with MySQL 50642, now running 50724.**跨版本升级导致了一些问题，需要使用**mysql_upgrade**命令进行修复

    $> cd /mdata/mysql_test_data
    $>  mysql_upgrade -s 
    The --upgrade-system-tables option was used, databases won't be touched.
    Checking if update is needed.
    Checking server version.
    Running queries to upgrade MySQL server.
    Upgrading the sys schema.
    Upgrade process completed successfully.
    Checking if update is needed.

    # -s 的意思是只更新系统表(元数据表)而不要更新我的数据，数据的话5.7肯定是兼容5.6的不需要对数据进行升级，因为如果数据很大几百G的数据也对其进行升级，升级可能就会重建代价就非常大

MySQL称这种方式为In-Place Upgrade(原地更新)，升级速度非常快，但是有个缺点就是需要停机

升级MySQL数据的时候最好先进行备份，关闭mysql，备份完整的数据目录，然后再进行升级

一般在线上进行升级，都有主从，可以先从机升级，从机升级完成后做一个飘移把IP飘过来，再做主机升级

**此方法支持从5.0升级到5.7**

相关文档

https://dev.mysql.com/doc/refman/5.7/en/upgrade-binary-package.html
https://mysqlserverteam.com/upgrading-directly-from-mysql-5-0-to-5-7-using-an-in-place-upgrade/


这里推荐一个MySQL开发人员的网站 http://mysqlserverteam.com/ 文章质量比较高

## MySQL 5.7， 5.6不同之处

1. 初始化数据库的命令不同

2. 5.6初始化后没有密码，5.7的密码初始化后可以从error.log文件中查看到

## 当前配置文件

    [client]
    user = root
    password = 123

    [mysql]
    prompt = (\\u@\\h) [\\d]>\\_

    [mysqld]
    port = 3306
    user = mysql
    datadir = /mdata/mysql_test_data
    log_error = error.log