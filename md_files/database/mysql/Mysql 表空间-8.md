## Mysql 表空间-8

https://blog.csdn.net/u010647035/article/details/105009979

敖丙这个例子，可以看完演示下

https://cloud.tencent.com/developer/article/1751646

### 参数控制

innodb-data-home-dir: 默认表空间目录

```sql
show variables like '%innodb_data_home_dir%';
+----------------------+-------------------------------------+
| Variable_name        | Value                               |
+----------------------+-------------------------------------+
| innodb_data_home_dir | /data/mysql/mysqldata3316/innodb_ts |
+----------------------+-------------------------------------+
1 row in set (0.00 sec)

```

datadir: 默认数据目录,Path to the database root 即Mysql数据文件所在目录，每个

按 databas name划分一个个目录，如果开启了`innodb-file-per-table=on`， 则每个表的表空间文件都在其所在的数据库目录下。

```sql
 show variables like "%datadir%";
+---------------+-----------------------------------+
| Variable_name | Value                             |
+---------------+-----------------------------------+
| datadir       | /data/mysql/mysqldata3316/mydata/ |
+---------------+-----------------------------------+
1 row in set (0.00 sec)

$ tree  -d /data/mysql/mysqldata3316/mydata/
/data/mysql/mysqldata3316/mydata/
├── hotdb_cloud_config
├── mysql
├── performance_schema
├── sys
├── test
└── user

6 directories
...
└── user
    ├── db.opt
    --- 表定义文件，mysql 8.0 后合并
    ├── user.frm
    --- 表空间文件，默认是96 K
    └── user.ibd

```

end

### 表空间：TableSpace

在innodb存储引擎中数据是按照表空间来组织存储的”。即，表空间是表空间文件是实际存在的物理文件。

InnoDB 表空间（Tablespace）可以看做一个逻辑概念，InnoDB 把数据保存在表空间，本质上是一个或多个磁盘文件组成的虚拟文件系统。InnoDB 表空间不仅仅存储了表和索引，它还保存了回滚日志（redo log）、插入缓冲（insert buffer）、双写缓冲（doublewrite buffer）以及其他内部数据结构。
默认情况下InnoDB存储引擎有一个共享表空间ibdata1，即所有数据都放在这个表空间内。

从 InnoDB 逻辑存储结构来看，所有的数据都被逻辑的存放在一个空间中，这个空间就叫做表空间（tablespace）。表空间有 段（segment）、区（extent）、页（page）组成。

#### 段：segment

段(Segment)分为索引段，数据段，回滚段等。其中索引段就是非叶子结点部分，而数据段就是叶子结点部分，回滚段用于数据的回滚和多版本控制。一个段包含256个区(256M大小)。

#### 区：extent

区是页的集合，一个区包含64个连续的页，默认大小为 1MB (64*16K)。

#### 页：page

页是 InnoDB 管理的最小单位，常见的有 FSP_HDR，INODE, INDEX 等类型。所有页的结构都是一样的，分为文件头(前38字节)，页数据和文件尾(后8字节)。页数据根据页的类型不同而不一样。



### 表空间分类

#### 系统表空间

默认情况下，MySQL会初始化一个大小为12MB，名为ibdata1文件，并且随着数据的增多，它会自动扩容。

这个ibdata1文件是系统表空间，也是默认的表空间，也是默认的表空间物理文件，也是传说中的共享表空间。

```bash
mysql> show variables like '%innodb_data_file_path%';
+-----------------------+------------------------+
| Variable_name         | Value                  |
+-----------------------+------------------------+
| innodb_data_file_path | ibdata1:12M:autoextend |
+-----------------------+------------------------+
1 row in set (0.00 sec)

mysql> select @@innodb_data_home_dir;
+-------------------------------------+
| @@innodb_data_home_dir              |
+-------------------------------------+
| /data/mysql/mysqldata3316/innodb_ts |
+-------------------------------------+
1 row in set (0.00 sec)


# 默认参数文件大小
root@79cea6c18bc6:/# ls /data/mysql/mysqldata3316/innodb_ts /ibdata1
/data/mysql/mysqldata3316/innodb_ts /ibdata1
root@79cea6c18bc6:/# du -sh /data/mysql/mysqldata3316/innodb_ts/ibdata1
76M     /data/mysql/mysqldata3316/innodb_ts/ibdata1


```

end

#### 独立表空间

独立表空间就是每个表单独创建一个 *.ibd* 文件，该文件存储着该表的索引和数据。由 `innodb_file_per_table` 变量控制。禁用 `innodb_file_per_table=OFF`会导致InnoDB在系统表空间中创建表。

https://blog.51cto.com/u_15230485/2821365

```bash
mysql> show variables like '%innodb_file_per%';
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| innodb_file_per_table | ON    |
+-----------------------+-------+
1 row in set (0.00 sec)


root@79cea6c18bc6:/# ls /var/lib/mysql/
auto.cnf  ib_logfile0  ib_logfile1  ibdata1  mysql  performance_schema  test_db
## test_db 是创建的数据库名字
root@79cea6c18bc6:/# ls /var/lib/mysql/test_db/
account.frm  db.opt  e4.ibd                   prediction_function.ibd  tb2.ibd   test.ibd   test2.ibd  test3.ibd  test4.ibd  tt.ibd
account.ibd  e4.frm  prediction_function.frm  tb2.frm                  test.frm  test2.frm  test3.frm  test4.frm  tt.frm

## table_name + ibd 命名规则
mysql> show tables;
+---------------------+
| Tables_in_test_db   |
+---------------------+
| account             |
| e4                  |
| prediction_function |
| tb2                 |
| test                |
| test2               |
| test3               |
| test4               |
| tt                  |
+---------------------+
9 rows in set (0.00 sec
$ ls /var/lib/mysql/test_db/*.ibd -l
-rw-rw---- 1 mysql mysql 114688 Jul 21 16:40 /var/lib/mysql/test_db/account.ibd
-rw-rw---- 1 mysql mysql 114688 Jul 22 09:13 /var/lib/mysql/test_db/e4.ibd
-rw-rw---- 1 mysql mysql 131072 Jul 19 09:38 /var/lib/mysql/test_db/prediction_function.ibd
-rw-rw---- 1 mysql mysql 114688 Jul 22 14:44 /var/lib/mysql/test_db/tb2.ibd
-rw-rw---- 1 mysql mysql 114688 Jul 22 04:06 /var/lib/mysql/test_db/test.ibd
-rw-rw---- 1 mysql mysql 131072 Jul 22 08:29 /var/lib/mysql/test_db/test2.ibd
-rw-rw---- 1 mysql mysql 114688 Jul 22 06:10 /var/lib/mysql/test_db/test3.ibd
-rw-rw---- 1 mysql mysql 131072 Jul 22 06:19 /var/lib/mysql/test_db/test4.ibd
-rw-rw---- 1 mysql mysql 114688 Jul 22 18:09 /var/lib/mysql/test_db/tt.ibd

```

`frm`是定义表结构的文件。

独立表空间文件中仅存放该表对应数据、索引、insert buffer bitmap。
其余的诸如：undo信息、insert buffer 索引页、double write buffer 等信息依然放在默认表空间，也就是共享表空间中。
即使你设置了innodb_file_per_table=ON 共享表空间的体量依然会不断的增长，并且你即使你不断的使用undo进行rollback，共享表空间大小也不会缩减。

优点：

- 提升容错率，表A的表空间损坏后，其他表空间不会收到影响。
- 使用MySQL Enterprise Backup快速备份或还原在每表文件表空间中创建的表，不会中断其他InnoDB 表的使用

缺点：

对fsync系统调用来说不友好，如果使用一个表空间文件的话单次系统调用可以完成数据的落盘，但是如果你将表空间文件拆分成多个。原来的一次fsync可能会就变成针对涉及到的所有表空间文件分别执行一次fsync，增加fsync的次数。

#### 临时表空间(看，重要)

**在 mysql5.7 时，杀掉会话，临时表会释放，但是仅仅是在 ibtmp 文件里标记一下，空间是不会释放回操作系统的。如果要释放空间，需要重启数据库；在 mysql8.0 中可以通过杀掉会话来释放临时表空间。**

> Mysql 8.0 之前重启 mysql 实例是最有效的清临时文件方式



临时表空间用于存放用户创建的临时表和磁盘内部临时表。

参数innodb_temp_data_file_path定义了临时表空间的一些名称、大小、规格属性如下

不同版本的Mysql对临时表空间处理不一样

为了避免ibtmp1文件无止境的暴涨导致再次出现此情况,可以修改参数，限制其文件最大尺寸。

如果文件大小达到上限时，需要生成临时表的SQL无法被执行

`innodb_temp_data_file_path = ibtmp1:12M:autoextend:max:5G  # 12M代表文件初始大小，5G代表最大size`

```bash
## Mysql 5.6

mysql> select @@version;
+-----------+
| @@version |
+-----------+
| 5.6.51    |
+-----------+
1 row in set (0.12 sec)

mysql>  show variables like '%innodb_temp%';
Empty set (0.02 sec)

mysql> show variables like "%tmp%";
+----------------------------+----------+
| Variable_name              | Value    |
+----------------------------+----------+
| default_tmp_storage_engine | InnoDB   |
| innodb_tmpdir              |          |
| max_tmp_tables             | 32       |
| slave_load_tmpdir          | /tmp     |
| tmp_table_size             | 16777216 |
| tmpdir                     | /tmp     |
+----------------------------+----------+
6 rows in set (0.00 sec)

## Mysql 5.7
mysql> select @@version;
+-----------+
| @@version |
+-----------+
| 5.7.35    |
+-----------+
1 row in set (0.01 sec)

mysql> show variables like '%innodb_temp%';
+----------------------------+-----------------------+
| Variable_name              | Value                 |
+----------------------------+-----------------------+
| innodb_temp_data_file_path | ibtmp1:12M:autoextend |
+----------------------------+-----------------------+
1 row in set (0.22 sec)




```

##### 5.6

在MySQL5.6中，磁盘临时表位于tmpdir下，文件名类似#sql_4d2b_8_0，其中#sql是固定的前缀，4d2b是进程号的十六进制表示，8是MySQL线程号的十六进制表示(show processlist中的id)，0是每个连接从0开始的递增值，ibd是innodb的磁盘临时表(通过参数default_tmp_storage_engine控制)。在5.6中，磁盘临时表创建好后，对应的frm以及引擎文件就可以在tmpdir下查看到。在连接断开后，相应文件会自动删除。因此，在5.6的tmpdir里面看到很多类似格式文件名，可以通过文件名来判断是哪个进程，哪个连接使用的临时表，这个技巧在排查tmpdir目录占用过多空间的问题时尤其适用。用户显式创建的这种临时表，在连接释放的时候，会自动释放并把空间释放回操作系统。临时表的undolog存在undo表空间中。

##### 5.7

在MySQL5.7中，临时磁盘表位于ibtmp1文件中，ibtmp1文件位置及大小控制方式由参数innodb_temp_data_file_path控制。显式创建的表的数据和undo都在ibtmp1里面。用户连接断开后，临时表会释放，但是仅仅是在ibtmp1文件里面标记一下，空间是不会释放回操作系统的。如果要释放空间，需要重启数据库。另外，需要注意的一点是，5.6可以在tmpdir下直接看到创建的文件，但是5.7是创建在ibtmp1这个表空间里面，因此是看不到具体的表文件的。如果需要查看，则需要查看INFORMATION_SCHEMA.INNODB_TEMP_TABLE_INFO这个表，里面有一列name，这里可以看到表名。

##### 5.8

通过杀掉会话来释放临时表空间。

 在MySQL8.0中，临时表的数据和undo被进一步分开，数据存放在ibt文件中(由参数innodb_temp_tablespaces_dir控制)，undo依然存放在ibtmp1文件中(由参数innodb_temp_data_file_path控制)。存放ibt文件的叫做会话临时表空间，存放undo的ibtmp1叫做全局临时表空间。会话临时表空间，在磁盘上的表现是一组以ibt文件组成的文件池。

##### 临时表空间导致磁盘不足

###### 原因

数据库 data 磁盘不足，磁盘占用 80% 以上

登陆告警的服务器，查看磁盘空间，并寻找大容量文件后，发现端口号为 4675 的实例临时表空间 ibtmp1 的大小有 955G，导致磁盘被使用了 86%；

###### 定位

使用`explain SQL_Segment` 分析Sql语句，Extra中出现`using temporary;` 字段的，说明该sql会用到临时计算就会导致临时表空间增大。

很明显看到图中标记的这一点为使用了临时计算，说明临时表空间的快速增长和它有关系。这条 SQL 进行了三表关联，每个表都有几十万行数据，三表关联并没有在 where 条件中设置关联字段，形成了笛卡尔积，所以会产生大量临时数据；而且都是全表扫描，加载的临时数据过多;还涉及到排序产生了临时数据；这几方面导致 ibtmp1 空间快速爆满。

###### 解决

感觉这就是代码的一个个版本更新啊，5.8将临时表空间的句柄有mysqld把持转交给session来管理--

5.6 和 5.7 需要重启

**会话被杀掉后，临时表是释放的，只是在 ibtmp1 中打了删除标记，空间并没有还给操作系统，只有重启才可以释放空间。**

**在正常关闭或初始化中止时，将删除临时表空间，并在每次启动服务器时重新创建。重启能够释放空间的原因在于正常关闭数据库，临时表空间就被删除了，重新启动后重新创建，也就是重启引发了临时表空间的重建，重新初始化，所以，重建后的大小为 12M。**

5.8 只需要kill掉会话即可

**在 mysql8.0 中可以通过杀掉会话来释放临时表空间。**

##### 如何避免临时表空间增大

1、设置临时表空间文件上限

对临时表空间的大小进行限制，允许自动增长，但最大容量有上限,本例中由于 innodb_temp_data_file_path 设置的自动增长，但未设上限，所以导致 ibtmp1 有 955G。

正确方法配置参数 

`innodb_temp_data_file_path=ibtmp1:12M:autoextend:max:500M`

设置了上限的大小，当数据文件达到最大大小时，查询将失败，并显示一条错误消息，表明表已满，查询不能往下执行，避免 ibtmp1 过大。

2、加强SQL审核

在发送例如本例中的多表关联 SQL 时应确保有关联字段而且有索引，避免笛卡尔积式的全表扫描，对存在 group by、order by、多表关联的 SQL 要评估临时数据量，对 SQL 进行审核，没有审核不允许上线执行

3、在执行前通过 explain 查看执行计划，对 Using temporary 需要格外关注。



##### 什么情况下会用到临时表

当EXPLAIN 查看执行计划结果的 Extra 列中，如果包含 Using Temporary 就表示会用到临时表,例如如下几种常见的情况通常就会用到：

**3.1  GROUP BY 无索引字段或GROUP  BY+ ORDER  BY 的子句字段不一样时**

```javascript
/**  先看一下表结构 */
mysql> show  create table  test_tmp1\G
*************************** 1. row ***************************
       Table: test_tmp1
Create Table: CREATE TABLE `test_tmp1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(50) DEFAULT NULL,
  `col2` varchar(25) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `name` (`name`)
) ENGINE=InnoDB AUTO_INCREMENT=16 DEFAULT CHARSET=utf8
1 row in set (0.00 sec)


/**  group  by无索引字段*/
mysql> explain select * from test_tmp1 group by  col2 ;
+----+-------------+-----------+------------+------+---------------+------+---------+------+------+----------+---------------------------------+
| id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                           |
+----+-------------+-----------+------------+------+---------------+------+---------+------+------+----------+---------------------------------+
|  1 | SIMPLE      | test_tmp1 | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    8 |   100.00 | Using temporary; Using filesort |
+----+-------------+-----------+------------+------+---------------+------+---------+------+------+----------+---------------------------------+




/**  group by 与order by字段不一致时，及时group by和order by字段有索引也会使用 */
mysql> explain select name from test_tmp1 group by  name order by id desc;
+----+-------------+-----------+------------+-------+---------------+------+---------+------+------+----------+-----------------------------------------------------------+
| id | select_type | table     | partitions | type  | possible_keys | key  | key_len | ref  | rows | filtered | Extra                                                     |
+----+-------------+-----------+------------+-------+---------------+------+---------+------+------+----------+-----------------------------------------------------------+
|  1 | SIMPLE      | test_tmp1 | NULL       | range | name          | name | 153     | NULL |    3 |   100.00 | Using index for group-by; Using temporary; Using filesort |
+----+-------------+-----------+------------+-------+---------------+------+---------+------+------+----------+-----------------------------------------------------------+
1 row in set, 1 warning (0.02 sec)
```

**3.2  order by  与distinct 共用，其中distinct与order by里的字段不一致（主键字段除外）**

```javascript
/**  例子中有无索引时会存在，如果2个字段都有索引会如何*/
mysql> alter table  test_tmp1 add key col2(col2);
Query OK, 0 rows affected (1.07 sec)
Records: 0  Duplicates: 0  Warnings: 0


/**   结果如下，其实该写法与group by +order by 一样*/
mysql> explain select distinct col2  from test_tmp1 order  by  name;
+----+-------------+-----------+------------+-------+---------------+------+---------+------+------+----------+---------------------------------+
| id | select_type | table     | partitions | type  | possible_keys | key  | key_len | ref  | rows | filtered | Extra                           |
+----+-------------+-----------+------------+-------+---------------+------+---------+------+------+----------+---------------------------------+
|  1 | SIMPLE      | test_tmp1 | NULL       | index | col2          | col2 | 78      | NULL |    8 |   100.00 | Using temporary; Using filesort |
+----+-------------+-----------+------------+-------+---------------+------+---------+------+------+----------+---------------------------------+
1 row in set, 1 warning (0.00 sec)
```

 c)  UNION查询（MySQL5.7后union all已不使用临时表）

```javascript
/**  先测一下union all的情况*/


mysql> explain select name from test_tmp1 union all  select name from test_tmp1 where id <10;
+----+-------------+-----------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
| id | select_type | table     | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-----------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
|  1 | PRIMARY     | test_tmp1 | NULL       | index | NULL          | name    | 153     | NULL |    8 |   100.00 | Using index |
|  2 | UNION       | test_tmp1 | NULL       | range | PRIMARY       | PRIMARY | 4       | NULL |    8 |   100.00 | Using where |
+----+-------------+-----------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
2 rows in set, 1 warning (0.01 sec)


/**  再看一下union 作为对比,发现出现了使用临时表的情况*/
mysql> explain select name from test_tmp1 union   select name from test_tmp1 where id <10;
+----+--------------+------------+------------+-------+---------------+---------+---------+------+------+----------+-----------------+
| id | select_type  | table      | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra           |
+----+--------------+------------+------------+-------+---------------+---------+---------+------+------+----------+-----------------+
|  1 | PRIMARY      | test_tmp1  | NULL       | index | NULL          | name    | 153     | NULL |    8 |   100.00 | Using index     |
|  2 | UNION        | test_tmp1  | NULL       | range | PRIMARY       | PRIMARY | 4       | NULL |    8 |   100.00 | Using where     |
| NULL | UNION RESULT | <union1,2> | NULL       | ALL   | NULL        | NULL    | NULL    | NULL | NULL |     NULL | Using temporary |
+----+--------------+------------+------------+-------+---------------+---------+---------+------+------+----------+-----------------+
3 rows in set, 1 warning (0.00 sec)
```

**3.4  insert into select ...from ...**

```javascript
/**  简单看一下本表的数据重复插入的情况 */
mysql> explain insert into test_tmp1(name,col2)  select name,col2 from test_tmp1;
+----+-------------+-----------+------------+------+---------------+------+---------+------+------+----------+-----------------+
| id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra           |
+----+-------------+-----------+------------+------+---------------+------+---------+------+------+----------+-----------------+
|  1 | INSERT      | test_tmp1 | NULL       | ALL  | NULL          | NULL | NULL    | NULL | NULL |     NULL | NULL            |
|  1 | SIMPLE      | test_tmp1 | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    8 |   100.00 | Using temporary |
+----+-------------+-----------+------------+------+---------------+------+---------+------+------+----------+-----------------+
2 rows in set (0.00 sec)
```

小结：  上面列举的是最常见的使用临时表的情况,其中基本都是引起慢查询的因素，因此，如果遇到临时表空间文件暴涨是需要查看一下是否有大量的慢查询。

#### undo表空间

在MySQL的设定中，有一个表空间可以专门用来存放undolog的日志文件。

然而，在MySQL的设定中，默认的会将undolog放置到系统表空间中

```bash
## mysql 5.6
mysql> select @@version;
+-----------+
| @@version |
+-----------+
| 5.6.51    |
+-----------+

mysql> show variables like '%undo%';
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| innodb_undo_directory   | .     |
| innodb_undo_logs        | 128   |
| innodb_undo_tablespaces | 0     |
+-------------------------+-------+
3 rows in set (0.00 sec)


## mysql 5.7
mysql> select @@version;
+-----------+
| @@version |
+-----------+
| 5.7.35    |
+-----------+
1 row in set (0.01 sec)

mysql> show variables like '%undo%';
+--------------------------+------------+
| Variable_name            | Value      |
+--------------------------+------------+
| innodb_max_undo_log_size | 1073741824 |
| innodb_undo_directory    | ./         |
| innodb_undo_log_truncate | OFF        |
| innodb_undo_logs         | 128        |
| innodb_undo_tablespaces  | 0          |
+--------------------------+------------+
5 rows in set (0.40 sec)

```

InnoDB的undo log是其实现多版本的关键组件，在物理上以数据页的形式进行组织。在早期版本中(<5.6)，undo tablespace是在ibdata中，因此一个常见的问题是由于大事务不提交导致ibdata膨胀，这时候通常只有重建数据库一途来缩小空间。到了MySQL5.6版本， InnoDB开始支持独立的undo tablespace，也就是说，undo log可以存储于ibdata之外。但这个特性依然鸡肋：

1. 首先你必须在install实例的时候就指定好独立Undo tablespace, 在install完成后不可更改。
2. Undo tablepsace的space id必须从1开始，无法增加或者删除undo tablespace。

到了MySQL5.7版本中，终于引入了一个期待已久的功能：即在线truncate undo tablespace。DBA终于摆脱了undo空间膨胀的苦恼。

到了5.8 开始从 SQL 层面非常方便的管理 Undo 表空间

##### MySQL 5.5时代的undo log

在MySQL5.5以及之前，大家会发现随着数据库上线时间越来越长，ibdata1文件（即InnoDB的共享表空间，或者系统表空间）会越来越大，这会造成2个比较明显的问题： 
（1）磁盘剩余空间越来越小，到后期往往要加磁盘； 
（2）物理备份时间越来越长，备份文件也越来越大。

这是怎么回事呢？ 
原因除了数据量自然增长之外，在MySQL5.5以及之前，InnoDB的undo log也是存放在ibdata1里面的。一旦出现大事务，这个大事务所使用的undo log占用的空间就会一直在ibdata1里面存在，即使这个事务已经关闭。

那么问题来了，有办法把上面说的空闲的undo log占用的空间从ibdata1里面清理掉吗？答案是没有直接的办法，只能全库导出sql文件，然后重新初始化mysql实例，再全库导入。

 

##### MySQL 5.6时代的undo log

MySQL 5.6增加了参数innodb_undo_directory、innodb_undo_logs和innodb_undo_tablespaces这3个参数，可以把undo log从ibdata1移出来单独存放。

下面对这3个参数做一下解释：

（1） **innodb_undo_directory**，指定单独存放undo表空间的目录，默认为.（即datadir），可以设置相对路径或者绝对路径。该参数实例初始化之后虽然不可直接改动，但是可以通过先停库，修改配置文件，然后移动undo表空间文件的方式去修改该参数；

（2） **innodb_undo_tablespaces**，指定单独存放的undo表空间个数，例如如果设置为3，则undo表空间为undo001、undo002、undo003，每个文件初始大小默认为10M。该参数我们推荐设置为大于等于3，原因下文将解释。该参数实例初始化之后不可改动；

（3） **innodb_undo_logs**，指定回滚段的个数（早期版本该参数名字是innodb_rollback_segments），默认128个。每个回滚段可同时支持1024个在线事务。这些回滚段会平均分布到各个undo表空间中。该变量可以动态调整，但是物理上的回滚段不会减少，只是会控制用到的回滚段的个数。

实际使用方面，在初始化实例之前，我们只需要设置innodb_undo_tablespaces参数（建议大于等于3）即可将undo log设置到单独的undo表空间中。如果需要将undo log放到更快的设备上时，可以设置innodb_undo_directory参数，但是一般我们不这么做，因为现在SSD非常普及。innodb_undo_logs可以默认为128不变。

 

##### MySQL 5.7时代的undo log

那么问题又来了，undo log单独拆出来后就能缩小了吗？MySQL 5.7引入了新的参数，innodb_undo_log_truncate，开启后可在线收缩拆分出来的undo表空间。在满足以下2个条件下，undo表空间文件可在线收缩：

（1） **innodb_undo_tablespaces**>=2。因为truncate undo表空间时，该文件处于inactive状态，如果只有1个undo表空间，那么整个系统在此过程中将处于不可用状态。为了尽可能降低truncate对系统的影响，建议将该参数最少设置为3；

（2） **innodb_undo_logs**>=35（默认128）。因为在MySQL 5.7中，第一个undo log永远在系统表空间中，另外32个undo log分配给了临时表空间，即ibtmp1，至少还有2个undo log才能保证2个undo表空间中每个里面至少有1个undo log；

满足以上2个条件后，把 **innodb_undo_log_truncate**设置为ON即可开启undo表空间的自动truncate，这还跟如下2个参数有关：

（1） **innodb_max_undo_log_size**，undo表空间文件超过此值即标记为可收缩，默认1G，可在线修改；

（2） **innodb_purge_rseg_truncate_frequency,**指定purge操作被唤起多少次之后才释放rollback segments。当undo表空间里面的rollback segments被释放时，undo表空间才会被truncate。由此可见，该参数越小，undo表空间被尝试truncate的频率越高。

 

##### MySQL 5.7的undo表空间的truncate示例

（1） 首先确保如下参数被正确设置：

```bash
mysql> select @@version;
+-----------+
| @@version |
+-----------+
| 5.7.35    |
+-----------+
1 row in set (0.00 sec)

mysql> show variables like '%undo%';
+--------------------------+-----------+
| Variable_name            | Value     |
+--------------------------+-----------+
# 为了实验方便，我们减小该值 100M
| innodb_max_undo_log_size | 104857600 |
| innodb_undo_directory    | /tmp/     |
| innodb_undo_log_truncate | ON        |
| innodb_undo_logs         | 128       |
| innodb_undo_tablespaces  | 3         |
+--------------------------+-----------+
5 rows in set (0.00 sec)

# 为了实验方便，我们减小该值
mysql> show variables like '%innodb_purge_rseg%';
+--------------------------------------+-------+
| Variable_name                        | Value |
+--------------------------------------+-------+
| innodb_purge_rseg_truncate_frequency | 10    |
+--------------------------------------+-------+
1 row in set (0.00 sec)
```

（2） 创建表： 

```sql
mysql> create database test_db;
Query OK, 1 row affected (0.02 sec)

mysql> use test_db;
Database changed
mysql> create table t1( id int primary key auto_increment, name varchar(200));
Query OK, 0 rows affected (0.02 sec)

```

end

（3）插入测试数据 

```sql
mysql>  insert into t1(name) values(repeat('a',200));
Query OK, 1 row affected (0.01 sec)

mysql> insert into t1(name) select name from t1;
Query OK, 4 rows affected (0.00 sec)
Records: 4  Duplicates: 0  Warnings: 0

mysql> insert into t1(name) select name from t1;
Query OK, 8 rows affected (0.00 sec)
Records: 8  Duplicates: 0  Warnings: 0

mysql> insert into t1(name) select name from t1;
Query OK, 16 rows affected (0.00 sec)
Records: 16  Duplicates: 0  Warnings: 0

mysql> insert into t1(name) select name from t1;
Query OK, 32 rows affected (0.00 sec)
Records: 32  Duplicates: 0  Warnings: 0

```

这时undo表空间文件大小如下，可以看到有一个undo文件已经超过了100M：

```bash
## 我的栗子....无论怎么操作就是固定10M
root@505069ec2fda:/# du -sh /tmp/*
10M     /tmp/undo001
10M     /tmp/undo002
10M     /tmp/undo003

## 两个表花里胡哨的对插才搞大
mysql>  insert into t1(name) select name from t2;
Query OK, 3145730 rows affected (43.63 sec)
Records: 3145730  Duplicates: 0  Warnings: 0

mysql>  insert into t2(name) select name from t1;
Query OK, 3670018 rows affected (48.92 sec)
Records: 3670018  Duplicates: 0  Warnings: 0



## 别人的栗子
-rw-r----- 1 mysql mysql 13M Feb 17 17:59 undo001 
-rw-r----- 1 mysql mysql 128M Feb 17 17:59 undo002 
-rw-r----- 1 mysql mysql 64M Feb 17 17:59 undo003 
```



（4）运行purge线程清理undo

```bash
## 哇哇哇 自动清理了啊

root@505069ec2fda:/# du -sh /tmp/*
105M    /tmp/undo001
10M     /tmp/undo002
13M     /tmp/undo003
root@505069ec2fda:/# du -sh /tmp/*
10M     /tmp/undo001
21M     /tmp/undo002
13M     /tmp/undo003



--------- 这是别人的栗子，我的自动清理了。
此时，为了，让purge线程运行，可以运行几个delete语句：

mysql> delete from t1 limit 1; 
mysql> delete from t1 limit 1; 
mysql> delete from t1 limit 1; 
mysql> delete from t1 limit 1;

再查看undo文件大小：

-rw-r----- 1 mysql mysql 13M Feb 17 18:05 undo001 
-rw-r----- 1 mysql mysql 10M Feb 17 18:05 undo002 
-rw-r----- 1 mysql mysql 64M Feb 17 18:05 undo003 
可以看到，超过100M的undo文件已经收缩到10M了。

```

end

 

 

MySQL5.7 可以回收（收缩）undo log回滚日志物理文件空间

undo log回滚日志是保存在共享表空间ibdata1文件里，随着业务的不停运转，ibdata1文件会越来越大，想要回收（收缩空间大小）极其困难和复杂， 必须先mysqldump -A全库的导出，然后删掉data目录，然后重新初始化安装，最后再把全库的SQL文件导入，采用这种方法进行ibdata1文件的回收。直到MySQL5.7 ，才支持在线收缩。

```bash
innodb_undo_log_truncate参数设置为1，即开启在线回收（收缩）undo log日志文件，支持动态设置。
innodb_undo_tablespaces参数必须大于或等于2，即回收（收缩）一个undo log日志文件时，要保证另一个undo log是可用的。
innodb_undo_logs: undo回滚段的数量， 至少大于等于35，默认128。
innodb_max_undo_log_size：当超过这个阀值（默认是1G），会触发truncate回收（收缩）动作，truncate后空间缩小到10M。
innodb_purge_rseg_truncate_frequency：控制回收（收缩）undo log的频率。undo log空间在它的回滚段没有得到释放之前不会收缩，
想要增加释放回滚区间的频率，就得降低innodb_purge_rseg_truncate_frequency设定值。
```

end

#### Mysql 8.0：运行自定义表空间的好处

从MySQL 8.0开始允许用户自定义表空间，具体语法如下：

```javascript
CREATE TABLESPACE tablespace_name
    ADD DATAFILE 'file_name'               #数据文件名
    USE LOGFILE GROUP logfile_group        #自定义日志文件组，一般每组2个logfile。
    [EXTENT_SIZE [=] extent_size]          #区大小
    [INITIAL_SIZE [=] initial_size]        #初始化大小 
    [AUTOEXTEND_SIZE [=] autoextend_size]  #自动扩宽尺寸
    [MAX_SIZE [=] max_size]                #单个文件最大size，最大是32G。
    [NODEGROUP [=] nodegroup_id]           #节点组
    [WAIT]
    [COMMENT [=] comment_text]
    ENGINE [=] engine_name
```

这样的好处是可以做到数据的冷热分离，分别用HDD和SSD来存储，既能实现数据的高效访问，又能节约成本，比如可以添加两块500G硬盘，经过创建卷组vg，划分逻辑卷lv，创建数据目录并mount相应的lv，假设划分的两个目录分别是/hot_data 和 /cold_data。

这样就可以将核心的业务表如用户表，订单表存储在高性能SSD盘上，一些日志，流水表存储在普通的HDD上，主要的操作步骤如下：

```javascript
#创建热数据表空间
create tablespace tbs_data_hot add datafile '/hot_data/tbs_data_hot01.dbf' max_size 20G;
#创建核心业务表存储在热数据表空间
create table booking(id bigint not null primary key auto_increment, …… ) tablespace tbs_data_hot;
#创建冷数据表空间
create tablespace tbs_data_cold add datafile '/cold_data/tbs_data_cold01.dbf' max_size 20G;
#创建日志，流水，备份类的表存储在冷数据表空间
create table payment_log(id bigint not null primary key auto_increment, …… ) tablespace tbs_data_cold;
#可以移动表到另一个表空间
alter table payment_log tablespace tbs_data_hot;
```

end

 

## 例子

开启 

5.7以后System and status 变量需要从performance_schema中进行获取，information_schema仍然保留了GLOBAL_STATUS，GLOBAL_VARIABLES两个表做兼容。

如果希望沿用information_schema中进行查询的习惯，5.7提供了show_compatibility_56参数，设置为ON可以兼容5.7之前的用法，否则就会报错：

```sql
root@localhost:test 5.7.25-log 09:30:58> select * from information_schema.session_status;
ERROR 3167 (HY000): The 'INFORMATION_SCHEMA.SESSION_STATUS' feature is disabled; see the documentation for 'show_compatibility_56'
root@localhost:test 5.7.25-log 09:31:01> set @@global.show_compatibility_56=ON;
Query OK, 0 rows affected (0.00 sec)

root@localhost:test 5.7.25-log 09:33:59> select * from information_schema.session_status;
+-----------------------------------------------+--------------------------------------------------+
| VARIABLE_NAME                                 | VARIABLE_VALUE                                   |
+-----------------------------------------------+--------------------------------------------------+
...
```



end

```bash
create table user(id bigint not null primary key auto_increment, 
name varchar(20) not null default '' comment '姓名', 
age tinyint not null default 0 comment 'age', 
gender char(1) not null default 'M'  comment '性别',
phone varchar(16) not null default '' comment '手机号',
create_time datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
update_time datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '修改时间'
) engine = InnoDB DEFAULT CHARSET=utf8mb4 COMMENT '用户信息表';

>  show variables like '%datadir%';
+---------------+-----------------------------------+
| Variable_name | Value                             |
+---------------+-----------------------------------+
| datadir       | /data/mysql/mysqldata3316/mydata/ |
+---------------+-----------------------------------+
1 row in set (0.01 sec)


$ du -sh /data/mysql/mysqldata3316/mydata/test/user.ibd
96K     /data/mysql/mysqldata3316/mydata/test/user.ibd


## 插入数据
DELIMITER $$
CREATE PROCEDURE insert_user_data(num INTEGER) 
BEGIN
    DECLARE v_i int unsigned DEFAULT 0;
set autocommit= 0;
WHILE v_i < num DO
   insert into user(`name`, age, gender, phone) values (CONCAT('lyn',v_i), mod(v_i,120), 'M', CONCAT('152',ROUND(RAND(1)*100000000)));
 SET v_i = v_i+1;
END WHILE;
commit;
END $$

DELIMITER ;

## 执行
call insert_user_data(100000);

$ du -sh /data/mysql/mysqldata3316/mydata/test/user.ibd
14M     /data/mysql/mysqldata3316/mydata/test/user.ibd

## 删除数据
delete from user limit 50000;
## 没有减少
$ du -sh /data/mysql/mysqldata3316/mydata/test/user.ibd
14M     /data/mysql/mysqldata3316/mydata/test/user.ibd


 SELECT A.SPACE AS TBL_SPACEID, A.TABLE_ID, A.NAME AS TABLE_NAME, FILE_FORMAT, ROW_FORMAT, SPACE_TYPE,  B.INDEX_ID , B.NAME AS INDEX_NAME, PAGE_NO, B.TYPE AS INDEX_TYPE FROM INNODB_SYS_TABLES A LEFT JOIN INNODB_SYS_INDEXES B ON A.TABLE_ID =B.TABLE_ID WHERE A.NAME = 'test/user';
ERROR 1146 (42S02): Table 'test.innodb_sys_tables' doesn't exist


## 
select table_schema,        table_name,ENGINE,        round(DATA_LENGTH/1024/1024+ INDEX_LENGTH/1024/1024) total_mb,TABLE_ROWS,        round(DATA_LENGTH/1024/1024) data_mb, round(INDEX_LENGTH/1024/1024) index_mb, round(DATA_FREE/1024/1024) free_mb, round(DATA_FREE/DATA_LENGTH*100,2) free_ratio from information_schema.TABLES where  TABLE_SCHEMA= 'test' and TABLE_NAME= 'user'\G
*************************** 1. row ***************************
table_schema: test
  table_name: user
      ENGINE: InnoDB
    total_mb: 7
  TABLE_ROWS: 49875
     data_mb: 7
    index_mb: 0
     free_mb: 4
  free_ratio: 61.39
1 row in set (0.00 sec)

## 回收碎片空间
## 对于InnoDB的表，可以通过以下命令来回收碎片，释放空间，这个是随机读IO操作，会比较耗时，也会阻塞表上正常的DML运行，同时需要占用额外更多的磁盘空间，对于RDS来说，可能会导致磁盘空间瞬间爆满，实例瞬间被锁定，应用无法做DML操作，所以禁止在线上环境去执行。
 alter table user engine=InnoDB;
 
$ du -sh /data/mysql/mysqldata3316/mydata/test/user.ibd
10M     /data/mysql/mysqldata3316/mydata/test/user.ibd

```

end