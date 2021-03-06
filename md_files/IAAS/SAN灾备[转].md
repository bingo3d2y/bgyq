# SAN灾备[转]

## **1、hyper mirror：卷镜像**

是一种实现数据保护和业务连续性的技术。里面主要包括：一个镜像lun+副本A+副本B的架构

> 一个镜像lun最多两个副本

- 本地LUN：本端存储系统中的LUN（华为存储阵列）

- 外部LUN：异构存储系统中的LUN（第三方/托管的存储阵列）

- 镜像LUN：一种由两份同样大小存储空间的副本组成的LUN。当镜像lun的一个副本不可用时，不影响镜像LUN上业务运行。 镜像lun只有元数据，数据空间来自两个副本

  

- 镜像副本：组成镜像lun的一份副本，镜像副本是一种lun，但不能映射给主机使用。当移除镜像副本后，镜像副本会释放存储空间到存储池

- 写：双写，将主机对镜像LUN的写I/O请求同步写入镜像副本，所有副本都写完成后再返回主机写完成

- 读：轮询读，主机对镜像LUN读操作时，当镜像副本数据一致时，主机在两个镜像副本之间交替进行读操作

- DCL（Data Change log）:记录镜像副本的数据变更日志。DCL存放在镜像副本的原数据卷中

**通过hyper mirror + smart virtualization(异构接管)实现跨阵列的数据高可用保护，用于本地高可用的备选方案**

### **创建卷镜像**

- 对一个普通LUN（本地LUN或外部LUN）执行创建镜像LUN操作，此时普通LUN变成镜像LUN，同时会自动生成一个镜像副本
- 镜像副本完全继承普通LUN的存储空间和业务。对LUN ID为0的LUN执行创建镜像LUN操作，LUN 0变成了镜像LUN，同时生成LUN ID为1的镜像副本，镜像副本完全继承普通LUN 0的存储空间。此时镜像LUN就如同一个空壳，不再占用存储空间，但保留着LUN的基本属性（如LUN的名称、ID、WWN号等），对主机侧来说，由于映射给主机的HOST LUN ID和WWN号没有改变，所以主机侧业务没有任何影响
- 执行添加镜像LUN操作，添加一个镜像副本，其LUN ID为2。如果添加镜像副本为本地LUN，则需选择与LUN1不同的硬盘域，添加副本后，如果选择初始同步，则会将LUN1的数据完全同步到LUN2中。镜像副本是自动生成

## **2、hyper metro：双活特性**

和远程复制的差别在于：A、B阵列同时对外提供存储服务，双活的两个LUN都是读写状态，属于A to A模式

**实现hypermetro的过程**

1. 两个阵列存储复制链路：通常是优选用FC链路，并且两边站距控制在100km以内
2. 配置源端设备
3. 配置仲裁服务器（可选）、配置双活域
4. 配置双活pair

**写IO流程：**

1. 主机下发写I/O到双活管理模块。
2. 系统记录LOG。
3. 执行双写：双活管理模块同时将该写I/O写入本端Cache和远端Cache。
4. 本端Cache和远端Cache向双活管理模块返回写I/O结果。
5. 根据4的结果进行处理：

- 如果两端存储系统都返回写成功，则清除Log。

- 如果任意一端返回写失败，则进行以下处理：

- - 将Log转换成DCL，转换成功后清除Log，记录本端LUN和远端LUN的差异数据。
  - 双活Pair关系断开，双活Pair的运行状态变为待同步。I/O变成单写，写成功的一端继续提供主机业务，写失败的一端停止主机业务。

**读IO流程：**

1. 应用服务器向双活管理模块申请读权限。
2. 双活管理模块先从本端存储系统响应应用服务器的请求。
3. 如果本端存储系统正常，则本端存储系统将数据返回给双活管理模块。
4. 如果本端存储系统处于非正常状态，则通过双活管理模块去读远端存储系统的数据。远端存储系统将数据返回给双活管理模块。
5. 应用服务器读I/O成功。

**双活中有两个比较重要的机制**

- 锁机制：分布式锁保证两阵列数据的一致性

- 仲裁机制：防脑裂，当两阵列间心跳中断，必须通过仲裁机制来保证一端提供存储服务，另外一端停止存储，避免数据不一致

- - **1）静态优先级：** A--为优先站点，B--为非优先站点，检测原则如下

  - - 如果心跳中断，由优先站点提供存储服务，非优先站点停止存储服务
    - 如果优先站点故障间接造成心跳中断，非优先站点仍然停止存储服务，此时业务中断
    - 如果非优先站点故障接造成心跳中断，优先站点继续提供存储服务

- - **2）仲裁服务器：** 放在第三方站点通过GE链路与AB阵列交互，实现仲裁，检测原则如下

  - - 如果心跳中断，优先站点提供存储服务，非优先站点停止存储服务
    - 如果优先站点故障间接造成心跳中断，仲裁服务器触发仲裁让非优先站点提供存储服务，业务不中断
    - 如果非优先站点故障接造成心跳中断，优先站点继续提供存储服务

仲裁服务器优先级是高于静态优先级



**hypermetro通常用于两种解决方案：本地高可用和同城双活**

- 本地高可用：主机IO下发策略----用的是负载均衡（为什么用负载均衡，因为主机访问AB阵列路径长度相同）
- 同城双活：主机IO下发策略---是本地阵列优先（为什么是本地阵列优先，因为跨站点访问要考虑IO延迟和链带宽）

## **3、容灾解决方案分类**

**按照距离和保护的对象分为以下3种**

1. 本地容灾：本数据中心内部，或针对机柜或者机房级保护

- 本地高可用：存储层hypermetro，RPO=0、RTO=0；存储层异构+镜像，RPO=0、RTO>0
- 主备容灾：同步远程复制，RPO=0、RTO>0

1. 同城容灾：在距离不超过300KM范围内，针对生产数据中心做保护

- 同城双活：RPO=0、RTO>0
- 主备容灾：同步RPO=0、RTO>0；异步RPO>0、RTO>0
- 虚拟化网关复制：VRG主备容灾（异步复制） RPO>0、RTO>0

1. 异地容灾：在距离可以超过3000KM以上范围，针对生产数据中心所在区域保护

- 主备容灾：异步RPO>0、RTO>0

- - 两地三中心：根据组网和使用的技术不同，RPO和RTO会有所不同
  - 虚拟化网关复制：VRG主备容灾（异步复制）RPO>0、RTO>0

## **4、本地高可用容灾解决方案（重点，谨记）**

- 1）应用层：通过应用自身的集群机制实现高可用，或B/S / C/S架构实现
- 2）主机层：做主机集群，虚拟化场景下要集群开启HA，保证DRS
- 3）网络层：做双交换，交叉冗余组网
- 4）存储层：hyper metro（AB阵列都为华为存储并且都支持hypermetro）RPO=0 RTO=0 smart virtualization+hyper mirror（AB阵列一端华为一端第三方）RPO=0 RTO>0

## **5、本地主备容灾解决方案**

- 1）应用层：通过应用自身的集群机制实现高可用，或B/S / C/S架构实现
- 2）主机层：做主机集群，虚拟化场景下要集群开启HA，保证DRS
- 3）网络层：做双交换，交叉冗余组网
- 4）存储层：hyper replication 同步远程复制，RPO=0，RTO>0

## **6、同城双活容灾解决方案（重点，谨记）**

- 1）存储层：通过hypermetro实现（没有替代方案）

- 2）主机层：做主机集群，虚拟化场景下要集群开启HA，保证DRS

- 3）应用层：通过应用自身的集群机制实现高可用，或B/S / C/S架构实现

- 4）网络层：本数据中心内部双交换冗余组网，并且做链路冗余、板卡冗余达到可靠性，跨数据中心要做vxlan（大二层网络）

- 5）安全层：每数据中心内部1+1的防火墙部署，两数据中心传输数据加密

- 6）传输层：华为波分设备实现两数据中心DCI互联，将多对光路降为1对（一发一收）或者1+1保护的2对光路，并且通过波分的光放技术延长占据降低IO延时 通常情况下：两个DC超过25KM、并且互联链路超过4对，必须部署波分设备

  

## **7、同城主备容灾解决方案（重点，谨记）**

生产数据中心对外提供业务服务，容灾数据中心正常是不提供业务，只有生产中心故障，才会切换业务到容灾数据中心

两数据中心有相同的业务系统，相同的存储网络，存储阵列间做同步远程复制或者异步远程复制

- 同步：RPO=0、 RTO>0
- 异步：RPO>0、 RTO>0

## **8、两地三中心容灾解决方案（重点，谨记）**

同城+异地部署A/B/C三个数据中心，通过同城容灾去保护生产数据中心；通过异地容灾去保护生产中心所在区域，最大程度保证生产站点业务连续性，是高成本、高等级的容灾解决方案

**实现方式/组网**

- 级联组网：A-to-B-to-C，A生产站点压力较小，A--C的RPO比较大
- 并联组网：A-to-B，A-to-C，A生产站点压力较大，A--C的RPO较小

**技术组合** 同步+异步、异步+异步、双活+同步、双活+异步

**两地三中心优势：**

- 容灾等级高、有较好RTO和RPO保障
- 相对于同城容灾：抗区域性灾难
- 相对于异地容灾：较低的RPO（生产数据中心故障直接同城切换）

**两地三中心缺点：**

- 成本较高

## **9、fusionsphere容灾：主备容灾解决方案（重点，谨记）**

**1. 存储层容灾**

通过存储阵列间远程复制技术实现容灾保护，支持同步和异步

**实施过程**

- 1）生产端主lun映射给生产FC，通过存储资源--存储设备-数据存储操作创建业务VM
- 2）配置生产阵列与容灾阵列间的复制链路，并添加远端设备，容灾站点配置相同大小的从lun，主从lun归属控制器相同，配置远程复制pair关系，创建一致性组，并做初始同步
- 3）安装BCM（可以分布式部署-添加对端BCMserver信息、也可以单节点部署）
- 4）在BCM上配置站点对接站点资源（存储、FC），并且配置资源映射（集群、主机、端口组）
- 5）在BCM配置保护组（主要是存储层的保护组），创建保护策略
- 6）在BCM根据保护组配置恢复计划，和恢复VM的启动顺序
- 7）如果是分布式部署BCM，本端BCM配置数据同步到对端BCM

**BCM ereplication：** 容灾管理软件，针对存储层容灾、主机层容灾（需要安装agent）、虚拟化容灾做统一资源管理、保护策略下发、一键式操作

> 一键式操作包括： 测试、清理、计划性迁移、重保护、故障恢复 操作的区别，举例下3个

|                | 测试                                                     | 计划性迁移                                        | 故障恢复                                 |
| -------------- | -------------------------------------------------------- | ------------------------------------------------- | ---------------------------------------- |
| 生产端是否中断 | 不中断                                                   | 中断                                              | 中断                                     |
| 目的           | 测试容灾站点是否可用； 评估RTO RPO是否达标； 熟悉BCM使用 | 在已知故障前做切换； 最大程度降低故障对业务的影响 | 减少人工操作的错误； 降低RTO快速恢复业务 |

## **主机层容灾**

## **1、fusionsphere主备容灾：主机层复制**

- 它对存储设备没有严格要求，只要是能够给FC提供数据存储使用的即可（不一定要华为的设备）。存储间没有复制链路
- 主机层通过VRG（虚拟复制网关）的异步远程复制实现VM主备容灾

### **1）VRG的3个作用：**

- 聚合VM的IO数据并经过压缩、加密后发送到远端站点；
- 接收远端站点数据，并将数据路由发送到指定的主机上；
- 提供复制策略下发、状态查询等管理接口；

### **2）VRG部署方式：**

是模板部署，模板规格是2U/6G/15G系统盘/100G数据盘（100G数据盘为logcache盘，logcache盘主要用来做复制数据的物理缓存区）

- 一对VRG只能保护150台VM，及200个磁盘，所以要根据保护VM的数量确定VRG“成对”的数量
- 根据VRG的规格确定其应该 部署在哪些CNA节点上

**VRG3张网卡及作用：**

- 与CNA的业务管理接口互通，拥有收集VM的IO或者对端路由IO到CNA节点；
- 与BCM互通，用于对接BCM接收BCM下发的保护策略；
- 与对端VRG互通，就是异步复制的链路

### **3）主机层复制配置流程**

1. 生产端存储映射给生产FC，配置数据存储，创建业务虚拟机；容灾端存储也要映射给容灾站点FC，并配置数据存储，但不创建业务虚拟机（它叫做占位虚拟机是自动生成的）；
2. 按模板部署VRG（生产FC和容灾FC都要部署）；
3. 在本端BCM（分布式部署为例）去添加对端BCMserver
4. 在本端BCM创建站点，对接FC和VRG ；对端BCM也创建站点，对接FC和VRG；资源映射：集群、主机、端口组、数据存储
5. 在BCM上配置VRG配对，以及VRG保护的虚拟机；自动在对端FC上创建占位虚拟机；
6. 在BCM上根据受保护虚拟机配置保护策略
7. 在BCM上根据保护策略配置恢复计划，设置VM启动顺序
8. 容灾站点周期性对占位虚拟机打快照

### **4）占位虚拟机的概念：**

- 创建在容灾站点，配置、规格与被保护的虚拟机一致，一般处于关机状态
- 当容灾站点被启用时，占位虚拟机会将挂载容灾站点用于和生产站点同步数据的LUN，然后启动，从而将业务拉活

## **2、fusionsphere主备容灾，存储层复制和主机层复制的区别？**

|            | 原理                       | 支持的复制方式 | 复制粒度 | RPO  | 复制链路 | 虚拟化兼容性 | 存储兼容性                         |
| ---------- | -------------------------- | -------------- | -------- | ---- | -------- | ------------ | ---------------------------------- |
| 存储层复制 | 存储设备间hyperreplication | 同步/异步      | 基于LUN  | >=0  | FC/IP    | 高           | 华为V3/V5/dorado                   |
| 主机层复制 | CNA主机的iomirror+VRG      | 异步           | 针对VM   | >0   | IP       | 高           | 高（只要能给上层设备提供存储即可） |

**存储层复制应用场景：** 应用于对容灾要求比较高的关键性业务

- 生产站点和容灾站点存储设备同为华为存储
- 保护最小单位为LUN
- 实现同步远程复制
- 被保护的为私有云场景下的虚拟机

**主机层复制应用场景：** 应用于非关键性业务的容灾（目前FC版本已不支持）

- 无法使用存储层复制
- 保护的最小单位为虚拟机
- 被保护的为服务器虚拟化场景中的虚拟机

## **3、fusionsphere主机层容灾规划？（面试问题）**

### **1）BCM部署规划**

- 确定BCM部署方式，分布式还是单节点
- 如果分布式，打通两边站点BCM的管理网络，要确保网络带宽>10Mb/s ；如果是单节点，部署在容灾站点，需要两边FC管理网络互通

### **2）VRG规划**

- 根据受保护虚拟机数量去确定VRG对的数量（一对VRG最多保护150台VM、200个磁盘）
- 确定VRG安装的CNA节点（安装资源消耗）
- 规划VRG的网卡对应端口组以及IP地址
- VRG间复制链路带宽需要满足后面公式：保护虚拟机数量 * 周期内平均写IOPS * 数据块大小*8/0.7 `*8是把字节转换bit，0.7为70%`

### **3）BCM配置规划**

配置保护策略、配置恢复计划、虚拟机启动顺序等

### **4）存储规划**

只要能够为FC提供数据存储使用即可