## HA for everything -- draft

任何平台、软件和方案的高可用都得从这些方面考虑。

### 存储

#### DRBD：网络Raid 1

DRBD的英文全称是Distributed Replicated Block Deivce(分布式块设备复制)，是linux内核的存储层一个分布式存储系统。可以利用DRBD在两台服务器之间共享块设备，文件系统和数据。类似于一个网络RAID1的功能。

#### ceph

rbd 和cephFS

#### 备份

保障Recovery Time Objective，恢复时间目标。

### 硬件网络

#### 网卡bond

物理机高可用及host上vm的网络高可用

#### M

交换机链路高可用

### 软件网络

#### keepalived

vip 高可用

### 计算

#### 无状态设计

### 监控

监控的高可用又要从 存储、计算和HA考虑。