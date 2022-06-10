# 数据库基准测试与 mysqladmin 监控

## sysstat 工具安装

centos下安装：

    $ yum -y install sysstat

### iostat 命令

    $ iostat -xm 1 # 1表示每一秒输出一下结果
    Linux 4.18.0-348.7.1.el8_5.x86_64 (study.centos.neo)    05/29/2022      _x86_64_        (8 CPU)

    # 显示cpu的平均开销
    avg-cpu:  %user   %nice %system %iowait  %steal   %idle
              0.41    0.04    0.20    0.50    0.00   98.84

    Device            r/s     w/s     rMB/s     wMB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util
    sda             29.85    2.98      0.46      0.12     0.22     2.01   0.74  40.31    4.09    6.55   0.14    15.73    41.49   1.33   4.37

Device 中的 sda 表示一块磁盘，其中比较需要关注的参数如下：

| 参数名  | 说明                                                                                                                              |
| ------- | --------------------------------------------------------------------------------------------------------------------------------- |
| rrqm/s  | 合并读                                                                                                                            |
| wrqm/s  | 合并写                                                                                                                            |
| r/s     | 每秒读                                                                                                                            |
| w/s     | 每秒写                                                                                                                            |
| iops    | r/s和w/s加起来就是iops                                                                                                            |
| rMB/s   | 读带宽                                                                                                                            |
| wMB/s   | 写带宽                                                                                                                            |
| areq-sz | 也叫avgrq-sz，每秒读取的扇区数量，每个扇区的大小在磁盘上是固定的512字节，这个值乘以512字节就是整个带宽，用rMB+wMB计算得到是一样的 |
| aqu-sz  | 也叫avgqu-sz，平均队列深度，这个值比较重要，对ssd来说这个值最高时可以达到30                                                       |
| r_await | 读等待时间                                                                                                                        |
| w_await | 写等待时间                                                                                                                        |
| svctm   | 服务时间，不用太关注                                                                                                              |
| %util   | 磁盘使用率                                                                                                                        |

#### 查看某块盘

    $ iostat -xm 4 sda # sda是盘号
    Linux 4.18.0-348.7.1.el8_5.x86_64 (study.centos.neo)    05/29/2022      _x86_64_        (8 CPU)

    avg-cpu:  %user   %nice %system %iowait  %steal   %idle
              0.26    0.03    0.13    0.32    0.00   99.25

    Device            r/s     w/s     rMB/s     wMB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util
    sda             18.94    1.97      0.29      0.08     0.14     1.28   0.74  39.45    4.14    6.32   0.09    15.83    40.13   1.33   2.79

如何判断磁盘的性能是否达到了瓶颈，通常来说只看刺激的使用率达到90%以上的话就是比较繁忙的状态了，但是SSD有一个特点就是就算是比较繁忙的状态依然可以工作的不错

### sysbench

安装：

    $ git clone https://github.com/akopytov/sysbench.git
    $ cd sysbench
    $ ./autogen.sh
    $ # Add --with-pgsql to build with PostgreSQL support
    $ ./configure
    $ make -j
    $ make install

    # 如果碰到错误：sysbench: error while loading shared libraries: libmysqlclient.so.20: cannot open shared object file: No such file or directory 
    $ ldconfig /usr/local/mysql/lib/

使用：

sysbench 对存储进行测试是这样的一个模式，首先有一个叫 prepare 的模式用来产生测试文件，另外一个叫 run 就是用来进行测试，最后一个叫 clean 将测试产生的文件删除

prepare 生成测试文件：

    $ sysbench fileio --file-num=4 --file-block-size=16384 --file-total-size=20G prepare
    # fileio 测试模式
    # --file-num=4 表示现在测试文件由4个文件组成
    # --file-block-size=16384 测试过程中每个IO的大小是多少，这里是16K一个IO
    # --file-total-size=20G   4个文件总大小是多少，每个文件大小就是20G/4就是5G每个文件
    # prepare 表示帮你生成这4个测试文件
    sysbench 1.1.0-df89d34 (using bundled LuaJIT 2.1.0-beta3)

    4 files, 5242880Kb each, 20480Mb total
    Creating files for the test...
    Extra file open flags: (none)
    Creating file test_file.0
    Creating file test_file.1
    Creating file test_file.2
    Creating file test_file.3
    21474836480 bytes written in 160.41 seconds (127.67 MiB/sec).

    $ ls -lh test_file.*
    -rw-------. 1 root root 5.0G May 29 22:44 test_file.0
    -rw-------. 1 root root 5.0G May 29 22:44 test_file.1
    -rw-------. 1 root root 5.0G May 29 22:45 test_file.2
    -rw-------. 1 root root 5.0G May 29 22:46 test_file.3

跑随机读测试(--file-test-mode=rndrd)：

    $ sysbench fileio --file-num=4 --file-block-size=4096 --file-total-size=20G --file-test-mode=rndrd --file-extra-flags=direct --time=300 --events=0 --threads=16 --report-interval=1 run
    # fileio 测试模式
    # --file-block-size=4096 初始化的时候是16384，这里用4096是没关系的，16k产生的文件用4k测也是可以的
    # --file-test-mode=rndrd 文件测试模式，rnd表示随机，rd表示read，rndrd就是随机读
    # --time 最大测试时间单位秒
    # --events=0 最大请求数，0表示不限制请求数，但是前面有--time则以300秒作为结束条件
    # --threads=16 表示测试的时候有多少个线程去进行文件的测试
    # --report-interval=1 表示每秒钟输出结果
    sysbench 1.1.0-df89d34 (using bundled LuaJIT 2.1.0-beta3)

    Running the test with following options:
    Number of threads: 16
    Report intermediate results every 1 second(s)
    Initializing random number generator from current time


    Extra file open flags: directio
    4 files, 5GiB each
    20GiB total file size
    Block size 4KiB
    Number of IO requests: 0
    Read/Write ratio for combined random IO test: 1.50
    Periodic FSYNC enabled, calling fsync() each 100 requests.
    Calling fsync() at the end of test, Enabled.
    Using synchronous I/O mode
    Doing random read test
    Initializing worker threads...

    Threads started!

    [ 1s ] reads: 99.13 MiB/s writes: 0.00 MiB/s fsyncs: 0.00/s latency (ms,95%): 0.888
    [ 2s ] reads: 99.17 MiB/s writes: 0.00 MiB/s fsyncs: 0.00/s latency (ms,95%): 0.904
    [ 3s ] reads: 99.39 MiB/s writes: 0.00 MiB/s fsyncs: 0.00/s latency (ms,95%): 0.888
    [ 4s ] reads: 67.07 MiB/s writes: 0.00 MiB/s fsyncs: 0.00/s latency (ms,95%): 0.888
    ...
    [ 298s ] reads: 99.33 MiB/s writes: 0.00 MiB/s fsyncs: 0.00/s latency (ms,95%): 0.888
    [ 299s ] reads: 32.20 MiB/s writes: 0.00 MiB/s fsyncs: 0.00/s latency (ms,95%): 0.904

    # 整个测试完后会输出一个测试报告
    Throughput:
         read:  IOPS=20219.57 78.98 MiB/s (82.82 MB/s)    <--这个就是经过测试后得到的IOPS
         write: IOPS=0.00 0.00 MiB/s (0.00 MB/s)
         fsync: IOPS=0.00

    Latency (ms):                                         <--响应速度
            min:                                  0.14    <--最小的一次响应时间
            avg:                                  0.79    <--平均响应时间
            max:                                334.42    <--最大的一次响应时间
            95th percentile:                      0.89    <--95%响应速度
            sum:                            4792910.96
    
sysbench 提供的结果是用带宽来表示的，其中的 reads xx.xx MiB/s 表示带宽，乘以256(4k)，刚好可以和iostat中的r/s对应

跑起来后用 iostat 查看IO状态：

    $ iostat -xm 4 sda # sda是盘号
    avg-cpu:  %user   %nice %system %iowait  %steal   %idle
              0.41    0.00    3.67   43.48    0.00   52.45

    Device            r/s     w/s     rMB/s     wMB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util
    sda           25410.00    0.00     99.26      0.00     0.00     0.00   0.00   0.00    0.62    0.00  15.76     4.00     0.00   0.04 100.00

    avg-cpu:  %user   %nice %system %iowait  %steal   %idle
              0.82    0.00    3.89   38.54    0.00   56.76

    Device            r/s     w/s     rMB/s     wMB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util
    sda           23362.25    0.00     91.26      0.00     0.00     0.00   0.00   0.00    0.68    0.00  15.78     4.00     0.00   0.04 100.00

跑顺序读测试(--file-test-mode=seqrd)：

    $ sysbench fileio --file-num=4 --file-block-size=4096 --file-total-size=20G --file-test-mode=seqrd --file-extra-flags=direct --time=10 --events=0 --threads=16 --report-interval=1 run
    sysbench 1.1.0-df89d34 (using bundled LuaJIT 2.1.0-beta3)

    Running the test with following options:
    Number of threads: 16
    Report intermediate results every 1 second(s)
    Initializing random number generator from current time


    Extra file open flags: directio
    4 files, 5GiB each
    20GiB total file size
    Block size 4KiB
    Periodic FSYNC enabled, calling fsync() each 100 requests.
    Calling fsync() at the end of test, Enabled.
    Using synchronous I/O mode
    Doing sequential read test
    Initializing worker threads...

    Threads started!

    [ 1s ] reads: 104.94 MiB/s writes: 0.00 MiB/s fsyncs: 0.00/s latency (ms,95%): 0.797
    [ 2s ] reads: 106.25 MiB/s writes: 0.00 MiB/s fsyncs: 0.00/s latency (ms,95%): 0.597
    [ 3s ] reads: 69.02 MiB/s writes: 0.00 MiB/s fsyncs: 0.00/s latency (ms,95%): 1.082
    [ 4s ] reads: 101.34 MiB/s writes: 0.00 MiB/s fsyncs: 0.00/s latency (ms,95%): 1.184
    [ 5s ] reads: 105.43 MiB/s writes: 0.00 MiB/s fsyncs: 0.00/s latency (ms,95%): 0.619
    [ 6s ] reads: 102.96 MiB/s writes: 0.00 MiB/s fsyncs: 0.00/s latency (ms,95%): 0.954
    [ 7s ] reads: 106.33 MiB/s writes: 0.00 MiB/s fsyncs: 0.00/s latency (ms,95%): 0.587
    [ 8s ] reads: 106.33 MiB/s writes: 0.00 MiB/s fsyncs: 0.00/s latency (ms,95%): 0.587
    [ 9s ] reads: 73.89 MiB/s writes: 0.00 MiB/s fsyncs: 0.00/s latency (ms,95%): 0.587

    Throughput:
            read:  IOPS=25100.22 98.05 MiB/s (102.81 MB/s)
            write: IOPS=0.00 0.00 MiB/s (0.00 MB/s)
            fsync: IOPS=0.00

    Latency (ms):
            min:                                  0.09
            avg:                                  0.64
            max:                                329.25
            95th percentile:                      0.72
            sum:                             159870.55

顺序读的时候不能光看IOPS，顺序读的时候IOPS相当的高

还有三种度的测试分别是随机读(--file-test-mode=rndwr)、顺序读(--file-test-mode=seqwr)和随机读写(--file-test-mode=rndrw)一般只需要关心随机读就可以了，测试随机读的时候所使用的参数会对性能测试产生较大的影响

测顺序读的时候测单线程就可以了，如果测多线程是没有意义的，因为多线程顺序读差不读也是一种比较随机的读了

### 小结

所以这个测试的流程就是先生成测试文件，然后开启 sysbench run，同时配合 iostat 命令辅助查看

MySQL 一个连接通常来说都是单线程去执行的，MySQL 用 SSD 充分发挥它的性能的话比较适合的场景是大并发的场景，有多个线程在执行的场景

### 添加一个优化参数 innodb_flush_method

    $ vim /etc/my.cnf

    [mysqld]
    ...
    innodb_log_file_size=1G
    innodb_flush_method=O_DIRECT

    $ /etc/init.d/mysql.server restart

这样每次 MySQL 的写入就直接写盘了，不会写操作系统缓存，测试的时候也一定要为 sysbench 命令加上 --file-extra-flags=direct 选项

加入这个参数是为了防止 MySQL的数据写入策略，一般情况下数据会在 系统的 page cache 中积累到一定数量后，再一次性写入磁盘，这种方式一方面会占用额外的内存，另一方面可能会导致数据损坏（比如数据还在page cache中的时候服务器宕机、断电等）

MySQL 中的数据文件是用 O_DIRECT 直接写到磁盘，对日志文件的话都是写入到 page cache 然后再调用 fsync

### 还有一个非常重要的参数 innodb_io_capacity

这个参数表示数据库每次能用到的写入的 IPOS 有多少，这个参数默认只有 200，对于 SSD 来说太小了，一般随机写带宽有 70MiB 左右的，可以调到 4000左右

    $ vim /etc/my.cnf

    [mysqld]
    ...
    innodb_log_file_size=1G
    innodb_flush_method=O_DIRECT
    innodb_io_capacity=4000

    $ /etc/init.d/mysql.server restart