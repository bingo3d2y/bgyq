## Mysql 高可用产品设计-0

### 思路：

产品级Mysql应用包含以下几点：

* 部署：自动化高可用部署
* 监控：Service Level Indicate
* HA：完成mysql的故障自动切换，比如主从切换
* end

各个模块要高内聚，低耦合彼此功能不受影响。

### 部署：Ambari Stack

Ambari Redis Service / Custom Stack will allow you to install and manage Redis and Redis HA (with Sentinels and Slaves) via Ambari. To use simply download and copy the stack contents to: /var/lib/ambari-server/resources/stacks/HDP/${hdp.version}/services/REDIS/

On Ambari server and restart the Ambari server service (sudo service restart ambari-server), once restarted you should see the new service in the list of services that can be installed.

云平台下发的Redis和Mysql实例机器上都有ambari 进程。

使用 `lsof -p <pid>`查看ambari进程连接了那些应用。

mysql-53306获取和采集了mysql-集群状态信息 ，ambari 是 Apache 的插件，是代理捕获异常信息，往上级平台推送信息

```bash
$ ambari-server status
Using python  /usr/bin/python
Ambari-server status
Ambari Server running
Found Ambari Server PID: 21282 at: /var/run/ambari-server/ambari-server.pid
$ lsof -p 21282|grep mysql
java    21282 root  175r      REG              253,1   2389251  526338 /usr/share/java/mysql-connector-java.jar
java    21282 root  197u     IPv6            3764770       0t0     TCP mysqlet2a4x1d-mysql-master-1-9a366-0.hde.com:57388->node-mysqlet2a4x1d-vserver.hde.com:53306 (ESTABLISHED)
java    21282 root  198u     IPv6            3764009       0t0     TCP mysqlet2a4x1d-mysql-master-1-9a366-0.hde.com:57386->node-mysqlet2a4x1d-vserver.hde.com:53306 (ESTABLISHED)
java    21282 root  199u     IPv6            3764017       0t0     TCP mysqlet2a4x1d-mysql-master-1-9a366-0.hde.com:57392->node-mysqlet2a4x1d-vserver.hde.com:53306 (ESTABLISHED)
java    21282 root  201u     IPv6            3765794       0t0     TCP mysqlet2a4x1d-mysql-master-1-9a366-0.hde.com:57394->node-mysqlet2a4x1d-vserver.hde.com:53306 (ESTABLISHED)

```

end

Apache Ambari是一种基于Web的工具，支持Apache Hadoop集群的供应、管理和监控。 Ambari已支持大多数Hadoop组件，包括HDFS、MapReduce、Hive、Pig、 Hbase、Zookeeper、Sqoop和Hcatalog等。

Ambari主要具有以下特性：

- 通过一步一步的安装向导简化了集群供应。
- 预先配置好关键的运维指标（metrics），可以直接查看Hadoop Core（HDFS和MapReduce）及相关项目（如HBase、Hive和HCatalog）是否健康。
- 支持作业与任务执行的可视化与分析，能够更好地查看依赖和性能。
- 通过一个完整的RESTful API把监控信息暴露出来，集成了现有的运维工具。
- 用户界面非常直观，用户可以轻松有效地查看信息并控制集群。
- Ambari使用Ganglia收集度量指标，用Nagios支持系统报警，当需要引起管理员的关注时（比如，节点停机或磁盘剩余空间不足等问题），系统将向其发送邮件。
- 此外，Ambari能够安装安全的（基于Kerberos）Hadoop集群，以此实现了对Hadoop 安全的支持，提供了基于角色的用户认证、授权和审计功能，并为用户管理集成了LDAP和Active Directory。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/ambari-architecture.png)

Ambari Server 会读取 Stack 和 Service 的配置文件。 当用 Ambari 创建集群的时候，Ambari Server 传送 Stack 和 Service 的配置文件以及 Service 生命周期的控制脚本到 Ambari Agent。Agent 拿到配置文件后， 会下载安装公共源里软件包（如 Redhat，就是使用 yum 服务）。

> Ambari DB通常使用Mysql--

安装完成后，Ambari Server 会通知 Agent 去启动 Service。 之后 Ambari Server 会定期发送命令到 Agent 检查 Service 的状态，Agent 上报给 Server，并呈现在 Ambari 的 GUI 上。

Ambari Server 支持 Rest API，这样可以很容易的扩展和定制化 Ambari。 甚至于不用登陆 Ambari 的 GUI，只需要在命令行通过 curl 就可以控制 Ambari，以及控制 Hadoop 的 cluster。 具体的 API 可以参见 Apache Ambari 的官方网页 API reference。

对于安全方面要求比较苛刻的环境来说，Ambari 可以支持 Kerberos 认证的 Hadoop 集群。

支持自定义ambari stack，以便于支撑多种组件包安装（比如：大数据、算法、K8S等等）。

#### Ambari-redis-service

Ambari Redis Service / Custom Stack will allow you to install and manage Redis and Redis HA (with Sentinels and Slaves) via Ambari.

https://github.com/Symantec/ambari-redis-service

#### 疑惑

The Apache Ambari project is aimed at making Hadoop management simpler by developing software for provisioning, managing, and monitoring Apache Hadoop clusters. Ambari provides an intuitive, easy-to-use Hadoop management web UI backed by its RESTful APIs.

这是Ambari官网的介绍，它是个快速部署和管理Hadoop集群的工具，那么它的custom stack部署mysql、redis、mongodb说还需要依赖Hadoop集群吗。

为什么

### 监控

Ambari收集数据库实例数据存储到额外的Mysql实例中

Ambari 部署加管理全包了

### HA

加强版的keepalived实现，实现根据Mysql数据库状态，完成主从切换和VIP 漂移。

#### keepalive实现mysql 主从切换

https://blog.csdn.net/u010533511/article/details/88168410

```bash
# master.sh的作用是状态改为master以后执行的脚本。首先判断复制是否有延迟，如果有延迟，等1分钟后，不论是否有延迟，都并停止复制，并且记录binlog和pos点.

#!/bin/bash
 
. /home/mysql/.bashrc
 
Master_Log_File=$(mysql -uroot -S /data/mysql.sock -e "show slave status\G" | grep -w Master_Log_File | awk -F": " '{print $2}')
Relay_Master_Log_File=$(mysql -uroot -S /data/mysql.sock -e "show slave status\G" | grep -w Relay_Master_Log_File | awk -F": " '{print $2}')
Read_Master_Log_Pos=$(mysql -uroot -S /data/mysql.sock -e "show slave status\G" | grep -w Read_Master_Log_Pos | awk -F": " '{print $2}')
Exec_Master_Log_Pos=$(mysql -uroot -S /data/mysql.sock -e "show slave status\G" | grep -w Exec_Master_Log_Pos | awk -F": " '{print $2}')
 
i=1
 
while true
do
 
if [ $Master_Log_File = $Relay_Master_Log_File ] && [ $Read_Master_Log_Pos -eq $Exec_Master_Log_Pos ]
then
   echo "ok"
   break
else
   sleep 1
 
   if [ $i -gt 60 ]
   then
      break
   fi
   continue
   let i++
fi
done
 
mysql -uroot -S /data/mysql.sock -e "stop slave;"
mysql -uroot -S /data/mysql.sock -e "reset slave all;"
mysql -uroot -S /data/mysql.sock -e "reset master;"
mysql -uroot -S /data/mysql.sock -e "show master status;" > /tmp/master_status_$(date "+%y%m%d-%H%M").txt


# stop.sh表示Keepalived停止以后需要执行的脚本。检查是否还有复制写入操作，最后无论是否执行完毕都退出。
#!/bin/bash
 
. /home/mysql/.bashrc
 
M_File1=$(mysql -uroot -S /data/mysql.sock -e "show master status\G" | awk -F': ' '/File/{print $2}')
M_Position1=$(mysql -uroot -S /data/mysql.sock -e "show master status\G" | awk -F': ' '/Position/{print $2}')
sleep 1
M_File2=$(mysql -uroot -S /data/mysql.sock -e "show master status\G" | awk -F': ' '/File/{print $2}')
M_Position2=$(mysql -uroot -S /data/mysql.sock -e "show master status\G" | awk -F': ' '/Position/{print $2}')
 
i=1
 
while true
do
 
if [ $M_File1 = $M_File1 ] && [ $M_Position1 -eq $M_Position2 ]
then
   echo "ok"
   break
else
   sleep 1
 
   if [ $i -gt 60 ]
   then
      break
   fi
   continue
   let i++
fi
done
```

end

