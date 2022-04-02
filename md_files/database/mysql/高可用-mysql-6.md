## 高可用-mysql-6

## binlog

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

### binlog日志格式

MySQL 5.1.5 之前 binlog 的格式只有 STATEMENT，5.1.5 开始支持 ROW 格式的 binlog，从 5.1.8 版本开始，MySQL 开始支持 MIXED 格式的 binlog。

MySQL 5.7.7 之前，binlog 的默认格式都是 STATEMENT，在 5.7.7 及更高版本中，binlog_format 的默认值才是 ROW。

```bash
# 显示binlog日志格式

```



## 单主

## 多主

### MHA

MHA 由两个模块组成：Manager 和 Node。

https://www.infoq.cn/article/jty9bmxp9utrco9uhzm3

### MMM

https://tech.meituan.com/2017/06/29/database-availability-architecture.html

