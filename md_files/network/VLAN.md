## VLAN

Virtual Local Area Network， 虚拟局域网。 

Creating a Vlan means you are dividing up the BROADCAST DOMAINS. 

> VLAN 优点隔离广播域 BROADCAST DOMAINS

VLANs use software to emulate separate physical LANs. Each VLAN is thus a separate broadcast domain and a separate network.

### VLAN tag 

802.1Q Tag包含4个字段，其含义如下：

* Type

长度为2字节，表示帧类型。取值为0x8100时表示802.1Q Tag帧。如果不支持802.1Q的设备收到这样的帧，会将其丢弃。

* PRI

Priority，长度为3比特，表示帧的优先级，取值范围为0～7，值越大优先级越高。用于当交换机阻塞时，优先发送优先级高的数据包。

* CFI

Canonical Format Indicator，长度为1比特，表示MAC地址是否是经典格式。CFI为0说明是经典格式，CFI为1表示为非经典格式。用于区分以太网帧、FDDI（Fiber Distributed Digital Interface）帧和令牌环网帧。在以太网中，CFI的值为0。

* VID

VLAN ID，长度为12比特，表示该帧所属的VLAN。在VRP中，可配置 的VLAN ID取值范围为0～4095，但是0和4095协议中规定为保留的VLAN ID，不能给用户使用。

### Switch Port and Link-type

以太网端口有三种链路类型：

- Access：只能属于一个 VLAN，一般用于连接计算机的端口
- Trunk：可以属于多个 VLAN，可以接收和发送多个 VLAN 的报文，一般用于交换机之间连接的接口
- Hybrid：属于多个 VLAN，可以接收和发送多个 VLAN 报文，既可以用于交换机之间的连接，也可以 用户连接用户的计算机。 Hybrid 端口和 Trunk 端口的不同之处在于 Hybrid 端口可以允许多个 VLAN 的报文发送时不打标签，而 Trunk 端口只允许缺省 VLAN 的报文发送时不打标签

 vlan的链路类型可以分为接入链路和干道链路。

（1）接入链路（access link）指的交换机到用户设备的链路，即是接入到户，可以理解为由交换机向用户的链路。由于大多数电脑不能发送带vlan tag的帧，所以这段链路可以理解为不带vlan tag的链路。

（2）干道链路（trunk link）指的交换机到上层设备如路由器的链路，可以理解为向广域网走的链路。这段链路由于要靠vlan来区分用户或者服务，所以一般都带有vlan tag。



### VLAN的不足

#### VLAN 数量

VLAN 使用 12-bit 的 VLAN ID，因此第一个不足之处就是最多只支持 4096 个 VLAN 网络，无法满足虚拟化和云计算时代的用户需求。eg：公有云100W个用户，每个用户一个或N个租户。

##### QinQ

QinQ是为了扩大VLAN ID的数量而提出的技术（IEEE 802.1ad），外层tag称为Service Tag，而内层tag则称为Customer Tag。

通过将一层 VLAN Tag 封装到私网报文上，使其携带两层 VLAN Tag 穿越运营商的骨干网络(又称公网)，从而使运营商能够利用一个 VLAN 为包含多个 VLAN 的用户网络提供服务。

QinQ报文在运营商网络中传输时带有双层VLAN Tag：

- 内层 VLAN Tag：为用户的私网 VLAN Tag，Customer VLAN Tag (简称 CVLAN)。设备依靠该 Tag 在私网中传送报文。
- 外层 VLAN Tag：为运营商分配给用户的公网 VLAN Tag， Service VLAN Tag(简称 SVLAN)。设备依靠该 Tag 在公网中传送 QinQ 报文。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/QinQ.jpeg)

在公网的传输过程中，设备只根据外层 VLAN Tag 转发报文，而内层 VLAN Tag 将被当作报文的数据部分进行传输。

#### VLAN间通讯互联

VLAN在分割了二层广播域，也严格地隔离了各个VLAN之间的任何二层流量，属于不同VLAN的用户之间不能直接进行二层通信。需要借助”三层设备“实现不同VLAN之间的通讯。

##### 方法1：普通路由器互联

在路由器上为每个VLAN分配一个单独的接口，并使用一条物理链路连接到二层交换机上。

当VLAN间的主机需要通信时，数据会经由路由器进行三层路由，并被转发到目的VLAN内的主机，这样就可以实现VLAN之间的相互通信。

这种方式弊端很明显：

1. 路由器端口有限
2. 端口利用率低，比如某些VLAN之间的主机可能不需要频繁进行通信

现实中没人这么干

##### 方法2：单臂路由，router-on-a-stick

单臂路由（router-on-a-stick）是指在路由器的一个接口上通过配置子接口（或“逻辑接口”，并不存在真正物理接口）的方式，实现原来相互隔离的不同VLAN（虚拟局域网）之间的互联互通。

在交换机和路由器之间仅使用一条物理链路连接。在交换机上，把连接到路由器的端口配置成Trunk类型的端口，并允许相关VLAN的帧通过。在路由器上需要创建子接口，逻辑上把连接路由器的物理链路分成了多条。一个子接口代表了一条归属于某个VLAN的逻辑链路。

> 需要子接口的原因：
>
> 交换机的Trunk类型端口，只允许带有VLAN tag的帧通过，也就是说交换机Trunk端口出去的帧不做区分。
>
> 路由器的子接口，分别对接从交换机Trunk端口进入的VLAN 帧并根据不同路由子接口允许进入不同的VID帧，进行VLANs间的数据路由。

单臂路由缺点：

- “单臂”为网络骨干链路，容易形成网络瓶颈（带宽低，转发率不高）
- 子接口依然依托于物理接口，应用不灵活
- VLAN间转发需要查看路由表，严重浪费设备资源



![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/router-on-a-stick.png)



##### 方法3：三层交换机

三层交换机在原有的二层交换机上增加了路由功能，因为数据没有像单臂路由那样经过物理线路进行路由，很好解决了带宽瓶颈的问题。

在三层交换机上配置VLANIF接口来实现VLAN间路由。如果网络上有多个VLAN，则需要给每个VLAN配置一个VLANIF接口，并给每个VLANIF接口配置一个IP地址。用户设置的缺省网关就是三层交换机中VLANIF接口的IP地址。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/layer-3-switch.jpg)

##### 不同VLAN中IP(subnet)重复：大问题

公有云提供商的业务要求将实体网络租借给多个不同的用户，这些用户对于网络的要求有所不同，而不同用户租借的网络有很大的可能会出现IP地址、MAC地址的重叠，传统的VLAN仅仅解决了同一链路层网络广播域隔离的问题，而并没有涉及到网络地址重叠的问题，因此需要一种新的技术来保证在多个租户网络中存在地址重叠的情况下依旧能有效通信的技术。

##### MAC表数量限制

TOR（Top Of Rack）交换机的MAC表大小限制。

这个MAC地址表是通过交换机的flood-learn学习并记录在交换机的内存。交换机的内存比较宝贵，所以MAC地址表的大小通常是有限的。现在因为虚拟化，整个数据中心的MAC地址多了几十倍，那相应的交换机里面的MAC地址表也需要扩大几十倍。如果交换机不支持这么大的MAC地址表，那么就会导致MAC地址表溢出。溢出之后，交换机不能将新的MAC地址学习到自己的MAC地址表。如果交换机收到这些MAC地址的数据帧，因为不能通过查表转发，会flood到所有的端口。这不但增加了交换机的负担，还增加了网络中其他设备的负担。为了避免这个问题，可以用一些更大容量的交换机，但是相应的成本也要上升，而且还不能从根本上解决这个问题。



### VLAN and IP Subnetwork：666

穷举一下，VLAN和IP子网，各种划分，会有4种情形

1、A和B相同IP子网，相同VLAN：可以正常通讯，单播和广播通讯都会送达。

2、A和B不同IP子网，相同VLAN：需要路由器才能单播通讯，但是会有广播跨IP子网互相干扰。

3、A和B相同IP子网，不同VLAN：完全无法通讯，即使有路由器也不行。因为IP协议认为是发给近邻直连网络，数据不会路由给网关，会进行ARP请求广播，企图直接与目的主机通讯，可是由于B在另一个VLAN，网关不予转发ARP请求广播，B收不到ARP请求。结局是网络层ARP无法解析获得数据链路层需要的目的MAC，通讯失败。（除非：路由器上对两个VLAN之间进行桥接；或者路由器上进行静态NAT，若生成树设置不当容易把交换机搞死千万别作）

> 不同vlan分配不同子网，方便路由管理，避免额外手工配置。

4、A和B不同IP子网，不同VLAN：需要路由器才能进行单播通讯，不会有广播跨子网干扰。



情形1是常见的小型局域网；

情形2不应出现在正确配置的网络里，因为花了钱买路由器（三层交换机）但是钱白花了，没能隔离广播风暴；

情形3是常见的家庭WiFi路由器配置故障，一些运营商进户线路经过NAT是192.168.1.0/24的地址段，家用WiFi路由器恰好也用192.168.1.0/24这个地址段作为Lan口默认地址，路由器Lan端和WAN端冲突，傻掉了（可以这样理解：家用路由器的Lan端和WAN端分别处于两个不同的VLAN）；

情形4是常见的企业局域网。

三层交换机用在什么地方？三层交换机最主要最擅长的应用就是进行VLAN之间的数据路由转发，这种应用场合下它的转发速度要远胜过专业路由器，它可以做到以二层帧交换的速度进行跨VLAN三层路由转发作业。但是通常来说它NAT效率不如专业路由器或防火墙，不能跨二层协议处理数据包，通常把它用在局域网网络核心节点。在局域网网络末节，比如说Internet接入处，通常都会再专设一台路由器或防火墙来进行链路层协议转换和NAT转换。可以这么简单的理解，三层交换机是一种只有以太网接口，只能处理以太网协议的路由器。（难道除了以太网还有别的？当然有，比如串行、PPP）虽然三层交换机和路由器在内部运作机制上不一样，对于用户而言，数据路由转发这个功能它俩都具备。



### Do VLANs require different subnet？: 有意思

https://community.spiceworks.com/topic/1116440-do-vlans-require-different-subnets

No, VLANs don't require different subnets. Different subnets require different subnet addresses if they ever need to be able to route and/or talk to each other) and by extension if one VLAN wants to talk to a different VLAN it must use different addresses so we can make a routing decision to the right place. **Most people make some relationship between the subnet address assignment and the VLAN ID (similart to your example) because it's convenient, otherwise they would be constantly consulting a table of what subnet addressing is used on what VLAN**.

Just take the V out of VLAN, different VLANs are effectively different LANs but thanks to the V can use the same switches, routers and trunks or to put it another way, if you want a separate LAN you can use a VLAN and not have to buy two of everything to build a separate LAN entirely.

In one instance the customer is already using the addresses I need to use but their network never needs to talk to mine so I just use a separate VLAN on the same switch which uses the same subnet address they do and because they never need to communicate, the separate VLAN keeps another LAN with the same address scheme on the same switch totally separate.  This was a VoIP installation and their data network was already using the same IP address block I needed to use but it didn't matter, in some cases the two separate subnets with overlapping address space are carried on the same port (e.g. switchport voice VLAN x) and they don't interfere or have duplicate IPs (even though they do have duplicates) because they are on separate LANs (or VLANs).

#### Multiple VLANs in the same subnet

I want to avoid having many Subnets (and vNIC's in TMG) just to isolate sets of a few hosts.

```
IP: 10.0.0.1         (TMG server)       VLAN:1 ~ 3

IP: 10.0.0.11 ~ 20   (Hosts group 1)    VLAN:1

IP: 10.0.0.21 ~ 30   (Hosts group 2)    VLAN:2

IP: 10.0.0.31 ~ 40   (Hosts group 3)    VLAN:3
```

Note that I don't want them to connect to each other, so ARP/inter-vlan routing (within the subnet) is not required.

Of course you can do that, but it is not the recommended way.

VLANs use software to emulate separate physical LANs. Each VLAN is thus a separate broadcast domain and a separate network.

As you have identified, routing between these VLANs would be difficult, because they are the same subnet. If the addresses are all different it is possible to route traffic using a very large number of rules which don't correspond to the actual subnet configuration and will confuse anyone who inherits this from you. However, it is completely permissible to use the same RFC1918 subnets on different physical networks. You could likely even make all the addresses the same.

#### Example

Let's say I have four PCs connected to one CISCO switch.

- **PC 1** has IP address `192.168.10.1` and is connected to the access port - *vlan ID 10*.
- **PC 2** has IP address `192.168.10.2` and is connected to the access port - *vlan ID 10*.
- **PC3** has IP address `192.168.10.1` (same IP address as PC1) and is connected to the access port - *vlan ID 20*
- **PC4** has IP address `192.168.10.2` ( same IP address as PC2) and is connected to the access port - *vlan ID 20*

Can PC1 ping PC2?
Can PC3 ping PC4?



### 引用

1. https://blog.csdn.net/weixin_52122271/article/details/112383249
2. https://blog.51cto.com/u_13212728/2516078
3. 

