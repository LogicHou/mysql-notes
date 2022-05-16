# 加密连接与密码插件

## 连接MySQL实例

* 通过本地socket进行连接
  * mysql -S /tmp/mysql.sock -u root -p
* 通过TCP/IP协议远程连接
  * mysql -h 127.0.0.1 -P 3306 -u root -p
* 通过配置my.cnf免密码输入

数据库配置项

    [client]
    user = root
    password = 123
    socket = /tmp/mysql.sock

查看socket文件位置

    (root@localhost) [(none)]> show variables like 'socket%';
    +---------------+-----------------+
    | Variable_name | Value           |
    +---------------+-----------------+
    | socket        | /tmp/mysql.sock |
    +---------------+-----------------+
    1 row in set (0.00 sec)

这里连接的时候没有在配置文件中加入socket = /tmp/mysql.sock，在命令行连接MySQL的时候默认加上参数"-S /tmp/mysql.sock"

通过TCP/IP进行连接

    mysql -h127.0.0.1 -uroot -p123

MySQL状态信息中提供了当前连接方式的相关信息

    (root@localhost) [(none)]> status 或者\s
    --------------
    mysql  Ver 14.14 Distrib 5.7.25, for linux-glibc2.12 (x86_64) using  EditLine wrapper

    Connection id:          5
    Current database:
    Current user:           root@localhost
    SSL:                    Not in use  <--- 没有使用SSL
    Current pager:          stdout
    Using outfile:          ''
    Using delimiter:        ;
    Server version:         5.7.25 MySQL Community Server (GPL)
    Protocol version:       10
    Connection:             Localhost via UNIX socket  <--- 连接方式，这里是通过socket方式连接
    Server characterset:    latin1
    Db     characterset:    latin1
    Client characterset:    latin1
    Conn.  characterset:    latin1
    UNIX socket:            /tmp/mysql.sock
    Uptime:                 10 min 20 sec

    Threads: 1  Questions: 17  Slow queries: 0  Opens: 106  Flush tables: 1  Open tables: 99  Queries per second avg: 0.027
    --------------

SSL方式连接，如何查看见上一部分内容，显示并没有启用SSL

通过查看参数方式查看是否启用了SSL

    (root@localhost) [(none)]> show variables like '%ssl%';
    +---------------+-----------------+
    | Variable_name | Value           |
    +---------------+-----------------+
    | have_openssl  | DISABLED        |
    | have_ssl      | DISABLED        |
    | ssl_ca        | ca.pem          |
    | ssl_capath    |                 |
    | ssl_cert      | server-cert.pem |
    | ssl_cipher    |                 |
    | ssl_crl       |                 |
    | ssl_crlpath   |                 |
    | ssl_key       | server-key.pem  |
    +---------------+-----------------+
    9 rows in set (0.00 sec)

如果没有配置SSL，在error.log文件中也会有Warning的提示

    cat /mdata/mysql_test_data/error.log
    2017-03-22T07:41:53.975045Z 0 [Warning] Failed to set up SSL because of the following SSL library error: Unable to get private key

为MySQL添加SSL支持（如果安装的时候没有添加SSL支持）

    shell> mysql_ssl_rsa_setup
    shell> cd /mdata/mysql_test_data/
    shell> ll *.pem # 会发现在数据目录下生成了如下类似的公密钥文件
    -rw------- 1 root root 1675 Mar 21 03:29 ca-key.pem
    -rw-r--r-- 1 root root 1107 Mar 21 03:29 ca.pem
    -rw-r--r-- 1 root root 1107 Mar 21 03:29 client-cert.pem
    -rw------- 1 root root 1679 Mar 21 03:29 client-key.pem
    -rw------- 1 root root 1675 Mar 21 03:29 private_key.pem
    -rw-r--r-- 1 root root  451 Mar 21 03:29 public_key.pem
    -rw-r--r-- 1 root root 1107 Mar 21 03:29 server-cert.pem
    -rw------- 1 root root 1675 Mar 21 03:29 server-key.pem

    shell> chown mysql:mysql *.pem # 修改权限
    shell> /etc/init.d/mysql.server restart
    Shutting down MySQL..... SUCCESS!
    Starting MySQL. SUCCESS!

    shell> mysql

    (root@localhost) [(none)]> show variables like '%ssl%';
    +---------------+-----------------+
    | Variable_name | Value           |
    +---------------+-----------------+
    | have_openssl  | YES             |  <-- 已启用SSL
    | have_ssl      | YES             |  <-- 已启用SSL
    | ssl_ca        | ca.pem          |
    | ssl_capath    |                 |
    | ssl_cert      | server-cert.pem |  <-- 已加载密钥文件
    | ssl_cipher    |                 |
    | ssl_crl       |                 |
    | ssl_crlpath   |                 |
    | ssl_key       | server-key.pem  |  <-- 已加载密钥文件
    +---------------+-----------------+
    9 rows in set (0.00 sec)

    shell> mysql -h127.0.0.1 -uroot -p # 重新登录，注意要用TCP\IP方式登录
    (root@127.0.0.1) [(none)]> \s
    --------------
    mysql  Ver 14.14 Distrib 5.7.25, for linux-glibc2.12 (x86_64) using  EditLine wrapper

    Connection id:          3
    Current database:
    Current user:           root@localhost
    SSL:                    Cipher in use is DHE-RSA-AES256-SHA  <--- 已启用SSL
    Current pager:          stdout
    Using outfile:          ''
    Using delimiter:        ;
    Server version:         5.7.25 MySQL Community Server (GPL)
    Protocol version:       10
    Connection:             127.0.0.1 via TCP/IP
    Server characterset:    latin1
    Db     characterset:    latin1
    Client characterset:    latin1
    Conn.  characterset:    latin1
    TCP port:               3306
    Uptime:                 1 min 28 sec

    Threads: 1  Questions: 15  Slow queries: 0  Opens: 105  Flush tables: 1  Open tables: 98  Queries per second avg: 0.170
    --------------

强制不使用SSL进行连接

    mysql -h127.0.0.1 -uroot --ssl-mode=DISABLED

需不需要使用SSL

    开启SSL会对性能产生影响，如果是内网使用的多则不需要开启，如果数据很机密推荐打开

MySQL5.7默认使用SSL进行连接，可以强制用户使用SSL进行连接

    (root@localhost) [(none)]> alter user 'neo'@'%' require ssl;
    Query OK, 0 rows affected (0.01 sec)

    shell> mysql -h127.0.0.1 -uneo -p123 --ssl-mode=DISABLED

    mysql: [Warning] Using a password on the command line interface can be insecure.
    ERROR 1045 (28000): Access denied for user 'neo'@'localhost' (using password: YES)

X509 开启SSL的同时还需要提供公密钥文件，这里的公密钥文件存在于/mdata/mysql_test_data/ca.pem 下面

    (root@localhost) [(none)]> alter user 'neo'@'%' require x509;
    Query OK, 0 rows affected (0.01 sec)

## 密码插件 validate_password

这个插件强制要求使用的密码符合一些复杂密码的规范，其实就是限制你使用弱口令

### 安装方式

配置文件安装

    在my.cnf中加入
    [mysqld]
    plugin-load=validate_password.so

在线安装

    (root@localhost) [(none)]> INSTALL PLUGIN validate_password SONAME 'validate_password.so';
    Query OK, 0 rows affected (0.02 sec)

查看是否安装了该插件

    (root@localhost) [(none)]> show plugins;
    +----------------------------+----------+--------------------+----------------------+---------+
    | Name                       | Status   | Type               | Library              | License |
    +----------------------------+----------+--------------------+----------------------+---------+
    | binlog                     | ACTIVE   | STORAGE ENGINE     | NULL                 | GPL     |
    | mysql_native_password      | ACTIVE   | AUTHENTICATION     | NULL                 | GPL     |
    | sha256_password            | ACTIVE   | AUTHENTICATION     | NULL                 | GPL     |
    | ...............            | ACTIVE   | AUTHENTICATION     | NULL                 | GPL     |
    | ...............            | ACTIVE   | AUTHENTICATION     | NULL                 | GPL     |
    | ...............            | ACTIVE   | AUTHENTICATION     | NULL                 | GPL     |
    | validate_password          | ACTIVE   | VALIDATE PASSWORD  | validate_password.so | GPL     |
    +----------------------------+----------+--------------------+----------------------+---------+
    45 rows in set (0.00 sec)

此时尝试修改用户的密码，会提示你密码不符合密码插件的策略要求，因为太简单了

    (root@localhost) [(none)]> alter user 'neo'@'%' identified by '123';
    ERROR 1819 (HY000): Your password does not satisfy the current policy requirements
    (root@localhost) [(none)]>

查看密码策略强度，此处为MEDIUM(长度至少8位，大小写混合，数字和特殊字符至少出现一次)

    (root@localhost) [(none)]> show variables like 'validate%';
    +--------------------------------------+--------+
    | Variable_name                        | Value  |
    +--------------------------------------+--------+
    | validate_password_check_user_name    | OFF    |
    | validate_password_dictionary_file    |        |
    | validate_password_length             | 8      | <----- 长度至少8位
    | validate_password_mixed_case_count   | 1      | <----- 大小写混合
    | validate_password_number_count       | 1      | <----- 数字至少出现一次
    | validate_password_policy             | MEDIUM |
    | validate_password_special_char_count | 1      | <----- 特殊字符至少出现一次
    +--------------------------------------+--------+
    7 rows in set (0.00 sec)

在配置文件中加入选项让mysql可以自动启动密码插件

    shell> vim /etc/my.cnf
    加入
    [mysqld]
    ......
    ......
    plugin-load=validate_password.so
    
    重启MySQL
    shell> /etc/init.d/mysql.server restart
    Shutting down MySQL..                                      [  OK  ]
    Starting MySQL.                                            [  OK  ]

在密码插件的STRONG强度中可以使用字典文件，在这个字典中出现的字符都不能作为密码的一部分

设置字典文件，字典文件中出现的字符不能作为密码的一部分

    shell> cd /mdata/mysql_test_data
    shell> vim dic.file # 写入admin
    shell> chown mysql:mysql dic.file
    shell> mysql

    (root@localhost) [(none)]> set global validate_password_dictionary_file = '/mdata/mysql_test_data/dic.file';
    Query OK, 0 rows affected (0.01 sec)

    # 设置策略为STRONG
    (root@localhost) [(none)]> set global validate_password_policy = STRONG;
    Query OK, 0 rows affected (0.00 sec)

    # 尝试修改密码，此时密码中包含了字典中的admin字符，所以无法使用这样的密码
    (root@localhost) [(none)]> alter user 'neo'@'%' identified by '1111adminDDD__';
    ERROR 1819 (HY000): Your password does not satisfy the current policy requirements

## 当前配置文件

    [client]
    user = root
    password = 123
    socket = /tmp/mysql.sock

    [mysql]
    prompt = (\\u@\\h) [\\d]>\\_

    [mysqld]
    port = 3306
    user = mysql
    datadir = /mdata/mysql_test_data
    log_error = error.log
    plugin-load=validate_password.so