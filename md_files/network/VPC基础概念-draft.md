## VPC基础概念-draft

VPC: virtual private cloud

### VPC疑问

已知，我的vpc通过VxLAN技术来实现。

用户创建了N个VPC subnetwork，比如`10.200.18.0/24`, 这些网段可以访问外网和存储网，这些VPC网段要预先在核心交换机上声明配置路由吗？

不对，vpc网段都是私有吧，应该考虑vpc网段怎么和undelay对接、

evpn类似bgp，实现vxlan路由的自动建立、那么没有启用evpn时，用户可以创建的vpc网段需要预先声明吗。

答案：

1. 没有evpn或sdn结合的vxlan需要自己配置控制平面信息，包括不限于：tunnel、vxlan vni、vlan-vxlan-ref等
2. vxlan私有网段的映射信息，租户创建和声明了哪个网段就在设备上配置什么路由。

如下，当然这些都是SDN自动配置的，核心就是

* 创建vsi 和 vxlan
* 关联vxlan 和 tunnel
* 配置vpc网段静态路由，将vxlan流量路由到设备上一个vlan vsi上，在通过设备路由到外部或者其他vpc网段

```bash
## vsi information
[S6860-54HF_zhu_leaf]display l2vpn vsi name SDN_VSI_2041
Total number of VSIs: 1, 1 up, 0 down, 0 admin down
VSI Name                        VSI Index       MTU    State     
SDN_VSI_2041                    7               1500   Up       
## vsi 详细信息
[S6860-54HF_zhu_leaf]display interface Vsi-interface 2041
Vsi-interface2041
Current state: UP
Line protocol state: UP
Description: SDN_VSI_Interface_2041
Bandwidth: 1000000 kbps
Maximum transmission unit: 1444
IP packet frame type: Ethernet II, hardware address: 6805-ca21-d6e5
IPv6 packet frame type: Ethernet II, hardware address: 6805-ca21-d6e5
Physical: Unknown, baudrate: 1000000 kbps
Last clearing of counters: Never
Input (total):  636256851 packets, 259594183336 bytes
Output (total):  0 packets, 0 bytes
## vsi ip info
[S6860-54HF_zhu_leaf]display interface Vsi-interface brief
Interface Link Protocol Primary IP Description
Vsi2024   UP    UP      10.100.90.1 SDN_VSI_Interface_2024
...
Vsi2041   UP    UP      10.100.180.1 SDN_VSI_Interface_2041

## vxlan vni and tunnel
## tunnel和vxlan的关联关系
[S6860-54HF_zhu_leaf]display vxlan tunnel vxlan-id 2041
VXLAN ID: 2041, VSI name: SDN_VSI_2041

## 路由信息
[S6860-54HF_zhu_leaf]display ip routing-table 

Destinations : 76	Routes : 76

Destination/Mask   Proto   Pre Cost        NextHop         Interface
...
1.1.1.1/32         O_INTRA 10  1           172.100.1.1     Vlan300
...
10.1.34.20/30      Direct  0   0           10.1.34.21      Vlan352
10.1.34.20/32      Direct  0   0           10.1.34.21      Vlan352
10.1.34.23/32      Direct  0   0           10.1.34.21      Vlan352
...
## 这个10.1.34.22 是谁的IP，啧啧 interface是vlan352
10.100.180.0/24    Static  60  0           10.1.34.22      Vlan352
...




## vlan interface

[S6860-54HF_zhu_leaf]display interface Vlan-interface brief
Brief information on interfaces in route mode:
Link: ADM - administratively down; Stby - standby
Protocol: (s) - spoofing
Interface            Link Protocol Primary IP      Description                
Vlan1                DOWN DOWN     --              
Vlan3                UP   UP       3.3.3.1         for-srm
Vlan80               UP   UP       80.80.80.1      srm
Vlan110              UP   UP       172.100.20.1    cunchuwai
Vlan120              UP   UP       172.100.30.1    cunchunei
Vlan130              UP   UP       172.100.40.1    daiwaiguanli
Vlan140              UP   UP       172.100.50.1    chaorongheguanli
Vlan200              UP   UP       172.100.60.1    yuanchengguanli
Vlan300              UP   UP       172.100.1.2     HuLian-S6860-S6800
Vlan342              UP   UP       10.2.34.5       root-vpc-srm-test-out
Vlan343              UP   UP       10.1.34.9       srm-test1-out
Vlan350              UP   UP       10.1.34.13      chenggui-OCC2
Vlan351              UP   UP       10.1.34.17      NOSC
Vlan352              UP   UP       10.1.34.21      AF-XWZX

```

end

### VPC作业

1、 安装ensp

ensp+virtual box

界面导入image就是在virtual box上创建了一个虚拟机。

启动一个 ce6800实例失败了，发现在virtual box上启动ce镜像的虚拟机也会失败。

Ensp安装过程在在virtualbox配置虚拟机 

还需要设置win7 兼容运行ensp+virtual

2、vxlan+sdn组网

### UDP Tunnel:star:

基于几点特质,UDP tunnel 越来越流行了，

1）无障碍穿越任何 NAT 设备无论高端的商用路由器、还是家用路由器，都 支持，所以无需额外的配置就可以工作，省时省心。

2 ）UDP 是无状态的。由于 建立 TCP tunnel 之前需要建立连接，然后再建立 tunnel，发送数据的初始延迟就 会稍大。外层的 TCP 是有状态的，内层负载如果也是有状态的，双状态机不利 于排错、debug。同时，使用 UDP 端口提供了更灵活的负载均衡的潜能。

所以 L2TP、VxLAN、IKEv2 IP Security 都是采用 UDP 封装。

为什么用udp而不是tcp，不是因为udp“更好”，而是因为tcp根本就不能用！



tcp是可靠的，over tcp后还是可靠的； 如果是 over udp也是可靠的。所以本质上TCP Tunnel 没带来任何好处



一旦发生丢包，因为上层tcp不知道下层tcp丢包了（下层tcp丢包了--真正丢包，会通过重传搞定），上层tcp能感知到的只是他的rtt暴增，进而影响上层tcp剧烈降速（这是错误的降速），tcp流控依赖rtt，用了错误的rtt那么就导致了错误的流控，明明带宽很多、网络也稳定，非要用远超出实际的rtt来再次慢启动



综上，tcp over tcp 没有任何优点，反而会导致网络不稳定



作者：plantegg
链接：https://www.zhihu.com/question/39382183/answer/463690235
来源：知乎
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

### VPC中共享服务

在数据中心，一些服务需要为多个租户提供服务，例如DNS和DHCP，这些服务通常部署在共享的VRF中，所有的租户的VRF都可以访问这个VRF，通常由两种实现方式，路由泄露和下游VNI分配。

#### 路由泄漏

路由泄露时通过RT互导方式实现,路由泄露发生在本地，如图：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/路由泄漏.png)

配置举例，VRF A RT import 为65501:50002，是VRF B 的export 。

```bash
# VRF Configuration at Ingress VTEP (V1)
# VRF： Virtual Routing and Forwarding
vrf context VRF-A
vni 50001
rd auto
address-family ipv4 unicast
route-target both auto
route-target both auto evpn
route-target import 65501:50002
route-target import 65501:50002 evpn
## 
vrf context VRF-B
vni 50002
rd auto
address-family ipv4 unicast
route-target both auto
route-target both auto evpn
route-target import 65501:50001
route-target import 65501:50001 evpn


```

数据平面在跨VRF转发时，携带VNI标签时起源VRF的VNI标签，跨VRF转发发生在出方向路由器上。如图：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/路由泄漏-流量.png)

#### 下游VNI分配

下游VNI 分配解决方案中，出方向设备向入方向设备规定了某个eVNP路由使用的VNI标签，这类似于MPLS L3VNP的解决方案，MPLS L3VNP中使用动态分配，而下游VNI分配使用静态的方式。
如图：VTEP2 向VTEP1 通告，到达VTEP 192.168.2.102的路由使用VNI 5002,数据包将直接从VTEP1的VRF A转发到VTEP2的VRF B。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/下游VNI分配.png)


版权声明：本文为CSDN博主「灵气小王子」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/qq_43691045/article/details/103598254

### 什么是VPC

VPC实际上就是SDN在公有云的应用。软件可控，Overlay使得服务商的硬件利用率提高，对硬件厂商的依赖程度降低。在这个基础上，公有云服务商还能够提供更好的网络服务。

VPC全称是Virtual Private Cloud，VPC是一种云，也是一种网络模式，不过应该从服务和技术的角度分别来看。

VPC最早是由AWS在2009年提出，不过VPC的一些组成元素在其提出之前就已经存在。VPC只是将这些元素以私有云的视角重新包装了一下。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vpc-base.jpg)

#### 服务角度

首先从服务的角度来看，VPC指的是一种云（Cloud），这与它的字面意思相符。对于基础架构服务（IaaS），云就是指资源池。你或许听过公有云（Public Cloud）、私有云（Private Cloud）、混合云（Hybrid Cloud）。不过，VPC不属于这三种云中任一种。这是一种运行在公有云上，将一部分公有云资源为某个用户隔离出来，给这个用户私有使用的资源的集合。VPC是这么一种云，它由公有云管理，运行在公共资源上，但是保证每个用户之间的资源是隔离，用户在使用的时候不受其他用户的影响，感觉像是在使用自己的私有云一样。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vpc-unit.png)

Security Group, Subnet, Network ACL, Routing Table, Router，这些都是老生常谈了。以VPC为单位来划分这些资源，可以更好的突出私有的感觉。

从这种意义上看，VPC不是网络，从公有云所提供的服务来说，VPC应该理解成，向用户提供的隔离资源的集合。

用户可以在公有云上创建一个或者多个VPC，每个部门一个VPC。对于需要连通的部门创建VPC连接。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vpc-as-service.png)

同时，用户也可以通过VPN将自己内部的数据中心与公有云上的VPC连接，构成混合云

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vpc-connect-underlay.png)

不论哪种用例，VPC都以更加直观形象让用户来设计如何在公有云上存放自己的数据。

##### VPC硬件租用模式

VPC硬件租用模式（Hardware Tenancy）本身也是公有云提供的一种服务模式。VPC的硬件租用模式有两种，一种是共享（shared），一种是专属（dedicated）。共享是指VPC中的虚拟机运行在共享的硬件资源上，不同VPC中的虚拟机通过VPC进行隔离。专属是指VPC中的虚拟机运行在专属的硬件资源上，不同VPC中的虚拟机在物理上就是隔离的，同时VPC帮助实现网络上的隔离。专属模式相当于用户直接向公有云服务商租用物理主机。专属模式适合那些对于数据安全比较敏感的用户，不过这些物理主机还是由公有云服务商管理。
不论是共享模式还是专属模式，VPC都运行在公有云资源上，由公有云服务商管理。

#### 技术角度

从技术角度来看，VPC是用户专属的一个二层网络，VPC指的只是网络的话，那它跟VPN的概念是重复的。

> 所以，我会看到 EVPN跨VPC三层互通的PPT

VPC能够为每个用户一个专属独立的二层网络。因为VPC是一个用户专属的网络，用户可以任意定义VPC内云主机的IP地址。二层隔离了，IP地址想怎么玩就怎么玩，不会出现N个用户使用同一个IP导致一个IP对应多个个MAC影响网络通讯的情况。

> 当地址发生冲突时，根据免费ARP引起的弹框和日志告警，用户或者管理员便可以对IP地址进行修改，从而解决通信问题。

从AWS公布的资料看，VPC的数据封装与VXLAN这类网络Overlay技术也很像。从下图可以看出，桔色的VPC中，10.0.0.2发往10.0.0.3的网络数据，最终被封装成主机之间的通信报文。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/aws-L2-vpc.png)

原始的二层帧，被VPC标签封装，之后封装在另一个IP报文里面。这与VXLAN的封装方式可以说是一模一样。不过需要澄清的是，AWS在2010年就已经开始应用VPC，而VXLAN标准是2014年[3]才终稿。AWS的VPC或许和VXLAN不一样，但是按照VXLAN理解VPC的overlay会更容易些。

> 只要VPC的封装标签ID不同，这样就是VPC内定义的IP相同，封装后也不影响传播。

##### VPC的实现：参考

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vpc模型映射.png)

VPC租户模型

n租户模型基于OpenStack逻辑概念

①vRouter对应OpenStack内的vRouter，代表逻辑三层网关/网络

②vNet对应OpenStack内的Network，代表逻辑二层网络

③Subnet对应OpenStack内的Subnet，代表某个子网网段

④每个vNet对应一个VLAN（或VxLAN），vNet内多个Subnet代表该VLAN（或VxLAN）覆盖多个子网网段

* L2层面（接入），使用 VLAN ID（或VxLAN ID）来提供租户隔离
* 对于标准租户（Standard Tenant），使用vRouter实现在路由层面的租户隔离，每个租户使用单独的vRout
* 对于VPC类租户，使用 VRF实现在路由层面的租户隔离 ，每个VPC类租户使用单独的VRF
* 除非明确配置，数据中心内不允许跨租户的流量访问（即租户间互访流量建议通过广域互通）
* 网络服务隔离，为每个租户提供单独的FW、LB及NAT服务



#### VPC的好处

因为用户层面使用静态的vxlan租户网络，承载vxlan overlay的underlay 网络，可以处于不同机房、不同数据中心甚至不同的城市，只要VTEP建立的vxlan tunnel延时满足要求即可。

这就是Openstack提出的Hierarchical Port Binding吧--

VPC使用网络Overlay之后，可以构建一个L3之上的L2。这样一个VPC内的虚机，可以任意的在数据中心分布。实际中云主机肯定不是任意分布的，会有一些主机的调度优化算法，但是至少，网络不会成为限制云主机部署的因素。举个反例，**如果使用VLAN，虚机必须部署在支持相应VLAN的设备上**，（交换机的网卡划分了不同VLAN并连接了不同的服务器）哪怕这个设备已经接近饱和，而其他的设备却是空置的。如下图，因为左边的机架不支持相应的网络，对应的云主机只能往右边的机架塞，直到塞满。而同时，左边的机架负载还不到50%。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/VPC-overlay-pro.png)

Overlay使得VPC不再受网络硬件的限制，VPC内的云主机可以部署在整个机房。



#### Classic and VPC:star2:

Classic和VPC。它们之间最核心的区别是：经典网络提供的是多用户共享的网络，而VPC提供的是用户专属的网络。

因为VPC是一个用户专属的网络，用户可以任意定义VPC内云主机的IP地址。**二层隔离了，IP地址想怎么玩就怎么玩（二层隔离一个IP对应的MAC就唯一了）**。而在经典网络模式下，大家挤在一个二层网络里面，IP地址首先要保证不要重合，这对用户和服务商来说都不是一件心情愉快的事情。

网络就是指二层网络，经典网络模型本身有很多问题，其中最大的问题就是安全问题。除非加了特定的防火墙规则去拦截，二层网络内的所有设备默认是可以通信的。这就好比大家都挤在一个房间里，彼此的隐私很难保障一样。稍有不慎，云主机就可能被同网络的其他用户恶意攻击。而VPC能够为每个用户一个专属独立的二层网络。这样相当于给每个用户分了个房间，用户的隐私更容易得到保障。就算有恶意攻击，一般也要走到网关或者VPN设备，在这些集中的设备上，网络流量更可控。
由于每个用户都有专属的二层网络，那说明VPC模式下的可用二层网络的数量是远超经典模式的。虽然各家都没有公布自己的实现细节，但是这里有点类似VXLAN和VLAN的关系。VXLAN可以有1600万个二层网络，VLAN只有4000多个二层网络。

##### 二层隔离

？？？为什么二层隔离就可以随便分配IP。

在TCP/IP网络模型中，二层协议有ARP和RARP。二层不隔离，如果IP冲突会导致网络质量下降。

AB两台电脑虽然IP地址相同，但是MAC地址不同。A电脑与网关通信时，交换机能够正常检查MAC目的地址，把数据送到23口。但是AB两台电脑IP相同，会不停广播自己的IP与MAC对应关系，所以网关的ARP列表也在不停刷新(刷新间隔极短，频率极高)，导致**网关回包时IP不变，但依据ARP信息封装不同的目的MAC**，交换机收到的数据帧目的MAC一会A电脑一会B电脑，导致AB皆网络卡顿。

数据链路层的网络包，也叫“帧”，我们常说的网卡的MAC地址，就是帧的地址，MAC，其实是“媒体访问控制”（**media access control**）的简称，这是数据链路层的一个子层；

那为什么要在这个二层上搞隔离呢？

因为二层的帧，其中一些帧的地址是广播地址，在同一个二层的设备都可以、也必须接收这些帧，交换机一般认为工作在二层，对这些广播包，也都要转发，所以二层通常被称为一个“广播域”。

#### VPC边缘设备

https://zhuanlan.zhihu.com/p/137963921



现实中这些都通过虚拟化手段实现了吧，SDN+NFV

VPC从服务的角度来看是虚拟私有云，表示的公有云运营商提供给用户的隔离资源的集合。它相当于是漂浮在公有云上的孤岛。真正让VPC变得强大的是它各式各样的连接技术。AWS提供了一个Edge设备（Blackfoot Edge Device），VPC通过这个Edge设备可以：

1. 与别的VPC相连
2. 与互联网相连
3. 与用户的私有云建立VPN连接。
4. 与AWS的其他服务建立连接。

这才是公有云服务商在构建VPC网络时，真正的竞争力所在。有了这样的Edge设备，VPC不再是孤岛，而是有了连接其他陆地的桥梁。这里的Edge设备，可以看成是VNF，AWS需要用户在VPC内部手动配置路由来引流到这个Edge设备

AWS所抽象出来的三种网关类型，来实现VPC直接的互通。

##### IGW

IGW，Internet Gateway，访问Internet的必经之地。当你为EC2分配公网IP地址的时候，这个地址总是存在于IGW上的，IGW做了EC2私网地址到公网地址的1对1转换。

##### VGW

VGW，当VPC需要通过VPN/Direct connect连接到本地数据中心/站点时，就需要VGW。VGW存在的合理性很容易分析，如果是我设计机房，接入区肯定被单独安排在某个或者某几个物理位置**，所以从用户所在的位置到达接入区一定是存在物理链路/物理网络的，那么对应的，在虚拟网络里就需要一个VGW这样的角色，用于完成底层的流量转发。**

##### TGW

TGW，当你有多个VPC，且多个VPC需要互访，切多个VPC还需要和本地互访的时候，TGW就派上用场了。Transit Gateway，顾名思义，中转用的网络节点，在这种情况下，VPN可以直接安排在TGW上。

##### NGW

IGW applied in VPC level  whereas NGW is applied at instance level

A NAT Gateway does something similar, but with two main differences:

1. It allows resources in a private subnet to access the internet (think yum updates, external database connections, wget calls, etc), and
2. it only works one way. The internet at large cannot get through your NAT to your private resources unless you explicitly allow it.

### VPC 多租户

作为 MP-BGP 的扩展，MP-BGP EVPN 继承了 VPN 通过 VRF 实现的对多租户的支持。

**同一 租户的 VNI 的不同子网，都属于同一个 L3 VRF 。**

在 MP-BGP EVPN 中，多个租户可以共存，它们共享同一个 IP 传输网络（underlay），而 在 VXLAN overlay 网络中拥有独立的 VPN。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/19-mp-bgp-evpn-multi-tenancy.png)

在 VXLAN MP-BGP EVPN spine-and-leaf 网络中，**VNI 定义了二层域**，不允许 L2 流量跨越 VNI 边界。类似地，**VXLAN 租户的三层 segment 是通过 VRF 技术隔离的**，通过将不同 VNI 映射到不同的 VRF 实例来隔离租户的三层网络。每个租户都有自己的 VRF 路由实例。**同一 租户的 VNI 的不同子网，都属于同一个 L3 VRF 。**

#### VRF

VRF：Virtual Routing Forwarding，虚拟路由转发。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/what-is-vrf.png)

举个栗子

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vrf-case.gif)

假设PC1与R2这一侧的网络属于一个独立的业务；

PC2与R3这一侧的网络属于另一个独立的业务；

由于设备资源有限或者其他方面的原因，这两个独立的业务的相关节点连接在R1上，也就是同一台设备上。

那么在完成相关配置后，R1的路由表如上图所示。

由于，共用一个Router R1导致，PC1可以直接访问R3连接的`3.3.3.0/24`网络。

但是实际上，从业务的角度考虑，我们禁止PC1访问3.3.3.0/24网络。那么怎么办？

两个思路：

* 在增加一个单独的路由器，实现物理隔离（增加成本）
* 通过ACL规则，实现逻辑隔离 （增加管理成本或可能影响全局的或者不同业务系统有不同的组网需求通过ACL无法满足）

这时，VRF就很有用了，类似虚拟化技术在路由器设备上的应用。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/VRF-solution-1.gif)

在R1上创建两个VRF：VRF1及VRF2，创建完成后，我们可以理解为，拥有了两台虚拟路由器。

当然，刚刚创建两个VRF，现在这两台虚拟路由器上什么也没有。

```bash
$ system-view
$ ip vrf vrf-name
```

接下去我们将GE0/0/1口及GE0/0/2口绑定到VRF1；将GE0/0/3及GE0/0/4口绑定到VRF2。

```bash
$ interface name
$ ip vrf forwarding vrf-name
```

这样这两台虚拟路由器就各自拥有了两个物理接口。这两台虚拟路由器是虽然都在同一台物理设备上，但是却是隔离的，他们将有自己的接口，自己的路由表，自己的ARP表等等相关的内容。路由环境就变成有点像这样：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/VRF-solution-2.gif)

由上图可以看出，VRF1及VRF2有了自己的接口，也有了自己的路由表。并且相互之间是隔离的。
现在PC1要发送一个数据包到2.2.2.2，R1从接口GE0/0/1收到了这个数据包，由于此时GE0/0/1已经绑定到了VRF1，因此在执行目的IP的路由查找的时候，查的是VRF1的路由表，查找到匹配的路由条目后，间个数据包从其指示的GE0/0/1口转发给下一跳192.168.100.2。

那么如果PC1要访问3.3.3.3呢？数据包发到了R1，R1从接口GE0/0/1收到了这个数据包，于是它在做路由查找的时候，查的仍然是VRF1的路由表。经过查表后，它发现并无匹配的条目，因此将数据包丢弃。

完美实现了隔离，此外VRF还可以单独配置不同的组网方式：

不同的VRF可以配置不同的路由协议，类似虚拟机可以装不同的操作系统。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/VRF-pro.gif)



### VPN and VPC

VPC是Overlay网络，VxLAN-Overlay的建立依赖Tunnel，tunnel就是vpn的体现。

```bash
## 华三 vxlan隧道
# 创建模式为VXLAN隧道的Tunnel接口
interface tunnel tunnel-number mode vxlan
# 隧道的源端地址或源接口
source or destination

## 华为 vxlan隧道
# 创建NVE接口，并进入NVE接口视图
interface nve nve-number
# 配置VXLAN隧道源端VTEP的IP地址
source ip-address
```

VPN（Virtual Private Network）。VPN在公共的网络资源上虚拟隔离出一个个用户网络，例如IPsec VPN可以是在互联网上构建连接用户私有网络的隧道，MPLS VPN更是直接在运营商的PE设备上划分隔离的VRF给不同的用户。从提供服务的角度来，说如果VPC指的只是网络的话，那它跟VPN的概念是重复的。

VPC可以包含很多：

* subnet
* firewall
* Router
* and so on

VPC还可以横跨多个Available Zone

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vpc-diagram.png)



### 服务链

服务链定义：数据报文在网络中传递时，需要经过各种安全服务节点，按特定策略进行流分类后的报文，再按照一定顺序经过一组抽象业务功能节点，完成对应业务功能处理。这种方式打破了常规的网络转发逻辑，因此称为服务链。

服务链常见的服务节点（Service Node）：防火墙（FW）、负载均衡(LB)、入侵检测（IPS）、VPN等。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/overlay服务链.jpg)

上图是一个基于SDN的服务链流程。 SDN Controller实现对于SDN Overlay、NFV设备、vSwitch的统一控制；NFV提供虚拟安全服务节点；vSwitch支持状态防火墙的嵌入式安全；同时SDN Controller提供服务链的自定义和统一编排。我们看一下，假设用户自定义从VM1的VM3的业务流量，必须通过中间这样FW和LB等几个环节，通过SDN的服务链功能，业务流量一开始就严格按照控制器的编排顺序经过这组抽象业务功能节点，完成对应业务功能的处理，最终才回到VM3，这就是一个典型的基于SDN的服务链应用方案。

### Switch in VPC:field_hockey:

数据中心交换机都用上了，你还在玩堆叠？vrrp？确定不用上DRNI/VPC/M-LAG ？

现在几家网络设备厂商都有类似的功能。Cisco的nexus VPC、华为的M-LAG、华三的DRNI主体功能都是类似的，而这些功能都是用在自家的数据中心交换机上。也就是说，这种功能，在一般的园区网络交换机上是不具备的，而较常规的堆叠技术，这种DRNI、VPC、M-LAG则通过其独有的特性，屹立在数据中心级的设备之上，取代传统的堆叠技术。


#### huawei M-LAG

#### h3c DRNI

IRF是将多个物理设备虚拟成一台逻辑设备。

DRNI是跨设备的链路聚合。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/DRNI.png)

H3C的数据中心交换机组网建议使用DRNI，这种方式主要通过A、B两台设备作为主要节点，与堆叠的主要区别在于控制平面，即：组网后两台核心节点为单独的，数据交互通过中间的互联线路，两台的XGE1/0/52作为本次测试的keeplive线路，而FGE53、54则为peerlink线路。

有趣的是，这种控制平面不在同一台设备（两台单机设备）也可以跨设备创建类似堆叠时候用的port-channel，从而使得下联设备的网络质量得到保障，无论上联哪台核心节点故障，都不会影响到下联设备的数据转发，由于两台设备逻辑上为两台设备，那么此时也无论升级谁的底层固件，也都不会影响任何一个数据包的转发（比堆叠好）


DRNI和堆叠技术的区别

* 更高的可靠性：把链路可靠性从单板级提高到了设备级。
* 简化组网及配置：提供了一个没有环路的二层拓扑，同时实现冗余备份，不再需要繁琐的生成树协议配置，极大地简化了组网及配置。
* 独立升级：两台设备可以分别进行升级，保证有一台设备正常工作即可，对正在运行的业务几乎没有影响。

### 引用：

1. https://www.sdnlab.com/20510.html（全抄）
1. https://blog.csdn.net/Kangyucheng/article/details/88051969
1. https://blog.csdn.net/xupeng5200/article/details/123507076
