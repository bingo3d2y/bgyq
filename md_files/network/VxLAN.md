## VxLAN

在underlay环境下不同网络的设备需要连接至不同的交换机下，如果要改变设备所属的网络，则要调整设备的连线。引入vlan后，调整设备所属网络只需要将设备加入目标vlan下，避免了设备的连线调整。

vxlan使得网络变更更加灵活了。

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

数据中心的虚拟化给网络设备带来的最直接影响就是：之前TOR（Top Of Rack）交换机的一个端口连接一个物理主机对应一个MAC地址，但现在交换机的一个端口虽然还是连接一个物理主机但是可能进而连接几十个甚至上百个虚拟机和相应数量的MAC地址。传统交换机是根据MAC地址表实现二层转发。

MAC地址表是通过交换机的flood-learn学习并记录在交换机的内存。交换机的内存比较宝贵，所以MAC地址表的大小通常是有限的。整个数据中心的MAC地址多了几十倍，那相应的交换机里面的MAC地址表也需要扩大几十倍。如果交换机不支持这么大的MAC地址表，那么就会导致MAC地址表溢出。溢出之后，交换机不能将新的MAC地址学习到自己的MAC地址表。如果交换机收到这些MAC地址的数据帧，因为不能通过查表转发，会flood到所有的端口。这不但增加了交换机的负担，还增加了网络中其他设备的负担。为了避免这个问题，可以用一些更大容量的交换机，但是相应的成本也要上升，而且还不能从根本上解决这个问题。

如果使用VXLAN，虚拟机的Ethernet Frame被VTEP封装在UDP里面，一个VTEP可以被一个物理主机上的所有虚拟机共用。从交换机的角度，交换机看到的是VTEP之间在传递UDP数据。通常，一个物理主机对应一个VTEP，所以交换机的MAC地址表，只需要记录与物理主机数量相当条目就可以了，虚拟化带来的MAC地址表暴增的问题也不存在了。这是VXLAN能解决的，而现有的VLAN没有办法回避的问题。

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



### VxLAN 应用： Openstack，通透

Openstack的租户网络使用VxLAN，可以提供1600w，满足各种需求，主机层面仍使用vlan即可，因为一个host上不可能跑4000个虚拟机。

####  VNI 和 VID 转换: vxlan的意义

在openstack中，尽管br-tun上的vni数量增多，但br-int上的网络类型只能是vlan，所有vm都有一个内外vid（vni）转换的过程，将用户层的vni转换为本地层的vid。
尽管br-tun上vni的数量为16777216个，但br-int上vid只有4096个，那引入vxlan是否有意义？

答案是肯定的，以目前的物理机计算能力来说，假设每个vm属于不同的tenant，1台物理机上也不可能运行4094个vm，所以这么映射是有意义的。

**一个host上的vm可能属于不同的vlans. openstack 同一个vxlan下可能对应不同了vlans.**

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/openstack-vxlan-vlan.png)



有的vm属于同一个tenant，即在用户层vni一致（同一个vxlan），但在本地层，同一tenant由nova-compute分配的vid可以不一致，同一宿主机上同一tenant的相同subnet(相同VLAN)之间的vm相互访问不需要经过内外vid（vni）转换，不同宿主机上相同tenant的vm之间相互访问则需要经过vid（vni）转换。**如果所有宿主机上vid和vni对应关系一致，整个云环境最多只能有4094个tenant，引入vxlan才真的没有意义。**

#### br-tun：6

Neutron如何做VID的转换：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/openstack-br-tun.jpg)

把br-tun一分为二，设想为两部分：

* 上层是VTEP，对VXLAN隧道进行了终结；

* 下层是一个普通的VLAN Bridge。

所以，对于Host来说，它有两重网络，如图所示，虚线以上是VXLAN网络，虚线以下是VLAN网络。如此一来，VXLAN内外VID的转换则变成了不得不做的工作，因为它不仅仅是表面上看起的那样，仅仅是VID数值的转变，而且背后还蕴含着网络类型的转变。

**VLAN类型的网络并不存在VXLAN这样的问题**。当Host遇到VLAN时，它并不会变成两重网络，可为什么也要做内外VID的转换呢？这主要是为了避免内部VLAN ID的冲突。内部VLAN ID是体现在br-int上的，而每一个Host内装有一个br-int，也就是说VLAN和VXLAN是共用一个br-int。假设VLAN网络不做内外VID的转换，则很可能引发br-int上的内部VLAN ID冲突。

如下图所示，Openstack 有两个类型的租户网络一个是VLAN类型，一个是VXLAN类型的网络，但是host上都只有由br-int提供的VLAN类型网络，所以必然发生VID转化。当不由Neutron同一管理时，可能发生冲突。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/openstack-vlan-and-vxlan.png)

VXLAN做内外VID转换时，并不知道VLAN的外部VID是什么，所以它就根据自己的算法将内部VID转换为100，结果很不幸中枪，与VLAN网络的外部VID相等。**因为VLAN的内部VID没有做转换，仍然是等于外部VID**，所以两者的内部VID在br-int上产生了冲突。正是这个原因，所以VLAN类型的网络，也要做内外VID的转换，而且是所有网络类型都需要做内外VID的转换。这样的话Neutron就能统一掌控，从而避免内部VID的冲突。

##### why need turn VID？

原因1： 满足租户自定义网络需求

vlan 网络也得转换，因为要对租户提供可自定义的VLAN网络，如果租户想创建自定义创建VLAN ID为`111`的网络，他提交了申请单，但是Openstack集群创建的时候VLAN网络模式没有预留`VLAN 111`，怎么办，这不是没有办法满足客户自定义的要求了吗？

这时，就需要VID 的转换了。管理员可以根据租户申请单，给租户创建`VLAN 111`网络，将这个`VLAN 111`映射到Openstack集群预留的`VLAN 217`，即可满足租户自定义网络的需求。

原因2： 避免VxLAN转换VLAN ID时冲突

假如`VxLAN VNI 1000`在br-tun时，转换时转换成了`VLAN 100`，这时候不巧的是underlay网络正好有`Vlan 100`这时候数据包怎么传输？？？？







#### VTEP Trunk网络规划

公有云架构中，vtep角色通过宿主机上的ovs实现，宿主机上联至接入交换机的接口类型为trunk，在物理网络中为vtep专门规划出一个网络平面。

vm在经过vtep时，通过流表规则，去除vid，添加上vni.

交换机Trunk互联，VLAN Trunk是一种网络设备间的point-to-point的连接。一条VLAN Trunk线路可以同时传输多个甚至所有的VLAN数据。

```bash
[SW1] interface gigabitEthernet0/0/24
[SW1-GigabitEthernet0/0/24] port link-type trunk
[SW1-GigabitEthernet0/0/24] port trunk allow-pass vlan 10 20
```

通过VLAN Trunk port，两个交换机之间不论需要连接多少个VLAN，只需要一个VLAN Trunk连接（一对Trunk port）即可。

如果有多个交换机需要连接，通常会用另外一个交换机连接它们，如下图所示，交换机之间都通过Trunk port相连。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/swtich-trunk.jpeg)

有两个交换机，各有两个VLAN，为了让VLAN能正常工作，将VLAN1和VLAN2分别连接起来。如果是两个交换机，两个VLAN还好，如果是100个VLAN，那么这里需要有100条线路，200个交换机端口。这么连接可以吗？可以，只要有钱，因为这里的线路和交换机端口都是钱。这种方法，运维和成本支出比较大，不实用。

#### VxLAN隧道建立

##### 子接口模式

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



##### vlan模式

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

### VxLAN: base
#### VxLAN网络模型

VXLAN 网络架构的组件有：

- 实体设备：VM、NVE和transit设备。
- 逻辑设备：VNI、VTEP、VAP、VXLAN隧道

##### 实体组件

1. **VM（Virtual Machine，虚拟机）**：在一台服务器上可以创建多台虚拟机，不同的虚拟机可以属于不同的VXLAN。属于相同VXLAN 的虚拟机处于同一个逻辑二层网络，彼此之间二层互通；属于不同VXLAN 的虚拟机之间二层隔离。

2. **NVE：NVE（Network Virtualization Edge）**：VXLAN网络虚拟边缘节点NVE，是实现网络虚拟化功能的**网络实体**。报文经过NVE封装转换后，NVE间就可基于三层基础网络建立二层虚拟化网络。**硬件设备、vSwitch或者软OVS都可以作为NVE。**

   NVE是参与underlay network的组网

   > 典型的Openvswitch

3. **transit设备**：transit设备不参与VXLAN处理，仅需要根据封装后报文的目的IP地址对报文进行三层转发。

##### 逻辑组件

1. **VNI(VXLAN Network Identitifier，VXLAN标示符)**：VXLAN 通过VXLAN ID 来标识，其长度为24 比特，VNI是一种类似于VLAN ID的用户标识，一个VNI代表一个租户，属于不同VNI的虚机之间不能直接进行二层通信。
2. **VTEP（VXLAN Tunnel End Point，VXLAN 隧道端点）**：VTEP是VXLAN隧道端点，封装在NVE中，用于VXLAN报文的封装和解封装。**VTEP与物理网络相连，分配有物理网络的IP地址，该地址与虚拟网络无关。说白了，就是VXLAN隧道两端网络可达的IP地址。**VXLAN报文中源IP地址为本节点的VTEP地址，VXLAN报文中目的IP地址为对端节点的VTEP地址，一对VTEP地址就对应着一个VXLAN隧道。
3. **VAP（Virtual Access Point）：虚拟接入点VAP统一为二层子接口**，用于接入数据报文。**为二层子接口配置不同的流封装**，可实现不同的数据报文接入不同的二层子接口。
4. **VXLAN 隧道**：两个VTEP 之间的点到点逻辑隧道。VTEP 为数据帧封装VXLAN 头、UDP 头和IP 头后，通过VXLAN 隧道将封装后的报文转发给远端VTEP，远端VTEP 对其进行解封装。

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

#### 组网例子

##### 1.  VxLAN内大二层互通：缺配置

https://blog.csdn.net/weixin_40228200/article/details/119303394

类似，传统VLAN 二层互通转发，实现PC_1 和 PC_2的通讯。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vxlan-basic.png)

缺了配置，Route 和 CE Switch怎么配置的

1、VxLAN业务配置

要配置VXLAN业务，就必须先配置BD域，BD域使用不同的数字进行区分，每一个BD域只能关联一个VNI，并且一个VNI也只能被一个BD域所关联。但是BD域具体的数字并没有太大的意义，关于VXLAN业务首先需要配置BD域以及VNI，相关配置如下

```bash
bridge-domain 10
vxlan vni 10
```

end

2、VXLAN数据包归属配置

在完成上述配置后，还必须告诉CE交换机，什么样的数据包要进行VXLAN封装，在这里我们将CE下行交换机的接口设置为Trunk类型，并且在CE交换机上设置二层子接口，并在子接口上配置关联，这样，所有属于VLAN 10的流量就和VXLAN相关联了。相关配置如下

```bash
interface GE1/0/1.10 mode l2
encapsulation dot1q vid 10
bridge-domain 10
```

end

3、VxLAN隧道配置

在完成上述配置后，就要配置一条VXLAN的隧道，在这里，默认情况下以Loopbake接口作为VXLAN的端点口，相关配置如下：

```bash
interface Nve1
 source 1.1.1.1
 vni 10 head-end peer-list 2.2.2.2
```

end





##### 2. 相同子网互通

某企业在不同的数据中心中都拥有自己的VM，服务器1上的VM1属于VLAN10，服务器2上的VM1属于VLAN20，且位于同网段。现需要通过VXLAN隧道实现不同数据中心同网段VM的互通。

**拓扑**

Device_1 连接数据中心1

Device_3 连接数据中心2

Device_2 连接Device_1 和 Device_3

**配置思路：**

1. 分别在Device1、Device2和Device3上配置路由协议，保证网络三层互通。
2. 分别在Device1和Device3上配置业务接入点实现区分业务流量。
3. 分别在Device1和Device3上配置VXLAN隧道实现转发业务流量。

**数据准备：**

1. VM所属的VLAN ID分别是VLAN10和VLAN20。
2. 网络中设备互连的接口IP地址。
3. 网络中使用的路由类型是OSPF（Open Shortest Path First）。
4. 广播域BD ID是BD10。
5. VXLAN网络标识VNI ID是VNI 5010。
6. #

##### 3. 不同子网互通（集中式）

**Spine-Leaf拓扑**

两个Spine交换机 Spine_1 and Spine_2 做高可用

Leaf_1、Leaf_2和Leaf_3代表三个子网。

Leaf 1-3分别连接 Spine 1-2.

**组网需求：**

该数据中心流量主要为虚拟机迁移等形成的东西向流量，因此，客户要求在现有基础承载网络上构建一个大二层网络，并且要求网络中的Spine设备同时作为网关设备，可以对Leaf设备发送过来的报文进行负载分担。

如图2-53所示，某企业数据中心内部网络为“Spine-Leaf”两层结构：

- Spine1和Spine2为基础承载网络中的骨干节点，位于汇聚层；
- Leaf1～Leaf3为基础承载网络中的叶子节点，位于接入层；
- Leaf设备与Spine设备之间全连接，构成ECMP，保证网络的高可用性；Spine节点之间互不连接，Leaf节点之间也不互联。

**配置思路：**采用如下思路配置集中式多活网关：

1. 在Leaf1～Leaf3、Spine1～Spine2上配置OSPF，保证网络三层互通。
2. 在Leaf1～Leaf3、Spine1～Spine2上配置VXLAN隧道，实现在基础三层网络上创建虚拟大二层VXLAN网络。
3. 在Leaf1～Leaf3上配置业务接入点，区分出服务器的流量并转发至VXLAN网络。
4. 在Spine1～Spine2上配置VXLAN三层网关实现VXLAN与非VXLAN网络、不同网段VXLAN之间的通信。

注意：如果Spine1/Spine2接入上行网络的链路出现故障，当用户流量到达Spine1/Spine2时，由于没有可用的上行出接口，Spine1/Spine2会将用户流量全部丢弃。此时，可以配置monitor-link关联Spine1/Spine2的上行接口和下行接口，当Spine1/Spine2的上行出接口DOWN时，下行接口状态也会变为DOWN，这样用户侧流量就不会通过Spine1/Spine2进行转发，从而防止流量被丢弃。monitor-link的配置可以参见配置Monitor Link组的上行接口和下行接口。

**数据准备：**

1. 网络中设备互连的接口IP地址。
2. 网络中使用的路由类型是OSPF（Open Shortest Path First）。
3. VM所属的VLAN ID分别是VLAN10、VLAN20和VLAN30。
4. 广播域BD ID分别是BD10、BD20和BD30。
5. VXLAN网络标识VNI ID分别是VNI5000、VNI5001和VNI5002。

**配置步骤：**

**配置OSPF(略)**

1. 在Leaf1～Leaf3、Spine1～Spine2配置OSPF。
2. 配置OSPF时，注意需要发布32位Loopback接口地址。
3. OSPF成功配置后，设备之间可通过OSPF协议发现对方的Loopback接口的IP地址，并能互相ping通。

**配置隧道模式并使能NVO3的ACL扩展功能**

```bash
# 配置Leaf1。Spine2、Leaf1、Leaf2和Leaf3的配置与Leaf1类似，这里不再赘述。
[~Spine1] ip tunnel mode vxlan
[*Spine1] assign forward nvo3 acl extend enable
[*Spine1] commit

注意：
1.    从V100R005C10版本开始，支持使能NVO3的ACL扩展功能。
2.    配置隧道模式、使能NVO3的ACL扩展功能后，需要保存配置并重启设备才能生效，您可以选择立即重启或完成所有配置后再重启。
```

**配置VXLAN业务接入点**

```bash
在Leaf1～Leaf3配置VXLAN业务接入点。
配置Leaf1。
[~Leaf1] bridge-domain 10
[*Leaf1-bd10] quit
[~Leaf1] interface 10ge 1/0/3.1 mode l2
[*Leaf1-10GE1/0/3.1] encapsulation dot1q vid 10
[*Leaf1-10GE1/0/3.1] bridge-domain 10
[*Leaf1-10GE1/0/3.1] quit
[*Leaf1] commit

配置Leaf2。
[~Leaf2] bridge-domain 20
[*Leaf2-bd10] quit
[~Leaf2] interface 10ge 1/0/3.1 mode l2
[*Leaf2-10GE1/0/3.1] encapsulation dot1q vid 20
[*Leaf2-10GE1/0/3.1] bridge-domain 20
[*Leaf2-10GE1/0/3.1] quit
[*Leaf2] commit

配置Leaf3。
[~Leaf3] bridge-domain 30
[*Leaf3-bd10] quit
[~Leaf3] interface 10ge 1/0/3.1 mode l2
[*Leaf3-10GE1/0/3.1] encapsulation dot1q vid 30
[*Leaf3-10GE1/0/3.1] bridge-domain 30
[*Leaf3-10GE1/0/3.1] quit
[*Leaf3] commit
```

**配置VXLAN隧道**

三个Leaf的VxLAN对端都是Spine 1 and Spine 2的 vip 10.10.10.1

```bash
在Leaf1～Leaf3、Spine1～Spine2上配置VXLAN隧道，组建VXLAN网络

配置Leaf1。
[~Leaf1] bridge-domain 10
[*Leaf1-bd10] vxlan vni 5000
[*Leaf1-bd10] quit
[*Leaf1] interface nve 1
[*Leaf1-Nve1] source 10.10.10.3
# //指定对端VTEP地址，并调用VNI
[*Leaf1-Nve1] vni 5000 head-end peer-list 10.10.10.1
[*Leaf1-Nve1] quit
[*Leaf1] commit

配置Leaf2。
[~Leaf2] bridge-domain 20
[*Leaf2-bd10] vxlan vni 5001
[*Leaf2-bd10] quit
[*Leaf2] interface nve 1
[*Leaf2-Nve1] source 10.10.10.4
# //指定对端VTEP地址，并调用VNI
[*Leaf2-Nve1] vni 5001 head-end peer-list 10.10.10.1
[*Leaf2-Nve1] quit
[*Leaf2] commit

配置Leaf3。
[~Leaf3] bridge-domain 30
[*Leaf3-bd10] vxlan vni 5002
[*Leaf3-bd10] quit
[*Leaf3] interface nve 1
[*Leaf3-Nve1] source 10.10.10.5
# //指定对端VTEP地址，并调用VNI
[*Leaf3-Nve1] vni 5002 head-end peer-list 10.10.10.1
[*Leaf3-Nve1] quit
[*Leaf3] commit

# 配置Spine1。Spine2的配置与Spine1一样，此处不再赘述。
[~Spine1] bridge-domain 10
[*Spine1-bd10] vxlan vni 5000
[*Spine1-bd10] quit
[*Spine1] bridge-domain 20
[*Spine1-bd20] vxlan vni 5001
[*Spine1-bd20] quit
[*Spine1] bridge-domain 30
[*Spine1-bd30] vxlan vni 5002
[*Spine1-bd30] quit

# 10.10.10.1 就是 Spine Switch IP
[*Spine1] interface nve 1
[*Spine1-Nve1] source 10.10.10.1
[*Spine1-Nve1] vni 5000 head-end peer-list 10.10.10.3
[*Spine1-Nve1] vni 5001 head-end peer-list 10.10.10.4
[*Spine1-Nve1] vni 5002 head-end peer-list 10.10.10.5
[*Spine1-Nve1] quit
[*Spine1] commit
```

**上述配置成功后，在Leaf1～Leaf3、Spine1～Spine2上执行display vxlan vni命令可查看到VNI的状态是up；执行display vxlan tunnel命令可查看到VXLAN隧道的信息。以Spine1显示为例。**

```bash
[~Spine1] display vxlan vni
Number of vxlan vni : 3
VNI BD-ID State
---------------------------------------
5000 10 up
5001 20 up
5002 30 up

[~Spine1] display vxlan tunnel
Number of vxlan tunnel : 3
Tunnel ID Source Destination State Type
--------------------------------------------------------------
4026531842 10.10.10.1 10.10.10.3 up static
4026531843 10.10.10.1 10.10.10.4 up static
4026531844 10.10.10.1 10.10.10.5 up static
```

**配置VXLAN三层网关**

```bash
在Spine1～Spine2上配置VXLAN三层网关
# 配置Spine1。Spine2的配置与Spine1一样，此处不再赘述。
[~Spine1] interface vbdif 10
[*Spine1-Vbdif10] ip address 192.168.10.1 24
[*Spine1-Vbdif10] mac-address 0000-5e00-0101
[*Spine1-Vbdif10] quit
[*Spine1] interface vbdif 20
[*Spine1-Vbdif20] ip address 192.168.20.1 24
[*Spine1-Vbdif20] mac-address 0000-5e00-0102
[*Spine1-Vbdif20] quit
[*Spine1] interface vbdif 30
[*Spine1-Vbdif30] ip address 192.168.30.1 24
[*Spine1-Vbdif30] mac-address 0000-5e00-0103
[*Spine1-Vbdif30] quit
[*Spine1] commit

注意：
由于Spine1和Spine2作为多活网关设备，请确保这两台设备上配置的NVE接口的IP地址、BDIF接口的IP地址和MAC地址相同。
```

**配置VXLAN三层网关多活**

```bash
在Spine1～Spine2上配置多活网关
# 配置Spine1。Spine2的配置与Spine1类似，此处不再赘述。
[~Spine1] dfs-group 1
[*Spine1-dfs-group-1] source ip 10.10.10.10
[*Spine1-dfs-group-1] active-active-gateway
[*Spine1-dfs-group-1-active-active-gateway] peer 10.10.10.20
[*Spine1-dfs-group-1-active-active-gateway] quit
[*Spine1-dfs-group-1] quit
[*Spine1] commit
```

检查配置结果，上述配置成功后，在Spine1、Spine2上执行命令display dfs-group 1 active-activegateway，可以查看到DFS Group多活网关的信息。以Spine1显示为例。

```bash
[~Spine1] display dfs-group 1 active-active-gateway
A:Active I:Inactive
-------------------------------------------------------------------
Peer System name State Duration
10.10.10.20 Spine2 A 0:0:8
```

end

### 引用

0. https://blog.csdn.net/weixin_33912453/article/details/92187910

1. https://www.zhihu.com/question/51675361/answer/263300150
2. http://www.3542xyz.com/?p=1331
3.  https://www.cnblogs.com/mlgjb/p/8087612.html
3.  https://www.jianshu.com/p/cccfb481d548
3.  https://blog.51cto.com/u_11555417/2364533
3.  https://lk668.github.io/2020/12/13/2020-12-13-ECMP/
3.  https://bbs.huaweicloud.com/blogs/110072

