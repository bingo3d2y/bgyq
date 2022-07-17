## 存储

github的全文搜索真香--

### lvm卷扩容

LVM是逻辑盘卷管理（LogicalVolumeManager）的简称，在Linux环境下对磁盘分区进行管理的一种机制，LVM是建立在硬盘和 分区之上的一个逻辑层，来提高磁盘分区管理的灵活性。通过LVM系统管理员可以轻松管理磁盘分区，如：将若干个磁盘分区连接为一个整块的卷（volumegroup），形成一个存储池。管理员可以在卷组上随意创建逻辑卷组（logicalvolumes），并进一步在逻辑卷组上创建文件系统。

**LVM组成**

Logical Volume Manager(逻辑卷管理)

PV:是物理的磁盘分区

VG:LVM中的物理的磁盘分区，也就是PV，必须加入VG，可以将VG理解为一个仓库统一管理了几个大的硬盘，形成了一个统一虚拟的存储资源池。

LV：也就是从VG中划分的逻辑分区

#### 扩容物理磁盘

修改虚拟机配置或者扩充物理机磁盘

```bash
## 确定物理磁盘是否扩容成功
$ lsblk 
NAME          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
# 已扩容到500G
sda             8:0    0  500G  0 disk 
├─sda1          8:1    0  600M  0 part /boot/efi
├─sda2          8:2    0    1G  0 part /boot
└─sda3          8:3    0 98.4G  0 part 
  ├─klas-root 252:0    0   20G  0 lvm  /
  ├─klas-swap 252:1    0   16G  0 lvm  [SWAP]
  ├─klas-home 252:2    0   10G  0 lvm  /home
  ├─klas-var  252:3    0   10G  0 lvm  /var
  ├─klas-tmp  252:4    0   10G  0 lvm  /tmp
  └─klas-opt  252:5    0 32.4G  0 lvm  /opt
sr0            11:0    1 1024M  0 rom  
```

#### 磁盘分区

parted 命令用于创建，查看，删除和修改磁盘分区。它是一个磁盘分区和分区大小调整工具。这个命令算是对fdisk命令的一个补充，使用gpt(全局唯一标示磁盘分区表格式)的分区表，因为如果磁盘大小大于2TB就无法使用fdisk命令进行分区操作了（MBR 的最大可循地址为2T）。

```bash
$ parted   /dev/sda
GNU Parted 3.3
Using /dev/sda
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) help
  align-check TYPE N                       check partition N for TYPE(min|opt) alignment
  help [COMMAND]                           print general help, or help on COMMAND
  mklabel,mktable LABEL-TYPE               create a new disklabel (partition table)
  mkpart PART-TYPE [FS-TYPE] START END     make a partition
  name NUMBER NAME                         name partition NUMBER as NAME
  print [devices|free|list,all|NUMBER]     display the partition table, available devices, free space, all found partitions, or a particular partition
  quit                                     exit program
  rescue START END                         rescue a lost partition near START and END
  resizepart NUMBER END                    resize partition NUMBER
  rm NUMBER                                delete partition NUMBER
  select DEVICE                            choose the device to edit
  disk_set FLAG STATE                      change the FLAG on selected device
  disk_toggle [FLAG]                       toggle the state of FLAG on selected device
  set NUMBER FLAG STATE                    change the FLAG on partition NUMBER
  toggle [NUMBER [FLAG]]                   toggle the state of FLAG on partition NUMBER
  # (parted) unit s 可以设置磁盘的计数单位是磁柱即fdisk的格式
  unit UNIT                                set the default unit to UNIT
  version                                  display the version number and copyright information of GNU Parted
# 简写输入 p 也行
(parted) print                                                            
Warning: Not all of the space available to /dev/sda appears to be used, you can fix the GPT to use all of the space (an extra 838860800 blocks) or continue with the
current setting? 
## 因为磁盘扩容了，需要修复 GPT信息
Fix/Ignore? Fix
Model: QEMU QEMU HARDDISK (scsi)
Disk /dev/sda: 537GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

# start： 开始大小
# end：结束大小 ，类似fdisk分区时指定的磁柱numbers，但是这个比较直观
Number  Start   End     Size    File system  Name                  Flags
 1      1049kB  630MB   629MB   fat32        EFI System Partition  boot, esp
 2      630MB   1704MB  1074MB  xfs
 3      1704MB  107GB   106GB                                      lvm

(parted) mk                                                               
mklabel  mkpart   mktable  
# make a partition，创建一个新的磁盘分区
(parted) mkpart 4
## 默认使用ext2文件系统，因为是给lv扩容，所以无所谓文件系统？ 这里选择默认即可
File system type?  [ext2]?   
## 分区开始位置大小，从partition 3 结束的107G开始
Start? 108GB                                
## 100%即将磁盘剩余大小都给partition 4
End? 100%                                                                 
(parted) p                                                                
Model: QEMU QEMU HARDDISK (scsi)
Disk /dev/sda: 537GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name                  Flags
 1      1049kB  630MB   629MB   fat32        EFI System Partition  boot, esp
 2      630MB   1704MB  1074MB  xfs
 3      1704MB  107GB   106GB                                      lvm
 # 多了一个新的分区
 4      108GB   537GB   429GB   ext2         4
# quit
(parted) q                                                                
Information: You may need to update /etc/fstab.
```

#### lv扩容

logical volume扩容分为两部：

* 所属vg扩容：将磁盘分区加入volume group

  这里不需要使用pvcreate 将磁盘分区创建为pv？

* logical volume扩容

  这里logical volume filesystem会覆盖disk partition filesystem

* 文件系统扩容

```bash
# 将磁盘加入到 VG卷组。
# vgdisplay 显示vg信息
$ vgextend klas /dev/sda4
  Physical volume "/dev/sda4" successfully created.
  Volume group "klas" successfully extended

# 扩容逻辑卷 -l 指定的是PE数量 -L +800GB
# Extend an LV by a specified size.
#  lvextend -L|--size [+]Size[m|UNIT] LV
#	[ -l|--extents [+]Number[PERCENT] ]
# lvdisplay显示lv信息和名字
$ lvextend -L +300G /dev/klas/root
  Size of logical volume klas/root changed from 20.00 GiB (5120 extents) to 320.00 GiB (81920 extents).
  Logical volume klas/root successfully resized.

# mount | grep root 查看使用的文件系统信息
# 修改文件系统的大小，xfs 文件系统使用xfs_growfs。 
$ xfs_growfs /dev/klas/root
meta-data=/dev/mapper/klas-root  isize=512    agcount=4, agsize=1310720 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1
data     =                       bsize=4096   blocks=5242880, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 5242880 to 83886080

# 验证扩容结果
$ df -h
Filesystem             Size  Used Avail Use% Mounted on
devtmpfs               7.1G     0  7.1G   0% /dev
tmpfs                  7.3G  192K  7.3G   1% /dev/shm
tmpfs                  7.3G   22M  7.3G   1% /run
tmpfs                  7.3G     0  7.3G   0% /sys/fs/cgroup
/dev/mapper/klas-root  320G   18G  303G   6% /
...
/dev/sda1              599M  6.5M  593M   2% /boot/efi
tmpfs                  1.5G     0  1.5G   0% /run/user/1002
```

end

### 开机自动挂载磁盘

#### 获取磁盘uuid

```bash
#  blkid - locate/print block device attributes
root@Dodo:~# blkid
/dev/vda1: LABEL="/" UUID="7fe02849-d810-401f-b7f8-e5782ca5939d" TYPE="ext4" PARTUUID="0f77bf27-01"

```

#### 配置自动挂载

详细参数后续补充

```bash
$ cat /etc/fstab
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# / was on /dev/vda1 during installation
UUID=7fe02849-d810-401f-b7f8-e5782ca5939d /               ext4    errors=remount-ro 0       1
/dev/fd0        /media/floppy0  auto    rw,user,noauto,exec,utf8 0       0

```

end

### 存储容量计算

鲲鹏存储服务器配置：2个鲲鹏48核 CPU，256G内存，2个480G SSD，1个2G缓存raid卡，5个7.68T NVME SSD，2个双口万兆网卡，2个千兆四口网卡，双电源。

采用三副本的存储设计，：

逻辑可以用整体容量：`(4 * 5 * 7.68)/3` = 51.2 

实际可用容量：`51.2 * 0.92 * 0.9` = 42.3

因为硬盘7.6T 还要考虑磁盘实际容量与标称值的差值，一般是92% 再加上 预留一定空间用于磁盘故障恢复 一般是90%

### 存储/数据高可用：复制

数据复制（Replication）是指利用复制软件把数据从一个磁盘复制到另一个磁盘，生成一个数据副本。这个数据副本是数据处理系统直接可以访问的，不需要进行任何的数据恢复操作，这一点是复制与D2D备份的最大区别。

> Disk to Disk,D2D备份技术的设计初衷，旨在**加快**数据备份和容灾恢复的进程。D2D还是依赖数据恢复的操作的。

数据复制依据复制启动点的不同可分为同步复制、异步复制、基于数据增量的复制等几种。

* 同步复制，数据复制是在向主机返回写请求确认信号之前实时进行的；
* 异步复制，数据复制是在向主机返回写请求确认信号之后实时进行的；
* 基于数据增量的复制是一种非实时的复制方式，它依据一定的策略（如设定数据变化量门限值、日历安排等）来启动数据复制。

#### 同步复制

数据复制是在向主机返回写请求确认信号之前实时进行的.

应用于最高等级的容灾方案（RPO等于0）中，需要关闭主机Cache来保证数据一致性。对于连接生产中心和灾备中心的链路带宽和QoS要求很高，一般采用光纤直连、波分设备来保证，方案部署成本很高。

#### 异步复制

数据复制是在向主机返回写请求确认信号之后实时进行的

应用于较高级别的容灾方案（RPO接近于0）中，无法有效保证数据一致性（关闭主机中的Cache和快照都不适合）。但对于连接生产中心和灾备中心的链路带宽和QoS要求一般，理论上带宽只要达到“日新增数据量/（24×3600×8）”即可。

#### 增量复制

应用于较高级别的容灾方案（RPO小于1小时）中，可以结合快照技术有效保证数据一致性。对于连接生产中心和灾备中心的链路带宽和QoS要求一般，理论上带宽只要达到“数据增量/复制间隔”即可。

#### 存储双活

**当存储服务器中断时，通过存储仲裁、波分链路，实现存储永不中断**，存储双活架构，为两个数据中心存储同时提供读写服务，且整个存储系统架构全冗余，任意数据中心故障时，另外一个数据中心有一份存储设备和相同数据可用，最大化提高了业务连续性。

#### 数据备份

数据双活也是一直架构设计实现，当然有挂掉的风险，还是需要一份备份的。

一般数据备份采用定期全量备份（如七天），更短周期数据增量备份（如一天或秒级）的方式。具体的实现原理有多种：硬盘分区级的物理备份（硬盘虚机快照等）、文件级的物理备份（Veritas等）、数据库级的逻辑备份（MysqlDump、Oracle DataGuard等）。

### RPO and RTO

RPO（Recovery Point Objective）即数据恢复点目标，主要指的是业务系统所能容忍的数据丢失量。

RTO（Recovery Time Objective）即恢复时间目标，主要指的是所能容忍的业务停止服务的最长时间，也就是从灾难发生到业务系统恢复服务功能所需要的最短时间周期。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/RTO-and-RPO.png)





### LUN高可用：热迁移只丢一个包

服务器识别到的最小的存储资源，就是LUN级别的。那么主机的HBA卡看到的存储上的存储资源就主要 靠 两个东西来定位，一个就是存储系统的控制器(Target)，一个就是LUN ID，这个LUN是由存储的控制系统给定的，是存储系统的某部分存储资源。

在给客户部署Hyper-V集群时，遇到一个问题：如果集群中的某台虚机需要访问SAN里的LUN，如何确保其保持高可用性？

以下是该群集的磁盘结构图示：有2个CSV卷，还有2个LUN，将用作虚机的PassTrough磁盘。

在故障转移群集管理器里，给这台虚机添加存储，确保把2个Pass-through磁盘添加到该虚机里。

打开虚机的属性对话框，切换到“依赖关系”标签页，把这两个Pass-through磁盘添加为依赖条件，这样虚机在迁移的时候，会自动带上两个LUN一起迁移。

配置好以后，**这台虚机可以实时迁移，经Ping测试发现，会丢2个包而其他正常虚机会丢1个包），原因是所挂接的这2个Pass-trough磁盘需要重新连接到新的Node上。**
通常情况下，这并不会有问题，因为只有在对Hyper-V服务器进行维护时，才需要实时迁移。
如果不希望看到多1个丢包，**可以在虚机里直接用ISCSI Initiator连接这2个LUN，而不要用Pass-trough的方式经由HOST去转接。**

另外，我们还可以修改实时迁移所使用的网络，将其优先设置为使用Live Migration这个网络。

https://blog.51cto.com/markwin/620100

#### multi-LUN管理

**现在，存储网络越来越发达了，一个lun有多条通路可以访问也不是新鲜事了。** 
服务器使用多个HBA连接到存储网络，存储网络又可能是由多个交换设备组成，而存储系统又可能有多个控制器和链路，lun到服务器的存储网络链路又可能存在着多条不同的逻辑链路。那么，必然的，同一个physical lun在服务器上必然被识别为多个设备。因为os区别设备无非用的是总线，target id，lun id来，只要号码不同，就认为是不同的设备。 
由于上面的情况，多路径管理软件应运而生了，比如emc的powerpath，这个软件的作用就是让操作系统知道那些操作系统识别到lun实际上是一个真正的physical lun，具体的做法，就是生成一个特别的设备文件，操作系统操作这个特殊的设备文件。而我们知道，设备文件+driver+firmware的一个作用，就是告诉操作系统该怎么使用这个设备。那么就是说，多路径管理软件从driver和设备文件着手，告诉了操作系统怎么来处理这些身份复杂的lun。 

### 精简配置 and 厚配置

使用精简配置时，无限卷的大小不受与其关联的聚合大小的限制。您可以在小容量存储上创建大容量卷，仅在需要时添加磁盘。例如，可以使用一个仅有 250 TB 可用空间的聚合创建一个 500 TB 的卷。聚合提供的存储空间仅在写入数据时才会使用。精简配置也称为聚合过量。

精简配置的另一种替代方案是厚配置（即"完全模式"），它会立即分配物理空间，而不考虑该空间是否已用于数据存储。已分配的空间无法供任何其他卷使用。在使用厚配置时，将在创建卷时从聚合中为卷分配所需的全部空间。

### HHD and SSD and

传统硬盘（HDD，Hard Disk Drive的缩写）: 即硬盘驱动器，最基本的电脑存储器，我们电脑中常说的电脑硬盘

固态硬盘（Solid State Drive）: 用固态电子存储芯片阵列而制成的硬盘。

混合硬盘（hybrid harddrive，HHD）: 是既包含传统硬盘又有闪存（flashmemory）模块的大容量存储设备。

### NAND闪存

闪存存储是一种数据存储技术，基于可电子编程的高速内存。顾名思义，闪存存储可以闪速写入数据并执行随机 I/O 操作。

闪存存储采用集成电路技术，是一种固态技术，也就是说，它没有可拆卸的部件。

**比如iPhone。**闪存是16G、64G、128G ROM 断电不影响数据存储。内存是1GB RAM 断电影响数据存储。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/闪存and量子隧道.jpg)



闪存存储利用一种称为闪存的非易失性内存（非易失性内存标准 (Nonvolatile Memory Express, NVMe）。这种非易失性内存不需要通过供电来维护所存储数据的完整性，因此即使断电，数据也不会丢失。换句话说，当磁盘断电后，非易失性内存不会“忘记”所存储的数据。

#### NVMe

NVMe是目标是取代SATA SSD硬盘接口，主要应用在计算机平台。NVMe硬盘需要M.2 插槽和 NVMe 支持。NVMe是数据中心高性能固态盘的新选择。

NVMe 是一款用于通过 PCI Express (PCIe) 总线访问闪存存储的接口协议。不同于只能使用单一串行命令序列的传统全闪存架构，NVMe 支持成千上万个并行序列，每一个序列都能支持成千上万个并发命令。

支持PCIe总线，NVMe协议的SSD，实际传输速度将超过1000MB/s。

Non-Volatile Memory Express (NVMe) 是专为固态存储器设计的新型传输协议。SATA (Serial Advanced Technology Attachment) 仍是行业存储协议标准，但它并非专为固态硬盘等闪存存储器设计。

##### SATA 弊端

SATA取代IDE并行接口之后，SATA串行接口就一直在不紧不慢地提速，1.5Gbps、3Gbps、6Gbps……面对机械硬盘，这一切都绰绰有余，但是这几年固态硬盘突飞猛进，SATA接口完全吃不消了。新的接口不断被提出来，mSATA，SATA Express，M.2，U.2等等。

#### NVMe-oF

基于网络的NVMe，NVMe over Fabrics。

NVMe-oF 是存储系统中的一个主机端接口，可以通过远程直接内存访问 (Remote Direct Memory Access, RDMA) 或光纤通道网络结构提供许多相关的 NVMe 功能。利用 NVMe-oF，可以横向扩展到许多 NVMe 设备，甚至支持远距离 NVMe 设备。

#### SCM

存储级内存 ，Storage Class Memory。

### DAS： without any network device

Direct Attached Storage（DAS）是专用的数字存储设备，可通过电缆直接连接到服务器或PC。 

Direct attached storage (DAS) is the connection between storage and a server/servers without a storage network or any network device, such as a router, switch, hub, or director. 

DAS can have many different types of interfaces that are connected to a server, such as a Host Bus Adapter (HBA), IDE/ATA, SATA, SAS, SCSI, eSATA and Fibre Channel (FC). These interfaces are **also** applied to other types of network storage. We will have further discussion on this in the next section.

#### Internal DAS

With internal DAS, the storage disk/disks are directly located inside of a hosting server. The common interface connection is through HBA. The main function of HBA is to provide high-speed bus connectivity or communication channels between a host server and storage devices or a storage network.

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/hba-cards.jpg)

With an internal DAS, the physical space is limited. If a business application needs further expansion for larger storage capacity and the physical space within the host server is not available, then we have to locate a disk array externally.

容易存储受限。

#### External DAS：解耦

存储和计算解耦，可以便捷的扩容磁盘，但是受到距离限制scsi和光纤不能无限拉长吧。

For an external DAS arrangement, the storage disk array is still directly attached to a server without any network device. However, it interfaces with the host server with different protocols. **The popular interface protocols are SCSI and Fibre Channel (FC).** In comparison with internal DAS, external DAS overcomes the issues of physical space and distance between the host server and **storage disk array**. In addition, the storage array can be shared by more than one host server。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/das-scsi-fc.jpg)

#### IDE

集成驱动器电子设备（IDE）是用于将主板连接到存储设备（例如硬盘驱动器和CD-ROM / DVD驱动器）的标准接口。 原始的IDE具有一个16位接口，该接口将两个设备连接到单条带状电缆。 这种具有成本效益的IDE设备具有自己的电路，并包括一个集成的磁盘驱动器控制器。 在IDE之前，控制器是独立的外部设备。

#### SCSI:duck:

Small Computer System Interface小型计算机系统接口，一种用于计算机和智能设备之间（硬盘、软驱、光驱、打印机、扫描仪等）系统级接口的独立处理器标准。 **SCSI是一种智能的通用接口标准。**

> SCSI 很通用这就是它的优点，ceph很棒但是VMWare EXSi、Windows/Solaris操作系统是无法直接使用librbd(librbd是基于librados的用户态接口库，而krbd是继承在GNU/Linux内核中的一个内核*模块*。)，通用性差了。

主机带有一个SCSI控制器与SCSI设备相连，我们把控制SCSI进行数据存储的一端叫Initiator,而把SCSI设备（存储数据的）叫做Target

主机通过控制器与Target相连，而Target也可以通过SCSI总线与其他的SCSI设备相连，但最后一般都会连接一个终结器。

##### SCSI总线

SCSI的总线分为宽带和窄带两种，宽带有16个接口，除了一个连接initiator外，最多可以连接15个Target，而窄带有8个接口，最多链接7个Target

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/scsi总线.png)

系统中的每个SCSI设备都必须有自己唯一的SCSI ID（即target ID），SCSI ID实际上就是这些设备的地址，而每个target上可以连接多个逻辑单元（一个逻辑单元对应一个SCSI设备），用LUN（Logical Unit number）逻辑单元号区别不同的逻辑单元，每个SCSI ID上最多有32 个LUN（宽带的），一个LUN对应一个逻辑设备（SCSI设备）

##### SCSI终结器

​    如果SCSI总线保持开放状态，沿总线发送的电信号会反射回来，从而干扰设备和SCSI控制器之间的通信。解决方法是终结总线，用电阻电路闭合每一端。如果总线同时支持内部和外部设备，则必须终结每个系列的最后一个设备。

##### SCSI target

**target**端即磁盘阵列或其他装有磁盘的主机，通过iscsi target工具将磁盘空间映射到网络上，**initiator**端就可以寻找发现并使用该磁盘。

##### SCSI initiator

**initiator**作为ISCSI的使用者，发送SCSI命令，用于寻找发现网络上的**target**，并使用网络上的磁盘。

##### SCSI和IDE

SCSI与IDE 相比

1.IDE的工作方式需要CPU的全程参与，CPU读写数据的时候不能再进行其他操作，而SCSI接口，则完全通过独立的高速的SCSI卡来控制数据的读写操作，CPU就不必浪费时间进行等待，显然可以提高系统的整体性能。

2.SCSI的扩充性比IDE大，一般每个IDE系统可有2个IDE通道，总共连4个IDE设备，而SCSI接口可连接7—15个设备，比IDE要多很多，而且连接的电缆也远长于IDE。

3.虽然SCSI设备价格高些，与IDE相比,SCSI的性能更稳定、耐用，可靠性也更好。

#### SSD 接口-总线-协议

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/das-int-protocol.png)



以固态硬盘的物理接口SATA和M.2为例说明，阐述接口、总线和协议的关系。

光凭接口类型并不能完全明确硬盘的质量和性能，还有总线、协议、颗粒和主控等等，其中总线和协议适合接口息息相关的，用形象点的话来说

**传输总线是公路、数据协议是交通规则、物理接口可以类比为汽车**，要从硬盘这个仓库中运输数据，车到底能跑多快由这三点共同来决定，我之前画过一张完整的科普图

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/stat-and-m2.jpg)

PCIe总线是高速公路，速度很快，高性能的固态硬盘走的都是这条路，SATA总线是一条玩去的小路，性能稍差一些的硬盘会走这条路（注意这个和SATA接口是两回事）

在两条路上分别有交通规则，也就是数据协议，一共两类，**AHCI和NVMe，AHCI只能运用在SATA总线的这条路上，NVMe则可以运用在PCIe这条路上**

SATA接口的硬盘只能走SATA总线的小路，而M.2固态则既可以走大路也可以走小路。

此外，固态盘颗粒**SLC、MLC、TLC、QLC**也有区别。

#### Raid

RAID是Redundant Array of Inexpensive Disks的缩写，即独立磁盘冗余阵列，通常简称为磁盘阵列。简单地说， RAID 是由多个独立的高性能磁盘驱动器组成的磁盘子系统，从而提供比单个磁盘更高的存储性能和数据冗余的技术。

RAID目的是提升磁盘的存取速度，CPU、内存及磁盘间的不平衡将使CPU及内存的改进形成浪费。

磁盘阵列中针对不同的应用使用的不同技术，称为RAID 等级。而每一等级代表一种技术。目前业界最经常应用的RAID等级是RAID 0~RAID 5。

* RAID 0： Striped Disk Array without Fault Tolerance

  RAID 0是把所有的硬盘并联起来成为一个大的硬盘组。其容量为所有属于这个组的硬盘的总和。所有数据的存取均以并行分割方式进行。由于所有存取的数据均以平衡方 式存取到整组硬盘里，存取的速度非常快。越是多硬盘数量的RAID 0阵列其存取的速度就越快。但RAID 0有一个致命的缺点–就是它跟普通硬盘一样没有一点的冗余能力。

* RAID 1： Mirroring

  RAID 1是硬盘镜像备份操作。由两个硬盘所组成。其中一个是主硬盘而另外一个是镜像硬盘，所有主硬盘的数据会不停地镜像到另外一个硬盘上， 故RAID 1具有很高的冗余能力。达到最高的100%。可是正由于这个镜像做法不是以算法操作，故它的容量效率非常的低，只有50%。RAID 1只支持两个硬盘操作。

* RAID 0 + 1：Mirroring and Striping

  RAID 0+1即由两组RAID 0的硬盘作RAID 1的镜像容错，但RAID 0+1的容量效率还是与RAID 1一样只有50%，故同样地没有被普及使用。

* RAID 3： Striping with dedicated parity

  RAID 3在安全方面以奇偶校验（parity check）做错误校正及检测，只需要一个额外的校检磁盘（parity disk）。

  若你有n块盘，其中1块盘作为校验盘，剩余n-1块盘相当于作RAID 0同时读写，当其中一块盘坏掉时，可以通过校验码还原出坏掉盘的原始数据。这个校验方式比较特别，奇偶检验，1 XOR 0 XOR 1=0，0 XOR 1 XOR 0=1，最后的数据时校验数据，当中间缺了一个数据时，可以通过其他盘的数据和校验数据推算出来。但是这有个问题，由于n-1块盘做了RAID 0，每一次读写都要牵动所有盘来为它服务，而且万一校验盘坏掉就完蛋了。最多允许坏一块盘。n最少为3.

  很明显，校验盘单点了。感觉raid 3也不受待见。

* RAID 5：Striping with distributed parity

  RAID 5也是一种具容错能力的RAID 操作方式，但与RAID 3不一样的是RAID 5的容错方式不应用专用容错硬盘，容错信息是平均的分布到所有硬盘上。在RAID 3的基础上有所区别，同样是相当于是1块盘的大小作为校验盘，n-1块盘的大小作为数据盘，但校验码分布在各个磁盘中，不是单独的一块磁盘，也就是分布式校验盘，这样做好处多多。最多坏一块盘。n最少为3.

* RAID 6：在RAID 5的基础上，又增加了一种校验码，和解方程似的，一种校验码一个方程，最多有两个未知数，也就是最多坏两块盘。

##### raid 组合：50

RAID 5与RAID 0的组合，先作RAID 5，再作RAID 0，也就是对多组RAID 5彼此构成Stripe访问。由于RAID 50是以RAID 5为基础，而RAID 5至少需要3颗硬盘，因此要以多组RAID 5构成RAID 50，至少需要6颗硬盘。以RAID 50最小的6颗硬盘配置为例，先把6颗硬盘分为2组，每组3颗构成RAID 5，如此就得到两组RAID 5，然后再把两组RAID 5构成RAID 0。

RAID 50在底层的任一组或多组RAID 5中出现1颗硬盘损坏时，仍能维持运作，不过如果任一组RAID 5中出现2颗或2颗以上硬盘损毁，整组RAID 50就会失效。

RAID 50由于在上层把多组RAID 5构成Stripe，性能比起单纯的RAID 5高，容量利用率比RAID5要低。比如同样使用9颗硬盘，由各3颗RAID 5再组成RAID 0的RAID 50，每组RAID 5浪费一颗硬盘，利用率为(1-3/9)，RAID 5则为(1-1/9)。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/raid50.jpg)



##### raid and cc

 CC是RAID卡保证数据完整性的手段，对VD（RAID0无CC）进行检测，确保数据一致性，并修复块错误，会占用RAID卡资源（并且数据重构的过程会影响RAID资源），所以存在影响性能的情况。
  CC操作给数据完整性性提供了很好的帮助，不建议直接关闭，但是可以结合业务运行，实际影响情况调整周期。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/raid1-and-cc.jpg)

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/raid2-and-cc.jpg)

 end

### WWID and UUID

SCSI标准中，每个SCSI磁盘都会有一个WWID，与网卡的MAC地址很相似，其在全世界唯一的。磁盘的WWID永久不变，Linux上可以通过如下方法查看SCSI磁盘WWID及其相应的磁盘标识。

* 查看/dev/disk/by-id目录

  ```bash
  $  ll /dev/disk/by-id/
  ```

* scsi_id 命令

  ```bash
  $ scsi_id -g /dev/sda
  $ scsi_id --whitelist /dev/sda 
  ```

UUID是创建文件系统时生成的，用于标记该文件系统，与WWID类似，UUID也是唯一的。因此，UUID用来标识SCSI磁盘，也能保证磁盘路径的不变。Linux中/dev/disk/by-uuid目录下，包含所有已创建文件系统的磁盘设备及其UUID间的对应关系。

```bash
$ ll /dev/disk/by-uuid/

lrwxrwxrwx 1 root root 10 Aug  3 18:26 7054388f-d4c1-4070-8c09-1cd01d5c110d -> ../../sda1
lrwxrwxrwx 1 root root 10 Aug  3 18:26 d2dc2acf-621a-43b6-9a51-45165d9f4e97 -> ../../dm-1
lrwxrwxrwx 1 root root 10 Aug  3 18:26 d8cd03b7-58cf-4e11-98d7-938c80f7d4d1 -> ../../dm-0
```

为了实现Linux系统重启后系统上的目录和文件系统间的绑定关系不变，/etc/fstab文件中应使用uuid进行标识，因为/etc/fstab中记录的是文件系统信息，因此，其中只能使用UUID而不能使用WWID，例如：

```bash
$ cat /etc/fstab

/dev/mapper/vg_rac1-lv_root /                       ext4    defaults        1 1
UUID=7054388f-d4c1-4070-8c09-1cd01d5c110d /boot                   ext4    defaults        1 2
/dev/mapper/vg_rac1-lv_swap swap                    swap    defaults        0 0
tmpfs                   /dev/shm                tmpfs   defaults        0 0
devpts                  /dev/pts                devpts  gid=5,mode=620  0 0
sysfs                   /sys                    sysfs   defaults        0 0
proc                    /proc                   proc    defaults        0 0
```

UUID进行文件系统挂载时，可通过blkid查看分区UUID，通过UUID挂载文件系统时，需指定-t文件系统类型。

### multipath

服务器端，同一个LUN或WWID或UUID通常对应多个路径，这为服务器访问同一设备提供了多个路径选择，即可提高磁盘设备访问的性能，同时也提供了磁盘设备的高可用性。一般地，一个存储LUN配置路径数的算法为
LUN绑定某台服务器的`HBA数*存储机头数*光纤交换机数`
通常情况下，存储的一个LUN会绑定服务器的两块HBA，而同一存储服务器里有两台机头，为了冗余也会配备两台交换机，这样，一个LUN的访问路径数为：2\*2\*2=8
因此，多路径映射后，/dev目录下会有8个/dev/sd*磁盘设备对应同一WWID，对于Lunix自带的multipath，可以通过如下命令查看多路径：

```bash
$ multipath -ll
mpath0 (360060e80058e980000008e9800000007)

[size=20 GB][features="0"][hwhandler="0"]

\_ round-robin 0 [prio=1][active]

\_ 3:0:0:7 sdaa 65:160 [active][ready]

\_ round-robin 0 [prio=1][enabled]

\_ 4:0:0:7 sdas 66:192 [active][ready]

\_ round-robin 0 [prio=1][enabled]

\_ 5:0:0:7 sdbk 67:224 [active][ready]

\_ round-robin 0 [prio=1][enabled]

\_ 2:0:0:7 sdi 8:128 [active][ready]

这说明，已由四条链路sdaa/sdas/sdbk/sdi复合成一条链路，设备名为mpath0。
```

#### 配置维护

```bash
# 安装配置
$ modprobe dm-multipath
$ service multipathd start
$ multipath -v0
```

配置文件只有一个：/etc/multipath.conf 。配置前，请用fdisk -l 确认已可正确识别盘柜的所有LUN，HDS支持多链路负载均衡，因此每条链路都是正常的；而如果是类似EMC CX300这样仅支持负载均衡的设备，则冗余的链路会出现I/O Error的错误。

默认情况下，multipath会把所有设备都加入到黑名单（devnode "*"），也就是禁止使用。我们首先需要取消该设置，把配置文件修改为类似下面的内容：

```bash
devnode_blacklist {

#devnode "*"

devnode "hda"

wwid 3600508e000000000dc7200032e08af0b

}
```

这里禁止使用hda，也就是光驱。另外，还限制使用本地的sda设备,这个wwid，可通过下面的命令获得

```bash
# scsi_id -g -u -s /block/sda
3600508e000000000dc7200032e08af0b
```

使用错误的`""`导致，挂载失败

```bash
[root@cloudos02 ~]# diff -u /etc/multipath.conf /etc/multipath.conf.56
--- /etc/multipath.conf 2022-05-06 17:34:47.265091695 +0800
+++ /etc/multipath.conf.56      2022-05-06 17:34:00.655984526 +0800
@@ -30,8 +30,8 @@
 wwid "3600c0ff000293a07a782626201000000"
 wwid "3600c0ff000293a072629696201000000"
 wwid "3600c0ff000293a070d29696201000000"
-wwid "3600c0ff000293a07b343756201000000"
-wwid "3600c0ff000293a07d443756201000000"
-wwid "3600c0ff000293a07f643756201000000"
-wwid "3600c0ff000293a077d43756201000000"
+wwid “3600c0ff000293a07b343756201000000”
+wwid “3600c0ff000293a07d443756201000000”
+wwid “3600c0ff000293a07f643756201000000”
+wwid “3600c0ff000293a077d43756201000000”
 }
 
 [root@cloudos02 ~]# cat /etc/multipath.conf
defaults {
  user_friendly_names   "yes"
  path_checker "tur"
  prio "const"
  path_grouping_policy "group_by_prio"
  no_path_retry 25
  max_fds "max"
  failback "immediate"
}
blacklist {
    wwid "3600605b01133c01029e93d0f091e1935"
    wwid "3600605b01133c01029e9488e081c1ab7"

}
blacklist_exceptions {
    property "(ID_SCSI|ID_WWN)"

wwid "3600c0ff000293a07bbe15e6201000000"
wwid "3600c0ff000293a071ae15e6201000000"
wwid "3600c0ff000293a07f1e05e6201000000"
wwid "3600c0ff000293a0759e05e6201000000"
wwid "3600c0ff000293a072be05e6201000000"
wwid "3600c0ff000293a07a7e05e6201000000"
wwid "3600c0ff000293a0793e25e6201000000"
wwid "3600c0ff000293a0760e15e6201000000"
wwid "3600c0ff000293a073fe15e6201000000"
wwid "3600c0ff000293a07b5e25e6201000000"
wwid "3600c0ff000293a071282626201000000"
wwid "3600c0ff000293a075d82626201000000"
wwid "3600c0ff000293a07a782626201000000"
wwid "3600c0ff000293a072629696201000000"
wwid "3600c0ff000293a070d29696201000000"
wwid "3600c0ff000293a07b343756201000000"
wwid "3600c0ff000293a07d443756201000000"
wwid "3600c0ff000293a07f643756201000000"
wwid "3600c0ff000293a077d43756201000000"
}
```

end

### 存储网络

storage network

如果每个新的应用服务器都要有它自己的存储器，这样造成数据处理复杂，随着应用服务器的不断增加，网络系统效率会急剧下降。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/storage-networks.jpg)

针对I/O是整个网络系统效率低下的瓶颈问题，最有效的办法是：**将数据从通用的应用服务器中分离出来以简化存储管理。**

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/storage-networks-2.jpg)

将存储器从应用服务器中分离出来，进行集中管理。这就是所说的存储网络（Storage Networks）。 

**使用存储网络的好处**：

* 统一性：形散神不散，在逻辑上是完全一体的。 
* 实现数据集中管理，因为它们才是企业真正的命脉。 
* 容易扩充，即收缩性很强。 
* 具有容错功能，整个网络无单点故障。 

#### 实现方式

存储网络有两种不同的实现手段，即SAN(Storage Area Networks)存储区域网络和NAS（Network Attached Storage）网络接入存储。

* SAN：通过专用光纤通道交换机访问数据，采用SCSI、FC-AL接口。
* NAS：用户通过TCP/IP协议访问数据，采用业界标准文件共享协议如：NFS、HTTP、CIFS实现共享。 



### SAN

Storage Area Network-->存储区域网络，它首先是一个网络，而不是指存储设备，这个网络是专门用来给主机连接存储设备使用的。SAN网络与LAN网络相隔离，存储数据流不会占用业务网络带宽。

A SAN is block-based storage, leveraging a high-speed architecture that connects servers to their logical disk units (LUNs). A **LUN** is a range of blocks provisioned from a pool of shared storage and presented to the server as a logical disk. 

SAN解决方案是从基本功能剥离出存储功能，所以运行备份操作就无需考虑它们对网络总体性能的影响。



#### SAN类型/主要协议

FC-SAN: 利用光纤通道/FC协议上加载SCSI协议来达到可靠的块级数据传输

IP-SAN: 使用TCP/IP通道加载SCSI协议

FCIP：在IP上传输的FC数据帧 （FCIP：Entire Fibre Channel Frame Over IP）

iFCP：Internet FC协议 （iFCP：Internet Fibre Channel Protocol）

iSCSI：Internet 小型计算机系统接口 （iSCSI：Internet Small Computer System Interface）

iSNS：Internet存储名称服务 （iSNS：Internet Storage Name Service）

NDMP：网络数据管理协议 （NDMP：Network Data Management Protocol）

##### IP-SAN连接方式

IP-SAN根据主机与存储的连接方式不同,可以分为三种:

1、以太网卡 + initiator软件  

2、硬件TOE网卡+ initiator软件  

3、 iSCSI HBA卡

- 以太网卡 + initiator软件：主机使用标准的以太网卡(NIC)与网络进行连接。iSCSI 和TCP/IP协议栈功能通过主机CPU进行处理。这种方式直接使用传统主机系 统通用的NIC卡,所以成本最低,但是由于需要占用CPU资源进行iSCSI协议和 TCP/IP协议处理,所以会导致主机系统性能的下降。
- 硬件TOE网卡+ initiator软件：主机使用TOE(TCP offload Engine,TCP卸载引擎)网卡 ,iSCSI协议的功能仍然由主机的CPU完成,但是TCP协议处理则交由TOE网卡完成,从而有效减轻了主机CPU的负担
- iSCSI HBA卡：采用这种方式的主机,其iSCSI和TCP/IP协议功能均由iSCSI HBA卡完成,对主机的开销占用最小。

#### FC-SAN

有人说，FC存储: FC在虚拟机场景下根本行不通。

FC-SAN还挺麻烦的--

Functionally, a fibre channel SAN is hardly different from an iSCSI SAN. Fibre channel is actually older technology and uses dedicated hardware instead: special controllers on the SAN hardware, host bus adapters or HBAs on the client machines, and special fibre channel cables and switches to interconnect the components.

Like iSCSI, a Fibre Channel SAN transfers SCSI commands between initiators and targets establishing a connectivity that is almost identical to direct disk access. However, whereas iSCSI uses TCP/IP, a Fibre Channel SAN uses **Fibre Channel Protocol (FCP).** The same concepts from the iSCSI SAN,apply equally to the Fibre Channel SAN. Again, generic and vendor-specific Storage Connect plug-ins exist. Your storage hardware supplier can provide proper documentation with the Oracle VM Storage Connect plug-in.

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/fc-san.jpg)

##### VM access FC-SAN

When a virtual machine interacts with its virtual disk stored on a SAN, the following process takes place:

1. When the guest operating system in a virtual machine reads or writes to a SCSI disk, it sends SCSI commands to the virtual disk.
2. Device drivers in the virtual machine’s operating system communicate with the virtual SCSI controllers.
3. The virtual SCSI controller forwards the command to the VMkernel.
4. The VMkernel performs the following tasks.
   1. Locates the appropriate virtual disk file in the VMFS volume.
   2. Maps the requests for the blocks on the virtual disk to blocks on the appropriate physical device.
   3. Sends the modified I/O request from the device driver in the VMkernel to the physical HBA.
5. The physical HBA performs the following tasks.
   1. Packages the I/O request according to the rules of the FC protocol.
   2. Transmits the request to the SAN.
6. Depending on a port the HBA uses to connect to the fabric, one of the SAN switches receives the request. The switch routes the request to the appropriate storage device.



#### iSCSI:star:

iSCSI实现的是IP SAN，数据传输基于以太网。

传统的SCSI技术是存储设备最基本的标准协议，但通常需要设备互相靠近并用SCSI总线连接，因此受到物理环境的限制。

**iSCSI(Internet Small Computer System Interface)，顾名思义，iSCSI是网络上的SCSI，也就是通过网络连接的SCSI。**它是由IBM公司研究开发用于实现在IP网络上运行SCSI协议的存储技术，能够让SCSI接口与以太网技术相结合，使用iSCSI协议基于以太网传送SCSI命令与数据，克服了SCSI需要直接连接存储设备的局限性，使得可以跨越不同的服务器共享存储设备，并可以做到不停机状态下扩展存储容量。

##### iSCSI 核心

**iSCSI 利用了 TCP/IP 的 port 860 和 3260 作为沟通的渠道**，透过两台计算机之间利用 iSCSI 的协议来交换 SCSI 命令，让计算机可以透过高速的局域网集线来把 SAN 模拟成为本地的存储装置。

本质上，iSCSI 让两个主机通过 IP 网络相互协商然后交换 SCSI 命令，这样一来，ISCSI 就是用广域网仿真了一个常用的高性能本地存储总线，从而创建了一个存储局域网 (SAN)，不像某些 SAN 协议，iSCSI 不需要专用的电缆，它可以在已有的交换和 IP 基础架构上运行，然而，如果不使用专用的网络或者子网 (LAN 或者 VLAN)，iSCSI SAN 的部署性能可能会严重下降，于是，iSCSI 尝尝被认为是光纤通道的一个低成本替代方法，光纤通道是需要专用的基础架构，但是，基于以太网的光纤通道则不需要专用的基础架构。

iSCSI 技术有以下三个革命性的改变：

 (1) 把原来只用于本机的 SCSI 协议透过 TCP/IP 网络发送，使连接距离可作无限的区域延伸。 

(2) 连接的服务器数量无线 (原来的 SCSI-3 的上限是 15) 

(3) 由于是服务器架构，因此也可以实现在线扩容已致动态部署

##### iSCSI架构

iSCSI架构依然遵循典型的SCSI模式即：initiator and target

**iSCSI是一种使用TCP/IP协议，在现有IP网络上传输SCSI块命令的工业标准**，它是一种在现有的IP网络上无需安装单独的光纤网络即可同时传输消息和块数据的突破性技术。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/iscsi-arc.png)

##### iscsi 数据封装方式

initiator向target发起scsi命令后，在数据报文从里向外逐层**封装SCSI协议报文、iSCSI协议报文、tcp头、ip头。**

封装是需要消耗CPU资源的，如果完全以软件方式来实现iscsi，那么所有的封装过程都由操作系统来完成。在很繁忙的服务器上，这样的资源消耗可能不太能接受，但好在它是完全免费的，对于不是非常繁忙的服务器采用它可以节省一大笔资金。

除了软件方式实现，还有硬件方式的initiator(TOE卡和HBA卡)，通过硬件方式实现iSCSI。由于它们有控制芯片，可以帮助或者完全替代操作系统对协议报文的封装。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/iscsi-package.png)

Initiator驱动程序最差、TOE居中、iSCSI HBA卡最佳。但是iSCSI HBA只能走iSCSI协议，而无法透过NFS(Network File System，SUN制定)或CIFS(Common Internet File System，微软制定)等档案系统协议与应用服务器沟通。但Initiator驱动程序及TOE则同时支持iSCSI、NFS及CIFS三种协议。

###### OS+CPU

消耗服务器资源完成iscsi协议数据包封装

###### TOE卡

TOE网卡是指带有TOE功能的网卡。TOE是TCP/IP Offload Engine的缩写，它是网卡采用的一种新技术，采用TOE技术的网卡自身支持TCP/IP协议栈的处理，这样能够减轻应用服务器的处理负担。TOE网卡通常用于高速网络接口，如千兆和万兆网络。

TOE卡，操作系统首先封装SCSI和iSCSI协议报文，而TCP/IP头则交由TOE内的芯片来封装，这样就能减少一部分系统资源消耗。

###### HBA卡

iSCSI HBA（Host Bus Adapter）卡是专用的连接iSCSI设备的主机适配卡，它支持TOE的同时还支持对iSCSI协议的处理，能够更进一步的减轻应用服务器的处理负担。

HBA卡，操作系统只需封装SCSI，剩余的iSCSI协议报文还有TCP/IP头由HBA芯片负责封装。

##### iSCSI Client/initiator

 iSCSI客户端或主机（也称为iSCSI initiator），诸如服务器，连接在IP网络并对iSCSI target发起请求以及接收响应。每一个iSCSI主机通过唯一的IQN来识别，类似于光纤通道的WWN。

要在IP网络上传递SCSI块命令，必须在iSCSI主机上安装iSCSI驱动。推荐通过GE适配器（每秒1000 megabits）连接至iSCSI target。如同标准10/100适配器，大多数Gigabit适配器使用Category 5 或Category 6E线缆。适配器上的各端口通过唯一的IP地址来识别。

##### iSCSI Target

iSCSI target是接收iSCSI命令的设备。此设备可以是终端节点，如存储设备，或是中间设备，如IP和光纤设备之间的连接桥。

每一个iSCSI target通过唯一的IQN来标识，存储阵列控制器上（或桥接器上）的各端口通过一个或多个IP地址来标识。



### NAS

NAS（Network Attached Storage—网络附加存储），即将存储设备通过标准的网络拓扑结构（例如以太网），连接到一群计算机上。NAS是部件级的存储方法，它的重点在于帮助工作组和部门级机构解决迅速增加存储容量的需求。需要共享文件的工程小组就是典型的例子。

**NAS没有解决与文件服务器相关的一个关键性问题，即备份过程中的带宽消耗。**与将备份数据流从LAN中转移出去的存储区域网（SAN）不同（存储内网实现备份即存储的东西流量？），NAS仍使用网络进行备份和恢复。NAS 的一个缺点是它将存储事务由并行SCSI连接转移到了网络上。这就是说LAN除了必须处理正常的最终用户传输流外，还必须处理包括备份操作的存储磁盘请求。



#### NAS 协议

* **公共 Internet 文件服务/服务器消息块 (Common Internet File Services / Server Message Block, CIFS/SMB)。**这是 Windows 通常使用的协议。
* **网络文件系统 (NFS)。**NFS 最早为 UNIX 服务器而开发，也是通用的 Linux 协议。

#### NAS通讯过程

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/nas-and-nfs.png)

 ① 主机客户端通过NFS Client对NAS上的一个输出目录/nas/export进行挂载操作，将其挂载到本地/mnt/nas路径下；

② 主机客户端某程序发起对/mnt/nas/nas.txt文件的读取操作，读取偏移从0字节往后1024字节（即读取文件的前1024字节）。该动作通过调用操作系统提供的文件操作API执行，例如Read();

③ I/O请求发送到NFS client，NFS client知道/mnt/nas路径对应得其实是NAS server的/nas/export目录，所以NFS client将上层下发的这个读取请求封装成NFS协议规定的标准格式通过网络传送到NFS server；

④ NFS server 接收到请求后，将请求通过操作系统API传送到文件系统模块处理；

⑤ 文件系统接收到针对/nas/export/nas.txt文件的读取请求后，首先查询缓存内是否有对应数据，如果有，直接返回结果；如果没有，则需要将这段字节所落入的底层存储空间的块信息取回，通过查询INode表等元数据获得。得到底层块位置后，文件系统通过调用操作系统API将块的读取请求交给卷管理层；

⑥ 卷管理层是将底层物理磁盘设备虚拟化封装的层次。当卷管理层收到针对某个卷某个段的LBA地址的请求后，它要进行翻译，将目标虚拟卷的地址翻译成对应得底层物理块设备地址；翻译完后，卷管理层将对应的目标地址请求再次通过操作系统API发生给驱动程序层。

⑦ 块设备驱动是负责对相应块设备进行I/O的角色，它将这些I/O发送给SCSI CDB Generator（SCSI指令的翻译中心）；

⑧ SCSI CDB Generator将对应得IO请求描述为SCSI协议标准格式。之后，这些指令被发送到SCSI/FC 适配器的驱动程序处；

⑨ 设备驱动程序接收到SCSI指令后，将其封装到对应得链路帧中通过内部总线网络传送到目标；

⑩ SCSI指令传送到目标设备，目标设备执行相应指令并返回结果。


### DAS and NAS and SAN： summary

https://www.cnblogs.com/zxqstrong/p/4727912.html

核心：DAS不借助任何网络设备！！！NAS和SAN有存储专网！！！

NFS 是以文件为单位的，共享出去的是文件
iSCSI是以block为单位，共享出去的是设备，端口：3260/tcp

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/DAS-and-NAS-and-SAN.png)







### WHY SCSI/iSCSI:star:

https://www.szsandstone.com/technical/article/42.html

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/why iscsi.jpg)

CEPH提供了对象、块、CEPHFS，块接口主要通过librbd或者KRBD支持，librbd是应用态接口，普通应用不能直接使用，**krbd只能部署在linux高内核版本系统**。但是企业环境里面常用到的VMWare EXSi、Windows/Solaris操作系统是无法直接使用librbd，而krbd也无法运行在这些系统中。所以一个标准的iSCSI接口就成为这些系统使用CEPH的最优方案



### iscsi报错： iscsi device not exist

错误原因：

基于iscsi协议挂载了 lun ID为 12 和 15 的块设备，然后由于块设备资源池不够，导致删除了 lun ID 12 和 15的块设备，调整块设备的资源分配，重新创建了3个块设备。但是 cloudos node没有自动卸载掉已挂载的iscsi device，导致lun id 12和15 被删除后，node还保留这挂载，导致cloudos plat读取lun 12和15块设备时出错。

解决办法：

根据报错日志，一个个删除掉 iscsi device

```bash
[INFO ] 15:16:30.912 [ForkJoinPool.commonPool-worker-2] c.h.c.c.storage.utils.SshUtils - command is :/lib/udev/scsi_id  -g  -u /dev/sdq
[INFO ] 15:16:31.020 [ForkJoinPool.commonPool-worker-2] c.h.c.c.storage.utils.SshUtils - exec cmd result is:
[INFO ] 15:16:31.020 [ForkJoinPool.commonPool-worker-2] c.h.c.c.storage.utils.SshUtils - exec cmd exitCode is:1

[INFO ] 14:50:28.867 [ForkJoinPool.commonPool-worker-2] c.h.c.c.s.service.PvServiceImpl - capacity: 2147483648
[INFO ] 14:50:28.867 [ForkJoinPool.commonPool-worker-2] c.h.c.c.storage.utils.SshUtils - command is :/lib/udev/scsi_id  -g  -u /dev/sdcj
[INFO ] 14:50:28.931 [ForkJoinPool.commonPool-worker-2] c.h.c.c.storage.utils.SshUtils - exec cmd result is:
[INFO ] 14:50:28.931 [ForkJoinPool.commonPool-worker-2] c.h.c.c.storage.utils.SshUtils - exec cmd exitCode is:1
[INFO ] 14:50:28.931 [ForkJoinPool.commonPool-worker-2] c.h.c.c.s.service.PvServiceImpl - query device blkid: /lib/udev/scsi_id  -g  -u /dev/sdcj error!
[INFO ] 14:50:28.931 [http-nio-8080-exec-10] c.h.c.c.s.service.PvServiceImpl - original iscsi info: [DaemonResponseVo{id='null', status=false, errorCode='null', operType=null, groupUuid='null', message='null', data=null, code='null', list=null, iscsiIqnDeviceEntityList=null, ip='null'}]
[INFO ] 14:50:28.931 [http-nio-8080-exec-10] c.h.c.c.s.service.PvServiceImpl - transferIscsiDevice: iscsi device not exist.
[INFO ] 14:50:28.931 [http-nio-8080-exec-10] c.h.c.c.s.service.PvServiceImpl - getTarget: after transferIscsiDevice, iscsi device is empty.
[ERROR] 14:50:28.931 [http-nio-8080-exec-10] c.h.c.c.s.controller.PvController - 获取iscsi target失败：无可用的iqn和lun

## 日志中的错误命令
[root@cloudos2-node1 ~]# ll /dev/disk/by-path/ | grep sdcj
lrwxrwxrwx. 1 root root 10 Mar 19 11:14 ip-172.200.20.11:3260-iscsi-iqn.2018-01.com.h3c.onestor:4dee75064df6429f8a163074f4995859-lun-15 -> ../../sdcj
[root@cloudos2-node1 ~]# /lib/udev/scsi_id  -g  -u /dev/sdcj
[root@cloudos2-node1 ~]# echo $?
1

## 删之前稳一点lsblk确认下是否真的不存在了
[root@cloudos2-node1 ~]# lscsi|grep sdcm
[root@cloudos2-node1 ~]# lsscsi |grep sdcm
[9:0:0:15]   disk    IET      VIRTUAL-DISK     0001  /dev/sdcm
[root@cloudos2-node1 ~]#
echo "1" > /sys/class/scsi_device/8\:0\:0\:15/device/delete
```

使用命令`echo 1 > /sys/block/device-name/device/delete`删除磁盘，device-name以sd开头，比如sda、sdb；或者使用命令`echo 1 > /sys/class/scsi_device/h:c:t:l/device/delete`删除磁盘，h代表HBA卡号，c代表HBA卡channel，t代表SCSI target ID，i代表LUN ID。h:c:t:l这些信息可以通过`lsscsi`、`/lib/udev/scsi_id`，`multipath –l`，`ls –l  /dev/disk/by-*`方式查看。



### LUN

0.0

https://blog.csdn.net/xlh1991/article/details/42873921

LUN的全称是Logical Unit Number，也就是逻辑单元号。

SCSI总线上可挂接的设备数量是有限的，一般为6个或者15个，可以用Target ID(也有称为SCSI ID的)来描述这些设备，设备只要一加入系统，就有一个代号，使用Target ID来标识设备。

但实际上我们需要用来描述的对象，是远远超过该数字的，于是引进了LUN的概念，也就是说LUN ID的作用就是扩充了Target ID。每个Target下都可以有多个LUN Device，我们通常简称LUN Device为LUN，这样就可以说每个设备的描述就有原来的`Target x`变成`Target x LUN y`了，显而易见描述设备数量的能力增强了。

#### what is lun?

LUN的神秘之处(相对于一些新手来说)在于，它很多时候不是什么可见的实体，而是一些虚拟的对象。比如一个阵列柜，主机那边看作是一个Target Device，那为了某些特殊需要，我们要将 磁盘阵列柜的磁盘空间划分成若干个小的单元给主机来用，于是就产生了一些什么逻辑驱动器的说法，也就是比Target Device级别更低的逻辑对象，我们习惯于把这些更小的磁盘资源称之为LUN0、LUN1、LUN2…什么的。而操作系统的机制使然，操作系统识别的最小存储对象级别就是LUN Device，这是一个逻辑对象，所以很多时候被称为Logical Device。

https://blog.51cto.com/lookingdream/1793612

#### LUN and Disk

LUN不等于Physical Disk/块存储

磁盘的属性里可以看到有一个LUN的值，当Disk没有被划分为多个存储资源对象即没有分为多个LUN ID，而将整个磁盘当作 一个LUN来用，LUN ID默认为零，此时磁盘/块设备看来和LUN相等。

#### LUN and PV/LV

　那么什么又是存储卷呢？

这要从存储的卷管理器说起。存储卷管理器是操作系统中的一个对象，他主要负责存储块设备的在线管理。

当我们的一个存储LUN接入计算机后，计算机发现这个设备的存在，就需要在卷管理器上注册，卷管理器为存储卷提供注册的虚拟接口，获取存储LUN的基础信息，如空间大小，三元地址，块大小，起止地址，健康情况等，再为其创建一个对应的数据结构的抽象，这样计算机通过卷管理器，就能够动态的扑捉被注册的存储LUN的实时信息，实现动态管理。

一个存储LUN被卷管理器进行注册抽象之后，就被卷管理器认为是一个可被鱼肉的直接下属，它可以再次被分割成更小区域，当然也可以不分割，再对分割后或者没分割后的存储空间进行数据抽象，建立相关的数据结构，供文件系统层调用。

因此，存储LUN和卷在物理上可能是同一个东西，只是从不同的角度，不同的层次去看它，去理解它。当然，对计算机来说，这些不同确实数据处理过程的需要，也有必要弄清楚的。


https://blog.51cto.com/lookingdream/1793612

#### 看lun id 

lsblk + ll /dev/disk



### iscsi initiator and target：重点

https://www.cnblogs.com/f-ck-need-u/p/9067906.html#11-scsi%E5%92%8Ciscsi

#### initiator

一般而言，对于initiator的配置文件/etc/iscsi/iscsid.conf，里面默认设置了重启服务就自动对已发现过的target进行关联，所以**重启iscsi服务的时候会自动进行关联**。

##### initiator 标识

initiator也需要一个独特的名称来标识自己让target识别。initiator在连接target的时候，会读取`/etc/iscsi/initiatorname.iscsi`中的内容作为自己的iname。

##### iscsiadm

iscsiadm也是一个模式化的命令，使用-m指定mode。mode有:discovery、node、session、iface。一般就用前两个mode。

- `discovery`：发现某服务器是否有target输出，以及输出了哪些target。发现target后会生成target数据库discoverydb。
- `node`：管理跟某target的关联关系。在discovery发现了target后，是否要跟target建立关系，是否要删除已有的关系或者解除已有的关系等。删除关联关系不仅会解除关联，还会删除发现target后生成的discoverydb。
- `session`：会话管理。
- `iface`：接口管理。

```bash
# -m, --mode op 
# specify the mode. op must  be  one  of  discovery,  discoverydb, node, fw, host iface or session.

$ iscsiadm -m discovery -t sendtargets -p 192.168.11.99:3260
192.168.11.99:3260,1 iqn.2018-01.com.h3c.onestor:da7956107efe485ea0c608702c1667b6


#首先discovery目标主机上有哪些可用的target
iscsiadm -m discovery -t st -p 192.168.100.151:3260

# 然后登陆target
iscsiadm -m node -T iqn.2017-03.com.longshuai:test1.disk1 -p 192.168.100.151:3260 -l
```

end

#### target

##### target的命名方式

```css
iqn.YYYY-mm.<reversed domain name>[:identifier]
```

- `iqn`是iscsi qualified name的缩写，就像fqdn一样。
- `YYYY-mm`描述的是此target是何年何月创建的，如2016-06。
- `<reversed domain name>`是域名的反写，起到了唯一性的作用，如longshuai.com写为com.longshuai。
- `identifier`是可选的，是为了知道此target相关信息而设的描述性选项，如指明此target用到了哪些硬盘。

示例：

```makefile
iqn.2018-01.com.h3c.onestor:da7956107efe485ea0c608702c1667b6
iqn.2016-06.com.longshuai:test.disk1
```

end

##### tgtadm

```bash
tgtadm -L iscsi -m target -o new -t 1 -T iqn.2017-03.com.longshuai:test1.disk1
tgtadm -L iscsi -m logicalunit -o new -t 1 -l 1 -b /dev/sdb1
tgtadm -L iscsi -m logicalunit -o new -t 1 -l 2 -b /dev/sdc
tgtadm -L iscsi -m target -o bind -t 1 -I 192.168.100.0/24
```

创建lun？

#### session

建立session即完成initiator和target的关联，建立session后，initiator上就可以查看、访问、操作target上的scsi设备

有两种关联方法，一是关联所有，一是指定单个target进行关联。

```bash
iscsiadm -m node [-d debug_level] [-L all,manual,automatic] [-U all,manual,automatic]
iscsiadm -m node [-d debug_level] [[-T targetname -p ip:port -I ifaceN] [-l | -u ]] [-o operation]

-d：指定debug级别，有0-8个级别, 8日志最多。
-L和-U：分别是登录和登出target，可以指定ALL表示所有发现的target，或者manual指定。
-l和-u：分别是登录和登出某一个target。
-T：用于-l或-u时指定要登录和登出的targetname。
-o：对discoverydb做某些操作，可用的操作有new/delete/update/show，一般只会用到delete和show。
```

例如，使用单个的关联方式关联target。其中target_name可以通过命令`iscsiadm -m node`查看。

```bash
$ ls /var/lib/iscsi/nodes/
iqn.2018-01.com.h3c.onestor:da7956107efe485ea0c608702c1667b6
## 输出了target的IP
$ iscsiadm -m node iqn.2018-01.com.h3c.onestor:da7956107efe485ea0c608702c1667b6
192.168.11.99:3260,1 iqn.2018-01.com.h3c.onestor:da7956107efe485ea0c608702c1667b6

## -u 退出登陆
$ iscsiadm -d 4 -m node  iqn.2018-01.com.h3c.onestor:da7956107efe485ea0c608702c1667b6 -u
...
Logging out of session [sid: 2, target: iqn.2018-01.com.h3c.onestor:da7956107efe485ea0c608702c1667b6, portal: 192.168.11.99,3260]
Logout of [sid: 1, target: iqn.2018-01.com.h3c.onestor:da7956107efe485ea0c608702c1667b6, portal: 192.168.11.99,3260] successful.
iscsiadm: Could not logout of [sid: 5, target: iqn.2018-01.com.h3c.onestor:da7956107efe485ea0c608702c1667b6, portal: 192.168.11.99,3260].
...

## -l 登陆
$ iscsiadm -m node -T iqn.2017-03.com.longshuai:test.disk1 -p 192.168.100.151:3260 -l
Logging in to [iface: default, target: iqn.2017-03.com.longshuai:test.disk1, portal: 192.168.100.151,3260] (multiple)
Login to [iface: default, target: iqn.2017-03.com.longshuai:test.disk1, portal: 192.168.100.151,3260] successful.
```

##### 查看挂载信息:star:

关联后，可以在initiator上看到相关信息

```bash
## 查看iscsi挂载

$ ll /dev/disk/by-path/ | grep sdcj
lrwxrwxrwx. 1 root root 10 Mar 19 11:14 ip-172.200.20.11:3260-iscsi-iqn.2018-01.com.h3c.onestor:4dee75064df6429f8a163074f4995859-lun-15 -> ../../sdcj
...
lrwxrwxrwx. 1 root root  9 Oct  1 16:55 pci-0000:5e:00.0-scsi-0:0:0:0 -> ../../sda
lrwxrwxrwx. 1 root root 10 Oct  1 16:55 pci-0000:5e:00.0-scsi-0:0:0:0-part1 -> ../../sda1
lrwxrwxrwx. 1 root root 10 Oct  1 16:55 pci-0000:5e:00.0-scsi-0:0:0:0-part2 -> ../../sda2

$ /lib/udev/scsi_id  -g  -u /dev/sdcj
$ echo $?
0

$ lsscsi |grep sdcm
[9:0:0:15]   disk    IET      VIRTUAL-DISK     0001  /dev/sdcm

$ lsblk
...

```

end



### iscsi and ceph RBD

https://www.suse.com/c/rbd-vs-iscsi-how-best-to-connect-your-hosts-to-a-ceph-cluster/

ceph 集群目前支持三种形式的存储接口：文件、对象、块，其中块接口（即 RBD）与 SCSI 块设备读写所要求的接口一致，因此可以基于 ceph 的 RBD 提供 SCSI 存储系统后端，当然如果有足够信心的话也可以完全抛弃 ceph 提供的这三种基础接口，而在原始的 RADOS 接口上开发新的块接口，当然除非原始的 RBD 接口有重要缺陷，否则暂时还看不到重新发明轮子的必要，注意后文的讨论都将基于这一基本假设。

**基于 RBD 的 iSCSI 支持有多种实现思路，目前考虑到的有如下三类，概**述如下：

1. 为 iSCSI 客户端增加 RBD 客户端的功能，即在客户端实现 iSCSI initiator 与 RBD client 的融合；
2. 类似于 MDS 的实现，为 ceph 集群增加 LUS(Logical Unit Server) 服务器，由 LUS 负责 iSCSI 业务接入；
3. 选择合适的 iSCSI target 为其增加 RBD 后端支持，并在终端用户与 ceph 集群之间架设 iSCSI 网关（可以扩展成其它类型的如 FC 网关）；

#### 企业级ceph实践

重新编写了新的iSCSI Target，模块名称为STGT。STGT实现了分布式锁机制，从而解决了多个Active target之间支持release/reserve和 Persistent Reservation的问题。同时，将LUN映射信息存储在RADOS集群中，同时在STGT中缓存所有的元数据以加快访问速度。
当LUN映射信息发生变更时，会通知所有STGT的缓存进行变更。
其次，在性能方面，我们通过多线程池提高IO的并发度。

再次，由于是重新开发的软件栈，STGT网络接收到iSCSI数据时，就是用buffer_list保存数据，之后不再需要任何拷贝就通过librados发送给OSD了，从而避免了两个不同开源模块之间的内存拷贝。优化后，单个STGT我们相对于开源TGT的性能，在并发IO下提升超过10倍的性能。

#### ceph iscsi gateway

The iSCSI Gateway presents a Highly Available (HA) iSCSI target that exports RADOS Block Device (RBD) images as SCSI disks. The iSCSI protocol allows clients (initiators) to send SCSI commands to storage devices (targets) over a TCP/IP network, enabling clients without native Ceph client support to access Ceph block storage. These include Microsoft Windows and even BIOS.



### 引用

0. https://www.cnblogs.com/zxqstrong/p/4727912.html
1. https://blog.51cto.com/dcx86team/904499
2. https://www.jianshu.com/p/82eee3b3010c
3. https://blog.csdn.net/karamos/article/details/80127024
4. https://blog.csdn.net/liukuan73/article/details/45506441
5. https://www.zhihu.com/question/52811023/answer/2183837101
6. https://www.zhihu.com/question/20131784/answer/28026813
7. https://www.zhihu.com/question/20131784/answer/90235520
8. https://blog.51cto.com/u_11107124/1884637
9. https://blog.csdn.net/Jacky_Feng/article/details/121579494
10. https://blog.csdn.net/tuning_optmization/article/details/107759698
11. https://bbs.huaweicloud.com/blogs/102289