## 事务：隔离性和锁-mysql-5

## Mysql Shell



## 事务简介



MySQL事务是访问并更新数据库中各种数据项的一个程序执行单元。在事务中的操作，要么都执行修改，要么都不执行，这就是事务的目的，也是事务模型区别于文件系统的重要特征之一。

MySQL事务主要用于处理操作量大，复杂度高的数据。

> 单个表的增删改查完全没得必要吖

事务广泛的运用于订单系统、银行系统等多种场景。如果有以下一个场景：A用户和B用户是银行的储户。现在A要给B转账500元。那么需要做以下几件事：

1）检查A的账户余额>500元；
2）A账户扣除500元；
3）账户增加500元；

　　正常的流程走下来，A账户扣了500，B账户加了500，皆大欢喜。那如果A账户扣了钱之后，系统出故障了呢？A白白损失了500，而B也没有收到本该属于他的500。

　　以上的案例中，隐藏着一个前提条件：A扣钱和B加钱，要么同时成功，要么同时失败。事务的需求就在于此！

## Mysql引擎

### InnoDB and MyISAM

在MySQL中只有使用了InnoDB数据库引擎的数据库或者表才支持事务。

MySQL 5.5 才将 InnoDB 代替 MyISAM 成为 MySQL 默认的存储引擎，而事务才有隔离级别一说，MyISAM 本就不支持事务。

MySQL从版本5.6开始支持InnoDB引擎的全文索引，不过从5.7.6版本开始才提供一种内建的全文索引ngram parser，支持CJK字符集。从5.7开始新增ngram插件以支持对中文的全文索引，以及用MeCab解析日文。

### 最大的区别

最主要的差别就是Innodb 支持事务处理与外键和行级锁。

> 行级锁并发高-不会像MyISAM一样容易锁表-

InnoDB是聚集索引，而MyISAM是非聚集索引。

 InnoDB 不保存表的具体行数，执行 select count(*) from table 时需要全表扫描。而MyISAM 用一个变量保存了整个表的行数，执行上述语句时只需要读出该变量即可，速度很快



## 事务的隔离性：看

查看当前事务，锁，进程

```sql
select * from information_schema.innodb_trx\G;
SELECT * FROM information_schema.INNODB_TRX\G
到information_schema库下面，查看下面这个表： 
innodb_trx ## 当前运行的所有事务 
innodb_locks ## 当前出现的锁 
innodb_lock_waits ## 锁等待的对应关系
```

end

### 准备数据

```bash
## create table
CREATE TABLE `account` (
  `id` int(11) NOT NULL,
  `name` varchar(255) DEFAULT NULL,
  `balance` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `un_name_idx` (`name`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

## insert into
insert into account (id, name, balance) values (1, 'a', 12);
insert into account (id, name, balance) values (2, 'b', 31);
insert into account (id, name, balance) values (3, 'ba', 349);

## 查看当前 事务隔离性
mysql> select @@tx_isolation;
+-----------------+
| @@tx_isolation  |
+-----------------+
| REPEATABLE-READ |
+-----------------+
1 row in set (0.00 sec)



```

end

### 读未提交RU &  脏读

Read  Uncommitted & Dirty read，最低的隔离级别，允许读取尚未提交的数据变更，**可能会导致脏读、幻读或不可重复读**。

假设现在有两个事务A、B：

- 假设现在A的余额是100，事务A正在准备查询Jay的余额
- 这时候，事务B先扣减Jay的余额，扣了10
- 最后A 读到的是扣减后的余额

由上图可以发现，事务A、B交替执行，事务A被事务B干扰到了，因为事务A读取到事务B未提交的数据,这就是**脏读**。

```bash
# 设置当前session的tx_isolation为 RU
# console 1
mysql> set session transaction isolation level read uncommitted;
Query OK, 0 rows affected (0.00 sec)

mysql> select @@tx_isolation;
+------------------+
| @@tx_isolation   |
+------------------+
| READ-UNCOMMITTED |
+------------------+
1 row in set (0.00 sec)

# console 2
mysql> select @@tx_isolation;
+-----------------+
| @@tx_isolation  |
+-----------------+
| REPEATABLE-READ |
+-----------------+
1 row in set (0.00 sec)

mysql> set session transaction isolation level read uncommitted;
Query OK, 0 rows affected (0.00 sec)

mysql> select @@tx_isolation;
+------------------+
| @@tx_isolation   |
+------------------+
| READ-UNCOMMITTED |
+------------------+
1 row in set (0.00 sec)

## console 1 开始事务

## 开始事务
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from account where id=1;
+----+------+---------+
| id | name | balance |
+----+------+---------+
|  1 | a    |      12 |
+----+------+---------+
1 row in set (0.00 sec)
## 将数据账户余额加20
mysql> update account set balance=balance+20 where id =1;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> select * from account where id=1;
+----+------+---------+
| id | name | balance |
+----+------+---------+
|  1 | a    |      32 |
+----+------+---------+
1 row in set (0.00 sec)

## console 2 也开始事务
mysql> begin;
Query OK, 0 rows affected (0.00 sec)
# 此时 console 1 还没commit，但是console 2已经读到事务1的数据了，这就是脏读
mysql> select * from account where id=1;
+----+------+---------+
| id | name | balance |
+----+------+---------+
|  1 | a    |      32 |
+----+------+---------+
1 row in set (0.00 sec)

## 在console 3中查看正在运行的事务
SELECT * FROM information_schema.INNODB_TRX\G
mysql> SELECT * FROM information_schema.INNODB_TRX\G
*************************** 1. row ***************************
                    trx_id: 2348
                 trx_state: RUNNING
               trx_started: 2021-07-19 14:37:44
     trx_requested_lock_id: NULL
          trx_wait_started: NULL
                trx_weight: 0
       trx_mysql_thread_id: 10
                 trx_query: NULL
       trx_operation_state: NULL
         trx_tables_in_use: 0
         trx_tables_locked: 0
          trx_lock_structs: 0
     trx_lock_memory_bytes: 360
           trx_rows_locked: 0
         trx_rows_modified: 0
   trx_concurrency_tickets: 0
       trx_isolation_level: READ UNCOMMITTED
         trx_unique_checks: 1
    trx_foreign_key_checks: 1
trx_last_foreign_key_error: NULL
 trx_adaptive_hash_latched: 0
 trx_adaptive_hash_timeout: 10000
          trx_is_read_only: 0
trx_autocommit_non_locking: 0
*************************** 2. row ***************************
                    trx_id: 2347
                 trx_state: RUNNING
               trx_started: 2021-07-19 14:36:15
     trx_requested_lock_id: NULL
          trx_wait_started: NULL
                trx_weight: 3
                # 线程ID
       trx_mysql_thread_id: 9
                 trx_query: NULL
       trx_operation_state: NULL
         trx_tables_in_use: 0
         trx_tables_locked: 0
          trx_lock_structs: 2
     trx_lock_memory_bytes: 360
           trx_rows_locked: 1
         trx_rows_modified: 1
   trx_concurrency_tickets: 0
       trx_isolation_level: READ UNCOMMITTED
         trx_unique_checks: 1
    trx_foreign_key_checks: 1
trx_last_foreign_key_error: NULL
 trx_adaptive_hash_latched: 0
 trx_adaptive_hash_timeout: 10000
          trx_is_read_only: 0
trx_autocommit_non_locking: 0
2 rows in set (0.00 sec)

mysql> kill 9

## kill 掉 console 1的事务
mysql> kill 9;
Query OK, 0 rows affected (0.00 sec)

## 再在console 1中查看数据
mysql> select * from account where id=1;
ERROR 2006 (HY000): MySQL server has gone away
No connection. Trying to reconnect...
## 因为kill 掉了了console 1的线程，所以这里生成了新的 线程id 为 14.
Connection id:    14
Current database: test_db

+----+------+---------+
| id | name | balance |
+----+------+---------+
|  1 | a    |      12 
+----+------+---------+
1 row in set (0.01 sec)

## 再在console 1中查看数据
mysql> select * from account where id=1;
+----+------+---------+
| id | name | balance |
+----+------+---------+
|  1 | a    |      12 |
+----+------+---------+
1 row in set (0.00 sec)

## 可以看到console 1新的进程就是 14
mysql> show processlist;
+----+------+-----------+---------+---------+-------+-------+------------------+
| Id | User | Host      | db      | Command | Time  | State | Info             |
+----+------+-----------+---------+---------+-------+-------+------------------+
|  8 | root | localhost | test_db | Sleep   | 18719 |       | NULL             |
| 10 | root | localhost | test_db | Sleep   |   180 |       | NULL             |
| 14 | root | localhost | test_db | Sleep   |   207 |       | NULL             |
| 15 | root | localhost | NULL    | Query   |     0 | init  | show processlist |
+----+------+-----------+---------+---------+-------+-------+------------------+
4 rows in set (0.00 sec)

```

可以发现，在**读未提交（Read Uncommitted）** 隔离级别下，一个事务会读到其他事务未提交的数据的，即存在**脏读**问题。事务B都还没commit到数据库呢，事务A就读到了，直接乱套了。。。实际上，读未提交是隔离级别最低的一种。

### 读已提交RC & 不可重复读

READ COMMITTED 可以解决 dirty read

RC，顾名思义，只能读到事务已提交的数据，所以可以避免脏读的问题。

隔离级别设置为**已提交读（READ COMMITTED）** 时，已经不会出现脏读问题了，当前事务只能读取到其他事务提交的数据。但是，站在事务A的角度想想，存在其他问题吗？

在同一个事务A里，相同的查询sql，读取同一条记录（id=1），读到的结果是不一样的，即**不可重复读**。所以，隔离级别设置为read committed的时候，还会存在**不可重复读**的并发问题。

RC级别相对要严格一些，至少是要等其它事务把变更提交到db，才能读取到，听上去蛮靠谱的。但是有些业务场景，比如会员系统中，如果要在一个事务中，多次读取用户身份，判断是否会员，如果刚开始读取到该用户是会员，做了一些逻辑处理，后面又读到用户不是会员了，这就有点崩溃，不知道如何继续。

这种希望同1个事务中，关键数据不管读取多次，而且结果都一样，RC级别就不行了。

```bash
## console 1
mysql> select @@tx_isolation;
+-----------------+
| @@tx_isolation  |
+-----------------+
| REPEATABLE-READ |
+-----------------+
1 row in set (0.00 sec)

mysql> set session transaction isolation level read committed;
Query OK, 0 rows affected (0.00 sec)

mysql> select @@tx_isolation;
+----------------+
| @@tx_isolation |
+----------------+
| READ-COMMITTED |
+----------------+
1 row in set (0.00 sec)

mysql> select * from account where id=1;
+----+------+---------+
| id | name | balance |
+----+------+---------+
|  1 | a    |      62 |
+----+------+---------+
1 row in set (0.00 sec)

## console 2 新起一个事务来修改id为1的balance
mysql> select @@tx_isolation;
+----------------+
| @@tx_isolation |
+----------------+
| READ-COMMITTED |
+----------------+
1 row in set (0.00 sec)
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from account where id=1;
+----+------+---------+
| id | name | balance |
+----+------+---------+
|  1 | a    |      62 |
+----+------+---------+
1 row in set (0.00 sec)

mysql> update account set balance=balance+20 where id=1;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> select * from account where id=1;
+----+------+---------+
| id | name | balance |
+----+------+---------+
|  1 | a    |      82 |
+----+------+---------+
1 row in set (0.00 sec)

mysql> commit;
Query OK, 0 rows affected (0.01 sec)

## console 继续select ，发现一个事务中的多次的select 看到的值已经不一样了，
mysql> begin;
...

mysql> select * from account where id=1;
+----+------+---------+
| id | name | balance |
+----+------+---------+
|  1 | a    |      82 |
+----+------+---------+
1 row in set (0.00 sec)


mysql> commit;
Query OK, 0 rows affected (0.00 sec)


```

在同个事务中，查询结果必须是一致的，需要用到RR。

 RC 允许读取并发事务已经提交的数据，**可以阻止脏读，但是幻读或不可重复读仍有可能发生**。

### 可重复读 RR & 幻读

Repeatable Read， 对同一字段的多次读取结果都是一致的，除非数据是被本身事务自己所修改，

这，在transaction 开始时，先读取下目标数据，然后存在一个位置，后续涉及到该数据时都不去库了读取了，直接使用transaction刚开始时，读取的数据，这样就能实现repeatable read了。

```bash
## mysql的默认隔离级别就是RR
mysql> select @@tx_isolation;
+-----------------+
| @@tx_isolation  |
+-----------------+
| REPEATABLE-READ |
+-----------------+
1 row in set (0.00 sec)

## console 1： 一个事务中多次select 一条数据
begin;
select * from account where id=1;
# select after update
# 同一个事务中数据保持一致性
select * from account where id=1;
commit;

## console 2： 一个事务中修改一条记录
begin;
update account set balance=balance+20 where id=1;
commit;
```

**可以阻止脏读和不可重复读，但幻读仍有可能发生**。

即在RR模式下，虽然“快照”了事务开始前的数据，可以实现Repeatable Read 但是当其他事务插入或者删除数据时，当前事务是感知不到的。

现实中的栗子就是：商品超卖。

比如秒杀订单系统中，事务A第1次读取商品库存，发现还有1个，可以下单，赶紧继续，但是此时，可能有另一个事务，也在下单，已经提交了订单，把库存减为0了，事务A并不知道，因为多次读取库存的值是一样的，还是1，最后仍然把订单创建了，形成超卖。

幻读，是指两次当前读的对比，下面不算幻读？？？

所谓幻读，指的是当某个事务在读取某个范围内的记录时，另外一个事务又在该范围内插入了新的记录，当之前的事务再次读取该范围的记录时，会产生幻行（Phantom Row）

RR 隔离级别下 mysql 并不会自动的给 SELECT 操作添加行锁/next-key lock锁，所以会存在幻读的问题，如果你 select ... for update 对记录手动加上行锁，自然不会幻读。

```bash
# console 1
mysql> select @@tx_isolation;
+-----------------+
| @@tx_isolation  |
+-----------------+
| REPEATABLE-READ |
+-----------------+
1 row in set (0.00 sec)

mysql> begin;
Query OK, 0 rows affected (0.00 sec)
Database changed
mysql> select * from account where id > 2;
+----+------+---------+
| id | name | balance |
+----+------+---------+
|  3 | ba   |     349 |
+----+------+---------+
1 row in set (0.00 sec)

# console 2 : 插入和删除数据
mysql> use test_db;
Database changed
mysql> select @@tx_isolation;
+-----------------+
| @@tx_isolation  |
+-----------------+
| REPEATABLE-READ |
+-----------------+
1 row in set (0.00 sec)

mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> delete from account where id=3;
Query OK, 1 row affected (0.01 sec)

mysql> select * from account where id > 2;
Empty set (0.00 sec)


## console 1 再次查看
## id 3的数据都已经被 console 2删除了，这里依然显示。
mysql> select * from account where id > 2;
+----+------+---------+
| id | name | balance |
+----+------+---------+
|  3 | ba   |     349 |
+----+------+---------+
1 row in set (0.00 sec)


## console 2 再插入一条数据， 暂时不commit
mysql> insert into account (id, name, balance) values (5, 'dodo', 78);
Query OK, 1 row affected (0.00 sec)

mysql> select * from account where id > 2;
+----+------+---------+
| id | name | balance |
+----+------+---------+
|  5 | dodo |      78 |
+----+------+---------+
1 row in set (0.00 sec)



## console 1 ：数据仍不变，在其他事务中 id=3，已经被删除了。
mysql> select * from account where id > 2;
+----+------+---------+
| id | name | balance |
+----+------+---------+
|  3 | ba   |     349 |
+----+------+---------+
1 row in set (0.00 sec)

## console 2
mysql> commit;
Query OK, 0 rows affected (0.00 sec)

## console 1 数据才会改变...
mysql> select * from account where id > 2;
+----+------+---------+
| id | name | balance |
+----+------+---------+
|  5 | dodo |      78 |
+----+------+---------+
1 row in set (0.00 sec)


```

幻读和不可重复读有些相似之处 ，但是不可重复读的重点是修改，幻读的重点在于新增或者删除。

### 串行化

SERIALIZABLE

从db层面，要想同时解决脏读、不可重复读、幻读，只有串行化这个级别可以做到

数据库隔离级别设置为serializable的时候，事务B对表的写操作，在等事务A的读操作。其实，这是隔离级别中最严格的，读写都不允许并发。它保证了最好的安全性，性能却是个问题~

```bash
## console 1 和 console 2都设置为serializable 
mysql> set session transaction isolation level serializable;
Query OK, 0 rows affected (0.00 sec)

mysql> select @@tx_isolation;
+----------------+
| @@tx_isolation |
+----------------+
| SERIALIZABLE   |
+----------------+
1 row in set (0.00 sec)


## console 1：select...where 行锁
mysql> begin;
Query OK, 0 rows affected (0.00 sec)
# 锁住id为1这行记录
mysql> select * from account where id=1;
+----+------+---------+
| id | name | balance |
+----+------+---------+
|  1 | a    |      82 |
+----+------+---------+
1 row in set (0.00 sec)

## console 2 ： 插入数据
## 在console 1 对某行数据加锁时，console 2 还是可以操作其他的
mysql> insert into account (id, name, balance) values (11, 'aaa', 89);
Query OK, 1 row affected (0.00 sec)


## 但是此时 console 1 使用select all 读整个表，就会导致，console 2 无法操作
mysql> select * from account;
+----+------+---------+
| id | name | balance |
+----+------+---------+
|  1 | a    |      82 |
|  2 | b    |      41 |
|  5 | dodo |      78 |
| 11 | aaa  |      89 |
+----+------+---------+
4 rows in set (0.00 sec)

## console 2 超时
mysql> insert into account (id, name, balance) values (110, 'aaa', 89);
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction

```

如上所见，这就是在serializable模式下，不同读写的并发都不运行，这样可以避免幻读。





### 自己的理解

数据库要解决读写并发问题，要保证事务的特性。像写代码一个，不断的解决问题，引入bug，再发现再解决。

* v1：Read Uncommitted and Read Committed，未提交的数据就被其他事务读到影响太大，就修改事务的逻辑，只能读到已经处于 committed 状态的数据。
* v2： RR版本。 RC状态下，会出现不可重复读的现象即其他事务修改了该行数据（因为事务1 读不加锁），导致同一个事务中，多次读同一行数据，读到的结果不一致。于是，增加了类似“快照读”的概念，当事务开始时，就对要涉及的数据进行快照。可以通过读快照来实现可重复读。
* v3： Serializable模式下，不同读写的并发都不运行，这可以解决幻读，但是效率奇差。因为一旦一个事务，执行了`select * form XXX;` 就会导致整个表被锁住。



### 为什么默认是RR（不错）

为了规避 MySQL5.0 以前版本的主从复制问题，然后一直被沿用了下来而已。

https://www.cnblogs.com/youzhibing/p/13131485.html#autoid-4-2-0

那Mysql在5.0这个版本以前，binlog只支持`STATEMENT`这种格式！而这种格式在**读已提交(Read Commited)**这个隔离级别下主从复制是有bug的，因此Mysql将**可重复读(Repeatable Read)**作为默认的隔离级别！
接下来，就要说说当binlog为`STATEMENT`格式，且隔离级别为**读已提交(Read Commited)**时，有什么bug呢？如下图所示，在主(master)上执行如下事务

即，时间线顺序为

S1_Begin --> S1_Delete --> S2 _Begin-->S2_instert-->S2_Commit --> S1_Commit.

这时候Master上数据是正常的，但是同步到Slave的binlog会出问题，由于是S2先commit，所以Slave上Sql执行的语句是先插入再删除。如果Delete的条件包含了新的Instert，就会导致Slave上数据丢失。。。。


解决方案有两种！
(1)隔离级别设为**可重复读(Repeatable Read)**,在该隔离级别下引入间隙锁。当`Session 1`执行delete语句时，会锁住间隙。那么，`Ssession 2`执行插入语句就会阻塞住！
(2)将binglog的格式修改为row格式，此时是基于行的复制，自然就不会出现sql执行顺序不一样的问题！奈何这个格式在mysql5.1版本开始才引入。因此由于历史原因，mysql将默认的隔离级别设为**可重复读(Repeatable Read)**，保证主从复制不出问题！





### 互联网推荐RC隔离？

公司的MySQL隔离级别是Read Commited，已经没有了gap lock，而且代码里的sql都再简单不过，没有显式加锁的sql语句。

http://www.fanyilun.me/2017/04/20/MySQL%E5%8A%A0%E9%94%81%E5%88%86%E6%9E%90/

先明确一点！项目中是不用**读未提交(Read UnCommitted)**和**串行化(Serializable)**两个隔离级别，原因有二

- 采用**读未提交(Read UnCommitted)**,一个事务读到另一个事务未提交读数据，这个不用多说吧，从逻辑上都说不过去！
- 采用**串行化(Serializable)**，每个次读操作都会加锁，快照读失效，一般是使用mysql自带分布式事务功能时才使用该隔离级别！(笔者从未用过mysql自带的这个功能，因为这是XA事务，是强一致性事务，性能不佳！互联网的分布式方案，多采用最终一致性的事务解决方案！)

也就是说，我们该纠结都只有一个问题，究竟隔离级别是用读已经提交呢还是可重复读？
接下来对这两种级别进行对比，讲讲我们为什么选**读已提交(Read Commited)**作为事务隔离级别！
假设表结构如下

```
 CREATE TABLE `test` (
`id` int(11) NOT NULL,
`color` varchar(20) NOT NULL,
PRIMARY KEY (`id`)
) ENGINE=InnoDB
```

数据如下

```
+----+-------+
| id | color |
+----+-------+
|  1 |  red  |
|  2 | white |
|  5 |  red  |
|  7 | white |
+----+-------+
```

为了便于描述，下面将

- **可重复读(Repeatable Read)**，简称为RR；
- **读已提交(Read Commited)**，简称为RC；

*缘由一：在RR隔离级别下，存在间隙锁，导致出现死锁的几率比RC大的多！*
此时执行语句

```
select * from test where id <3 for update;
```

在RR隔离级别下，存在间隙锁，可以锁住(2,5)这个间隙，防止其他事务插入数据！
而在RC隔离级别下，不存在间隙锁，其他事务是可以插入数据！

`ps`:在RC隔离级别下并不是不会出现死锁，只是出现几率比RR低而已！

*缘由二：在RR隔离级别下，条件列未命中索引会锁表！而在RC隔离级别下，只锁行*
此时执行语句

```
update test set color = 'blue' where color = 'white'; 
```

在RC隔离级别下，其先走聚簇索引，进行全部扫描。加锁如下：
![img](https://img2018.cnblogs.com/blog/725429/201903/725429-20190311135044150-2146259021.png)
但在实际中，MySQL做了优化，在MySQL Server过滤条件，发现不满足后，会调用unlock_row方法，把不满足条件的记录放锁。
实际加锁如下
![img](https://img2018.cnblogs.com/blog/725429/201903/725429-20190311135057008-2073392330.png)
然而，在RR隔离级别下，走聚簇索引，进行全部扫描，最后会将整个表锁上，如下所示
![img](https://img2018.cnblogs.com/blog/725429/201903/725429-20190311135808965-164228503.jpg)

*缘由三：在RC隔离级别下，半一致性读(semi-consistent)特性增加了update操作的并发性！*
在5.1.15的时候，innodb引入了一个概念叫做“semi-consistent”，减少了更新同一行记录时的冲突，减少锁等待。
所谓半一致性读就是，一个update语句，如果读到一行已经加锁的记录，此时InnoDB返回记录最近提交的版本，由MySQL上层判断此版本是否满足update的where条件。若满足(需要更新)，则MySQL会重新发起一次读操作，此时会读取行的最新版本(并加锁)！
具体表现如下:
此时有两个Session，Session1和Session2！
Session1执行

```
update test set color = 'blue' where color = 'red'; 
```

先不Commit事务！
与此同时Ssession2执行

```
update test set color = 'blue' where color = 'white'; 
```

session 2尝试加锁的时候，发现行上已经存在锁，InnoDB会开启semi-consistent read，返回最新的committed版本(1,red),(2，white),(5,red),(7,white)。MySQL会重新发起一次读操作，此时会读取行的最新版本(并加锁)!
而在RR隔离级别下，Session2只能等待！

####两个疑问

*在RC级别下，不可重复读问题需要解决么？*
不用解决，这个问题是可以接受的！毕竟你数据都已经提交了，读出来本身就没有太大问题！Oracle的默认隔离级别就是RC，你们改过Oracle的默认隔离级别么？

*在RC级别下，主从复制用什么binlog格式？*
OK,在该隔离级别下，用的binlog为row格式，是基于行的复制！Innodb的创始人也是建议binlog使用该格式！

互联网项目推荐使用RC？？：https://www.cnblogs.com/rjzheng/p/10510174.html



https://juejin.cn/post/6844904115353436174



RR解决幻读是通过手动加锁来实现的，如果加锁不正确，还是会出现幻读，在不进行全表锁的情况下，很难避免其他事务不会对当前事务有影响。所以，serializable的好处就是完全隔绝了其他事务对当前事务的影响，在开发人员对数据库理解不够的时候，使用serializable比手动加锁要安全

## InnoDB存储引擎

MySQL InnoDB存储引擎，实现的是基于多版本的并发控制协议——MVCC (Multi-Version Concurrency Control) (注：与MVCC相对的，是基于锁的并发控制，Lock-Based Concurrency Control)。

### LBCC

**LBCC，基于锁的并发控制，Lock Based Concurrency Control。**

使用锁的机制，在当前事务需要对数据修改时，将当前事务加上锁，同一个时间只允许一条事务修改当前数据，其他事务必须等待锁释放之后才可以操作。

按照模式划分，可分为共享锁、排它锁、意向锁（意向共享锁、意向排它锁）、自增锁：

- 共享锁（行锁）—— Shared Locks（简称S锁）：
  说明：InnoDB引擎的行锁机制锁的是索引，不是行记录
  又叫读锁，多个事务可以共用同一把共享锁读取数据，但是无法修改，想要修改，必须所有的共享锁释放完成之后才可以。
  共享锁加锁方式： select * from table where id = 1 lock in share mode
  释放锁的方式：Commit 提交、Rollback 回滚
- 排它锁（行锁）——Exclusive Locks （简称X锁）
  一个事务对某一个资源上了排它锁后，只允许当前事务对其进行增删改查，其它人无法对当前资源进行任何操作，包括都操作。**即排它锁和其他锁匙互斥关系**
  排它锁加锁方式：
  自动：DML语句默认会自动加锁
  手动：select * from student where id = 1 for update
  释放锁的方式：Commit 提交、Rollback 回滚
- 意向锁（表锁）—— Intention Locks：
  意向锁不是用来锁定数据的，而是用来告诉这个表中是否加了排他锁、共享锁，这样以后再创建表锁的时候不用去扫描表中排它锁、共享锁的状态，直接根据意向锁状态就可以知道是否可以创建，可以理解成一个标记。目的是为了提升加表锁的效率。
- 自增锁（表锁） Auto-inc Locks：
  当向使用含有AUTO_INCREMENT列的表中插入数据时需要获取的一种特殊的表级锁
  自增锁并不是事务锁，为了保证效率，自增锁的释放可以通过一个参数innodb_autoinc_lock_mode控制
  该参数可以设置三个值 0 | 1 | 2，三个值分别代表的意思如下：
  0：语句执行结束后才释放锁
  1：普通insert语句，申请完锁后就释放、insert … select 语句执行结束后释放，保证顺序
  2：申请后就释放锁

锁的算法分为 记录锁（Record Locks）、临键锁（Next-Key Locks）、间隙锁（Gap Locks）


### MVCC

**多版本并发控制**技术的英文全称是 **Multiversion Concurrency Control**，简称 **MVCC**。

**MVCC只是工作在两种事务级别底下：Read Committed and Repeatable Read;**

因为其他两种：

* READ UNCOMMITTED==》总是读取最新的数据，不符合当前事务版本的数据行，

* Serializable则会对所有的读写操作加锁

**多版本并发控制（MVCC）** 是通过保存数据在某个时间点的快照来实现并发控制的。也就是说，不管事务执行多长时间，事务内部看到的数据是不受其它事务影响的，根据事务开始的时间不同，每个事务对同一张表，同一时刻看到的数据可能是不一样的。



在MVCC并发控制中，读操作可以分成两类：快照读 (snapshot read)与当前读 (current read)。

快照读，读取的是记录的可见版本 (有可能是历史版本)，不用加锁。

当前读，读取的是记录的最新版本，并且当前读返回的记录，会加上锁，保证其他事务不会再并发修改这条记录。

#### 总结

太秒了，太妙了...

read view 和 行记录的数据结构，相互关联(通过txc_id)，实现了mvcc。

####Read View(重要)

Read View 就是一个保存事务ID的list列表，记录是的本事务执行时，MySQL还有哪些事务在执行。RC是每次执行读SQL语句的时候都创建一个Read View，而RR是在事务启动的时候创建一个Read View。

按照可重复读的定义，一个事务启动的时候，能够看到所有已经提交的事务结果。但是之后，这个事务执行期间，其他事务的更新对它不可见。

因此，一个事务只需要在启动的时候声明说，“以我启动的时刻为准，如果一个数据版本是在我启动之前生成的，就认；如果是我启动以后才生成的，我就不认，我必须要找到它的上一个版本”。

当然，如果“上一个版本”也不可见，那就得继续往前找。还有，如果是这个事务自己更新的数据，它自己还是要认的

在实现上， InnoDB为每个事务构造了一个数组，用来保存这个事务启动瞬间，当前正在“活跃”的所有事务ID。**“活跃”指的就是，启动了但还没提交**。



InnoDB支持MVCC多版本，其中RC（Read Committed）和RR（Repeatable Read）隔离级别是利用consistent read view（一致读视图）方式支持的。 所谓consistent read view就是在某一时刻给事务系统trx_sys打snapshot（快照），把当时trx_sys状态（包括活跃读写事务数组）记下来，之后的所有读操作根据其事务ID（即trx_id）与snapshot中的trx_sys的状态作比较，以此判断read view对于事务的可见性。

ReadView是什么呢？在我们平时执行一个事务的时候，就会生成一个ReadView，ReadView的组成结构大致如下

MySQL实现MVCC的关键——ReadView。

ReadView说白了就是一个数据结构，在SQL开始的时候被创建。


参数说明:

creator_trx_id:当前事务id
m_ids:所有事务的事务id（所用活跃事务，活跃 == 未提交？？？）
min_trx_id: 表示在生成ReadView时当前系统中活跃的读写事务中最小的事务id，也就是m_ids中的最小值
max_trx_id: 表示生成ReadView时系统中应该分配给下一个事务的id值。

> 注意max_trx_id并不是m_ids中的最大值，事务id是递增分配的。比方说现在有id为1，2，3这三个事务，之后id为3的事务提交了。那么一个新的读事务在生成ReadView时，m_ids就包括1和2，min_trx_id的值就是1，max_trx_id的值就是4。

我们来具体分析一下ReadView的作用，以及是如何解决脏读幻读，不可重复读的问题的

现在数据有一条数据，如下原始值，上一个已经提交事务的事务txr_id=30

![readview1](D:\worker\h3c\数据库&&mysql\images\readview1.png)

> 每个行记录都有这三个隐藏列太重要了：
>
> 1. row_id :隐藏的行 ID ,用来生成默认的聚集索引。如果创建数据表时没指定聚集索引，这时 InnoDB 就会用这个隐藏 ID 来创建聚集索引。采用聚集索引的方式可以提升数据的查找效率。
> 2. trx_id: 操作这个数据事务 ID ，也就是最后一个对数据插入或者更新的事务 ID 。
> 3. roll_ptr:回滚指针，指向这个记录的 Undo Log 信息。

此时两个事务并发执行， 事务A(txr_id= 40),事务B(txr_id=50),事务A要读取这行数据，事务B要更新这行数据。

此时事务A在读取时开启ReadView，此时的ReadView的值如下：

m_ids:=[40,50]

min_trx_id=40

max_trx_id=51， 51有点意思--

creator_trx_id=40

事务A在第一次查询发起一个判断，判断当前数据的txr_id是否小于ReadView中的min_trx_id(此时txr_id (30)< min_trx_id(40)),说明在A事务开启前，修改这行数据的事务已经提交了,所以可以查询这条数据

![readview2](D:\worker\h3c\数据库&&mysql\images\readview2.png)

接着事务B开始修改数据为B值，然后**将这行数据的txr_id设置为自己的txr_id(50**),然后将roll_pointer指向之修改前生成的undo log备份，然后提交事务

> 每行记录的隐藏列txr_id总是记录最后修改它的事务id number.

![readview3](D:\worker\h3c\数据库&&mysql\images\readview3.png)

此时A再重复查询，当读取B修改的数据时，发现在ReadView中min_txr_id(40)<txr_id(50)<max_trx_id(51),同时txr_id=50在m_ids（m_ids=[40,50]），可以确定修改数据的事务和自己处于同一时间并发提交的，为了保证可重复读，这行数据是不能查询的，所以需要通过roll_pointer顺着undo log链表找到最近的一条undo log，即修改前的值，找到txr_id = 30（B未修改的值） 发现此时原始值的 txr_id < min_trx_id，说明这条数据肯定是在事务A开启钱前提交的，所以可以查询，就查询txr_id=30的值，即查询的的还是原始的值，这样就保证了可重复读

> 如果此时的数据隔离级别是Read committed，即B事务一旦提交，A下次查询就可以马上查询到事务B提交的值了,ReadView是如何做到这一点的呢？
> 那就是每次事务A查询时都新生成ReadView，来看看在事务B提交后A重新查询后生成的ReadView是什么样的
>
> 新的ReadView：
>
> * m_ids = [40], 因为之前的活跃事务已经提交了。
> * min_trx_id = 40
> * max_trx_id = 51
> * create_trx_id = 40
>
> ![readview4](D:\worker\h3c\数据库&&mysql\images\readview4.png)
>
> 这次A重复查询在ReadView中 min_trx_id(40)<trx_id(50)<max_trx_id(51)，但是事务B的事务id不在m_ids中，说明事务B虽然和自己同一时间开启，但是已经提交了事务，所以可以查询B修改的数据即txr_id = 50的值
>
> 在RC模式下，trx_id > min_trx_id，则说明当前记录行是其他事务提交的，根据RC则是可以读的。
>
> 在RR模式下，trx_id > min_trx_id，且min_trx_id在m_ids, 则肯定是不可读的.

接着事务A再更新这行数据，此时这行数据的trx_id=40,此时数据如下

![readview5](D:\worker\h3c\数据库&&mysql\images\readview5.png)

此时事务A重新查询，发现txr_id=40 = creator_trx_id，说明这条数据就是自己修改的，也是可以读取到，所以就读取到

如果此时再开启一个事务C，事务id=60,更新这行数据

![readview6](D:\worker\h3c\数据库&&mysql\images\readview6.png)

如果A再次查询会发现trx_id=60 > max_trx_id(id=51),说明事务C是事务A开启后开启的，自己也是不能读取到，所以也是顺着undo log向下找到 txr_id = 40 读取到值

同理如果事务隔离级别是 Repeatable read，C事务插入一条数据也是如此，A是读取不到的，从而也解决了幻读问题
————————————————
版权声明：本文为CSDN博主「weihubeats」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/qq_42651904/article/details/110622818



#### snapshot read

读取的是记录的可见版本 (有可能是历史版本)，不用加锁。

#### current read

当前读，读取的是记录的最新版本，并且，当前读返回的记录，都会加上锁，保证其他事务不会再并发修改这条记录。

特殊的读操作，插入/更新/删除操作，属于当前读，需要加锁。

```sql
select * from table where ? lock in share mode;
select * from table where ? for update;
insert into table values (…);
update table set ? where ?;
delete from table where ?;
```

所有以上的语句，都属于当前读，读取记录的最新版本。并且，读取之后，还需要保证其他并发事务不能修改当前记录，对读取记录加锁。其中，除了第一条语句，对读取记录加S锁 (共享锁)外，其他的操作，都加的是X锁 (排它锁)。

一个Update操作的具体流程：

当Update SQL被发给MySQL后，MySQL Server会根据where条件，读取第一条满足条件的记录，然后InnoDB引擎会将第一条记录返回，并加锁 (current read)。待MySQL Server收到这条加锁的记录之后，会再发起一个Update请求，更新这条记录。一条记录操作完成，再读取下一条记录，直至没有满足条件的记录为止。因此，Update操作内部，就包含了一个当前读。同理，Delete操作也一样。Insert操作会稍微有些不同，简单来说，就是Insert操作可能会触发Unique Key的冲突检查，也会进行一个当前读。

#### MVCC思考

**在RC（读已提交）和RR（可重复度）级别下，MVCC都会生效，那么为什么RC不可以解决幻读，而RR可以解决幻读？**

https://blog.csdn.net/qq_35634181/article/details/113280233

**原因：** 两种隔离界别下的核心处理逻辑就是判断所有版本中哪个版本是当前事务可见的处理。针对这个问题InnoDB在设计上增加了ReadView的设计，ReadView中主要包含当前系统中还有哪些活跃的读写事务，把它们的事务id放到一个列表中，我们把这个列表命名为为m_ids。

 以上内容是对于 RR 级别来说，而对于 RC 级别，其实整个过程几乎一样，唯一不同的是生成 ReadView 的时机，RR 级别只在事务开始时生成一次，之后一直使用该 ReadView。而 RC 级别则在每次 select 时，都会生成一个 ReadView。

 流弊

RC为什么避免不了不可重复读？

因为RC每次执行语句前都会重新建立一个新的read view，所以在第一个SELECT之后有新的事务提交了对某一行数据的更新操作，那么当前RC级别事务的第二个SELECT 操作是可以读到这个变更的，因为此时read view是空的。

RR为什么可以避免不可重复读？

因为RR是在事务开始的时候构建一次read view一直到事务提交之前不会重新构建，所以对于一个RR级别事务执行中也在执行更新的事务来说，后者在前者的read view中，所以无法看到它的数据。即使第二个事务提交了，对于第一个RR级别的事务来说，由于后者已经在它的read view中了，所以是看不到的。




简易版的说：
       当前活跃的事务列表集合，从小到大的顺序排列，在这个集合中的事务列表说明并未提交，对于本事务来说是不可见的，比该事件列表最大的id还要大，说明本事务执行时，修改记录的事务还未开始，所以比该事件列表最大的id还要大的，那么就是不可见的

在RR级别下，事务执行第一个 select 语句的时候，会生成一个当前时间点的事务快照 ReadView，主要包含以下几个属性：
trx_ids：生成 ReadView 时当前系统中活跃的事务 Id 列表，活跃的事务 Id 列表从小到大的顺序排列，就是还未执行事务提交的。
creator_trx_id：生成该 ReadView 的事务的事务 Id。


在访问某条记录时，会根据该事务中的事务快照 ReadView进行判断：

行记录中的事务id是否为该事务本身的id，相同则说明自己创建的，可见
行记录中的事务id比事务快照 ReadView中最小的还小，表明该记录为之前提交的，能够被当前事务访问。
行记录中的事务id比事务快照 ReadView中最大的事务id还要大，表明该记录为之后提交的，不能被当前事务访问
行记录中的事务id在活跃的事务 Id 列表之间，那么就使用二分，判断是否能在活跃集合中找到该事务id，能找到，说明该事务快照时，还未提交，本事务无法访问，如不在活跃事务集合中，说明已提交，可以访问
      在进行判断时，首先会拿记录的最新版本来比较，如果该版本无法被当前事务看到，则通过记录的 DB_ROLL_PTR 找到上一个版本，重新进行比较，直到找到一个能被当前事务看到的版本。
————————————————
版权声明：本文为CSDN博主「goodluckwj」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/qq_35634181/article/details/113280233

#### MVCC的实现

##### 事务版本号

每开启一个事务，我们都会从数据库中获得一个事务 ID（也就是事务版本号），这个事务 ID 是自增长的，通过 ID 大小，我们就可以判断事务的时间顺序。

##### 行记录的隐藏列

这三个隐藏列太重要了：

1. row_id :隐藏的行 ID ,用来生成默认的聚集索引。如果创建数据表时没指定聚集索引，这时 InnoDB 就会用这个隐藏 ID 来创建聚集索引。采用聚集索引的方式可以提升数据的查找效率。
2. trx_id: 操作这个数据事务 ID ，也就是最后一个对数据插入或者更新的事务 ID 。
3. roll_ptr:回滚指针，指向这个记录的 Undo Log 信息。

InnoDB 的叶子段存储了数据页，数据页中保存了行记录，而在行记录中有一些重要的隐藏字段：

- `DB_ROW_ID`：6-byte，隐藏的行 ID，用来生成默认聚簇索引。如果我们创建数据表的时候没有指定聚簇索引，这时 InnoDB 就会用这个隐藏 ID 来创建聚集索引。采用聚簇索引的方式可以提升数据的查找效率。

- `DB_TRX_ID`：6-byte，操作这个数据的事务 ID，也就是最后一个对该数据进行插入或更新的事务 ID。

- `DB_ROLL_PTR`：7-byte，回滚指针，也就是指向这个记录的 Undo Log 信息。

  即叶子节点段(Leaf Node Segment) 中的回滚指针指向回滚段(undo segment)

  下图所示：

  ![Undo Log回滚历史记录](https://segmentfault.com/img/bVbyzV9)

  从图中能看到回滚指针将数据行的所有快照记录都通过链表的结构串联了起来，每个快照的记录都保存了当时的 db_trx_id，也是那个时间点操作这个数据的事务 ID。这样如果我们想要找历史快照，就可以通过遍历回滚指针的方式进行查找。

##### MVCC实现RR隔离

MVCC的实现是select操作的时候做了一个活跃事务列表的快照，mvcc 就是通过这个事务列表到 undo log 版本链中查找满足条件的数据的。

###### **版本链对比规则**
在RR隔离级别下当mysql 在进行查询操作的时候，会生成一致性视图readview。是通过未提交事务和最大事务ID形成的快照列表，在进行数据查找的时候有一个版本链对比规则：

1. 当前trx_id < 最小事务id，说明当前事务时最早进来的，说明当前数据是可见的

2. 当前trx_id > 最大事务id，说明是其它事务生成的时候，后来生成的事务id，肯定是不可见的

   合理，很合理。

3. 如果当前在最大事务和最小事务中间，则分两种情况

   a、row 的 trx id 在最大事务和最小事务中间，表示当前版本是在还没有提交的事务的基础上提交的，所以不可见

   b、row 的trx id 不在活跃事务id 范围内，说明当前版本是已经提交了事务id的，说明可见、



###### 查询（SELECT）

这和我想的代码逻辑差不多。哈哈哈 ，它的很巧妙，存每个事务的TRX_ID对比就行了，而且所有的数据都使用一个数据结构，真的厉害。

InnoDB 会根据以下两个条件检查每行记录：

1. InnoDB只查找版本早于当前事务版本的数据行（也就是，行的系统版本号小于或等于事务的系统版本号），这样可以**确保事务读取的行，要么是在事务开始前已经存在的，要么是事务自身插入或者修改过的**。
2. 行的删除版本要么未定义，要么大于当前事务版本号。这可以确保**事务读取到的行，在事务开始之前未被删除**。

只有符合上述两个条件的记录，才能返回作为查询结果。

###### 插入（INSERT）

InnoDB为新插入的每一行保存当前系统版本号作为行版本号。

###### 删除（DELETE）

InnoDB为删除的每一行保存当前系统版本号作为行删除标识。
删除在内部被视为更新，行中的一个特殊位会被设置为已删除。

**对于删除的情况，可以认为是update的修改操作：**

在进行删除操作的，会复制一份最新的数据到版本链，并修改事务ID为delete操作的事务id，同时在当前记录的 header 中添加一个标记为 （deleted flag= true），用来标识当前记录被删除，在进行查询的时候如果检查到 deleted flag 为true的情况下，表示当前记录已经被删除，则不返回数据

###### 更新（UPDATE）

InnoDB为插入一行新记录，保存当前系统版本号作为行版本号，同时保存当前系统版本号到原来的行作为行删除标识。

#### 实际应用

##### 问题

下面两条简单的SQL，他们加什么锁？

```sql
select * from t1 where id = 10;
delete from t1 where id = 10;
```

SQL1：不加锁。因为MySQL是使用多版本并发控制的，读不加锁。
SQL2：对id = 10的记录加写锁 (走主键索引)。

要回答这个问题，缺少哪些前提条件

1. id列是不是主键？
2. 当前系统的隔离级别是什么？
3. id列如果不是主键，那么id列上有索引吗？
4. id列上如果有二级索引，那么这个索引是唯一索引吗？
5. 两个SQL的执行计划是什么？索引扫描？全表扫描？

没有这些前提，直接就给定一条SQL，然后问这个SQL会加什么锁，都是很业余的表现。而当这些问题有了明确的答案之后，给定的SQL会加什么锁，也就一目了然。

##### 分析

1. id列是主键，RC隔离级别
2. id列是二级唯一索引，RC隔离级别
3. id列是二级非唯一索引，RC隔离级别
4. id列上没有索引，RC隔离级别
5. id列是主键，RR隔离级别
6. id列是二级唯一索引，RR隔离级别
7. id列是二级非唯一索引，RR隔离级别
8. id列上没有索引，RR隔离级别
9. Serializable隔离级别

##### 答案（厉害了）

1. 组合一：id主键+RC

   id是主键，Read Committed隔离级别，给定SQL：delete from t1 where id = 10; 只需要将主键上，id = 10的记录加上X锁即可。

   ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190114161101816.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzg0Mzc5Nw==,size_16,color_FFFFFF,t_70)

2. 组合二：id唯一索引+RC

   这个组合，id不是主键，而是一个Unique的二级索引键值。那么在RC隔离级别下，delete from t1 where id = 10; 需要加什么锁呢？见下图：

   ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190114161125686.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzg0Mzc5Nw==,size_16,color_FFFFFF,t_70)

   此组合中，id是unique索引，而主键是name列。此时，加锁的情况由于组合一有所不同。由于id是unique索引，因此delete语句会选择走id列的索引进行where条件的过滤，在找到id=10的记录后，首先会将unique索引上的id=10索引记录加上X锁，同时，会根据读取到的name列，回主键索引(聚簇索引)，然后将聚簇索引上的name = ‘d’ 对应的主键索引项加X锁。为什么聚簇索引上的记录也要加锁？试想一下，如果并发的一个SQL，是通过主键索引来更新：update t1 set id = 100 where name = ‘d’; 此时，如果delete语句没有将主键索引上的记录加锁，那么并发的update就会感知不到delete语句的存在，违背了同一记录上的更新/删除需要串行执行的约束。

   结论：若id列是unique列，其上有unique索引。那么SQL需要加两个X锁，一个对应于id unique索引上的id = 10的记录，另一把锁对应于聚簇索引上的[name=‘d’,id=10]的记录。

3. 组合三：id非唯一索引+RC

   相对于组合一、二，组合三又发生了变化，隔离级别仍旧是RC不变，但是id列上的约束又降低了，id列不再唯一，只有一个普通的索引。假设delete from t1 where id = 10; 语句，仍旧选择id列上的索引进行过滤where条件，那么此时会持有哪些锁？同样见下图：

   ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190114161146784.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzg0Mzc5Nw==,size_16,color_FFFFFF,t_70)

   根据此图，可以看到，首先，id列索引上，满足id = 10查询条件的记录，均已加锁。同时，这些记录对应的主键索引上的记录也都加上了锁。与组合二唯一的区别在于，组合二最多只有一个满足等值查询的记录，而组合三会将所有满足查询条件的记录都加锁。

   结论：若id列上有非唯一索引，那么对应的所有满足SQL查询条件的记录，都会被加锁。同时，这些记录在主键索引上的记录，也会被加锁。

4. 组合四：id无索引+RC

   相对于前面三个组合，这是一个比较特殊的情况。id列上没有索引，where id = 10;这个过滤条件，没法通过索引进行过滤，那么只能走全表扫描做过滤。对应于这个组合，SQL会加什么锁？或者是换句话说，全表扫描时，会加什么锁？这个答案也有很多：有人说会在表上加X锁；有人说会将聚簇索引上，选择出来的id = 10;的记录加上X锁。那么实际情况呢？请看下图：
   ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190114161207944.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzg0Mzc5Nw==,size_16,color_FFFFFF,t_70)

   由于id列上没有索引，因此只能走聚簇索引，进行全部扫描。从图中可以看到，满足删除条件的记录有两条，但是，聚簇索引上所有的记录，都被加上了X锁。无论记录是否满足条件，全部被加上X锁。既不是加表锁，也不是在满足条件的记录上加行锁。

   有人可能会问？为什么不是只在满足条件的记录上加锁呢？这是由于MySQL的实现决定的。如果一个条件无法通过索引快速过滤，那么存储引擎层面就会将所有记录加锁后返回，然后由MySQL Server层进行过滤。因此也就把所有的记录，都锁上了。

   注：在实际的实现中，MySQL有一些改进，在MySQL Server过滤条件，发现不满足后，会调用unlock_row方法，把不满足条件的记录放锁 (违背了2PL的约束)。这样做，保证了最后只会持有满足条件记录上的锁，但是每条记录的加锁操作还是不能省略的。

   结论：若id列上没有索引，SQL会走聚簇索引的全扫描进行过滤，由于过滤是由MySQL Server层面进行的。因此每条记录，无论是否满足条件，都会被加上X锁。但是，为了效率考量，MySQL做了优化，对于不满足条件的记录，会在判断后放锁，最终持有的，是满足条件的记录上的锁，但是不满足条件的记录上的加锁/放锁动作不会省略。同时，优化也违背了2PL的约束。

5. id主键+RR

   上面的四个组合，都是在Read Committed隔离级别下的加锁行为，接下来的四个组合，是在Repeatable Read隔离级别下的加锁行为。

   组合五，id列是主键列，Repeatable Read隔离级别，针对delete from t1 where id = 10; 这条SQL，加锁与组合一：[id主键，Read Committed]一致。

6. 组合六：id唯一索引+RR

   与组合五类似，组合六的加锁，与组合二：[id唯一索引，Read Committed]一致。两个X锁，id唯一索引满足条件的记录上一个，对应的聚簇索引上的记录一个。

7. id非唯一索引+RR

   RC隔离级别允许幻读，而RR隔离级别，不允许存在幻读。但是在组合五、组合六中，加锁行为又是与RC下的加锁行为完全一致。那么RR隔离级别下，如何防止幻读呢？

   组合七，Repeatable Read隔离级别，id上有一个非唯一索引，执行delete from t1 where id = 10; 假设选择id列上的索引进行条件过滤，最后的加锁行为，是怎么样的呢？同样看下面这幅图：

   ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190114161237655.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzg0Mzc5Nw==,size_16,color_FFFFFF,t_70)

   此图，相对于组合三：[id列上非唯一锁，Read Committed]看似相同，其实却有很大的区别。最大的区别在于，这幅图中多了一个GAP锁，而且GAP锁看起来也不是加在记录上的，倒像是加载两条记录之间的位置，GAP锁有何用？

   其实这个多出来的GAP锁，就是RR隔离级别，相对于RC隔离级别，不会出现幻读的关键。确实，GAP锁锁住的位置，也不是记录本身，而是两条记录之间的GAP。所谓幻读，就是同一个事务，连续做两次当前读 (例如：select * from t1 where id = 10 for update;)，那么这两次当前读返回的是完全相同的记录 (记录数量一致，记录本身也一致)，第二次的当前读，不会比第一次返回更多的记录 (幻象)。

   如何保证两次当前读返回一致的记录，那就需要在第一次当前读与第二次当前读之间，其他的事务不会插入新的满足条件的记录并提交。为了实现这个功能，GAP锁应运而生。

   如图中所示，有哪些位置可以插入新的满足条件的项 (id = 10)，考虑到B+树索引的有序性，满足条件的项一定是连续存放的。记录[6,c]之前，不会插入id=10的记录；[6,c]与[10,b]间可以插入[10, aa]；[10,b]与[10,d]间，可以插入新的[10,bb],[10,c]等；[10,d]与[11,f]间可以插入满足条件的[10,e],[10,z]等；而[11,f]之后也不会插入满足条件的记录。因此，为了保证[6,c]与[10,b]间，[10,b]与[10,d]间，[10,d]与[11,f]不会插入新的满足条件的记录，MySQL选择了用GAP锁，将这三个GAP给锁起来。

   Insert操作，如insert [10,aa]，首先会定位到[6,c]与[10,b]间，然后在插入前，会检查这个GAP是否已经被锁上，如果被锁上，则Insert不能插入记录。因此，通过第一遍的当前读，不仅将满足条件的记录锁上 (X锁)，与组合三类似。同时还是增加3把GAP锁，将可能插入满足条件记录的3个GAP给锁上，保证后续的Insert不能插入新的id=10的记录，也就杜绝了同一事务的第二次当前读，出现幻象的情况。

   有心的朋友看到这儿，可以会问：既然防止幻读，需要靠GAP锁的保护，为什么组合五、组合六，也是RR隔离级别，却不需要加GAP锁呢？

   首先，这是一个好问题。其次，回答这个问题，也很简单。GAP锁的目的，是为了防止同一事务的两次当前读，出现幻读的情况。而组合五，id是主键；组合六，id是unique键，都能够保证唯一性。一个等值查询，最多只能返回一条记录，而且新的相同取值的记录，一定不会在新插入进来，因此也就避免了GAP锁的使用。其实，针对此问题，还有一个更深入的问题：如果组合五、组合六下，针对SQL：select * from t1 where id = 10 for update; 第一次查询，没有找到满足查询条件的记录，那么GAP锁是否还能够省略？此问题留给大家思考。

   结论：Repeatable Read隔离级别下，id列上有一个非唯一索引，对应SQL：delete from t1 where id = 10; 首先，通过id索引定位到第一条满足查询条件的记录，加记录上的X锁，加GAP上的GAP锁，然后加主键聚簇索引上的记录X锁，然后返回；然后读取下一条，重复进行。直至进行到第一条不满足条件的记录[11,f]，此时，不需要加记录X锁，但是仍旧需要加GAP锁，最后返回结束。

8. id无索引+RR

   组合八，Repeatable Read隔离级别下的最后一种情况，id列上没有索引。此时SQL：delete from t1 where id = 10; 没有其他的路径可以选择，只能进行全表扫描。最终的加锁情况，如下图所示：

   ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190114161259587.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzg0Mzc5Nw==,size_16,color_FFFFFF,t_70)

   如图，这是一个很恐怖的现象。首先，聚簇索引上的所有记录，都被加上了X锁。其次，聚簇索引每条记录间的间隙(GAP)，也同时被加上了GAP锁。这个示例表，只有6条记录，一共需要6个记录锁，7个GAP锁。试想，如果表上有1000万条记录呢？

   在这种情况下，这个表上，除了不加锁的快照度，其他任何加锁的并发SQL，均不能执行，不能更新，不能删除，不能插入，全表被锁死。

   当然，跟组合四：[id无索引, Read Committed]类似，这个情况下，MySQL也做了一些优化，就是所谓的semi-consistent read。semi-consistent read开启的情况下，对于不满足查询条件的记录，MySQL会提前放锁。针对上面的这个用例，就是除了记录[d,10]，[g,10]之外，所有的记录锁都会被释放，同时不加GAP锁。semi-consistent read如何触发：要么是read committed隔离级别；要么是Repeatable Read隔离级别，同时设置了innodb_locks_unsafe_for_binlog 参数。更详细的关于semi-consistent read的介绍，可参考我之前的一篇博客：MySQL+InnoDB semi-consitent read原理及实现分析 。

   结论：在Repeatable Read隔离级别下，如果进行全表扫描的当前读，那么会锁上表中的所有记录，同时会锁上聚簇索引内的所有GAP，杜绝所有的并发 更新/删除/插入 操作。当然，也可以通过触发semi-consistent read，来缓解加锁开销与并发影响，但是semi-consistent read本身也会带来其他问题，不建议使用。

9. Serializable

   针对前面提到的简单的SQL，最后一个情况：Serializable隔离级别。对于SQL2：delete from t1 where id = 10; 来说，Serializable隔离级别与Repeatable Read隔离级别完全一致，因此不做介绍。

   Serializable隔离级别，影响的是SQL1：select * from t1 where id = 10; 这条SQL，在RC，RR隔离级别下，都是快照读，不加锁。但是在Serializable隔离级别，SQL1会加读锁，也就是说快照读不复存在，MVCC并发控制降级为Lock-Based CC。

   结论：在MySQL/InnoDB中，所谓的读不加锁，并不适用于所有的情况，而是隔离级别相关的。Serializable隔离级别，读不加锁就不再成立，所有的读操作，都是当前读。





MVCC判断了记录的可见性，比如 **select count(\*) from table where col_name = xxx** 时(属于快照读)，在RR 级别下，这条事务在事务一开始就生成了readview，通过这个readview 这条语句将会找到符合条件的行并且计算数量。 那么关于与如何找到这些符合条件的行，满足where 条件的**同时也得满足本事务对这些行的可见性**。 所以在同一事务里并不会产生幻读的现象、





### RR 下解决幻读

所谓幻读，就是同一个事务，连续做两次当前读 (例如：select * from t1 where id = 10 for update;)，那么这两次当前读返回的是完全相同的记录 (记录数量一致，记录本身也一致)，第二次的当前读，不会比第一次返回更多的记录 (幻象)。



https://github.com/Yhzhtk/note/issues/42

MVCC是实现的是快照读，next-key locking 是对当前读 都可以避免幻读。

RR 下，innodb下的幻读是由MVCC 或者 GAP 锁 或者是next-key lock 解决的。

https://www.cnblogs.com/aspirant/p/9177978.html

## 秒杀系统实现

秒杀要解决的是，高并发流量和数据库操作瓶颈的问题。

### 秒杀的本质

**秒杀的本质，就是对库存的抢夺**，每个秒杀的用户来你都去数据库查询库存校验库存，然后扣减库存，撇开性能因数，你不觉得这样好繁琐，对业务开发人员都不友好，而且数据库顶不住啊。

我们都知道数据库顶不住但是他的兄弟非关系型的数据库**Redis**能顶啊！

那不简单了，我们要开始秒杀前你通过定时任务或者运维同学**提前把商品的库存加载到Redis中**去，让整个流程都在Redis里面去做，然后等秒杀介绍了，再异步的去修改库存就好了。

但是用了Redis就有一个问题了，我们上面说了我们采用**主从**，就是我们会去读取库存然后再判断然后有库存才去减库存，正常情况没问题，但是高并发的情况问题就很大了。

这里我就不画图了，我本来想画图的，想了半天我觉得语言可能更好表达一点。

**多品几遍！！！**就比如现在库存只剩下1个了，我们高并发嘛，4个服务器一起查询了发现都是还有1个，那大家都觉得是自己抢到了，就都去扣库存，那结果就变成了-3，是的只有一个是真的抢到了，别的都是超卖的。咋办？



> **Lua** 脚本功能是 Reids在 2.6 版本的最大亮点， 通过内嵌对 Lua 环境的支持， Redis 解决了长久以来不能高效地处理 **CAS** （check-and-set）命令的缺点， 并且可以通过组合使用多个命令， 轻松实现以前很难实现或者不能高效实现的模式。
>  

**Lua脚本是类似Redis事务，有一定的原子性，不会被其他命令插队，可以完成一些Redis事务性的操作。**这点是关键。

知道原理了，我们就写一个脚本把判断库存扣减库存的操作都写在一个脚本丢给Redis去做，那到0了后面的都Return False了是吧，一个失败了你修改一个开关，直接挡住所有的请求，然后再做后面的事情嘛。



作者：敖丙
链接：https://www.zhihu.com/question/54895548/answer/923987542
来源：知乎
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。



### 基于Mysql实现：悲观锁

这效率肯定不行吧

按照正常的购买流程：查询商品库存，库存大于0时，生成订单，去库存。如果出现并发，导致在查询商品库存的时候，库存会一直出现大于0的情况，出现超卖现象。

基于mysql的事务和锁实现方式：

- 1：开启事务

- 2：查询库存，并显示的设置写锁（排他锁）：SELECT * FROM table_name WHERE … FOR UPDATE

- 3：生成订单

- 4：去库存，隐示的设置写锁（排他锁）：UPDATE goods SET counts = counts – 1 WHERE id = 1

  udpate stock set count = count - 1 where product_id = xxxxx and count > 0

- 5：commit，释放锁

注意：

如果不开启事务，第二步即使加锁，第一个会话读库存结束后，变会释放锁，第二个会话仍有机会在去库存前读库存，出现超卖。

如果开启事务，第二步不加锁，第一个会话读库存结束后，第二个会话容易出现【脏读】，出现超卖。

即加事务，又加读锁：开启事务，第一个会话读库存时加读锁，并发时，第二个会话也允许获得读库存的读锁，但是在第一个会话执行写操作时，写锁便会等待第二个会话的读锁，第二个会话执行写操作时，写锁便会等待第一个会话的读锁，出现死锁

即加事务，又加写锁：第一个会话读库存时加写锁，写锁会阻止其它事务的读锁和写锁。直到commit才会释放，允许第二个会话查询库存，不会出现超卖现象。

缺点:

 1) 会占用大量线程资源，因为一个线程更新不成功，其他线程全部等待。比如说Mysql的线程资源全部耗尽，从前端客户的角度看起来是一直在等待。

2) 更新会很慢。
使用场景：在流量不大，对数据一致性要求又非常高的时候，可以用悲观锁。

`悲观锁`是一种悲观思想，它总认为最坏的情况可能会出现，它认为数据很可能会被其他人所修改，所以悲观锁在持有数据的时候总会把`资源` 或者 `数据` 锁住，这样其他线程想要请求这个资源的时候就会阻塞，直到等到悲观锁把资源释放为止。传统的关系型数据库里边就用到了很多这种锁机制，**比如行锁，表锁等，读锁，写锁等，都是在做操作之前先上锁**。悲观锁的实现往往依靠数据库本身的锁功能实现。

一般来说，悲观锁不仅会对写操作加锁还会对读操作加锁，一个典型的悲观锁调用：

```
select * from student where name="cxuan" for update
```

这条 sql 语句从 Student 表中选取 name = "cxuan" 的记录并对其加锁，那么其他写操作再这个事务提交之前都不会对这条数据进行操作，起到了独占和排他的作用。

> 详见数据库的行锁，表锁等

独占锁(排他锁)就是一种悲观锁思想的实现，不管是否持有资源，它都会尝试去加锁，生怕自己心爱的宝贝被别人拿走。

### 基于Mysql实现：乐观锁

注意这里更新是要结合版本号。如果结合版本号查询有库存，则扣库存， 否则说明版本号被更新。


SQL实现：
查version字段，如果相等，说明可以更新，同时把version+1，否则说明更新失败(因为其他线程更新过了)。

udpate stock set count = count - 1 and version = verison + 1 where product_id = xxxxx and count > 0 and version = xxxxx

> 确实流弊,因为version变成version + 1了所以where version = xxx 条件不生效了

一个更新失败的例子：

1. select version 1

2. do some logic. At this time, another thread change the verison to 2

3. update stock version param 1 //发现失败了，则更新version to 1，再重新来过。 

   > 即更新失败了，再次获取version，从走logic...

使用场景：如果只要保证数据最终一致性，可以用乐观锁。
秒杀场景不适合乐观锁，因为verion字段会一直变! 后台会一直忙于更新，大量线程hold住。

### 基于Redis

由于MySQL 数据库单点能支撑 1000 QPS，但是 Redis 单点能支撑 10万 QPS。所以解决方案是将库存信息加载到Redis中，将MySQL的访问压力转移到Redis上，直接通过 Redis 来判断并扣减库存。

**防止超卖的解决方案是：将库存信息加载到Redis中，将MySQL的访问压力转移到Redis上，直接通过 Redis 来判断并扣减库存。**

![秒杀](D:\worker\h3c\数据库&&mysql\images\秒杀.jpg)



表设计

```bash
### Activty活动      《------》    ## Stock库存
ID：int                           ID： int
活动编码： int                     商品编码： varchar
活动名：varchart                   商品名称： varchar
商品变化：varchar                  活动编码： int
活动类型 int                       总库存数量： bigint
                                  锁定库存数量：bigint
                                  
## Produc商品                      #Order订单
ID： int                           ID： int
商品编码： varchar                  订单编码： varchar
商品名称： varchar                  商品编码： varchar
商品标题：                          订单金额： 
商品描述：                          下单时间
								用户编码
								购买数量
```



几个关于秒杀系统中Redis的常见问题：

**1.什么时候把库存写入到Redis？**

秒杀活动创建/维护时写入Redis。

**2.如何保证活动数据库和库存数据一致？**

可以使用分布式事务或消息队列。

**分布式事务：**保证多个数据库的操作同时成功或者同时失败。对强一致性有要求的业务场景可以考虑使用分布式事务，比如银行转账

**消息队列：**基于生产者/消费者模型的组件，一般实现异步任务（非实时处理）时会引入消息队列。消息队列的好处是任务可以慢慢处理，不必同步处理等着响应结果。目前主流的消息队列有RocketMQ、Kafka等。使用场景除了异步任务之外，一般还用于失败的情况下重试处理，重复消费直到消费成功。

**3.下单减库存/支付减库存？**

下单锁定库存，支付减库存。

**4.如何防止商品被超卖？**

把库存数据放入到缓存中，利用缓存的原子特性保证同时只有一个线程操作库存。

**5.库存写回数据库的时机？**

采用定时任务同步Redis的数据写回数据库。

最后，**4S分析法的第四步，Scale扩展**。



### 基于redis + mq

我们公司运营活动 领/抽奖，奖品信息在活动上线就全部初始化到redis。然后ssd做持久化。不用数据库。中奖记录mq发送。

### 其他限流



先把没预约活动的都踢掉，直接返回秒杀失败，再从预约里随机踢出一半，最后剩下的再根据各种需求（比如消费额度等）再踢一批用户，最后剩下的放进队列…

为什么要真的接收这些数据？直接随你选择接收约30个请求，然后30个抽10个。其他全部不接收，页面显示正在秒杀。反正本来就是靠运气，靠算法随机和靠运气随机不一样吗？而且你这算法还是伪随机哈哈

哈哈哈，这样不能按请求时间排序了

通过悲观锁和乐观锁来解决秒杀系统中的超卖问题，当然这两个方法是解决改问题的核心，我们还可以用一些额外的方法来优化请求数量，减少并发，环节服务端的压力，例如使用**令牌桶算法限流**，**redis缓存限时**，**消息队列存储成功请求**，加快处理时间等

### 削峰填谷

削峰填谷本身是电力行业的概念，电力企业通过必要的技术和管理手段，降低电网的高峰负荷，提高低谷负荷，平滑负荷曲线，提高负荷率，保证电网的稳定运行。

假设一个应用，它能够每秒处理 1000 个请求。如果在第一秒接收到 2000 个请求，而接下来的两秒都没有请求到达。

[![img](https://s5.51cto.com/oss/202008/13/244e3ada32358f6cfe6326ba9deaa334.png-wh_600x-s_3461813098.png)](https://s5.51cto.com/oss/202008/13/244e3ada32358f6cfe6326ba9deaa334.png-wh_600x-s_3461813098.png)

整个应用必然面临两个问题：

(1)在第一秒被 2000 个请求直接压垮;

(2)假设第一秒没有被压垮，它在这一秒时间内只能处理 1000 请求，第二第三秒却完全空闲，浪费了系统资源。

所以，我们可以通过 MQ 把请求突刺均摊到一段时间内，让系统负载保持在请求处理水位之内，同时尽可能地处理更多请求，从而起到“削峰填谷”的效果。

红色的部分是超出系统处理能力的部分，可以把红色的那部分消息平摊到后面空闲时去处理，这样既可以保证系统负载处在一个稳定的水位，又可以尽可能地处理更多消息。通过配置流控规则，可以达到消息匀速处理的效果。



## 锁问题排查

模拟问题：

1. 两个session的事务隔离级别都设置成serializable...

2. 操作同一个表，S1执行select * from, S2执行update 语句

3. 查看锁的状态

   获取当前会话的 trx

   `SELECT TRX_ID FROM INFORMATION_SCHEMA.INNODB_TRX  WHERE TRX_MYSQL_THREAD_ID = CONNECTION_ID();`

   查看哪个表被锁

   `show OPEN TABLES where In_use > 0;`

   查看正在锁的事务

   `SELECT * FROM INFORMATION_SCHEMA.INNODB_LOCKS;`

   查看等待锁的事务

   `SELECT * FROM INFORMATION_SCHEMA.INNODB_LOCK_WAITS;`

```bash
mysql> show engine innodb status\G
*************************** 1. row ***************************
  Type: InnoDB
  Name:
Status:
=====================================
2021-07-21 12:02:06 7f496fcf5700 INNODB MONITOR OUTPUT
=====================================
Per second averages calculated from the last 12 seconds
-----------------
BACKGROUND THREAD
-----------------
srv_master_thread loops: 20 srv_active, 0 srv_shutdown, 186060 srv_idle
srv_master_thread log flush and writes: 186076
----------
SEMAPHORES
----------
OS WAIT ARRAY INFO: reservation count 29
OS WAIT ARRAY INFO: signal count 30
Mutex spin waits 9, rounds 62, OS waits 2
RW-shared spins 25, rounds 810, OS waits 27
RW-excl spins 0, rounds 15, OS waits 0
Spin rounds per wait: 6.89 mutex, 32.40 RW-shared, 15.00 RW-excl
------------
TRANSACTIONS
------------
Trx id counter 2390
Purge done for trx's n:o < 2385 undo n:o < 0 state: running but idle
History list length 21
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 2389, ACTIVE 46 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 360, 1 row lock(s)
MySQL thread id 26, OS thread handle 0x7f496fcb3700, query id 394 localhost root updating
update account set balance=balance+10 where id=1
## 这里说了，这个事务transaction 已经被锁了 46 秒了
------- TRX HAS BEEN WAITING 46 SEC FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 8 page no 3 n bits 72 index `PRIMARY` of table `test_db`.`account` trx id 2389 lock_mode X locks rec but not gap waiting
Record lock, heap no 2 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 4; hex 80000001; asc     ;;
 1: len 6; hex 00000000093f; asc      ?;;
 2: len 7; hex 29000001560110; asc )   V  ;;
 3: len 1; hex 61; asc a;;
 4: len 4; hex 80000052; asc    R;;

------------------
---TRANSACTION 2388, ACTIVE 91 sec
2 lock struct(s), heap size 360, 5 row lock(s)
MySQL thread id 24, OS thread handle 0x7f496fcf5700, query id 398 localhost root init
show engine innodb status
--------
FILE I/O
--------
I/O thread 0 state: waiting for completed aio requests (insert buffer thread)
...

## --
mysql> mysql> SELECT * FROM INFORMATION_SCHEMA.INNODB_LOCKS;
+------------+-------------+-----------+-----------+---------------------+------------+------------+-----------+----------+-----------+
| lock_id    | lock_trx_id | lock_mode | lock_type | lock_table          | lock_index | lock_space | lock_page | lock_rec | lock_data |
+------------+-------------+-----------+-----------+---------------------+------------+------------+-----------+----------+-----------+
| 2389:8:3:2 | 2389        | X         | RECORD    | `test_db`.`account` | PRIMARY    |          8 |         3 |        2 | 1         |
| 2388:8:3:2 | 2388        | S         | RECORD    | `test_db`.`account` | PRIMARY    |          8 |         3 |        2 | 1         |
+------------+-------------+-----------+-----------+---------------------+------------+------------+-----------+----------+-----------+
2 rows in set (0.00 sec)


## 两个x锁

mysql> select @@tx_isolation;                                                                   +-----------------+
| @@tx_isolation  |
+-----------------+
| REPEATABLE-READ |
+-----------------+
1 row in set (0.01 sec)

mysql> use test_db;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> update account set balance=balance+10 where id=1;
Query OK, 1 row affected (0.01 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> SELECT * FROM INFORMATION_SCHEMA.INNODB_LOCKS;
+------------+-------------+-----------+-----------+---------------------+------------+------------+-----------+----------+-----------+
| lock_id    | lock_trx_id | lock_mode | lock_type | lock_table          | lock_index | lock_space | lock_page | lock_rec | lock_data |
+------------+-------------+-----------+-----------+---------------------+------------+------------+-----------+----------+-----------+
| 2391:8:3:2 | 2391        | X         | RECORD    | `test_db`.`account` | PRIMARY    |          8 |         3 |        2 | 1         |
| 2390:8:3:2 | 2390        | X         | RECORD    | `test_db`.`account` | PRIMARY    |          8 |         3 |        2 | 1         |
+------------+-------------+-----------+-----------+---------------------+------------+------------+-----------+----------+-----------+
2 rows in set (0.01 sec)

mysql>


## console 2
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> update account set balance=balance+10 where id=1;
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
mysql> update account set balance=balance+10 where id=1;
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
mysql>


## 行锁
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from account where id=1 for update;
+----+------+---------+
| id | name | balance |
+----+------+---------+
|  1 | a    |     102 |
+----+------+---------+
1 row in set (0.00 sec)

mysql> SELECT * FROM INFORMATION_SCHEMA.INNODB_LOCKS;
Empty set (0.00 sec)

mysql> SELECT * FROM INFORMATION_SCHEMA.INNODB_LOCKS;
+------------+-------------+-----------+-----------+---------------------+------------+------------+-----------+----------+-----------+
| lock_id    | lock_trx_id | lock_mode | lock_type | lock_table          | lock_index | lock_space | lock_page | lock_rec | lock_data |
+------------+-------------+-----------+-----------+---------------------+------------+------------+-----------+----------+-----------+
| 2395:8:3:2 | 2395        | X         | RECORD    | `test_db`.`account` | PRIMARY    |          8 |         3 |        2 | 1         |
| 2394:8:3:2 | 2394        | X         | RECORD    | `test_db`.`account` | PRIMARY    |          8 |         3 |        2 | 1         |
+------------+-------------+-----------+-----------+---------------------+------------+------------+-----------+----------+-----------+
2 rows in set (0.00 sec)

mysql> SELECT * FROM INFORMATION_SCHEMA.INNODB_LOCKS;
+------------+-------------+-----------+-----------+---------------------+------------+------------+-----------+----------+-----------+
| lock_id    | lock_trx_id | lock_mode | lock_type | lock_table          | lock_index | lock_space | lock_page | lock_rec | lock_data |
+------------+-------------+-----------+-----------+---------------------+------------+------------+-----------+----------+-----------+
| 2395:8:3:2 | 2395        | X         | RECORD    | `test_db`.`account` | PRIMARY    |          8 |         3 |        2 | 1         |
| 2394:8:3:2 | 2394        | X         | RECORD    | `test_db`.`account` | PRIMARY    |          8 |         3 |        2 | 1         |
+------------+-------------+-----------+-----------+---------------------+------------+------------+-----------+----------+-----------+
2 rows in set (0.00 sec)

mysql> SELECT * FROM INFORMATION_SCHEMA.INNODB_LOCK_WAITS;
+-------------------+-------------------+-----------------+------------------+
| requesting_trx_id | requested_lock_id | blocking_trx_id | blocking_lock_id |
+-------------------+-------------------+-----------------+------------------+
| 2395              | 2395:8:3:2        | 2394            | 2394:8:3:2       |
+-------------------+-------------------+-----------------+------------------+
1 row in set (0.00 sec)

mysql> SELECT TRX_ID FROM INFORMATION_SCHEMA.INNODB_TRX  WHERE TRX_MYSQL_THREAD_ID = CONNECTION_ID();
+--------+
| TRX_ID |
+--------+
| 2394   |
+--------+
1 row in set (0.00 sec)

mysql>

```

end





## 事务日志

https://blog.csdn.net/qq_35246620/article/details/79345359

redo and undo



## 乐观锁：主流



乐观锁的思想与悲观锁的思想相反，它总认为资源和数据不会被别人所修改，所以读取不会上锁，但是乐观锁在进行写入操作的时候会判断当前数据是否被修改过。乐观锁的实现方案一般来说有两种：`版本号机制` 和 `CAS实现` 。乐观锁多适用于多读的应用类型，这样可以提高吞吐量。

乐观锁用于读多写少的情况，即很少发生冲突的场景，这样可以省去锁的开销，增加系统的吞吐量。

乐观锁的适用场景有很多，典型的比如说成本系统，柜员要对一笔金额做修改，为了保证数据的准确性和实效性，使用悲观锁锁住某个数据后，再遇到其他需要修改数据的操作，那么此操作就无法完成金额的修改，对产品来说是灾难性的一刻，使用乐观锁的版本号机制能够解决这个问题

> etcd不就是基于版本号机制的乐观锁？？？

### 版本号机制

版本号机制是在数据表中加上一个 `version` 字段来实现的，表示数据被修改的次数，当执行写操作并且写入成功后，version = version + 1，当线程A要更新数据时，在读取数据的同时也会读取 version 值，在提交更新时，若刚才读取到的 version 值为当前数据库中的version值相等时才更新，否则重试更新操作，直到更新成功。

乐观锁在进行写操作的时候会判断是否能够写入成功，如果写入不成功将触发等待 -> 重试机制，这种情况是一个自旋锁，简单来说就是适用于短期内获取不到，进行等待重试的锁，它不适用于长期获取不到锁的情况，另外，自旋循环对于性能开销比较大。

### CAS

CAS的一般操作

```bash
bool compare_and_swap (int *oldval, int *dest, int newval) {
  if (*oldval == *dest) {
      *dest = newval;
      return true;
  }
  return false;
}
```

CAS 即 `compare and swap（比较与交换）`，是一种有名的无锁算法。即不使用锁的情况下实现多线程之间的变量同步，也就是在没有线程被阻塞的情况下实现变量的同步，所以也叫非阻塞同步（Non-blocking Synchronization)

CAS：对于内存中的某一个值V，提供一个旧值A和一个新值B。如果提供的旧值V和A相等就把B写入V。这个过程是原子性的。
CAS执行结果要么成功要么失败，对于失败的情形下一班采用不断重试。或者放弃。

ABA：如果另一个线程修改V值假设原来是A，先修改成B，再修改回成A。当前线程的CAS操作无法分辨当前V值是否发生过变化。

关于ABA问题我想了一个例子：在你非常渴的情况下你发现一个盛满水的杯子，你一饮而尽。之后再给杯子里重新倒满水。然后你离开，当杯子的真正主人回来时看到杯子还是盛满水，他当然不知道是否被人喝完重新倒满。解决这个问题的方案的一个策略是每一次倒水假设有一个自动记录仪记录下，这样主人回来就可以分辨在她离开后是否发生过重新倒满的情况。这也是解决ABA问题目前采用的策略。

> 我的理解是ABA本质是可循环利用的指针导致数据错乱和内存回收无关,是因为数据错乱了而立马主动回收pop后的A导致有可能多线程竞争的其他pop会出现非法访问已经释放的A指针

作者：寻寒


CAS 中涉及三个要素：

- 需要读写的内存值 V
- 进行比较的值 A
- 拟写入的新值 B

当且仅当预期值A和内存值V相同时，将内存值V修改为B，否则什么都不做。

CAS有三个操作参数：内存地址，期望值，要修改的新值，当期望值和内存当中的值进行比较不相等的时候，表示内存中的值已经被别线程改动过，这时候失败返回，只有相等时，才会将内存中的值改为新的值，并返回成功。

### ABA 问题

通俗来讲就是你大爷还是你大爷，你大妈已经不是你大妈了^_^ 

举个例子 

A --> B --> C

假设你有个用单链表实现的栈，如上面所示，有个head指针指向栈顶的A 用CAS原子操作，你可能会这样实现push和pop

```c
push(node):
    curr := head
    old := curr
    node->next = curr
    // CAS返回旧值，old是期待的值，应该要用!=来比较
    while (old != (curr = CAS(&head, curr, node))) {
        old = curr
        node->next = curr
    }

pop(node):
    curr := head
    old := curr
    node->next = curr
    while (old != (curr = CAS(&head, curr, node))) {
        old = curr
        node = curr->next
    }
```

ABA的问题在于，pop函数中，next = curr->next 和 while之间，线程被切换走，然后其他线程先把A弹出，又把B弹出，然后又把A压入，**栈变成 了A --> C**，此时head还是指向A，等pop被切换回来继续执行，就把head指向B了。 

> "pop函数中，next = curr->next 和 while之间，线程被切换走，然后其他线程先把A弹出" head在跑第一次while的时候已经指向了B，其他程序再pop的时候就没有“先弹出A”的情况了 ????
>
> ???

因此ABA问题的本质是内存回收的问题，对于上面的例子，就是A被弹出后，需要保证它的内存不能立即释放（因为还有线程引用它），也就不能立即被重用。这是新手使用CAS最常见的坑，实际项目中，通常配合128位CAS、引用计数、序列号或者HazardPointer等技术来避免ABA问题。



ABA问题是指在CAS操作时，其他线程将变量值A改为了B，但是又被改回了A，等到本线程使用期望值A与当前变量进行比较时，发现变量A没有变，于是CAS就将A值进行了交换操作，但是实际上该值已经被其他线程改变过，这与乐观锁的设计思想不符合。ABA问题的解决思路是，每次变量更新的时候把变量的版本号加1，那么A-B-A就会变成A1-B2-A3，只要变量被某一线程修改过，改变量对应的版本号就会发生递增变化，从而解决了ABA问题。

### ABA的危害

下面是一段伪代码，将就着看一下。场景是用链表来实现一个栈，初始化向栈中压入B、A两个元素，栈顶head指向A元素。

在某个时刻，线程1试图将栈顶换成B，但它获取栈顶的oldValue（为A）后，被线程2中断了。线程2依次将A、B弹出，然后压入C、D、A。然后换线程1继续运行，线程1执行compareAndSet发现head指向的元素确实与oldValue一致，都是A，所以就将head指向B了。但是，注意我标黄的那行代码，线程2在弹出B的时候，将B的next置为null了，因此在线程1将head指向B后，栈中只剩了一个孤零零的元素B。但按预期来说，栈中应该放的是B → A → D → C。

```
Node head;
head = B;
A.next = head;
head = A;


Thread thread1 = new Thread(
    ->{
          oldValue = head;
          sleep(3秒);
          compareAndSet(oldValue, B);

    }
);

Thread thread2 = new Thread(
    ->{
        // 弹出A
          newHead = head.next;
          head.next = null; //即A.next = null;
          head = newHead;
         // 弹出B
          newHead = head.next;
          head.next = null; // 即B.next = null;
          head = newHead; // 此时head为null
          
          // 压入C
          head = C;
          // 压入D
          D.next = head;
          head = D;
          // 压入A
          A.next = D;
          head = A;
          

    }
);

thread1.start();
thread2.start();
```

end