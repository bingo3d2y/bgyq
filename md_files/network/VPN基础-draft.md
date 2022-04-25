## VPN基础-draft

VPN的英文全称是“Virtual Private Network”，即“虚拟专用网络”。VPN是为了替代物理专用网络而出现，物理专用网络是通过架设或者租用物理线路来实现跨地域的互联。

### VPN技术的定义：

VPN: virtual private network

多个站点之间的客户通过部署在相同的基础设施之上进行互联，它们之间的访问及安全策略与专用网络（专线）相同。通俗点说就是使用公共网络设施实现私有的连接，各个私有连接在公共网络上对其它的私有网络是不可见的。相对于专线的物理隔离技术来说，VPN技术更多意义上是一种逻辑隔离技术。它主要是在客户要求各个站点通过服务提供商公共网络进行连接的需求背景下，由专线网络的概念引申而来的。

任何符合如下两个条件的网络我们都可以泛泛地称它为VPN网络：

1、使用共享的公共环境实现各个私有网络的连接；

2、不同的私有网络间（除非有特殊要求，如两个公司间的互访要求）是相互不可见的。

#### 关键组件

VPN网络的关键组件如下:

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vpn-basic.jpg)

* CE（Custom Edge）：直接与服务提供商相连的用户设备。
* PE（Provider Edge Router）：指骨干网上的边缘设备（如路由器、ATM交换机、帧中继交换机等），与CE相连,主要负责VPN业务的接入。
* P （Provider Router）：指骨干网上的核心路由器，主要完成路由和快速转发功能。P设备根据网络结构及规模可有可无。
* SP：Service Provider，服务提供商

任何一个VPN网络都是由这几个组件全部或部分组成的。

### VPN and Tunnel

A **virtual private network** (**VPN**) extends a private network across a public network, and enables users to send and receive data across shared or public networks as if their computing devices were directly connected to the private network. 

>  **private network** is a network that uses private IP address space.

VPN is Virtual Private Network. This is not a protocol. This is a concept. There are many different types of VPN technologies, but they have one thing in common: **they provide traffic separation.** The simplest example of VPN is a VLAN. Traffic inside the VLAN is segregated from traffic in other VLANs and to communicate with external networks (not belonging to this VLAN) it needs a layer 3 gateway. There are VPNs that provide data confidentiality via encryption, like IPSEC VPN. And there are VPNs that only provide traffic segregation, like VLAN or MPLS VPN (also called IP VPN). The main feature of any VPN is to make sure that all segments of the network stay under your administrative control. 

Tunneling is one of the VPN techniques that can be used with or without encryption.

 VPN是一个概念，任何一个可以是公网进行数据交流的私有网络都可以叫vpn. vpn 共同点： 流量分离

#### Tunnel

In computer networks, a **tunneling protocol** is a communications protocol that allows for the movement of data from one network to another.

典型应用场景：

ServerA 和 ServerB 可以想象成RouteA 和 RouteB.

```bash
                          |                                           |
                          |       1.1.1.1               2.2.2.2       |
             Piviate      |     +---------+  Public   +---------+     | Private
                          +-----| ServerA +-----------+ ServerB +-----+
             Networl      |     +---------+  Network  +---------+     | Network
                          |                                           |
192.168.0.0/24            |                                           | 192.168.1.0/24 
```

如果只通过，NAT手段，最多实现`192.168.0.0/24 ---> 2.2.2.2`,无法实现两个私有网段的直接通信即：

`192.168.0.0/24 <--> 192.168.1.0/24`.

假如需要将对外网ip202.103.96.112网口eth1的访问，映射到对内网ip192.168.0.112的访问；

`iptables -t nat -A PREROUTING -i eth1 -p tcp -d 202.103.96.112 --dport 80 -j DNAT --to-destination 192.168.0.112:80`

但通过ip tunnel建立ipip隧道，再通过iptables进行nat，便可以实现。

**Step 1. 建立ip隧道**
ServerA配置iptunnel,并给tunnel接口配置上ip

```bash
$ ip tunnel add a2b mode ipip remote 2.2.2.2 local 1.1.1.1
$ ifconfig a2b 192.168.2.1 netmask 255.255.255.0
```

ServerB配置iptunnel,并给tunnel接口配置上ip

```bash
$ ip tunnel add a2b mode ipip remote 1.1.1.1 local 2.2.2.2
$ ifconfig a2b 192.168.2.2 netmask 255.255.255.0
```

隧道配置完成后，请在ServerA上192.168.2.2，看是否可以ping通，ping通则继续，ping不通需要再看一下上面的命令执行是否有报错
**Step 2. 添加路由和nat**
ServerA上，添加到192.168.1.0/24的路由

```bash
$ /sbin/route add -net 192.168.1.0/24 gw 192.168.2.2
```

ServerB上，添加iptables nat，将ServerA过了访问192.168.1.0/24段的包进行NAT，并开启ip foward功能

```bash
$ iptables -t nat -A POSTROUTING -s 192.168.2.1 -d 192.168.1.0/24 -j MASQUERADE
$ sysctl -w net.ipv4.ip_forward=1
$ sed -i '/net.ipv4.ip_forward/ s/0/1/'  /etc/sysctl.conf
```

至此，完成了两端的配置，ServerA可以直接访问ServerB 所接的私网了。

#### **What's the difference between tunneling and VPN?**

Tunnelling is actually a protocol that allows secure data transfer from one network to another. It uses a process called ‘encapsulation’ through which the private network communications are sent to the public networks. On the other hand, **VPN is based on the idea of tunnelling. It involves establishing and maintaining a logical network connection.** On this connection, packets constructed in a specific VPN protocol format are encapsulated within some other base or carrier protocol, then transmitted between VPN client and server, and finally de-encapsulated on the receiving side. Hope this helps.

A VPN is a secure, encrypted connection over a publicly shared network. **Tunneling is the process by which VPN packets reach their intended destination, which is typically a private network**. Many VPNs use the IPsec protocol suite.

有些隧道协议可能是不安全的比如,IPIP Tunnel，但是VPN都是安全加秘密的传输协议。



### VPN运营模式分类

VPN的分类，VPN的分类有许多种方式。我们一般常接触的分类方式是按照运营模式进行分类:
  1）、CPE-Based VPN: 设备放置在用户侧(Custom Premise Equipment)

由用户管理或委托运营商进行管理；
   2）、Network-Based VPN: 设备放置在运营商网络侧

用户设备不需要感知VPN由运营商管理，又称为Provider Provide VPN。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vpn运营模式分类.jpg)



### VPN技术的分类

VPN的概念早在10多年前便已经提出，至今经历了较长的演进过程。

随着新技术（如加密技术、隧道技术、MPLS技术）的不断涌现，及新的客户需求的出现，目前VPN的概念变得越来越复杂。所以引入了VPN的分类，以区分各种不同的VPN技术。

VPN技术主要从四个方面进行分类：

1、VPN要解决的业务问题；

2、服务提供商在哪一层与客户交换拓扑信息；

3、在服务提供商中用于实现VPN服务的第二层或第三层技术；

4、网络的拓扑结构。

每一个分类与其它的分类间都不是独立的，都是有相互关联的，它们共同决定了VPN网络的架构。

#### 专线网络

VPN的概念最早是从专线引发的。专线网络具有以下特点：

1、安全性高，线路为客户专用，不同用户间是物理隔离的；

2、价格昂贵；

3、带宽浪费严重。

正因为专线网络具有的一些固有缺陷，随着统计复用技术的出现，一些新的共享带宽技术逐渐替代了专用线路，并可以提供与专线相同的服务。如帧中继、X.25技术等，通过VC在公共的交换设备上提供虚链路，实现用户的私有通信。由于是共享带宽，所以价格比专线便宜很多。这些技术构成了早期的VPN网络。

VPN是为了替代物理专用网络而出现，物理专用网络是通过架设或者租用物理线路来实现跨地域的互联。

#### Overlay VPN

Overlay VPN的主要特点是客户的路由协议总是在客户设备之间交换，而服务提供商对客户网络的内部结构一无所知。

典型的Overlay VPN技术有：

第二层隧道技术如X.25、帧中继(Frame Relay)、ATM；

> 需要ATM交换机和帧中继交换机支持

第三层隧道技术如IP-over-IP隧道技术等。

其中IP-over-IP隧道技术通过专用IP主干或INTERNET来实现覆盖VPN网络，最常用的技术是GRE VPN和IP See VPN等。

> OpenStack中的VPNaaS就是基于IPSec实现的。

Overlay VPN的主要缺陷是：

* 连接性比较复杂时管理开销非常大；
* 用户需要在自己的设备上管理各种路由，例如到Beta的路由，从哪个VPN tunnel出，去Gamma的路由，又该从哪个VPN tunnel出。当用户规模增大的时候，这尤其不方便。
* 要正确提供VC的容量，必须了解站点间的流量情况，比较难统计。

##### Overlay VPN模型

VPN最开始的实现，就是服务提供商（Service provider）向用户提供模拟的线路，用户基于模拟线路来实现跨地域互连，这里的模拟线路也叫做VC（Virtual circuit）。这种VPN的实现方式，就是overlay 模型。VC的提供者，也就是服务提供商不参与用户网络的组建。用户需要在自己的设备CPE（Customer Premises Edge）上，建立跨地域的连接。服务提供商的角色如下图所示：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/overlay-vpn-arc.png)

从用户的角度来说，逻辑上，用户各个地域之间的设备，是按照下图方式连接的。用户的CPE之间路由协议数据通常是在用户设备之间直接交换。例如，Alpha要访问Beta，那用户需要自己在Alpha网络里面配置好目的地是Beta的路由（或者通过IGP，BGP配置），再通过VC线路转发。

#### Peer-to-Peer VPN

Peer-to-Peer VPN的主要特点是服务提供商的PE设备直接参与CE设备的路由交换。该VPN模型的实现依据是：如果去往某一特定网络的路由未被安装在路由器的转发表中，在那台路由器上，该网络不可达。实施Peer-to-Peer VPN的前提是所有CE端的地址是全局唯一的！

MPLS L3 VPN是一种peer-to-peer模式的实现，但不是唯一的实现。实际上，现在的服务提供商网络里面，存活着其他的peer-to-peer模型的VPN网络，这里就不展开了。相比其他的peer-to-peer 模式VPN，MPLS L3 VPN的优势在于控制简单，成本低，支持IP地址重叠等等其他。总之就是那么好，所以现在MPLS L3 VPN已经成为peer-to-peer 模式VPN的主流实现。

典型的Peer-to-Peer VPN技术有：

1、专用PE接入技术

2、共享PE接入技术

##### 共享PE p2p

其中，**共享PE Peer-to-Peer VPN**（如下图所示）的主要特点是：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/p2p-vpn-share.jpg)

1、所有VPN用户的CE都连到同一台PE上，PE与不同的CE之间运行不同的路由协议（或者是相同路由协议的不同进程，比如OSPF）。

2、由始发PE将VPN路由发布到公网上，在接收端的PE上将这些路由过滤后再发给相应的CE设备。

3、共享路由接入方式为避免不同VPN用户间的路由外漏，需要配置大量的ACL进行过滤，对于设备的性能及管理开销都会有很大的负担。

##### 专用PE p2p

**专用PE Peer-to-Peer VPN**（如下图所示）的主要特点是：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/p2p-vpn-private.jpg)

为每一个VPN单独准备一台PE路由器，PE和CE之间可以运行任意的路由协议，与其它VPN无关。PE与P之间运行BGP或其他路由协议，并使用路由属性进行过滤。

优点：无需配置任何ACL,大大减轻了管理员的配置工作量。

缺点：每一个VPN用户都要新增一台专用的PE，代价过于昂贵。

##### P2P VPN 模型

peer-to-peer被提出。peer-to-peer model下，服务商的edge设备是一个路由器（PE），直接与用户的edge设备（CE）交换路由。也就是说，服务提供商不仅参与数据的传递，同时也参与路由协议数据的传递。 peer-to-peer模式下，服务商的角色如下图所示：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/P2P-vpn-arc.jpg)



这与最开始的租用线路已经很不一样了。同时服务提供商也参与用户网络的构建，两者分工变得模糊。但是peer-to-peer解决了overlay模型的实际问题。

1. 用户在Delta新开公司，用户只需要与Delta对应的PE连接和配置即可。服务提供商再对相应的PE配置做修改，即可完成Delta与其他所有分公司的连接。
2. 用户CE只与PE连接，用户可以把路由都指向相应的PE。同时因为这个原因，对用于CE的要求降低。
3. 服务提供商可以提供最优的路由路径来传递用户的私有网络服务。

没有悬念，MPLS L3 VPN是一种peer-to-peer模式的实现，但不是唯一的实现



##### n2n

[n2n](https://www.ntop.org/products/n2n/#)是一个开源的`2`层`P2P`架构`VPN`，有如下特点：

1. 基于`P2P`协议的加密`2`层专用网络
2. 边缘节点(`edge node`)的加密是使用带有用户自定义密码的开放协议
3. 每个`n2n`用户可以同时加入不同的网络（或称为社区）
4. `n2n`能够以反向流量方向（即从外部到内部）跨越`NAT`和防火墙。防火墙不再是`IP`级别直接通信的障碍。
5. `n2n`网络并不意味着是独立的：可以通过`n2n`和非`n2n`网络连接。

