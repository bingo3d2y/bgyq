# 光纤：WWPN和WWNN的区别-转

### 网卡分类

传输协议的不同的，网卡可分为三种：

* 以太网卡
* FC网卡
* iSCSI网卡

#### 以太网卡

以太网卡，学名Ethernet Adapter,传输协议为IP协议,一般通过光纤线缆或双绞线与以太网交换机连接。接口类型分为光口和电口。光口一般都是通过光纤线缆来进行数据传输，接口模块一般为SFP（传输率2Gb/s）和GBIC（1Gb/s）,对应的接口为SC、ST和LC。电口目前常用接口类型为RJ45,用来与双绞线连接，也有与同轴电缆连接的接口，不过现在已经用的比较少了。

#### FC网卡

FC网卡，一般也叫光纤网卡，学名Fibre Channel HBA。传输协议为光纤通道协议，一般通过光纤线缆与光纤通道交换机连接。接口类型分为光口和电口。光口一般都是通过光纤线缆来进行数据传输，接口模块一般为SFP（传输率2Gb/s）和GBIC（1Gb/s）,对应的接口为SC和LC。电口的接口类型一般为DB9针HSSDC。

#### iSCSI网卡

ISCSI网卡，学名ISCSI HBA，传输ISCSI协议，接口类型与以太网卡相同。



### HBA

HBA卡全称主机总线适配器(Host Bus Adapter,HBA)是一个在服务器和存储装置间提供输入/输出(I/O)处理和物理连接的电路板和/或集成电路适配器。
WWNN 和 WWPN 是由 16 个 16 进制数字组成的 64 位地址，每 2 个数字为一对，用冒号分开。其中第一位数字通常是1、2或 5。

* Emulex initiator HBA卡通常是1
* Qlogic initiator HBA卡通常是 2
* NetApp存储端的 HBA卡通常是 5。

对于单口的FC HBA来说，它应该只有一个WWNN和一个WWPN
对于双口的FC HBA来说，它应该只有一个WWNN和和两个WWPN

对于主机来说：
单个hba卡（单口）的情况下： wwnn只有一个 wwpn和wwnn一样
单个hba卡（双口）的情况下： wwnn只有一个 wwpn有两个
两个hba卡（单口）的情况下： wwnn有两个 wwpn有两个
两个hba卡（双口）的情况下： wwnn有两个 wwpn有四个

### WWN
WWN是HBA卡用的编号，每一个光纤通道设备都有一个唯一的标识，称为WWN（world wide name），由IEEE负责分配。在有多台主机使用磁盘阵列时，通过WWN号来确定哪台主机正在使用指定的LUN（或者说是逻辑驱动器），被使用的LUN其他主机将无法使用。

WWN概念包含WWPN、WWNN，wwn有两种表示方法： wwpn wwnn。

#### WWPN

World Wide Port Name, 是对于FC HBA卡上的一个端口的全球唯一标识符，它相当于网卡中的MAC地址，全球只有唯一一个。

#### WWNN

World Wide Node Name, 是对于每一个节点（终端或者设备）的全球唯一标示符。

### FC交换机:star2:

对于FC交换机来说，它应该只有一个WWNN，每一个接口对应一个WWPN。

一个不可拆分的独立的设备有WWNN，一个端口有WWPN。

比如一台SAN交换机，不可拆分，有一个WWNN，它有一堆端口，每个端口有一个WWPN。一块多口光纤HBA，卡本身有一个WWNN，每个端口有一个WWPN，单口的HBA也是，不过只有一个WWNN和一个WWPN。但主机就没有WWNN，因为卡和主机是可以分离的，单纯一个主机本身并不一定是SAN环境中的设备。

有WWNN的好处是：即使不去看连线，也可以清楚地知道，哪些端口是在一个物理设备上





When configuring World Wide Name (WWN) based zoning, it is important to always use the World Wide Port Name (WWPN), not the World Wide Node Name (WWNN). With many systems, the WWNN is based on the Port WWN of the first adapter detected by the HBA driver. If the adapter the WWNN was based on were to fail, and you based your zoning on the WWNN, your zoning configuration would become invalid. Subsequently, the host with the failing adapter would completely lose access to the storage attached to that switch.

### 原文大佬

https://blog.moper.net/2182.html