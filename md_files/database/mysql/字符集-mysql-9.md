### 字符集-mysql-9

### 懂了

Unicode是字库表的一个编码字符集.... 纯数学概念。

UTF8是对编码字符集的一种编码手段，所以叫作... 字符编码。

### 字符集

字符的集合就叫字符集。不同集合支持的字符范围自然也不一样，譬如ASCII只支持英文，GB18030支持中文等等



在字符集中，有一个码表的存在，每一个字符在各自的字符集中对应着一个唯一的码。但是同一个字符在不同字符集中的码是不一样的，譬如字符“中”在Unicode和GB18030中就分别对应着不同的码(`20013`与`54992`)。（纯数学概念，至于字符编码是将这些数字用编码手段方便在计算机存储）

字符集只是一个规则集合的名字，对应到真实生活中，字符集就是对某种语言的称呼。例如：英语，汉语，日语。对于一个字符集来说要正确编码转码一个字符需要三个关键元素：字库表（character repertoire）、编码字符集（coded character set）、字符编码（character encoding form）。

举个栗子：

有四个字符：A、B、a、b，这四个字符的编码分别是A = 0, B = 1, a = 2, b = 3。这里的字符 + 编码就构成了字符集(character set)。

### 字库表：现实字符

character repertoire

字库表是一个相当于所有可读或者可显示字符的数据库，字库表决定了整个字符集能够展现表示的所有字符的范围。

### 编码字符集/CCS：内存映射/计算机识别

coded character set，即字库表里的抽象字符编上一个数字，也就是字库表集合到一个整数集合的映射。跟计算机里的什么进制啊没有任何关系，**它是完全数学的抽象的。**

编码字符集，即用一个编码值`code point`来表示一个字符在字库中的位置。

不同的编码字符集可以表示的字符数量不同。

eg: GBK即中文汉字字符集、ASCII 常见的英文和数字字符集。 Unicode 也是属于这层概念。

例如在ASCII中`A`在表中排第65位，而编码后`A`的数值是`0100 0001`也即十进制的65的二进制转换结果。 

比如，在Unicode中每个字符对应的数字，称为一个码点，code poin

### 字符编码：存储传输

字符编码，定义字符集中的字符如何编码为特定的二进制数，以便在计算机中存储。

将 CCS 里字符对应的**整数转换成有限长度的比特值**，便于以后计算机使用一定长度的二进制形式表示该整数。这个对应关系被称为**字符编码表**（CEF:Character Encoding Form）UTF-8, UTF-16 都属于这层。

> 将编码字符集再转成方便计算机存储和计算的二进制数。
>
> 太好理解了，字库表即现实世界需要用到的全部期望字符，然后有各种编码字符集用于将字库表的字符用数字标识，用于计算机在处理时识别。

`字库表`和`编码字符集`看来是必不可少的，那既然字库表中的每一个字符都有一个自己的序号，直接把序号作为存储内容就好了。为什么还要多此一举通过`字符编码`把序号转换成另外一种存储格式呢？其实原因也比较容易理解：统一字库表的目的是为了能够涵盖世界上所有的字符，但实际使用过程中会发现真正用的上的字符相对整个字库表来说比例非常低。例如中文地区的程序几乎不会需要日语字符，而一些英语国家甚至简单的ASCII字库表就能满足基本需求。而如果把每个字符都用字库表中的序号来存储的话，每个字符就需要3个字节（这里以Unicode字库为例），这样对于原本用仅占一个字符的ASCII编码的英语地区国家显然是一个额外成本（存储体积是原来的三倍）。

算的直接一些，同样一块硬盘，用ASCII可以存1500篇文章，而用3字节Unicode序号存储只能存500篇。于是就出现了UTF-8这样的变长编码。在UTF-8编码中原本只需要一个字节的ASCII字符，仍然只占一个字节。而像中文及日语这样的复杂字符就需要2个到3个字节来存储。

字符编码相当于一个中间件，提高了空间利用率和编码解码效率。

### 栗子/总结：

编码字符集是计算机用来识别和标志现实世界字符的集合（这个完全是个数学概念），即它在计算机中内存中出现。字符编码，用来解决编码字符集的存储问题，即它出现在计算机的硬盘中。

用记事本编辑的时候，从文件（电脑硬盘）读取的UTF-8字符被转换为Unicode字符到内存里，编辑完成后，保存的时候再把Unicode转换为UTF-8保存到文件。



### Unicode

Unicode 源于一个很简单的想法：将全世界所有的字符包含在一个集合里，计算机只要支持这一个字符集，就能显示所有的字符，再也不会有乱码了。

Unicode从 0 开始，为每个符号指定一个编号，这叫做”码点”（**code point**）。比如，码点 0 的符号就是 null（表示所有二进制位都是 0）。

```
// 一个十六进制的数可以转成4位的二进制
U+0000 = null
// first plan 
U+0000 ~ U+FFFF
```

上式中，U+表示紧跟在后面的十六进制数是 Unicode 的码点。

Unicode 不是一次性定义的，而是分区定义。**每个区可以存放 65536 个（`2^16`）字符，称为一个平面（plane）。目前，一共有 17 个平面**，也就是说，整个 Unicode 字符集的大小现在是 `2^21`。

**最前面的 65536 个字符位，称为基本平面（缩写 BMP），它的码点范围是从 0 一直到 `2^16-1`，写成 16 进制就是从 U+0000 到 U+FFFF。所有最常见的字符都放在这个平面**，这是 Unicode 最先定义和公布的一个平面。

**剩下的字符都放在辅助平面（缩写 SMP），码点范围从 U+010000 一直到 U+10FFFF**。

Unicode 只规定了每个字符的码点，到底用什么样的字节序表示这个码点，就涉及到编码方法。

UTF-8就是字符编码，即Unicode规则字库的一种实现形式，而且UTF8只实现了第一个Plane。



### UTF8

UTF8 完美 实现了对ASSII码的向后兼容。

**UTF-8 是目前互联网上使用最广泛的一种 Unicode 编码方式**，它的最大特点就是**可变长。它可以使用 1 - 4 个字节表示一个字符，根据字符的不同变换长度**。编码规则如下：

1. 对于单个字节的字符，第一位设为 0，后面的 7 位对应这个字符的 Unicode 码点。因此，对于英文中的 0 - 127 号字符，与 ASCII 码完全相同。这意味着 ASCII 码那个年代的文档用 UTF-8 编码打开完全没有问题。
2. 对于需要使用 N 个字节来表示的字符（N > 1），第一个字节的前 N 位都设为 1，第 N + 1 位设为 0，剩余的 N - 1 个字节的前两位都设位 10，剩下的二进制位则使用这个字符的 Unicode 码点来填充。

真秒啊UTF8

```bash
Unicode符号范围     |        UTF-8编码方式
(十六进制)        |              （二进制）
----------------------+---------------------------------------------
0000 0000-0000 007F | 0xxxxxxx
0000 0080-0000 07FF | 110xxxxx 10xxxxxx
0000 0800-0000 FFFF | 1110xxxx 10xxxxxx 10xxxxxx
0001 0000-0010 FFFF | 11110xxx 10xxxxxx 10xxxxxx 10xxxxxx
```



比如，淦字需要三个字节标识，且码点为FA97，则utf8的编码为`1110XXXX 10XXXXXX 10XXXXXX` ，把码点的十六进制转成二进制即可。

### UTF8mb4

如果在移动端发布文本内容时包含了这种 Emoji 表情符号，通过接口传递到服务器端，服务器端再存入 MySQL 数据库：

- 对 gbk 字符集的数据库，写入数据库的数据，在回显时，变成 ‘口口’ 无法回显；
- 对 utf8 字符集的数据库，则根本无法写入数据库，程序直接报出异常信息 java.io.exception xxxxxxxx.

原因分析：

这是由于字符集不支持的异常，因为 **Emoji 表情是四个字节，而 mysql 的 utf-8 编码最多三个字节，所以导致数据插不进去**。

**真正的 utf8 编码(大家都使用的标准)，最大支持 4 个 bytes。正是由于 mysql 的 utf8 少一个 byte，导致中文的一些特殊字符和 emoji 都无法正常的显示**。**mysql 真正的 utf8 其实是 utf8mb4，这是在 5.5.3 版本之后加入的。而目前的“utf8”其实是 utf8mb3。所以尽量不要使用默认的 utf8，使用 utf8mb4 才是正确的选择**。

从 mysql 5.5.3 之后版本基本可以无缝升级到 utf8mb4 字符集。同时，utf8mb4 兼容 utf8 字符集，utf8 字符的编码、位置、存储在 utf8mb4 与 utf8 字符集里一样的，不会对有现有数据带来损坏。

Emoji字符表情会对我们平时的开发运维带来什么影响呢？最常见的问题就在于将他存入MySQL数据库的时候。一般来说MySQL数据库的默认字符集都会配置成UTF-8（三字节），而utf8mb4在5.5以后才被支持，也很少会有DBA主动将系统默认字符集改成utf8mb4。那么问题就来了，当我们把一个需要4字节UTF-8编码才能表示的字符存入数据库的时候就会报错：`ERROR 1366: Incorrect string value: '\xF0\x9D\x8C\x86' for column` 。 如果认真阅读了上面的解释，那么这个报错也就不难看懂了。我们试图将一串Bytes插入到一列中，而这串Bytes的第一个字节是`\xF0`意味着这是一个四字节的UTF-8编码。但是当MySQL表和列字符集配置为UTF-8的时候是无法存储这样的字符的，所以报了错。 那么遇到这种情况我们如何解决呢？有两种方式：升级MySQL到5.6或更高版本，并且将表字符集切换至utf8mb4。第二种方法就是在把内容存入到数据库之前做一次过滤，将Emoji字符替换成一段特殊的文字编码，然后再存入数据库中。之后从数据库获取或者前端展示时再将这段特殊文字编码转换成Emoji显示。





### 使用Mysql指定字符集解码

```bash

```

end

### Mysql 字符集和字符序(看)

在数据的存储上，MySQL提供了不同的字符集支持。而在数据的对比操作上，则提供了不同的字符序支持。MySQL提供了不同级别的设置，包括server级、database级、table级、column级，可以提供非常精准的设置。

什么是字符集、字符序?简单的来说：

- 字符集(character set)：定义了字符以及字符的编码。
- 字符序(collation)：定义了字符的比较规则。

**有四个地方涉及到字符集和字符序**

- 服务器端(server level)
- 数据库(database level)
- 表(table level)
- char varchar text类型的字段(column level)

字符集和字符序集成顺序

1. 数据库服务，建库，建表，建字段，倘若其中有指定character和collation，则`字段`继承`表`，`表`继承`库`，`库`继承`数据库服务`。比如建库指定了字符集为utf8，那么该库下面的表如果不指定字符集，则表的字符集也为utf8，char、varchar、text的字段字符集也是utf8。
2. **有个例外，如果修改了表， 那么该表下面的字段的字符集和字符序也会变成表的字符集和字符序。**

```bash
# 查看 Mysql系统支持的字符集和字符序
mysql> show character set;
mysql> show collation;

# 查看创建的数据库默认的字符集和字符序
mysql> select @@character_set_database;
+--------------------------+
| @@character_set_database |
+--------------------------+
| latin1                   |
+--------------------------+
1 row in set (0.00 sec)


mysql> select @@collation_database;
+----------------------+
| @@collation_database |
+----------------------+
| latin1_swedish_ci    |
+----------------------+
1 row in set (0.00 sec)


## 查看表的字符集
## 要指定 数据库后才能查看某个表的字符集和字符序
mysql> SHOW TABLE STATUS where Name like "user"\G;
*************************** 1. row ***************************
           Name: user
         Engine: MyISAM
        Version: 10
     Row_format: Dynamic
           Rows: 2
 Avg_row_length: 132
    Data_length: 484
Max_data_length: 281474976710655
   Index_length: 2048
      Data_free: 220
 Auto_increment: NULL
    Create_time: 2021-07-19 08:18:14
    Update_time: 2021-07-19 08:18:21
     Check_time: NULL
     # 字符序 这里面包含了 字符集的信息吖
      Collation: utf8_bin
       Checksum: NULL
 Create_options:
        Comment: Users and global privileges
1 row in set (0.00 sec)

ERROR:
No query specified
有3种方式可以查看table的字符集/字符序。

方式一：通过SHOW TABLE STATUS查看table状态，注意Collation为utf8_general_ci，对应的字符集为utf8。
 SHOW TABLE STATUS FROM test_schema \G; 
方式二：查看information_schema.TABLES的信息。
 USE test_schema; 
 SELECT TABLE_COLLATION FROM information_schema.TABLES WHERE  
方式三：通过SHOW CREATE TABLE确认。
 SHOW CREATE TABLE test_table; 

## 查看某一列的字符集以及字符序
SELECT CHARACTER_SET_NAME, COLLATION_NAME FROM information_schema.COLUMNS WHERE TABLE_SCHEMA="test_schema" AND TABLE_NAME="test_table" AND COLUMN_NAME="char_column";

```

end

#### 字符集

...就是Mysql系统支持的一种种 Character Set...

```bash
mysql> show character set;
+----------+-----------------------------+---------------------+--------+
| Charset  | Description                 | Default collation   | Maxlen |
+----------+-----------------------------+---------------------+--------+
| big5     | Big5 Traditional Chinese    | big5_chinese_ci     |      2 |
...
| gbk      | GBK Simplified Chinese      | gbk_chinese_ci      |      2 |
| latin5   | ISO 8859-9 Turkish          | latin5_turkish_ci   |      1 |
| armscii8 | ARMSCII-8 Armenian          | armscii8_general_ci |      1 |
| utf8     | UTF-8 Unicode               | utf8_general_ci     |      3 |
|...
| utf8mb4  | UTF-8 Unicode               | utf8mb4_general_ci  |      4 |
|...
+----------+-----------------------------+---------------------+--------+

```

end

eg：utf8

#### 字符序

每一种字符集对应了多种排序规则，且每个排序规则只对应一种字符集，是一种典型的1对多的关系。

以utf8字符集举例

> - **utf8_general_ci**： 查询时不区分大小写匹配
> - **utf8_general_cs**： 查询时区分大小写匹配
> - **utf8_bin**： 字符串每个字符串用二进制数据编译存储。 区分大小写，而且可以存二进制的内容，与utf8_general_cs一样，区分大小写
> - **utf8_unicode_ci** ： 和utf8_general_ci一样，不区分大小写



字符序（Collation）: 是一组在指定字符集中进行字符比较的规则，比如是否忽略大小写，是否按二进制比较字符等等。

字符序表示的含义

- 一般来说分为三段，也存在一段或者两段的情况，常见的三段如`utf8mb4_general_ci`,两段的如`utf8mb4_bin`(**这类情况，其实只存在第一段和第三段，第二段不存在**)
- 第一段代表字符集。
- 第二段代表语言(chinese,swedish),也有general这种通用的，或者unicode类型。
- 第三段代表是否敏感，是否为bin。

对于第三段的解释

```
# 以_ci(表示大小写不敏感)、_cs(表示大小写敏感)
_ai	Accent insensitive
_as	Accent sensitive
_ci	Case insensitive
_cs	case-sensitive
_bin	Binary
```

- Accent是否为sensitive表现为，如果为sensitive，则比较a和á是不同的，如果为insensitive则a和á比较为相同。
- Case insensitive为大小写不敏感，case-sensitive为大小写敏感。



#### 字符集常见操作

创建表时指定字符集

```sql
CREATE TABLE `prediction_function` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `group_id` int(11) NOT NULL,
  `title` varchar(500) NOT NULL COMMENT '收集类型名(Collection type name)',
  `function` varchar(1024) NOT NULL COMMENT '虚拟曲线函数(Virtual curve function)',
  `degree` int(6) NOT NULL DEFAULT 0 COMMENT '阶数(degree)',
  PRIMARY KEY (`id`),
  KEY `groupId_index` (`group_id`) USING BTREE,
  KEY `title_index` (`title`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8mb4 ROW_FORMAT=DYNAMIC COMMENT='数据量预测函数(Data volume prediction function)';

mysql> CREATE TABLE `prediction_function` (
    ->   `id` int(11) NOT NULL AUTO_INCREMENT,
    ->   `group_id` int(11) NOT NULL,
    ->   `title` varchar(500) NOT NULL COMMENT '(Collection type name)',
    ->   `function` varchar(1024) NOT NULL COMMENT '(Virtual curve function)',
    ->   `degree` int(6) NOT NULL DEFAULT 0 COMMENT '(degree)',
    ->   PRIMARY KEY (`id`),
    ->   KEY `groupId_index` (`group_id`) USING BTREE,
    ->   KEY `title_index` (`title`) USING BTREE
    -> ) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8mb4 ROW_FORMAT=DYNAMIC COMMENT='(Data volume prediction function)';
Query OK, 0 rows affected, 3 warnings (0.02 sec)

mysql> SHOW TABLE STATUS where Name like "prediction_function"\G
*************************** 1. row ***************************
           Name: prediction_function
         Engine: InnoDB
        Version: 10
     Row_format: Compact
           Rows: 0
 Avg_row_length: 0
    Data_length: 16384
Max_data_length: 0
   Index_length: 32768
      Data_free: 0
 Auto_increment: 1
    Create_time: 2021-07-19 09:35:06
    Update_time: NULL
     Check_time: NULL
      Collation: utf8mb4_general_ci
       Checksum: NULL
 ## 显示的太清楚了
 Create_options: row_format=DYNAMIC
        Comment: (Data volume prediction function)
1 row in set (0.00 sec)

```

end

修改字符集/字符序

```bash
# 修改字段类型utf8mb4
alter table tablename convert to character set utf8mb4 collate utf8mb4_unicode_ci;

# 验证语句
SHOW FULL COLUMNS FROM dbname.tablename;

mysql> alter table prediction_function convert to character set utf8 collate utf8_general_ci;
Query OK, 0 rows affected, 2 warnings (0.06 sec)
Records: 0  Duplicates: 0  Warnings: 2

mysql> SHOW TABLE STATUS where Name like "prediction_function"\G
*************************** 1. row ***************************
           Name: prediction_function
         Engine: InnoDB
        Version: 10
     Row_format: Compact
           Rows: 0
 Avg_row_length: 0
    Data_length: 16384
Max_data_length: 0
   Index_length: 32768
      Data_free: 0
 Auto_increment: 1
    Create_time: 2021-07-19 09:38:39
    Update_time: NULL
     Check_time: NULL
      Collation: utf8_general_ci
       Checksum: NULL
 Create_options: row_format=DYNAMIC
        Comment: (Data volume prediction function)
1 row in set (0.01 sec)

```

end

mysql字符集转换

http://mysql.taobao.org/monthly/2017/03/06/

在MySQL的server和client之间、server和connection之间、已经connection和result set之间、所使用的字符集可能不一致，这就需要字符集之间的转换，才能保证字符存储和显示的正确。

```bash
## 编码
root@localhost : (none) 05:05:08> select hex(convert('a' using gbk));
+-----------------------------+
| hex(convert('a' using gbk)) |
+-----------------------------+
| 61                          |
+-----------------------------+
1 row in set (0.00 sec)

root@localhost : (none) 05:05:16> select hex(convert('a' using utf8));
+------------------------------+
| hex(convert('a' using utf8)) |
+------------------------------+
| 61                           |
+------------------------------+
1 row in set (0.00 sec)

## 解码
root@localhost : (none) 05:05:16> select convert(0x61 using utf8);
+------------------------------+
| convert('a' using utf8)      |
+------------------------------+
| a                          m |
+------------------------------+
1 row in set (0.00 sec)
```

end



### Mysql 的 UTF8问题

Mysql 中的 utf8 为什么只支持持最长三个字节的 UTF-8字符呢？

鬼知道

In *MySQL* `utf8` is currently an alias for `utf8mb3` which **is deprecated** and will be removed in a future *MySQL* release. At that point `utf8` **will become a reference to** `utf8mb4`.

- `utf8mb4`: A *UTF-8* encoding of the *Unicode* character set using **one to four bytes** per character.
- `utf8mb3`: A *UTF-8* encoding of the *Unicode* character set using **one to three bytes** per character.



### 不推荐使用utf8（一个观点）

UTF-8虽然是一个当今接受度最广的字符集编码，但是它并没有涵盖整个Unicode的字库，这也造成了它在某些场景下对于特殊字符的处理困难。

因为Mysql的utf8实际是utfmb3，三字节表示一个字符，存在严重问题，和标准的utf8不一致啊。

推荐大家平常还是使用utf8mb4吧，可能更好的兼容utf-8编码。兼的死死的。

不要在MySQL上使用utf8字符编码，推荐使用`utf8mb4`，至于为什么，引用国外友人的一段话：

> Never use utf8 in MySQL — always use utf8mb4 instead. Updating your databases and code might take some time, but it’s definitely worth the effort. Why would you arbitrarily limit the set of symbols that can be used in your database? Why would you lose data every time a user enters an astral symbol as part of a comment or message or whatever it is you store in your database? There’s no reason not to strive for full Unicode support everywhere. Do the right thing, and use utf8mb4. 🍻



### Mysql字符集与索引限制(太经典了)

很细，这点明白了，字符集就明白了.

#### 问题

Mysql 5.6 版本创建表出错

```sql
CREATE TABLE `prediction_function` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `group_id` int(11) NOT NULL,
  `title` varchar(500) NOT NULL COMMENT '收集类型名(Collection type name)',
  `function` varchar(1024) NOT NULL COMMENT '虚拟曲线函数(Virtual curve function)',
  `degree` int(6) NOT NULL DEFAULT 0 COMMENT '阶数(degree)',
  PRIMARY KEY (`id`),
  KEY `groupId_index` (`group_id`) USING BTREE,
  KEY `title_index` (`title`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8mb4  COMMENT='数据量预测函数(Data volume prediction function)';

```

提示，错误信息：`ERROR 1071 (42000): Specified key was too long; max key length is 767 bytes`

#### 原因

索引字段`title`是`varchar(500)`大于`191`了，所以导致索引太长，表创建失败。

自己尝试解决的时候，将`title`改成`varchar(255)` ，还是提示索引太长... 因为我当时没有注意到表的字符集是`utf8mb4`，还是简单认为是 500 大于 255的问题...而且我还不清楚这个255是从哪来的。

**mysql的varchar主键只支持不超过767个字节或者768/2=384个双字节 或者767/3=255个三字节的字段 而GBK是双字节的，UTF8是三字节的。**

总结：

不同的字符编码，会影响索引支持的字符数。

> 5.6 以前索引最多支持767字节，5.7以后支持 3072 bytes

* GBK: 最大索引字节数 / 2
* UTF8: 最大索引字节数 / 3 
* UTF8mb4: 最大索引字节数 / 4

#### 本质

因为Mysql5.6及以前版本， 默认索引最大长度767bytes，5.7及之后版本， 限制放开到3072 bytes。

Mysql若使用utf8mb4格式编码（utf8字符占用3字节，utf8mb4字符占用4字节）， 则单个字段长度不能超过191。

> 767 / 4 = 191 从而有了 varchar(191)

mysql的utf8编码并不是真正UTF-8，它使用三个字节(2^24)来表示一个字符，所以varchar最大值是`767 / 3 = 255`

所以呢，使用`utf8mb4`比`utf8`更容易出现索引超出限制的问题。

#### 解决

通过上面的分析已经知道是什么问题引起的了，那就好解决了：

* 方法1：升级Mysql到5.7版本，这样索引字节限制扩大到3072 bytes 。这样纵容采用`utf8mb4`也可以有768个字符，妥妥大于`varchar(500)` 够用了。

* 方法2: 在Mysql为5.6及其以前的版本下，修改`my.cnf`和`crate table stagement`。

  在MySQL5.6版本后引入了参数innodb_large_prefix可以解决这个问题。该参数控制是否允许单列的索引长度超过767字节，有ON和OFF两个取值：

  - ON ：Innodb表的行记录格式（row format）是Dynamic或Compressed的前提下，单列索引长度上限扩展到3072个字节
  - OFF：Innodb表的单例索引长度最多为767个字节，索引长度超出后，主键索引会创建失败，辅助索引会被截断成为前缀索引

  ```bash
  $ cat /etc/my.cnf
  ...
  innodb_file_format=barracuda
  # 独立表空间
  innodb_file_per_table=true
  
  innodb_large_prefix=ON
  ...
  ```

  在创建表时，需要 加 `ROW_FORMAT=DYNAMIC` 参数：

  ```sql
  CREATE TABLE `prediction_function` (
    `id` int(11) NOT NULL AUTO_INCREMENT,
    `group_id` int(11) NOT NULL,
    `title` varchar(500) NOT NULL COMMENT '收集类型名(Collection type name)',
    `function` varchar(1024) NOT NULL COMMENT '虚拟曲线函数(Virtual curve function)',
    `degree` int(6) NOT NULL DEFAULT 0 COMMENT '阶数(degree)',
    PRIMARY KEY (`id`),
    KEY `groupId_index` (`group_id`) USING BTREE,
    KEY `title_index` (`title`) USING BTREE
  ) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8mb4 ROW_FORMAT=DYNAMIC COMMENT='数据量预测函数(Data volume prediction function)';
  
  ```

  可以通过mysql client使用`set global`来实现，但也要修改配置文件，避免重启后，由于配置没有持久化出现问题。

  



### 乱码问题

`乱码`也就是英文常说的`mojibake`（由日语的`文字化け`音译）。 简单的说乱码的出现是因为：编码和解码时用了不同或者不兼容的字符集。对应到真实生活中，就好比是一个英国人为了表示祝福在纸上写了`bless`（编码过程）。而一个法国人拿到了这张纸，由于在法语中bless表示受伤的意思，所以认为他想表达的是`受伤`（解码过程）。这个就是一个现实生活中的乱码情况。在计算机科学中一样，一个用UTF-8编码后的字符，用GBK去解码。由于两个字符集的字库表不一样，同一个汉字在两个字符表的位置也不同，最终就会出现乱码。

#### 本质

乱码的本质都是一样的，**读取二进制的编码和最初将字符串转化成二进制的编码方式不一致。**

> **编码指将字符串转化成二进制，解码指将二进制转化成字符串**。

