## 索引和锁-mysql-4

## 索引目的

索引的目的在于提高查询效率。

## 索引类型

其实主键和索引都是键，不过主键是逻辑键，索引是物理键，意思就是主键不实际存在，而索引实际存在在数据库中，主键一般都要建，主要是用来避免一张表中有相同的记录，索引一般可以不建，但如果需要对该表进行查询操作，则最好建，这样可以加快检索的速度。

普通索引/非唯一索引：使用KEY关键，仅加速查询（关键字 UNIQUE 把它定义为一个唯一索引）

唯一索引：使用 KEY + UNIQUE关键字，加速查询 + 列值唯一（可以有null）

主键索引：使用PRIMARY KEY关键字， 加速查询 + 列值唯一（不可以有null）+ 表中只有一个组合索引：多列值组成一个索引，专门用于组合搜索，其效率大于索引合并

全文索引：对文本的内容进行分词，进行搜索

唯一索引和普通索引使用的结构都是 B-tree，执行时间复杂度都是 O(log n)

主键与唯一索引不同的是：
a. 有not null属性；
b. 每个表只能有一个。

### 非聚簇索引/二级索引

非聚簇索引的叶子中存储的value是主键的值。

我们平时在使用的Mysql中，使用下述语句

```sql
CREATE [UNIQUE|FULLTEXT|SPATIAL] INDEX index_name
    [USING index_type]
    ON tbl_name (index_col_name,...)
 
index_col_name:
    col_name [(length)] [ASC | DESC]
```

创建的索引，如复合索引、前缀索引、唯一索引，都是属于非聚簇索引，在有的书籍中，又将其称为**辅助索引**(secondary index)。也称其为非聚簇索引，其数据结构为B+树。

二级索引需要两次索引查找，而不是一次才能取到数据，因为存储引擎第一次需要通过二级索引找到索引的叶子节点，从而找到数据的主键，然后在聚簇索引中用主键再次查找索引，再找到数据。

即，二级索引/辅助索引/非聚簇索引的叶子存储的是primary key。

如果主键比较大的话，那辅助索引将会变的更大，因为**辅助索引的叶子存储的是主键值；过长的主键值，会导致非叶子节点占用占用更多的物理空间**。

```sql
root@localhost:test 5.7.25-log 02:35:50> show create table geek;
+-------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                                                                                                                                                                       |
+-------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| geek  | CREATE TABLE `geek` (
  `a` int(11) NOT NULL,
  `b` int(11) NOT NULL,
  `c` int(11) NOT NULL,
  `d` int(11) NOT NULL,
  PRIMARY KEY (`a`,`b`),
  KEY `c` (`c`),
  KEY `ca` (`c`,`a`),
  KEY `cb` (`c`,`b`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 |
+-------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

root@localhost:test 5.7.25-log 02:39:10> selcet a from geek where c=2;
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'selcet a from geek where c=2' at line 1
root@localhost:test 5.7.25-log 02:42:16> select a from geek where c=2;
+---+
| a |
+---+
| 1 |
| 2 |
+---+
2 rows in set (0.00 sec)
--- a是主键，c是索引，所以不需要回表。
> explain select a from geek where c=2\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: geek
   partitions: NULL
         type: ref
possible_keys: c,ca,cb
          key: c
      key_len: 4
          ref: const
         rows: 2
     filtered: 100.00
        Extra: Using index
1 row in set, 1 warning (0.00 sec)

Note (Code 1003): /* select#1 */ select `test`.`geek`.`a` AS `a` from `test`.`geek` where (`test`.`geek`.`c` = 2)

--- d不是主键，所以需要通过二级索引c拿到主键a，再回表才能找到d。
> explain select d from geek where c=2\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: geek
   partitions: NULL
         type: ref
possible_keys: c,ca,cb
          key: c
      key_len: 4
          ref: const
         rows: 2
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0.00 sec)

Note (Code 1003): /* select#1 */ select `test`.`geek`.`d` AS `d` from `test`.`geek` where (`test`.`geek`.`c` = 2)

```

end



### 聚簇索引（流弊）

https://juejin.cn/post/6844903845554814983

https://www.cnblogs.com/rjzheng/p/9915754.html

聚簇索引，在Mysql中是没有语句来另外生成的。在Innodb中，Mysql中的数据是按照主键的顺序来存放的。那么聚簇索引就是按照每张表的主键来构造一颗B+树，叶子节点存放的就是整张表的行数据。由于表里的数据只能按照一颗B+树排序，因此一张表只能有一个聚簇索引。

在Innodb中，聚簇索引默认就是主键索引。

如果没有主键，则按照下列规则来建聚簇索引

- 没有主键时，会用一个**唯一且不为空**的索引列做为主键，成为此表的聚簇索引
- 如果没有这样的索引，InnoDB会隐式定义一个主键来作为聚簇索引。

### 自增主键的吃处

自增主键和uuid作为主键的区别么？由于主键使用了聚簇索引，如果主键是自增id，，那么对应的数据一定也是相邻地存放在磁盘上的，写入性能比较高。如果是uuid的形式，频繁的插入会使innodb频繁地移动磁盘块，写入性能就比较低了。

### MyISAM索引

MyISAM的`索引与行记录是分开存储的`，叫做**非聚集索引**（UnClustered Index）。

其主键索引与普通索引没有本质差异：

- 有连续聚集的区域`单独存储行记录`
- 主键索引的叶子节点，存储主键，与对应行记录的`指针`
- 普通索引的叶子结点，存储索引列，与对应行记录的`指针`

即，MyISAM的表可以没有主键。


## 索引原理（流弊）

https://tech.meituan.com/2014/06/30/mysql-index.html

https://www.cnblogs.com/rjzheng/p/9915754.html

没创建一个索引就会创建一个非聚簇B+树。

记住

- innodb一定存在聚簇索引，默认以主键作为聚簇索引
- 有几个索引，就有几棵B+树(不考虑hash索引的情形)
- 聚簇索引的叶子节点为磁盘上的真实数据。非聚簇索引的叶子节点还是索引，指向聚簇索引B+树。

## 索引实现

Mysql目前主要有以下几种索引类型：FULLTEXT，HASH，BTREE，RTREE。

先来一张带主键的表，如下所示，pId是主键

| pId  | name     | birthday   |
| ---- | -------- | ---------- |
| 5    | zhangsan | 2016-10-02 |
| 8    | lisi     | 2015-10-04 |
| 11   | wangwu   | 2016-09-02 |
| 13   | zhaoliu  | 2015-10-07 |

画出该表的结构图如下

一个B+树，是聚簇索引，每个叶子节点都是磁盘上的真实数据即一个节点对应一行数据，

![image](https://www.cnblogs.com/images/cnblogs_com/rjzheng/1281019/o_index01.png)

如上图所示，分为上下两个部分，上半部分是由主键形成的B+树，下半部分就是磁盘上真实的数据！那么，当我们， 执行下面的语句

```sql
select * from table where pId='11'
```

那么，执行过程如下

从聚簇索引树上，使用高效的查找树算法，找到数据所在的叶子节点。

![image](https://www.cnblogs.com/images/cnblogs_com/rjzheng/1281019/o_index02.png)
如上图所示，从根开始，经过3次查找，就可以找到真实数据。如果不使用索引，那就要在磁盘上，进行逐行扫描，直到找到数据位置。显然，使用索引速度会快。但是在写入数据的时候，需要维护这颗B+树的结构，因此写入性能会下降！
OK，接下来引入非聚簇索引!我们执行下面的语句

```sql
create index index_name on table(name);
```

此时结构图如下所示

当再创建一个非聚簇索引时，会再创建一个B+树，其叶子节点的数据是 Primary_Key+Index_Key两个字段。根据非聚簇索引查找数据时，先在非聚簇树上根据索引字段找到其叶子节点，然后读取叶子节点的内容就可知道这行全部数据所在的聚簇树上的位置，然后根据在聚簇树上的位置信息，拿出全部的数据。

> 怪不得 `explain select * from` 不走索引... 

![image](https://www.cnblogs.com/images/cnblogs_com/rjzheng/1281019/o_index04.png)
大家注意看，会根据你的索引字段生成一颗新的B+树。因此， 我们每加一个索引，就会增加表的体积， 占用磁盘存储空间。然而，**注意看叶子节点**，非聚簇索引的叶子节点并不是真实数据，它的叶子节点依然是索引节点，存放的是该索引字段的值以及对应的主键索引(聚簇索引)。
如果我们执行下列语句

```sql
select * from table where name='lisi'
```

此时结构图如下所示
![image](https://www.cnblogs.com/images/cnblogs_com/rjzheng/1281019/o_index05.png)
通过上图红线可以看出，先从非聚簇索引树开始查找，然后找到聚簇索引后。根据聚簇索引，在聚簇索引的B+树上，找到完整的数据！
那

> **什么情况不去聚簇索引树上查询呢？**

还记得我们的非聚簇索引树上存着该索引字段的值么。如果，此时我们执行下面的语句

```sql
select name from table where name='lisi'
```

此时结构图如下

当要获取的字段就是索引字段即非聚簇索引数据时，就不需要再去查找聚簇索引树了，直接在非聚簇索引树上获取到全部的数据信息。

![image](https://www.cnblogs.com/images/cnblogs_com/rjzheng/1281019/o_index06.png)
如上图红线所示，如果在非聚簇索引树上找到了想要的值，就不会去聚簇索引树上查询。还记得，博主在[《select的正确姿势》](https://www.cnblogs.com/rjzheng/p/9902911.html)提到的索引问题么：

> **当执行select col from table where col = ?，col上有索引的时候，效率比执行select \* from table where col = ? 速度快好几倍！**

看完上面的图，你应该对这句话有更深层的理解了。

那么这个时候，我们执行了下述语句，又会发生什么呢？

```sql
create index index_birthday on table(birthday);
```

此时结构图如下
![image](https://www.cnblogs.com/images/cnblogs_com/rjzheng/1281019/o_index07.png)
看到了么，多加一个索引，就会多生成一颗非聚簇索引树。因此，很多文章才说，索引不能乱加。因为，有几个索引，就有几颗非聚簇索引树！你在做插入操作的时候，需要同时维护这几颗树的变化！因此，如果索引太多，插入性能就会下降!



## 索引作用

那么我们需要在什么情况下建立索引呢？一般来说，在WHERE和JOIN中出现的列需要建立索引，但也不完全如此，因为MySQL只对<，<=，=，>，>=，BETWEEN，IN，以及某些时候的LIKE才会使用索引。



## 索引原则

"最左前缀匹配原则"，可以用来精简索引。

1.最左前缀匹配原则，非常重要的原则，mysql会一直向右匹配直到遇到范围查询(>、<、between、like)就停止匹配，比如a = 1 and b = 2 and c > 3 and d = 4 如果建立(a,b,c,d)顺序的索引，d是用不到索引的，如果建立(a,b,d,c)的索引则都可以用到，a,b,d的顺序可以任意调整。



2.=和in可以乱序，比如a = 1 and b = 2 and c = 3 建立(a,b,c)索引可以任意顺序，mysql的查询优化器会帮你优化成索引可以识别的形式。

3.尽量选择区分度高的列作为索引，区分度的公式是count(distinct col)/count(*)，表示字段不重复的比例，比例越大我们扫描的记录数越少，唯一键的区分度是1，而一些状态、性别字段可能在大数据面前区分度就是0，那可能有人会问，这个比例有什么经验值吗？使用场景不同，这个值也很难确定，一般需要join的字段我们都要求是0.1以上，即平均1条扫描10条记录。

4.索引列不能参与计算，保持列“干净”，比如from_unixtime(create_time) = ’2014-05-29’就不能使用到索引，原因很简单，b+树中存的都是数据表中的字段值，但进行检索时，需要把所有元素都应用函数才能比较，显然成本太大。所以语句应该写成create_time = unix_timestamp(’2014-05-29’)。

5.尽量的扩展索引，不要新建索引。比如表中已经有a的索引，现在要加(a,b)的索引，那么只需要修改原来的索引即可。

## 索引删除

```sql
mysql> show create table test2;
+-------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                                                                                                                                                            |
+-------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| test2 | CREATE TABLE `test2` (
  `id` int(11) NOT NULL,
  `num` int(11) DEFAULT NULL,
  `age` int(10) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `num` (`num`) USING BTREE,
  KEY `age` (`age`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8 |
+-------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)
--- 删除索引
mysql> drop index age on test2;
Query OK, 0 rows affected (0.02 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> show create table test2;
+-------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                                                                                                                           |
+-------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| test2 | CREATE TABLE `test2` (
  `id` int(11) NOT NULL,
  `num` int(11) DEFAULT NULL,
  `age` int(10) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `num` (`num`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8 |
+-------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

```

end

## 慢查询优化

https://tech.meituan.com/2014/06/30/mysql-index.html

MySQL的慢查询日志是MySQL提供的一种日志记录，它用来记录在MySQL中响应时间超过阀值的语句，具体指运行时间超过long_query_time值的SQL，则会被记录到慢查询日志中。long_query_time的默认值为10，意思是运行10S以上的语句。默认情况下，MySQLl数据库并不启动慢查询日志，需要我们手动来设置这个参数，当然，如果不是调优需要的话，一般不建议启动该参数，因为开启慢查询日志会或多或少带来一定的性能影响。慢查询日志支持将日志记录写入文件，也支持将日志记录写入数据库表。

### 慢查询优化基本步骤

0.先运行看看是否真的很慢，注意设置SQL_NO_CACHE

1.where条件单表查，锁定最小返回记录表。这句话的意思是把查询语句的where都应用到表中返回的记录数最小的表开始查起，单表每个字段分别查询，看哪个字段的区分度最高

2.explain查看执行计划，是否与1预期一致（从锁定记录较少的表开始查询）

3.order by limit 形式的sql语句让排序的表优先查

4.了解业务方使用场景

5.加索引时参照建索引的几大原则

6.观察结果，不符合预期继续从0分析

### 索引失效

https://segmentfault.com/a/1190000021464570



### 联合索引（看）

```sql

CREATE TABLE `geek` (
  `a` int(11) NOT NULL,
  `b` int(11) NOT NULL,
  `c` int(11) NOT NULL,
  `d` int(11) NOT NULL,
  PRIMARY KEY (`a`,`b`),
  KEY `c` (`c`),
  -- ca key 可以不要
  KEY `ca` (`c`,`a`),
  KEY `cb` (`c`,`b`)
) ENGINE=InnoDB;

--- 即先order by c，再order by a
select * from geek where c=N order by a limit 1;

--- 即先order by c，再order by b
select * from geek where c=N order by b limit 1;
```

主键 a，b 的聚簇索引组织顺序相当于 order by a,b ，也就是先按 a 排序，再按 b 排序，c 无序。即表的默认顺序就行`key(a,b)`的顺序

索引` key(c,a)` 的组织是先按 c 排序，再按 a 排序，同时记录主键b, 这个排序和`key(c)` 索引是一致的。

索引`key(c,b)` 的组织是先按 c 排序，在按 b 排序，同时记录主键b, 这个排序和`key(c)` 索引是不一致的。

所以，`key(c,a)` 可以省略。



```sql
--- 数据操作
select * from geek;
+---+---+---+---+
| a | b | c | d |
+---+---+---+---+
| 1 | 2 | 3 | 6 |
| 1 | 3 | 2 | 6 |
| 1 | 4 | 3 | 6 |
| 2 | 1 | 3 | 6 |
| 2 | 2 | 2 | 6 |
| 2 | 3 | 4 | 6 |
+---+---+---+---+
--- -- key(c) 和 key(c,a)结果一致
root@localhost:test 5.7.25-log 02:33:31> select * from geek where c=2 order by a;
+---+---+---+---+
| a | b | c | d |
+---+---+---+---+
| 1 | 3 | 2 | 6 |
| 2 | 2 | 2 | 6 |
+---+---+---+---+
2 rows in set (0.01 sec)

root@localhost:test 5.7.25-log 02:34:06> select * from geek where c=2;
+---+---+---+---+
| a | b | c | d |
+---+---+---+---+
| 1 | 3 | 2 | 6 |
| 2 | 2 | 2 | 6 |
+---+---+---+---+
2 rows in set (0.00 sec)


----- key(c) 和 key(c,b)结果不一致，流弊
root@localhost:test 5.7.25-log 02:34:20> select * from geek where c=2;
+---+---+---+---+
| a | b | c | d |
+---+---+---+---+
| 1 | 3 | 2 | 6 |
| 2 | 2 | 2 | 6 |
+---+---+---+---+
2 rows in set (0.00 sec)

root@localhost:test 5.7.25-log 02:35:41> select * from geek where c=2 order by b;
+---+---+---+---+
| a | b | c | d |
+---+---+---+---+
| 2 | 2 | 2 | 6 |
| 1 | 3 | 2 | 6 |
+---+---+---+---+
2 rows in set (0.00 sec)

root@localhost:test 5.7.25-log 02:35:50>

```

对于二级索引C，会默认和主键做联合索引。所以索引c的排序为cab，索引cb的排序顺序为cba。所以，结论是 ca 可以去掉，cb 需要保留。

---



对列col1、列col2和列col3建一个联合索引

```sql
KEY test_col1_col2_col3 on test(col1,col2,col3);
```

联合索引 test_col1_col2_col3 实际建立了(col1)、(col1,col2)、(col,col2,col3)三个索引。

> 但是不包含`(col2,col3)`

- **减少开销**。建一个联合索引(col1,col2,col3)，实际相当于建了(col1),(col1,col2),(col1,col2,col3)三个索引。每多一个索引，都会增加写操作的开销和磁盘空间的开销。对于大量数据的表，使用联合索引会大大的减少开销！
- **覆盖索引**。对联合索引(col1,col2,col3)，如果有如下的sql: select col1,col2,col3 from test where col1=1 and col2=2。那么MySQL可以直接通过遍历索引取得数据，而无需回表，这减少了很多的随机io操作。减少io操作，特别的随机io其实是dba主要的优化策略。所以，在真正的实际应用中，覆盖索引是主要的提升性能的优化手段之一。
- **效率高**。索引列越多，通过索引筛选出的数据越少。有1000W条数据的表，有如下`select *from table where col1=1 and col2=2 and col3=3`,假设假设每个条件可以筛选出10%的数据，如果只有单值索引，那么通过该索引能筛选出`1000W*10%=100w`条数据，然后再回表从100w条数据中找到符合`col2=2 and col3= 3`的数据，然后再排序，再分页；如果是联合索引，通过索引筛选出`1000w*10%* 10% *10%=1w`，效率提升可想而知！

#### explain : ref

对于联合索引(col1,col2,col3)，查询语句`SELECT * FROM test WHERE col2=2;`是否能够触发索引？
大多数人都会说NO，实际上却是YES。
**原因**：

```sql
CREATE TABLE `st` (
id bigint(20) not null auto_increment,
col1 bigint(20),
col2 bigint(20),
col3 bigint(20),
primary key (id),
key test_col1_col2_col3(col1,col2,col3)
)engine = InnoDB DEFAULT CHARSET=utf8mb4;

insert into st select 1,1,1,1;
insert into st select 2,2,2,2;
insert into st select 3,3,3,3;


root@localhost:test 5.7.25-log 08:01:34> select * from st;
+----+------+------+------+
| id | col1 | col2 | col3 |
+----+------+------+------+
|  1 |    1 |    1 |    1 |
|  2 |    2 |    2 |    2 |
|  3 |    3 |    3 |    3 |
+----+------+------+------+
3 rows in set (0.00 sec)

--- col1 直接使用了unio key
root@localhost:test 5.7.25-log 08:03:00> explain select * from st where col1=2\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: st
   partitions: NULL
         type: ref
possible_keys: test_col1_col2_col3
          key: test_col1_col2_col3
      key_len: 9
          ref: const
         rows: 1
     filtered: 100.00
        Extra: Using index
1 row in set, 1 warning (0.00 sec)

Note (Code 1003): /* select#1 */ select `test`.`st`.`id` AS `id`,`test`.`st`.`col1` AS `col1`,`test`.`st`.`col2` AS `col2`,`test`.`st`.`col3` AS `col3` from `test`.`st` where (`test`.`st`.`col1` = 2)
--- 
root@localhost:test 5.7.25-log 08:06:43> explain select * from st where col2=2\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: st
   partitions: NULL
         type: index
possible_keys: NULL
          key: test_col1_col2_col3
      key_len: 27
          ref: NULL
         rows: 3
     filtered: 33.33
        Extra: Using where; Using index
1 row in set, 1 warning (0.00 sec)

Note (Code 1003): /* select#1 */ select `test`.`st`.`id` AS `id`,`test`.`st`.`col1` AS `col1`,`test`.`st`.`col2` AS `col2`,`test`.`st`.`col3` AS `col3` from `test`.`st` where (`test`.`st`.`col2` = 2)
root@localhost:test 5.7.25-log 08:06:45>

```

观察上述两个explain结果中的type字段。查询中分别是：

1. type: index
2. type: ref

**index**：Full Index Scan，这种类型表示mysql会对整个该索引进行扫描。要想用到这种类型的索引，对这个索引并无特别要求，只要是索引，或者某个**联合索引的一部分**，mysql都可能会采用index类型的方式扫描。但是呢，缺点是效率不高，mysql会从索引中的第一个数据一个个的查找到最后一个数据，直到找到符合判断条件的某个索引。所以，上述语句会触发索引。
**ref**：使用非唯一索引或非唯一索引前缀进行的查找。这种类型表示mysql会根据特定的算法快速查找到某个符合条件的索引，而不是会对索引中每一个数据都进行一一的扫描判断，也就是所谓你平常理解的使用索引查询会更快的取出数据。而要想实现这种查找，索引却是有要求的，要实现这种能快速查找的算法，索引就要满足特定的数据结构。简单说，**也就是索引字段的数据必须是有序的，才能实现这种类型的查找，才能利用到索引。**

--



mysql的b+树索引遵循“最左前缀”原则，继续以上面的例子来说明，为了提高语句B的执行速度，我们添加了一个联合索引（username,password）,特别注意这个联合索引的顺序，如果我们颠倒下顺序改成（password,username),这样查询能使用这个索引吗？答案是不能的！这是最左前缀的第一层含义：**联合索引的多个字段中，只有当查询条件为联合索引的一个字段时，查询才能使用该索引。**
 现在，假设我们有一下三种查询情景：
 1、查出用户名的第一个字是“张”开头的人的密码。即查询条件子句为"where username like '张%'"
 2、查处用户名中含有“张”字的人的密码。即查询条件子句为"where username like '%张%'"
 3、查出用户名以“张”字结尾的人的密码。即查询条件子句为"where username like '%张'"

以上三种情况下，只有第1种能够使用（username,password）联合索引来加快查询速度。这就是最左前缀的第二层含义：**索引可以用于查询条件字段为索引字段，根据字段值最左若干个字符进行的模糊查询。**

维护索引需要代价，所以有时候我们可以利用“最左前缀”原则减少索引数量，上面的（username,password）索引，也可用于根据username查询age的情况。当然，使用这个索引去查询age的时候是需要进行回表的，当这个需求（根据username查询age）也是高频请求时，我们可以创建（username,password,age）联合索引，这样，我们需要维护的索引数量不变。

创建索引时，我们也要考虑空间代价，使用较少的空间来创建索引
 假设我们现在不需要通过username查询password了，相反，经常需要通过username查询age或通过age查询username,这时候，删掉（username,password）索引后，我们需要创建新的索引，我们有两种选择
 1、（username,age）联合索引+age字段索引
 2、（age,username）联合索引+username单字段索引
 一般来说，username字段比age字段大的多，所以，我们应选择第一种，索引占用空间较小。



### 索引覆盖

覆盖索引就是指索引中包含了查询中的所有字段，这种情况下就不需要再进行回表查询了

减少 `select *` 操作

### 索引下推

MySQL 5.6引入了索引下推优化，可以在索引遍历过程中，**对索引中包含的字段先做判断，过滤掉不符合条件的记录，减少回表字数**。

0.0

```sql
name age ID
张柳     ID3
张三     ID4
张三     ID5
张三     ID6
--- sql 1
select * from tuser where name like '张 %' and age=10 and ismale=1;



name age  ID
张柳  30   ID3
张三  10   ID4
张三  10   ID5
张三  10   ID6
--- sql 2 
select * from tuser where name like '张 %' and age=10 and ismale=1;
```

Sql 1： 在 (name,age) 索引里面特意去掉了 age 的值即{name}，**这个过程 InnoDB 并不会去看 age 的值**，只是按顺序把“name 第一个字是’张’”的记录一条条取出来回表。因此，需要回表 4 次。

Sql 2：(name,age) 索引中包含{name, age} 。InnoDB 在 (name,age) 索引内部就判断了 age 是否等于 10，对于不等于 10 的记录，直接判断并跳过。只需要对 ID4、ID5 这两条记录回表取数据判断，就只需要回表 2 次。而第一条SQL语句要回表4次。



如果没有索引下推优化（或称ICP优化），当进行索引查询时，**首先根据索引来查找记录，然后再根据where条件来过滤记录**；在支持ICP优化后，MySQL会在取出索引的同时，**判断是否可以进行where条件过滤再进行索引查询**，也就是说提前执行where的部分过滤操作，在某些场景下，可以大大减少回表次数，从而提升整体性能。





链接：https://www.jianshu.com/p/d0d3de6832b9


### session(重要)

https://www.cnblogs.com/yasmi/articles/5587868.html

### limit 1 and force index

MySQL 优化器认为在 limit 1 的情况下，走主键索引能够更快的找到那一条数据，并且如果走联合索引需要扫描索引后进行排序，而主键索引天生有序，所以优化器综合考虑，走了主键索引。

实际上，MySQL 遍历了 8000w 条数据也没找到那个天选之人(符合条件的数据)，所以浪费了很多时间。

#### 问题

```sql
select 
  * 
from 
  sample_table 
where 
    1 = 1 
    and (city_id = 565) 
    and (type = 13) 
order by 
  id desc 
--- LIMIT 5,10; // 检索记录行 6-15 
--- limit offset 0,1 即显示第一条记录
limit 
  0, 1 
```

看起来语句很简单，没什么特别的，但是每个执行的查询时间达到了惊人的 44s。

可以看到表数据量较大，预估行数在 83683240，也就是 8000w 左右，千万数据量的表...

#### 分析

Explain 用来分析 SELECT 查询语句。

Explain 比较重要的字段有：

- select_type：查询类型，有简单查询、联合查询、子查询等。
- key：使用的索引。
- rows：预计需要扫描的行数。

`explian SQL_stagement \G`，可以看到,它本来期望走索引字段的，但是经过mysql自己优化后，直接走主键了。

```bash
possible_keys: un_city_idx
key: PRIMARY
```

通过现象可以看到MySQL在order by 主键id时，limit值的大小达到了某个临界值后，改变了执行计划，选择了主键索引，但不知道具体的规则究竟是怎样。

#### 解决

临时解决：使用`force index()`强制走索引字段，但是这样的问题是，耦合性太强，万一索引名字变了，代码里的sql还得改，而且很多ORM都封装了sql，这个`force index()`也不容易加进去

**本质解决：干涉优化器选择**

https://www.modb.pro/db/77897

##### 增大limit

通过增大limit，我们可以让预估扫描行数快速增加，比如改成下面的limit 0, 1000

```sql
SELECT * FROM sample_table where city_id = 565 and type = 13 order by id desc LIMIT 0,1000
```

这样就会走上联合索引，然后排序，但是这样强行增长limit，其实总有种面向黑盒调参的感觉。我们还有更优美的解决方案吗？

#####增加包含order by id字段的联合索引

我们这句慢查询使用的是order by id，但是我们却没有在联合索引中加入id字段，导致了优化器认为联合索引后还要排序，干脆就不太想走这个联合索引了。

我们可以新建`city_id`
,`type`
和`id`
的联合索引，来解决这个问题。

这样也有一定的弊端，比如我这个表到了8000w数据，建立索引非常耗时，而且通常索引就有3.4个g，如果无限制的用索引解决问题，可能会带来新的问题。表中的索引不宜过多。

#####写成子查询

还有什么办法？我们可以用子查询，在子查询里先走city_id和type的联合索引，得到结果集后在limit1选出第一条。

但是子查询使用有风险，一版DBA也不建议使用子查询，会建议大家在代码逻辑中完成复杂的查询。当然我们这句并不复杂啦~

```sql
Select * From sample_table Where id in (Select id From `newhome_db`.`af_hot_price_region` where (city_id = 565 and type = 13)) limit 0, 1
```

end




## Mysql 排序不对

https://segmentfault.com/a/1190000016251056

```sql
--- 表定义
mysql> show create table test;
+-------+------------------------------------------------------------------------                                                           ---------------------------------------------------------------------------------                                                           ----------------+
| Table | Create Table                                                                                                                                                                                                                                                                                  |
+-------+------------------------------------------------------------------------                                                           ---------------------------------------------------------------------------------                                                           ----------------+
| test  | CREATE TABLE `test` (
  `id` int(11) NOT NULL,
  `num` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `num` (`num`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8 |
+-------+------------------------------------------------------------------------                                                           ---------------------------------------------------------------------------------                                                           ----------------+
1 row in set (0.00 sec)
--- 查看执行计划
mysql> explain select * from test\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: test
         type: index
possible_keys: NULL
          key: num
      key_len: 5
          ref: NULL
         rows: 7
        Extra: Using index
1 row in set (0.00 sec)

--- 查看执行计划
mysql> explain select * from test order by id\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: test
         type: index
possible_keys: NULL
          key: PRIMARY
      key_len: 4
          ref: NULL
         rows: 7
        Extra: NULL
1 row in set (0.01 sec)

--- 结果对比
mysql> select * from test order by id;
+----+------+
| id | num  |
+----+------+
|  3 |    3 |
|  4 |    3 |
|  5 |  555 |
| 10 |    7 |
| 14 |    9 |
| 16 |   10 |
| 20 |   20 |
+----+------+
7 rows in set (0.00 sec)

mysql> select * from test;
+----+------+
| id | num  |
+----+------+
|  3 |    3 |
|  4 |    3 |
| 10 |    7 |
| 14 |    9 |
| 16 |   10 |
| 20 |   20 |
|  5 |  555 |
+----+------+
7 rows in set (0.00 sec)


```

end

## select * 索引失效??

https://www.cnblogs.com/rjzheng/p/9950951.html

https://www.cnblogs.com/rjzheng/p/9950951.html

https://www.cnblogs.com/rjzheng/p/9950951.html

```sql
mysql> select * from test4;
+----+------+------+
| id | num  | age  |
+----+------+------+
|  1 |    3 |   11 |
|  2 |    1 |   10 |
|  3 |    7 |    9 |
|  4 |    5 |    1 |
|  5 |    4 |   19 |
+----+------+------+
5 rows in set (0.00 sec)

mysql> explain select * from test4 where num between 3 and 5 for update;                                                                    +----+-------------+-------+------+---------------+------+---------+------+------+-------------+
| id | select_type | table | type | possible_keys | key  | key_len | ref  | rows | Extra       |
+----+-------------+-------+------+---------------+------+---------+------+------+-------------+
|  1 | SIMPLE      | test4 | ALL  | num           | NULL | NULL    | NULL |    5 | Using where |
+----+-------------+-------+------+---------------+------+---------+------+------+-------------+
1 row in set (0.00 sec)
-- 使用num过滤但是显示了 age
mysql> explain select age from test4 where num between 3 and 5 for update;
+----+-------------+-------+------+---------------+------+---------+------+------+-------------+
| id | select_type | table | type | possible_keys | key  | key_len | ref  | rows | Extra       |
+----+-------------+-------+------+---------------+------+---------+------+------+-------------+
|  1 | SIMPLE      | test4 | ALL  | num           | NULL | NULL    | NULL |    5 | Using where |
+----+-------------+-------+------+---------------+------+---------+------+------+-------------+
1 row in set (0.00 sec)

mysql> explain select num from test4 where num between 3 and 5 for update;
+----+-------------+-------+-------+---------------+------+---------+------+------+--------------------------+
| id | select_type | table | type  | possible_keys | key  | key_len | ref  | rows | Extra                    |
+----+-------------+-------+-------+---------------+------+---------+------+------+--------------------------+
|  1 | SIMPLE      | test4 | range | num           | num  | 5       | NULL |    3 | Using where; Using index |
+----+-------------+-------+-------+---------------+------+---------+------+------+--------------------------+
1 row in set (0.00 sec)

mysql> explain select id,num from test4 where num between 3 and 5 for update;
+----+-------------+-------+-------+---------------+------+---------+------+------+--------------------------+
| id | select_type | table | type  | possible_keys | key  | key_len | ref  | rows | Extra                    |
+----+-------------+-------+-------+---------------+------+---------+------+------+--------------------------+
|  1 | SIMPLE      | test4 | range | num           | num  | 5       | NULL |    3 | Using where; Using index |
+----+-------------+-------+-------+---------------+------+---------+------+------+--------------------------+
1 row in set (0.00 sec)

--- 同样使用了age过滤，但是显示了非主键 num
mysql> explain select id,num from test4 where age between 3 and 15 for update;
+----+-------------+-------+------+---------------+------+---------+------+------+-------------+
| id | select_type | table | type | possible_keys | key  | key_len | ref  | rows | Extra       |
+----+-------------+-------+------+---------------+------+---------+------+------+-------------+
|  1 | SIMPLE      | test4 | ALL  | age           | NULL | NULL    | NULL |    5 | Using where |
+----+-------------+-------+------+---------------+------+---------+------+------+-------------+
1 row in set (0.00 sec)

mysql> explain select id,age from test4 where age between 3 and 15 for update;
+----+-------------+-------+-------+---------------+------+---------+------+------+--------------------------+
| id | select_type | table | type  | possible_keys | key  | key_len | ref  | rows | Extra                    |
+----+-------------+-------+-------+---------------+------+---------+------+------+--------------------------+
|  1 | SIMPLE      | test4 | range | age           | age  | 5       | NULL |    3 | Using where; Using index |
+----+-------------+-------+-------+---------------+------+---------+------+------+--------------------------+
1 row in set (0.00 sec)

--- 就离谱
mysql> select num from test2 where num between 10 and 40 for update;
+------+
| num  |
+------+
|   10 |
|   11 |
|   20 |
|   21 |
+------+
4 rows in set (0.00 sec)

mysql> explain select num from test2 where num between 10 and 40 for update;
+----+-------------+-------+-------+---------------+------+---------+------+------+--------------------------+
| id | select_type | table | type  | possible_keys | key  | key_len | ref  | rows | Extra                    |
+----+-------------+-------+-------+---------------+------+---------+------+------+--------------------------+
|  1 | SIMPLE      | test2 | range | num           | num  | 5       | NULL |    4 | Using where; Using index |
+----+-------------+-------+-------+---------------+------+---------+------+------+--------------------------+
1 row in set (0.00 sec)

mysql> explain select id from test2 where num between 10 and 40 for update;
+----+-------------+-------+-------+---------------+------+---------+------+------+--------------------------+
| id | select_type | table | type  | possible_keys | key  | key_len | ref  | rows | Extra                    |
+----+-------------+-------+-------+---------------+------+---------+------+------+--------------------------+
|  1 | SIMPLE      | test2 | range | num           | num  | 5       | NULL |    4 | Using where; Using index |
+----+-------------+-------+-------+---------------+------+---------+------+------+--------------------------+
1 row in set (0.00 sec)

mysql> explain select age from test2 where num between 10 and 40 for update;
+----+-------------+-------+------+---------------+------+---------+------+------+-------------+
| id | select_type | table | type | possible_keys | key  | key_len | ref  | rows | Extra       |
+----+-------------+-------+------+---------------+------+---------+------+------+-------------+
|  1 | SIMPLE      | test2 | ALL  | num           | NULL | NULL    | NULL |    8 | Using where |
+----+-------------+-------+------+---------------+------+---------+------+------+-------------+
1 row in set (0.00 sec)

mysql> explain select * from test2 where num between 10 and 40 for update;
+----+-------------+-------+------+---------------+------+---------+------+------+-------------+
| id | select_type | table | type | possible_keys | key  | key_len | ref  | rows | Extra       |
+----+-------------+-------+------+---------------+------+---------+------+------+-------------+
|  1 | SIMPLE      | test2 | ALL  | num           | NULL | NULL    | NULL |    8 | Using where |
+----+-------------+-------+------+---------------+------+---------+------+------+-------------+
1 row in set (0.00 sec)

```

end

## Mysql 锁

在MySQL数据库中，MyISAM引擎只支持表锁(table-level locking)，而InnoDB(row-level locking)引擎既支持表锁，也支持行锁。

### 表锁

**不会出现死锁，发生锁冲突几率高，并发低。**

MyISAM和MEMORY存储引擎采用的是表级锁（table-level locking）；BDB存储引擎采用的是页面锁（page-level locking），但也支持表级锁。

MyISAM引擎在执行select时会自动给相关表加读锁，在执行update、delete和insert时会自动给相关表加写锁。

MySQL的表级锁有两种模式：

- 表共享读锁
- 表独占写锁

```bash
## 手动显式上锁
lock table tableName read;//读锁
lock table tableName write;//写锁
## 解锁
unlock tables;//所有锁表

## 默认，自动加锁自动释放
select //读锁
insert、update、delete //上写锁
```

end



### InnoDB 行锁

`SHOW ENGINE INNODB STATUS\G` 输出信息分析

- 记录锁（LOCK_REC_NOT_GAP）: lock_mode X locks rec but not gap
- 间隙锁（LOCK_GAP）: lock_mode X locks gap before rec
- Next-key 锁（LOCK_ORNIDARY）: lock_mode X
- 插入意向锁（LOCK_INSERT_INTENTION）: lock_mode X locks gap before rec insert intention

https://www.cnblogs.com/jojop/p/13982679.html

https://www.cnblogs.com/jojop/p/13982679.html

https://www.cnblogs.com/jojop/p/13982679.html

在 RR 级别下，如果查询条件能使用上唯一索引，或者是一个唯一的查询条件，那么仅加行锁，如果是一个范围查询，那么就会给这个范围加上 gap 锁或者 next-key锁 (行锁+gap锁)。



InnoDB存储引擎的行锁是通过锁住索引实现的，而不是记录。这是理解很多数据库锁问题的关键。

由于InnoDB特殊的索引机制，数据库操作使用主键索引时，InnoDB会锁住主键索引；使用非主键索引时，InnoDB会先锁住非主键索引，再锁定主键索引。

**会出现死锁，发生锁冲突几率低，并发高。**

**InnoDB的默认加锁方式是next-key 锁。**

MySQL的行锁是通过索引加载的，也就是说，行锁是加在索引响应的行上的。当id列上没有索引时，SQL会走聚簇索引的全表扫描进行过滤，由于过滤是在MySQL Server层面进行的。因此每条记录（无论是否满足条件）都会被加上X锁。但是，为了效率考量，MySQL做了优化，对于不满足条件的记录，会在判断后放锁，最终持有的，是满足条件的记录上的锁。但是不满足条件的记录上的加锁/放锁动作是不会省略的**。所以在没有索引时，不满足条件的数据行会有加锁又放锁的耗时过程。**

InnoDB 行锁是通过给索引上的索引项加锁来实现的，这一点 MySQL 与 Oracle 不同，后者是通过在数据块中对相应数据行加锁来实现的。InnoDB 这种行锁实现的特点意味着：**只有通过索引条件检索数据，InnoDB 才使用行级锁，否则，InnoDB 将使用表锁**

#### 总结

首先，锁作用在索引上。InnoDB默认就是Next-key Lock，如果值在索引上没找到值就会退化成Gap Lock

要分析锁住了哪些行，先要列出所有的间隙

num列的记录值为3,7,9,10,20

 (-∞, 3),

 (3, 7),

 (7, 9),

（9,10）,

 (10, 20)，

 (20, +∞)

[5 ~15]的范围，先生成间隙锁(3, 20]， 由于`num=5`和 `num=15`的记录不存在，所以退化成**gap lock即(3,20)**

其次，不同索引类型生效的锁也不一样。

栗子：

table: tt(id,num),其中id为Primary key ，num为普通key。即id为聚簇索引，num为非聚簇索引且是一个普通的key

|  id  | num  |
| :--: | :--: |
|  1   |  1   |
|  3   |  3   |
|  5   |  5   |
|  7   |  7   |
|  9   |  9   |

那么`select * from test where id >= 4 and id <= 6;`， 它锁定的范围是 id_lock + num_lock.

* id_lock : 因为id_lock是pk，所以只加Record_lock即 id=5
* num_lock：因为num是普通索引，所以加Gap Lock即(3, 5)、(5,7)

这么说，懂了吧。这就是Next-Lock。针对主键和普通索引加不同的锁。

**对于主键索引id，仅仅对值为5的索引加上Record Lock（因为之前的规则）。而对于索引num，需要加上Next-Key Lock索引，锁定的范围是(3,5]。**除此之外，还会对其下一个键值加上Gap Lock，即还有一个范围为(5,7)的锁。  Next key... 有点意思

```sql
--- console 1
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from tt where id >=4 and id<=6 for update;
+----+------+
| id | num  |
+----+------+
|  5 |    5 |
+----+------+
1 row in set (0.00 sec)

mysql> explain select * from tt where id >=4 and id<=6 for update;
+----+-------------+-------+-------+---------------+---------+---------+------+------+-------------+
| id | select_type | table | type  | possible_keys | key     | key_len | ref  | rows | Extra       |
+----+-------------+-------+-------+---------------+---------+---------+------+------+-------------+
|  1 | SIMPLE      | tt    | range | PRIMARY       | PRIMARY | 4       | NULL |    1 | Using where |
+----+-------------+-------+-------+---------------+---------+---------+------+------+-------------+
1 row in set (0.00 sec)


--- console 2
mysql> begin;
Query OK, 0 rows affected (0.00 sec)
--- 失败，因为6在 (5,7)的区间
mysql> insert into tt select 6,6;
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
--- 成功
mysql> insert into tt select 8,8
    -> ;
Query OK, 1 row affected (0.00 sec)
Records: 1  Duplicates: 0  Warnings: 0
-- 成功
mysql> insert into tt select 2,2;
Query OK, 1 row affected (0.00 sec)
Records: 1  Duplicates: 0  Warnings: 0
--- 失败，因为4在(3,5) 的区间
mysql> insert into tt select 4,4;
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
mysql>


--- 插入意向锁失败
insert into tt select 4,4
------- TRX HAS BEEN WAITING 7 SEC FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 15 page no 3 n bits 80 index `PRIMARY` of table `test_db`.`tt` trx id 2750 lock_mode X locks gap before rec insert intention waiting

```

end

#### 标准行锁

InnoDB 则对表中的所有记录加锁，实际效果就和表锁一样

InnoDB存储引擎实现了如下两种标准的行级锁：

共享锁（行锁）：Shared Locks 即S锁

排它锁（行锁）：Exclusive Locks 即X锁，记录锁，间隙锁，临键锁都是排它锁。

```bash
## InnoDB引擎 默认RR隔离模式下，手动上行锁

mysql> begin;
Query OK, 0 rows affected (0.00 sec)
## 尝试锁住整个表是失败的...

mysql> select * from account lock in share mode;
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction

## 在其他中断查看原因是因为... 反正没看懂
mysql>  show engine innodb status\G;
...
select * from account lock in share mode
------- TRX HAS BEEN WAITING 4 SEC FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 8 page no 3 n bits 72 index `PRIMARY` of table `test_db`.`account` trx id 2396 lock mode S waiting
Record lock, heap no 2 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
...
## 范围也不行...
mysql> select * from account where id < 5 lock in share mode;
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction


## 行锁才可以...
## 释放锁的方式：Commit 提交、Rollback 回滚
mysql> select * from account where id = 5 lock in share mode;
+----+------+---------+
| id | name | balance |
+----+------+---------+
|  5 | dodo |      78 |
+----+------+---------+
1 row in set (0.00 sec)

## 
-- A用户对id=1的记录进行加锁
select * from user where id=1 for update;

-- B用户无法对该记录进行操作
update user set count=10 where id=1;
# 这个是可以操作的
update user set count=10 where id=2;

-- A用户commit以后则B用户可以对该记录进行操作



```

in share mode 子句的作用就是将查找的数据加上一个share锁，这个就是表示其他的事务只能对这些数据进行简单的 select 操作，而不能进行 DML 操作。

```bash
## session 1
mysql> SELECT * FROM e4 WHERE a>1 and a<7 lock in share mode;
+---+------+
| a | b    |
+---+------+
| 3 |    1 |
| 5 |    3 |
+---+------+
2 rows in set (0.00 sec)

## session 2
## 可以共享读，但是不能进行其他操作。
mysql> SELECT * FROM e4 WHERE a>1 and a<7 lock in share mode;
+---+------+
| a | b    |
+---+------+
| 3 |    1 |
| 5 |    3 |
+---+------+
2 rows in set (0.00 sec)

mysql> insert into e4 select 4,4;
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction


```

end

#### 记录锁：Record Lock（有问题）

https://www.jianshu.com/p/74f0993667f8   ???

Mysql 5.8 提供了`select ... for update nowait`

使用for update锁定行，对这行执行update,delete,select .. for update语句都会阻塞，即等待锁的释放后继续执行。

使用for update nowait锁定行，对这行执行update,delete,select .. for udapte语句，会马上返回一个“ORA-00054:resource busy”错误，不用一直等待锁的释放后继续执



行锁直接加在索引记录上面，锁住的是key。

- 在事务中启用for update（直到commit 或者rollback 此次更新操作结束 释放锁）
- mysql暂无for update nowait  需要封装，增加控制超时时间的逻辑，这样子伪nowait
- select命中索引或者主键，则为行锁，没有命中则为表锁（需要注意 避免影响业务）



A `SELECT FOR UPDATE` locks the row you selected for update until the transaction you created ends. Other transactions can only read that row but they cannot update it as long as the select for update transaction is still open.

In order to lock the row(s):

```sql
START TRANSACTION;
SELECT * FROM test WHERE id = 4 FOR UPDATE;
# Run whatever logic you want to do
COMMIT;
```

The transaction above will be alive and will lock the row until it is committed.

In order to test it, there are different ways. I tested it using two terminal instances with the MySQL client opened in each one.

On the `first terminal` you run the SQL:

```sql
START TRANSACTION;
SELECT * FROM test WHERE id = 4 FOR UPDATE;
# Do not COMMIT to keep the transaction alive
```

On the `second terminal` you can try to update the row:

```sql
UPDATE test SET parent = 100 WHERE id = 4;
```

Since you create a select for update on the `first terminal` the query above will wait until the select for update transaction is committed or it will timeout.

Go back to the `first terminal` and commit the transaction:

```sql
COMMIT;
```

Check the `second terminal` and you will see that the update query was executed (if it did not timed out)

#### 间隙锁：Gap Lock

**间隙锁是封锁索引记录中的间隔**，或者第一条索引记录之前的范围，又或者最后一条索引记录之后的范围。

**产生间隙锁的条件（仅RR事务隔离级别下；）：**

1. 使用普通索引锁定；（非唯一索引不行？？？）
2. 使用多列唯一索引；
3. 使用唯一索引锁定多行记录。

`innodb_locks_unsafe_for_binlog`：默认值为0，即启用gap lock。

```bash
mysql> show variables like 'innodb_locks_unsafe_for_binlog';
+--------------------------------+-------+
| Variable_name                  | Value |
+--------------------------------+-------+
| innodb_locks_unsafe_for_binlog | OFF   |
+--------------------------------+-------+
1 row in set (0.00 sec)

## 关闭 gap lock
[mysqld]
innodb_locks_unsafe_for_binlog = 1
```



构建表test(id pk,num key)。

pk： primary key

```sql
CREATE TABLE `test` (
  `id` int(11) NOT NULL,
  `num` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `num` (`num`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

```

并插入数据： `insert into test (id, num) values (20, 20);`

| id   | num  |
| ---- | ---- |
| 4    | 3    |
| 10   | 7    |
| 14   | 9    |
| 16   | 10   |
| 20   | 20   |

```sql
--- 唯一索引/主键+范围查询
select * from test where num between 7 and 15 for update；
--- 等价于下面的查询
select * from test where num >= 7 and num <= 15 for update;

mysql> select * from test where num between 7 and 15 for update;              
+----+------+
| id | num  |
+----+------+
| 10 |    7 |
| 14 |    9 |
| 16 |   10 |
+----+------+
3 rows in set (0.00 sec)

--- 卧槽讲究啊
--- prossible_keys： 可能选择的索引
--- key ：实际选择的索引
mysql> explain select * from test where num between 7 and 15 for update;
+----+-------------+-------+-------+---------------+------+---------+------+------+--------------------------+
| id | select_type | table | type  | possible_keys | key  | key_len | ref  | rows | Extra                    |
+----+-------------+-------+-------+---------------+------+---------+------+------+--------------------------+
|  1 | SIMPLE      | test  | range | num           | num  | 5       | NULL |    3 | Using where; Using index |
+----+-------------+-------+-------+---------------+------+---------+------+------+--------------------------+
1 row in set (0.00 sec)


```

间隙锁的触发条件必然是命中索引的，当我们查询数据用范围查询而不是相等条件查询时，**查询条件命中索引**，并且**没有查询到符合条件的记录**，此时就会将查询条件中的范围数据进行锁定(即使是范围库中不存在的数据也会被锁定)，

该句显示num=7,9,10这3行，但不仅只锁住这三行--
还会锁num=4,5,6,8,9,11,12,13,15,17,18,19这几行，  当num为3或20时，取决与id列的值。

当num为3，且id>4 时，不能被修改，id<4时，可以被修改。

而且`update test set num=6 where id=4;` 不能执行，尽管顺序没变也不行-

滚吧，写的狗屎，是保证那几行数据的整体性不被破坏就行



要分析锁住了哪些行，先要列出所有的间隙

 (-∞, 3),

 (3, 7),

 (7, 9),

（9,10）,

 (10, 20)，

 (20, +∞)

[5 ~15]的范围，先生成间隙锁(3, 20]， 由于`num=5`和 `num=15`的记录不存在，所以退化成gap lock即(3,20)

gap 锁只锁记录之间的范围，不对记录产生影响。

```sql
--- 事务A：
mysql>  select * from test where num between 5 and 15 for update;
+----+------+
| id | num  |
+----+------+
| 10 |    7 |
| 14 |    9 |
| 16 |   10 |
+----+------+

--- 事务B：
mysql> begin;
Query OK, 0 rows affected (0.00 sec)
--- 失败
mysql>  insert into test (id, num) values (5, 5);
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
mysql>  insert into test (id, num) values (6, 6);
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction

--- 成功
---  但如果先将(4,3)改成(4,2)，那么insert(3,3)就会失败
--- mysql> update test set num=2 where num=3;

mysql>  insert into test (id, num) values (3, 3);
Query OK, 1 row affected (0.00 sec)
--- 失败
mysql> update test set num=8 where id=4;
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
--- 失败
mysql>  insert into test (id, num) values (5, 3);
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
--- 失败
mysql>  insert into test (id, num) values (17, 17);
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
--- 成功
mysql>  insert into test (id, num) values (5, 555);
Query OK, 1 row affected (0.00 sec)
-- 失败
mysql>  insert into test (id, num) values (17, 20);
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
-- 成功
mysql>  insert into test (id, num) values (21, 20);
Query OK, 1 row affected (0.00 sec)
--- 失败
mysql>  update test set num=6 where id=4;
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction

--- 主键索引并没有排序....
mysql> select * from test;
+----+------+
| id | num  |
+----+------+
|  3 |    3 |
|  4 |    3 |
| 10 |    7 |
| 14 |    9 |
| 16 |   10 |
| 20 |   20 |
|  5 |  555 |
+----+------+
7 rows in set (0.00 sec)

--- \G 后面不需要加;
mysql> explain select * from test\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: test
         type: index
possible_keys: NULL
          key: num
      key_len: 5
          ref: NULL
         rows: 7
        Extra: Using index
1 row in set (0.00 sec)

ERROR:
No query specified

mysql> explain select * from test\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: test
         type: index
possible_keys: NULL
          key: num
      key_len: 5
          ref: NULL
         rows: 7
        Extra: Using index
1 row in set (0.00 sec)


--- \G 后面不需要加;
```

end

#### 临键锁：Next-key Lock

A next-key lock is a combination of a record lock on the index record and a gap lock on the gap before the index record.

Next-Key Lock 是记录上的索引的Record Lock和该索引到上一个索引之间的Gap Lock的组合。

gap就是索引树中插入新记录的空隙。相应的gap lock就是加在gap上的锁，还有一个next-key锁，是记录+记录前面的gap的组合的锁。

Next-key Lock左开右闭区间，实际上临界锁就是间隙锁加一个行锁，右闭就是行锁。

```sql
-- 事务 1
root@localhost:test 5.7.25-log 07:53:43> begin;
Query OK, 0 rows affected (0.00 sec)

--- 上锁--
--- 因为 num=20的记录确实存在，所以这里生成的是Next-key Lock 范围为 (3,20]
root@localhost:test 5.7.25-log 07:53:47> select * from test where num >=7 and num <=20 for update;
+----+------+
| id | num  |
+----+------+
| 10 |    7 |
| 14 |    9 |
| 16 |   10 |
| 20 |   20 |
+----+------+
4 rows in set (0.00 sec)
...
--- 不commit或rollback 终止事务就行
...

--- 查看锁--
SHOW ENGINE INNODB STATUS\G
...
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 1136, 1 row lock(s), undo log entries 1
MySQL thread id 39, OS thread handle 140576563775232, query id 47323583 localhost root executing
insert into test select 21,20
------- TRX HAS BEEN WAITING 76 SEC FOR THIS LOCK TO BE GRANTED:
--- lock_mode X : next-key lock
RECORD LOCKS space id 150 page no 4 n bits 80 index num of table `test`.`test` trx id 7752928 
lock_mode X insert intention waiting
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0


--- 事务 2
root@localhost:test 5.7.25-log 07:53:36> begin;
Query OK, 0 rows affected (0.00 sec)
--- 死锁超时失败
--- 如果是Gap Lock, 则插入(21, 20)会成功
root@localhost:test 5.7.25-log 07:54:04> insert into test select 21,20;
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction

--- 成功
root@localhost:test 5.7.25-log 08:13:16> insert into test select 2,3;
Query OK, 1 row affected (0.00 sec)
Records: 1  Duplicates: 0  Warnings: 0

root@localhost:test 5.7.25-log 07:59:08> rollback;
Query OK, 0 rows affected (0.00 sec)

```

end

RR级别下，加锁的基本单位就是next-key lock，（有些特定的条件下会退化成行锁或间隙锁，比如唯一索引的等值查询）。

例如一个索引有10,11,13,20这四个值。InnoDB可以根据需要使用Record Lock将10，11，13，20四个索引锁住，也可以使用Gap Lock将(-∞,10)，(10,11)，(11,13)，(13,20)，(20, +∞)五个范围区间锁住。Next-Key Locking类似于上述两种锁的结合，它可以锁住的区间有为(-∞,10]，(10,11]，(11,13]，(13,20]，(20, +∞)，可以看出它即锁定了一个范围，也会锁定记录本身。

**默认隔离级别REPEATABLE-READ下，InnoDB中行锁默认使用算法Next-Key Lock，只有当查询的索引是唯一索引或主键时，InnoDB会对Next-Key Lock进行优化，将其降级为Record Lock，即仅锁住索引本身，而不是范围。当查询的索引为辅助索引时，InnoDB则会使用Next-Key Lock进行加锁。InnoDB对于辅助索引有特殊的处理，不仅会锁住辅助索引值所在的范围，还会将其下一键值加上Gap LOCK。**

废话不多说，我们来看一下相关的实验，先做一下准备。

```sql
CREATE TABLE e4 (a INT, b INT, PRIMARY KEY(a), KEY(b));
INSERT INTO e4 SELECT 1,1;
INSERT INTO e4 SELECT 3,1;
INSERT INTO e4 SELECT 5,3;
INSERT INTO e4 SELECT 7,6;
INSERT INTO e4 SELECT 10,8;
```

 然后开启一个会话执行下面的语句。

```sql
--- Session 1
begin;
SELECT * FROM e4 WHERE b=3 FOR UPDATE; 
```

 因为通过索引b来进行查询，所以InnoDB会使用Next-Key Lock进行加锁，并且索引b是非主键索引，所以还会对主键索引a进行加锁。**对于主键索引a，仅仅对值为5的索引加上Record Lock（因为之前的规则）。而对于索引b，需要加上Next-Key Lock索引，锁定的范围是(1,3]。**除此之外，还会对其下一个键值加上Gap Lock，即还有一个范围为(3,6)的锁。 大家可以再新开一个会话，执行下面的SQL语句，会发现都会被阻塞。

```sql
--- Seesion 2
begin;
------- TRX HAS BEEN WAITING 19 SEC FOR THIS LOCK TO BE GRANTED:
--- RECORD LOCKS space id 14 page no 3 n bits 72 index `PRIMARY` of table `test_db`.`e4` trx id 2725 lock_mode X locks rec but not gap waiting
SELECT * FROM e4 WHERE a = 5 FOR UPDATE;  -- 主键a被锁


------- TRX HAS BEEN WAITING 3 SEC FOR THIS LOCK TO BE GRANTED:
--- RECORD LOCKS space id 14 page no 4 n bits 72 index `b` of table `test_db`.`e4` trx id 2725 -lock_mode X locks gap before rec insert intention waiting
INSERT INTO e4 SELECT 4,2;  -- 插入行b的值为2，在锁定的(1,3]范围内
INSERT INTO e4 SELECT 6,5;  --插入行b的值为5，在锁定的(3,6)范围内

```

 InnoDB引擎采用Next-Key Lock来解决幻读问题。因为Next-Key Lock是锁住一个范围，所以就不会产生幻读问题。但是需要注意的是，InnoDB只在Repeatable Read隔离级别下使用该机制。

end

#### 插入意向锁

首先让我们来看一下 [MySql 手册](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html#innodb-insert-intention-locks) 是如何解释 InnoDB 中的插入意向锁的：

```
An insert intention lock is a type of gap lock set by INSERT operations prior to row insertion. This lock signals the intent to insert in such a way that multiple transactions inserting into the same index gap need not wait for each other if they are not inserting at the same position within the gap. Suppose that there are index records with values of 4 and 7. Separate transactions that attempt to insert values of 5 and 6, respectively, each lock the gap between 4 and 7 with insert intention locks prior to obtaining the exclusive lock on the inserted row, but do not block each other because the rows are nonconflicting.
```

**插入意向锁**是在插入一条记录行前，由 `INSERT` 操作产生的一种**间隙锁**。该锁用以表示插入**意向**，当多个事务在**同一区间（gap）**插入位置不同的多条数据时，事务之间**不需要互相等待**。假设存在两条值分别为 4 和 7 的记录，两个不同的事务分别试图插入值为 5 和 6 的两条记录，每个事务在获取插入行上独占的（排他）锁前，都会获取（4，7）之间的间隙锁，但是因为数据行之间并不冲突，所以两个事务之间**并不会产生冲突（阻塞等待）。**

`插入意向锁`的特性可以分成两部分：

1. `插入意向锁`是一种特殊的`间隙锁`  —— `间隙锁`可以锁定**开区间**内的部分记录。
2. `插入意向锁`之间互不排斥，所以即使多个事务在同一区间插入多条记录，只要记录本身（`主键`、`唯一索引`）不冲突，那么事务之间就不会出现**冲突等待**。

需要强调的是，虽然`插入意向锁`中含有`意向锁`三个字，但是它并不属于`意向锁`而属于`间隙锁`，因为`意向锁`是**表锁**而`插入意向锁`是**行锁**。




存在普通Gap Lock时，无法插入。

```sql
--- console 1 gap lock 锁住区间
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql>  select * from test;
+----+------+
| id | num  |
+----+------+
|  4 |    3 |
| 10 |    7 |
| 14 |    9 |
| 16 |   10 |
| 20 |   20 |
+----+------+
5 rows in set (0.00 sec)

mysql>  select * from test where num between 7 and 15 for update;
+----+------+
| id | num  |
+----+------+
| 10 |    7 |
| 14 |    9 |
| 16 |   10 |
+----+------+
3 rows in set (0.00 sec)


--- console 2：insert 失败
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into test select 17,17;
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction

--- console 3: 查看锁
mysql> SHOW ENGINE INNODB STATUS\G
...
--- 插入意向锁
---TRANSACTION 2607, ACTIVE 91 sec inserting
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 360, 1 row lock(s), undo log entries 1
MySQL thread id 63, OS thread handle 0x7f496fc2f700, query id 964 localhost root executing
insert into test select 17,17
------- TRX HAS BEEN WAITING 3 SEC FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 9 page no 4 n bits 80 index `num` of table `test_db`.`test` trx id 2607 lock_mode X locks gap before rec insert intention waiting
Record lock, heap no 10 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 4; hex 80000014; asc     ;;
 1: len 4; hex 80000014; asc     ;;

------------------

...
--- 行锁
---TRANSACTION 2607, ACTIVE 217 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 1184, 1 row lock(s)
MySQL thread id 63, OS thread handle 0x7f496fc2f700, query id 968 localhost root updating
delete from test where id=10
------- TRX HAS BEEN WAITING 4 SEC FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 9 page no 3 n bits 80 index `PRIMARY` of table `test_db`.`test` trx id 2607 lock_mode X locks rec but not gap waiting
Record lock, heap no 5 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 6; hex 000000000977; asc      w;;
 2: len 7; hex d2000001690110; asc     i  ;;
 3: len 4; hex 80000007; asc     ;;

------------------
```

end

##### 流弊

如果只是使用普通的`间隙锁`会怎么样呢？还是使用我们文章开头的数据表为例：

MySql，InnoDB，Repeatable-Read：users(id PK, name, age KEY)

| id   | name | age  |
| ---- | ---- | ---- |
| 1    | Mike | 10   |
| 2    | Jone | 20   |
| 3    | Tony | 30   |

首先`事务 A` 插入了一行数据，并且没有 `commit`：

```sql
--- 插入意向锁--
INSERT INTO users SELECT 4, 'Bill', 15;
```

此时 `users` 表中存在**三把锁**：

>  流弊，Gap Lock不会作用在单列的唯一索引上

1. id 为 4 的记录行的`记录锁`。
2. age 区间在（10，15）的`间隙锁`。
3. age 区间在（15，20）的`间隙锁`。

最终，`事务 A` 插入了该行数据，并锁住了（10，20）这个区间。

随后`事务 B` 试图插入一行数据：

```sql
--- 插入意向锁 --
INSERT INTO users SELECT 5, 'Louis', 16;
```

因为 16 位于（15，20）区间内，而该区间内又存在一把`间隙锁`，所以`事务 B` 别说想申请自己的`间隙锁`了，它甚至不能获取该行的`记录锁`，自然只能乖乖的等待 `事务 A` 结束，才能执行插入操作。

很明显，这样做事务之间将会频发陷入**阻塞等待**，**插入的并发性**非常之差。

总结

1. **MySql InnoDB** 在 `Repeatable-Read` 的事务隔离级别下，使用`插入意向锁`来控制和解决并发插入。
2. `插入意向锁`是一种特殊的`间隙锁`。
3. `插入意向锁`在**锁定区间相同**但**记录行本身不冲突**的情况下**互不排斥**。

### InnoDB意向锁

InnoDB存储引擎支持一种额外的锁方式，称之为意向锁，意向锁分为：

意向共享锁（表锁）：Intention Shared Locks

意向排它锁（表锁）：Intention Exclusive Locks



https://www.cnblogs.com/zhoujinyi/p/3435982.html



```bash
mysql> mysql> SELECT * FROM INFORMATION_SCHEMA.INNODB_LOCKS;
+------------+-------------+-----------+-----------+---------------------+------------+------------+-----------+----------+-----------+
| lock_id    | lock_trx_id | lock_mode | lock_type | lock_table          | lock_index | lock_space | lock_page | lock_rec | lock_data |
+------------+-------------+-----------+-----------+---------------------+------------+------------+-----------+----------+-----------+
| 2389:8:3:2 | 2389        | X         | RECORD    | `test_db`.`account` | PRIMARY    |          8 |         3 |        2 | 1         |
| 2388:8:3:2 | 2388        | S         | RECORD    | `test_db`.`account` | PRIMARY    |          8 |         3 |        2 | 1         |
+------------+-------------+-----------+-----------+---------------------+------------+------------+-----------+----------+-----------+
2 rows in set (0.00 sec)
```



**InnoDB有三种行锁的算法：**

1，Record Lock：单个行记录上的锁。

2，Gap Lock：间隙锁，锁定一个范围，但不包括记录本身。GAP锁的目的，是为了防止同一事务的两次当前读，出现幻读的情况。

3，Next-Key Lock：1+2，锁定一个范围，并且锁定记录本身。对于行的查询，都是采用该方法，主要目的是解决幻读的问题。



```bash
## console 1
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> update account set balance=balance+30 where id=1;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> select * from account where id=1;
+----+------+---------+
| id | name | balance |
+----+------+---------+
|  1 | a    |      62 |
+----+------+---------+
1 row in set (0.00 sec)


## console 2
## 行锁？？？
mysql> update account set balance=balance+10 where id=1;

ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction

mysql> update account set balance=balance+10 where id=2;
Query OK, 1 row affected (0.01 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> update account set balance=balance+10 where id=1;
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction


```

end

### shoot？？为什么锁住了

草

```sql
mysql> show create table test2;
+-------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                                                                                                                           |
+-------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| test2 | CREATE TABLE `test2` (
  `id` int(11) NOT NULL,
  `num` int(11) DEFAULT NULL,
  `age` int(10) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `num` (`num`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8 |
+-------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)



--- console 1
mysql>  select * from test2;
+----+------+------+
| id | num  | age  |
+----+------+------+
|  4 |    3 |   20 |
| 10 |    7 |    3 |
| 14 |    9 |    6 |
| 16 |   10 |    2 |
| 20 |   20 |   20 |
+----+------+------+
5 rows in set (0.00 sec)

mysql>  select * from test2 where num between 7 and 10 for update;
+----+------+------+
| id | num  | age  |
+----+------+------+
| 10 |    7 |    3 |
| 14 |    9 |    6 |
| 16 |   10 |    2 |
+----+------+------+
3 rows in set (0.00 sec)


--- console 2: 草，为什么锁住了
mysql> insert into test2 select 21,21,21;
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
mysql> insert into test2 select 1,1,1;
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction



--- 草

mysql> explain select num from test2 where num between 7 and 10 for update;
+----+-------------+-------+-------+---------------+------+---------+------+------+--------------------------+
| id | select_type | table | type  | possible_keys | key  | key_len | ref  | rows | Extra                    |
+----+-------------+-------+-------+---------------+------+---------+------+------+--------------------------+
|  1 | SIMPLE      | test2 | range | num           | num  | 5       | NULL |    3 | Using where; Using index |
+----+-------------+-------+-------+---------------+------+---------+------+------+--------------------------+
1 row in set (0.00 sec)
--- 带* 号怎么回事
--- 直接type就是 ALL了而且没用索引。
mysql> explain select * from test2 where num between 7 and 10 for update;
+----+-------------+-------+------+---------------+------+---------+------+------+-------------+
| id | select_type | table | type | possible_keys | key  | key_len | ref  | rows | Extra       |
+----+-------------+-------+------+---------------+------+---------+------+------+-------------+
|  1 | SIMPLE      | test2 | ALL  | num           | NULL | NULL    | NULL |    5 | Using where |
+----+-------------+-------+------+---------------+------+---------+------+------+-------------+
1 row in set (0.00 sec)



mysql> show create table test3;                                                                                                             +-------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                                                                                                                           |
+-------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| test3 | CREATE TABLE `test3` (
  `id` int(11) NOT NULL,
  `num` int(11) DEFAULT NULL,
  `age` int(10) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `num` (`num`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8 |
+-------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

mysql> select * from test3;
+----+------+------+
| id | num  | age  |
+----+------+------+
|  1 |    2 |    3 |
|  2 |    1 |    7 |
|  3 |   10 |   17 |
|  4 |   17 |   17 |
+----+------+------+

--- 离谱，加上非索引列age 就不行了。。
mysql> explain select * from test3 where num >=1 and num <=15;
+----+-------------+-------+------+---------------+------+---------+------+------+-------------+
| id | select_type | table | type | possible_keys | key  | key_len | ref  | rows | Extra       |
+----+-------------+-------+------+---------------+------+---------+------+------+-------------+
|  1 | SIMPLE      | test3 | ALL  | num           | NULL | NULL    | NULL |    4 | Using where |
+----+-------------+-------+------+---------------+------+---------+------+------+-------------+
1 row in set (0.00 sec)

mysql> explain select num,id,age from test3 where num >=1 and num <=15;
+----+-------------+-------+------+---------------+------+---------+------+------+-------------+
| id | select_type | table | type | possible_keys | key  | key_len | ref  | rows | Extra       |
+----+-------------+-------+------+---------------+------+---------+------+------+-------------+
|  1 | SIMPLE      | test3 | ALL  | num           | NULL | NULL    | NULL |    4 | Using where |
+----+-------------+-------+------+---------------+------+---------+------+------+-------------+
1 row in set (0.00 sec)

mysql> explain select num,id from test3 where num >=1 and num <=15;
+----+-------------+-------+-------+---------------+------+---------+------+------+--------------------------+
| id | select_type | table | type  | possible_keys | key  | key_len | ref  | rows | Extra                    |
+----+-------------+-------+-------+---------------+------+---------+------+------+--------------------------+
|  1 | SIMPLE      | test3 | range | num           | num  | 5       | NULL |    3 | Using where; Using index |
+----+-------------+-------+-------+---------------+------+---------+------+------+--------------------------+
1 row in set (0.00 sec)

mysql> explain select age from test3 where num >=1 and num <=15;
+----+-------------+-------+------+---------------+------+---------+------+------+-------------+
| id | select_type | table | type | possible_keys | key  | key_len | ref  | rows | Extra       |
+----+-------------+-------+------+---------------+------+---------+------+------+-------------+
|  1 | SIMPLE      | test3 | ALL  | num           | NULL | NULL    | NULL |    4 | Using where |
+----+-------------+-------+------+---------------+------+---------+------+------+-------------+
1 row in set (0.00 sec)

```

end



### 死锁阅读分析+ explain

```sql

mysql> SHOW ENGINE INNODB STATUS\G
...
---TRANSACTION 2593, ACTIVE 681 sec inserting
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 1184, 1 row lock(s)
MySQL thread id 63, OS thread handle 0x7f496fc2f700, query id 899 localhost root update
insert into test2 (id, num, age) values (1,1,1)
Trx read view will not see trx with id >= 2594, sees < 2532
------- TRX HAS BEEN WAITING 14 SEC FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 10 page no 3 n bits 80 index `PRIMARY` of table `test_db`.`test2` trx id 
------ 插入意向锁--
2593 lock_mode X locks gap before rec insert intention waiting
Record lock, heap no 2 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 4; hex 80000004; asc     ;;
 1: len 6; hex 000000000a03; asc       ;;
 2: len 7; hex 2f000001980110; asc /      ;;
 3: len 4; hex 80000003; asc     ;;
 4: len 4; hex 80000014; asc     ;;

------------------
...
```

end

- 记录锁（LOCK_REC_NOT_GAP）: lock_mode X locks rec but not gap
- 间隙锁（LOCK_GAP）: lock_mode X locks gap before rec
- Next-key 锁（LOCK_ORNIDARY）: lock_mode X
- 插入意向锁（LOCK_INSERT_INTENTION）: lock_mode X locks gap before rec insert intention

- “heap no 4” 代表记录的序号，0 代表 infimum 记录、1 代表 supremum 记录，用户记录从 2 开始

这里有一点要注意的是，并不是在日志里看到 lock_mode X 就认为这是 Next-key 锁，因为还有一个例外：如果在 supremum record 上加锁，`locks gap before rec` 会省略掉，间隙锁会显示成 `lock_mode X`，插入意向锁会显示成 `lock_mode X insert intention`。譬如下面这个：

```
RECORD LOCKS space id 0 page no 307 n bits 72 index `PRIMARY` of table `test`.`test` trx id 50F lock_mode X Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
```

**看起来像是 Next-key 锁，但是看下面的 `heap no 1` 表示这个记录是 supremum record（另外，infimum record 的 heap no 为 0），所以这个锁应该看作是一个间隙锁。**

然后使用 explain 逐个语句分析，看看哪个语句没走索引直接锁了全部表。



### Mysql 5.6/7 定义锁时间

https://www.jianshu.com/p/74f0993667f8

