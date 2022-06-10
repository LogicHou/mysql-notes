[淘宝数据库内核月报](http://mysql.taobao.org/monthly/)

[mysql官方开发者博客](https://dev.mysql.com/)

## MySQL二进制包下载链接

version 8.0 

    wget https://dev.mysql.com/get/Downloads/MySQL-8.0/mysql-8.0.29-linux-glibc2.12-x86_64.tar.xz

version 5.7

    wget https://dev.mysql.com/get/Downloads/MySQL-5.7/mysql-5.7.37-linux-glibc2.12-x86_64.tar.gz

version 5.6

    wget https://dev.mysql.com/get/Downloads/MySQL-5.6/mysql-5.6.51-linux-glibc2.12-x86_64.tar.gz

## MySQL数据库版本比较

|                | MySQL5.5             | MySQL5.6             | MySQL5.7               |
| -------------- | -------------------- | -------------------- | ---------------------- |
| 索引           | 全局读写锁，并发一般 | 全局读写锁，并发一般 | 锁优化，并发性能提升   |
| 多线程复制     | 不支持               | 基于库的多线程复制   | 基于组提交的多线程复制 |
| 数据一致性     | 半同步，数据有丢失   | 半同步，数据有丢失   | 无损复制，数据无丢失   |
| 半结构化数据   | BLOB                 | BLOB                 | 原生JSON格式           |
| InnoDB全文索引 | 不支持               | 支持，但不支持中文   | 支持                   |
| 地理空间索引   | MyISAM支持           | MyISAM支持           | InnoDB也支持           |
| 事务组提交     | 有bug，不支持        | 支持                 | 支持                   |

## MySQL相关从业人员

| 运维DBA                  | 开发DBA            | 内核DBA                |
| ------------------------ | ------------------ | ---------------------- |
| 负责数据库的相关运维工作 | 负责应用程序的建表 | 负责数据库内核级的开发 |
| 监控数据库状态           | 负责表的索引创建   | 处理内核级的相关bug    |
| 搭建高可用集群           | 审核开发SQL语句    | 跟踪最新数据库内核     |

建议从事开发DBA，更靠近业务，如果不跟着业务走会不知道现在流行的东西是什么。比如微信红包功能，那么就可以去了解数据库这一层应该怎么设计

## 关于安装方式

推荐使用二进制包方式安装，编译安装并不会提升性能

## 服务器准备

开始1核1G内存的虚拟机或者云主机就可以了，配置上不封顶

## 官方文档地址

https://dev.mysql.com/doc/

https://dev.mysql.com/doc/refman/8.0/en/

https://dev.mysql.com/doc/refman/5.7/en/

https://dev.mysql.com/doc/refman/5.6/en/

## 实例数据库

https://dev.mysql.com/doc/index-other.html

Example Databases