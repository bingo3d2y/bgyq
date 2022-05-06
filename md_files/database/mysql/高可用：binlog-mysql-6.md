## 高可用：binlog-mysql-6

核心，不还是操作写入文件，然后同步文件，再在其他机器上还原操作。

但是，如何高效的记录、传输和还原等都需要技术的实现。

核心还是：半同步复制。

## 安全配置

`sql_safe_updates=ON`

系统变量sql_safe_updates分会话级别和全局级别。可以动态修改。默认是关闭（值为0）。

```bash
SET MySQL_variable [ = { 0 | 1 }];
SET SESSION MySQL_variable [ = { 0 | 1 }];

#设置会话级别的系统变量

mysql> set sql_safe_updates=1;

Query OK, 0 rows affected (0.00 sec)
 

#设置全局级别的系统变量

mysql> set global sql_safe_updates=1;

Query OK, 0 rows affected (0.00 sec)
 

如果要设置全局级别的系统变量sql_safe_updates，最好在配置文件my.cnf中设置，这样就能避免MySQL重启后，SQL命令设置的sql_safe_updates系统全局变量失效。
```

update 时，在没有 where 条件或者where 后不是索引字段时，必须使用 limit ,否则无法执行；在有 where 条件时，为索引字段

误删更重要的是事前预防:

1. 把 sql_safe_updates 参数设置为 on。这样一来，如果我们忘记在 delete 或者 update 语句中写 where 条件，或者 where 条件里面没有包含索引字段的话，这条语句的执行就会报错。
2. 码上线前，必须经过 SQL 审计。
3. 账号分离，避免写错命令。
4. 第二条建议是，制定操作规范。这样做的目的，是避免写错要删除的表名
   - 在删除数据表之前，必须先对表做改名操作。然后，观察一段时间，确保对业务无影响以后再删除这张表。
   - 改表名的时候，要求给表名加固定的后缀（比如加 _to_be_deleted)，然后删除表的动作必须通过管理系统执行。并且，管理系删除表的时候，只能删除固定后缀的表。
5. 脚本分别是：备份脚本、执行脚本、验证脚本和回滚脚本。如果能够坚持做到，即使出现问题，也是可以很快恢复的，一定能降低出现故障的概率。

恢复之前为了避免产生没有用的二进制日志，可以关闭二进制日志的记录
SET SESSION sql_log_bin=0;

恢复完成，启用记录二进制日志
SET SESSION sql_log_bin=1;

## binlog

https://www.cnblogs.com/youzhibing/p/13131485.html

https://www.cnblogs.com/michael9/p/11923483.html

https://www.cnblogs.com/xinysu/p/6607658.html#autoid-2-2-0

日常开发，运维中，经常会出现误删数据的情况。误删数据的类型大致可分为以下几类：

- 使用 delete 误删行
- 使用 drop table 或 truncate table 误删表
- 使用 drop database 语句误删数据库
- 使用 rm 命令误删整个 MySQL 实例。

不同的情况，都会有其优先的解决方案：

- 针对误删行，可以通过 Flashback 工具将数据恢复
- 针对误删表或库，一般采用通过 BINLOG 将数据恢复。
- 而对于误删 MySQL 实例，则需要我们搭建 HA 的 MySQL 集群，并保证我们的数据跨机房，跨城市保存。

### binlog配置

#### 基础配置

- 文件大小
  - max_binlog_size
    - 范围4k-1G，默认为1G；这里注意下，并非设置了 max_binlog_size=1G，binlog文件最大就为1G，当事务短且小的情况下，binlog接近1G的时候，就会flush log，生成新的binlog文件，但是，但是，但是，但是同个事务是不能够跨多个binlog文件存储，一个事务只能存储在一个binlog文件。如果这个时候，有个大事务，假设单个SQL UPDATE了100w行数据，SQL产生的binlog日志记录有5G，那么当前的binlog文件则会出现大于5G的情况，该事务结束后，才会切换binlog文件。
- 缓存大小
  - binlog_cache_size
    - binlog写缓冲区设置大小，由于是内存，写速度非常快，可以有效提高binlog的写效率，如果数据库中经常出现大事务，可以酌情提高该参数。
    - 那么，如果观察自家DB实例的binlog_cache_size设置是否合理呢？可以通过show status like 'Binlog_cache%';查看Binlog_cache_use and Binlog_cache_disk_use的使用情况，Binlog_cache_use表示用到binlog缓冲区的次数，Binlog_cache_disk_use ，使用临时文件来存放binlog cache的次数，如果Binlog_cache_disk_use的次数过多，可以酌情提高该参数。详见下图。
      - ![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/mysql-cache-size.png)
  - binlog_stmt_cache_size
    - 保留一个事务内，非事务语句文本的缓存大小。默认32k。
    - 与binlog_cache_size一样，也可以通过show status like 'binlog_stmt_cache%'来查看是否设置合理。查看参数为：Binlog_stmt_cache_use （使用缓存区的次数），Binlog_stmt_cache_disk_use（使用临时文件的次数）
      - ![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/binlog-stmt-cache.png)
  - max_binlog_cache_size
    - 默认为4G，如果发生大事务占用binlog cache超过设置值，则会报错 ： multi-statement transaction required more than 'max_binlog_cache_size' bytes of storage。
    - 这时候，就有个疑问了，为啥存在了 binlog_cache_size的设置，还需要 max_binlog_cache_size呢？
      - 其实是这样，当一个线程连接进来并开始执行事务的时候，数据库会按照binlog_cache_size的大小分配给它一个缓冲区域，如果使用到的空间要大于binlog_cache_size，则会使用临时文件来存储，线程结束后再删除临时文件。
      - 而max_binlog_cache_size则是严格限制了一个多SQL事务总的使用binlog cache的大小，保留分配缓冲区域跟临时文件，总大小不能超过max_binlog_cache_size的限制值，一旦超过，则会报错multi-statement transaction required more than 'max_binlog_cache_size' bytes of storage。
  - max_binlog_stmt_cache_size
    - 默认4G。超过则报错。注意事项跟 max_binlog_cache_size 类似。
- binlog文件相关
  - log_bin_basename
    - binlog文件的命名方式
  - log_bin_index
    - binlog索引文件的绝对路径
  - expire_logs_days
    - binlog保留的有效天数，过期的会自动删除
    - 这里有个小tips，假设当前binlog文件过多且大占用磁盘空间，可以修改小改参数，改参数只有在切换新的binlog文件时，才会删除过期文件，也就是可以等数据库把当前binlog写满后切换到新文件的时候删除，也可以手动执行flush logs，手动切换binlog，同时会触发对过期binlog文件的删除。

#### 重要配置

- binlog开关

  - **log_bin**

    - 需要在数据库配置文件中添加或者指定--log-bin=[base-name]启动DB服务，重启后修改才生效

  - server-id

    ```bash
    [mysqld]
    log-bin         = /var/lib/mysql/mysql-bin
    server-id       = 3
    ```

    上面是开启binlog的最小配置。

- 日志记录内容相关

  - **binlog_format**

    - 超重要-
    - 设置binlog的记录格式
    - 5.7.6前默认statement，5.7.7后默认row，可选row，mixed，statement

  - **binlog_row_image**

    - 主要针对当binlog_format=row格式 下的设置，
    - 默认full，可选full，minimal，noblob

  - **binlog_rows_query_log_events**

    - 主要针对当binlog_format=row格式 下的设置，如果基于row记录binlog日志，默认是只记录变化的行数据，不记录涉及执行的SQL语句，如果开启此参数，则会一同记录执行的SQL语句
    - 默认false

  - binlog_gtid_simple_recovery

    - GTID复制会使用到，该参数控制 配置了的GTID复制到实例，在重启时或者清理binlog文件时，数据库只需要打开最老跟最新两个binlog文件取出gtid_purged and gtid_executed，不需要打开所有文件
    - 默认为false，这个参数是社区反馈给官方添加，调整这个选项设置为True，对性能会有所提高，但是在某些环境下，由于只打开两个文件来计算，所以计算gtids值可能会出错。而保持这个选项值为false，能确保计算总是正确。
    - 组提交（提高binary log并发提交的数据量）

  - binlog_group_commit_sync_delay

    - 默认为0
    - 结合binlog_group_commit_sync_no_delay_count来理解，见下文

  - binlog_group_commit_sync_no_delay_count

    - 默认为0
    - MySQL等待binlog_group_commit_sync_delay毫秒的时间直到binlog_group_commit_sync_no_delay_count个数时进行一次组提交，如果binlog_group_commit_sync_delay毫秒内也还没有到达指定的个数，也会提交。
    - flush disk相关

  - sync_binlog

    - 5.7.7前默认为0，之后默认为1，范围0-4294967295

    - sync_binlog =0，则是依赖操作系统刷新文件的机制，MySQL不会主动同步binlog内容到磁盘文件中去，而是依赖操作系统来刷新binary log。

    - sync_binlog =N (N>0) ，则是MySQL 在每写 N次 二进制日志binary log时，会使用fdatasync()函数将它的写二进制日志binary log同步到磁盘中去。

    - 注: 如果启用了autocommit，那么每一个语句statement就会有一次写操作；否则每个事务对应一个写操作。

    - 如果设置sync_binlog =0 ，发生crash事件（服务器），数据库最高丢失binlog内容为1s内写在file system buffer的内容；

    - 如果设置sync_binlog =N ，发生crash事件（服务器），数据库最高丢失binlog内容为写在file system buffer内 N个binlog events；

    - 这个参数经常跟innodb_flush_log_at_trx_commit结合调整，提高性能或者提高安全性（详细可查看上周博文：http://www.cnblogs.com/xinysu/p/6555082.html 中 “redo参数” 一节），**这里提2个推荐的配置：**

      innodb_flush_log_at_trx_commit和sync_binlog 都为 1（俗称双一模式），在mysqld 服务崩溃或者服务器主机crash的情况下，binary log 只有可能丢失最多一个语句或者一个事务。但是有得必有舍，这个设置是最安全但也是最慢的。适合数据一致性要求较高，核心业务使用。

      innodb_flush_log_at_trx_commit=2 ，sync_binlog=N (N为500 或1000) **，但是但是但是，服务器一定要待用蓄电池后备电源来缓存cache**，在服务器crash后，还能支持把file system buffer中的内容写入到binlog file中，防止系统断电异常。这种适合当磁盘IO无法满足业务需求时，比如节假日或者周年活动产生的数据库IO压力，则推荐这么设置。、

#### server-id：挺有意思

4字节，产生这个事件的mysqld服务器id。这个是在服务器中配置文件配置的，用于主从复制。server id会在循环复制时避免死循环（开启了--log-slave-updates配置）。假设M1、M2和M3的server id分别是1、2、3，而且他们循环复制：M1是M2的主，M2是M3的主，M3是M1的主。他们的主从关系如下：

```php
M1---->M2
 ^      |
 |      |
 +--M3<-+
```

客户端发起了一个插入语句给M1，M1执行了之后，写入了M1的binlog文件中，包含了server id为1。这个事件被发给了M2，M2也执行了这个语句，然后写binlog的时候，server id还是1，因为这个事件最初是由M1发起的。然后M3也收到了这个事件，执行完成后写入到M3的binlog文件时，server id还是1。然后M1收到了这个事件之后，在执行插入语句之前，可以分析出server id=1，也就是这个语句最初是本机发起的，这个语句会被忽略执行。



作者：端木轩
链接：https://www.jianshu.com/p/6523693d6fd7

### binlog常用命令

#### mysql client

```bash
# 查询 BINLOG 格式
show VARIABLES like 'binlog_format';

# 查询 BINLOG 位置
show VARIABLES like 'datadir';

# 查询当前数据库中 BINLOG 名称及大小
show binary logs;

# 查看 master 正在写入的 BINLOG 信息
show master status\G;

# 通过 offset 查看 BINLOG 信息
show BINLOG events in 'mysql-bin.000034' limit 9000,  10;

# 通过 position 查看 binlog 信息
show BINLOG events in 'mysql-bin.000034' from 1742635 limit 10;
```

使用 `show BINLOG events` 的问题：

- 使用该命令时，如果当前 binlog 文件很大，而且没有指定 `limit`，会引发对资源的过度消耗。因为 MySQL 客户端需要将 binlog 的全部内容处理，返回并显示出来。为了防止这种情况，mysqlbinlog 工具是一个很好的选择。

#### mysqlbinlog

对于一些大型的 BINLOG 文件，使用 mysqlbinlog 会更加的方便和效率。

在介绍 mysqlbinlog 工具使用前，先来看下 BINLOG 文件的内容：

```
# 查询 BINLOG 的信息
mysqlbinlog  --no-defaults mysql-bin.000034 | less
# at 141
#100309  9:28:36 server id 123  end_log_pos 245
  Query thread_id=3350  exec_time=11  error_code=0
```

- `at` 表示 offset 或者说事件开始的起始位置
- `100309 9:28:36 server id 123` 表示 server 123 开始执行事件的日期
- `end_log_pos 245` 表示事件的结束位置 + 1，或者说是下一个事件的起始位置。
- `exec_time` 表示在 master 上花费的时间，在 salve 上，记录的时间是从 Master 记录开始，一直到 Slave 结束完成所花费的时间。
- `rror_code=0` 表示没有错误发生。

在大致了解 binlog 的内容后，mysqlbinlog 的用途有哪些？：

- mysqlbinlog 可以作为代替 cli 读取 binlog 的工具。
- mysqlbinlog 可以将执行过的 SQL 语句输出，用于数据的恢复或备份。

查询 BINLOG 日志：

```
# 查询规定时候后发生的 BINLOG 日志
mysqlbinlog --no-defaults --base64-output=decode-rows -v \
 --start-datetime  "2019-11-22 14:00:00" \
 --database sync_test  mysql-bin.000034 | less
```

导出 BINLOG 日志，用于分析和排查 sql 语句：

```
mysqlbinlog --no-defaults --base64-output=decode-rows -v \
 --start-datetime  "2019-11-22 14:00:00" \
 --database sync_test \
 mysql-bin.000034 > /home/mysql_backup/binlog_raw.sql
```

导入 BINLOG 日志

```
# 通过 BINLOG 进行恢复。
mysqlbinlog --start-position=1038 --stop-position=1164 \
 --database=db_name  mysql-bin.000034 | \
 mysql  -u cisco -p db_name

# 通过 BINLOG 导出的 sql 进行恢复。
mysql -u cisco -p db_name < binlog_raw.sql
```

mysqlbinlog 的常用参数：

- `--database` 仅仅列出配置的数据库信息

- `--no-defaults` 读取没有选项的文件, 指定的原因是由于 mysqlbinlog 无法识别 BINLOG 中的 `default-character-set=utf8` 指令

  ```bash
  
  ```

  end

- `--offset` 跳过 log 中 N 个条目

- `--verbose`将日志信息重建为原始的 SQL 陈述。

  - `-v` 仅仅解释行信息
  - `-vv` 不但解释行信息，还将 SQL 列类型的注释信息也解析出来

- `--start-datetime`显示从指定的时间或之后的时间的事件。

  - 接收 `DATETIME` 或者 `TIMESTRAMP` 格式。

- `--base64-output=decode-rows`将 BINLOG 语句中事件以 base-64 的编码显示，对一些二进制的内容进行屏蔽。

  - `AUTO` 默认参数，自动显示 BINLOG 中的必要的语句
  - `NEVER` 不会显示任何的 BINLOG 语句，如果遇到必须显示的 BINLOG 语言，则会报错退出。
  - `DECODE-ROWS` 显示通过 `-v` 显示出来的 SQL 信息，过滤到一些 BINLOG 二进制数据。

如果想知道当前 MySQL 中正在写入的 BINLOG 的名称，大小等基本信息时，可以通过 Cli 相关的命令来查询。

但想查询，定位，恢复 BINLOG 中具体的数据时，要通过 mysqlbinlog 工具，因为相较于 Cli 来说，mysqlbinlog 提供了 `--start-datetime`，`--stop-position` 等这样更为丰富的参数供我们选择。这时 Cli 中 `SHOW BINLOG EVENTS` 的简要语法就变得相形见绌了。

### binlog日志格式

MySQL 5.1.5 之前 binlog 的格式只有 STATEMENT，5.1.5 开始支持 ROW 格式的 binlog，从 5.1.8 版本开始，MySQL 开始支持 MIXED 格式的 binlog。

MySQL 5.7.7 之前，binlog 的默认格式都是 STATEMENT，在 5.7.7 及更高版本中，binlog_format 的默认值才是 ROW。

```bash
# 显示binlog日志格式
mysql> show variables like '%binlog%';

# Mysql 5.8 之后默认 binlog_format 真的是row

[root@sunny ~]# docker run -itd -v /root/custom:/etc/mysql/conf.d -e MYSQL_ROOT_PASSWORD=root  mysql:8.0
Unable to find image 'mysql:8.0' locally
8.0: Pulling from library/mysql
Digest: sha256:52b8406e4c32b8cf0557f1b74517e14c5393aff5cf0384eff62d9e81f4985d4b
Status: Downloaded newer image for mysql:8.0
be50d88d6536368c26d57d54b39e34b283c3596561f620a17812b90c5bb1f8ae
[root@sunny ~]# docker exec -it be5 bash
root@be50d88d6536:/# mysql -u root -p
Enter password:

mysql> show variables like '%binlog_format%';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| binlog_format | ROW   |
+---------------+-------+
1 row in set (0.01 sec)


```

end

#### STATEMENT：SBR

基于SQL语句的复制(statement-based replication, SBR)

```bash
mysql> show variables like "%binlog_format";
+---------------+-----------+
| Variable_name | Value     |
+---------------+-----------+
| binlog_format | STATEMENT |
+---------------+-----------+
1 row in set (0.00 sec)

mysql> flush logs;
Query OK, 0 rows affected (0.00 sec)

## 插入数据
mysql> insert into tmp_test values(52,'eee',55);
Query OK, 1 row affected (0.00 sec)

mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000007 |      381 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)

mysql> show binlog events in  'mysql-bin.000007';
+------------------+-----+-------------+-----------+-------------+-------------------------------------------------------------+
| Log_name         | Pos | Event_type  | Server_id | End_log_pos | Info                                                        |
+------------------+-----+-------------+-----------+-------------+-------------------------------------------------------------+
| mysql-bin.000007 |   4 | Format_desc |         3 |         120 | Server ver: 5.7.5-m15-log, Binlog ver: 4                    |
| mysql-bin.000007 | 120 | Query       |         3 |         222 | BEGIN                                                       |
| mysql-bin.000007 | 222 | Query       |         3 |         350 | use `test_binlog`; insert into tmp_test values(52,'eee',55) |
| mysql-bin.000007 | 350 | Xid         |         3 |         381 | COMMIT /* xid=212 */                                        |
+------------------+-----+-------------+-----------+-------------+-------------------------------------------------------------+
4 rows in set (0.00 sec)

```



删除100万条记录 ，在STATEMENT模式下，只需1条delete * from test记录；就可以删除100万条记录

每一条会修改数据的sql都会记录在binlog中优点：只需要记录执行语句的细节和上下文环境，避免了记录每一行的变化，在一些修改记录较多的情况下相比ROW level能大大减少binlog日志量，节约IO，提高性能；还可以用于实时的还原；同时主从版本可以不一样，从服务器版本可以比主服务器版本高 。

缺点：为了保证sql语句能在slave上正确执行，必须记录上下文信息，以保证所有语句能在slave得到和在master端执行时候相同的结果；另外，主从复制时，存在部分函数（如sleep）及存储过程在slave上会出现与master结果不一致的情况，而相比Row level记录每一行的变化细节，绝不会发生这种不一致的情况。

如下所示，SBR记录的信息太少了不好恢复。

你永远无法知道一个 delete删除了多少语句...

```bash
$ mysqlbinlog /var/lib/mysql/mysql-bin.000001
...
# start position
# at 2327
#210714  3:02:55 server id 3  end_log_pos 2447 CRC32 0x9ac9b0de         Query   thread_id=4     exec_time=0     error_code=0
SET TIMESTAMP=1626231775/*!*/;
delete from sync_test where id<3
/*!*/;
# stop position
# at 2447
#210714  3:02:55 server id 3  end_log_pos 2478 CRC32 0x947c227a         Xid = 34
COMMIT/*!*/;
...
```

end

#### ROW：RBR

这个模下日志量很大--

https://segmentfault.com/a/1190000023076034

基于行的复制(row-based replication, RBR)

对于 ROW 格式的 Binlog，所有的 DML 语句都是记录在 ROWS_EVENT 中。

ROWS_EVENT分为三种：

- WRITE_ROWS_EVENT
- UPDATE_ROWS_EVENT
- DELETE_ROWS_EVENT

分别对应 insert，update 和 delete 操作。

对于 insert 操作，WRITE_ROWS_EVENT 包含了要插入的数据。

对于 update 操作，UPDATE_ROWS_EVENT 不仅包含了修改后的数据，还包含了修改前的值。

对于 delete 操作，仅仅需要指定删除的主键（在没有主键的情况下，会给定所有列）。

**对比 QUERY_EVENT 事件，是以文本形式记录 DML 操作的。而对于 ROWS_EVENT 事件，并不是文本形式，所以在通过 mysqlbinlog 查看基于 ROW 格式的 Binlog 时，需要指定 `-vv --base64-output=decode-rows`。**

```bash
mysql> show variables like "%binlog_format"                                                                                                    -> ;
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| binlog_format | ROW   |
+---------------+-------+
1 row in set (0.00 sec)

# 刷新日志
mysql> flush logs;
Query OK, 0 rows affected (0.00 sec)


mysql> use test_binlog;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
## 插入数据
mysql> insert into tmp_test values(5,'eee',55);
Query OK, 1 row affected (0.01 sec)

mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000006 |      349 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
## event_type是Write_rows
mysql> show binlog events in  'mysql-bin.000006';
+------------------+-----+-------------+-----------+-------------+------------------------------------------+
| Log_name         | Pos | Event_type  | Server_id | End_log_pos | Info                                     |
+------------------+-----+-------------+-----------+-------------+------------------------------------------+
| mysql-bin.000006 |   4 | Format_desc |         3 |         120 | Server ver: 5.7.5-m15-log, Binlog ver: 4 |
| mysql-bin.000006 | 120 | Query       |         3 |         208 | BEGIN                                    |
| mysql-bin.000006 | 208 | Table_map   |         3 |         270 | table_id: 100 (test_binlog.tmp_test)     |
| mysql-bin.000006 | 270 | Write_rows  |         3 |         318 | table_id: 100 flags: STMT_END_F          |
| mysql-bin.000006 | 318 | Xid         |         3 |         349 | COMMIT /* xid=191 */                     |
+------------------+-----+-------------+-----------+-------------+------------------------------------------+
5 rows in set (0.00 sec)



```



row模式生成的sql编码需要解码，不能用常规的办法去生成，需要加上相应的参数(--base64-output=decode-rows -v)才能显示出sql语句; 

删除100万条记录，在ROW模式下，则记录100万条删除命令。

```bash
root@b3bcbbb0f6ce:/# mysqlbinlog --no-defaults --base64-output=decode-rows -v /var/lib/mysql/mysql-bin.000004
.
/*!*/;
# at 208
#210714  6:38:54 server id 3  end_log_pos 270 CRC32 0x309cf6da  Table_map: `test_binlog`.`sync_test` mapped to number 97
# at 270
#210714  6:38:54 server id 3  end_log_pos 329 CRC32 0xd5f7d83b  Delete_rows: table id 97 flags: STMT_END_F
### DELETE FROM `test_binlog`.`sync_test`
### WHERE
###   @1=3
###   @2='c'
### DELETE FROM `test_binlog`.`sync_test`
### WHERE
###   @1=4
###   @2='d'
### DELETE FROM `test_binlog`.`sync_test`
### WHERE
###   @1=5
###   @2='e'
# at 329
#210714  6:38:54 server id 3  end_log_pos 360 CRC32 0xb95dab66  Xid = 109
COMMIT/*!*/;
DELIMITER ;
# End of log file
ROLLBACK /* added by mysqlbinlog */;
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;

```



优点

- 因为binlog的格式row，所以可以应用于任何SQL的复制，包括非确定函数，存储过程、自定义函数等。因为它同步过去的是值，举个例子，UUID，从库执行的时候不是重新执行UUID，而是把主库的这个已经生成的值直接同步到从节点上。
- 可以减少数据库锁的使用

缺点

- 要**求主从数据库的表结构必须要相同**，对于相同的列，即使顺序不同，有可能会引起主从中断
- 无法在从节点单独执行触发器，有的时候你只想在从库上执行一些操作，是不行的。 但基于SQL语句的没问题，执行那些变更的SQL就行了，但是基于行的就不行了。

仅保存记录被修改细节，不记录sql语句上下文相关信息优点：能非常清晰的记录下每行数据的修改细节，不需要记录上下文相关信息，因此不会发生某些特定情况下的procedure、function、及trigger的调用触发无法被正确复制的问题，任何情况都可以被复制，且能加快从库重放日志的效率，保证从库数据的一致性 缺点:由于所有的执行的语句在日志中都将以每行记录的修改细节来记录，因此，可能会产生大量的日志内容，干扰内容也较多；比如一条update语句，如修改多条记录，则binlog中每一条修改都会有记录，这样造成binlog日志量会很大，特别是当执行alter table之类的语句的时候，由于表结构修改，每条记录都发生改变，那么该表每一条记录都会记录到日志中，实际等于重建了表。 

tip: - row模式生成的sql编码需要解码，不能用常规的办法去生成，需要加上相应的参数(--base64-output=decode-rows -v)才能显示出sql语句; 

新版本binlog默认为ROW level，且5.6新增了一个参数：binlog_row_image；把binlog_row_image设置为minimal以后，binlog记录的就只是影响的列，大大减少了日志内容



#### MIXED：MBR

MySQL 从 V5.1.8 开始提供 Mixed 模式，V5.7.7 之前的版本默认是Statement 模式，之后默认使用Row模式， 但是在 8.0 以上版本已经默认使用 Mixed 模式了。

When running in `MIXED` logging format, the server automatically switches from statement-based to row-based logging under the following conditions:

* When a DML statement updates an [`NDBCLUSTER`](https://dev.mysql.com/doc/refman/5.7/en/mysql-cluster.html) table.
* 当 DML 语句更新一个 NDB 表时；
  当函数中包含 UUID() 时；
  2 个及以上包含 AUTO_INCREMENT 字段的表被更新时；
  执行 INSERT DELAYED 语句时；
  用 UDF 时；
  视图中必须要求运用 row 时，例如建立视图时使用了 UUID() 函数；

官网配置：https://dev.mysql.com/doc/refman/5.7/en/binary-log-mixed.html

混合模式复制(mixed-based replication, MBR)

以上两种level的混合使用经过前面的对比，可以发现ROW level和statement level各有优势，如能根据sql语句取舍可能会有更好地性能和效果；Mixed level便是以上两种leve的结合。不过，新版本的MySQL对row level模式也被做了优化，并不是所有的修改都会以row level来记录，像遇到表结构变更的时候就会以statement模式来记录，如果sql语句确实就是update或者delete等修改数据的语句，那么还是会记录所有行的变更；因此，现在一般使用row level即可。

#### 日志记录选取规则

选取规则如果是采用 INSERT，UPDATE，DELETE 直接操作表的情况，则日志格式根据 binlog_format 的设定而记录 如果是采用 GRANT，REVOKE，SET PASSWORD 等管理语句来做的话，那么无论如何都采用statement模式记录。

已测试，DELETE语句在MIXED模式下仍使用statement格式。

```bash
mysql> show variables like '%binlog_format';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| binlog_format | MIXED |
+---------------+-------+
1 row in set (0.00 sec)
mysql> delete from tmp_test where id < 60;
Query OK, 3 rows affected (0.00 sec)

mysql> show binlog events in  'mysql-bin.000007';                                                                                          +------------------+-----+-------------+-----------+-------------+-------------------------------------------------------------+
| Log_name         | Pos | Event_type  | Server_id | End_log_pos | Info                                                        |
+------------------+-----+-------------+-----------+-------------+------------------------------...                                       |
| mysql-bin.000007 | 381 | Query       |         3 |         483 | BEGIN                                                       |
| mysql-bin.000007 | 483 | Query       |         3 |         605 | use `test_binlog`; delete from tmp_test where id < 60       |
| mysql-bin.000007 | 605 | Xid         |         3 |         636 | COMMIT /* xid=230 */                                        |
+------------------+-----+-------------+-----------+-------------+-------------------------------------------------------------+
7 rows in set (0.00 sec)

```

end

##### 事务隔离对binlog日志的影响（重）

事务隔离也影响这binlog的记录格式行为。

Note
In MySQL 5.7, when READ COMMITTED isolation level is used, or the deprecated innodb_locks_unsafe_for_binlog system variable is enabled, 
there is no InnoDB gap locking except for foreign-key constraint checking and duplicate-key checking. Also, record locks for nonmatching 
rows are released after MySQL has evaluated the WHERE condition.

**If you use READ COMMITTED or enable innodb_locks_unsafe_for_binlog, you must use row-based binary logging.**



#### 修改binlog format

#### 全局修改

在源库设置了global级别的binlog_format=ROW之后，还需要中断之前所有的业务连接，因为设置之前的连接使用的还是非ROW格式的binlog写入。

可以通过如下两种方式确保修改后的源库binlog_format格式立即生效。

**方法一：**

1. 选择一个非业务的时间段，中断当前数据库上的所有业务连接。

   1. 通过如下命令查询当前数据库上的所有业务连接(所有的binlog Dump连接及当前连接除外)。

      ```
      show processlist;
      kill ID_NUMBER
      ```

   2. 中断上面查出的所有业务连接。

2. 为了避免源库binlog_format格式因为数据库重启失效，请在源库的启动配置文件(my.ini或my.cnf等)中添加或修改配置参数binlog_format并保存。

   ```
   binlog_format=ROW
   ```

**方法二：**

1. 为了避免源库binlog_format格式因为数据库重启失效，请在源库的启动配置文件(my.ini或my.cnf等)中添加或修改配置参数binlog_format并保存。

   ```
   binlog_format=ROW
   ```

2. 确保上述配置参数binlog_format添加或修改成功后，选择一个非业务时间段，重启源数据库即可

如下，修改并使新的binlog_format生效。

```bash
mysql> show variables like '%binlog_format';
+---------------+-----------+
| Variable_name | Value     |
+---------------+-----------+
| binlog_format | STATEMENT |
+---------------+-----------+
1 row in set (0.00 sec)

mysql> SET GLOBAL binlog_format=ROW;
Query OK, 0 rows affected (0.00 sec)

mysql> show variables like '%binlog_format';
+---------------+-----------+
| Variable_name | Value     |
+---------------+-----------+
| binlog_format | STATEMENT |
+---------------+-----------+
1 row in set (0.00 sec)

mysql> select @@global.binlog_format;
+------------------------+
| @@global.binlog_format |
+------------------------+
| ROW                    |
+------------------------+
1 row in set (0.00 sec)

mysql> flush logs;
Query OK, 0 rows affected (0.01 sec)

# 没有生效
mysql> show variables like '%binlog_format';
+---------------+-----------+
| Variable_name | Value     |
+---------------+-----------+
| binlog_format | STATEMENT |
+---------------+-----------+
1 row in set (0.00 sec)


```

end

##### 临时会话修改

```bash
mysql>SET binlog_format = 'STATEMENT';
mysql>show variables like '%binlog_format%';
```

一般情况下，我们不随意修改数据库级别的binlog格式，因为有可能会对程序不兼容。但是当人为导数的时候，比如insert into tb select * from .. 涉及100万行记录的时候，如果binlog_formant为row格式，那么产生的binlog文件将非常大，而且再传给从库落地为relay log也很大，这占用了一定量的IO资源，这个时候，可以在操作之前，先修改当前会话级别为 set binlog_formant='statement'; 再执行insert into tb select * from ..，那么它就仅记录这单条SQL，会话级别的binlog_format修改，不会影响整体的同步情况。

### binlog日志结构

https://www.jianshu.com/p/6523693d6fd7

### binlog事件类型

Binlog 日志中的事件，不同的操作会对应着不同的事件类型，且不同的 Binlog 日志模式同一个操作的事件类型也不同，下面我们一起看看常见的事件类型。

首先我们看看源码中的事件类型定义：

源码位置：/libbinlogevents/include/binlog_event.h

这么多的事件类型我们就不一一介绍，挑出来一些常用的来看看。

**FORMAT_DESCRIPTION_EVENT**

FORMAT_DESCRIPTION_EVENT 是 Binlog V4 中为了取代之前版本中的 START_EVENT_V3 事件而引入的。它是 Binlog 文件中的第一个事件，而且，该事件只会在 Binlog 中出现一次。MySQL 根据 FORMAT_DESCRIPTION_EVENT 的定义来解析其它事件。

它通常指定了 MySQL 的版本，Binlog 的版本，该 Binlog 文件的创建时间。

**QUERY_EVENT**

QUERY_EVENT 类型的事件通常在以下几种情况下使用：

- 事务开始时，执行的 BEGIN 操作
- STATEMENT 格式中的 DML 操作
- ROW 格式中的 DDL 操作

比如上文我们插入一条数据之后的 Binlog 日志：

```mysql
Copymysql> show binlog events in 'mysql-bin.000002';
+------------------+-----+----------------+-----------+-------------+-------------------------------------------------------------------------+
| Log_name         | Pos | Event_type     | Server_id | End_log_pos | Info                                                                    |
+------------------+-----+----------------+-----------+-------------+-------------------------------------------------------------------------+
| mysql-bin.000002 |   4 | Format_desc    |         1 |         125 | Server ver: 8.0.21, Binlog ver: 4                                       |
| mysql-bin.000002 | 125 | Previous_gtids |         1 |         156 |                                                                         |
| mysql-bin.000002 | 156 | Anonymous_Gtid |         1 |         235 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                    |
| mysql-bin.000002 | 235 | Query          |         1 |         323 | BEGIN                                                                   |
| mysql-bin.000002 | 323 | Intvar         |         1 |         355 | INSERT_ID=13                                                            |
| mysql-bin.000002 | 355 | Query          |         1 |         494 | use `test_db`; INSERT INTO `test_db`.`test_db`(`name`) VALUES ('xdfdf') |
| mysql-bin.000002 | 494 | Xid            |         1 |         525 | COMMIT /* xid=192 */                                                    |
+------------------+-----+----------------+-----------+-------------+-------------------------------------------------------------------------+
7 rows in set (0.00 sec)
```

**XID_EVENT**

在事务提交时，不管是 STATEMENT 还 是ROW 格式的 Binlog，都会在末尾添加一个 XID_EVENT 事件代表事务的结束。该事件记录了该事务的 ID，在 MySQL 进行崩溃恢复时，根据事务在 Binlog 中的提交情况来决定是否提交存储引擎中状态为 prepared 的事务。

**ROWS_EVENT**

对于 ROW 格式的 Binlog，所有的 DML 语句都是记录在 ROWS_EVENT 中。

ROWS_EVENT分为三种：

- WRITE_ROWS_EVENT
- UPDATE_ROWS_EVENT
- DELETE_ROWS_EVENT

分别对应 insert，update 和 delete 操作。

对于 insert 操作，WRITE_ROWS_EVENT 包含了要插入的数据。

对于 update 操作，UPDATE_ROWS_EVENT 不仅包含了修改后的数据，还包含了修改前的值。

对于 delete 操作，仅仅需要指定删除的主键（在没有主键的情况下，会给定所有列）。

**对比 QUERY_EVENT 事件，是以文本形式记录 DML 操作的。而对于 ROWS_EVENT 事件，并不是文本形式，所以在通过 mysqlbinlog 查看基于 ROW 格式的 Binlog 时，需要指定 `-vv --base64-output=decode-rows`。**

我们来测试一下，首先将日志格式改为 Rows：

```mysql
Copymysql> set binlog_format=row;
Query OK, 0 rows affected (0.00 sec)

mysql>
mysql> flush logs;
Query OK, 0 rows affected (0.01 sec)
```

然后刷新一下日志文件，重新开始一个 Binlog 日志。我们插入一条数据之后看一下日志：

```mysql
Copymysql> show binlog events in 'binlog.000008';
+---------------+-----+----------------+-----------+-------------+--------------------------------------+
| Log_name      | Pos | Event_type     | Server_id | End_log_pos | Info                                 |
+---------------+-----+----------------+-----------+-------------+--------------------------------------+
| binlog.000008 |   4 | Format_desc    |         1 |         125 | Server ver: 8.0.21, Binlog ver: 4    |
| binlog.000008 | 125 | Previous_gtids |         1 |         156 |                                      |
| binlog.000008 | 156 | Anonymous_Gtid |         1 |         235 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS' |
| binlog.000008 | 235 | Query          |         1 |         313 | BEGIN                                |
| binlog.000008 | 313 | Table_map      |         1 |         377 | table_id: 85 (test_db.test_db)       |
| binlog.000008 | 377 | Write_rows     |         1 |         423 | table_id: 85 flags: STMT_END_F       |
| binlog.000008 | 423 | Xid            |         1 |         454 | COMMIT /* xid=44 */                  |
+---------------+-----+----------------+-----------+-------------+--------------------------------------+
7 rows in set (0.01 sec)
```

end

### binlog恢复数据

https://www.cnblogs.com/michael9/p/11923483.html

#### 总结

使用row-based replication格式的binlog日志，不太好方便使用mysqlbinlog来恢复数据。因为它的格式太变态...

```bash
### DELETE FROM `test_binlog`.`sync_test`
### WHERE
###   @1=3
###   @2='c'
```

恢复之前为了避免产生没有用的二进制日志，可以关闭二进制日志的记录
SET SESSION sql_log_bin=0;

恢复完成，启用记录二进制日志
SET SESSION sql_log_bin=1;

#### 误删行恢复

1. 使用`DELETE`语句误删除了**数据行**，可以使用`Flashback`通过闪回把数据恢复
2. `Flashback`恢复数据的原理：修改`binlog`的内容，然后拿到原库重放
   - 前提：`binlog_format=ROW`和`binlog_row_image=FULL`
3. 针对单个事务
   - 对于`INSERT`语句，将`Write_rows event`改成`Delete_rows event`
   - 对于`DELETE`语句，将`Delete_rows event`改成`Write_rows event`
   - 对于`UPDATE`语句，`binlog`里面记录了数据行修改前和修改后的值，**对调两行的位置即可**
4. 针对多个事务
   - 误操作
     - (A)DELETE
     - (B)INSERT
     - (C)UPDTAE
   - Flashback
     - (REVERSE C)UPDATE
     - (REVERSE B)DELETE
     - (REVERSE A)INSERT
5. 不推荐直接在主库上执行上述操作，避免造成二次破坏
   - 比较安全的做法是先恢复出一个备份或找一个从库作为**临时库**
   - 在临时库上执行上述操作，然后再将**确认过**的临时库的数据，恢复到主库
6. 预防措施
   - `sql_safe_updates=ON`，下列情况会报错
     - 没有`WHERE`条件的`DELETE`或`UPDATE`语句
     - `WHERE`条件里面**没有包含索引字段的值**
   - 上线前，必须进行**SQL审计**
7. 删全表的性能
   - `DELETE`全表**很慢**，因为需要生成`undolog`、写`redolog`和写`binlog`
   - 优先考虑使用`DROP TABLE`或`TRUNCATE TABLE`

使用 delete 语句误删了数据行，可以用 Flashback 工具通过闪回把数据恢复回来。而能够使用这个方案的前提是，需要确保 binlog_format=row 和 binlog_row_image=FULL。

Flashback 恢复数据的原理，是修改 binlog 的内容，拿回原库重放。

1. 对于 insert 语句，对应的 binlog event 类型是 Write_rows event，把它改成 Delete_rows event 即可；
2. 对于 delete 语句，也是将 Delete_rows event 改为 Write_rows event；
3. 而如果是 Update_rows 的话，binlog 里面记录了数据行修改前和修改后的值，对调这两行的位置即可。

但是，delete 全表是很慢的，需要生成回滚日志、写 redo、写 binlog。所以，从性能角度考虑，你应该优先考虑使用 truncate table 或者 drop table 命令。

**truncate直接是以一条语句复杂过来，而delete命令是直接把每条删除的语句一条条记录下来，所以假如要删除全表的数据，用truncate命令会比用delete命令少记录很多内容，日志内容会大减少，肯效率会更高。**

1) 准备数据

```bash
# 创建临时数据库
CREATE DATABASE IF NOT EXISTS test_binlog \
default charset utf8 COLLATE utf8_general_ci; 


# 创建临时表
CREATE TABLE `sync_test` (`id` int(11) \
NOT NULL AUTO_INCREMENT, `name` varchar(255) NOT NULL,  \
PRIMARY KEY (`id`) ) ENGINE=InnoDB DEFAULT CHARSET=utf8;

# 添加数据
insert into sync_test (id, name) values (null, 'a');
insert into sync_test (id, name) values (null, 'b');
insert into sync_test (id, name) values (null, 'c');

# 查看添加的数据
select * from sync_test;
```

end

2) 模拟误删除删除记录

```bash
# 删除 id 小于3 的数据
delete from sync_test where id<3

# 插入几条数据
insert into sync_test (id, name) values (null, 'd');
insert into sync_test (id, name) values (null, 'e');
insert into sync_test (id, name) values (null, 'f');

# 最终表里数据

mysql> select * from sync_test;
+----+------+
| id | name |
+----+------+
|  3 | c    |
|  4 | d    |
|  5 | e    |
|  6 | f    |
+----+------+

```

end

3) 使用binlog恢复数据

删除行简单啊，使用binlog日志定位到delete的时间点，然后将delete 语句变成inster即可。简单

```bash
##  先生成新的binlog 文件，方便区别
flush logs;

## 定位delete 语句位置
## binlog_format = statement, 如下很不清晰不好恢复
$ mysqlbinlog /var/lib/mysql/mysql-bin.000001
...
# at 2327
#210714  3:02:55 server id 3  end_log_pos 2447 CRC32 0x9ac9b0de         Query   thread_id=4     exec_time=0     error_code=0
SET TIMESTAMP=1626231775/*!*/;
delete from sync_test where id<3
/*!*/;
# at 2447
#210714  3:02:55 server id 3  end_log_pos 2478 CRC32 0x947c227a         Xid = 34
COMMIT/*!*/;

...

```

end

##### RBR模式: 很清晰

这个FlashBack工具就是帮你把这个binlog中的日志提取出去，然后转换成insert操作，进行还原的。搜噶--

```bash
mysql> show variables like '%binlog_format';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| binlog_format | ROW   |
+---------------+-------+
1 row in set (0.00 sec)

mysql> select @@global.binlog_format;                                                                                                      +------------------------+
| @@global.binlog_format |
+------------------------+
| ROW                    |
+------------------------+
1 row in set (0.00 sec)

mysql> select * from test_sync;
ERROR 1146 (42S02): Table 'test_binlog.test_sync' doesn't exist
mysql> select * from sync_test;
+----+------+
| id | name |
+----+------+
|  3 | c    |
|  4 | d    |
|  5 | e    |
|  6 | f    |
+----+------+
4 rows in set (0.00 sec)

mysql> flush logs;
Query OK, 0 rows affected (0.00 sec)

mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000004 |      360 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
Query OK, 3 rows affected (0.00 sec)

mysql> select * from sync_test;
+----+------+
| id | name |
+----+------+
|  6 | f    |
+----+------+
1 row in set (0.00 sec)

mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000004 |      360 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)

mysql> show BINLOG events in 'mysql-bin.000004'
    -> ;
+------------------+-----+-------------+-----------+-------------+------------------------------------------+
| Log_name         | Pos | Event_type  | Server_id | End_log_pos | Info                                     |
+------------------+-----+-------------+-----------+-------------+------------------------------------------+
| mysql-bin.000004 |   4 | Format_desc |         3 |         120 | Server ver: 5.7.5-m15-log, Binlog ver: 4 |
| mysql-bin.000004 | 120 | Query       |         3 |         208 | BEGIN                                    |
| mysql-bin.000004 | 208 | Table_map   |         3 |         270 | table_id: 97 (test_binlog.sync_test)     |
| mysql-bin.000004 | 270 | Delete_rows |         3 |         329 | table_id: 97 flags: STMT_END_F           |
| mysql-bin.000004 | 329 | Xid         |         3 |         360 | COMMIT /* xid=109 */                     |
+------------------+-----+-------------+-----------+-------------+------------------------------------------+
5 rows in set (0.00 sec)

## 这样可以看到 执行的详细sql...
mysql> ^DBye
root@b3bcbbb0f6ce:/# mysqlbinlog --no-defaults --base64-output=decode-rows -v /var/lib/mysql/mysql-bin.000004
...
/*!*/;
# at 208
#210714  6:38:54 server id 3  end_log_pos 270 CRC32 0x309cf6da  Table_map: `test_binlog`.`sync_test` mapped to number 97
# at 270
#210714  6:38:54 server id 3  end_log_pos 329 CRC32 0xd5f7d83b  Delete_rows: table id 97 flags: STMT_END_F
### DELETE FROM `test_binlog`.`sync_test`
### WHERE
###   @1=3
###   @2='c'
### DELETE FROM `test_binlog`.`sync_test`
### WHERE
###   @1=4
###   @2='d'
### DELETE FROM `test_binlog`.`sync_test`
### WHERE
###   @1=5
###   @2='e'
# at 329
#210714  6:38:54 server id 3  end_log_pos 360 CRC32 0xb95dab66  Xid = 109
COMMIT/*!*/;
DELIMITER ;
# End of log file
ROLLBACK /* added by mysqlbinlog */;
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
```

end

#### 误删表恢复（重要）

核心思路：

* 通过`show master status`查看当前binlog 文件编号，并定位到drop active position

* 将binlog 编号之前的所以binlog files 读取到一个sql文件中

  > 因为drop所在的binlog 文件不一定或者说肯定没包含改表的创建定义语句。

* 过滤sql文件，仅找到于删除表相关的sql语句，用来恢复table

如果从上次备份刷新binlog，到发现表被删掉的过程中产生了多个binlog。则要按照binlog产生的顺序进行恢复，那
么恢复的次序应该是按照binglog的产生的序号，从小到大依次恢复。
假如从上次备份，到发现表被删除，共有两个binlog文件，分别是test-150-bin.000002，test-150-bin.000003 ，则按照binlog序号从小到大的排列，恢复的顺序应该是：

```bash
root@b3bcbbb0f6ce:/# mysqlbinlog -d test_binlog  /var/lib/mysql/mysql-bin.000001  > /tmp/table_sync_test.sql
...
# 下面是追加哦
# 这个 --stop-position比较重要，找到drop table的position 然后避免它。
root@b3bcbbb0f6ce:/# mysqlbinlog -d test_binlog  --stop-position=1552   /var/lib/mysql/mysql-bin.000007  >> /tmp/table_sync_test.sql
```



使用 truncate /drop table 和 drop database 命令删除的数据，就没办法通过 Flashback 来恢复了。因为即使我们配置了 binlog_format=row，执行这三个命令时，记录的 binlog 还是 statement 格式。binlog 里面就只有一个 truncate/drop 语句，这些信息是恢复不出数据的。

这个时候就需要使用全量备份，加增量日志了，这个方案要求线上有定期的全量备份，并且实时备份 binlog。

恢复数据的流程如下:

1. 取最近一次全量备份，假设这个库是一天一备，上次备份是当天 0 点；
2. 用备份恢复出一个临时库；
3. 从日志备份里面，取出凌晨 0 点之后的日志；
4. 把这些日志，除了误删除数据的语句外，全部应用到临时库

模拟有两条误删的操作，数据行的误删和表的误删。有两种方式进行恢复。

- 方式一：首先恢复到删除表操作之前的位置，然后再单独恢复误删的数据行。
- 方式二：首先恢复到误删数据行的之前的位置，然后跳过误删事件再恢复数据表操作之前的位置。

这里采用方式一的方案进行演示，由于是演示，就不额外找一个临时库进行全量恢复了，直接进行操作。

```bash
## 还使用上面的sync_test表
## 插入数据
insert into sync_test (id, name) values (null, 'gg');
insert into sync_test (id, name) values (null, 'kk');
insert into sync_test (id, name) values (null, 'aaa');

## 查看当前binlog日志文件
mysql> select * from sync_test;
+----+------+
| id | name |
+----+------+
|  6 | f    |
|  7 | aa   |
|  8 | gg   |
|  9 | kk   |
| 10 | aaa  |
+----+------+
5 rows in set (0.00 sec)

mysql> drop table sync_test;
Query OK, 0 rows affected (0.01 sec)

mysql> flush logs;
Query OK, 0 rows affected (0.00 sec)

mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000008 |      120 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)

mysql> ^DBye

## 定位 drop table position
$ mysqlbinlog -d test_binlog  --stop-position=1552 /var/lib/mysql/mysql-bin.000007

## 恢复数据
root@b3bcbbb0f6ce:/# mysqlbinlog -d test_binlog  /var/lib/mysql/mysql-bin.000001  > /tmp/table_sync_test.sql
...
# 下面是追加哦
# 这个 --stop-position比较重要，找到drop table的position 然后避免它。
root@b3bcbbb0f6ce:/# mysqlbinlog -d test_binlog  --stop-position=1552   /var/lib/mysql/mysql-bin.000007  >> /tmp/table_sync_test.sql

# 过滤数据 仅恢复指定表 sync_test
$ cat /tmp/table_sync_test.sql | grep  -A1 -B3 -i -E '^create|^insert|^update|^delete|^replace|^alter' | grep -A1 -B3 sync_test > recover.sql

# 导入数据
mysql> source /recover.sql
...
Query OK, 1 row affected (0.00 sec)
# ...多了几条怎么回事
# 是因为上次的delete是在 binlog_format=row小操作的日志格式不正确，使用grep -A1 -B3 过滤不出来
mysql> select * from sync_test;
+----+------+
| id | name |
+----+------+
|  3 | c    |
|  4 | d    |
|  5 | e    |
|  6 | f    |
|  7 | aa   |
|  8 | gg   |
|  9 | kk   |
| 10 | aaa  |
+----+------+

```

end

#### binlog 落盘时机

对于 `InnoDB `存储引擎而言，只有在事务提交时才会记录 `biglog `，此时记录还在内存中，那么 `biglog`
是什么时候刷到磁盘中的呢？ `mysql `通过 `sync_binlog `参数控制 `biglog `的刷盘时机，取值范围是 `0-N`
：

- 0：不去强制要求，由系统自行判断何时写入磁盘；
- 1：每次 `commit `的时候都要将 `binlog `写入磁盘；
- N：每N个事务，才会将 `binlog `写入磁盘。

从上面可以看出， `sync_binlog `最安全的是设置是 `1 `，这也是 `MySQL 5.7.7`
之后版本的默认值。但是设置一个大一些的值可以提升数据库性能，因此实际情况下也可以将值适当调大，牺牲一定的一致性来获取更好的性能。



#### binlog日志清理

- expire_logs_days+flush logs
  * 设置expire_logs_days，假设当前binlog文件过多且大占用磁盘空间，可以修改小改参数，改参数只有在切换新的binlog文件时，才会删除过期文件，也就是可以等数据库把当前binlog写满后切换到新文件的时候删除，也可以手动执行flush logs，手动切换binlog，同时会触发对过期binlog文件的删除。注意从库的同步情况来设置。 

- purge binary logs to 'xxxx'
  - 清理 'xxxx' 之前的 binary log文件，只保留 'xxxx' 之后的binary log文件
  - 但是使用这个指令的时候，要注意主从同步情况，如果在主库执行，一定要去从库 SHOW SLAVE STATUS，看下当前同步到哪个binlog文件，防止清理报错

## master-slave复制核心

主库已经开启了binlog，并正常的记录binlog。

首先从库启动**I/O线程**，跟主库建立客户端连接。

主库启动**binlog dump**线程，读取主库上的binlog event发送给从库的I/O线程，I/O线程获取到binlog event之后将其写入到自己的Relay Log中。

然后从库启动**SQL线程**，将Relay中的数据进行重放，完成从库的数据更新。

总结来说，主库上只会有一个线程，而从库上则会有两个线程。

![binlog-and-relay](D:\worker\h3c\数据库&&mysql\images\binlog-and-relay.jpg)

记住，这几个进程--

relay log其实和binlog没有太大的区别，在MySQL 4.0 之前是没有Relay Log这部分的，整个过程中只有两个线程。但是这样也带来一个问题，那就是复制的过程需要同步的进行，很容易被影响，而且效率不高。例如主库必须要等待从库读取完了才能发送下一个binlog事件。这就有点类似于一个阻塞的信道和非阻塞的信道。

**阻塞信道**就跟你在柜台一样，你要递归柜员一个东西，但是你和柜员之间没有可以放东西的地方，你就只能一直把文件拿着，直到柜员接手；而**非阻塞信道**就像你们之间有个地方可以放文件，你就直接放上去就好了，不用等柜员接手。

引入了Relay Log之后，让原本同步的获取事件、重放事件解耦了，两个步骤可以异步的进行，Relay Log充当了缓冲区的作用。Relay Log有一个`relay-log.info`的文件，用于记录当前复制的进度，下一个事件从什么Pos开始写入，该文件由SQL线程负责更新。

###  relay耗时问题导致数据不一致：看

场景，主节点的binlog被 Binlog Dump线程 无差别的传输到从节点的relay log中了，但是由于binlog日志格式可能是RAW模式，大量数据写入Relay Log，这时Slave要耗时6-7小时去relay 日志到 slave的binlog中，在这relay的过程中如果主库down掉，那么切到slave上必然是会出现数据不同步的。

理解了这三个线程的作用就好理解了。

所以，业界的多主双活几乎是不存在的...

一旦，relay没结束时，master挂掉了，slave必然和master数据不一致。



## relay-log（流弊）

Are relay logs necessary for replication?

Yes.

relay的作用是从上游 MySQL/MariaDB 读取 binlog event 并写入到本地的 relay log file 中，然后再将relay中的sql写入slave 的 binlog。

你肯定会问，为什么你不直接将上游的Mysql binlog 同步到 下游slave的binlog中呢，非得新加一个relay.

首先 ，relay log像中间件一样，保证了收到的binlog event的完整性，避免了event错乱或丢失。

其次，这也是很重要的，relay log 保证了binlog的高内聚性。binlog就是复制记录实例执行的sql语句，很存粹，不掺杂其他。relay log 记录上游的binlog，然后读取relay log(经过relay log的清洗可以确保sql event的完整性)在本地实例执行sql event，再写入本地实例的binlog。这样完成了一次同步。

binlog 和 relay 的分离，完全符合高内聚，低耦合的特性。

这样的好处，当master 挂掉后，直接切换slave变成master使用，这时binlog还是保持它纯粹的功能，只记录本实例上执行的sql。

栗子：

```bash
## master 执行了下面操作


## slave 收到的同步数据

```



由于网络等原因，Binary log不可能一口气存到 I/O thread中，所以Relay log中用来缓存Binary log的事件。（Relay log存储在从服务器slave缓存中，开销比较小）

Relay log，我们翻译成中文，一般叫做中继日志，一般情况下它在MySQL主从同步读写分离集群的从节点才开启。主节点一般不需要这个日志。

master主节点的binlog传到slave从节点后，被写道relay log里，从节点的slave sql线程从relaylog里读取日志然后应用到slave从节点本地。从服务器I/O线程将主服务器的二进制日志读取过来记录到从服务器本地文件，然后SQL线程会读取relay-log日志的内容并应用到从服务器，从而使从服务器和主服务器的数据保持一致。

relay log是一个中介临时的日志文件，用于存储从master节点同步过来的binlog日志内容，它里面的内容和master节点的binlog日志里面的内容是一致的。然后slave从节点从这个relay log日志文件中读取数据应用到数据库中，来实现数据的主从复制。

relay相关参数

http://www.dreamwu.com/post-1865.html、

## WAL

数据真正持久化之前先把变更写入 log 的方式就叫做 WAL（Write Ahead Logging）既然实现的方式是 write ahead，那么它的作用自然就是 read after 了。

转账的例子我们假设 A 一开始有 100 块钱，B 有 0 一个完整的 log 就是：

\1. Begin

\2. <1, A, 100, 0>

\3. <1, B, 0, 100>

\4. Commit

由于 WAL 是 write ahead 的，也就意味着即使 Commit 日志已经写成功了，但数据库还没真正把事务提交。而且一般数据库掉电后也不会让人和 Word 一样手工上去判断每个事务是否提交，这就需要数据库能根据 log 来自动的 commit 和 rollback 事务。

这个事情也简单只要看一下 WAL 中有哪些事务是数据库还没做完的，把这些事务的 log 找出来。如果这个事务最后一条记录是 Commit，那么由于 Write ahead 的特性可能还没写进去我们只需要根据 log 里的 Entry 和 NewValue 把值不管三七二十一覆盖一下就好了，这样就达到了和 WAL 一致的状态。如果事务最后一条记录不是 Commit，那么这个事务肯定没有执行完，我们根据 OldValue 的值进行覆盖就相当于 rollback 到了事务开始前的值。

还是刚才那个 log，如果在 4 之后发生断电，那么恢复后数据库要执行的就是把 A set 成 0 把 B set 成 100。如果 3 和 4 之间断电那么就要把 A set 成 100，B set 成 0。如果 2 和 3 之间断电需要把 A set 成 100。1 和 2 之间，由于没记录也就不用操作了。这样无论哪种情况事务最终的一致性都是得到保证的。

除了带来事务一致性的保证，由于只需要把操作写到 WAL 里就可以认为操作完成而无需等待持久化真正的数据库变更完成就可以返回，数据库操作的效率也得到了一些提升。你可能要问了写 log 也要持久化呀那里性能提升了？窍门在于 WAL 是顺序写入的一直在文件末尾 append，而持久化数据库的数据是一个随机写入操作，顺序写会节省大量的磁盘悬臂来回寻址的过程，效率要高好几个量级。

一般来说 SSD 的顺序写还是要比随机写好点....



**「预写式日志」**（Write-ahead logging，缩写 WAL）是关系数据库系统中用于提供原子性和持久性（ACID 属性中的两个）的一系列技术。在使用 WAL 的系统中，所有的修改在提交之前都要先写入 log 文件中

og 文件中通常包括 redo 和 undo 信息。这样做的目的可以通过一个例子来说明。假设一个程序在执行某些操作的过程中机器掉电了。在重新启动时，程序可能需要知道当时执行的操作是成功了还是部分成功或者是失败了。如果使用了 WAL，程序就可以检查 log 文件，并对突然掉电时计划执行的操作内容跟实际上执行的操作内容进行比较。在这个比较的基础上，程序就可以决定是撤销已做的操作还是继续完成已做的操作，或者是保持原样。

WAL 允许用 in-place 方式更新数据库。另一种用来实现原子更新的方法是 shadow paging，它并不是 in-place 方式。用 in-place 方式做更新的主要优点是减少索引和块列表的修改。ARIES 是 WAL 系列技术常用的算法。在文件系统中，WAL 通常称为 journaling。PostgreSQL也是用 WAL 来提供 point-in-time 恢复和数据库复制特性。

###备份实现原子性和持久性

我们想一想，如果想保证对一个数据的操作可以恢复。可以怎么做？你不用去想数据库是怎么实现的，也不用想太高深。其实这是一个很简单的问题，我们常常在处理这种问题。最简单的方法其实就是备份一份数据：当我需要对一条数据做更新操作前，先将这条数据备份在一个地方，然后去更新，如果更新失败，可以从备份数据中回写回来。这样就可以保证事务的回滚，就可以保证数据操作的原子性了。其实 SQLite 引入 WAL 之前就是通过这种方式来实现原子事务，称之为 rollback journal， rollback journal 机制的原理是：在修改数据库文件中的数据之前，先将修改所在分页中的数据备份在另外一个地方，然后才将修改写入到数据库文件中；如果事务失败，则将备份数据拷贝回来，撤销修改；如果事务成功，则删除备份数据，提交修改。

### WAL实现原子性和持久性

再继续上面的问题？如何做到数据的可恢复（原子性）和提交成功的数据被持久化到磁盘（持久性）？另一种机制就是WAL，WAL 机制的原理也很简单：**「修改并不直接写入到数据库文件中，而是写入到另外一个称为 WAL 的文件中；如果事务失败，WAL 中的记录会被忽略，撤销修改；如果事务成功，它将在随后的某个时间被写回到数据库文件中，提交修改。」**

### WAL 的优点

1. 读和写可以完全地并发执行，不会互相阻塞（但是写之间仍然不能并发）。
2. WAL 在大多数情况下，拥有更好的性能（因为无需每次写入时都要写两个文件）。
3. 磁盘 I/O 行为更容易被预测。
4. 使用更少的 fsync()操作，减少系统脆弱的问题。

我们都知道，数据库的最大性能挑战就是磁盘的读写，许多先辈在提供数据存储性能上绞尽脑汁，提出和实验了一套又一套方法。其实所有方案最终总结出来就三种：**「随机读写改顺序读写」**、**「缓冲单条读写改批量读写」**、**「单线程读写改并发读写」**。WAL 其实也是这两种思路的一种实现，一方面 WAL 中记录事务的更新内容，通过 WAL 将随机的脏页写入变成顺序的日志刷盘，另一方面，WAL 通过 buffer 的方式改单条磁盘刷入为缓冲批量刷盘，再者从 WAL 数据到最终数据的同步过程中可以采用并发同步的方式。这样极大提升数据库写入性能，因此，WAL 的写入能力决定了数据库整体性能的上限，尤其是在高并发时。

### WAL checkpoint

上面讲到，使用 WAL 的数据库系统不会再每新增一条 WAL 日志就将其刷入数据库文件中，一般积累一定的量然后批量写入，通常使用**「页」**为单位，这是磁盘的写入单位。同步 WAL 文件和数据库文件的行为被称为 checkpoint（检查点），一般在 WAL 文件积累到一定页数修改的时候；当然，有些系统也可以手动执行 checkpoint。执行 checkpoint 之后，WAL 文件可以被清空，这样可以保证 WAL 文件不会因为太大而性能下降。

> WAL 也不能太小--

有些数据库系统读取请求也可以使用 WAL，通过读取 WAL 最新日志就可以获取到数据的最新状态。

## redo and undo 日志详解

https://www.cnblogs.com/f-ck-need-u/p/9010872.html#auto_id_14

## redo

http://mysql.taobao.org/monthly/2015/05/01/

https://blog.csdn.net/qq_35246620/article/details/79345359

https://www.huaweicloud.com/articles/3d4508df9e2945703fbebff812f4541b.html



redo log 和 binlog 的区别：

1. redo log 是 InnoDB 引擎特有的；binlog 是 MySQL 的 Server 层实现的，所有引擎都可以使用。

   > Mysql 整体就分为两部分：Server层实现Mysql的功能和引擎层来实现存储相关。

2. redo log 是物理日志，记录的是在某个数据页上做了什么修改；binlog 是逻辑日志，记录的是这个语句的原始逻辑。

3. redo log 是循环写的，空间固定会用完；binlog 是可以追加写入的。追加写是指 binlog 文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。

redo log 具有 **crash-safe 的能力**，而 binlog 没有。

`redo log`不是二进制日志。它记录的是物理持久化日志。

二进制日志是在存储引擎的上层产生的，不管是什么存储引擎，对数据库进行了修改都会产生二进制日志。而`redo log`是 InnoDB 层产生的，只记录该存储引擎中表的修改。并且二进制日志先于`redo log`被记录

在概念上，InnoDB 通过`force log at commit`机制实现事务的持久性，即在事务提交的时候，必须先将该事务的所有事务日志写入到磁盘上的`redo log file`和`undo log file`中进行持久化。

为了确保每次日志都能写入到事务日志文件中，在每次将`log buffer`中的日志写入日志文件的过程中都会调用一次操作系统的`fsync`操作（即`fsync()`系统调用）。因为 MariaDB/MySQL 是工作在用户空间的，MariaDB/MySQL 的`log buffer`处于用户空间的内存中。要写入到磁盘上的`log file`中（`redo:ib_logfileN`文件、`undo:share tablespace`或`.ibd`文件），中间还要经过操作系统内核空间的`os buffer`，调用`fsync()`的作用就是将`os buffer`中的日志刷到磁盘上的`log file`中。

也就是说，从`redo log buffer`写日志到磁盘的`redo log file`中



那就是 InnoDB 并不是 MySQL 的原生存储引擎。MySQL 的原生引擎是 MyISAM，设计之初就有没有支持崩溃恢复。InnoDB 在作为 MySQL 的插件加入 MySQL 引擎家族之前，就已经是一个提供了崩溃恢复和事务支持的引擎了。InnoDB 接入了 MySQL 后，发现既然 binlog 没有崩溃恢复的能力，那就用 InnoDB 原有的 redo log 好了。



InnoDB 引擎使用的是 WAL 技术，执行事务的时候，写完内存和日志，事务就算完成了。如果之后崩溃，要依赖于日志来恢复数据页。

binlog 里面并没有记录数据页的更新细节，是补不回来的。你如果要说，那我优化一下 binlog 的内容，让它来记录数据页的更改可以吗？但，这其实就是又做了一个 redo log 出来。所以，至少现在的 binlog 能力，还不能支持崩溃恢复。

binlog 有着 redo log 无法替代的功能。

一个是归档。redo log 是循环写，写到末尾是要回到开头继续写的。这样历史日志没法保留，redo log 也就起不到归档的作用。

一个就是 MySQL 系统依赖于 binlog。binlog 作为 MySQL 一开始就有的功能，被用在了很多地方。其中，MySQL 系统高可用的基础，就是 binlog 复制。

PS: Mysql最初设计的引擎是MISAM，InnoDB一开始只是插件，后来才上位--

MyISAM was the default storage engine from MySQL 3.23 until it was replaced by InnoDB in MariaDB and MySQL 5.5. It's a light, non-transactional engine with great performance, is easy to copy between systems and has a small data footprint.

### redo日志

https://segmentfault.com/a/1190000023827696

`redo log `包括两部分：一个是内存中的日志缓冲( `redo log buffer `)，另一个是磁盘上的日志文件( ` redo log
file `)。 `mysql `每执行一条 `DML `语句，先将记录写入 `redo log buffer `
，后续某个时间点再一次性将多个操作记录写到 `redo log file `。这种 **先写日志，再写磁盘** 的技术就是 `MySQL`
里经常说到的 `WAL(Write-Ahead Logging) `技术。

在计算机操作系统中，用户空间( `user space `)下的缓冲区数据一般情况下是无法直接写入磁盘的，中间必须经过操作系统内核空间( `
kernel space `)缓冲区( `OS Buffer `)。因此， `redo log buffer `写入 `redo log
file `实际上是先写入 `OS Buffer `，然后再通过系统调用 `fsync() `将其刷到 `redo log file `
中，过程如下：

`mysql `支持三种将 `redo log buffer `写入 `redo log file `的时机，可以通过 `
innodb_flush_log_at_trx_commit ` 参数配置，各参数值含义如下：

| 参数值              | 含义                                                         |
| ------------------- | ------------------------------------------------------------ |
| 0（延迟写）         | 事务提交时不会将 `redo log buffer `中日志写入到 `os buffer `，而是每秒写入 `os buffer `并调用 `fsync() `写入到 `redo log file `中。也就是说设置为0时是(大约)每秒刷新写入到磁盘中的，当系统崩溃，会丢失1秒钟的数据。 |
| 1（实时写，实时刷） | 事务每次提交都会将 `redo log buffer `中的日志写入 `os buffer `并调用 `fsync() `刷到 `redo log file `中。这种方式即使系统崩溃也不会丢失任何数据，但是因为每次提交都写入磁盘，IO的性能较差。 |
| 2（实时写，延迟刷） | 每次提交都仅写入到 `os buffer `，然后是每秒调用 `fsync() `将 `os buffer `中的日志写入到 `redo log file `。 |

![img](https://segmentfault.com/img/remote/1460000023827700)

### redo log记录形式

 `redo log `实际上记录数据页的变更，而这种变更记录是没必要全部保存，因此 `redo log`
实现上采用了大小固定，循环写入的方式，当写到结尾时，会回到开头循环写日志。

数据页：InnoDB 的叶子段存储了数据页，数据页中保存了行记录。

因为之前的数据已经



## undo

Undo log是存放在共享表空间里面的（ibdata*文件）。

`undo log`有两个作用：提供回滚和多个行版本控制（`MVCC`）

在数据修改的时候，不仅记录了`redo log`，还记录了相对应的`undo log`，如果因为某些原因导致事务失败或回滚了，可以借助该`undo log`进行回滚。

`undo log`和`redo log`记录物理日志不一样，它是逻辑日志。可以认为当`delete`一条记录时，`undo log`中会记录一条对应的`insert`记录，反之亦然，当`update`一条记录时，它记录一条对应相反的`update`记录。

`undo log`也会产生`redo log`，因为`undo log`也要实现持久性保护。

**原子性** 底层就是通过 `undo log `实现的。 `undo log `主要记录了数据的逻辑变化，比如一条 ` INSERT
`语句，对应一条 `DELETE `的 `undo log `，对于每个 `UPDATE `语句，对应一条相反的 `UPDATE `的`
undo log `，这样在发生错误时，就能回滚到事务之前的数据状态。同时， `undo log `也是 `MVCC `(多版本并发控制)实现的关键，

undo log 主要用于实现 MVCC，从而实现 MySQL 的 ”读已提交“、”可重复读“ 隔离级别。在每个行记录后面有两个隐藏列，"trx_id"、"roll_pointer"，分别表示上一次修改的事务id，以及 "上一次修改之前保存在 undo log中的记录位置 "。在对一行记录进行修改或删除操作前，会先将该记录拷贝一份到 undo log 中，然后再进行修改，并将修改事务 id，拷贝的记录在 undo log 中的位置写入 "trx_id"、"roll_pointer"。

### undo落盘机制？

MySQL中的Undo Log严格的讲不是Log，而是数据，因此他的管理和落盘都跟数据是一样的：

- Undo的磁盘结构并不是顺序的，而是像数据一样按Page管理
- Undo写入时，也像数据一样产生对应的Redo Log
- Undo的Page也像数据一样缓存在Buffer Pool中，跟数据Page一起做LRU换入换出，以及刷脏。Undo Page的刷脏也像数据一样要等到对应的Redo Log 落盘之后

之所以这样实现，首要的原因是MySQL中的Undo Log不只是承担Crash Recovery时保证Atomic的作用，更需要承担MVCC对历史版本的管理的作用，设计目标是高事务并发，方便的管理和维护。因此当做数据更合适

但既然还叫Log，就还是需要有Undo Log的责任，那就是保证Crash Recovery时，如果看到数据的修改，一定要能看到其对应Undo的修改，这样才有机会通过事务的回滚保证Crash Atomic。标准的Undo Log这一步是靠WAL实现的，也就是要求Undo写入先于数据落盘。而InnoDB中Undo Log作为一种特殊的数据，这一步是通过redo的min-transaction保证的，简单的说就是数据的修改和对应的Undo修改，他们所对应的redo log被放到同一个min-transaction中，同一个min-transaction中的所有redo log在Crash Recovery时以一个整体进行重放，要么全部重放，要么全部丢弃。。

作者：CatKang
链接：https://www.zhihu.com/question/357887214/answer/2204930465
来源：知乎
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。



## 2PC

MySQL 使用两阶段提交主要解决 binlog 和 redo log 的数据一致性的问题。

redo log 和 binlog 都可以用于表示事务的提交状态，而两阶段提交就是让这两个状态保持逻辑上的一致。下图为 MySQL 二阶段提交简图：

两阶段提交原理描述:

1. InnoDB redo log 写盘，InnoDB 事务进入 prepare 状态。
2. 如果前面 prepare 成功，binlog 写盘，那么再继续将事务日志持久化到 binlog，如果持久化成功，那么 InnoDB 事务则进入 commit 状态(在 redo log 里面写一个 commit 记录)

备注: 每个事务 binlog 的末尾，会记录一个 XID event，标志着事务是否提交成功，也就是说，recovery 过程中，binlog 最后一个 XID event 之后的内容都应该被 purge。



## redo log 和 binlog 关联

https://www.huaweicloud.com/articles/3d4508df9e2945703fbebff812f4541b.html

redo log 和 binlog 有一个共同的数据字段，叫 XID。崩溃恢复的时候，会按顺序扫描 redo log：

- 如果碰到既有 prepare、又有 commit 的 redo log，就直接提交；
- 如果碰到只有 parepare、而没有 commit 的 redo log，就拿着 XID 去 binlog 找对应的事务。

MySQL 怎么知道 binlog 是完整的?

一个事务的 binlog 是有完整格式的：

- statement 格式的 binlog，最后会有 COMMIT
- row 格式的 binlog，最后会有一个 XID event

在 MySQL 5.6.2 版本以后，还引入了 binlog-checksum 参数，用来验证 binlog 内容的正确性。对于 binlog 日志由于磁盘原因，可能会在日志中间出错的情况，MySQL 可以通过校验 checksum 的结果来发现。所以，MySQL 是有办法验证事务 binlog 的完整性的。



## 数据库Crash后恢复

MySQL 整体来看，其实就有两块：一块是 Server 层，它主要做的是 MySQL 功能层面的事情；还有一块是引擎层，负责存储相关的具体事宜。redo log 是 InnoDB 引擎特有的日志，而 Server 层也有自己的日志，称为 binlog（归档日志）。

binlog 属于逻辑日志，是以二进制的形式记录的是这个语句的原始逻辑，依靠 binlog 是没有 crash-safe 能力的。

sync_binlog 这个参数设置成 1 的时候，表示每次事务的 binlog 都持久化到磁盘。这个参数也建议设置成 1，这样可以保证 MySQL 异常重启之后 binlog 不丢失。

> 这样磁盘压力是不是很大...



当数据库 crash 后，如何恢复未刷盘的数据到内存中？

根据 redo log 和 binlog 的两阶段提交，未持久化的数据分为几种情况：

1. change buffer 写入，redo log 虽然做了 fsync 但未 commit，binlog 未 fsync 到磁盘，这部分数据丢失。
2. change buffer 写入，redo log fsync 未 commit，binlog 已经 fsync 到磁盘，先从 binlog 恢复 redo log，再从 redo log 恢复 change buffer。
3. change buffer 写入，redo log 和 binlog 都已经 fsync，直接从 redo log 里恢复。



## Mysql replication

PS：任何数据冗余，必将引发一致性问题。



Mysql的复制原理大致如下：

1.主库记录binlog日志

在每次准备提交事务完成数据更新前，主库将数据更新的事件记录到二进制日志binlog中。主库上的sync_binlog参数控制binlog日志刷新到磁盘。

2.从库IO线程将主库的binlog日志复制到其本地的中继日志relay log中

由 GRANT REPLICATION 授权的账号, 自动将 SQL 语法 repl 到 Slave 的 DB 执行

从库会启动一个IO线程，IO线程会跟主库建立连接，然后主库会启动一个特殊的二进制转储线程(binlog dump)，二进制转储线程会读取主库上binlog中的事件，它不会一直对事件进行轮询，当它追赶上了主库就会进入睡眠状态，直到主库发送信号量通知其有新事件产生才会被唤醒。

3.从库的SQL线程进行重放

从库的SQL线程从中继日志relay log中读取事件并在从库执行，从而实现从库数据的更新。

简单理解是这三步：

1. 在主库上把数据更改，记录到二进制日志（Binary Log）中。
2. 从库将主库上的日志复制到自己的中继日志（Relay Log）中。
3. 备库读取中继日志中的事件，将其重放到备库数据之上。

### 异步复制

Asynchronous replication

MySQL默认的复制即是异步的，主库在执行完客户端提交的事务后，会立即将结果返给给客户端，并不关心从库是否已经接收并处理，这样就会有一个问题，主库如果crash掉了，此时主上已经提交的事务可能并没有传到从上，如果此时，强行将从提升为主，可能导致新主上的数据不完整（主库事务执行完，从库没同步完）

### 全同步复制

Fully syncronous replication

指当主库执行完一个事务，所有的从库都执行了该事务才返回给客户端。

因为需要等待所有从库执行完该事务才能返回，所以全同步复制的性能必然会收到严重的影响。

### 半同步复制（看）

https://www.cnblogs.com/xinysu/p/6674832.html#autoid-2-1-0

Semi synchronous replication： 主库在应答客户端提交的事务前需要保证至少一个从库接收并写到relay log中。

介于异步复制和全同步复制之间，主库在执行完客户端提交的事务后不是立刻返回给客户端，而是等待至少一个从库接收到并写到relay log（中继日志）中才返回给客户端。相对于异步复制，半同步复制提高了数据的安全性，同时它也造成了一定程度的延迟，这个延迟最少是一个TCP/IP往返的时间。所以，半同步复制最好在低延时的网络中使用。

**如果主库突然宕机，数据还没有同步到从库里，数据就有可能丢失了；**

解决办法有两个机制，1.半同步复制，解决主数据库数据丢失问题 2. 并行复制，用来解决主从同步延时问题

半同步复制（semi-sync半同步复制）： 主库要求写数据的时候，记录binlog日志而且至少有一台从库把binlog日志同步成功了，并且从库返回一个ack，这里只是拉到从库本地的relay日志里，还没有完全同步，这个时候才会认为写操作成功了，否则写操作过来以后只是写好了binlog日志 还没有同步，就会认为是失败的或者说写操作还没有完成； 如果此时主库挂了，宕机了，就认为写操作是不成功的，不成功的话客户端可以感知到，重试一次，重试的时候从库已经切换成主库了，就可以保证写操作的时候还没有同步到从库，宕机导致数据丢失。

配置半同步复制

核心：

* 安装半同步复制插件
* 配置半同步复制

https://www.jianshu.com/p/50884b449901

当主库commit一个事务后，数据库发生宕机，刚好它的binlog还没来得及传送到slave端，这个时候选任何一个slave端都会丢失这个事务，造成数据不一致情况。

为了避免出现主从数据不一致的情况，MySQL引入了半同步复制，添加多了一个从库反馈机制，这个有两种方式设置：

- 主库执行完事务后，同步binlog给从库，从库ack反馈接收到binlog，**主库提交commit**，反馈给客户端，释放会话；
- 主库执行完事务后，**主库提交commit** ，同步binlog给从库，从库ack反馈接收到binlog，反馈给客户端，释放会话；、、

## 应用中实际主从不一致问题(read)

主从同步有时延，这个时延期间读从库，可能读到不一致的数据。

异步复制时，这个问题最明显。

#### 应用如何配置主从分离

这些开发框架都已经实现了，不同框架实现不一样、实现大概就两种方法：

* ORM数据操作接口预留一个参数，显式指定去连接master或者slave
* 框架本身有个小组件，可以自动判断sql实现主从选择，最简单的，`get()==select`走从...

举个栗子：

框架配置中给了`master,slave,redis`等这些地址，你直接对应配置就行。

然后框架的ORM提供了Add()，Get()等操作接口

#### 半同步复制也不能完全解决主从不一致问题。

PS：任何数据冗余，必将引发一致性问题。

假如一主N从，主修改了密码，N个从库，没有完全同步过来数据（semi只保证了一个从库同步了），这时，一个读的连接进来了，读到没更新的数据（概率为 N-1/ N），出现主从数据不一致问题。

> N > 1， 如果N等于1 就是全同步复制了。

如下：

（1）服务发起了一个写请求

（2）服务又发起了一个读请求，此时同步未完成，读到一个不一致的脏数据

（3）数据库主从同步最后才完成

PS：任何数据冗余，必将引发一致性问题。

**问：如何避免这种主从延时导致的不一致？**

**答**：常见的方法有这么几种。

##### 方案一：忽略

任何脱离业务的架构设计都是耍流氓，绝大部分业务，例如：百度搜索，淘宝订单，QQ消息，58帖子都允许短时间不一致。

*画外音：如果业务能接受，最推崇此法。*

如果业务能够接受，别把系统架构搞得太复杂。

##### 方案二：强制读主

如上图：

（1）使用一个高可用主库提供数据库服务

（2）读和写都落到主库上

（3）采用缓存来提升系统读性能

这是很常见的微服务架构，可以避免数据库主从一致性问题。

#####方案三：选择性读主

强制读主过于粗暴，毕竟只有少量写请求，很短时间，可能读取到脏数据。

有没有可能实现，只有这一段时间，可能读到从库脏数据的读请求读主，平时读从呢？

可以利用一个缓存记录必须读主的数据。

如上图，当写请求发生时：

（1）写主库

（2）将哪个库，哪个表，哪个主键三个信息拼装一个key设置到cache里，这条记录的超时时间，设置为“主从同步时延”

画外音：key的格式为“db:table:PK”，假设主从延时为1s，这个key的cache超时时间也为1s。

如上图，当读请求发生时：

这是要读哪个库，哪个表，哪个主键的数据呢，也将这三个信息拼装一个key，到cache里去查询，如果，

（1）**cache里有这个key**，说明1s内刚发生过写请求，数据库主从同步可能还没有完成，此时就应该去主库查询

（2）**cache里没有这个key**，说明最近没有发生过写请求，此时就可以去从库查询

以此，保证读到的一定不是不一致的脏数据。

##### 其他

问题1

用户注册成功后，需要进行登录操作，注册是写 操作，登录是读操作，如果此时从库还没有用户的注册信息，那么用户登录会失败。
 **解决方法：**
 这个问题可以在业务层进行处理，注册成功之后，马上登录的，访问主库；
 这个问题也可以在访问从库失败之后，访问主库进行验证；

问题2

用户修改密码成功后，需要进行登录操作，修改是写 操作，登录是读操作，如果此时从库还没有更新用户的信息，那么用户登录会失败。
 **解决方法：**

缓存方案

此时从库的数据没有更新，如果用户登录会出现失败。可以在业务逻辑层进行处理，当用户密码修改后，在缓存中新增一条Id记录，记录用户的最新信息，并设置过期时间，每次请求的时候，先从缓存读取信息，如果没有在缓存读到，则到从库读取。



**总结**

数据库主库和从库不一致，常见有这么几种优化方案：

（1）业务可以接受，系统不优化

（2）强制读主，高可用主库，用缓存提高读性能

（3）在cache里记录哪些记录发生过写请求，来路由读主还是读从

文字很短，不能解决所有问题，但希望能给大家一些启示。

有更好的方案，欢迎交流。



## GTID模式（看，用于主从）

### 为什么需要GTID

简化复制的使用过程和降低复制集群维护的难度，不再依赖Master的binlog文件名和文件中的位置。
常规：CHANGE MASTER TO MASTER_LOG_FILE=‘Master-bin.000008’, MASTER_LOG_POS=‘216’;
使用GTID：CHANGE MASTER TO AUTO_POSITION=1;

AUTO_POSITION的原理
MySQL Server 记录了所有已经执行了的事务的GTID，包括复制过来的（可用过系统变量Gtid_executed查看）。
Slave记录了所有从master接收过来的事务的GTID（可通过Retrieve_gtid_set查看）。
Slave连接到Master时，会把gtid_executed中的gtid发给master，Master会自动跳过这些事务，只将没有复制的事物发送到Slave去。



https://www.cnblogs.com/kevingrace/p/5569753.html

https://gohalo.me/post/mysql-gtid.html

只有数据库版本是**5.7.6以及之后**的版本才能支持在线开启GTID. 

```sql
-- 5.6
root@localhost : (none) 04:06:07> select @@version;
+------------+
| @@version  |
+------------+
| 5.6.32-log |
+------------+
1 row in set (0.00 sec)

root@localhost : (none) 04:06:22> show variables like "%gtid%"
    -> ;
+---------------------------------+-----------+
| Variable_name                   | Value     |
+---------------------------------+-----------+
| binlog_gtid_simple_recovery     | OFF       |
| enforce_gtid_consistency        | OFF       |
| gtid_executed                   |           |
| gtid_mode                       | OFF       |
| gtid_next                       | AUTOMATIC |
| gtid_owned                      |           |
| gtid_purged                     |           |
| simplified_binlog_gtid_recovery | OFF       |
+---------------------------------+-----------+
8 rows in set (0.00 sec)

--- 5.7
mysql> select @@version;
+-----------+
| @@version |
+-----------+
| 5.7.35    |
+-----------+
1 row in set (0.00 sec)

mysql> select @@GLOBAL.GTID_MODE;
+--------------------+
| @@GLOBAL.GTID_MODE |
+--------------------+
| OFF                |
+--------------------+
1 row in set (0.00 sec)

mysql> set @@GLOBAL.GTID_MODE=ON;
ERROR 1788 (HY000): The value of @@GLOBAL.GTID_MODE can only be changed one step at a time: OFF <-> OFF_PERMISSIVE <-> ON_PERMISSIVE <-> ON. Also note that this value must be stepped up or down simultaneously on all servers. See the Manual for instructions.

mysql> SET @@GLOBAL.GTID_MODE = ON_PERMISSIVE;
Query OK, 0 rows affected (0.00 sec)

mysql> set @@GLOBAL.GTID_MODE=ON;
ERROR 3111 (HY000): SET @@GLOBAL.GTID_MODE = ON is not allowed because ENFORCE_GTID_CONSISTENCY is not ON.

mysql> select @@enforce_gtid_consistency;
+----------------------------+
| @@enforce_gtid_consistency |
+----------------------------+
| OFF                        |
+----------------------------+
1 row in set (0.00 sec)

mysql> set global  enforce_gtid_consistency=ON;
Query OK, 0 rows affected (0.00 sec)

mysql> set @@GLOBAL.GTID_MODE=ON;
Query OK, 0 rows affected (0.00 sec)

mysql> SELECT PLUGIN_NAME, PLUGIN_STATUS FROM INFORMATION_SCHEMA.PLUGINS  WHERE PLUGIN_NAME LIKE '%semi%';
Empty set (0.02 sec)

mysql>  show variables like "%gtid%";
+----------------------------------+-----------+
| Variable_name                    | Value     |
+----------------------------------+-----------+
| binlog_gtid_simple_recovery      | ON        |
| enforce_gtid_consistency         | ON        |
| gtid_executed_compression_period | 1000      |
| gtid_mode                        | ON        |
| gtid_next                        | AUTOMATIC |
| gtid_owned                       |           |
| gtid_purged                      |           |
| session_track_gtids              | OFF       |
+----------------------------------+-----------+
8 rows in set (0.03 sec)

```



GTID 是一个已提交事务的编号，并且是一个全局唯一的编号，在 MySQL 中，GTID 实际上是由 UUID+TID 组成的。其中 UUID 是一个 MySQL 实例的唯一标识；TID 代表了该实例上已经提交的事务数量，并且随着事务提交单调递增。

使用 GTID 功能具体可以归纳为以下两点：

- 可以确认事务最初是在哪个实例上提交的；
- 方便了 Replication 的 Failover 。

第一条显而易见，对于第二点稍微介绍下。

在 GTID 出现之前，在配置主备复制的时候，首先需要确认 event 在那个 binlog 文件，及其偏移量；假设有 A(Master)、B(Slave)、C(Slave) 三个实例，如果主库宕机后，需要通过 `CHANGE MASTER TO MASTER_HOST='xxx', MASTER_LOG_FILE='xxx', MASTER_LOG_POS=nnnn` 指向新库。

这里的难点在于，同一个事务在每台机器上所在的 binlog 文件名和偏移都不同，这也就意味着需要知道新主库的文件以及偏移量，对于有一个主库+多个备库的场景，如果主库宕机，那么需要手动从备库中选出最新的备库，升级为主，然后重新配置备库。

这就导致操作特别复杂，不方便实施，这也就是为什么需要 MHA、MMM 这样的管理工具。

之所以会出现上述的问题，主要是由于各个实例 binlog 中的 event 以及 event 顺序是一致的，但是 binlog+position 是不同的；而通过 GTID 则提供了对于事物的全局一致 ID，主备复制时，只需要知道这个 ID 即可。

另外，利用 GTID，MySQL 会记录那些事物已经执行，从而也就知道接下来要执行那些事务。当有了 GTID 之后，就显得非常的简单；因为同一事务的 GTID 在所有节点上的值一致，那么就可以直接根据 GTID 就可以完成 failover 操作。

写的真好，GTID 

https://gohalo.me/post/mysql-gtid.html

一个 GTID 的生命周期包括：

1. 事务在主库上执行并提交，此时会给事务分配一个 gtid，该值会被写入到 binlog 中；
2. 备库读取 relaylog 中的 gtid，并设置 session 级别的 gtid_next 值，以告诉备库下一个事务必须使用这个值；
3. 备库检查该 gtid 是否已经被使用并记录到他自己的 binlog 中；
4. 由于 gtid_next 非空，备库不会生成一个新的 gtid，而是使用从主库获得的 gtid 。

gohalo.me/post/mysql-gtid.html

https://severalnines.com/resources/database-management-tutorials/mysql-replication-high-availability-tutorial



GTID (Global Transaction IDentifier) 是全局事务标识。它具有全局唯一性，一个事务对应一个GTID。唯一性不仅限于主服务器，GTID在所有的从服务器上也是唯一的。一个GTID在一个服务器上只执行一次，从而避免重复执行导致数据混乱或主从不一致。

在传统的复制里面，当发生故障需要主从切换时，服务器需要找到binlog和pos点，然后将其设定为新的主节点开启复制。相对来说比较麻烦，也容易出错。在MySQL 5.6里面，MySQL会通过内部机制自动匹配GTID断点，不再寻找binlog和pos点。我们只需要知道主节点的ip，端口，以及账号密码就可以自动复制。



在原来基于二进制日志的复制中，从库需要告知主库要从哪个偏移量进行增量同步，如果指定错误会造成数据的遗漏，从而造成数据的不一致。

```sql
--- 确实挺烦的哈 -- 每次要查看master上binlog的position，然后才能在slave上配置--
change master to master_host='172.17.0.3', master_user='slave', master_password='slave', master_port=3306, master_log_file='binlog.000003', master_log_pos= 865, master_connect_retry=30;
```

end

### GTID的组成部分：

GDIT由两部分组成：GTID = source_id:transaction_id。 其中source_id是产生GTID的服务器，即是server_uuid，在第一次启动时生成（sql/mysqld.cc: generate_server_uuid()），并保存到DATADIR/auto.cnf文件里。transaction_id是序列号（sequence number），在每台MySQL服务器上都是从1开始自增长的顺序号，是事务的唯一标识。例如：3E11FA47-71CA-11E1-9E33-C80AA9429562:23 GTID 的集合是一组GTIDs，可以用source_id+transaction_id范围表示，例如：3E11FA47-71CA-11E1-9E33-C80AA9429562:1-5 复杂一点的：如果这组 GTIDs 来自不同的 source_id，各组 source_id 之间用逗号分隔；如果事务序号有多个范围区间，各组范围之间用冒号分隔，例如：3E11FA47-71CA-11E1-9E33-C80AA9429562:23,3E11FA47-71CA-11E1-9E33-C80AA9429562:1-5

### GTID如何产生：

GTID的生成受GTID_NEXT控制。

在主服务器上，GTID_NEXT默认值是AUTOMATIC，即在每次事务提交时自动生成GTID。它从当前已执行的GTID集合（即gtid_executed）中，找一个大于0的未使用的最小值作为下个事务GTID。同时在实际的更新事务记录之前，将GTID写入到binlog（set GTID_NEXT记录）。 在Slave上，从binlog先读取到主库的GTID(即get GTID_NEXT记录)，而后执行的事务采用该GTID。

```bash
$ mysqlbinlog Binlog_file
...
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
...


```

end

### GTID的工作原理：

GTID在所有主从服务器上都是不重复的。所以所有在从服务器上执行的事务都可以在bnlog找到。一旦一个事务提交了，与拥有相同GTID的后续事务都会被忽略。这样可以保证从服务器不会重复执行同一件事务。

当使用GTID时，从服务器不需要保留任何非本地数据。使用数据都可以从replicate data stream。从DBA和开发者的角度看，从服务器无保留file-offset pairs以决定如何处理主从服务器间的数据流。

###GTID的生成和使用由以下几步组成：

- 主服务器更新数据时，会在事务前产生GTID，一同记录到binlog日志中。
- binlog传送到从服务器后，被写入到本地的relay log中。从服务器读取GTID，并将其设定为自己的GTID（GTID_NEXT系统）。
- sql线程从relay log中获取GTID，然后对比从服务器端的binlog是否有记录。
- 如果有记录，说明该GTID的事务已经执行，从服务器会忽略。
- 如果没有记录，从服务器就会从relay log中执行该GTID的事务，并记录到binlog。

### GTID + 半同步复制（实验）

半同步复制的潜在问题

客户端事务在存储引擎层提交后，在得到从库确认的过程中，主库宕机了，此时，可能的情况有两种：

1.事务还没发送到从库上

此时，客户端会收到事务提交失败的信息，客户端会重新提交该事务到新的主上，当宕机的主库重新启动后，以从库的身份重新加入到该主从结构中，会发现，该事务在从库中被提交了两次，一次是之前作为主的时候，一次是被新主同步过来的。

2.事务已经发送到从库上

此时，从库已经收到并应用了该事务，但是客户端仍然会收到事务提交失败的信息，重新提交该事务到新的主上。

**无数据丢失的半同步复制**

针对上述潜在问题，MySQL 5.7引入了一种新的半同步方案：Loss-Less半同步复制。

![Mysql5.7基于GTID的半同步复制1](https://res-static.hc-cdn.cn/fms/img/5e44d72fdb36b64781758574a1c929841603429371565.png)


当然，之前的半同步方案同样支持，MySQL 5.7引入了一个新的参数进行控制-rpl_semi_sync_master_wait_point

rpl_semi_sync_master_wait_point有两种取值：

AFTER_SYNC：这个即新的半同步方案，Waiting Slave dump在Storage Commit之前。
AFTER_COMMIT：老的半同步方案

**无数据丢失的半同步复制的部署**

核心：

* 容器环境，先启动mysql，再安装semi_sync插件，然后手动启动半同步复制

  `[ERROR] unknown variable 'rpl_semi_sync_master_enabled=1` 不安装semi_sync插件时，使用配置参数报错

  使用mysql client修改的配置一定要持久化到`conf`文件中。

* 编译安装mysql时，可以通过`--plugin_name` 启动安装某参加

这两参数可以启动时加载 semi_sync_replica 吗

```bash
--plugin-load=name  Optional semicolon-separated list of plugins to load,
                      where each plugin is identified as name=library, where
                      name is the plugin name and library is the plugin library
                      in plugin_dir.
  --plugin-load-add=name
                      Optional semicolon-separated list of plugins to load,
                      where each plugin is identified as name=library, where
                      name is the plugin name and library is the plugin library
                      in plugin_dir. This option adds to the list specified by
                      --plugin-load in an incremental way. Multiple
                      --plugin-load-add are supported.

```



##### 启动两个容器：

```bash
docker run --name master --rm  -v /tmp/custom/gtid/master.cnf:/etc/mysql/conf.d/master.cnf -e MYSQL_ROOT_PASSWORD=root -d mysql:5.7
docker run --name slave  --rm -v /tmp/custom/gtid/slave.cnf:/etc/mysql/conf.d/slave.cnf -e MYSQL_ROOT_PASSWORD=root -d mysql:5.7

## master
## [ERROR] unknown variable 'rpl_semi_sync_master_enabled=1'
[mysqld]
#For GTID Replication
server_id=18
gtid_mode=on
enforce_gtid_consistency=on
#For Semi Sync config
# 暂时注释掉
#rpl_semi_sync_master_enabled=1
#rpl_semi_sync_master_timeout=3000  # 3 second
#For binlog
log_bin=master-binlog
binlog_format=row
log_slave_updates=1


## slave
[mysqld]
#For GTID Replication
server_id=19
gtid_mode=on
enforce_gtid_consistency=on
#For binlog
log_bin=master-binlog
binlog_format=row
log_slave_updates=1
#For as slave. Please mark it if switch to master.
read_only=1
skip_slave_start=1
relay_log_recovery=1
##For Semi Sync config
#rpl_semi_sync_slave_enabled=1

## 根据配置 gtid已经开启了
mysql> show variables like '%gtid%';
+----------------------------------+-----------+
| Variable_name                    | Value     |
+----------------------------------+-----------+
| binlog_gtid_simple_recovery      | ON        |
| enforce_gtid_consistency         | ON        |
| gtid_executed_compression_period | 1000      |
| gtid_mode                        | ON        |
| gtid_next                        | AUTOMATIC |
| gtid_owned                       |           |
| gtid_purged                      |           |
| session_track_gtids              | OFF       |
+----------------------------------+-----------+
8 rows in set (0.00 sec)

```

end

#####master

```sql
mysql> INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
Query OK, 0 rows affected (0.01 sec)

mysql> SELECT PLUGIN_NAME, PLUGIN_STATUS FROM INFORMATION_SCHEMA.PLUGINS  WHERE PLUGIN_NAME LIKE '%semi%';
+----------------------+---------------+
| PLUGIN_NAME          | PLUGIN_STATUS |
+----------------------+---------------+
| rpl_semi_sync_master | ACTIVE        |
+----------------------+---------------+
1 row in set (0.00 sec)

--- 安装插件后才有rpl_semi_sync_master_enabled  参数

mysql> show variables like '%rpl%';
+-------------------------------------------+------------+
| Variable_name                             | Value      |
+-------------------------------------------+------------+
| rpl_semi_sync_master_enabled              | OFF        |
| rpl_semi_sync_master_timeout              | 10000      |
| rpl_semi_sync_master_trace_level          | 32         |
| rpl_semi_sync_master_wait_for_slave_count | 1          |
| rpl_semi_sync_master_wait_no_slave        | ON         |
| rpl_semi_sync_master_wait_point           | AFTER_SYNC |
| rpl_stop_slave_timeout                    | 31536000   |
+-------------------------------------------+------------+
7 rows in set (0.00 sec)

--- 启动 semi_sync
mysql> SET GLOBAL rpl_semi_sync_master_enabled = 1;
Query OK, 0 rows affected (0.00 sec)

mysql> show variables like '%rpl%';
+-------------------------------------------+------------+
| Variable_name                             | Value      |
+-------------------------------------------+------------+
| rpl_semi_sync_master_enabled              | ON         |
| rpl_semi_sync_master_timeout              | 10000      |
| rpl_semi_sync_master_trace_level          | 32         |
| rpl_semi_sync_master_wait_for_slave_count | 1          |
| rpl_semi_sync_master_wait_no_slave        | ON         |
| rpl_semi_sync_master_wait_point           | AFTER_SYNC |
| rpl_stop_slave_timeout                    | 31536000   |
+-------------------------------------------+------------+
7 rows in set (0.00 sec)

--- 授权复制账号
mysql> grant replication slave on *.* to repl@'%' identified by '123456';

Query OK, 0 rows affected, 1 warning (10.01 sec)


```

end

##### slave

```bash
## 这里安装semi_sync_slave
root@7c0e2d330c41:/# mysql -u root -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.35-log MySQL Community Server (GPL)

Copyright (c) 2000, 2021, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>  install plugin rpl_semi_sync_slave soname 'semisync_slave.so';
Query OK, 0 rows affected (0.02 sec)

mysql> set global rpl_semi_sync_slave_enabled= 1;
Query OK, 0 rows affected (0.00 sec)

mysql>  show global variables like '%semi%';
+-------------------------------------------+------------+
| Variable_name                             | Value      |
+-------------------------------------------+------------+
| rpl_semi_sync_master_enabled              | OFF        |
| rpl_semi_sync_master_timeout              | 10000      |
| rpl_semi_sync_master_trace_level          | 32         |
| rpl_semi_sync_master_wait_for_slave_count | 1          |
| rpl_semi_sync_master_wait_no_slave        | ON         |
| rpl_semi_sync_master_wait_point           | AFTER_SYNC |
| rpl_semi_sync_slave_enabled               | ON         |
| rpl_semi_sync_slave_trace_level           | 32         |
+-------------------------------------------+------------+
8 rows in set (0.00 sec)


--- 重启Slave上的IO线程 和 配置 change master
mysql> STOP SLAVE IO_THREAD;

mysql> CHANGE MASTER TO  MASTER_HOST='172.17.0.2', MASTER_USER='repl', MASTER_PASSWORD='123456', MASTER_PORT=3306, MASTER_CONNECT_RETRY=10,MASTER_AUTO_POSITION=1;
Query OK, 0 rows affected, 2 warnings (0.01 sec)

mysql>  STart SLAVE IO_THREAD;
Query OK, 0 rows affected (0.00 sec)

mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.17.0.2	
              ...
             Slave_IO_Running: Yes
             --- 这里有问题--
            Slave_SQL_Running: No
		...
	##	同步了但没执行
		 Retrieved_Gtid_Set: 82d583be-edcc-11eb-8b9a-0242ac110002:1-7
          Executed_Gtid_Set: cc827051-edcd-11eb-a01c-0242ac110003:1-5
		...
mysql> START SLAVE IO_THREAD;

--- 启动slave
mysql> start slave;
Query OK, 0 rows affected (0.00 sec)

mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.17.0.2
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 10
              Master_Log_File: master-binlog.000003
          Read_Master_Log_Pos: 666
               Relay_Log_File: 7c0e2d330c41-relay-bin.000003
                Relay_Log_Pos: 887
        Relay_Master_Log_File: master-binlog.000003
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
         ## 同步并执行了
        Retrieved_Gtid_Set: 82d583be-edcc-11eb-8b9a-0242ac110002:1-7
         Executed_Gtid_Set: cc827051-edcd-11eb-a01c-0242ac110003:1-5
  
...

## 测试库已创建
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| test_semi_sync     |
+--------------------+
5 rows in set (0.00 sec)

```

end



## 延时备份

一般的主备复制结构存在的问题是，如果主库上有个表被误删了，这个命令很快也会被发给所有从库，进而导致所有从库的数据表也都一起被误删了。

延迟复制的备库是一种特殊的备库，通过 `CHANGE MASTER TO MASTER_DELAY = N`命令，可以指定这个备库持续保持跟主库有 N 秒的延迟。只要在延迟的时间内发现误删，这个命令就还没有在这个延迟复制的备库执行。这时候到这个备库上执行 stop slave，再通过之前介绍的方法，跳过误操作命令，就可以恢复出需要的数据。



## Mutli-source replication

MySQL从5.7版本开始支持多源复制的。MySQL 5.7之前只能实现一主一从、一主多从或者多主多从的复制。如果想实现多主一从的复制，只能使用 MariaDB，但是 MariaDB 又与官方的 MySQL 版本不兼容。MySQL 5.7 开始支持了多主一从的复制方式，也就是多源复制。所以，多源复制至少需要两个Master和一个Slave。

## flush logs

我们知道，在mysql中flush logs操作会生成一个新的binlog文件。

如果在从库执行flush logs 不仅会生成一个新的binlog文件，而且会生成一个新的relaylog文件。

不仅如此，flush logs 还影响slow log和general log,

当删除slow log或者general log,然后执行flush logs，此时会再重新生成一个新的slow log或者general log

## reset master

你好好结合下 数据备份和分布式的思想，master和slave这两个线程不就是实现复制备份的关键吗--

功能说明：删除所有的binglog日志文件，并将日志索引文件清空，重新开始所有新的日志文件。用于第一次进行搭建主从库时，进行主库binlog初始化工作。

> 当数据库要清理binlog文件的时候，可以通过操作系统进行删除，也可以运行reset master进行删除。

RESET MASTER removes all binary log files that are listed in the index file, leaving only a single, empty binary log file with a numeric suffix of .000001, whereas the numbering is not reset by PURGE BINARY LOGS.

  RESET MASTER is not intended to be used while any replication slaves are running. The behavior. of RESET MASTER when used while slaves are running is undefined (and thus unsupported), whereas PURGE BINARY LOGS may be safely used while replication slaves are running.

  大概的意思是RESET MASTER将删除所有的二进制日志，创建一个.000001的空日志。RESET MASTER并不会影响SLAVE服务器上的工作状态，所以盲目的执行这个命令会导致slave找不到master的binlog，造成同步失败。

## reset slave

功能说明：用于删除SLAVE数据库的relaylog日志文件，并重新启用新的relaylog文件；

reset slave 将使slave 忘记主从复制关系的位置信息。该语句将被用于干净的启动, 它删除master.info文件和relay-log.info 文件以及所有的relay log 文件并重新启用一个新的relay log文件。

使用reset slave之前必须使用stop slave 命令将复制进程停止。

这里，需要看下 relay log的作用。

## purge binary log

**reset master** 会删除所有的二进制日志

 **purge binary logs** 是一种基于时间点的删除

```bash
PURGE BINARY LOGS TO 'mysql-bin.010';
PURGE BINARY LOGS BEFORE '2019-04-02 22:46:26';
```

清除二进制日志语句删除指定的日志文件名或日期之前的日志索引文件中列出的所有二进制日志文件。



## 单点

原理：删除后，使用全量的备份 + 删除时间节点前的增量操作，可以达到恢复数据的功能。

如果是，基于ROW格式的binlog日志，可以逆操作delete达到恢复数据的功能？？？



使用BINLOG恢复数据的大致流程如下：

1. 会创建数据库和表，并插入数据。
2. 误删一条数据。
3. 继续插入数据。
4. 误删表。
5. 最后将原来以及之后插入的数据进行恢复。

### 准备数据

准备数据库，表及数据：

```bash
# 创建临时数据库
CREATE DATABASE IF NOT EXISTS test_binlog \
default charset utf8 COLLATE utf8_general_ci; 


# 创建临时表
CREATE TABLE `sync_test` (`id` int(11) \
NOT NULL AUTO_INCREMENT, `name` varchar(255) NOT NULL,  \
PRIMARY KEY (`id`) ) ENGINE=InnoDB DEFAULT CHARSET=utf8;

# 添加数据
insert into sync_test (id, name) values (null, 'a');
insert into sync_test (id, name) values (null, 'b');
insert into sync_test (id, name) values (null, 'c');

# 查看添加的数据
select * from sync_test;
```

### 删除表或者数据

误删操作：

```bash
# 删除 id 小于3 的数据
delete from sync_test where id<3

# 插入几条数据
insert into sync_test (id, name) values (null, 'd');
insert into sync_test (id, name) values (null, 'e');
insert into sync_test (id, name) values (null, 'f');

# 恢复 删除id小于3的数据
mysql> flush logs;
$ mysqlbinlog /var/lib/mysql/mysql-bin.000001 |less
...
BEGIN
/*!*/;
# at 2327
#210714  3:02:55 server id 3  end_log_pos 2447 CRC32 0x9ac9b0de         Query   thread_id=4     exec_time=0     error_code=0
SET TIMESTAMP=1626231775/*!*/;
delete from sync_test where id<3
/*!*/;
# at 2447
#210714  3:02:55 server id 3  end_log_pos 2478 CRC32 0x947c227a         Xid = 34
COMMIT/*!*/;
...
# 定位到删除语句的start position 210714，作为恢复的stop position
mysqlbinlog --no-defaults --start-position "" --stop-position "210714" \
 --database test_binlog  mysql-bin.000034 \
 > /home/mysql_backup/test_binlog_step1.sql
 
# 模拟删除表恢复
# 删除表
DROP TABLE sync_test;
```

### 数据的恢复

在执行数据恢复前，如果操作的是生产环境，会有如下的建议：

- 使用 `flush logs` 命令，替换当前主库中正在使用的 binlog 文件，好处如下：
  - 可将误删操作，定位在一个 BINLOG 文件中，便于之后的数据分析和恢复。
  - 避免操作正在被使用的 BINLOG 文件，防止发生意外情况。
- 数据的恢复不要在生产库中执行，先在临时库恢复，确认无误后，再倒回生产库。防止对数据的二次伤害。

通常来说，恢复主要有两个步骤：

1. 在临时库中，恢复定期执行的全量备份数据。
2. 然后基于全量备份的数据点，通过 BINLOG 来恢复误操作和正常的数据。

**使用 BINLOG 做数据恢复前：**

```bash
# 查看正在使用的 Binlog 文件
show master status\G;
# 显示结果是： mysql-bin.000034

# 执行 flush logs 操作，生成新的 BINLOG
# 隔离恢复操作和原数据
flush logs;

# 查看正在使用的 Binlog 文件
show master status\G;
# 结果是：mysql-bin.000035
```

**确定恢复数据的步骤：**

这里主要是有两条误删的操作，数据行的误删和表的误删。有两种方式进行恢复。

- 方式一：首先恢复到删除表操作之前的位置，然后再单独恢复误删的数据行。
- 方式二：首先恢复到误删数据行的之前的位置，然后跳过误删事件再恢复数据表操作之前的位置。

这里采用方式一的方案进行演示，由于是演示，就不额外找一个临时库进行全量恢复了，直接进行操作。

**查询创建表的事件位置和删除表的事件位置**

```bash
#  根据时间确定位置信息
mysqlbinlog --no-defaults --base64-output=decode-rows -v \
 --start-datetime  "2019-11-22 14:00:00" \
 --database test_binlog  mysql-bin.000034 | less
```

**创建表的开始位置：**

[![创建表的开始位置](https://img2018.cnblogs.com/blog/1861307/201911/1861307-20191124181830281-1061896540.png)](https://img2018.cnblogs.com/blog/1861307/201911/1861307-20191124181830281-1061896540.png)

**删除表的结束位置：**

[![删除表的结束位置](https://img2018.cnblogs.com/blog/1861307/201911/1861307-20191124181841418-936533762.png)](https://img2018.cnblogs.com/blog/1861307/201911/1861307-20191124181841418-936533762.png)

**插入 name='xiaob' 的位置：**

[![插入 name='xiaob' 的位置](https://img2018.cnblogs.com/blog/1861307/201911/1861307-20191124181849655-1021216310.png)](https://img2018.cnblogs.com/blog/1861307/201911/1861307-20191124181849655-1021216310.png)

```bash
# 根据位置导出 SQL 文件
mysqlbinlog --no-defaults --base64-output=decode-rows -v \
 --start-position "2508132" --stop-position "2511004" \
 --database test_binlog  mysql-bin.000034 \
 > /home/mysql_backup/test_binlog_step1.sql
 
 
mysqlbinlog --no-defaults --base64-output=decode-rows -v \
 --start-position "2508813" --stop-position "2509187" \
 --database test_binlog  mysql-bin.000034 \
 > /home/mysql_backup/test_binlog_step2.sql
 

# 使用 mysql 进行恢复
mysql -u cisco -p < /home/mysql_backup/test_binlog_step1.sql
mysql -u cisco -p < /home/mysql_backup/test_binlog_step2.sql
```

> MySQL 5.7 中无论是否打开 GTID 的配置，在每次事务开启时，都首先会出 GTID 的一个事务，用于并行复制。所以在确定导出开始事务位置时，要算上这个事件。
>
> 在使用 --stop-position 导出时，会导出在指定位置的前一个事件，所以这里要推后一个事务。
>
> 对于 DML 的语句，主要结束位置要算上 COMMIT 的位置。



在文章开始时，我们熟悉了操作 BINLOG 的两种方式 CLI 和 mysqlbinlog 工具，接着介绍了其间的区别和使用场景，对于一些大型的 BINLOG 文件，使用 mysqlbinlog 会更加的方便和效率。并对 mysqlbinlog 的一些常见参数进行了介绍。

接着通过使用 mysqlbinlog 实际模拟了数据恢复的过程，并在恢复数据时，提出了一些需要注意的事项，比如 `flush logs` 等。

最后在恢复数据时，要注意 `start-position` 和 `end-position` 的一些小细节，来保证找到合适的位置。

## 

## sql_log_bin

如果想在主库上执行一些操作，但不复制到slave库上，可以通过修改参数sql_log_bin来实现。比如说,这里模拟主从同步复制异常。

 还有一种场景，就是导入或者删除大批量数据时，分别在主库和从库上执行，而不需要通过复制方式来达到一致。

```bash
-- 在从库上执行
mysql> set sql_log_bin=0;#设为0后，在Master数据库上执行的语句都不记录binlog
mysql> delete from t1 where id = 3; # 例如，该删除操作，将不会记录binlog中
mysql> set sql_log_bin=1;
-- 要慎重使用global修饰符（set global sql_log_bin=0)，这样会导致所有在Master数据库上执行的语句都不记录到binlog，这肯定不是你想要的结果
```

end

## slave and slave  io_thread

slave io_thread 负责将master的binlog 复制到 slave实例上的relay日志中。

slave 则执行存储在relay从主库同步过来的sql event

`mysql> start slave`
不带任何参数，表示同时启动I/O 线程和SQL线程。

相当于：

`mysql > start slave sql_thread;` 加上`mysql > start slave io_thread;`

I/O线程从主库读取bin log，并存储到relay log中继日志文件中。

SQL线程读取中继日志，解析后，在从库重放,重放是本机的binlog日志会增加。

原文链接：https://blog.csdn.net/lanyang123456/article/details/84900476

在MySQL复制技术中，涉及到三个线程，分别为binlog Dump线程，IO线程，SQL回放线程。本文针对于这三个线程，简要说明。
**I/O线程**
上图所示，IO线程位于从实例，其作用就是作为一个客户端，建立到master实例上的TCP链接，并且发送认证信息，bingbinlog同步协议信息，然后实时的去同步阻塞的读取master发送的binlog，并且设置有超时时间，为slave_net_timeout，如果在超过设置时间内没有收到master发送的日志信息，则认为主从之间的网络异常，或者master异常，进行网络重试。其生命周期开始于start slave或者start slave io_thread,结束于stop slave或者stop slave io_thread;

Dump线程
如上图所示，Dump线程位于master实例，此线程与普通线程类似，只不过接收到的是客户端发送的binlog dump命令，所以，会发送binlog，而不是其他的数据给客户端。其生命周期开始于slave的start slave或者start slave io_thread,结束于stop slave或者stop slave io_thread。

**SQL线程**
SQL回放线程位于slave实例，主要作用就是读取relay log，并且进行回放操作。其生命周期开始于start slave或者start slave sql_thread,结束于stop slave或者stop slave sql_thread

## 单主N从

流弊啊 Mysql的设计

master ： stop master 、reset master 、 start master 和 change master 太强了

slave: start slave、stop slave、reset slave

show master/slave status\G

太妙了~~

默认master指向localhost的mysql，然后通过`change master to master_host`指向slave要复制的mstart地址。

然后`start slave` 可真是太妙了。

主从切换就是两件事？

1. master 到 slave binlog的同步与还原
2. master故障时，切换slave为master

https://juejin.cn/post/6844903558412763144

1)在主库与从库都安装mysql数据库; 

2) 在主库的my.cnf配置文件中配置server-id 和log-bin; 

3) 在登陆主库后创建认证用户并做授权; 

4) 在从库的my.cnf配置文件中配置server-id;

 5) 登陆从库后，指定master并开启同步开关。

### master 授权

1. 创建一个用于replication的新账号
2. 给新账号授权登陆地址，尽量不要使用`%`，避免风险

```sql
--- Master: 172.17.0.3
mysql> CREATE USER 'slave'@'%' IDENTIFIED BY 'slave';
Query OK, 0 rows affected (0.08 sec)

mysql> GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'slave'@'172.17.0.4' IDENTIFIED BY 'slave';
Query OK, 0 rows affected, 1 warning (0.04 sec)
--- 刷新MySQL的系统权限相关表
mysql> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.06 sec)

mysql> show master status;
+---------------+----------+--------------+------------------+-------------------+
| File          | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+---------------+----------+--------------+------------------+-------------------+
| binlog.000003 |      865 |              |                  |                   |
+---------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)

```

### slave配置

多个slave，只需要重复N次这个即可。

1. change slave master_host
2. start slave

在`START SLAVE`之前，先通过执行`CHANGE REPLICATION FILTER REPLICATE_DO_TABLE=(tbl_name)`

- 让临时库**只同步误操作的表**，利用**并行复制**技术，来加速整个数据恢复过程

这段没看懂

`master_log_pos` 这个参数还要知道--

```sql
--- slave : 172.17.0.4

mysql> change master to master_host='172.17.0.3', master_user='slave', master_password='slave', master_port=3306, master_log_file='binlog.000003', master_log_pos= 865, master_connect_retry=30;
ERROR 1201 (HY000): Could not initialize master info structure; more error messages can be found in the MySQL error log
--- 网上百度的先reset slave
mysql> stop slave;
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> reset slave;
Query OK, 0 rows affected (0.00 sec)

mysql> show slave status\G
Empty set (0.00 sec)

--- 再次运行还是报错
mysql> change master to master_host='172.17.0.3', master_user='slave', master_password='slave', master_port=3306, master_log_file='binlog.000003', master_log_pos= 865, master_connect_retry=30;
ERROR 1201 (HY000): Could not initialize master info structure; more error messages can be found in the MySQL error log
--- 查看error日志
--- 这里显示是stderr，因为是容器环境所以需要使用docker log查看
mysql> show variables like 'log_error';
+---------------+--------+
| Variable_name | Value  |
+---------------+--------+
| log_error     | stderr |
+---------------+--------+
1 row in set (0.01 sec)

--- error ：少了目录文件--
2021-07-23T15:11:58.392654Z 2 [ERROR] Path '/tmp/relay/' is a directory name, please specify a file name for --relay-log option.
2021-07-23T15:14:01.591944Z 2 [ERROR] Path '/tmp/relay/' is a directory name, please specify a file name for --relay-log option.
是因为配置写错了
[mysqld]
log-bin   = /tmp/binlog
-- 多了一个 /
relay_log = /tmp/relay/
server-id = 4
mysql> show variables like '%relay%';
+---------------------------+-------------------+
| Variable_name             | Value             |
+---------------------------+-------------------+
| max_relay_log_size        | 0                 |
| relay_log                 | /tmp/relay/       |
| relay_log_basename        | /tmp/relay/       |
| relay_log_index           | /tmp/relay/.index |
| relay_log_info_file       | relay-log.info    |
| relay_log_info_repository | FILE              |
| relay_log_purge           | ON                |
| relay_log_recovery        | OFF               |
| relay_log_space_limit     | 0                 |
| sync_relay_log            | 10000             |
| sync_relay_log_info       | 10000             |
+---------------------------+-------------------+
11 rows in set (0.00 sec)

----

--- 查看主从同步状态

mysql> change master to master_host='172.17.0.3', master_user='slave', master_password='slave', master_port=3306, master_log_file='binlog.000003', master_log_pos= 865, master_connect_retry=30;
Query OK, 0 rows affected, 2 warnings (0.03 sec)

mysql> show slave status \G
*************************** 1. row ***************************
               Slave_IO_State:
                  Master_Host: 172.17.0.3
                  Master_User: slave
                  Master_Port: 3306
                Connect_Retry: 30
              Master_Log_File: binlog.000003
          Read_Master_Log_Pos: 865
               Relay_Log_File: relay.000001
                Relay_Log_Pos: 4
        Relay_Master_Log_File: binlog.000003
        -- 正常情况下，SlaveIORunning 和 SlaveSQLRunning 都是No
             Slave_IO_Running: No
            Slave_SQL_Running: No
         ....
         --- 用于排错
         	  Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
         ....

--- 开启 slave
mysql> start slave;
Query OK, 0 rows affected (0.00 sec)

mysql> show slave status \G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.17.0.3
                  Master_User: slave
                  Master_Port: 3306
                Connect_Retry: 30
              Master_Log_File: binlog.000003
          Read_Master_Log_Pos: 865
               Relay_Log_File: relay.000002
                Relay_Log_Pos: 317
        Relay_Master_Log_File: binlog.000003
        --- 变成yes..
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
...
```

end

### 测试主从复制

```sql
--- master 上创建一个新库
mysql> create database test_slave_io;
Query OK, 1 row affected (0.00 sec)

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| test_slave_io      |
+--------------------+
5 rows in set (0.02 sec)

--- 查看Mysql binlog position
mysql> show master status\G
*************************** 1. row ***************************
             File: binlog.000003
         Position: 1051
     Binlog_Do_DB:
 Binlog_Ignore_DB:
Executed_Gtid_Set:
1 row in set (0.00 sec)



--- slave 上自动创建了 test_slave_io
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| test_slave_io      |
+--------------------+
5 rows in set (0.00 sec)

mysql> show slave status \G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.17.0.3
                  Master_User: slave
                  Master_Port: 3306
                Connect_Retry: 30
              Master_Log_File: binlog.000003
              --- binlog position 和 master上一致
               Read_Master_Log_Pos: 1051
               Relay_Log_File: relay.000002
               ....
```

end

### 主从切换

#### master宕机

太简单了这样

1>  在备机上执行STOP SLAVE 和RESET MASTER

> reset master 秒啊，让slave mysql实例的master执行localhost

2>  查看show slave status \G;

3>  然后修改应用的连接地址。

一般大部分切换为直接宕机主机已经没法提供服务、

#### master没宕机

这有点讲究了，核心是比master宕机的情况，多了两步

* 先确认slave <--> master 事件是不是同步完成了
* master和slave要不要配置对调，将slave转成master，而master当slave

1）从服务器检查`SHOW PROCESSLIST`语句的输出，直到你看到`Has read all relaylogwaiting for the slave I/O thread to update it`

```sql
mysql> show processlist\G
...
     Id: 3
   User: system user
   Host:
     db: NULL
Command: Connect
   Time: 3559
  State: Waiting for master to send event
   Info: NULL
*************************** 3. row ***************************
     Id: 4
   User: system user
   Host:
     db: NULL
Command: Connect
   Time: 3256
  State: Slave has read all relay log; waiting for more updates
   Info: NULL
...

```

2）确保从服务器已经处理了日志中的所有语句。 mysql> STOP SLAVE IO_THREAD

当从服务器都执行完这些，它们可以被重新配置为一个新的设置。

```sql
mysql> STOP SLAVE IO_THREAD;
Query OK, 0 rows affected, 1 warning (0.00 sec)


mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State:
                  Master_Host: 172.17.0.3
                  Master_User: slave
                  Master_Port: 3306
                Connect_Retry: 30
              Master_Log_File: binlog.000003
          Read_Master_Log_Pos: 1051
               Relay_Log_File: relay.000002
                Relay_Log_Pos: 503
        Relay_Master_Log_File: binlog.000003
        --- 已经变成 No
             Slave_IO_Running: No
            Slave_SQL_Running: Yes

```



3）在被提升为主服务器的从服务器上，发出` STOP SLAVE`和`RESET MASTER`和`RESET SLAVE`操作。

因为要从转主，所以要重新配置`mysql master` 和 `mysql slave`

4）然后重启mysql服务。

5）在备用服务器（新的主服务器）创建用于复制的用户并授权。

`grant replication slave `

6) 在主服务器上RESET MASTER。

`CHANGE MASTER ...`

7)查看状态 show slave status \G;

Show master status \G;

8）修改应用的连接地址到新的主库

### 延迟复制备库

1. 上面`Master-Slave`的方案利用了并行复制来加速数据恢复的过程，但恢复时间不可控
   - 如果一个库特别大，或者误操作的时间距离上一个全量备份的时间较长（一周一备）
2. 针对核心业务，**不允许太长的恢复时间**，可以搭建**延迟复制的备库**（MySQL 5.6引入）
3. 延迟复制的备库是一种特殊的备库
   - 通过`CHANGE MASTER TO MASTER_DELAY=N`命令来指定备库持续与主库有N秒的延迟
   - 假设N=3600，如果能在一小时内发现误删除命令，这个误删除的命令尚未在延迟复制的备库上执行
   - 这时在这个备库上执行`STOP SLAVE`，再通过前面的方法，跳过误删除命令，就可以恢复数据
   - 这样，可以得到一个恢复时间可控（最多1小时）的备库

## 多主N从

常见方案：https://cloud.tencent.com/developer/article/1120513

https://developer.aliyun.com/article/776634

### MMM

数据冗余会引发数据的一致性问题，因为数据的同步有一个时间差，并发的写入可能导致数据同步失败，引起数据丢失。

多主数据同步--

https://tech.meituan.com/2017/06/29/database-availability-architecture.html

https://colobu.com/2019/12/02/How-to-Setup-MySQL-Master-Master-Replication/

MMM（Master-Master replication manager for MySQL）

MMM的架构如下。

![MMM架构](https://awps-assets.meituan.net/mit-x/blog-images-bundle-2017/a281fec7.png)

MMM架构

如上所示，整个MySQL集群提供1个写VIP（Virtual IP）和N（N>=1）个读VIP提供对外服务。每个MySQL节点均部署有一个Agent（mmm-agent），mmm-agent和mmm-manager保持通信状态，定期向mmm-manager上报当前MySQL节点的存活情况（这里称之为心跳）。当mmm-manager连续多次无法收到mmm-agent的心跳消息时，会进行切换操作。

mmm-manager分两种情况处理出现的异常。

1. 出现异常的是从节点
   - mmm-manager会尝试摘掉该从节点的读VIP，并将该读VIP漂移到其它存活的节点上，通过这种方式实现从库的高可用。
2. 出现异常的是主节点
   - 如果当时节点还没完全挂，只是响应超时。则尝试将Dead Master加上全局锁（flush tables with read lock）。
   - 在从节点中选择一个候选主节点作为新的主节点，进行数据补齐。
   - 数据补齐之后，摘掉Dead Master的写VIP，并尝试加到新的主节点上。
   - 将其它存活的节点进行数据补齐，并重新挂载在新的主节点上。

mmm-manager检测到master1发生了故障，对数据进行补齐之后，将写VIP漂移到了master2上，应用写操作在新的节点上继续进行。

然而，MMM架构存在如下问题：

- VIP的数量过多，管理困难（曾经有一个集群是1主6从，共计7个VIP）。某些情况下会导致集群大部分VIP同时丢失，很难分清节点上之前使用的是哪个VIP。
- mmm-agent过度敏感，容易导致VIP丢失。同时mmm-agent自身由于没有高可用，一旦挂掉，会造成mmm-manager误判，误认为MySQL节点异常。
- mmm-manager存在单点，一旦由于某些原因挂掉，整个集群就失去了高可用。
- VIP需要使用ARP协议，跨网段、跨机房的高可用基本无法实现，保障能力有限。

### MHA

https://www.infoq.cn/article/jty9bmxp9utrco9uhzm3

MHA 由两个模块组成：Manager 和 Node。

Manager 部署在独立的机器上，负责检查 MySQL 复制状态、主库状态以及执行切换操作。Node 运行在每台 MySQL 机器上，主要负责保存和复制 master binlog、识别主库宕机时各 Slave 差异的中继日志并将差异的事务应用到其他的 Slave，同时还负责清除 Slave 上的 relay_log。

MHA 虽然已经比较成熟，但也存在一些的缺点：

- 使用配置文件管理主备关系、不能重复切换
- 实例增减需要重启 Manager
- Manager 是单点，虽然有 standby 的节点，但不能自动切换

MySQL Master High Availability, MHA只负责MySQL主库的高可用。主库发生故障时，MHA会选择一个数据最接近原主库的候选主节点（这里只有一个从节点，所以该从节点即为候选主节点）作为新的主节点，并补齐和之前Dead Master 差异的Binlog。数据补齐之后，即将写VIP漂移到新主库上。

MHA要增加主节点的故障检测，从而避免脑裂、比如DB服务器的上联交换机出现了抖动，导致主库无法访问，被管理节点判定为故障，触发MHA切换，VIP被漂到了新主库上。随后交换机恢复，主库可被访问，但由于VIP并没有从主库上摘除，因此2台机器同时拥有VIP，会产生脑裂。需要对MHA Manager加入了向同机架上其他物理机的探测，通过对比更多的信息来判断是网络故障还是单机故障。

### PXC

不支持XA事务，那没救了

https://juejin.cn/post/6844904039063224327

Percona XtraDB Cluster (简称 PXC) 是 Percona 公司开源的实现 MySQL 高可用的解决方案。它将 Percona Server 和 Percona XtraBackup 与 Galera 库集成，以实现多主同步复制。和 MySQL 传统的异步复制相比，它能够保证数据的强一致性，在任何时刻集群中任意节点上的数据状态都是完全一致的，并且整个架构实现了去中心化，所有节点都是对等的，即允许你在任意节点上进行写入和读取，集群会把数据状态同步至其他所有节点。但目前 PXC 集群只支持 InnoDB 存储引擎，并具有以下限制：

- 添加新节点时，必须从现有节点之一复制完整数据集。如果是 100GB，则复制 100GB。为了减少网络开销，建议在搭建集群前使用备份的方式将所有节点初始的数据状态调整至一致。
- 不支持 LOCK TABLES ，在多主设置的情况下也不支持 UNLOCK TABLES。
- 不支持锁定功能，如 GET_LOCK()，RELEASE_LOCK() 等。
- 由于可能在提交时回滚，因此也不支持 XA 事务 (分布式事务) 。
- 所有表必须具有主键。
- 由于节点是对等的，所以整个集群的写吞吐量受限于性能最差的节点，如果一个节点变慢，则整个群集都会变慢。因此应该保证所有节点的硬件配置一致，并避免单个节点超负载运行。
- 允许的最大事务大小由 wsrep_max_ws_rows 和 wsrep_max_ws_size 参数共同定义，因此超大型事务会被拆分为一系列小型事务，如加载大数据集 LOAD DATA INFILE。
- 由于在集群级别采用乐观锁进行并发控制，所以事务在 COMMIT 阶段仍然有被中止的可能。如两个事务在不同的集群节点上提交对相同的行的写入，此时只有其中一个可以成功提交，另一个将被中止。





### MGR

有意思，有意思啊。Mysql Group Replication 只需要更改master变量的指向就行了。

不使用分布式事务的化，MGR可以实现多主的高可用。

Paxos协议是用来解决分布式事务分歧的，以Mgr的多个节点为例，如果多个节点只有一个写节点(主)，是不存在分布式事务的；如果多个节点同时写入，就可能会有分歧，在同一时刻A节点要修改账户增加100块，B节点同时要修改账户增加200块，这个账户就会有疑惑，到底是听A的还是听B的，还是两个同时听还是干脆两个都不听，这里有矛盾了，因此Paxos分布式协议就可以完美的解决这个分歧。



GR插件负责执行分布式内容，侦测和处理冲突，恢复分布式集群，推送事务给其它的组成员，接收其它成员的事务以及决定事务最终的结果。

GCS API将通信系统的实现进行抽象化，并管理这个接口。通信引擎是基于Paxos开发的，是实现跨服务器的组件。

> Paxos是一个共识（consensus）算法,不是一致性(consistency)算法。

集群中每个DB都有MGR层，MGR层功能也可简单理解为由Paxos模块和冲突检测Certify模块实现。Paxos模块是基于Paxos算法确保所有节点收到相同广播消息，transaction message就是广播消息的内容结构；冲突检测Certify模块进行冲突检测确保数据最终一致性，其中certification info是冲突检测中内存结构



总结：MGR 很好，有多个master，但是application怎么选择呢？？？ 事务怎么处理--

 mysql官方基于组复制概念并充分参考MariaDB Galera Cluster和Percona XtraDB Cluster结合而来的新的高可用集群架构。只支持5.7以上版本

mgr 优点：

高一致性，**基于原生复制及paxos协议的组复制技术.**

> paxos 流弊

高容错性，有自动检测机制，当出现宕机后,会自动剔除问题节点,其他节点可以正常使用(类似zk集群),当不同节点产生资源争用冲突时,会按照先到先得处理,并且内置了自动化脑裂防护机制.

高扩展性，可随时在线新增和移除节点,会自动同步所有节点上状态，直到新节点和其他节点保持一致，自动维护新的组信息.

高灵活性，直接插件形式安装(5.7.17后自带.so插件),有单主模式和多主模式，单主模式下，只有主库可以读写,其他从库会加上super_read_only状态,只能读取不可写入,出现故障会自动选主.



缺点：目前不太稳定，太新有BUG（如新加入集群宕机,并行复制有不一致bug）、管理不方便（需配合mysql-shell）



注意：多主模式下最好有三台以上的节点,单主模式则视实际情况而定,不过同个Group最多节点数为9.

服务器配置尽量保持一致,因为和PXC一样,也会有"木桶短板效应".

需要特别注意,mysql数据库的服务端口号和MGR的服务端口不是一回事,需要区分开来.

不能有级联外键？？？

#### docker部署mgr

##### my.cnf配置文件

自己Google吧

https://www.modb.pro/db/59048

```bash
# 这个很重要mgr实例间指定使用IP，而非默认的主机名
report_host=ip

# 开启GTID,必须开启
gtid_mode=on

# 强制GTID的一致性
enforce-gtid-consistency=on

# binlog格式,MGR要求必须是ROW,不过就算不是MGR,也最好用row
binlog_format=row

# server-id必须是唯一的
server-id=2

# MGR使用乐观锁,所以官网建议隔离级别是RC,减少锁粒度
transaction_isolation=READ-COMMITTED
...
# 主要是用来区分整个内网里边的各个不同的GROUP,而且也是这个group内的GTID值的UUID
loose-group_replication_group_name = 'cc5e2627-2285-451f-86e6-0be21581539f'

#是否随服务器启动而自动启动组复制,不建议直接启动,怕故障恢复时有扰乱数据准确性的特殊情况
loose-group_replication_start_on_boot = OFF

# 本地MGR的IP地址和端口，host:port,是MGR的端口,不是数据库的端口
loose-group_replication_local_address = 'slave-1:33066'

# 需要接受本MGR实例控制的服务器IP地址和端口,是MGR的端口,不是数据库的端口
loose-group_replication_group_seeds = 'master:33066,slave-1:33066,slave-2:33066'

# 开启引导模式,添加组成员，用于第一次搭建MGR或重建MGR的时候使用,只需要在集群内的其中一台开启,
#loose-group_replication_bootstrap_group = OFF

# 是否启动单主模式，如果启动，则本实例是主库，提供读写，其他实例仅提供读,如果为off就是多主模式了
#loose-group_replication_single_primary_mode = off

# 多主模式下,强制检查每一个实例是否允许该操作,如果不是多主,可以关闭
#loose-group_replication_enforce_update_everywhere_checks = on
loose-group_replication_ip_whitelist="master_ip,slave_ip...";
```

end

启动三个mysql容器

```bash
docker run --name master --rm -v /tmp/custom/master.cnf:/etc/mysql/conf.d/master.cnf -e MYSQL_ROOT_PASSWORD=root -d mysql:5.7

docker run --name s1 --rm -v /tmp/custom/slave-1.cnf:/etc/mysql/conf.d/slave.cnf -e MYSQL_ROOT_PASSWORD=root -d mysql:5.7

docker run --name s2 --rm -v /tmp/custom/slave-2.cnf:/etc/mysql/conf.d/slave.cnf -e MYSQL_ROOT_PASSWORD=root -d mysql:5.7

```

end

##### mgr配置

核心：

* 配置复制账号
* 安装mgr插件
* 配置mgr集群...

```sql
--- 集群节点都要配置一个用于replication的账号 因为都是master
SET SQL_LOG_BIN=0;
CREATE USER rpl_user@'%' IDENTIFIED BY '123456';
GRANT REPLICATION SLAVE ON *.* TO rpl_user@'%';
FLUSH PRIVILEGES;
SET SQL_LOG_BIN=1;

CHANGE MASTER TO MASTER_USER='rpl_user', MASTER_PASSWORD='123456'  FOR CHANNEL 
'group_replication_recovery';

--- 安装mgr插件，mysql 5.7 之后已经自带了安装就行
INSTALL PLUGIN group_replication SONAME 'group_replication.so';

--- master 配置mgr
SET GLOBAL group_replication_bootstrap_group=ON;
START GROUP_REPLICATION;
--- 开启GR后可以看到集群member，如果报错了就看error日志。
--- show variables like 'log_error';
--- 因为是docker 跑的实例直接docker logs就行了因为它把error log重定向到stderr了
mysql> select * from `performance_schema`.replication_group_members;
+---------------------------+--------------------------------------+--------------+-------------+--------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST  | MEMBER_PORT | MEMBER_STATE |
+---------------------------+--------------------------------------+--------------+-------------+--------------+
| group_replication_applier | 64619c8a-ec6e-11eb-bf45-0242ac110003 | c4e8da83c0bb |        3306 | ONLINE       |
+---------------------------+--------------------------------------+--------------+-------------+--------------+
1 row in set (0.00 sec)
-- SET GLOBAL group_replication_ip_whitelist='172.17.0.3,172.17.0.4,172.17.0.5';
SET GLOBAL group_replication_bootstrap_group=OFF;

--- slave 配置mgr，所有slave都一样
SET GLOBAL group_replication_allow_local_lower_version_join=ON;
set global group_replication_allow_local_disjoint_gtids_join=ON;
START GROUP_REPLICATION;
--- 成功后验证，失败了查日志
 select * from `performance_schema`.replication_group_members;
```

end

##### 遇到问题

###### 主机名无法解析

错误信息： `Member with address 9c7a56694871:3306 is reachable again.`

通过`SELECT * FROM performance_schema.replication_group_members;`查询，发现MEMBER_STATE一直是recoving

解决办法

**方法一 配置hosts**

```
172.17.84.71 mysql001
172.17.84.72 msyql002
172.17.84.73 mysql003
```

**方法二**

或者在配置文件my.cnf使用report_host=ip，显示指定使用IP，而非默认的主机名

###### slave 开启gr失败

```bash
2021-07-24T14:13:57.821636Z 0 [ERROR] Plugin group_replication reported: 'This member has more executed transactions than those present in the group. Local transactions: b5eb70e2-ec87-11eb-83b1-0242ac110004:1-5 > Group transactions: 64619c8a-ec6e-11eb-bf45-0242ac110003:1-5,
cc5e2627-2285-451f-86e6-0be21581539f:1'
2021-07-24T14:13:57.821665Z 0 [ERROR] Plugin group_replication reported: 'The member contains transactions not present in the group. The member will now exit the group.'
2021-07-24T14:13:57.821670Z 0 [Note] Plugin group_replication reported: 'To force this member into the group you can use the group_replication_allow_local_disjoint_gtids_join option

```

解决: 其实错误日志中已经给了错误解决方法了....
`set global group_replication_allow_local_disjoint_gtids_join=ON;`（但是实际上这种方法治标不治本）
建议还是在搭建group_replication的时候，在start group_replication之前，reset master，重置所有binary log，这样就不会出现各个实例之间的日志超前影响；但是这里要考虑是否影响到旧主从。

###### failed on flush_net()

如果mysql的错误日志中出现failed on flush_net()这类的错误时，即说明主库的mysqld在向客户端发送网络包时失败导致的。在主从这种复制场景下则说明是复制过程中master向slave推送binlog写网络数据包失败。

##### 验证

```bash
## master 上创建库
mysql> create database test_group_replication;
Query OK, 1 row affected (0.01 sec)

## slave上
mysql> show databases;
+------------------------+
| Database               |
+------------------------+
| information_schema     |
| mysql                  |
| performance_schema     |
| sys                    |
| test                   |
| test_group_replication |
+------------------------+
6 rows in set (0.00 sec)

mysql> show slave status;
Empty set (0.00 sec)

## master
mysql> show master status\G
*************************** 1. row ***************************
             File: mysql-bin.000003
         Position: 3033273
     Binlog_Do_DB:
 Binlog_Ignore_DB:
Executed_Gtid_Set: 64619c8a-ec6e-11eb-bf45-0242ac110003:1-5,
b84178a3-ec87-11eb-9a10-0242ac110005:1-6,
cc5e2627-2285-451f-86e6-0be21581539f:1-15
1 row in set (0.00 sec)


```



故障转移

```bash
##日志中记录了哪个是master
2021-07-25T05:02:59.659795Z 51 [Note] Slave I/O thread for channel 'group_replication_recovery': connected to master 'rpl_user@c4e8da83c0bb:3306',replication started in log 'FIRST' at position 4


## 可以看到集群全部成员
mysql> select * from performance_schema.replication_group_members\G
*************************** 1. row ***************************
CHANNEL_NAME: group_replication_applier
   MEMBER_ID: 64619c8a-ec6e-11eb-bf45-0242ac110003
 MEMBER_HOST: c4e8da83c0bb
 MEMBER_PORT: 3306
MEMBER_STATE: ONLINE
*************************** 2. row ***************************
CHANNEL_NAME: group_replication_applier
   MEMBER_ID: b5eb70e2-ec87-11eb-83b1-0242ac110004
 MEMBER_HOST: 9c7a56694871
 MEMBER_PORT: 3306
MEMBER_STATE: ONLINE
*************************** 3. row ***************************
CHANNEL_NAME: group_replication_applier
   MEMBER_ID: b84178a3-ec87-11eb-9a10-0242ac110005
 MEMBER_HOST: 07cbd7fade28
 MEMBER_PORT: 3306
MEMBER_STATE: ONLINE
3 rows in set (0.00 sec)


## 验证故障转移
## 正好由于网络问题 master 失联了。。。
Plugin group_replication reported: 'This server is not able to reach a majority of members in the group. This server will now block all updates. The server will remain blocked until contact with the majority is restored. It is possible to use group_replication_force_members to force a new group membership.'
##  master error
mysql> select * from performance_schema.replication_group_members;
+---------------------------+--------------------------------------+--------------+-------------+--------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST  | MEMBER_PORT | MEMBER_STATE |
+---------------------------+--------------------------------------+--------------+-------------+--------------+
| group_replication_applier | 64619c8a-ec6e-11eb-bf45-0242ac110003 | c4e8da83c0bb |        3306 | ERROR        |
+---------------------------+--------------------------------------+--------------+-------------+--------------+
1 row in set (0.00 sec)

## 这是两个slave的日志。。。
1 row in set (0.00 sec)
 ⚡ root@ning  ~  docker logs 07 --tail 10
2021-07-24T14:43:30.075875Z 0 [Warning] Plugin group_replication reported: 'Member with address c4e8da83c0bb:3306 has become unreachable.'
2021-07-24T14:43:30.075884Z 0 [Warning] Plugin group_replication reported: 'Member with address 9c7a56694871:3306 has become unreachable.'
2021-07-24T14:43:30.075890Z 0 [ERROR] Plugin group_replication reported: 'This server is not able to reach a majority of members in the group. This server will now block all updates. The server will remain blocked until contact with the majority is restored. It is possible to use group_replication_force_members to force a new group membership.'
2021-07-24T14:43:30.772606Z 0 [Warning] Plugin group_replication reported: 'Member with address 9c7a56694871:3306 is reachable again.'
2021-07-24T14:43:30.772626Z 0 [Warning] Plugin group_replication reported: 'The member has resumed contact with a majority of the members in the group. Regular operation is restored and transactions are unblocked.'
2021-07-24T14:43:31.776346Z 0 [Warning] Plugin group_replication reported: 'Members removed from the group: c4e8da83c0bb:3306'
2021-07-24T14:43:31.776360Z 0 [Note] Plugin group_replication reported: 'Primary server with address c4e8da83c0bb:3306 left the group. Electing new Primary.'
2021-07-24T14:43:31.776449Z 0 [Note] Plugin group_replication reported: 'A new primary with address 9c7a56694871:3306 was elected, enabling conflict detection until the new primary applies all relay logs.'
2021-07-24T14:43:31.776498Z 32 [Note] Plugin group_replication reported: 'This server is working as secondary member with primary member address 9c7a56694871:3306.'
2021-07-24T14:43:31.776522Z 0 [Note] Plugin group_replication reported: 'Group membership changed to 9c7a56694871:3306, 07cbd7fade28:3306 on view 16271373283458219:4.'
 ⚡ root@ning  ~  docker logs 9c --tail 10
2021-07-24T14:43:30.073436Z 0 [Warning] Plugin group_replication reported: 'Member with address 07cbd7fade28:3306 has become unreachable.'
2021-07-24T14:43:30.073440Z 0 [ERROR] Plugin group_replication reported: 'This server is not able to reach a majority of members in the group. This server will now block all updates. The server will remain blocked until contact with the majority is restored. It is possible to use group_replication_force_members to force a new group membership.'
2021-07-24T14:43:30.772644Z 0 [Warning] Plugin group_replication reported: 'Member with address 07cbd7fade28:3306 is reachable again.'
2021-07-24T14:43:30.772661Z 0 [Warning] Plugin group_replication reported: 'The member has resumed contact with a majority of the members in the group. Regular operation is restored and transactions are unblocked.'
2021-07-24T14:43:30.879186Z 0 [Note] Plugin group_replication reported: '[GCS] Removing members that have failed while processing new view.'
2021-07-24T14:43:31.775976Z 0 [Warning] Plugin group_replication reported: 'Members removed from the group: c4e8da83c0bb:3306'
2021-07-24T14:43:31.776000Z 0 [Note] Plugin group_replication reported: 'Primary server with address c4e8da83c0bb:3306 left the group. Electing new Primary.'
2021-07-24T14:43:31.776076Z 0 [Note] Plugin group_replication reported: 'A new primary with address 9c7a56694871:3306 was elected, enabling conflict detection until the new primary applies all relay logs.'
2021-07-24T14:43:31.776163Z 63 [Note] Plugin group_replication reported: 'This server is working as primary member.'
2021-07-24T14:43:31.776187Z 0 [Note] Plugin group_replication reported: 'Group membership changed to 9c7a56694871:3306, 07cbd7fade28:3306 on view 16271373283458219:4.'
 ⚡ root@ning  ~ 

```

end



#### MGR的限制与要求

https://www.jianshu.com/p/b9831f15e10c

http://blog.itpub.net/28413242/viewspace-2652823/

docker集群部署mgr

https://www.modb.pro/db/59048

#### MGR实践

https://www.jianshu.com/p/b9831f15e10c

#### MGR故障恢复

https://blog.csdn.net/William0318/article/details/105597003

#### Mysql Router

MySQL Router是轻量级的中间件，可在您的应用程序与任何后端MySQL Server之间提供透明的路由。它可用于多种使用案例，例如通过有效地将数据库流量路由到适当的后端MySQL服务器来提供高可用性和可伸缩性。



## 异地多活

**异地**指地理位置上的不同，**多活**指不同地理位置上的系统都能够提供业务服务。

判断标准：

1. 正常情况下，用户无论访问哪一个地点的业务系统，都能够得到正确的业务服务。
2. 某地异常时，用户访问其他地方正常的业务系统，能够得到正确的业务服务。

异地多活的代价：

1. 系统复杂度会有质的变化。
2. 成本大大增加。

### 两地三中心

随着互联网业务快速发展，多IDC的业务支撑能力和要求也逐步提升，行业内的“两地三中心”方案较为流行。其中**两地**是指同城、异地；**三中心**是指生产中心、同城容灾中心、异地容灾中心。

两地三中心方案中，基于设定的短期目标可以明确同城双活和异地容灾的方案组合。设计重点为同城双活，即在同城的数据中心之间，一般通过高速光纤相连，在网络带宽有保障的前提下，网络延迟一般在可接受范围内，两个机房之间可以认为在同一个局域网内。



### **1. 同城异区**

部署在同一个城市不同区的机房，用专用网络连接。

同城异区两个机房距离一般就是几十千米，网络传输速度几乎和同一个机房相同，降低了系统复杂度、成本。

这个模式无法解决极端的灾难情况，例如某个城市的地震、水灾，此方式是用来解决一些常规故障的，例如机房的火灾、停电、空调故障。

### **2. 跨城异地**

部署在不同城市的多个机房，距离要远一些，例如北京和广州。

此模式就是用来处理极端灾难情况的，例如城市的地震、相邻城市的大停电。

**跨城异地主要问题就是网络传输延迟**，例如北京到广州，正常情况下的RTT（Round-Trip Time 往返时延）是50毫秒，当遇到网络波动等情况，会升到500毫秒甚至1秒，而且会有丢包问题。

物理距离必然导致数据不一致，这就得从“数据”特性来解决，如果是**强一致性要求**的数据（如存款余额），就无法做异地多活。

举例，存款余额支持跨城异地多活，部署在北京和广州，当光缆被挖断后，会发生：

- 用户A余额有10000，北京和广州是一致的。
- A在广州机房给B转5000，广州余额为5000，由于网络不通，北京此时的余额为10000。
- A在北京发现余额还是10000，给C转10000，由于网络不通，广州的余额为5000。
- A在广州又可以给D转5000。

这就出事儿了，所以这类数据不会做跨城异地多活，只能用同城异区架构，因为同城的网络环境要好很多，可以搭建多条互联通道，成本也不会太高。

跨城异地模式适用于对数据一致性要求不高的场景，例如：

1. 用户登录，数据不一致时重新登录即可。
2. 新闻网站，一天内新闻数据变化较少。
3. 微博网站，即使丢失一点微博或评论数据也影响不大。

### **3. 跨国异地**

部署在不同国家的多个机房。

此模式的延时就更长了，正常情况也得几秒，无法满足正常的业务访问。

所以，跨国异地多活适用于：

- 为不同地区用户提供服务

例如，亚马逊美国是为美国用户服务的，亚马逊中国的账号是无法登录美国亚马逊的。

- 只读类业务

例如谷歌的搜索，不管用户在哪个国家搜索，得到的结果基本相同，对用户来说，跨国异地的几秒延迟，对搜索结果没什么影响。

## 备份

常用的数据备份方式为逻辑备份、物理备份与快照：

- 逻辑备份：数据库对象级备份，备份内容是表、索引、存储过程等数据库对象，常见工具为MySQL mysqldump、Oracle exp/imp等。
- 物理备份：数据库文件级备份，备份内容是操作系统上数据库文件，常见工具为MySQL XtraBackup、Oracle RMAN等。
- 快照：基于快照技术获取指定数据集合的一个完全可用拷贝，随后可以选择仅在本机上维护快照，或者对快照进行数据跨机备份，常见工具为文件系统Veritas File System、卷管理器Linux LVM、存储子系统NetApp NAS等。