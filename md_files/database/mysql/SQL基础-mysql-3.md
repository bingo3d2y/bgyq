## SQL基础-mysql

## SQL语句分类

### 数据定义语言（DDL）:淦

Delete是DML，不是DDL....

Data Definition Language ， 用于定义数据结构。

DDL 用于指定数据库模式。它用于在数据库中创建表，模式，索引，约束等。让我们看看我们可以使用 DDL 在数据库上执行的操作：

- 创建数据库实例 - `CREATE`
- 改变数据库的结构 - **`ALTER`**
- 删除数据库实例 - `DROP`
- 删除数据库实例中的表 - **`TRUNCATE`**
- 重命名数据库实例 - **`RENAME`**
- 从数据库中删除对象，例如表 - **`DROP`**
- 注释 - **注释**

所有这些命令都定义或更新数据库模式，这就是它们归入数据定义语言的原因。

### 数据处理语言（DML）：即增删改查

Data Manipulation Language，用于检索或者修改数据

 `mysql `每执行一条 `DML `语句，先将记录写入 `redo log buffer `，后续某个时间点再一次性将多个操作记录写到 `redo log file `。

DML 用于访问和操作数据库中的数据。数据库的以下操作属于 DML：

- 从表中读取记录 - `SELECT`

  ```sql
   --- 有点意思哈，使用database_name.table_name可以直接查询00
   select * from `performance_schema`.replication_group_members;
   select * from `mysql`.user\G
  ```

  end

- 将记录插入表 - **`INSERT`**

- 更新表中的数据 - `UPDATE`

- 删除表中的所有记录 - `DELETE`

  delete删除记录只是给数据库中的记录加一个删除标识了，这样数据库空间并不是减少了，只是将page标记为可复用==

### 数据控制语言（DCL）

Data Control language，用于定义数据库用户权限

DCL 用于授予和撤销数据库上的用户访问权限：

- 授予用户访问权限 - `GRANT`
- 撤消用户的访问权限 - `REVOKE`

**在实际数据定义语言中，数据处理语言和数据控制语言不是单独的语言，而是它们是单个数据库语言（如 SQL）的一部分。**

### 事务控制语言（TCL）

我们使用 DML 命令进行的数据库更改是使用 TCL 执行或回滚的。

- 提交 DML 命令在数据库中所做的更改 - `COMMIT`
- 要回滚对数据库所做的更改 - `ROLLBACK`

## 默认SQL执行：隐式事务

MySQL 命令行的默认配置中事务都是自动提交的，即执行SQL语句后就会马上执行 COMMIT 操作。如果要显式地开启一个事务需要使用命令：`START TARNSACTION 或 begin`。

`begin`或 `start transaction` 都是显式开启一个事务；

`commit` 或 `commit work` 都是等价的；

`rollback` 或 `rollback work` 也是等价的；

而且Mysql还是默认自动提交事务

```bash
mysql> show global variables like 'autocommit';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| autocommit    | ON    |
+---------------+-------+
1 row in set (0.00 sec)

```

end

有时，表名或者字段名，可能用到了特殊字符（比如和系统指令重合），则可以是使用``来将字符仅当作字面量。

```sql
--- table_name like 有点问题  where a like "aa"
root@localhost:test 5.7.25-log 08:27:38> select * from `like`;                                    +----+---------+----------+---------------+
| id | user_id | liker_id | relation_ship |
+----+---------+----------+---------------+
|  1 |       1 |        1 |             7 |
+----+---------+----------+---------------+

```

end

## show

### 查看系统变量

`SHOW VARIABLES` shows the values of MariaDB system variables.

```bash
# 查询 BINLOG 格式
show VARIABLES like 'binlog_format';

# 查询 BINLOG 位置
show VARIABLES like 'datadir';

```

end

### 使用系统视图

### 管理连接

```bash
mysql> show processlist
    -> ;
+----+------+-----------+-------------+---------+-------+----------+------------------+
| Id | User | Host      | db          | Command | Time  | State    | Info             |
+----+------+-----------+-------------+---------+-------+----------+------------------+
|  4 | root | localhost | test_binlog | Sleep   | 11975 |          | NULL             |
|  5 | root | localhost | test_binlog | Sleep   |  4670 |          | NULL             |
| 11 | root | localhost | NULL        | Query   |     0 | starting | show processlist |
+----+------+-----------+-------------+---------+-------+----------+------------------+
3 rows in set (0.00 sec)

mysql> kill 4;
Query OK, 0 rows affected (0.00 sec)

mysql> kill 5;
Query OK, 0 rows affected (0.00 sec)

mysql> kill 11;
ERROR 1317 (70100): Query execution was interrupted
mysql>

```

end

### information_schema

https://zhuanlan.zhihu.com/p/88342863

## grant

```sql
CREATE USER 'sbtest'@'%' IDENTIFIED BY 'sbtest';
Query OK, 0 rows affected (0.00 sec)

root@localhost:(none) 5.7.25-log 08:51:51> use sbtest
Database changed

-- on database_name
-- grant all privileges on sbtest to 'sbtest'@'127.0.0.1' 这样也行
GRANT SELECT,INSERT,UPDATE,DELETE,CREATE,DROP,ALTER,INDEX on sbtest  to 'sbtest'@'%';
Query OK, 0 rows affected (0.00 sec)

flush privileges;

select authentication_string  from mysql.user where User='sbtest';
+-------------------------------------------+
| authentication_string                     |
+-------------------------------------------+
| *2AFD99E79E4AA23DE141540F4179F64FFB3AC521 |
+-------------------------------------------+
1 row in set (0.00 sec)

select password('sbtest');
+-------------------------------------------+
| password('sbtest')                        |
+-------------------------------------------+
| *2AFD99E79E4AA23DE141540F4179F64FFB3AC521 |
+-------------------------------------------+
1 row in set, 1 warning (0.00 sec)

--- 删除用户
drop user 'sbtest'@'%';
Query OK, 0 rows affected (0.03 sec)


```



## alert

`alter table A engine=InnoDB`,重建表空间，释放空洞的空间，缩容。 MySQL 5.6 之前要求在整个 DDL 过程中，表 A 中不能有更新。也就是说，这个 DDL 不是 Online 的。 如果在这个过程中，有新的数据要写入到表 A 的话，就会造成数据丢失。

Online DDL 之后，重建表的流程：

1. 建立一个临时文件，扫描表 A 主键的所有数据页；

2. 用数据页中表 A 的记录生成 B+ 树，存储到临时文件中；

3. 生成临时文件的过程中，将所有对 A 的操作记录在一个日志文件（row log）中

   > 重建表的过程中，允许对表 A 做增删改操作

4. 临时文件生成后，将日志文件（row log）中的操作应用到临时文件，得到一个逻辑数据上与表 A 相同的数据文件

5. 用临时文件替换表 A 的数据文件。

如果要收缩一个表，只是 delete 掉表里面不用的数据的话，表文件的大小是不会变的，你还要通过 alter table 命令重建表，才能达到表文件变小的目的。

但也又可能在执行alert之后，新表空间反而变大，可能原因：

* 这个表，本身就已经没有空洞的了
* 在重建表的时候，InnoDB 不会把整张表占满，每个页留了 1/16 给后续的更新用。
* 表重建过程中，有大量更新，又产生了空洞。5.7 支持 Online DDL

## insert

mysql插入数据有3中方式：

1. insert into 如果主键重复会报错 
2. replace into 如果主键或者唯一索引重复的话，会替换掉原数据
3. insert ignore 如果主键或则唯一索引重复，则会跳过该数据



## select

### 连接：重

因为数据库是 关系模型啊，一个个表需要通过 join来来接使用...才能将一个个实体串联起来描绘复杂的现实关系，这才能正确的反映业务逻辑。

连接运算可分为三种，分别是内连接、外连接和交叉连接（笛卡尔积）。

> 笛卡尔积： 两个集合相乘。
>
> 实例： 集合X:{A,K,Q,... 1}，13个元素和集合Y的:{spade, heart, club, diamond}，它们的笛卡尔积就是扑克牌的52个元素

例子：

```sql
 a表       id   name     b表     id     job   parent_id  
           1   张3                 1     23     1  
           2   李四                2     34     2  
           3   王武                3     34     4  
```

end

#### 内连接

仅将两个 表中满足连接条件的行组合起来作为结果集。 
在内连接中，只有在两个表中匹配的行才能在结果集中出现 

自然连接可以看作特殊的等值连接，且自然连接会去掉重复的列即`on`指定的条件连接列。

自然连接(Natural join)是一种特殊的等值连接，它要求两个关系中进行比较的分量必须是相同的属性组，并且在结果中把重复的属性列去掉。 而等值连接并不去掉重复的属性列

```sql
## 条件连接
select   a.*,b.*   from   a   inner   join   b     on   a.id=b.parent_id  
   
  结果是    
  1   张3                   1     23     1  
  2   李四                  2     34     2   

## 自然连接
## 自然连接有一个使用要求：待连接的两个表必须至少含有一个同名字段。
select   a.*,b.*   from   a   natural   join   b     on   a.id=b.parent_id  
 结果是    
 id                                  parent_id
  1   张3                      23     1
  2   李四                      34     2   
```

en

#### 外连接

##### 左连接

定义：在内连接的基础上，还包含左表中所有不符合条件的数据行，并在其中的右表列填写NULL 

左表所有记录 + 右表匹配的记录。

如果左表中的某条记录在右表中没有匹配的记录可以连接，就将左表记录与空值元组进行连接。

```sql
select   a.*,b.*   from   a   left   join   b     on   a.id=b.parent_id  
   
  结果是    
  1   张3                    1     23     1  
  2   李四                  2     34     2  
  3   王武                  null   
```

end

##### 右连接

**定义：**在内连接的基础上，还包含右表中所有不符合条件的数据行，并在其中的左表列填写NULL 

左表匹配的记录 + 右表所有记录。

如果右表中的某条记录在左表中没有匹配的记录可以连接，就将空值元组与右表记录进行连接。

SQL 语句为：

```sql
  select   a.*,b.*   from   a   right   join   b     on   a.id=b.parent_id  
   
  结果是    
  1   张3                   1     23     1  
  2   李四                 2     34     2  
  null                       3     34     4   
```

end

##### 完全连接

在内连接的基础上，还包含两个表中所有不符合条件的数据行，并在其中的左表、和右表列填写NULL 。

```sql
select   a.*,b.*   from   a   full   join   b     on   a.id=b.parent_id   

  结果是    
  1   张3                   1     23     1  
  2   李四                 2     34     2  
  null                 3     34     4  
  3   王武                 null
```

end

#### 交叉连接

交叉连接就是笛卡尔积，即左表与右表所有记录的所有连接组合，不需要加任何条件就是求笛卡尔积，对应 SQL 语句：

```sql
select * from a, b;

结果：
  1   张3                 1     23     1  
  2   李四                1     23     1
  3   王武                1     23     1
  1   张3				2     34     2
  2   李四				2     34     2
  3   王武                 2     34     2
  1   张3                   3     34     4
  2   李四				 3     34     4
  3   王武 				 3     34     4
```

end

#### limit

LIMIT 子句可以被用于强制 SELECT 语句返回指定的记录数。**LIMIT 接受一个或两个数字参数**。参数必须是一个整数常量。如果给定两个参数，**第一个参数**指定第一个返回记录行的**`偏移量`**，**第二个参数**指定**返回记录行的最大数目**

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

end

### group by

### 使用系统函数，6

```bash
## 查看版本
select version();

## mod function
select mod(101, 4);
+-------------+
| mod(101, 4) |
+-------------+
|           1 |
+-------------+
1 row in set (0.00 sec)

## concat + rand
select CONCAT('152',ROUND(RAND(1)*100000000));
+----------------------------------------+
| CONCAT('152',ROUND(RAND(1)*100000000)) |
+----------------------------------------+
| 15240540354                            |
+----------------------------------------+
1 row in set (0.00 sec)

## 当计算器
select 6291456/1024/1024;
+-------------------+
| 6291456/1024/1024 |
+-------------------+
|        6.00000000 |
+-------------------+
1 row in set (0.00 sec)

```

end

### 格式化

https://www.bbsmax.com/A/kjdw9bo2JN/

#### offset 分页

使用SELECT查询时，如果结果集数据量很大，比如几万行数据，放在一个页面显示的话数据量太大，不如分页显示，每次显示100条。

要实现分页功能，实际上就是从结果集中显示第1~100条记录作为第1页，显示第101~200条记录作为第2页，以此类推。

因此，分页实际上就是从结果集中“截取”出第M~N条记录。这个查询可以通过`LIMIT <M> OFFSET <N>`子句实现。

每页3条记录。要获取第1页的记录，可以使用`LIMIT 3 OFFSET 0`

```sql
SELECT id, name, gender, score
FROM students
ORDER BY score DESC
LIMIT 3 OFFSET 0;
```

查看第二页，此时offset应该从3开始了。

```sql
SELECT id, name, gender, score
FROM students
ORDER BY score DESC
LIMIT 3 OFFSET 3;
```

栗子

```bash
root@localhost : information_schema 08:38:27>  select CREATE_OPTIONS from TABLES limit 3 offset 0;
+----------------+
| CREATE_OPTIONS |
+----------------+
| max_rows=5461  |
| max_rows=9078  |
| max_rows=10754 |
+----------------+
3 rows in set (0.01 sec)

root@localhost : information_schema 08:38:49>  select CREATE_OPTIONS from TABLES limit 3 offset 2;
+----------------+
| CREATE_OPTIONS |
+----------------+
| max_rows=10754 |
| max_rows=349   |
| max_rows=817   |
+----------------+
3 rows in set (0.01 sec)

root@localhost : information_schema 08:38:54>  select CREATE_OPTIONS from TABLES limit 3 offset 1;
+----------------+
| CREATE_OPTIONS |
+----------------+
| max_rows=9078  |
| max_rows=10754 |
| max_rows=349   |
+----------------+
3 rows in set (0.00 sec)

root@localhost : information_schema 08:38:59>
```

end

#### 使用`\G`按行垂直显示
如果一行很长，需要这行显示的话，看起结果来就非常的难受。在SQL语句或者命令后使用G而不是分号结尾，可以将每一行的值垂直输出。这个可能也是大家对于MySQL最熟悉的区别于其他数据库工具的一个特性了

```bash
mysql> select * from db_archivelog\G
*************************** 1. row ***************************
id: 1
check_day: 2008-06-26
db_name: TBDB1
arc_size: 137
arc_num: 166
per_second: 1.6
avg_time: 8.7
```

end

#### 使用pager显示

如果select出来的结果集超过几个屏幕，那么前面的结果一晃而过无法看到。使用pager可以设置调用os的more或者less等显示查询结果，和在os中使用more或者less查看大文件的效果一样。

执行`pager more` 之后，之后执行的sql 语句都会自动more 分页。 

```bash
# 使用more
mysql> pager more
PAGER set to ‘more’
mysql> P more
PAGER set to ‘more’
# 使用less
mysql> pager less
PAGER set to ‘less’
mysql> P less
PAGER set to ‘less’
# 还原成stdout
mysql> nopager
PAGER set to stdout
```

end

#### 使用tee保存运行结果到文件

挺好用

```bash
这个类似于sqlplus的spool功能，可以将命令行中的结果保存到外部文件中。如果指定已经存在的文件，则结果会附加到文件中。
mysql> tee output.txt
Logging to file ‘output.txt’
或者
mysql> T output.txt
Logging to file ‘output.txt’
mysql> notee
Outfile disabled.
或者
mysql> t
Outfile disabled

root@localhost : information_schema 08:38:59> tee /tmp/test_tee.txt
Logging to file '/tmp/test_tee.txt'
root@localhost : information_schema 08:39:38>  select CREATE_OPTIONS from TABLES limit 7 offset 1;
+----------------+
| CREATE_OPTIONS |
+----------------+
| max_rows=9078  |
| max_rows=10754 |
| max_rows=349   |
| max_rows=817   |
| max_rows=4279  |
| max_rows=77    |
| max_rows=788   |
+----------------+
7 rows in set (0.00 sec)

root@localhost : information_schema 08:39:43> ^DBye
[root@localhost ~]# cat /tmp/test_tee.txt
root@localhost : information_schema 08:39:38>  select CREATE_OPTIONS from TABLES limit 7 offset 1;
+----------------+
| CREATE_OPTIONS |
+----------------+
| max_rows=9078  |
| max_rows=10754 |
| max_rows=349   |
| max_rows=817   |
| max_rows=4279  |
| max_rows=77    |
| max_rows=788   |
+----------------+
7 rows in set (0.00 sec)

```

end

#### 修改命令提示符

使用mysql的–prompt=选项，或者进入mysql命令行环境后使用prompt命令，都可以修改提示符

其中u表示当前连接的用户，d表示当前连接的数据库，其他更多的可选项可以参考man mysql.

有点意思

```bash
# 原连接符配置
$ cat /etc/my.cnf
...
[mysql]
no-auto-rehash
prompt="\\u@\\h : \\d \\r:\\m:\\s> "
default-character-set=utf8
show-warnings
...
[root@localhost ~]# mysql  -S "/data/mysqldata3306/sock/mysql.sock" -u root -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
...
# 使用 default PROMPT
root@localhost : (none) 08:43:40> prompt
Returning to default PROMPT of mysql>
mysql>
mysql>
mysql>
mysql>
mysql> ^DBye
```

end

### @@+system_variables

原来select + @@ + variable_name 和 show variables like "variable_name"  一个意思

```bash
mysql> show variables like "tx_isolation";
+---------------+-----------------+
| Variable_name | Value           |
+---------------+-----------------+
| tx_isolation  | REPEATABLE-READ |
+---------------+-----------------+
1 row in set (0.00 sec)

mysql> select @@tx_isolation;
+-----------------+
| @@tx_isolation  |
+-----------------+
| REPEATABLE-READ |
+-----------------+
1 row in set (0.00 sec)

mysql> SELECT @@GLOBAL.tx_isolation, @@tx_isolation, @@session.tx_isolation;
+-----------------------+-----------------+------------------------+
| @@GLOBAL.tx_isolation | @@tx_isolation  | @@session.tx_isolation |
+-----------------------+-----------------+------------------------+
| REPEATABLE-READ       | REPEATABLE-READ | REPEATABLE-READ        |
+-----------------------+-----------------+------------------------+
1 row in set (0.00 sec)


mysql> select 1;
+---+
| 1 |
+---+
| 1 |
+---+
1 row in set (0.00 sec)

mysql> select 2;
+---+
| 2 |
+---+
| 2 |
+---+
1 row in set (0.00 sec)
```

end

#### insert + select

table: tt(id,num),其中id为Primary key ，num为普通key。

```sql
mysql> insert into tt select 2,2;
Query OK, 1 row affected (0.00 sec)
Records: 1  Duplicates: 0  Warnings: 0
```

end

## explain



```sql
mysql> explain select * from user_info where id = 2\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: user_info
   partitions: NULL
   --- range 或者 all (直接锁表了)
         type: const
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 8
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0.00 sec)
```

各列的含义如下:

- id: SELECT 查询的标识符. 每个 SELECT 都会自动分配一个唯一的标识符.
- select_type: SELECT 查询的类型.
- table: 查询的是哪个表
- partitions: 匹配的分区
- type: join 类型
- possible_keys: 此次查询中可能选用的索引
- key: 此次查询中确切使用到的索引.
- ref: 哪个字段或常数与 key 一起被使用
- rows: 显示此查询一共扫描了多少行. 这个是一个估计值.
- filtered: 表示此查询条件所过滤的数据的百分比
- extra: 额外的信息

`type` 字段比较重要, 它提供了判断查询是否高效的重要依据依据. 通过 `type` 字段, 我们判断此次查询是 `全表扫描` 还是 `索引扫描` 等.

**以下全部详细解析explain各个属性含义：**

  ![clipboard.png](https://segmentfault.com/img/bVbaepd?w=729&h=47)

  **各属性含义：**
  **id:** 查询的序列号
  **select_type:** 查询的类型，主要是区别普通查询和联合查询、子查询之类的复杂查询

- `SIMPLE`：查询中不包含子查询或者`UNION`
- 查询中若包含任何复杂的子部分，最外层查询则被标记为：`PRIMARY`
- 在`SELECT`或`WHERE`列表中包含了子查询，该子查询被标记为：`SUBQUERY`

  **table:** 输出的行所引用的表
  **type:** 访问类型

  **从左至右，性能由差到好**

1. ALL: 扫描全表
2. index: 扫描全部索引树
3. range: 扫描部分索引，索引范围扫描，对索引的扫描开始于某一点，返回匹配值域的行，常见于between、<、>等的查询
4. ref: 使用非唯一索引或非唯一索引前缀进行的查找
   **（`eq_ref和const的区别：`）**
5. `eq_ref`：唯一性索引扫描，对于每个索引键，表中只有一条记录与之匹配。常见于主键或唯一索引扫描`
6. `const, system`: 单表中最多有一个匹配行，查询起来非常迅速，例如根据主键或唯一索引查询。system是const类型的特例，当查询的表只有一行的情况下， 使用system。`
7. NULL: 不用访问表或者索引，直接就能得到结果，如`select 1 from test where 1`



  **possible_keys:** 表示查询时可能使用的索引。如果是空的，没有相关的索引。这时要提高性能，可通过检验WHERE子句，看是否引用某些字段，或者检查字段不是适合索引

  **key:** 显示MySQL实际决定使用的索引。如果没有索引被选择，是NULL

  **key_len:** 使用到索引字段的长度

  注：key_len显示的值为索引字段的最大可能长度，并非实际使用长度，即key_len是根据表定义计算而得，不是通过表内检索出的。

  **ref:** 显示哪个字段或常数与key一起被使用

  **rows:** 这个数表示mysql要遍历多少数据才能找到，表示MySQL根据表统计信息及索引选用情况，估算的找到所需的记录所需要读取的行数，在innodb上可能是不准确的

  **Extra:** 执行情况的说明和描述。包含不适合在其他列中显示但十分重要的额外信息。

1. Using index：表示使用索引，如果只有 Using index，说明他没有查询到数据表，只用索引表就完成了这个查询，这个叫覆盖索引。
2. Using where：表示条件查询，如果不读取表的所有数据，或不是仅仅通过索引就可以获取所有需要的数据，则会出现 Using where。



#### type 常用类型

type 常用的取值有:

- `system`: 表中只有一条数据. 这个类型是特殊的 `const` 类型.
- `const`: 针对主键或唯一索引的等值查询扫描, 最多只返回一行数据. const 查询速度非常快, 因为它仅仅读取一次即可.
- `range`: 表示使用索引范围查询, 通过索引字段范围获取表中部分数据记录. 这个类型通常出现在 =, <>, >, >=, <, <=, IS NULL, <=>, BETWEEN, IN() 操作中.
- ALL: 表示全表扫描, 这个类型的查询是性能最差的查询之一. 通常来说, 我们的查询不应该出现 ALL 类型的查询, 因为这样的查询在数据量大的情况下, 对数据库的性能是巨大的灾难. 如一个查询是 ALL 类型查询, 那么一般来说可以对相应的字段添加索引来避免.
  下面是一个全表扫描的例子, 可以看到, 在全表扫描时, possible_keys 和 key 字段都是 NULL, 表示没有使用到索引, 并且 rows 十分巨大, 因此整个查询效率是十分低下的.
- `index`: 表示全索引扫描(full index scan), 和 ALL 类型类似, 只不过 ALL 类型是全表扫描, 而 index 类型则仅仅扫描所有的索引, 而不扫描数据.
  `index` 类型通常出现在: 所要查询的数据直接在索引树中就可以获取到, 而不需要扫描数据. 当是这种情况时, Extra 字段 会显示 `Using index`.

## order by

Mysql中的数据是按照主键的顺序来存放的。那么聚簇索引就是按照每张表的主键来构造一颗B+树，叶子节点存放的就是整张表的行数据。由于表里的数据只能按照一颗B+树排序，因此一张表只能有一个聚簇索引。

但是每次`select column_name `时看到的数据可能和`select *`不相同。

是按照主键排序的。以往我都以往mysql的排序默认是按主键来排序的。这才发现其实不是这样的。

```sql
CREATE TABLE `test` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `name` char(100) DEFAULT NULL,
  `age` char(5) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `age` (`age`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8
```

创建一个测试数据库

```sql
INSERT INTO test VALUES(NULL,'张三','5');
INSERT INTO test VALUES(NULL,'李四','15');
INSERT INTO test VALUES(NULL,'王五','5');
INSERT INTO test VALUES(NULL,'赵信','15');
INSERT INTO test VALUES(NULL,'德玛','20');
INSERT INTO test VALUES(NULL,'皇子','5');
INSERT INTO test VALUES(NULL,'木木','17');
INSERT INTO test VALUES(NULL,'好汉','22');
INSERT INTO test VALUES(NULL,'水浒','18');
INSERT INTO test VALUES(NULL,'小芳','17');
INSERT INTO test VALUES(NULL,'老王','5');
```

按照正常的主键递增的顺序插入一些数据`SELECT * FROM test LIMIT 5 `，然后查询五条记录

```sql
+----+------+------+
| id | name | age  |
+----+------+------+
|  1 | 张三     | 5    |
|  2 | 张三     | 5    |
|  3 | 李四     | 15   |
|  4 | 王五     | 5    |
|  5 | 赵信     | 15   |
+----+------+------+
5 rows in set (0.00 sec)
```

现在我们只查询两个字段，id,age `select id,age from test limit 5`;

```sql
  +----+------+
| id | age  |
+----+------+
|  3 | 15   |
|  5 | 15   |
|  8 | 17   |
| 11 | 17   |
| 10 | 18   |
+----+------+
5 rows in set (0.00 sec)
```

这个时候可以看到 两次没有使用`order by` 得到的查询结果并不一样。这个时候我们来分析下查询语句

```sql
mysql> explain select * from test limit 5 \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: test
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 12
        Extra:
1 row in set (0.00 sec)

mysql> explain select id,age from test limit 5 \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: test
         type: index
possible_keys: NULL
          key: age
      key_len: 16
          ref: NULL
         rows: 12
        Extra: Using index
1 row in set (0.00 sec)
```

可以看出，第一个查询语句是没有使用到任何的索引的，而第二个查询则是使用了`age`作为索引。

> 最后

可以看出，mysql在不给定order by条件的时候，得到的数据结果的顺序是跟查询列有关的。因为在不同的查询列的时候，可能会使用到不同的索引条件。Mysql在使用不同索引的时候，得到的数据顺序是不一样的。这个可能就跟Mysql的索引建立机制，以及索引的使用有关了。更深的东西，在这里就不深追了。为了避免这种情况，在以后的项目中，切记要加上`order by`



### replace 语句

REPLACE的工作机制有点像INSERT,只不过如果在表里如果一行有PRIMARY KEY或者UNIQUE索引，那么就会把老行删除然后插入新行。

replace的原理的最高代价是delete+insert，执行两个DML语句，对索引产生两次影响

但是，减少使用--

- 如果业务逻辑强依赖自增ID，建议不要用REPLACE
- 当存在PK冲突的时候是先DELETE再INSERT
- 当存在UK冲突的时候是直接UPDATE，UPDATE操作不会涉及到AUTO_INCREMENT的修改
- 很大程度上会导致主备中断，存在容灾风险

### insert on duplicate

nsert on duplicate的最高代价是select+update， 一个DML操作，影响一次索引--

mysql的insert操作中也给了一种方式，语法如下：

```
INSERT INTO table (a,b,c) VALUES (1,2,3)
  ON DUPLICATE KEY UPDATE c=c+1;
```

在insert时判断是否已有主键或索引重复，如果有，一句update后面的表达式执行更新，否则，执行插入。

## Mysql数据加密方式

* AES_ENCRYPT and AES_DECRYPT 一个加密一个解密，需要指定key

  ```sql
  ---
  INSERT INTO user VALUES (1, '张三', AES_ENCRYPT(('123456'), 'coco'))
  
  ---
  SELECT id, name, AES_DECRYPT(password, 'coco') as password
  FROM user
  WHERE id=1
  ```

  end

* 开发语言自带的加密库或者加密逻辑处理数据

  eg：python往数据库中插入数据时，自己用自己的加密逻辑得到一个字符串，从数据库中取出数据后，自己解密即可。

## 数据表归档: PARTITION

` explain partitions` 这都是什么东西--

```bash
#1. 创建中间表
CREATE TABLE `ota_order_2020` (........) ENGINE=InnoDB DEFAULT CHARSET=utf8
 BY RANGE (to_days(create_time)) ( 
PARTITION p201808 VALUES LESS THAN (to_days('2018-09-01')), 
PARTITION p201809 VALUES LESS THAN (to_days('2018-10-01')), 
PARTITION p201810 VALUES LESS THAN (to_days('2018-11-01')), 
PARTITION p201811 VALUES LESS THAN (to_days('2018-12-01')), 
PARTITION p201812 VALUES LESS THAN (to_days('2019-01-01')), 
PARTITION p201901 VALUES LESS THAN (to_days('2019-02-01')), 
PARTITION p201902 VALUES LESS THAN (to_days('2019-03-01')), 
PARTITION p201903 VALUES LESS THAN (to_days('2019-04-01')), 
PARTITION p201904 VALUES LESS THAN (to_days('2019-05-01')), 
PARTITION p201905 VALUES LESS THAN (to_days('2019-06-01')), 
PARTITION p201906 VALUES LESS THAN (to_days('2019-07-01')), 
PARTITION p201907 VALUES LESS THAN (to_days('2019-08-01')), 
PARTITION p201908 VALUES LESS THAN (to_days('2019-09-01')), 
PARTITION p201909 VALUES LESS THAN (to_days('2019-10-01')), 
PARTITION p201910 VALUES LESS THAN (to_days('2019-11-01')), 
PARTITION p201911 VALUES LESS THAN (to_days('2019-12-01')), 
PARTITION p201912 VALUES LESS THAN (to_days('2020-01-01')));

#2. 插入原表中有效的数据，如果数据量在100W左右可以在业务低峰期直接插入，如果比较大，建议采用dataX来做，可以控制频率和大小，之前我这边用Go封装了dataX可以实现自动生成json文件，自定义大小去执行。
insert into ota_order_2020 select * from ota_order where create_time between '2020-08-01 00:00:00' and '2020-08-31 23:59:59';

#3. 表重命名
alter table ota_order rename to ota_order_bak;  
alter table ota_order_2020 rename to ota_order;
#4. 插入差异数据
insert into ota_order select * from ota_order_bak a where not exists (select 1 from ota_order b where a.id = b.id);
#5. ota_order_bak改造成分区表，如果表比较大不建议直接改造，可以先创建好分区表，通过dataX把导入进去即可。

#6. 后续的归档方法
#创建中间普遍表
create table ota_order_mid like ota_order;
#交换原表无效数据分区到普通表
alter table ota_order exchange partition p201808 with table ota_order_mid; 
##交换普通表数据到归档表的相应分区
alter table ota_order_bak exchange partition p201808 with table ota_order_mid;
```

end

## 复杂的SQL栗子

要给“like”表增加一个字段，比如叫作 relation_ship，并设为整型，取值 1、2、3。

> 值是 1 的时候，表示 user_id 关注 liker_id;
>
> 值是 2 的时候，表示 liker_id 关注 user_id;
>
> 值是 3 的时候，表示互相关注。

然后，当 A 关注 B 的时候，逻辑改成如下所示的样子：

```sql
--- 应用代码里面，比较 A 和 B 的大小，如果 A
mysql> begin; /*启动事务*/
insert into `like`(user_id, liker_id, relation_ship) values(A, B, 1) on duplicate key update relation_ship=relation_ship | 1;
select relation_ship from `like` where user_id=A and liker_id=B;
/*代码中判断返回的 relation_ship，
  如果是1，事务结束，执行 commit
  如果是3，则执行下面这两个语句：
  */
insert ignore into friend(friend_1_id, friend_2_id) values(A,B);
commit;

----如果 A>B，则执行下面的逻辑
--- 为了保证数据都是 user_id < liker_id
mysql> begin; /*启动事务*/
insert into `like`(user_id, liker_id, relation_ship) values(B, A, 2) on duplicate key update relation_ship=relation_ship | 2;
select relation_ship from `like` where user_id=B and liker_id=A;
/*代码中判断返回的 relation_ship，
  如果是2，事务结束，执行 commit
  如果是3，则执行下面这两个语句：
*/
insert ignore into friend(friend_1_id, friend_2_id) values(B,A);
commit;

```

end

当插入已存在主键的记录时，将插入操作变为修改 relation_ship | 1 进行位运算 2|1 ==3

这个设计里，让“like”表里的数据保证 user_id < liker_id，这样不论是 A 关注 B，还是 B 关注 A，在操作“like”表的时候，如果反向的关系已经存在，就会出现行锁冲突。

> 确保两者操作同一行数据，进而通过行锁强制两者串行执行。
>
> 真流弊啊，这样一个表，A和B，通过一个relation来表示双方的关系--

然后，insert … on duplicate 语句，确保了在事务内部，执行了这个 SQL 语句后，就强行占住了这个行锁，之后的 select 判断 relation_ship 这个逻辑时就确保了是在行锁保护下的读操作。

操作符 “|” 是按位或，连同最后一句 insert 语句里的 ignore，是为了保证重复调用时的幂等性。

这样，即使在双方“同时”执行关注操作，最终数据库里的结果，也是 like 表里面有一条关于 A 和 B 的记录，而且 relation_ship 的值是 3， 并且 friend 表里面也有了 A 和 B 的这条记录。

-

这样是为了，解决并发事务问题，原则是A、B 两个用户，如果互相关注，则成为好友。

即，AB一开始都没有互相关注，然后两天同时点击了关注按钮，但是它们没有成为好友关系。

```sql
--- transaction A
begin;
select * from `like` where; (return null)
....                                              --- transaction B
                                                   begin;
Delay                                              select * from `like` where; (return null)
                                                    insert into ....
....
insert into ....                                     ....

commit;                                             commit;
```

这就导致了不和逻辑的数据。。。





## Mysql系统函数



## char and varchar

char是一种`固定长度`的类型，
varchar则是一种`可变长度`的类型，
它们的区别是：

char(M)类型的数据列里，每个值都占用M个字节，如果某个长度小于M，MySQL就会在它的右边用空格字符补足。（在检索操作中那些填补出来的空格字符将被去掉）

varchar(M)类型的数据列里，每个值只占用刚好够用的字节再加上一个用来记录其长度的字节。（即总长度为L+1字节）

在MySQL中用来判断是否需要进行对据列类型转换的规则

1、在一个数据表里，如果每一个数据列的长度都是固定的，那么每一个数据行的长度也将是固定的。

2、只要数据表里有一个数据列的长度的可变的，那么各数据行的长度都是可变的。

3、如果某个数据表里的数据行的长度是可变的，那么，为了节约存储空间，MySQL会把这个数据表里的固定长度类型的数据列转换为相应的可变长度类型。





## drop、delete 和 truncate

DELETE语句执行删除的过程是每次从表中删除一行，并且同时将该行的删除操作作为事务记录在日志中保存以便进行进行回滚操作。

TRUNCATE TABLE 则一次性地从表中删除所有的数据并不把单独的删除操作记录记入日志保存，删除行是不能恢复的。并且在删除的过程中不会激活与表有关的删除触发器。执行速度快。

drop语句将表所占用的空间全释放掉。

如果想删除表，当然用drop；  

如果想保留表而将所有数据删除，如果和事务无关，用truncate即可；

 如果和事务有关，或者想触发trigger，还是用delete； 如果是整理表内部的碎片，可以用truncate跟上reuse stroage，再重新导入/插入数据。



另外：delete与truncate还有以下的不同：

1) truncate和 delete只删除数据不删除表的结构(定义) 

2) delete语句是dml,这个操作会放到rollback segement中,事务提交之后才生效;如果有相应的trigger,执行的时候将被触发. 
truncate是ddl, 操作立即生效,原数据不放到rollback segment中,不能回滚. 操作不触发trigger. 

3) delete语句不影响表所占用的extent, 高水线(high watermark)保持原位置不动 
显然drop语句将表所占用的空间全部释放 
truncate 语句缺省情况下见空间释放到 minextents个 extent,除非使用reuse storage; truncate 会将高水线复位(回到最开始). 

4) 速度,一般来说: truncate > delete 

5)安全性:小心使用drop 和truncate,尤其没有备份的时候.否则哭都来不及. 

使用上,想删除部分数据行用delete,注意带上where子句. 回滚段要足够大. 
想保留表而将所有数据删除. 如果和事务无关,用truncate即可. 如果和事务有关,或者想触发trigger,还是用delete. 
如果是整理表内部的碎片,可以用truncate跟上reuse stroage,再重新导入/插入数据/

## 正确Delete table 姿势

```sql
   INSERT INTO New
      SELECT * FROM Main
         WHERE ...;  -- just the rows you want to keep
   RENAME TABLE main TO Old, New TO Main;
   DROP TABLE Old;   -- Space freed up here
```

end





## Mysql 不建议delete删除表

这里需要知道一些 undo和redo日志的知识。

The bigger the table (number of and size of columns) the more expensive it becomes to delete and insert rather than update. Because you have to pay the price of UNDO and REDO. DELETEs consume more UNDO space than UPDATEs, and your REDO contains twice as many statements as are necessary.

通过从InnoDB存储空间分布，delete对性能的影响可以看到，delete物理删除既不能释放磁盘空间，而且会产生大量的碎片，导致索引频繁分裂，影响SQL执行计划的稳定性；

同时在碎片回收时，会耗用大量的CPU，磁盘空间，影响表上正常的DML操作。

在业务代码层面，应该做逻辑标记删除，避免物理删除；为了实现数据归档需求，可以用采用MySQL分区表特性来实现，都是DDL操作，没有碎片产生。

truncate是ddl, 操作立即生效,原数据不放到rollback segment中,不能回滚. 操作不触发trigger. 



敖丙，这篇mysql 不建议delete删除表讲的很好。

https://cloud.tencent.com/developer/article/1751646

### delete + optimize

在OPTIMIZE TABLE运行过程中，MySQL会锁定表。因此，这个操作一定要在网站访问量较少的时间段进行。

```bash
## 原表空间大小
[root@localhost ~]# du -sh /data/mysql/mysqldata3316/mydata/test/user.ibd
10M     /data/mysql/mysqldata3316/mydata/test/user.ibd

## 执行delete语句
delete from   user where id < 100000;
Query OK, 49999 rows affected (0.15 sec)
## 查看表空间，大小不变
[root@localhost ~]# du -sh /data/mysql/mysqldata3316/mydata/test/user.ibd
10M     /data/mysql/mysqldata3316/mydata/test/user.ibd

## 执行 optimize
root@localhost:test 5.7.25-log 04:06:00> optimize table user;
+-----------+----------+----------+-------------------------------------------------------------------+
| Table     | Op       | Msg_type | Msg_text                                                          |
+-----------+----------+----------+-------------------------------------------------------------------+
| test.user | optimize | note     | Table does not support optimize, doing recreate + analyze instead |
| test.user | optimize | status   | OK                                                                |
+-----------+----------+----------+-------------------------------------------------------------------+
2 rows in set (0.03 sec)


## 表空间释放
[root@localhost ~]# du -sh /data/mysql/mysqldata3316/mydata/test/user.ibd
96K     /data/mysql/mysqldata3316/mydata/test/user.ibd

```



## 视图

Mysql 不怎么使用视图！！ 因为在veiw的功用被ORM所替代了。

显示所有视图

https://bingslient.github.io/2019/08/16/MySQL%20%E6%95%B0%E6%8D%AE%E5%BA%93%E7%B3%BB%E7%BB%9F%E8%A1%A8%E7%9A%84%E5%88%A9%E7%94%A8/#%E7%94%A8%E6%88%B7%E5%AF%86%E7%A0%81%E7%88%86%E7%A0%B4



mysql的视图不是一种物化视图，它相当于一个虚拟表，本身并不存储数据，当sql在操作视图时所有数据都是从其他表中查出来的。这带来的问题是使用视图并不能将常用数据分离出来，优化查询速度。且操作视图的很多命令都与普通表一样，这会导致在业务代码中无法通过sql区分表和视图，使代码变得复杂。
实现视图的方式有两种，分别为合并算法和临时表算法，合并算法是指查询视图时将视图定义的sql合并到查询sql中，比如create view v1 as select * from user where sex=m;当我们要查询视图时，mysql会将select id,name from v1;合并成select id,name from user where sex=m……;临时表算法是先将视图查出来的数据保存到一个临时表中，查询的时候查这个临时表。不管是合并算法和临时表算法都会带来额外的开销，;且如果使用临时表后会使mysql的优化变得很困难，比如索引。
而且视图还引入了一些其他的问题，使得其背后的逻辑非常复杂。
当然，视图在某些情况下可以帮助提升性能，但视图的性能很难预测。且在mysql的优化器中，视图的代码执行路径也完全不同，无法直观的预测其执行性能。

创建视图标准语法：

```sql
CREATE
    [OR REPLACE]
    [ALGORITHM = {UNDEFINED | MERGE | TEMPTABLE}]
    [DEFINER = user]
    [SQL SECURITY { DEFINER | INVOKER }]
    VIEW view_name [(column_list)]
    AS select_statement
    [WITH [CASCADED | LOCAL] CHECK OPTION]
```

语法解读：

1）OR REPLACE：表示替换已有视图，如果该视图不存在，则CREATE OR REPLACE VIEW与CREATE VIEW相同。

2）ALGORITHM：表示视图选择算法，默认算法是UNDEFINED(未定义的)：MySQL自动选择要使用的算法 ；merge合并；temptable临时表，一般该参数不显式指定。

3）DEFINER：指出谁是视图的创建者或定义者，如果不指定该选项，则创建视图的用户就是定义者。

4）SQL SECURITY：SQL安全性，默认为DEFINER。

5）select_statement：表示select语句，可以从基表或其他视图中进行选择。

6）WITH CHECK OPTION：表示视图在更新时保证约束，默认是CASCADED。

视图作为一个访问接口，不管基表的表结构和表名有多复杂。一般情况下视图只用于查询，视图本身没有数据，因此对视图进行的dml操作最终都体现在基表中，对视图进行delete、update、insert操作，原表同样会更新，drop视图原表不会变，视图不可以truncate。但是一般情况下我们要避免更新视图，dml操作可以直接对原表进行更新。

```sql
root@localhost:test 5.7.25-log 08:09:34> select * from old;
+----+------+---------+
| id | name | balance |
+----+------+---------+
|  1 | dodo |      23 |
|  2 | do   |     232 |
+----+------+---------+
2 rows in set (0.00 sec)

root@localhost:test 5.7.25-log 08:09:41> create view test_view(name, balance)
    -> as
    -> select name, balance from old
    -> where id < 5
    -> WITH CHECK OPTION;
Query OK, 0 rows affected (0.08 sec)

root@localhost:test 5.7.25-log 08:10:31> desc test_view;
+---------+--------------+------+-----+---------+-------+
| Field   | Type         | Null | Key | Default | Extra |
+---------+--------------+------+-----+---------+-------+
| name    | varchar(255) | YES  |     | NULL    |       |
| balance | int(11)      | YES  |     | NULL    |       |
+---------+--------------+------+-----+---------+-------+
2 rows in set (0.04 sec)

root@localhost:test 5.7.25-log 08:10:39> select * from test_view;
+------+---------+
| name | balance |
+------+---------+
| dodo |      23 |
| do   |     232 |
+------+---------+
2 rows in set (0.00 sec)

--- 单表没什么意义

root@localhost:test 5.7.25-log 08:18:41> create view test_old_and_new_view
    -> as
    -> select a.name,a.balance,b.age from old a, new b where a.id=b.id;
Query OK, 0 rows affected (0.00 sec)

root@localhost:test 5.7.25-log 08:20:25> desc test_old_and_new_view;
+---------+--------------+------+-----+---------+-------+
| Field   | Type         | Null | Key | Default | Extra |
+---------+--------------+------+-----+---------+-------+
| name    | varchar(255) | YES  |     | NULL    |       |
| balance | int(11)      | YES  |     | NULL    |       |
| age     | varchar(255) | YES  |     | NULL    |       |
+---------+--------------+------+-----+---------+-------+
3 rows in set (0.00 sec)

root@localhost:test 5.7.25-log 08:20:32> select * from test_old_and_new_view;
+------+---------+--------+
| name | balance | age    |
+------+---------+--------+
| dodo |      23 | male   |
| do   |     232 | female |
+------+---------+--------+
2 rows in set (0.00 sec)

root@localhost:test 5.7.25-log 08:20:39>

--- 这些视图像表一样--
show tables;
+-----------------------+
| Tables_in_test        |
+-----------------------+
| like                  |
| new                   |
| old                   |
| st                    |
| st_2                  |
| table_name            |
| test_old_and_new_view |
| test_view             |
| user                  |
+-----------------------+
9 rows in set (0.00 sec)

```

end

## 函数

Mysql使用`call`指令调用 函数和 存储过程。

```sql
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

call insert_user_data(10000)
DELIMITER ;


```

end

## 存储过程

```sql
create table user(id bigint not null primary key auto_increment, 
name varchar(20) not null default '' comment '姓名', 
age tinyint not null default 0 comment 'age', 
gender char(1) not null default 'M'  comment '性别',
phone varchar(16) not null default '' comment '手机号',
create_time datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
update_time datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '修改时间'
) engine = InnoDB DEFAULT CHARSET=utf8mb4 COMMENT '用户信息表';


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

call insert_user_data(10000)


--- 好玩
CREATE PROCEDURE insert_geek_data(num INTEGER) 
BEGIN
    DECLARE v_i int unsigned DEFAULT 0;
set autocommit= 0;
WHILE v_i < num DO
   insert into geek(a, b, c, d) values (v_i, mod(v_i,3)+3, mod(v_i,3), mod(v_i,120));
 SET v_i = v_i+1;
END WHILE;
commit;
END $$
```



工作里会经常遇到重复性的工作，这时候就可以把常用的SQL写好存储起来，这就是存储过程。

　存储过程（Stored Procedure）是一组为了完成特定功能的SQL 语句集，经编译后存储在数据库。用户通过指定存储过程的名字并给出参数（如果该存储过程带有参数）来执行它。

* 存储过程只在创造时进行编译，以后每次执行存储过程都不需再重新编译，而
  一般SQL 语句每执行一次就编译一次,所以使用存储过程可提高数据库执行速
  度。
* 当对数据库进行复杂操作时(如对多个表进行
  Update,Insert,Select,Delete 时），可将此复杂操作用存储过程封装起来
  与数据库提供的事务处理结合一起使用。
* 存储过程可以重复使用,可减少数据库开发人员的工作量。
* 安全性高,可设定只有某个用户才具有对指定存储过程的使用权。

```sql
CREATE PROCEDURE sp_name ([proc_parameter[,...]])
  [characteristic ...] routine_body
  
CREATE FUNCTION sp_name ([func_parameter[,...]])
  RETURNS type
  [characteristic ...] routine_body
   
  proc_parameter:
  [ IN | OUT | INOUT ] param_name type
   
  func_parameter:
  param_name type
  
type:
  Any valid MySQL data type
  
characteristic:
  LANGUAGE SQL
 | [NOT] DETERMINISTIC
 | { CONTAINS SQL | NO SQL | READS SQL DATA | MODIFIES SQL DATA }
 | SQL SECURITY { DEFINER | INVOKER }
 | COMMENT 'string'
  
routine_body:
  Valid SQL procedure statement or statements
```

主要解释characteristic部分：

LANGUAGE SQL：用来说明语句部分是SQL语句，未来可能会支持其它类型的语句。

[NOT] DETERMINISTIC：如果程序或线程总是对同样的输入参数产生同样的结果，则被认为它是“确定的”，否则就是“非确定”的。如果既没有给定DETERMINISTIC也没有给定NOT DETERMINISTIC，默认的就是NOT DETERMINISTIC（非确定的）CONTAINS SQL：表示子程序不包含读或写数据的语句。

NO SQL：表示子程序不包含SQL语句。

READS SQL DATA：表示子程序包含读数据的语句，但不包含写数据的语句。

MODIFIES SQL DATA：表示子程序包含写数据的语句。

SQL SECURITY DEFINER：表示执行存储过程中的程序是由创建该存储过程的用户的权限来执行。

SQL SECURITY INVOKER：表示执行存储过程中的程序是由调用该存储过程的用户的权限来执行。（例如上面的存储过程我写的是由调用该存储过程的用户的权限来执行，当前存储过程是用来查询Employee表，如果我当前执行存储过程的用户没有查询Employee表的权限那么就会返回权限不足的错误，如果换成DEFINER如果存储过程是由ROOT用户创建那么任何一个用户登入调用存储过程都可以执行，因为执行存储过程的权限变成了root）

COMMENT 'string'：备注，和创建表的字段备注一样。

**注意：**在编写存储过程和函数时建议明确指定上面characteristic部分的状态，特别是存在复制的环境中，如果创建函数不明确指定这些状态会报错，从一个非复制环境将带函数的数据库迁移到复制环境的机器上如果没有明确指定DETERMINISTIC, NO SQL, or READS SQL DATA该三个状态也会报错。

emm ，这个`delimiter`指令还挺好用

```sql
-- 将结束标志符更改为//
delimiter //
 
-- 创建存储过程
create procedure proce_user_count(OUT count_num INT)
reads sql data
begin
	select count(*) into count_num from tb_user;
end
//
 
-- 将结束标志符更改回分号
delimiter ;

```

end



## Mysql系统表

https://cloud.tencent.com/developer/article/1340819

mysql库的user表，存储了用户的连接信息： `select * from mysql.user\G`

统计表空间大小和索引大小，利有`information_schema`库，还是有不少误差的

因为这个表索引多，所以索引消耗空间比表的数据还多。

```sql
root@localhost:test 5.7.25-log 08:01:10> SELECT TABLE_NAME,TABLE_ROWS,CONCAT(ROUND((DATA_LENGTH)/1024/1024), "MB") as DATA_SIZE, CONCAT(ROUND((INDEX_LENGTH)/1024/1024), "MB") as INDEX_SIZE, CONCAT(ROUND((DATA_LENGTH+INDEX_LENGTH)/1024/1024), "MB") as SUM_SIZE FROM information_schema.TABLES WHERE TABLE_SCHEMA='test' AND TABLE_NAME='geek';
+------------+------------+-----------+------------+----------+
| TABLE_NAME | TABLE_ROWS | DATA_SIZE | INDEX_SIZE | SUM_SIZE |
+------------+------------+-----------+------------+----------+
| geek       |    9733991 | 347MB     | 497MB      | 844MB    |
+------------+------------+-----------+------------+----------+
1 row in set (0.00 sec)

root@localhost:test 5.7.25-log 08:00:47> select count(*) from test.geek;
+----------+
| count(*) |
+----------+
| 10000000 |
+----------+
1 row in set (1.53 sec)


--- 误差应该来源于 talbes_rows 不够10000000
[root@localhost ~]# du -sh /data/mysql/mysqldata3316/mydata/test/geek.ibd
885M    /data/mysql/mysqldata3316/mydata/test/geek.ibd

```

查看数据库大小

```sql
use 数据库名
SELECT sum(DATA_LENGTH)+sum(INDEX_LENGTH)
FROM information_schema.TABLES where TABLE_SCHEMA='数据库名';
```



很强，统计一个表使用的各个物理存储大小

```sql
CREATE TABLE `table_name` (
 
id bigint(20) not null auto_increment,
 
detail varchar(2000),
 
createtime  datetime,
 
validity int default '0',
 
primary key (id)
 
);


SELECT CONCAT(table_schema,'.',table_name) AS 'Table Name', 
CONCAT(truncate(table_rows/1000000,2),'M') AS 'Number of Rows', 
CONCAT(truncate(data_length/(1024*1024*1024),2),'G') AS 'Data Size', 
CONCAT(truncate(index_length/(1024*1024*1024),2),'G') AS 'Index Size' , 
CONCAT(truncate((data_length+index_length)/(1024*1024*1024),2),'G') AS  'Total' FROM information_schema.TABLES WHERE table_schema LIKE 'metadata';
```

end

**`TABLE_ROWS`**

The number of rows. Some storage engines, such as `MyISAM`, store the exact count. For other storage engines, such as `InnoDB`, this value is an approximation, and may vary from the actual value by **as much as 40% to 50%. In such cases, use `SELECT COUNT(\*)` to obtain an accurate count.**

`TABLE_ROWS` is `NULL` for `INFORMATION_SCHEMA` tables.

For `InnoDB` tables, the row count is only a rough estimate used in SQL optimization. (This is also true if the `InnoDB` table is partitioned.)



解释的意思，就是**针对 MyISAM引擎的表，行数是确定的值；**

但**针对InnoDB引擎来说（我们平常的库都是用这个引擎），行数就是个大概值**，误差最大可能会差距在40%-50%的，所以还是用count（*）统计其真实行数。

```sql
root@localhost:information_schema 5.7.25-log 05:47:04> SELECT TABLE_NAME,DATA_LENGTH+INDEX_LENGTH,TABLE_ROWS FROM TABLES WHERE TABLE_SCHEMA='test' AND TABLE_NAME='geek';
+------------+--------------------------+------------+
| TABLE_NAME | DATA_LENGTH+INDEX_LENGTH | TABLE_ROWS |
+------------+--------------------------+------------+
| geek       |                885260288 |    9733991 |
+------------+--------------------------+------------+
1 row in set (0.00 sec)
--- 确实差了很多
root@localhost:information_schema 5.7.25-log 05:47:51> use test;
Database changed
root@localhost:test 5.7.25-log 05:48:02> select count(*) from geek;
+----------+
| count(*) |
+----------+
| 10000000 |
+----------+
1 row in set (1.58 sec)

```

end