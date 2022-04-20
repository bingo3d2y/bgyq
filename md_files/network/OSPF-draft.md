## OSPF-draft



 Open Shortest Path First

### 作业



简单p2p拓扑

https://blog.csdn.net/weixin_40228200/article/details/118975991

```bash
 interface vlan 1
 ip address 192.168.1.254 255.255.255.0
 ip address 192.168.2.254 255.255.255.0
 
 interface GE1/0/1
 port link-mode route
 ip address unnumbered interface LoopBack0
 ospf network-type p2p
 ospf timer hello 3
 ospf 1 area 0.0.0.0
 save

[H3C-ospf-1-area-0.0.0.0]display ospf interface


         OSPF Process 1 with Router ID 2.2.2.2
                 Interfaces

 Area: 0.0.0.0
 IP Address      Type      State    Cost  Pri   DR              BDR
 192.168.2.1     PTP       P-2-P    1     1     0.0.0.0         0.0.0.0
 2.2.2.2         PTP       Loopback 0     1     0.0.0.0         0.0.0.0
## 配置静态路由
	
ip route-static dest-address { mask | mask-length } { next-hop-address | interface-type interface-number next-hop-address } [ preference preference-value ] [ description description-text ]
$ ip route-static 192.168.2.0 24 172.10.10.11 preference 80


===
interface GigabitEthernet0/0
 ip address 192.168.1.1 255.255.255.0 
 ospf network-type p2p
#
interface LoopBack0
 ip address 1.1.1.1 255.255.255.255 
#
ospf 1 
 area 0.0.0.0 
  network 192.168.1.0 0.0.0.255
  network 0.0.0.0 255.255.255.255 // 这个不行呦
#
arp static 192.168.2.1 00e0-fc34-51e4
#

### arp
interface GigabitEthernet0/0
 ip address 192.168.2.1 255.255.255.0 
 ospf network-type p2p
#
interface LoopBack0
 ip address 2.2.2.2 255.255.255.255 
#
ospf 1 
 area 0.0.0.0 
  network 192.168.2.0 0.0.0.255
  network 0.0.0.0 255.255.255.255 
#
arp static 192.168.1.1 00e0-fc37-3029
#

##　显示路由表
display ip routing-table

```

end

```bash
### 华三设备上就是不行
[H3C]display ospf routing

         OSPF Process 1 with Router ID 192.168.1.1
                  Routing Table

                Topology base (MTID 0)

 Routing for network
 Destination        Cost     Type    NextHop         AdvRouter       Area
 192.168.1.0/24     1        Stub    0.0.0.0         192.168.1.1     0.0.0.0
 192.168.2.0/24     2        Stub    192.168.2.1     192.168.2.1     0.0.0.0

 Total nets: 2
 Intra area: 2  Inter area: 0  ASE: 0  NSSA: 0


## ar2
[H3C]display  ospf routing

         OSPF Process 1 with Router ID 192.168.2.1
                  Routing Table

                Topology base (MTID 0)

 Routing for network
 Destination        Cost     Type    NextHop         AdvRouter       Area
 192.168.1.0/24     2        Stub    192.168.1.1     192.168.1.1     0.0.0.0
 192.168.2.0/24     1        Stub    0.0.0.0         192.168.2.1     0.0.0.0

 Total nets: 2
 Intra area: 2  Inter area: 0  ASE: 0  NSSA: 0
[H3C]

```



OSPF 路由协议是链路状态型路由协议，这里的链路指的是设备上的接口。链路状态型路由协议基于连接源和目标设备的链路状态作出路由的决定。链路状态是接口及其邻接网络设备的关系的描述，接口的信息即链路的信息，也就是链路的状态（信息）。这些信息包括接口的 IPv6 前缀（ prefix ）、子网掩码、接口连接的网络（链路）类型、与该接口在同一网络（链路）上的路由器等信息。这些链路状态信息由不同类型的 LSA 携带，在网络上传播。

### 网络类型

https://blog.csdn.net/qq_44667101/article/details/105733924?spm=1001.2101.3001.6650.1&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1.pc_relevant_default&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1.pc_relevant_default&utm_relevant_index=2



### 静态路由

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/static-route.png)

 说明：

　　AR1路由器直连A、B两个网段，C、D网段没有直连，你需要添加到C、D网段的路由。

　　AR2路由器直连B、C两个网段，A、D网段没有直连，你需要添加到A、D网段的路由。

　　AR3路由器直连C、D两个网段，A、B网段没有直连，你需要添加到A、B网段的路由。

要想实现全网络通信，必须添加路由条目，这里只是考虑添加静态路由

　　示列： ip route-static 到达的目标网络号 子网掩码 下一跳的路由器接口地址

　　[R1]ip route-static 172.16.1.0 24 172.16.0.2       到C段的路由条目

　　[R1]ip route-static 192.168.1.0 24 172.16.0.2　　　 到D段的路由条目

　　[R2]ip route-static 192.168.1.0 24 172.16.1.2　　   到A段的路由条目

　　[R2]ip route-static 192.168.0.0 24 172.16.0.1　　   到D段的路由条目

　　[R3]ip route-static 192.168.0.0 24 172.16.1.1      到A段的路由条目

　　[R3]ip route-static 172.16.0.0 24 172.16.1.1　　   到B段的路由条目 

如：上图里面的R1路由条目里到C段的路由条目的另一种写法

　　[R1]ip route-static 172.16.1.0 24 GigabitEthernet 1/1    到C段的路由条目  【GE接口写法】

　　[R1]ip route-static 172.16.1.0 24 Serial 1/1    到C段的路由条目  【Serial接口写法】

### 基础名词

#### arp

arp没有跨网段的说法.ARP工作在一个广播域，负责二层寻址而且是在广播网络（对于非广播网络，是没有ARP，比如ATM网络）。它完成了硬件MAC地址和IP之间的翻译。

R3的37.1.1.1 ping R4的38.1.1.1，要封装二层的时候，发现没有38.1.1.1对应的MAC地址，先把目的IP和本出接口IP与运算，发现不同网段，就去找路由表，发现路由表没有路由，于是直接丢弃！

OSPF在P2P网络类型不检查掩码.

#### `?` and history

有点意思

```bash
[H3C]display interface ?
  >                    Redirect it to a file
  >>                   Redirect it to a file in append mode
  FortyGigE            FortyGigE interface
  GigabitEthernet      GigabitEthernet interface
  InLoopBack           InLoopBack interface
  M-GigabitEthernet    MGE interface
  NULL                 NULL interface
  Register-Tunnel      Register Tunnel interface
  Ten-GigabitEthernet  Ten-GigabitEthernet interface
  brief                Brief information of status and configuration for
                       interface(s)
  range                Display range information
  |                    Matching output
  <cr>

## 显示history
[ar1]display history-command
 sys
 interface vlan 1
 ip address 192.168.1.254 255.255.255.0
 save
 exit
 display  interface brief

```

end、

#### MGE0 port

管理用以太网口不受交换芯片工作状态的影响，**一般用于连接计算机以进行系统的程序加载、调试等工作，也可以连接远端的网管工作站等设备以实现系统的远程管理。**

这是个带外网管接口--

#### LSA

LSA（ Link-State Advertisement ，链路状态广播）是链路状态协议使用的一个分组，它包括有关邻居和链路成本的信息。LSAs 被路由器接收用于维护它们的 RIB（路由表）。

路由器把收集到的 LSA 存储在链路状态数据库中，然后运行 SPF 算法计算出路由表。链路状态数据库和路由表的不同在于：数据库中包含的是完整的链路状态原始数据，而路由表中列出的是到达所有已知目标网络的最短路径的列表。

LSA的新旧比较

(1) 会先比较序列号（sequence number），序列号越大越优

(2) 如果序列号相同，会比较校验值（checksum），越大越优

(3) 如果校验值也相同，会比较LSA Age时间，是否等于MAX-age时间（3600S）

(4) 如果age时间不等于MAX-age时间，会比较他们的差值，如果差值>15分钟（900S），越小越优

(5) 如果age时间不等于MAX-age时间，会比较他们的差值，如果差值<15分钟（900S），说明是同一条LSA，忽略其中一条

**LSDB**，Link State DataBase，链路状态数据库。



#### DR/BDR/Drother

Designated Router/Backup Designated Router/Designated Route other

通过在多路访问网段中选择出一个核心路由器，称为DR，网段中所有的OSPF路由器都和DR互换LSA，这样一来，DR就会拥有所有的LSA，并且将所有的LSA转发给每一台路由器；DR就像是该网段的LSA中转站，所有的路由器都与该中转站互换LSA，如果DR失效后，那么就会造成LSA的丢失与不完整，所以在多路访问网络中除了选举出DR之外，还会选举出一台路由器作为DR的备份，称为BDR，BDR在DR不可用时，代替DR的工作，而既不是DR，也不是BDR的路由器称为Drother，事实上，Drother除了和DR互换LSA之外，同时还会和BDR互换LSA。

在广播网和NBMA网络中，任意两台路由器之间都要交换路由信息。如果网络中有n台路由器，则需要建立n（n-1）/2个邻接关系。这使得任何一台路由器的路由变化都会导致多次传递，浪费了带宽资源。为解决这一问题，OSPF提出了DR的概念，所有路由器只将信息发送给DR，由DR将网络链路状态发送出去。

另外，OSPF提出了BDR的概念。BDR是对DR的一个备份，在选举DR的同时也选举BDR，BDR也和本网段内的所有路由器建立邻接关系并交换路由信息。当DR失效后，BDR会立即成为新的DR。

OSPF网络中，既不是DR也不是BDR的路由器为DR Other。DR Other仅与DR和BDR建立邻接关系，DR Other之间不交换任何路由信息。这样就减少了广播网和NBMA网络上各路由器之间邻接关系的数量，同时减少网络流量，节约了带宽资源。

#### OSPF路由器类型

##### IR

区域内路由器，Internal Router，该类路由器的所有接口都属于同一个OSPF区域。

##### ABR

Area Border Router，区域边界路由器。

该类路由器可以同时属于两个以上的区域，但其中一个必须是骨干区域。ABR用来连接骨干区域和非骨干区域，它与骨干区域之间既可以是物理连接，也可以是逻辑上的连接。

##### BR

Backbone Router，骨干路由器。

该类路由器至少有一个接口属于骨干区域。因此，所有的ABR和位于Area0的内部路由器都是骨干路由器。

*Backbone routers* are routing devices that have one or more interfaces connected to the OSPF backbone area (area ID 0.0.0.0)

##### ASBR

ASBR，Autonomous System Boundary Router，自治系统边界路由器。

与其他AS交换路由信息的路由器称为ASBR。ASBR并不一定位于AS的边界，它有可能是区域内路由器，也有可能是ABR。只要一台OSPF路由器引入了外部路由的信息，它就成为ASBR。



#### OSPF 区域

##### 骨干区域

骨干区域（Backbone Area）

OSPF划分区域之后，并非所有的区域都是平等的关系。其中有一个区域是与众不同的，它的区域号是0，通常被称为骨干区域。骨干区域负责区域之间的路由，非骨干区域之间的路由信息必须通过骨干区域来转发。对此，OSPF有两个规定：

* 所有非骨干区域必须与骨干区域保持连通；
* 骨干区域自身也必须保持连通。

在实际应用中，可能会因为各方面条件的限制，无法满足上面的要求。这时可以通过配置OSPF虚连接予以解决。

An OSPF *backbone area* consists of all networks in area ID 0.0.0.0, their attached routing devices, and all ABRs. The backbone itself does not have any ABRs.

```bash
## 配置area 0
ospf 1
 non-stop-routing
 area 0.0.0.0

```

end

##### 虚链接

虚连接（Virtual Link）

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/opsf-virtual-link.png)

虚连接是指在两台ABR之间通过一个非骨干区域而建立的一条逻辑上的连接通道。它的两端必须是ABR，而且必须在两端同时配置方可生效。为虚连接两端提供一条非骨干区域内部路由的区域称为传输区（Transit Area)

虚连接的另外一个应用是提供冗余的备份链路，当骨干区域因链路故障不能保持连通时，通过虚连接仍然可以保证骨干区域在逻辑上的连通

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/opsf-virtual-link-bk.png)

虚连接相当于在两个ABR之间形成了一个点到点的连接，因此，在这个连接上，和物理接口一样可以配置接口的各参数，如发送Hello报文间隔等。

两台ABR之间直接传递OSPF报文信息，它们之间的OSPF路由器只是起到一个转发报文的作用。由于协议报文的目的地址不是中间这些路由器，所以这些报文对于它们而言是透明的，只是当作普通的IP报文来转发。

#### OSPF网络类型

OSPF根据链路层协议类型将网络四种类型:broadcast、nbma、p2mp和p2p

配置命令: `ospf network-type <type_name>`

##### 广播类型

广播（Broadcast）类型：当链路层协议是Ethernet、FDDI时，缺省情况下，OSPF认为网络类型是Broadcast。

在该类型的网络中，通常以组播形式（OSPF路由器的预留IP组播地址是224.0.0.5；OSPF DR的预留IP组播地址是224.0.0.6）发送Hello报文、LSU报文和LSAck报文；以单播形式发送DD报文和LSR报文。

##### NBMA

NBMA（Non-Broadcast Multi-Access，非广播多路访问）类型：当链路层协议是帧中继、ATM或X.25时，缺省情况下，OSPF认为网络类型是NBMA。

在该类型的网络中，以单播形式发送协议报文。

##### P2MP

P2MP（Point-to-MultiPoint，点到多点）类型：没有一种链路层协议会被缺省的认为是P2MP类型。P2MP必须是由其他的网络类型强制更改的，常用做法是将NBMA网络改为P2MP网络。

在该类型的网络中，缺省情况下，以组播形式（224.0.0.5）发送协议报文。可以根据用户需要，以单播形式发送协议报文。

NBMA与P2MP网络之间的区别如下：

* NBMA网络是全连通的；P2MP网络并不需要一定是全连通的。
* NBMA网络中需要选举DR与BDR；P2MP网络中没有DR与BDR。
* NBMA网络采用单播发送报文，需要手工配置邻居；P2MP网络采用组播方式发送报文，通过配置也可以采用单播发送报文

##### P2P：不同网段也互通

P2P（Point-to-Point，点到点）类型：当链路层协议是PPP、HDLC时，缺省情况下，OSPF认为网络类型是P2P。

在该类型的网络中，以组播形式（224.0.0.5）发送协议报文。

下面是什么意思：

OSPF在P2P网络类型不检查掩码、

P2P 网络中掩码之所以可以不一致是因为 P2P 中有 1LSA 的 stub 类型来描述每一个网络的掩码信息，并且在 PPP 链路中 NCP 阶段，两台路由器会互推自己的 IP地址，并且以 32 位主机路由的方式加入自己的路由表，所以 P2P 网络中建立邻居不需要掩码一致。



##### 实际应用

用户可以根据需要更改接口的网络类型，例如：

* 当NBMA网络通过配置地址映射成为全连通网络时（即网络中任意两台路由器之间都存在一条虚电路而直接可达），可以将网络类型更改为广播，不需要手工配置邻居，简化配置。
* 当广播网络中有部分路由器不支持组播时，可以将网络类型更改为NBMA。
* NBMA网络要求必须是全连通的，即网络中任意两台路由器之间都必须有一条虚电路直接可达；如果NBMA网络不是全连通而是部分连通时，可以将网络类型更改为P2MP，达到简化配置、节省网络开销的目的。
* 如果路由器在NBMA网络中只有一个对端，也可将接口类型配置为P2P，节省网络开销。

**如果接口配置为广播、NBMA或者P2MP网络类型，只有双方接口在同一网段才能建立邻居关系。**

#### 路由协议类型

```bash
$ display ip routing-table
# 发现该路由的路由协议类型：

·     O_INTRA：OSPF intra area

·     O_INTER：OSPF inter area

·     O_ASE1：OSPF external type 1

·     O_ASE2：OSPF external type 2

·     O_NSSA1：OSPF NSSA external type 1

·     O_NSSA2：OSPF NSSA external type 2

·     O_SUM：OSPF summary

·     IS_L1：IS-IS level-1

·     IS_L2：IS-IS level-2

·     IS_SUM：IS-IS summary
```

end

### 基础命令

#### 端口缩写

MGE为设备管理接口，XGE为万兆接口，FGE千兆接口



#### 交换机二层和三层

```bash
## 该交换机默认是三层网络
[H3C-GigabitEthernet0/0]port link-mode ?
  bridge  Switch to layer2 ethernet
  route   Switch to layer3 ethernet

## 切换成二层网络模式
## 命令都变了
[H3C-GigabitEthernet0/0]port link-mode bridge
[H3C-GigabitEthernet0/0]port ?
  access     Set access port attributes
  hybrid     Set hybrid port attributes
  link-mode  Switch the specified interface to layer2 or layer3 ethernet
  link-type  Set the link type
  trunk      Set trunk port attributes

[H3C-GigabitEthernet0/0]port access ?
  vlan  Assign the port to a VLAN
```

end

#### 查看当前配置

```bash
$ interface Vlanif 2
$ ip address 10.10.100.1 255.255.255.0
$ display current-configuration
```

end

#### 静态路由

命令解析学习，一步步的`?`真的好用呢

```bash
[Huawei]ip route-static ?
  X.X.X.X             Destination IP address
  default-preference  Preference-value for IPv4 static-routes
  selection-rule      Selection rule
  vpn-instance        VPN-Instance route information

[Huawei]ip route-static 0.0.0.0?
  X.X.X.X                                 
[Huawei]ip route-static 0.0.0.0 ?
  INTEGER<0-32>  Length of IP address mask
  X.X.X.X        IP address mask

[Huawei]ip route-static 0.0.0.0 0.0.0.0 ?
  GigabitEthernet  GigabitEthernet interface
  MEth             MEth interface
  NULL             NULL interface
  Vlanif           Vlan interface
  X.X.X.X          Gateway address
  vpn-instance     Destination VPN-Instance for Gateway address

[Huawei]ip route-static 0.0.0.0 0.0.0.0 10.10.10.1
```

实践例子--

```bash
## 添加静态路由
[Huawei]ip route-static 0.0.0.0 0.0.0.0 192.168.1.1
[Huawei]display ip routing-table 
Route Flags: R - relay, D - download to fib
------------------------------------------------------------------------------
Routing Tables: Public
         Destinations : 5        Routes : 5        

Destination/Mask    Proto   Pre  Cost      Flags NextHop         Interface

        0.0.0.0/0   Static  60   0          RD   192.168.1.1     Vlanif100
      127.0.0.0/8   Direct  0    0           D   127.0.0.1       InLoopBack0
      127.0.0.1/32  Direct  0    0           D   127.0.0.1       InLoopBack0
    192.168.1.0/24  Direct  0    0           D   192.168.1.2     Vlanif100
    192.168.1.2/32  Direct  0    0           D   127.0.0.1       Vlanif100
## 删除静态路由
[Huawei]undo ip route-static 0.0.0.0 0.0.0.0 192.168.1.1
[Huawei]display ip routing-table
Route Flags: R - relay, D - download to fib
Routing Tables: Public
         Destinations : 4        Routes : 4        

Destination/Mask    Proto   Pre  Cost      Flags NextHop         Interface

      127.0.0.0/8   Direct  0    0           D   127.0.0.1       InLoopBack0
      127.0.0.1/32  Direct  0    0           D   127.0.0.1       InLoopBack0
    192.168.1.0/24  Direct  0    0           D   192.168.1.2     Vlanif100
    192.168.1.2/32  Direct  0    0           D   127.0.0.1       Vlanif100

```

end

#### 删除IP地址

How do I remove a VLANIF IP from my Huawei?

```bash
# 查看 ip 地址
$ display ip interface brief

# physical port
$ undo -bind static ip-address 192.168.1.2 interface GigabitEthernet 0/0/1
Info: The operation may take a few seconds. Please wait for a moment.done.
0 item(s) deleted.

# svi
[H3C-Vlan-interface20]undo ipaddress //删除IP地址

[Huawei-Vlanif100]undo ip address 192.168.1.2 255.255.255.0
[Huawei-Vlanif100]
Apr  8 2022 15:47:28-08:00 Huawei %%01IFNET/4/LINK_STATE(l)[4]:The line protocol
 IP on the interface Vlanif100 has entered the DOWN state.
[Huawei-Vlanif100]display current-configuration
```

end

#### Serial接口

Ser口是serial接口的意思，也叫高速异步串口，主要是连接广域网的V.35线缆用的，即路由器和路由器连接时候用的，可以用命令设置带宽，一般也就在10M、8M左右，不过现在的路由器都没有这些口了，都用光口代替了。

#### LLDP

LLDP是一种邻近发现协议。它为以太网网络设备，如交换机、路由器和无线局域网接入点定义了一种标准的方法，使其可以向网络中其他节点公告自身的存在，并保存各个邻近设备的发现信息。例如设备配置和设备识别等详细信息都可以用该协议进行公告。

具体来说，LLDP定义了一个通用公告信息集、一个传输公告的协议和一种用来存储所收到的公告信息的方法。要公告自身信息的设备可以将多条公告信息放在一个局域网数据包内传输，传输的形式为类型长度值（TLV）域。

```bash
 lldp management-address arp-learning
 lldp tlv-enable basic-tlv management-address-tlv interface LoopBack0

## ?是查看啊
[H3C]lldp ?
  compliance     Enable compliance with another link layer discovery  protocol
  fast-count     The fast-start times of transmitting frames
  global         Specify global
  hold-multiplier   Hold multiplicator for TTL
  ignore-pvid-inconsistency  Ignore PVID inconsistency
  max-credit        Specify LLDP maximum transmit credit
  mode              Specify LLDP bridge mode
  timer             Timer of LLDP

```

end

#### 创建ospf

```bash
# 启动OSPF进程，进入OSPF视图
ospf [ process-id | router-id router-id | vpn-instance vpn-instance-name ... ]

# 创建opsf进程1，并设置router-id为1.1.1.1
$ ospf 1 router-id 1.1.1.1
```

* **process-id** 为进程号，缺省值为1。

  路由器支持OSPF多进程，可以根据业务类型划分不同的进程。进程号是本地概念，不影响与其它路由器之间的报文交换。因此，不同的路由器之间，即使进程号不同也可以进行报文交换.。

* **router-id** ： router-id为路由器的ID号。

  缺省情况下，路由器系统会从当前接口的IP地址中自动选取一个最大值作为Router ID。手动配置Router ID时，必须保证自治系统中任意两台Router ID都不相同。通常的做法是将Router ID配置为与该设备某个接口的IP地址一致。

* **vpn-instance** : vpn-instance-name 表示VPN实例。

#### 有向图？？？

LSDB有向图

Link State DataBase，链路状态数据库，OSPF干脆就给区域内每台路由器都搞一张地图，这样就不会上当受骗了，这个地图就是LSDB，这样就使得OSPF可以保证区域内无环，区域间无环，通过一些规则来限制，这样区域内外都能保证无环。

```bash
[ar1]display  ospf lsdb router

         OSPF Process 1 with Router ID 1.1.1.1
                 Link State Database

                         Area: 0.0.0.0

    Type      : Router
    LS ID     : 192.168.2.1
    Adv Rtr   : 192.168.2.1
    LS age    : 208
    Len       : 60
    Options   : O E
    Seq#      : 8000000c
    Checksum  : 0xed3d
    Link Count: 3
       Link ID: 1.1.1.1
       Data   : 192.168.2.1
       Link Type: P-2-P
       Metric : 1
       Link ID: 2.2.2.2
       Data   : 255.255.255.255
       Link Type: StubNet
       Metric : 0
       Link ID: 192.168.2.0
       Data   : 255.255.255.0
       Link Type: StubNet
       Metric : 1

    Type      : Router
    LS ID     : 1.1.1.1
    Adv Rtr   : 1.1.1.1
    LS age    : 207
    Len       : 48
    Options   : O E
    Seq#      : 8000000a
    Checksum  : 0xac03
    Link Count: 2
       Link ID: 192.168.2.1
       Data   : 192.168.1.1
       Link Type: P-2-P
       Metric : 1
       Link ID: 192.168.1.0
       Data   : 255.255.255.0
       Link Type: 
```

end

### 基础组网配置

网络中有三台交换机。现在需要实现三台交换机之间能够互通，且以后能依据SwitchA和SwitchB为主要的业务设备来继续扩展整个网络。

![ospf-base](D:\temp_files\download\ospf-base.png)

采用如下的思路配置OSPF基本功能：

1. 在各交换机的VLANIF接口上配置IP地址并配置各接口所属VLAN，实现网段内的互通。
2. 在各交换机上配置OSPF基本功能，并且以SwitchA为ABR将OSPF网络划分为Area0和Area1两个区域，实现后续以SwitchA和SwitchB所在区域为骨干区域来扩展整个OSPF网络

https://support.huawei.com/enterprise/zh/doc/EDOC1000141427/d55d1d8c

```bash
# //创建进程号为1，Router ID为10.1.1.1的OSPF进程
[SwitchA] ospf 1 router-id 10.1.1.1   
# //创建area 0区域并进入area 0视图
[SwitchA-ospf-1] area 0   
# //配置area 0所包含的网段
[SwitchA-ospf-1-area-0.0.0.0] network 192.168.0.0 0.0.0.255   
[SwitchA-ospf-1-area-0.0.0.0] quit

## switch B
[SwitchB] ospf 1 router-id 10.2.2.2
[SwitchB-ospf-1] area 0
[SwitchB-ospf-1-area-0.0.0.0] network 192.168.0.0 0.0.0.255
[SwitchB-ospf-1-area-0.0.0.0] return

## switch C
[SwitchC] ospf 1 router-id 10.3.3.3
[SwitchC-ospf-1] area 1
[SwitchC-ospf-1-area-0.0.0.1] network 192.168.1.0 0.0.0.255
[SwitchC-ospf-1-area-0.0.0.1] return
```

配置总览：

Switch A

```bash
#
sysname SwitchA
#
vlan batch 10 20
#
interface Vlanif10
 ip address 192.168.0.1 255.255.255.0
#
interface Vlanif20
 ip address 192.168.1.1 255.255.255.0
#
interface GigabitEthernet0/0/1
 port link-type trunk
 port trunk allow-pass vlan 10
#
interface GigabitEthernet0/0/2
 port link-type trunk
 port trunk allow-pass vlan 20
#
ospf 1 router-id 10.1.1.1
 area 0.0.0.0
  network 192.168.0.0 0.0.0.255
 area 0.0.0.1
  network 192.168.1.0 0.0.0.255
#
return
```

Switch B

```bash
#
sysname SwitchB
#
vlan batch 10
#
interface Vlanif10
 ip address 192.168.0.2 255.255.255.0
#
interface GigabitEthernet0/0/1
 port link-type trunk
 port trunk allow-pass vlan 10
#
ospf 1 router-id 10.2.2.2
 area 0.0.0.0
  network 192.168.0.0 0.0.0.255
#
return
```

Switch C

```bash
#
sysname SwitchC
#
vlan batch 20
#
interface Vlanif20
 ip address 192.168.1.2 255.255.255.0
#
interface GigabitEthernet0/0/1
 port link-type trunk
 port trunk allow-pass vlan 20
#
ospf 1 router-id 10.3.3.3
 area 0.0.0.1
  network 192.168.1.0 0.0.0.255
#
return
```

end

### 交换机和路由器互联：basic

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/switch-router.png)

#### 交换机配置svi

```bash
[Switch] vlan batch 100
[Switch] interface gigabitethernet 0/0/1
[Switch-GigabitEthernet0/0/1] port link-type access
[Switch-GigabitEthernet0/0/1] port default vlan 100   
[Switch-GigabitEthernet0/0/1] quit
[Switch] interface vlanif 100
[Switch-Vlanif100] ip address 192.168.100.2 24
[Switch-Vlanif100] quit
```

end

#### 路由器接口

```bash
<Huawei> system-view
[Huawei] sysname Router
[Router] interface gigabitethernet 0/0/1
## 配置的IP地址192.168.100.1为交换机缺省路由的下一跳IP地址
[Router-GigabitEthernet0/0/1] ip address 192.168.100.1 255.255.255.0   
[Router-GigabitEthernet0/0/1] quit
```

end

#### 缺省和回程路由

```bash
## 交换机缺省路由
##  缺省交换机路由的下一跳是路由器接口的IP地址192.168.100.1
##　vlanIF IP is 192.168.100.2
[Switch] ip route-static 0.0.0.0 0.0.0.0 192.168.100.1 
## 路由器缺省和回程路由
## 配置路由器回程路由的下一跳就指向交换机上行接口的IP地址192.168.100.2
[Router] ip route-static 192.168.0.0 255.255.0.0 192.168.100.2  
## 配置路由器静态缺省路由的下一跳指向公网提供的IP地址200.0.0.1，路由器一端连着公网负责提供对外访问
[Router] ip route-static 0.0.0.0 0.0.0.0 200.0.0.1   

```

end







### OSPF排查思路

https://support.huawei.com/enterprise/en/knowledge/EKB1000116686



### 引用

1. https://ccie.lol/knowledge-base/ospf-ospfv3-lsa/
2. http://www.h3c.com/cn/d_201312/810777_30005_0.htm#_Toc373336023
3. https://blog.csdn.net/qq_38265137/article/details/80390740
4. https://blog.51cto.com/u_3258715/1735713
4. https://blog.51cto.com/u_12347226/2435104