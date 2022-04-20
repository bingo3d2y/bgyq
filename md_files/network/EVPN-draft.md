## VPN-EVPN-draft

EVPN全称是Ethernet VPN



### Why EVPN?

EVPN（Ethernet VPN）呢？最初的VXLAN方案（RFC7348）中没有定义控制平面，是**手工配置VXLAN隧道**，然后通过流量泛洪的方式进行主机地址的学习。这种方式实现上较为简单，但是会导致网络中存在很多泛洪流量、网络扩展起来困难。

为了解决上述问题，人们在VXLAN中引入了EVPN作为VXLAN的控制平面，（VXLAN是一种NVO协议）。EVPN还能作为一些其他协议的控制面。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/evpn-and-vxlan.png)



https://info.support.huawei.com/info-finder/encyclopedia/zh/EVPN.html

#### MG-BGP

MP-BGP：MultiProtocol BGP

传统的BGP-4使用Update报文在对等体之间交换路由信息。一条Update报文可以通告一类具有相同路径属性的可达路由，这些路由放在NLRI（Network Layer Reachable Information，网络层可达信息）字段中。因为BGP-4只能管理IPv4单播路由信息，为了提供对多种网络层协议的支持（例如IPv6、组播），发展出了MP-BGP。MP-BGP在BGP-4基础上对NLRI作了新扩展。玄机就在于新扩展的NLRI上，扩展之后的NLRI增加了地址族的描述，可以用来区分不同的网络层协议，例如IPv6单播地址族、VPN实例地址族等。

EVPN也是借用了MP-BGP的机制，在L2VPN地址族下定义了新的子地址族——EVPN地址族，在这个地址族下又新增了一种NLRI，即EVPN NLRI。EVPN NLRI定义了几种BGP EVPN路由类型，这些路由可以携带主机IP、MAC、VNI、VRF等信息。这样，当一个VTEP学习到下挂的主机的IP、MAC地址信息后，就可以通过MP-BGP路由将这些信息发送给其他的VTEP，从而在控制平面实现主机IP、MAC地址的学习，抑制了数据平面的泛洪。

采用EVPN作为VXLAN的控制平面具有以下优势：

- 可实现VTEP自动发现、VXLAN隧道自动建立，从而降低网络部署、扩展的难度。
- EVPN可以同时发布二层MAC信息和三层路由信息。
- 可以减少网络中的泛洪流量

#### 基础概念

- ES (Ethernet Segment)

  如果CE多归到两个或更多PE，这组CE接入到不同PE的以太链路集合就是一个ES。如上图所示，CE1归属于PE1和PE2，CE1分别到PE1和PE2的这两条以太网链路就是一个ES。

- ESI (Ethernet Segment Identifier)

  全局标识唯一一个ES的ID。如下图所示，不同PE连接相同CE设备的接口，需要有相同的ESI。当ESI的值为0时，表示PE连接的是个单归CE。

- EVI（EVPN Instance）

  表示EVPN实例，EVPN是虚拟化私网，在一台PE设备上存在多个EVPN实例。类似于VPLS的VSI，用来标识一个VPN客户。

- MAC-VRF

  用来存储EVPN实例通过BGP扩展协议学习到的MAC地址信息。每个EVI都有独立的MAC-VRF。

### EVPN技术带来的价值

传统L2VPN技术面临挑战以虚拟专用局域网服务VPLS（Virtual Private LANService）技术为例来介绍传统L2VPN所面临的挑战，VPLS是一种早期出现的[MPLS](https://info.support.huawei.com/info-finder/encyclopedia/zh/MPLS.html) VPN技术，被广泛的应用在用户数据中心互连场景，能为企业用户提供多点到多点的广域以太网服务。但是由于VPLS技术具有一定的局限性，使其无法满足大规模复杂数据中心的需求。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/traditional-L2VPN.jpg)

*传统的L2VPN在数据中心互联网络部署拓扑图*

- 网络部署难

  PE设备要学习所有CE设备的MAC地址，MAC的表容量是有限的。且由于存在大量的手工配置，对PE设备的规格要求很高。

- 适用的网络规模有限

  PE设备间需要建立全连接PW使VPLS不适合大规模网络。

  没有控制平面，一旦MAC地址变化或故障发生切换，需要重新泛洪学习L2转发表，收敛性差。

- 链路带宽利用率低

  为了避免CE侧到PE侧产生环路，CE侧到PE侧只支持单活模式，而不支持多活模式，这样就使得链路的利用率很低。

EVPN技术有效地解决了上述问题：

- 如下图所示EVPN通过扩展BGP协议使MAC地址学习和发布过程从数据平面转移到控制平面。这样可以使设备在管理MAC地址时像管理路由一样，使目的MAC地址相同但下一跳不同的多条EVPN路由实现负载分担。

  ![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/evpn-and-tradition-L2VPN.jpg)

  

- 通过使用EVPN技术，PE设备之间不再需要建立全连接。这是因为在EVPN网络中PE设备之间是通过BGP协议实现相互通信的。BGP协议自带路由反射器功能，所以EVPN支持部署路由反射器，所有PE设备与反射器建立邻居关系，通过路由反射器来反射EVPN路由，大大降低了网络复杂度，减少了网络信令数量。

- PE设备通过ARP协议和MAC/IP地址通告路由分别学习本地和远端的MAC地址信息以及其对应的IP地址，并将这些信息缓存至本地。当PE设备再收到其他ARP请求后，将先根据ARP请求中的目的IP地址查找本地缓存的MAC与IP地址的对应信息，如果查找到对应信息，PE将返回ARP响应报文，避免ARP请求报文向其他PE设备广播，减少网络资源消耗。

### 华三-evpn配置

集中式EVPN网关配置组网图

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/集中式evpn.png)

当设备作为EVPN网关时，需要配置VXLAN隧道工作在三层转发模式。当设备作为VTEP时，VXLAN隧道工作在二层转发模式、三层转发模式均可。

如果VXLAN隧道工作在三层转发模式，则设备将VXLAN封装后的报文转发给下一跳时是否携带VLAN tag，由本命令中的**tagged**、**untagged**参数决定，而不是由报文的出接口类型决定。请根据实际情况选择**tagged**、**untagged**参数

* 设备通过Access端口、Trunk或Hybrid端口的PVID连接下一跳时，需要指定**untagged**参数。
*  设备通过Trunk或Hybrid端口的非PVID连接下一跳时，需要指定**tagged**参数。

####  vtep evpn

```bash
(1)      配置IP地址和单播路由协议

# 在VM 1和VM 3上指定网关地址为10.1.1.1；在VM 2和VM 4上指定网关地址为10.1.2.1。（具体配置过程略）

# 请按照图2-1配置各接口的IP地址和子网掩码；在IP核心网络内配置OSPF协议，确保交换机之间路由可达。（具体配置过程略）

(2)      配置Switch A

# 开启L2VPN能力。

<SwitchA> system-view

[SwitchA] l2vpn enable

# 配置VXLAN隧道工作在二层转发模式。

[SwitchA] undo vxlan ip-forwarding

# 关闭远端MAC地址自动学习功能。

[SwitchA] vxlan tunnel mac-learning disable

# 在VSI实例vpna下创建EVPN实例，并配置自动生成EVPN实例的RD和RT。

[SwitchA] vsi vpna

[SwitchA-vsi-vpna] arp suppression enable

[SwitchA-vsi-vpna] evpn encapsulation vxlan

[SwitchA-vsi-vpna-evpn-vxlan] route-distinguisher auto

[SwitchA-vsi-vpna-evpn-vxlan] vpn-target auto

[SwitchA-vsi-vpna-evpn-vxlan] quit

# 创建VXLAN 10。

[SwitchA-vsi-vpna] vxlan 10

[SwitchA-vsi-vpna-vxlan-10] quit

[SwitchA-vsi-vpna] quit

# 在VSI实例vpnb下创建EVPN实例，并配置自动生成EVPN实例的RD和RT。

[SwitchA] vsi vpnb

[SwitchA-vsi-vpnb] arp suppression enable

[SwitchA-vsi-vpnb] evpn encapsulation vxlan

[SwitchA-vsi-vpnb-evpn-vxlan] route-distinguisher auto

[SwitchA-vsi-vpnb-evpn-vxlan] vpn-target auto

[SwitchA-vsi-vpnb-evpn-vxlan] quit

# 创建VXLAN 20。

[SwitchA-vsi-vpnb] vxlan 20

[SwitchA-vsi-vpnb-vxlan-20] quit

[SwitchA-vsi-vpnb] quit

# 配置BGP发布EVPN路由。

[SwitchA] bgp 200

[SwitchA-bgp-default] peer 4.4.4.4 as-number 200

[SwitchA-bgp-default] peer 4.4.4.4 connect-interface loopback 0

[SwitchA-bgp-default] address-family l2vpn evpn

[SwitchA-bgp-default-evpn] peer 4.4.4.4 enable

[SwitchA-bgp-default-evpn] quit

[SwitchA-bgp-default] quit

# 在接入服务器的接口HundredGigE1/0/1上创建以太网服务实例1000，该实例用来匹配VLAN 2的数据帧。

[SwitchA] interface hundredgige 1/0/1

[SwitchA-HundredGigE1/0/1] service-instance 1000

[SwitchA-HundredGigE1/0/1-srv1000] encapsulation s-vid 2

# 配置以太网服务实例1000与VSI实例vpna关联。

[SwitchA-HundredGigE1/0/1-srv1000] xconnect vsi vpna

[SwitchA-HundredGigE1/0/1-srv1000] quit

# 在接口HundredGigE1/0/1上创建以太网服务实例2000，该实例用来匹配VLAN 3的数据帧。

[SwitchA-HundredGigE1/0/1] service-instance 2000

[SwitchA-HundredGigE1/0/1-srv2000] encapsulation s-vid 3

# 配置以太网服务实例2000与VSI实例vpnb关联。

[SwitchA-HundredGigE1/0/1-srv2000] xconnect vsi vpnb

[SwitchA-HundredGigE1/0/1-srv2000] quit

[SwitchA-HundredGigE1/0/1] quit
```

end

#### gw evpn

```bash
# 开启L2VPN能力。

<SwitchC> system-view

[SwitchC] l2vpn enable

# 关闭远端MAC地址自动学习功能。

[SwitchC] vxlan tunnel mac-learning disable

# 在VSI实例vpna下创建EVPN实例，并配置自动生成EVPN实例的RD和RT。

[SwitchC] vsi vpna

[SwitchC-vsi-vpna] evpn encapsulation vxlan

[SwitchC-vsi-vpna-evpn-vxlan] route-distinguisher auto

[SwitchC-vsi-vpna-evpn-vxlan] vpn-target auto

[SwitchC-vsi-vpna-evpn-vxlan] quit

# 创建VXLAN 10。

[SwitchC-vsi-vpna] vxlan 10

[SwitchC-vsi-vpna-vxlan-10] quit

[SwitchC-vsi-vpna] quit

# 在VSI实例vpnb下创建EVPN实例，并配置自动生成EVPN实例的RD和RT。

[SwitchC] vsi vpnb

[SwitchC-vsi-vpnb] evpn encapsulation vxlan

[SwitchC-vsi-vpnb-evpn-vxlan] route-distinguisher auto

[SwitchC-vsi-vpnb-evpn-vxlan] vpn-target auto

[SwitchC-vsi-vpnb-evpn-vxlan] quit

# 创建VXLAN 20。

[SwitchC-vsi-vpnb] vxlan 20

[SwitchC-vsi-vpnb-vxlan-20] quit

[SwitchC-vsi-vpnb] quit

# 配置BGP发布EVPN路由。

[SwitchC] bgp 200

[SwitchC-bgp-default] peer 4.4.4.4 as-number 200

[SwitchC-bgp-default] peer 4.4.4.4 connect-interface loopback 0

[SwitchC-bgp-default] address-family l2vpn evpn

[SwitchC-bgp-default-evpn] peer 4.4.4.4 enable

[SwitchC-bgp-default-evpn] quit

[SwitchC-bgp-default] quit

# 创建VSI虚接口VSI-interface1，并为其配置IP地址，该IP地址作为VXLAN 10内虚拟机的网关地址。

[SwitchC] interface vsi-interface 1

[SwitchC-Vsi-interface1] ip address 10.1.1.1 255.255.255.0

[SwitchC-Vsi-interface1] quit

# 配置VXLAN 10所在的VSI实例和接口VSI-interface1关联。

[SwitchC] vsi vpna

[SwitchC-vsi-vpna] gateway vsi-interface 1

[SwitchC-vsi-vpna] quit

# 创建VSI虚接口VSI-interface2，并为其配置IP地址，该IP地址作为VXLAN 20内虚拟机的网关地址。

[SwitchC] interface vsi-interface 2

[SwitchC-Vsi-interface2] ip address 10.1.2.1 255.255.255.0

[SwitchC-Vsi-interface2] quit

# 配置VXLAN 20所在的VSI实例和接口VSI-interface1关联。

[SwitchC] vsi vpnb

[SwitchC-vsi-vpnb] gateway vsi-interface 2

[SwitchC-vsi-vpnb] quit

(5)      配置Switch D

```

end

#### border 

```bash
# 配置Switch D与其他交换机建立BGP连接。

<SwitchD> system-view

[SwitchD] bgp 200

[SwitchD-bgp-default] group evpn

[SwitchD-bgp-default] peer 1.1.1.1 group evpn

[SwitchD-bgp-default] peer 2.2.2.2 group evpn

[SwitchD-bgp-default] peer 3.3.3.3 group evpn

[SwitchD-bgp-default] peer evpn as-number 200

[SwitchD-bgp-default] peer evpn connect-interface loopback 0

# 配置BGP发布EVPN路由，并关闭BGP EVPN路由的VPN-Target过滤功能。

[SwitchD-bgp-default] address-family l2vpn evpn

[SwitchD-bgp-default-evpn] peer evpn enable

[SwitchD-bgp-default-evpn] undo policy vpn-target

# 配置Switch D为路由反射器。

[SwitchD-bgp-default-evpn] peer evpn reflect-client

[SwitchD-bgp-default-evpn] quit

[SwitchD-bgp-default] quit
```

验证

查看Switch C上的EVPN路由信息，可以看到Switch C发送了网关的MAC/IP路由和IMET路由，并接收到Switch A和Switch B发送的MAC/IP路由和IMET路由。

### 华为evpn配置

https://support.huawei.com/enterprise/zh/doc/EDOC1100164807#ZH-CN_TOPIC_0284778387


