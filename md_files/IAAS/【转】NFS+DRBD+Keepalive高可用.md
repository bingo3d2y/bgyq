## 【转】NFS+DRBD+Keepalive高可用

NFS做为业界经常使用的共享存储方案，被众多公司采用。我司也不列外，使用NFS做为共享存储，为前端WEB server提供服务，主要存储网页代码以及其余文件。

https://www.jianshu.com/p/864caf7ea98d

### 高可用方案

说道NFS，不得不说它的同步技术，同步技术有两种，前端

- 第一种就是借助RSYNC+inotify来实现主从同步数据。
- 第二种借助DRBD，实现文件同步。
  上诉两种方案都没有实现高可用，只是实现了二者数据同步。可是业务要求NFS服务器必须是高可用，因此咱们在第二种同步方案的基础上，在结合keepalive/+heartbeat来实现高可用。

### DRBD

DRBD官网：http://www.drbd.org/en/doc/users-guide-90
DRBD的英文全称是Distributed Replicated Block Deivce(分布式块设备复制)，是linux内核的存储层一个分布式存储系统。可以利用DRBD在两台服务器之间共享块设备，文件系统和数据。类似于一个网络RAID1的功能。
*它有三个特点：*

1. 实时复制：一端修改后马上复制过去。
2. 透明的传输： 应用程序不需要感知（或者认识）到这个数据存储在多个主机上。
3. 同步或者异步通过：同步镜像，应用程序写完成后会通知所有已连接的主机；异步同步；应用程序会在本地写完之前通知其他的主机。

*它有三个协议*

1. A 写I/O达到本地磁盘和本地的TCP发送发送缓存区后，返回操作成功。
2. B 写I/O达到本地磁盘和远程节点的缓存区之后，返回操作成功。
3. C 写I/O达到本地磁盘和远程节点的磁盘之后，返回操作成功。

架构图如下：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/linux-drbd.png)

当数据写入到本地主节点的文件系统时，这些数据会通过网络发送到另一台主节点上。本地节点和远程节点数据通过TCP/IP协议保持同步，主节点故障时，远程节点保存着相同的数据，可以接替主节点继续提供数据。两个节点之间通过使用keepalive/heartbeat来检测对方是否存活。

#### 典型配置

采用单主模式:典型的高可靠性集群方案。DRBD负责接收数据，把数据写到本地磁盘，然后发送给另一个主机，另一个主机再将数据存到自己的磁盘中。

### NFS

nfs安装

#### NFS问题排查

nfsd IO读写过高

从nfs服务器角度排查：

* 使用`ss`或者`netstat`查看那些机器连接了这个nfs server
* 排查定位可能出现问题的机器。比如：mysql server或者高并发web server

#### NFS 优化

https://blog.51cto.com/u_13272050/2115363

### keepalived

#### 检测脚本

```bash
## 监控脚本
[root@k8s-nfs-Master ~]# cat << EOF | tee /etc/keepalived/nfs_check.sh
#!/bin/bash
# 日志文件大于5M就只保留最后50行
[ `du -m /tmp/nfs-chk.log | awk '{print $1}'` -gt 5 ] && tail -50 /tmp/nfs-chk.log >/tmp/nfs-tmp && mv /tmp/nfs-tmp /tmp/nfs-chk.log
vip=`ip a |grep 53.150|wc -l`
if [ $vip -eq 1 ];then # 主keepalived机器检查
    service nfs status &>/dev/null # 检查nfs可用性
    if [ $? -ne 0 ];then # 如果服务状态不正常，先尝试重启服务
        time=`date "+%F %H:%M:%S"`
        echo -e "$time ------主机NFS服务故障，重启之！------\n" >>/tmp/nfs-chk.log
        service nfs start &>/dev/null
    fi
    nfsStatus=`ps -C nfsd --no-header | wc -l`
    if [ $nfsStatus -eq 0 ];then # 若重启nfs服务后，仍不正常
        time=`date "+%F %H:%M:%S"`
        echo -e "$time ------nfs服务故障且重启失败，切换到备用服务器------\n">>/tmp/nfs-chk.log
        service nfs stop &>>/tmp/nfs-chk.log # 停止nfs服务
        umount /dev/drbd1 &>>/tmp/nfs-chk.log # 卸载drbd设备
        drbdadm secondary r1 &>>/tmp/nfs-chk.log # 将drbd主降级为备
        service keepalived stop &>>/tmp/nfs-chk.log # 关闭keepalived(切换)
        time=`date "+%F %H:%M:%S"`
        echo -e "$time ------切换结束！------\n" >>/tmp/nfs-chk.log
        sleep 2
        service keepalived start &>>/tmp/nfs-chk.log # 再开启keepalived服务
    else
        # drbd置主没有，挂载没有
        drbdadm role r1|grep Primary &>/dev/null
        if [ $? -ne 0 ];then # drbd未置Primary
            time=`date "+%F %H:%M:%S"`
            echo -e "$time ------将本机置为DRBD主机并挂载/nfs目录------\n" >>/tmp/nfschk.log
            drbdadm primary r1 &>>/tmp/nfs-chk.log # 将drbd置为主
            mount /dev/drbd1 /nfs &>>/tmp/nfs-chk.log # 挂载drbd设备
        fi
    fi
else # keepalived备机检查
    service nfs status &>/dev/null
    if [ $? -eq 0 ];then # NFS服务必须处于关闭状态
        time=`date "+%F %H:%M:%S"`
        echo -e "$time ------关闭备机NFS服务------\n" >>/tmp/nfs-chk.log
        service nfs stop &>>/tmp/nfs-chk.log
    fi
    drbdadm role r1|grep Primary &>/dev/null
    if [ $? -eq 0 ];then # drbd必须置备并卸载drbd设备
        time=`date "+%F %H:%M:%S"`
        echo -e "$time ------备机置secondary并卸载备机drbd设备------\n" >>/tmp/nfschk.log
        drbdadm secondary r1 &>>/tmp/nfs-chk.log
        umount /dev/drbd1 &>>/tmp/nfs-chk.log &>>/tmp/nfs-chk.log
    fi
fi
EOF
# 配置keepalived停服脚本
[root@k8s-nfs-Master ~]# cat << EOF | tee /etc/keepalived/notify_stop.sh
#!/bin/bash

time=`date "+%F %H:%M:%S"`
echo -e "$time ------开始切换到备用服务器------\n" >>/tmp/nfs-chk.log
service nfs stop &>>/tmp/nfs-chk.log # 停止nfs服务
service smb stop &>>/tmp/nfs-chk.log # 停止smb服务
umount /dev/drbd1 &>>/tmp/nfs-chk.log # 卸载drbd设备
drbdadm secondary r1 &>>/tmp/nfs-chk.log # 将drbd主降级为备
time=`date "+%F %H:%M:%S"`
echo -e "$time ------切换结束！------\n" >>/tmp/nfs-chk.log
sleep 2
service keepalived start &>>/tmp/nfs-chk.log # 再开启keepalived
EOF
```

