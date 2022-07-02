## VxLAN

VxLAN打破了二层网络的局限性，通过L2-in-UDP使得只要三层路由可达就可以无限延伸二层网络。

传统网络架构以三层为主，主要是以控制南北数据流量为主（主要是服务器、虚拟机与外部通信），由于数据中心虚拟机的大规模使用，虚拟机迁移的特点以东西流量为主，在迁移后需要其IP地址、MAC地址等参数保持不变，如此则要求业务网络是一个二层网络。

### vxlan 路由:优雅！

包怎么封装的，包是无意识到呢，怎么vm发出的包就变成vxlan格式的了
哈哈哈，所以vxlan必须和tunnel绑定！在通过vsi做网关，引流，封装，路由！完事，路由到vxlan gw解封装。

**VxLAN必关联Tunnel隧道，Tunnel IP作为VxLAN声明的网段的网关，从而实现了overlay-vxlan网段的路由。**

> 完全类似calico的`169.254.1.1`的作用！！！

基于vxlan实现的overlay/vpc，各种内网IP怎么实现被规划的业务网段、管理网段和存储网段访问。

访问vxlan网段路由:`10.100.180.0/24  direct 10.1.34.22 Vlan352`

即，vxlan网段通过vsi将路由引导到Leaf/Spine上的vlanif，然后该交换机的其他端口与规划的存储、业务和管理网段配置互通。就可以实现vpc网段和underlay网络互通。



### What is VxLAN

VXLAN（Virtual eXtensible Local Area Network，虚拟扩展局域网），是由IETF定义的NVO3（Network Virtualization over Layer 3）标准技术之一，采用L2 over L4（MAC-in-UDP）的报文封装模式，将二层报文用三层协议进行封装，可实现二层网络在三层范围内进行扩展，同时满足数据中心大二层虚拟迁移和多租户的需求。

> NVO3是基于三层IP overlay网络构建虚拟网络的技术的统称，VXLAN只是NVO3技术之一。除此之外，比较有代表性的还有NVGRE、STT。

**VXLAN技术属于大二层Overlay网络技术范畴，是数据中心网络最核心的技术。**



### Why VxLAN

与NVGRE相比，VXLAN不需要改变报文结构即可支持L2~L4的链路负载均衡；与STT相比，VXLAN不需要修改传输层结构，与传统网络设备完美兼容。由此，VXLAN脱颖而出，成为了SDN环境下的主流Overlay技术。

> ???

VxLAN的引入是为了解决VLAN的缺陷：

1. VLAN的数量限制。4096个VLAN远不能满足大规模云计算数据中心的需求。
2. 物理网络基础设施的限制。基于IP子网的区域划分限制了需要二层网络连通性的应用负载的部署。
3. TOR交换机MAC表耗尽。虚拟化以及东西向流量导致更多的MAC表项。
4. 多租户场景。可能导致IP地址重叠。
5. 静态的VLAN TRUNK，并不适合在云计算和VM漫游的环境中使用。



VLAN地址可以重叠吗？？？



我还是不理解，vxlan的场景啊，它底层还是vlan，vlan不就4096个？？？

但是，不同机房vlan可以重复？

vxlan那么多的vni给谁用？

vxlan vni最终映射到vlan id，实现底层物理传输。

好处就是，vm直接可以画更多的vxlan，这样？？？

#### 大二层网络

VXLAN网络保证虚拟机动态迁移：采用“MAC in UDP”的封装方式，保证虚拟机迁移前后的IP和MAC不变。

在Overlay方案中，物理网络的东西向流量类型逐渐由二层向三层转变，通过增加封装，将网络拓扑由物理二层变为逻辑二层，同时提供了逻辑二层的划分管理，更好地满足了多租户的需求。

云计算场景下，传统服务器变成一个个运行在宿主机上的vm。vm是运行在宿主机的内存中，所以可以在不中断的情况下从宿主机A迁移到宿主机B，前提是迁移前后vm的ip和mac地址不能发生变化，这就要求vm处在一个二层网络。**毕竟在三层环境下，不同vlan要使用不同的ip段，否则路由器就犯难了**。

#### MAC表

业务中虚拟机的大规模部署，使二层地址（MAC）表项的大小限制了环境下虚拟机的规模，特别是对于接入设备而言，二层地址表项规格较小，限制了整个数据中心的业务规模。

**通过，vxlan可以减少mac表的压力。VxLAN对广播流量转化为组播流量，可以避免网络本身的无效流量带宽浪费。**

数据中心的虚拟化给网络设备带来的最直接影响就是：之前TOR（Top Of Rack）交换机的一个端口连接一个物理主机对应一个MAC地址，但现在交换机的一个端口虽然还是连接一个物理主机但是可能进而连接几十个甚至上百个虚拟机和相应数量的MAC地址。传统交换机是根据MAC地址表实现二层转发。

MAC地址表是通过交换机的flood-learn学习并记录在交换机的内存。交换机的内存比较宝贵，所以MAC地址表的大小通常是有限的。整个数据中心的MAC地址多了几十倍，那相应的交换机里面的MAC地址表也需要扩大几十倍。如果交换机不支持这么大的MAC地址表，那么就会导致MAC地址表溢出。溢出之后，交换机不能将新的MAC地址学习到自己的MAC地址表。如果交换机收到这些MAC地址的数据帧，因为不能通过查表转发，会flood到所有的端口。这不但增加了交换机的负担，还增加了网络中其他设备的负担。为了避免这个问题，可以用一些更大容量的交换机，但是相应的成本也要上升，而且还不能从根本上解决这个问题。

如果使用VXLAN，虚拟机的Ethernet Frame被VTEP封装在UDP里面，一个VTEP可以被一个物理主机上的所有虚拟机共用。从交换机的角度，交换机看到的是VTEP之间在传递UDP数据。通常，一个物理主机对应一个VTEP，所以交换机的MAC地址表，只需要记录与物理主机数量相当条目就可以了，虚拟化带来的MAC地址表暴增的问题也不存在了。这是VXLAN能解决的，而现有的VLAN没有办法回避的问题。

即，VxLAN通过头端复制或者组播，减少ARP请求和MAC表压力。

#### 资源分配均匀:strawberry:

使用网络Vxlan-Overlay之后，可以构建一个L3之上的L2。这样一个Overlay内的虚机，可以任意的在数据中心分布。实际中云主机肯定不是任意分布的，会有一些主机的调度优化算法，但是至少，网络不会成为限制云主机部署的因素。举个反例，**如果使用VLAN，虚机必须部署在支持相应VLAN的设备上**，（交换机的网卡划分了不同VLAN并连接了不同的服务器）哪怕这个设备已经接近饱和，而其他的设备却是空置的。如下图，因为左边的机架不支持相应的网络，对应的云主机只能往右边的机架塞，直到塞满。而同时，左边的机架负载还不到50%。

只有指定租户网络是vxlan 100，其他下的underlay 网络可以在不同的机房、不同的数据中心甚至不同的城市，只要VTEP建立的vxlan tunnel延时满足要求即可。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/VPC-overlay-pro.png)

Vxlan使得不再受网络硬件的限制，Overlay内的云主机可以部署在整个机房。

#### 多链路使用：？？

这个在bb什么，vxlan的underlay 网络不是还是依赖 单链路的VLAN协议？

突破单条网络链路 VLAN协议使用STP（Spanning Tree Protocol）来管理多条线路，STP根据优先级和cost，只会选出一条线路来工作，这样可以避免数据传递的环路。这种主备（active-passive）的模式，比只连接一条线路肯定是有优势，但是对于用户来说，相当于花了N倍的钱，却只用到了1倍的服务。当网络流量较大的时，也不能通过增加线路来提升性能。而VXLAN因为是通过UDP封装，在三层网络上传输。虽然传递的还是二层的Ethernet Frame，但是VXLAN可以利用一些基于三层的协议来实现多条线路共同工作（active-active），以实现负载均衡，例如ECMP，LACP。现在对于用户来说，花了N倍的钱，也用到了N倍的服务。当网络流量较大时，现在可以通过增加线路来减轻现有线路的负担。这在提升数据中心网络性能，尤其是东西向流量的性能时，尤其重要。这是VXLAN相比VLAN，能带来的另一个好处。

VxLAN 则不然，VxLAN 可以在 L3 层网络上，透明地传输 L2 层数据，这让它可以利用 ECMP (Equal-cost multi-path，等价多路径) 等协议实现多条路径同时工作，也就是 active-active 模式。这样当网络流量较大时，可以实现流量的负载均衡，提升数据传输性能。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/ecmp-vxlan.jpg)

> Network Virtualization

在实际网络中，通过冗余路由来提高网络可用性是普遍的做法，当其中一条路径发生故障时，流量可以切换到其他冗余路径。冗余路由可以分为两种情况，一种是等价路由，一种是非等价路由。等价路由即等价多路径（Equal Cost Multi Path，ECMP），ECMP 的各条路径在互为备份的同时实现了负载分担。非等价路径情况下，只有最优路径被启用作报文转发，次优路径只有当最优路径失效时才会被启用。

##### 路由收敛

当网络链路或设备节点发生故障时，在路由再次收敛前网络流量会发生中断，以现在使用最广泛的OSPF/ISIS协议来说，会经历如下五个过程：探测到故障、产生更新信息、泛洪到整个网络、重新计算路由表以及下刷到FIB表。语音、视频等实时性网络业务的兴起，对IP网络流量的快速倒换也提出了更高的要求，新的架构需要在小于50ms的时间内完成业务的倒换。目前，主流的快速收敛技术：LFA（Loop-Free Alternate，又称 IP FRR，IP Fast Reroute），Remote LFA（Remote Loop-Free Alternate），MRT-FRR（Maximally Redundant Trees – Fast Reroute）等。

##### ECMP 配置实践

https://lk668.github.io/2020/12/13/2020-12-13-ECMP/

### VxLAN: base

#### VxLAN诞生

传统交换网络用 VLAN 来隔离用户和虚拟机，但理论上只支持最多 4K 个 标签的 VLAN，已无法满足需求。为了解决上述局限性，不论是网络设备厂商， 还是虚拟化软件厂商，都提出了一些新的 Overlay 解决方案。

**网络设备厂商**，基于硬件设备开发出了 EVI（Ethernet Virtualization Interconnect）、 TRILL（Transparent Interconnection of Lots of Links)、SPB（Shortest Path Bridging） 等大二层技术。这些技术通过网络边缘设备对流量进行封装/解封装，构造一个 逻辑的二层拓扑，同时对链路充分利用、表项资源分担、多租户等问题采取各自 的解决方法。此类技术一般要求网络边缘设备必须支持相应的协议，优点是硬件 设备表项容量大、转发速度快。 

**虚拟化软件厂商**，从自身出发，提出了 VXLAN（Virtual eXtensible LAN）、 NVGRE（Network Virtualization Using Generic Routing Encapsulation）、STT（A  Stateless Transport Tunneling Protocol for Network Virtualization）等一系列技术。 这部分技术利用主机上的虚拟交换机（vSwitch）作为网络边缘设备，对流量进行 封装/解封装。优点是对网络硬件设备没有过多要求。VXLAN 是由 IETF 定义的 NVO3（Network Virtualization over Layer 3）标准技术之一，采用 MAC-in-UDP 的报文封装模式，可实现二层网络在三层范围内进行扩展，满足数据中心大二层 虚拟机迁移的需求。在 VXLAN 网络中，属于相同 VXLAN 的虚拟机处于同一个 逻辑二层网络，彼此之间二层互通；属于不同 VXLAN 的虚拟机之间二层隔离。 VXLAN 最初只在虚拟交换机实现，但虚拟交换机天然具有转发性能低下的缺点， 并不适合大流量的网络环境。

后续各硬件厂商也纷纷推出支持 VXLAN 的硬件 产品，与虚拟交换机一起，共同成为网络边缘设备，最终使 VXLAN 技术能够适应各种网络。

#### VxLAN网络模型

VXLAN 网络架构的组件有：

- 实体设备：VM、NVE和transit设备。
- 逻辑设备：VNI、VTEP、VAP、VXLAN隧道

##### 实体组件

1. **VM（Virtual Machine，虚拟机）**：在一台服务器上可以创建多台虚拟机，不同的虚拟机可以属于不同的VXLAN。属于相同VXLAN 的虚拟机处于同一个逻辑二层网络，彼此之间二层互通；属于不同VXLAN 的虚拟机之间二层隔离。

2. **NVE：NVE（Network Virtualization Edge）**：VXLAN网络虚拟边缘节点NVE，是实现网络虚拟化功能的**网络实体**。报文经过NVE封装转换后，NVE间就可基于三层基础网络建立二层虚拟化网络。**硬件设备、vSwitch或者软OVS都可以作为NVE。**

   NVE是参与underlay network的组网

   > 典型的Openvswitch

3. **transit设备/核心设备**：transit设备不参与VXLAN处理，仅需要根据封装后报文的目的IP地址对报文进行三层转发。

##### 逻辑组件:star2:

1. **VNI(VXLAN Network Identitifier，VXLAN标示符)**：VXLAN 通过VXLAN ID 来标识，其长度为24 比特，VNI是一种类似于VLAN ID的用户标识，一个VNI代表一个租户，属于不同VNI的虚机之间不能直接进行二层通信。

2. **VTEP（VXLAN Tunnel End Point，VXLAN 隧道端点）**：VTEP是VXLAN隧道端点，封装在NVE中，用于VXLAN报文的封装和解封装。**VTEP与物理网络相连，分配有物理网络的IP地址，该地址与虚拟网络无关。说白了，就是VXLAN隧道两端网络可达的IP地址。**VXLAN报文中源IP地址为本节点的VTEP地址，VXLAN报文中目的IP地址为对端节点的VTEP地址，一对VTEP地址就对应着一个VXLAN隧道。

   VTEP设备是VxLAN L2 GW设备完成vlan-vxlan的映射，进而实现under和Overlay的转换。

3. **VAP（Virtual Access Point）：虚拟接入点VAP统一为二层子接口**，用于接入数据报文。**为二层子接口配置不同的流封装**，可实现不同的数据报文接入不同的二层子接口。

4. **VXLAN 隧道**：两个VTEP 之间的点到点逻辑隧道。VTEP 为数据帧封装VXLAN 头、UDP 头和IP 头后，通过VXLAN 隧道将封装后的报文转发给远端VTEP，远端VTEP 对其进行解封装。

5. **Bridge Domain**：同一大二层域，就类似于传统网络中VLAN（虚拟局域网）的概念，只不过在VXLAN网络中，它有另外一个名字，叫做Bridge-Domain，简称BD。不同的BD通过VNI就行区分。通常BD与VNI是1：1的映射关系，这种映射关系是通过在VTEP设备上配置命令行建立起来的

   ```bash
   bridge-domain 10   //表示创建一个“大二层广播域”BD，其编号为10
   vxlan vni 5000  //表示在BD 10下，指定与之关联的VNI为5000
   ```

6. **VSI**：Virtual Switch Instance，虚拟交换实例，VTEP 上为一个 VXLAN 提供二层交换服务的虚拟交换实例。VXLAN ID 和VSI 是一对一的关系，所以每增加一个VXLAN ID的二层网络都要有唯一的VSI实例与之对应。

#### 控制平面 control plane：MP-BGP EVPN

 VXLAN flood-and-learn 模型中，**end-host（终端主机学习**和 **VTEP 发现**都是基于数据平面，并没有控制平面来在 VTEP 之间分发 end-host 可达性 信息。

所谓控制层面，就是路由数据怎么传递的。基于MP-BGP EVPN可以实现控制平面的控制

例如，CE2里面是怎么就有了CE1网络的路由，并且能正常工作。

#### 多租户

作为 MP-BGP 的扩展，MP-BGP EVPN 继承了 VPN 通过 VRF 实现的对多租户的支持。

在 MP-BGP EVPN 中，多个租户可以共存，它们共享同一个 IP 传输网络（underlay），而 在 VXLAN overlay 网络中拥有独立的 VPN。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/19-mp-bgp-evpn-multi-tenancy.png)

在 VXLAN MP-BGP EVPN spine-and-leaf 网络中，**VNI 定义了二层域**，不允许 L2 流量跨越 VNI 边界。类似地，**VXLAN 租户的三层 segment 是通过 VRF 技术隔离的**，通过将不同 VNI 映射到不同的 VRF 实例来隔离租户的三层网络。每个租户都有自己的 VRF 路由实例。同一 租户的 VNI 的不同子网，都属于同一个 L3 VRF 实例，这个 VRF 将租户的三层路由与其他 租户的路由隔离开来。

#### 实现VxLAN-Overlay报文模式

若要实现网络 Overlay，首先边缘的网络交换机必须能承担 VTEP 的角色，支持 VxLAN 特性，这些 VTEP 设备就是 VxLAN L2 GW 设备。VxLAN L2 GW 最关键的技术就是能完成 VLAN 与 VxLAN的映射。所谓 VLAN 与 VxLAN 的映射，指的是这些承担 VxLAN L2 GW 角色的交换机的 AC 口下接入的虚机或者服务器发出的某个 VLAN 的报文进入到的 VxLAN L2 GW 设备后，该设备会通过 VLAN与 VxLAN 的映射，将这个 VLAN 报文封装成对应的 VxLAN 报文，进而完成后续 VxLAN 报文的传递。以 H3C 设备为例讲解具体的实现。VxLAN L2 GW 角色接入交换机分为两种接入模式。Ethernet 模式和 VLAN 模式。应对不同的组网可以使用不同的接入模式，灵活配置。

##### Ethernet 模式

**Ethernet 模式的核心思想是不改变原始报文封装**。如果 AC 口收到的报文携带了 VLAN tag，那么 VxLAN L2 GW 交换机根据收到的 VLAN tag 信息，查找 VLAN 与 VxLAN 的映射关系，将这个VLAN 映射到指定的 VxLAN。在做外层 VxLAN 封装的时候将原始报文的 VLAN tag 一并携带封装在报文里面。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/Overlay-ethernet.png)

当然，如果上来的报文不带 VLAN tag，则只根据 VLAN PVID 所配置的 VLAN ID 映射到对应的 VxLAN 网络中。在做外层 VxLAN 封装的时候不会把 PVID 的 VLAN ID 信息封装在原始报文里面。

##### VLAN 模式

**VLAN 模式的核心思想是去除原始报文 VLAN tag 后再进行封装**。具体来说，如果 VxLAN L2 GW 交换机收到的报文是携带了 VLAN tag 的，那么 VxLAN L2 GW 交换机根据 AC 收到的报文中携带的 tag 信息查找 VLAN 与 VXALN 的映射关系，然后映射到相应的 VxLAN 中去。在做外层VxLAN 封装时会去掉原始报文的 VLAN tag，即不会将原始报文的 VLAN tag 携带封装在报文里面。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/overlay-vlan.png)

当然，如果 AC 口收到的报文是没有携带 VLAN tag 的，则只根据 VLAN PVID 所配置的 VLAN映射到对应的 VxLAN 网络中。在做外层 VxLAN 封装的时候将不会 PVID 的 VLAN tag 封装在原始报文报文里面。

不管是 Ethernet 模式还是 VLAN 模式，其最终的实现效果都是完成 VLAN 与 VxLAN 的映射。当 VxLAN L2 GW 交换机 AC 口下的虚机或者服务器发来的报文封装成了对应的 VxLAN 报文后，后面就是大家熟悉的 VxLAN 报文的处理流程了。

end

#### BUM报文处理:white_flower:

针对已知单播报文的数据转发流程为：

虚拟机发出的原始IP报文，到达TOR设备后增加VXLAN报头，并在VXLAN隧道进行转发，到达目标TOR设备解除VXLAN封装后得到原始IP报文，最后到达目标虚拟机。

如果涉及BUM（Broadcast, Unknown-unicast, Multicast）报文该如何处理？

即vm_1 不知道 vm_2的mac信息，需要发起arp 广播，这种时候怎么办呢~

##### 头端负责

假设VM1（所属VNI10099）需要与VM2(所属VNI10099)进行通信。在VM1上进行原始数据报文封装时缺少目标VM2的mac地址信息，于是VM1将向所属局域网发送ARP请求（目标MAC为全F）。TOR1在接收到VM1所发送的ARP请求后。TOR1根据头端复制列表将其转为单播方式，将ARP请求报文复制后并增加VXLAN报头向TOR3以单播方式进行发送，这就是对BUM帧简单的头端复制过程。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vxlan-头端负责.png)

当VTEP收到BUM（Broadcast&Unknown-unicast&Multicast，广播&未知单播&组播）报文时，会将报文复制并发送给Peer List中所列的所有对端VTEP（这就好比广播报文在VLAN内广播）。因此，这张表也被称为“头端复制列表”。当VTEP收到已知单播报文时，会根据VTEP上的MAC表来确定报文要从哪条VXLAN隧道走。

在传统局域中ARP请求通过广播方式在局域网内泛洪，占用局域网内大量带宽资源。但在云数据中心里，大二层局域网是通过VXLAN隧道构建，存在海量虚拟机，大量ARP广播报文通过VXLAN隧道进行广播，将极大的增加云数据中心TOR设备压力，消耗TOR设备链路及自身资源。

##### 组播方式

与”头端复制“实现方式不同，组播方式实现需要依赖PIM协议构建的组播树，并将相应VXLAN与组播组进行绑定，形成VXLAN报文转发路径

在组网中增加单独RP，使用S65系列（version S6500_RGOS 12.5(1)B0401S1）作为TOR设备。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vxlan-组播bum.png)

如所示，VM1与VM2均属于VNI 10099，且将VNI 10099加入组播组226.2.1.1后，设备将生成VNI与组播组绑定表项，如：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vxlan-multicast.png)

假设VM1需要与VM2进行通信。VM1依旧向所在局域网发送ARP广播请求（目标MAC为全F）。TOR1在接收到VM1所发送的ARP请求后，在原始ARP请求报文增加VXLAN封装后，根据已形成的组播路由表项，将ARP请求报文以组播复制方式转发到TOR2。TOR2最终将VXLAN报头解封装后，将原始ARP报文送到VM2。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vxlan-multicast-message.png)

以组播方式进行BUM报文效率较高，但进行VXLAN扩展时会带来更大挑战。例如，数据中心中大量不同VXLAN配置在同一组播组，将导致部分TOR设备接收大量自身并不感兴趣的BUM流量。但假如所有不同的VXLAN配置不同的播组，那么每台TOR将需要维护大量组播树与组播路由表项，这也会让TOR设备感觉到压力山大。



#### VxLAN数据流类型

在交换机中通过`encapsulation`命令设置接口封装类型

```bash
[Spine-1]interface GE1/0/0.1 mode l2
[Spine-1-GE1/0/0.1]encapsulation untag
[Spine-1-GE1/0/0.1] bridge-domain 10    
[Spine-1-GE1/0/0.1] q 
```



| 流封装类型 | 允许进入VXLAN隧道的报文类型                                  | 报文进行封装前的处理                                         | 收到VXLAN报文并解封装后                                      |
| ---------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Dot1q      | 只允许携**带指定VLAN Tag**的报文进入VXLAN隧道。 （这里的“指定VLAN Tag”是通过命令进行配置的） | 进行VXLAN封装前： 先剥掉原始报文的外层VLAN Tag。             | 进行VXLAN解封装后： 若内层原始报文带有VLAN Tag，则先将该VLAN Tag替换为指定的VLAN Tag，再转发； 若内层原始报文不带VLAN Tag，则先将其添加指定的VLAN Tag，再转发。 |
| untag      | 只允许**不携带VLAN Tag**的报文进入VXLAN隧道。                | 进行VXLAN封装前： 不对原始报文处理，即不添加/不替换/不剥掉任何VLAN Tag。 | 进行VXLAN解封装后：不对原始报文处理，即不添加/不替换/不剥掉任何VLAN Tag。 |
| default    | 允许**所有报文**进入VXLAN隧道，不论报文是否携带VLAN Tag。    | 进行VXLAN封装前： 不对原始报文处理，即不添加/不替换/不剥掉任何VLAN Tag。 | 进行VXLAN解封装后：不对原始报文处理，即不添加/不替换/不剥掉任何VLAN tag |

**注意：VXLAN隧道两端二层子接口的配置并不一定是完全对等的。正因为这样，才可能实现属于同一网段但是不同VLAN的两个VM通过VXLAN隧道进行通信**

场景一个Leaf下面，接的VM 属于不同的VxLAN：VM_1 属于VLAN10，VM_2属于VLAN20(交换的PVID是20，VM_2的报文是不带vlan tag的，有pvid赋值)，VM_3和VM_4都属于VLAN30

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vxlan-data-type.jpg)

基于二层物理接口10GE 1/0/1，分别创建二层子接口10GE 1/0/1.1和10GE 1/0/1.2，且分别配置其流封装类型为dot1q和untag。配置如下：

```bash
#
interface 10GE1/0/1.1 mode l2   //创建二层子接口10GE1/0/1.1
 encapsulation dot1q vid 10   //只允许携带VLAN Tag 10的报文进入VXLAN隧道
 bridge-domain 10   //报文进入的是BD 10，子接口和BD的绑定。
#
interface 10GE1/0/1.2 mode l2   //创建二层子接口10GE1/0/1.2
 encapsulation untag   //只允许不携带VLAN Tag的报文进入VXLAN隧道
 bridge-domain 20   //报文进入的是BD 20
#
基于二层物理接口10GE 1/0/2，创建二层子接口10GE 1/0/2.1，且流封装类型为default。配置如下：
#
interface 10GE1/0/2.1 mode l2   //创建二层子接口10GE1/0/2.1
 encapsulation default   //允许所有报文进入VXLAN隧道
 bridge-domain 30   //报文进入的是BD 30
#
```

此时你可能会有这样的疑问，为什么要在10GE 1/0/1上创建两个不同类型的子接口？是否还可以继续在10GE 1/0/1上创建一个default类型的二层子接口？

**Q1：我们先来解答下是否可以在10GE 1/0/1上再创建一个default类型的二层子接口。**

A1：答案是不可以。其实根据表3-1的描述，这一点很容易理解。因为default类型的二层子接口允许所有报文进入VXLAN隧道，而dot1q和untag类型的二层子接口只允许某一类报文进入VXLAN隧道。这就决定了，default类型的二层子接口跟其他两种类型的二层子接口是不可以在同一物理接口上共存的。否则，报文到了接口之后如何判断要进入哪个二层子接口呢。所以，default类型的子接口，一般应用在经过此接口的报文均需要走同一条VXLAN隧道的场景，即下挂的VM全部属于同一BD。例如，图3-3中VM3和VM4均属于BD 30，则10GE 1/0/2上就可以创建default类型的二层子接口。

**Q2：再来看下为什么可以在10GE 1/0/1上分别创建dot1q和untag类型的二层子接口。**

A2：如图所示，VM1和VM2分别属于VLAN 10和VLAN 20，且分别属于不同的大二层域BD 10和BD 20，显然他们发出的报文要进入不同的VXLAN隧道。如果VM1和VM2发出的报文在到达VTEP的10GE 1/0/1接口时，一个是携带VLAN 10的Tag的，一个是不携带VLAN Tag的（比如二层交换机上行连接VTEP的接口上配置的接口类型是Trunk，允许通过的VLAN为10和20，PVID为VLAN 20），则为了区分两种报文，就必须要在10GE 1/0/1上分别创建dot1q和untag类型的二层子接口。所以，当经过同一物理接口的报文既有带VLAN Tag的，又有不带VLAN Tag的，并且他们各自要进入不同的VXLAN隧道，则可以在该物理接口上同时创建dot1q和untag类型的二层子接口。



#### 交换机透传vxlan报文：？

现网部分用户会自己实现VXLAN转发，CE交换机作为中间节点设备，需要具备透传VXLAN报文的能力。默认情况下，当CE交换机本身已经开启了VXLAN功能，其他VXLAN报文的透传可能受到影响，需要做针对性的配置。相关场景分析如下：

1、 交换机二层端口的透传：

1.1、    如果此透传报文携带的VLAN TAG已经开启了VXLAN映射（VLAN绑定VNI），那么此场景下，需要在对应端口配置port nvo3 mode access，保证此报文的透传。

1.2、    如果此报文携带的VLAN TAG未开启VXLAN映射，那么次场景下，端口无需做配置，设备可以直接对报文进行相应的二三层转发。

2、 交换机三层端口的透传：

对于三层端口，CE交换机默认会对VXLAN报文进行解封装，因此默认情况会出现报文解封装后查不到下一步的转发表而在交换机内部丢弃；为避免此种情况出现，同样需要在对应的三层接口下配置port nvo3 mode access，保证报文的正常透传。

### VxLAN协议没有控制平面

#### flood-learn

如果VTEP L2 Table中没有找到对应的Remote VTEP，那就要通过flood-learn来获得对端的VTEP。
![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/VXLAB-flood-learn.png)
为了更好的描述flood-learn，我们假设最左侧虚机已经知道目的MAC了（VTEP中的L2 Table已经老化，虚机中的ARP cache还没老化)。当最左侧虚机想ping最右侧虚机，ping包送到VTEP，因为在VTEP中找不到对应的Remote VTEP，VTEP会做如下操作：

- 原始的Ethernet Frame被封装成VXLAN格式，VXLAN包的外层目的IP地址为组播地址。
- VXLAN数据包被发送给组播内所有其他VTEP。

其实这就是flood过程。因为组播内所有VTEP都是接收方，最右侧虚机可以受到组播的ping包。最右侧的VTEP首先从ping包中学习到了最左侧虚机的MAC地址，VXLAN ID和对应的VTEP。因为有了这些信息，当最右侧虚机返回时，会直接发送到最左侧VTEP。这样最左侧VTEP也能从返回包中学习到最右侧虚机的MAC地址，VXLAN ID和对应的VTEP，并记录在自己的L2 Table中，这就是learn过程。与交换机中的flood-learn不一样的是，交换机中记录的是对应的交换机端口和MAC的关系，这里记录的是Remote VTEP(IP Address)和MAC的关系。

下次最左侧虚机想访问最右侧虚机，不需要再flood，直接查VTEP L2 Table就能找到对应的remote VTEP。

所以从这里看出，VXLAN的转发信息，也是通过数据层的flood-learn获取，VXLAN不需要一个控制层也能工作，这与VPLS的情况很像啊！

#### EVPN control plane

EVPN，用作Overlay网络，例如VXLAN网络的控制层。VXLAN网络配合EVPN作为控制层，不仅能减少网络中的广播与组播数量，而且能带来EVPN作为控制层的一些优点。在一个VXLAN Fabric架构中，采用EVPN做控制层，借助功能完善的BGP（确切说是MP-BGP）协议，能够高效的连接不同的POD，甚至连接不同的site。所以从这个角度来说，EVPN作为VXLAN的控制层的应用，并不逊色与其作为L2 VPN的应用。

VXLAN由RFC7348定义，在RFC中，只定义了数据层的行为，并没有指定VXLAN控制层。在VXLAN技术早期，通过数据层的来获取转发信息，在实现上较为简单，相应的技术门槛较低，有利于厂商实现VXLAN。但是随着网络规模的发展，完全依赖数据层做控制会造成网络中广播组播风暴，因此VXLAN也需要有一个控制层。

SDN controller也可以作为VXLAN的控制层，OpenStack中普遍用SDN controller控制OpenVSwitch，VTEP直接通过OVSDB和OpenFlow流表进行管理。这些内容很有意思，功能也很强大，不过跟讨论vxlan control plane是两件事情，这里只讨论EVPN作为控制层的情况。

EVPN作为NVO的控制层由IETF草案：draft-ietf-bess-evpn-overlay定义。EVPN的实现是参考了BGP/MPLS L3 VPN，那么EVPN作为VXLAN的控制层时，仍然采用相同的架构，只是架构的组成元素发生了改变。
![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/VXLAN-control-plane.png)
具体的变换包括了：

- PE设备变成了VTEP，有的时候也称为NVE（Network Virtualization Endpoint）。相应的MP-BGP连接也建立在VTEP之间。
- 数据层变成了VXLAN，VXLAN是在Underlay网络传输。
- CE设备变成了Server，这里可以是Virtual Server也可以是Physical Server

### VLAN-VxLAN映射:tiger:

若要实现网络 Overlay，首先边缘的网络交换机必须能承担 VTEP 的角色，支持 VxLAN 特性，这些 VTEP 设备就是 VxLAN L2 GW 设备。VxLAN L2 GW 最关键的技术就是能完成 VLAN 与 VxLAN的映射。所谓 VLAN 与 VxLAN 的映射，指的是这些承担 VxLAN L2 GW 角色的交换机的 AC 口下接入的虚机或者服务器发出的某个 VLAN 的报文进入到的 VxLAN L2 GW 设备后，该设备会通过 VLAN与 VxLAN 的映射，将这个 VLAN 报文封装成对应的 VxLAN 报文，进而完成后续 VxLAN 报文的传递。

以太网服务实例与指定 VXLAN 对应的 VSI 关联，从而确保属于该 VLAN 的数据帧均通过指定的 VSI 转发

如果是SDN控制器，在承载网网络 vlan-vxlan映射配置页面配置映射表，并关联到交换机的下行口。对应交换机的下行接入口，是根据报文的VLAN ID值或接口的PVID（针对报文没带TAG的情况），映射到对应的VXLAN转发实例。

#### 设备级映射：未突破4K

将VLAN-VXLAN 的映射关系配置到通过设备映射。

Leaf 1 只需要配置支持哪些VLAN，它们直接的互通配置有SDN控制器直接下发处理了，减少人工配置。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vlan-vxlan-switch.png)

通过SDN控制器将映射表直接应用到设备上会有一些缺点，比如说 

1）单台设备只能维护一张映射表，交换机端口浪费。

2）在这台设备下接入的VLAN网络对应的 VXLAN网络就是固定的。

3）如果因业务规划需要改变这种映射关系，就得使用另一台 设备来做，操作不灵活，同时会导致交换机端口浪费的现象。

#### 端口级映射：未突破4K

SND控制器将 VLAN-VXLAN 的映射关系配置到通过接口映射。可以将 VLAN-VXLAN 的映射关系配置到 Ten-G2/0/15 或 Ten-G2/0/18 接口上。，即利用一个互联接口就可以满足不同的 VLANVXLAN 的映射关系，可以维护多张 VLAN-VXLAN 的映射表，实现了充分利用 的效果。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vlan-vxlan-switch-port.png)



#### 层次化绑定: 突破4K

层次化端口映射 主要是采用分层思想，即上层网络的静态 VxLAN ID 保持不变，下层网络不同计算节点动态分配 VLAN ID，不用在SDN控制器上创建 VLAN-XLAN 映射关系。

如下，在云平台(比如，openstack)的计算网络节点配置：

```bash
[ml2]
type_drivers = vxlan, flat, gre, vlan
tenant_network_types = vxlan
mechanism_drivers = openvswitch, l2population
extension_drivers = port_security
[ml2_type_flat]
[ml2_type_vlan]
[ml2_type_vxlan]
vni_ranges = 100:1000
[ml2_type_gre]
```

即，上层指定静态的VxLAN VNI，下次动态分配 VLAN ID，如下两个server上的vm的vlan ID相同但是长层的VxLAN ID却不同。如下，云平台指定了网络为vxlan 100 和 vxlan 300的租户网络，在其下创建vm时，vm在host上由br-int动态分配vlan ID

同理还有这种情况：上层vni都是100，映射到host上的vlan id可能不一样，一个为vlan 20，另一个为vlan 50.



![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vlan-vxlan-Hierarchical.png)



#### 总结

设备和端口级别的映射，可以减少VLAN互通的配置，也不赖。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vlan-vxlan-ref.png)



### VxLAN MTU

为了保证VXLAN机制通信过程的正确性，rfc7348标准中规定，涉及到VXLAN通信的IP报文一律不允许分片，这就要求物理网络的链路层实现中必须提供足够大的MTU值，保证VXLAN报文的顺利传输，这一点可以理解为当前VXLAN技术的局限性。

　　一般来说，虚拟机的默认MTU为1500 Bytes，也就是说原始以太网报文最大为1500字节。这个报文在经过VTEP时，会封装上50字节的新报文头（VXLAN头8字节+UDP头8字节+外部IP头20字节+外部MAC头14字节），这样一来，整个报文长度达到了1550字节。

　　而现有的VTEP设备，一般在解封装VXLAN报文时，要求VXLAN报文不能被分片，否则无法正确解封装。这就要求VTEP之间的所有网络设备的MTU最小为 1550字节。

　　如果中间设备的MTU值不方便进行更改，那么设置虚拟机的MTU值为1450，也可以暂时解决这个问题。



### 创建vxlan vni

华三使用vsi 10 

```bash
$ system-view
$ l2vpn enable
# 创建vsi并进入vsi视图
$ vsi vsi-name
$ vxlan vxlan-id
# 配置vxlan与vxlan隧道关联
$ tunnel tunnel-number
```

华为使用bridge-domain 10

1. 执行命令**system-view**，进入系统视图。

2. 执行命令**bridge-domain** *bd-id*，进入BD视图。

3. 执行命令**vxlan vni** *vni-id*，配置BD所对应的VXLAN的VNI。

   缺省情况下，没有配置BD所对应的VXLAN的VNI。

4. 执行命令**quit**，退出BD视图，返回到系统视图。

5. 执行命令**interface nve** *nve-number*，创建NVE接口，并进入NVE接口视图。

6. 执行命令**source** *ip-address*，配置VXLAN隧道源端VTEP的IP地址。

   缺省情况下，VXLAN隧道源端VTEP没有配置IP地址。

7. 执行命令**vni** *vni-id* **head-end peer-list** *ip-address* &<1-10>，配置头端复制列表。

   缺省情况下，没有配置VNI头端复制列表。

   头端是指VXLAN隧道的入节点，复制是指当VXLAN隧道的入节点收到一份BUM报文后，需要将其复制多份并发送给列表中的所有VTEP。头端复制列表就是用于指导VXLAN隧道的入节点进行BUM报文复制和发送的远端VTEP的IP地址列表。

8. 执行命令**quit**，退出NVE接口视图，返回到系统视图

end



### VxLAN网关:fire:



#### 二层网关

用于解决租户接入VXLAN虚拟网络的问题，也可用于同一VXLAN虚拟网络的子网通信。

VxLAN L2 GW 最关键的技术就是能完成 VLAN 与 VxLAN的映射。所谓 VLAN 与 VxLAN 的映射，指的是这些承担 VxLAN L2 GW 角色的交换机的 AC 口下接入的虚机或者服务器发出的某个 VLAN 的报文进入到的 VxLAN L2 GW 设备后，该设备会通过 VLAN与 VxLAN 的映射，将这个 VLAN 报文封装成对应的 VxLAN 报文，进而完成后续 VxLAN 报文的传递。

```bash
# 在Switch B上创建VXLAN 10和VXLAN 20。
[SwitchB] l2vpn enable
[SwitchB] vsi vpna
[SwitchB-vsi-vpna] vxlan 10
[SwitchB-vsi-vpna-vxlan10] quit
[SwitchB-vsi-vpna] quit

# 在Switch B上将VXLAN隧道Tunnel1与VXLAN 10关联。
[SwitchB] vsi vpna
[SwitchB-vsi-vpna] vxlan 10
[SwitchB-vsi-vpna-vxlan10] tunnel 1
[SwitchB-vsi-vpna-vxlan10] tunnel 2
[SwitchB-vsi-vpna-vxlan10] quit
[SwitchB-vsi-vpna] quit

# 在接口GigabitEthernet1/0/1上创建以太网服务实例1000，该实例用来匹配VLAN 12（VM 3）的数据帧，将该服务实例与vpna（VXLAN 10）关联。

[SwitchB] interface gigabitethernet 1/0/1
[SwitchB-GigabitEthernet1/0/1] service-instance 1000
[SwitchB-GigabitEthernet1/0/1-srv1000] encapsulation s-vid 12
[SwitchB-GigabitEthernet1/0/1-srv1000] xconnect vsi vpna
[SwitchB-GigabitEthernet1/0/1-srv1000] quit
```

end

#### 三层网关

用于VXLAN虚拟网络的跨子网通信以及**外部网络的访问**

虚拟机要想与外部网络进行三层通信，需要在接入虚拟机的本地分布式VXLAN IP网关上指定流量的下一跳为Border，可以通过如下方式来实现：

* 在本地分布式VXLANIP网关上配置静态路由，指定路由下一跳为Border上同一个VXLAN对应VSI虚接口的IP地址。
*  在本地分布式VXLANIP网关上配置策略路由，设置报文的下一跳为Border上同一个VXLAN对应VSI虚接口的IP地址。

```bash
# 创建VSI虚接口VSI-interface1，为其配置IP地址，该IP地址作为VXLAN 10内虚拟机的网关地址，指定该接口的MAC地址，并配置该接口定时发送免费ARP报文。

[SwitchD] interface vsi-interface 1
[SwitchD-Vsi-interface1] ip address 10.1.1.1 255.255.255.0
[SwitchD-Vsi-interface1] mac-address 1-1-1
[SwitchD-Vsi-interface1] arp send-gratuitous-arp interval 200000
[SwitchD-Vsi-interface1] quit

# 配置VXLAN 10所在的VSI实例与接口VSI-interface1关联。

[SwitchD] vsi vpna
[SwitchD-vsi-vpna] gateway vsi-interface 1
[SwitchD-vsi-vpna] quit

# 创建VSI虚接口VSI-interface2，为其配置IP地址，该IP地址作为VXLAN 20内虚拟机的网关地址，指定该接口的MAC地址，并配置该接口定时发送免费ARP报文。

[SwitchD] interface vsi-interface 2
[SwitchD-Vsi-interface2] ip address 10.1.2.1 255.255.255.0
[SwitchD-Vsi-interface2] mac-address 2-2-2
[SwitchD-Vsi-interface2] arp send-gratuitous-arp interval 200000
[SwitchD-Vsi-interface2] quit

# 配置VXLAN 20所在的VSI实例与接口VSI-interface2关联。

[SwitchD] vsi vpnb
[SwitchD-vsi-vpnb] gateway vsi-interface 2
[SwitchD-vsi-vpnb] quit

# 配置OSPF发布VSI虚接口、Vlan-interface100接口所在网段的路由。
# 或者简单点只写 静态路由
[SwitchD] ospf 2 router-id 4.4.4.4
[SwitchD-ospf-2] area 0
[SwitchD-ospf-2-area-0.0.0.0] network 10.1.1.0 0.0.0.255
[SwitchD-ospf-2-area-0.0.0.0] network 10.1.2.0 0.0.0.255
[SwitchD-ospf-2-area-0.0.0.0] network 100.1.1.0 0.0.0.255
[SwitchD-ospf-2-area-0.0.0.0] quit
[SwitchD-ospf-2] quit
```

end

### VxLAN外部连接方式

VRF： Virtual Routing and Forwarding

VXLAN通过边界节点和外部连接，边界节点负责对VXLAN进行解封装，和外部交换路由，边界节点有两种选择。

L2 and L3 gateway

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vxlan&&non-vxlan.png)

#### Spine作为边界：L3

使用spine作为边界节点。在使用spine作为边界节点时，为了使spine上路由的一致性，建议所有的spine都和外部连接。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vxaln-spine-border.png)



#### Leaf作为边界：L2

使用专门的leaf作为边界节点。这种情况下，对于南北流量，增加了额外的一跳；但也使得spine不用承担VTEP功能，减轻VTEP负担。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vxlan-leaf-border.png)

### VxLAN 3层外部连接

VXLAN边界节点对出方向的VXLAN流量进行解封装，同时保持数据包中关于VRF,或租户信息。可使用的方法包括VRF Lite , LISP, MPLS L3 VPN。

#### 全互联模型和U型模型

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vxlan-L3-mesh.png)

该模型中，每个边界节点和所有外部设备互联，保证网络具有足够的弹性；边界节点之间不需要线互联，直接和外部交换路由，无需在两台边界设备之间进行路由同步。全互联模型不会产生流量黑洞问题。
另一种可选的模型时U形连接。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vxlan-L3-U.png)

该模型中，每台边界节点和一台外部设备相连，两台边界节点相连

#### VRF Lite/Inter-AS Option A

在确定了可选的连接方式后，VXLAN外部连接需要考虑的时，VXLAN内部和外部之间如何传递路由。最简单的两个方式就是VRF Lite 或Inter-AS Option A 。
如图：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vxlan-L3-VRF.png)



实现方式如下：

* 使用option A的方式，在edge router 上为每个用户创建VRF,对应border leaf上的客户。
* border 和edge router 之间路由交换可以在vrf内运行路由协议，可以使用静态，或动态的路由协议。最简便的是使用eBGP路由协议，在border leaf上就不需要进行重分布；
* 如果使用其他路由协议，在将IBGP重分布到IGP时，需要进行过滤；只需要将IBGP 属性为internal 的路由通告到IGP中，同时进行路由汇总，尽量避免/32路由通告到IGP中。
* 在border leaf 和edge router 可以部署BFD加快故障检测和倒换

#### MPLS L3VPN:star2:

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vxlan-mpls-l3vpn.png)

该方式下，将border leaf当作MPLS PE，将eVNP路由重新生成L3VNP的路由，通过BGP eVNP传递给远端的PE，在通告给远端时，需要抑制主机路由；从远端PE收到VNPV4路由时，将VNPV4路由重新生成L2VNP路由。

### VxLAN 2层外部连接

二层连接使用以下拓扑，外部网络通过vPC连接到border-leaf。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vxlan-l2-gw.png)

由于BPDU报文不会通过VXLAN传播，且VTEP之间是二层互通，因此避免在让VXLAN fabric参与STP的计算，部署时，将VTEP当作STP的边缘设备。

图中，VPC应该表示一个数据中心中包含基础服务的vpc，如DNS、DHCP

### VxLAN控制平面

https://www.it8045.com/2103.html

VxLAN 的控制平面主要解决的是 VTEP 之间的 vlxan 隧道如何建立、虚拟机之间的报文通过哪条 VxLAN 隧道进行移动的问题。

VxLAN 的控制平面主要有种模式：松散控制模式与集中控制模式

任何数据的转发都用转发和控制平面，其中控制平面在于建立目标与相关逻辑标识的对应关系，在 VXLAN 中，如何知道目的 MAC 地址和对方 VTEP IP 的对应关系，是控制平面所需要完成的工作。在 VXLAN 中，如果没有引入第三 方独立的控制设备（在 SDN 网络环境中，由于控制和转发分离，控制平面可以 独立出来），则需要采用组播或头端复制的方式实现控制信息的传递。

#### 松散控制模式：基于硬件

在 VxLAN 的松散控制模式下，VxLAN 的控制平面与 VCFC 控制器解耦和，只能由全硬件交换机来实现 VxLAN VTEP、VxLAN GW、VxLAN IP GW。VTEP 之间如何建立隧道，流量在哪个隧道中进行转发由硬件的核心和接入设备自行决定，**具体的实现方式有自学习、EVPN、扩展 ISIS 的方式**。

在松散控制模式下，数据平面只用支持二三层的转发就可以了。而 SDN控制器也只用负责下发服务策略和配置，并不用下发控制流表。

#### 集中控制模式：基于SDN

在 VxLAN 的集中控制模式下，VxLAN 的控制平面是基于 SDN控制器的，可采用软硬件结合的 VxLAN 网络方案，即 VTEP 设备可以由硬件或软件交换机来担任。VTEP 之间如何建立隧道，流量在哪个隧道中进行转发由控制器决定。控制器通过通过 OpenFlow、Netconf、OVSDB 来对软硬件网元进行控制。数据平面支持二三层的转发，SDN控制器则同时需要负责下发配置、服务策略和控制流表。

### VxLAN隧道

Nested between Layer 2 and Layer 3 (L2), VXLAN allows users to overlay Layer 2 (L2) networks over Layer 3 (L3).

#### 隧道转发模式

缺省情况下，VXLAN隧道工作在三层转发模式

当设备作为VTEP时，需要配置VXLAN隧道工作在二层转发模式；

当设备作为VXLAN IP网关时，需要配置VXLAN隧道工作在三层转发模式。

```bash
# 配置VXLAN隧道工作在二层转发模式
undo vxlan ip-forwarding

# 配置VXLAN隧道工作在三层转发模式
vxlan ip-forwarding
```

end

#### 子接口模式

```bash
#  if-1.1
interface 10GE1/0/1.1 mode l2   //创建二层子接口10GE1/0/1.1   
encapsulation dot1q vid 10   //只允许携带VLAN Tag 10的报文进入VXLAN隧道   
bridge-domain 10   //报文进入的是BD 10  

#  if-1.2
interface 10GE1/0/1.2 mode  l2   //创建二层子接口10GE1/0/1.2   
encapsulation untag   //只允许不携带VLAN Tag的报文进入VXLAN隧道  
bridge-domain 20   //报文进入的是BD 20  

```

设置好子接口后，通过协议自动建立vxlan隧道隧道，或者手动指定vxlan隧道的源和目的ip地址在本端vtep和对端vtep之间建立静态vxlan隧道。对于华为CE系列交换机，以上配置是在nve（network virtualization Edge）接口下完成的。配置过程如下

```bash
interface Nve1   //创建逻辑接口  
NVE 1 source 1.1.1.1   //配置源VTEP的IP地址（推荐使用Loopback接口的IP地址）   
vni 5000 head-end peer-list 2.2.2.2    
vni 5000 head-end peer-list 2.2.2.3 
```

其中，vni 5000的对端vtep有两个，ip地址分别为2.2.2.2和2.2.2.3，至此，vxlan隧道建立完成。
VXLAN隧道两端二层子接口的配置并不一定是完全对等的

#### vlan模式

当有众多个vni的时候，需要为每一个vni创建一个子接口，会变得非常麻烦。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/sub-if-vxlan.png)



此时就应该采用vlan接入vxlan隧道的方法。vlan接入vxlan隧道只需要在物理接口下允许携带这些vlan的报文通过，然后再将vlan与bd绑定，建立bd与vni对应的bd信息，最后创建vxlan隧道即可。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vlan-vxlan.png)

vlan与bd绑定配置如下：

```bash
bridge-domain 10    //创建一个编号为10的bd 
l2 binding vlan 10 //将bd10与vlan10绑定  
vxlan vni 5000  //设置bd10对应的vni为5000   
```

end

### VxLAN and VPN

VPN tunneling is the backbone of VXLAN, which provides a communication protocol connecting data centers from Layer 2 ports to Layer 3 ports. A VXLAN overlay network sits on top of the physical network and allows users to use a Virtual Private Network.

VxLAN依靠VPN Tunnel，但是隧道封装协议是vxlan协议。

Basically VPN is used to connect between two sites, so that the communication between those sites will be secured. VPN uses either ESP or AH header to encapsulate the packet. If you use AH, then you provided only Authentication, when you use ESP, you provide both Authentication as well as Integrity.

Speaking about VXLAN, it is another encapsulation mechanism which uses UDP header. So basically an outer header is added with UDP encapsulation. VXLAN doesn't encrypt the data.



### VxLAN and SDN

VXLAN in SDN is used to automate configuration.

EVPN也可以做VXLAN的控制平面哦，替代手工配置VxLAN Tunnel！

vxlan的配置挺麻烦的，而且不同厂商的还不一样，如下：

```bash
## 华为配置vxlan
[*HUAWEI]bridge-domain 20
[*HUAWEI-bd20]vxlan vni 200
[*HUAWEI-bd20]commit 
[~HUAWEI-bd20]quit
[~HUAWEI]display vxlan vni
Number of vxlan vni : 2
VNI            BD-ID            State   
---------------------------------------
200            20               down        
1001           10               down   

## 华三配置vxlan
[H3C]l2vpn enable
[H3C]vsi 20
[H3C-vsi-20]vx
[H3C-vsi-20]vxlan 2020
[H3C-vsi-20-vxlan-2020]quit
[H3C-vsi-20]quit
# 显示vxlan
[H3C]display vxlan tunnel
Total number of VXLANs: 1
VXLAN ID: 2020, VSI name: 20
```

使用vxlan网络时，呈现给用户时，应该屏蔽掉这些细节。

### VXLAN and VPN

Virtual Extensible LAN (VXLAN) uses MAC-in-UDP encapsulation. It is a type of Network Virtualization over Layer 3 (NVO3) technology. Key reasons for selecting VXLAN can be:

* Low requirements on the physical network: Virtual systems can be built on top of any physical network.
* Low configuration complexity: SDN is used to automate configuration.
* Low networking requirements: Not all devices on the entire network are required to support VXLAN

**MPLS VPN** uses L3 on underlay network while VXLAN can be built on top of any layer of the physical network. Moreover, in MPLS VPN carrier grade technical skills are required to configure thorough CLI while VXLAN in SDN is used to automate configuration.
And MPLS VPN is supported on E2E links while Only VTEPs are required to support VXLAN. VXLAN does not need to be supported on E2E links.

I hope it clarifies it for you. In case of any further clarification, please do comment.

Thank you.

### 引用

0. https://blog.csdn.net/weixin_33912453/article/details/92187910
1. https://www.zhihu.com/question/51675361/answer/263300150
2. http://www.3542xyz.com/?p=1331
3.  https://www.cnblogs.com/mlgjb/p/8087612.html
3.  https://www.jianshu.com/p/cccfb481d548
3.  https://blog.51cto.com/u_11555417/2364533
3.  https://lk668.github.io/2020/12/13/2020-12-13-ECMP/
3.  https://bbs.huaweicloud.com/blogs/110072
3.   http://specs.openstack.org/openstack/neutron-specs/specs/kilo/ml2-hierarchical-port-binding.html
3.  https://www.cnblogs.com/dream397/p/12316493.html
3.  https://www.ruijie.com.cn/cp/xw-cp-fw-ywszj/87359/



