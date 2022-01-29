## Overlay and Underlay

### Underlay

**Underlay网络可以是由多个类型设备互联而成的物理网络，负责网络之间的数据包传输。**

在Underlay网络中，互联的设备可以是各类型交换机、路由器、负载均衡设备、防火墙等，但网络的各个设备之间必须通过路由协议来确保之间IP的连通性。

Underlay网络可以是二层也可以是三层网络。其中二层网络通常应用于以太网，通过VLAN进行划分。三层网络的典型应用就是互联网，其在同一个自治域使用OSPF、IS-IS等协议进行路由控制，在各个自治域之间则采用BGP等协议进行路由传递与互联。随着技术的进步，也出现了使用[MPLS](https://info.support.huawei.com/info-finder/encyclopedia/zh/MPLS.html)这种介于二三层的WAN技术搭建的Underlay网络。

然而传统的网络设备对数据包的转发都基于硬件，其构建而成的Underlay网络也产生了如下的问题：

- 由于硬件根据目的IP地址进行数据包的转发，所以传输的路径依赖十分严重。
- 新增或变更业务需要对现有底层网络连接进行修改，重新配置耗时严重。
- 互联网不能保证私密通信的安全要求。
- 网络切片和网络分段实现复杂，无法做到网络资源的按需分配。
- 多路径转发繁琐，无法融合多个底层网络来实现负载均衡。
- end

#### Underlay网络模型

**Underlay网络就是传统IT基础设施网络，由交换机和路由器等设备组成，借助以太网协议、路由协议和VLAN协议等驱动，它还是Overlay网络的底层网络，为Overlay网络提供数据通信服务**。





### Overlay

为了摆脱Underlay网络的种种限制，现在多采用网络虚拟化技术在Underlay网络之上创建虚拟的Overlay网络。

在Overlay网络中，设备之间可以通过逻辑链路，按照需求完成互联形成Overlay拓扑。

相互连接的Overlay设备之间建立隧道，数据包准备传输出去时，设备为数据包添加新的IP头部和隧道头部，并且被屏蔽掉内层的IP头部，数据包根据新的IP头部进行转发。当数据包传递到另一个设备后，外部的IP报头和隧道头将被丢弃，得到原始的数据包，在这个过程中Overlay网络并不感知Underlay网络。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/overlay-arch.png)

Overlay网络有着各种网络协议和标准，包括VXLAN、NVGRE、SST、GRE、NVO3、EVPN等。

VXLAN和NVGRE是实现隧道化的高级网络虚拟化技术，它们将虚拟网络的规模从4094扩展到了1600万，并且允许第2层的数据包在第3层的网络上进行传输，因此大型数据中心通常会添加支持NVGRE和VXLAN的网络设备来扩展网络，如，使用支持NVGRE和VXLAN的交换机克服虚拟局域网在大型数据中心中的限制，且提供更为敏捷的虚拟机网络环境。

#### SDN/NFV and Overlay: 666

设备要求 --> NFV --> H3C：666

Underlay网络正如其名，是Overlay网络的底层物理基础。

为了实现Overlay的组网需求，Underlay对设备也有要求，比如要支持VxLAN或者NVGRE协议，但是当交换机设备不支持时怎么办，那就使用sdn/NFV哇，用软件模拟一个支持VxLAN或者NVGRE协议的网络设备。这么多虚拟设备怎么管理呢，就需要一个SDN/NFV 控制器了。

在联系到新华三的产品：

* NFV：为SDN Overlay网络提供虚拟网络功能设备，然后还得使用交换或路由协议组网，实现高可用或者聚合等等

* VCF：VCFC-DC 接管整个 SDN Overlay 网络

* CAS虚拟平台，配置CVM和VCFC资源联动，即实现cas平台创建出来的虚拟机可以使用定义的SDN Overlay网络--

  > cvk则是cas中的计算节点 cvm是cas管理节点 cvm+cvm=cas

* end

#### 数据中心Overlay

随着数据中心架构演进，现在数据中心多采用Spine-Leaf架构构建Underlay网络，通过VXLAN技术构建互联的Overlay网络，业务报文运行在VXLAN Overlay网络上，与物理承载网络解耦。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/data-center-overlay.png)

Leaf与Spine全连接，等价多路径(ECMP)提高了网络的可用性。

Leaf节点作为网络功能接入节点，提供Underlay网络中各种网络设备接入VXLAN网络功能，同时也作为Overlay网络的边缘设备承担VTEP（VXLAN Tunnel EndPoint）的角色。

Spine节点即骨干节点，是数据中心网络的核心节点，提供高速IP转发功能，通过高速接口连接各个功能Leaf节点。

#### SD-WAN中的Overlay网络

SD-WAN的Underlay网络基于广域网，通过混合链路的方式达成总部站点、分支站点、云网站点之间的互联。通过搭建Overlay网络的逻辑拓扑，完成不同场景下的互联需求。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/SD-wan-overlay.png)

SD-WAN的网络主要由CPE设备构成，其中CPE又分为Edge和GW两种类型。

- Edge：是SD-WAN站点的出口设备。
- GW：是联接SD-WAN站点和其他网络（如传统VPN）的网关设备。

根据企业网络规模、中心站点数量、站点间互访需求可以搭建出多个不同类型的Overlay网络。

- Hub-spoke：适用于企业拥有1~2个数据中心，业务主要在总部和数据中心，分支通过WAN集中访问部署在总部或者数据中心的业务。分支之间无或者有少量的互访需求，分支之间通过总部或者数据中心绕行。
- Full-mesh：适用于站点规模不多的小企业，或者在分支之间需要进行协同工作的大企业中部署。大企业的协同业务，如VoIP和视频会议等高价值的应用，对于网络丢包、时延和抖动等网络性能具有很高的要求，因此这类业务更适用于分支站点之间直接进行互访。
- 分层组网：适应于网络站点规模庞大或者站点分散分布在多个国家或地区的大型跨国企业和大企业，网络结构清晰，网络可扩展性好。
- 多Hub组网：适用于有多个数据中心，每个数据中心均部署业务服务器为分支提供业务服务的企业。
- POP组网：当运营商/MSP面向企业提供SD-WAN网络接入服务时，企业一时间不能将全部站点改造为SD-WAN站点，网络中同时存在传统分支站点和SD-WAN站点这两类站点，且这些站点间有流量互通的诉求。一套IWG（Interworking Gateway，互通网关）组网能同时为多个企业租户提供SD-WAN站点和已有的传统MPLS VPN网络的站点连通服务。

end



### Underlay vs Overlay

Underlay网络 VS Overlay网络

| 对比项       | Underlay网络                                                 | Overlay网络                                                  |
| ------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 数据传输     | 通过网络设备例如路由器、交换机进行传输                       | 沿着节点间的虚拟链路进行传输                                 |
| 包封装和开销 | 发生在网络的二层和三层                                       | 需要跨源和目的封装数据包，产生额外的开销                     |
| 报文控制     | 面向硬件                                                     | 面向软件                                                     |
| 部署时间     | 上线新服务涉及大量配置，耗时多                               | 只需更改虚拟网络中的拓扑结构，可快速部署                     |
| 多路径转发   | 因为可扩展性低，所以需要使用多路径转发，而这会产生更多的开销和网络复杂度 | 支持虚拟网络内的多路径转发                                   |
| 扩展性       | 底层网络一旦搭建好，新增设备较为困难，可扩展性差             | 扩展性强，例如VLAN最多可支持4096个标识符，而VXLAN则提供多达1600万个标识符 |
| 协议         | 以太网交换、VLAN、路由协议（OSPF、IS-IS、BGP等）             | VXLAN、NVGRE、SST、GRE、NVO3、EVPN                           |
| 多租户管理   | 需要使用基于NAT]或者VRF的隔离，这在大型网络中是个巨大的挑战  | 能够管理多个租户之间的重叠IP地址                             |

end

### Overlay on VPN：好好看

MPLS/ L3 VPN中介绍的Overlay是怎样的。

VPN的两大类别：

Peer-to-Peer VPN
Overlay VPN

### 引用

1. https://www.cnblogs.com/fengdejiyixx/p/15567609.html
2. 