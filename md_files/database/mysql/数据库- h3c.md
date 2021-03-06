### 数据库- h3c

研究过存储引擎结构和性能优化的工程师都会知道，许多数据库的奇技淫巧都是在解决内存与磁盘的读写模式、性能的不匹配问题。

#### TSDB

时序数据库也是数据库的一种，只要它想持久化，就需要解决内存与磁盘的读写模式和性能不匹配问题。但与键值数据库相比，时序数据库存储的数据有更特殊的读写特征，Björn Rabenstein 将称其为：Vertical writes, horizontal(-ish) reads（垂直写，水平读）。

图中每条横线就是一个时序，每个时序由按照（准）固定间隔采集的样本数据构成，通常在时序数据库中会有很多活跃时序，因此数据写入可以用一个垂直的窄方框表示，即每个时序都要写入新的样本数据；用户在查询时，通常会观察某个、某几个时序在某个时间段内的变化趋势，或对其进行聚合计算，因此数据读取可以用一个水平的方框表示。是谓“垂直写、水平读”。

##### Data Model

尽管数据模型是存储层之上的抽象，理论上它不应该影响存储层的设计。但理解数据模型能够帮助我们更快地理解存储层。

在 Prometheus 中，每个时序实际上由多个标签（labels）标识，如：

```
api_http_requests_total{path="/users",status=200,method="GET",instance="10.111.201.26"} 
```

该时序的名字为 api_http_requests_total，标签为 path、status、method 和 instance，**只有时序名字和标签键值完全相同的时序才是同一个时序。**事实上，时序名字就是一个隐藏标签：

> eg: 不同的标签可以表示不同的数据，典型例子使用http code 2XX和5XX 来区分--

```
{__name__="api_http_requests_total",path="/users",status=200,method="GET",
instance="10.111.201.26"} 
```

对于用户来说，标签之间不存在先后顺序，用户可能关注：

- 所有 API 调用的 status
- 某个 path 调用的成功率、QPS
- 某个实例、某个 path 调用的成功率
- ……

##### LevelDB

https://chienlungcheung.github.io/2020/09/11/leveldb-annotations-0-usage-and-examples/

##### Gorilla

Gorilla 是 Facebook 于 2015 年开放的一个快速, 可扩展的, 内存式时序数据库. 它的一些设计理念影响了后来的 Prometheus. 本文就其设计和实现进行深入分析希望能为各位后续在系统研发中提供灵感.

https://chienlungcheung.github.io/2020/12/05/Gorilla-%E4%B8%80%E4%B8%AA%E5%BF%AB%E9%80%9F-%E5%8F%AF%E6%89%A9%E5%B1%95%E7%9A%84-%E5%86%85%E5%AD%98%E5%BC%8F%E6%97%B6%E5%BA%8F%E6%95%B0%E6%8D%AE%E5%BA%93/



#### BigTable && LSM-Tree

十多年前，谷歌发布了大名鼎鼎的"三驾马车"的论文，分别是GFS(2003年)，MapReduce（2004年），BigTable（2006年），为开源界在大数据领域带来了无数的灵感，其中在 “BigTable” 的论文中很多很酷的方面之一就是它所使用的文件组织方式，这个方法更一般的名字叫 Log Structured-Merge Tree。在面对亿级别之上的海量数据的存储和检索的场景下，我们选择的数据库通常都是各种强力的NoSQL，比如Hbase，Cassandra，Leveldb，RocksDB等等，这其中前两者是Apache下面的顶级开源项目数据库，后两者分别是Google和Facebook开源的数据库存储引擎。而这些强大的NoSQL数据库都有一个共性，就是其底层使用的数据结构，都是仿照“BigTable”中的文件组织方式来实现的，也就是LSM-Tree。

https://cloud.tencent.com/developer/article/1441835

LSM-Tree全称是Log Structured Merge Tree，是一种分层，有序，面向磁盘的数据结构，其核心思想是充分了利用了，磁盘批量的顺序写要远比随机写性能高出很多。

LSM tree （**log-structured merge-tree** ）通过一种叫做 **SSTable (Sorted Strings Table)** 的格式，持久化到硬盘上。正如其名，SSTable 是一种用来存储有序的键值对的格式，其中键的组织是有序存储的。一个SSTable 会包括多个有序的子文件，被称为 *segment* 。 这些 *segments* 一旦被写入硬盘，就不可以再修改了。

##### 写入数据

由于 LSM tree 只会进行顺序写入，所以自然而然地就会引出这样一个问题，写入的数据可能是任意顺序的，我们又如何保证数据能够保持 SSTable 要求的有序组织呢？
这就需要引入新的常驻内存 *(in-memory)* 数据结构: *memtable_了, _memtable* 的底层数据结构则有点像[红黑树](https://en.wikipedia.org/wiki/Red–black_tree),当由新的写入操作则将数据插入到红黑树中。

写入操作会先把数据存储到红黑树中，直至红黑树的大小达到了预先定义的大小。一旦红黑树的大小达到阈值，就会把数据整个刷到磁盘中，这个过程就可以把数据保证有序写入了。经过一层数据结构的承接，就可以保证单向顺序写入的同时，也能保证数据的有序了。

##### 读取数据  

如何从SSTable中查找数据的呢？一种naive的方法就是遍历所有的 segments，寻找我们需要的key。从最新的 segment 到最老的 segment 一一遍历，知道找到目标key为止。显然，这种方式在寻找刚刚写入的数据是比较快的，但是文件一多就不太行了。因此也有针对这个问题的优化，[稀疏索引](https://yetanotherdevblog.com/dense-vs-sparse-index/) 就是一种在内存中对数据检索进行优化的技术。



LSM tree 引擎是如何工作的：

1. 写入操作是先写入内存的（被成为 memtable）。所有的用于加速查询的数据结构（布隆过滤器和稀疏索引）都会被同时更新；
2. 当内存中的 memtable 太大了，将会被刷到磁盘中，注意是有序的；
3. 当查询时我们先回查询布隆过滤器，如果布隆过滤器返回说键不存在，则实际不存在，如果布隆过滤器说存在，进一步遍历 segment 文件；
4. 对于遍历 segment 文件的过程，我们将会先通过稀疏索引找到最小的文件范围，并开始由新到老开始遍历，找到一个key则直接返回。



#### 红黑树

红黑树的查询时间复杂度是O(log2 N)，2为底N的对数，10亿数据最坏情况需要30次。

流弊-



##### 关系模型

关系数据库是建立在关系模型上的。而关系模型本质上就是若干个存储数据的二维表，可以把它们看作很多Excel表。

表的每一行称为记录（Record），记录是一个逻辑意义上的数据。

表的每一列称为字段（Column），同一个表的每一行记录都拥有相同的若干字段。

此模型用二维表结构来表示实体及实体间联系的模型，并以二维表格的形式组织数据库中的数据。
**特点：**

- 建立在严格的数学概念基础之上
- 概念单一，统一用关系表示实体及实体间的联系，对数据的检索与更新结果同样也用关系（即表）表示。
- 关系模型的存取路径对用户透明，这样就具有更高的数据独立性、更好的安全保密性，简化了程序员的开发工作。目前流行的商用数据库大多基于关系模型（关系数据库管理系统）。