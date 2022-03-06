## SDN and NFV

cisco ： https://www.cisco.com/c/en/us/solutions/software-defined-networking/sdn-vs-nfv.html

### 网元

网元由一个或多个机盘或机框组成， 能够独立完成一定的传输功能。

网元是网络管理中可以监视和管理的最小单位。 

NFV 云化带来了网元架构的本质变 化，从软/硬件一体的网元架构变成了软/硬件分开的网元架构。

#### 物理网元功能PNF：瀑布开发

物理网元设备受限厂商一般都同厂商组网。

物理网元的建设方案主要受 厂商格局影响，共性是不同厂商的物理网元都采用专有硬件建设形式。传统设备商提供的是从硬件、组网、系统软件到业务软件的成套完 整设备，可以看作一种垂直式的应用部署；不同的网元硬件设备可能不一样，而网元相同、 厂商不同的硬件设备也会不一样。客户的维护人员熟悉了一个厂商的设备架构之后，对其他 厂商的设备引入都会有新的学习和熟悉过程。



#### 云化网元功能VNF：敏捷开发

云化后，云化网元打破了厂商的严格界限， 一个提供原有全部物理网元功能的云化网元被 分解为不同层硬件与软件功能的组合体，构成 包括基础设施层的 IT 通用硬件 x86 服务器、虚 拟资源层的 VM 及 Hypervisor 功能、网络功能 层的虚拟网元功能软件 VNF（即现有物理网元 如 CSCF、HSS 等网元的虚拟化功能的软件实 体）。

> 现实中，厂商出于各种目的会尽量避免自家的VNF与第三方VNF集成的方案出现。厂商可能会说，与第三方的集成需要额外收费，或者说兼容性不好，集成起来非常麻烦，甚至说与第三方的连接性能不如全采用自己一家。厂商的目的大部分时候是简单的做单，尽量做大单，而不是真正意义的开放，与第三方集成违背他们自己的本性。

|     对比方面     | 物理网元                                             | 云化网元                                                     |
| :--------------: | ---------------------------------------------------- | ------------------------------------------------------------ |
|     网元形式     | 专业设备专用功能，同厂商单一形式集成                 | 通用设备虚拟功能，不同厂商可多种形式混搭集成                 |
|  供货厂商多样性  | 受限于华为、中兴通讯、爱立信等传统运营商设备厂商     | 基础设施层可以引入多家IT设备厂商，虚拟化层也可以引入其他集成商 |
|  规划建设灵活性  | 方案单一，灵活性受限                                 | 提供方案多样化契机，回旋余地大，工程建设复杂                 |
|  维护管理复杂性  | 专用设备故障定位容易，网络维护简单                   | 通用设备分层集成故障定位困难，网络维护复杂                   |
| 设备及组网可靠性 | 专用设备可靠性                                       | IT设备自身可靠性低，导致云化网元需要借助更多的网络手段保障可靠性 |
|  业务能力开放性  | 运营商专用平台主要提供专用业务，不便于提供第三方业务 | 基于API的共享开放平台，除运营商专用业务外，更便于提供第三方业务 |



### 传统网络

**在传统的网络架构下，控制面和数据面位于同一个设备中。**

传统的IP网络是一个分布式的无中心架构，各个网络设备包含完整的控制面和数据面，单个设备通过网络协议探测网络中其他设备的状态，自主决定如何对流量进行路由。该架构的好处是容错性强，即使网络局部出现故障，整个网络也可以自主恢复，不会影响整个网络的运行。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/传统网络设备.jpg)

这种去中心的架构在基于文本和图片的web浏览器应用时代运作良好，但随着互联网业务的爆炸式增长，各种各样的业务纷纷承载在了IP网络上，包括各种实时业务如语音视频通话，对网络提出了新的挑战。

- 缺少网络质量保证： 绝大多数IP网络都是基于无连接的，只有基于大带宽的粗放带宽保障措施，缺乏质量保证和监控，业务体验较差。
- 低效的业务部署： 网络配置是通过命令行或者网管、由管理员手工配置到网络设备上，并且各种网络设备之间的控制命令互不兼容，导致业务的部署非常低效。
- 缓慢的业务适应： 业务由硬件实现，导致新业务的开发周期过长。需要持续数年的特性和架构调整、引入新设备，才能推出新的业务，无法快速适应市场，满足用户的需求。

#### 黑盒交换机

传统网络设备的软硬件开发均由设备厂商提供，这种黑盒架构致 使系统完全封闭，存在低利用率、高成本、低敏捷性和业务上线周期 长等问题，难以适应未来网络的发展需求。

比如，华为、华三和思科的设备操作命令不一致--，维护成本很高。

#### 白盒交换机

白盒交换机是一种软硬件解耦的开放网络设备

*  白盒交换机采用开放的 设备架构和软硬解耦思想,将设备硬件与软件分离，能够将标准化的硬件配置与不同的软件协议进行混合并匹配.
* 白盒交换机支持硬件数据面可编程和软件容器化部署，通过软件定义的方式定制数据面的转发逻辑。

一般而言，网络功能的实现需要控制面和转发 面两部分，白盒交换机需要在这两方面具备可编程能力。控制面较早 的实现了集中式的可编程功能；而大多数设备的转发面使用不可编程 的黑盒转发芯片，无法实现协议/功能的集中控制和定制。因此转发 面的可编程技术是设备白盒化发展的关键。

### OpenFlow

OpenFlow是SDN（Software Defined Network，软件定义网络）架构中定义的一个控制器与转发层之间的通信接口标准。OpenFlow允许控制器直接访问和操作网络设备的转发平面，这些网络设备可能是物理上的，也可能是虚拟的。

OpenFlow的思想是分离控制平面和数据平面，二者之间使用标准的协议通信；数据平面采用基于流的方式进行转发

OpenFlow网络由OpenFlow设备（Switch）和控制器（Controller）通过安全通道（OpenFlow channel）组成。Switch与Controller通过TLS或者TCP建立安全通道，进行OpenFlow消息交互，实现表项下发、查询以及状态上报等功能

### SDN

目标：？

SDN的思想是取消设备控制平面，由控制器统一计算，下发流表，SDN是全新的网络架构。

只将底层的网络配置互通和高可用即可，后续的网络配置和变更，通过SDN来实现。



SDN：Software-Defined Network

* 目标是简化网络管理和控制
* 核心思想是控制与转发的分离



为了解决传统网络的问题，提出了SDN（software-defined networking）解决方案：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/sdn-arch.png)



SDN从下至上划分为三层体系结构：

- 基础设施层（Infrastructure Layer）： 是网络的数据平面，由各种网络设备构成，根据控制层下发的路由规则对网络数据进行处理和转发。

  举例：openvswitch提供的vswitch

- 控制层（Control Layer）： 是一个逻辑上集中的控制平面，维护了整个网络的信息，如网络拓扑和状态信息等，控制层根据业务逻辑生成配置信息下发给基础设施层，以管理整个网络的运行。

  举例：OVN 作为 ovs的控制器或者是Neutron作为ovs的控制器

- 应用层（Application Layer）：通过控制层提供的编程接口，可以编写各种应用利用控制层的能力对网络进行灵活的配置。

SDN的不同层次之间采用标准接口进行通信：

- 南向接口（Southbound API）：数据面和控制面的接口，控制面通过该接口将路由和配置下发到数据面，该接口一般使用OpenFlow、NetConf等网络协议。标准的南向接口解耦了控制层和基础设施层，只要支持标准南向接口的交换机都可以接入到SDN网络中，这样基础设施层可以大量采用支持OpenFlow协议的白盒交换机，解除了厂商锁定，降低了网络成本。
- 北向接口（Northbound API）：控制面向应用层提供的可编程接口，例如RestConf接口。利用控制面提供的编程接口，新的网络业务可以通过存软件的方式实现，大大加快了推出新业务的周期。

#### SDN控制器

控制器是整个SDN网络的核心大脑，负责数据平面资源的编排、维护网络拓扑和状态信息等，并向应用层提供北向API接口。其核心技术包括

- 链路发现和拓扑管理
- 高可用和分布式状态管理
- 自动化部署以及无丢包升级

#### 北向接口

SDN北向接口是通过控制器向上层业务应用开放的接口，其目标是使得业务应用能够便利地调用底层的网络资源和能力。

eg：Neutron通过neutron-openvswitch-agent（Adaptor）调用底层的vswitch

https://drobp.github.io/2019/08/06/%E5%8C%97%E5%90%91%E6%8E%A5%E5%8F%A3/

REST风格，有一定的契约标准。

契约在软件上最基本的体现就是函数。当一个函数被定义出来时：它告诉它的使用者，你我之间应该如何合作。

比如，北向支持JAVA API 和 Rest API

通过控制器提供的北向接口快速开发各种SDN应用，而不需要对硬件进行升级，这种模式加快了新业务上线的周期，鼓励各种创新业务蓬勃发展

#### 南向接口

SDN通过标准的南向接口屏蔽了底层物理转发设备的差异，实现了资源的虚拟化，同时开放了灵活的北向接口供上层业务按需进行网络配置并调用网络资源。

比如，南向支持Openflow、OVSDN、Netconf等

南向接口协议是为了实现控制平面的控制器（或者说网络操作系统）和数据平面的交换机之间的信息交互而设计的协议，简单来说南向接口协议就是控制器与交换机的通信协议。
南向接口协议主要完成什么任务呢？
不同的南向接口协议有不同的实现目标，从已经实现的南向接口协议来看。概括一下，南向接口协议的主要设计目标有以下几类：
1.实现数据平面与控制平面的信息交互，向上收集交换机信息（比如交换机的特性、配置信息、工作状态等）， 向下下发控制策略，指导数据平面的转发行为。
2.实现网络的配置与管理
3.面向流量工程而设计，如实现路径计算，包括传送链路的带宽与开销等属性、链路状态和拓扑信息等。
简单介绍几个以实现南向接口协议

#### 华三 SDN应用：VCF

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/sdn应用.jpg)

VCF（Virtual Converged Framework，虚拟应用融合架构）控制器是 H3C 推出的一款 SDN 控制器， 它类似一个网络操作系统，为用户提供开发和运行 SDN 应用的平台。VCF 控制器可以控制 OpenFlow 网络中的各种资源，并为应用提供接口，以实现特定的网络转发需求。

> 部署VCF产品的VM就是VCFC即 VCF Controller

VCFC是H3C SDN Controller，提供了对传统经典网络、OpenFlow网络、Overlay网络、NFV网络的支持和网络集中控制，更重要的是提供了基于OF的各种SDN APP，实现SDN的价值，如：软件定义的L2、L3、QoS、TE转发APP、Overlay转发APP、服务链APP、SDN APP的大规模集群架构，以及基于VCF Controller SDK的第三方APP。

VCF 控制器的主要特点如下：

 • 是一个基于 Linux/Java/OSGi 的控制平台，支持 OpenFlow 1.3，并且提供内置服务和设备驱 动程序框架；

 • 是一个高可靠和可扩展的分布式平台； 

• 提供嵌入式 Java 应用程序、可扩展的 REST API 和 GUI、iMC（H3C 智能管理中心）管理接 口； 

• 支持独立运行模式和集群模式。

实际配置例子：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/h3c-vcf-example-1.jpg)



VCF 控制器和上行设备之间运行 BGP 协议，组网环境如下：

* TOR 1、TOR 2 分别通过直连链路与 VCFC 1 和 VCFC 2 建立 EBGP 邻居。 
* TOR 1 和 TOR 2 分别通过 EBGP 协议与 Router 建立连接。 
* 控制器集群 IP 为 215.129.129.129/32

TOR 1 设备通过 VCFC 1 只能学习到 VCFC 1 环回口和集群 IP 的路由信息，通过 VCFC 2 只能学习到 VCFC 2 环回口和集群 IP 的路由信息。

##### 详细配置

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/h3c-sdn-vcf-1.png)

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/h3c-sdn-vcf-2.jpg)

测试组网说明：

 (1) 蓝色实线提供配置网络互通，要求 VCFC1、VCFC2 配置加入集群的 IP 地址相互之间可达。

 (2) Switch1 设备和 Switch2 之间通过传统的路由协议(BGP)使三层网络互通。

 (3) 在进行 VCFC 路由配置之前，需要在控制器上安装并运行路由软件 Quagga。

 (4) VCFC1 和 VCFC2 上配置 BGP 协议，发布网卡、环回口及集群 IP 网段。

```bash
## switch 1
#
interface GigabitEthernet1/0/3
port link-mode route
ip address 215.10.0.1 255.255.255.0
#
interface GigabitEthernet1/0/4
port link-mode route
ip address 215.10.3.1 255.255.255.0
#
interface Vlan-interface400
ip address 215.10.2.1 255.255.255.0
interface GigabitEthernet1/0/19
port link-mode bridge
port access vlan 400
#
bgp 101
peer 215.10.0.3 as-number 99
peer 215.10.2.2 as-number 102
peer 215.10.3.3 as-number 103
peer 215.121.121.121 as-number 99
#
address-family ipv4 unicast
 import-route direct
 network 215.10.0.0 255.255.255.0
 network 215.10.2.0 255.255.255.0
 network 215.10.3.0 255.255.255.0
 network 215.121.121.121 255.255.255.255
 peer 215.10.0.3 enable(VCFC 1)
 peer 215.10.2.2 enable
 peer 215.10.3.3 enable(VCFC 2)
 peer 215.121.121.121 enable

## switch 2

#
interface GigabitEthernet1/0/17
port link-mode route
ip address 215.10.1.1 255.255.255.0
#
interface GigabitEthernet1/0/18
port link-mode route
ip address 215.10.4.1 255.255.255.0
#
interface Vlan-interface400
ip address 215.10.2.2 255.255.255.0
#
interface GigabitEthernet1/0/19
port link-mode bridge
port access vlan 400
#
bgp 102
peer 215.10.1.3 as-number 99
peer 215.10.2.1 as-number 101
peer 215.10.4.3 as-number 103
peer 215.122.122.122 as-number 103
#
address-family ipv4 unicast
 import-route direct
 network 215.10.1.0 255.255.255.0
 network 215.10.2.0 255.255.255.0
 network 215.10.4.0 255.255.255.0
 network 215.122.122.122 255.255.255.255
 peer 215.10.1.3 enable(VCFC 1)
 peer 215.10.2.1 enable
 peer 215.10.4.3 enable(VCFC 2)
 peer 215.122.122.122 enable
 
## vcfc 
VCFC1：
215.10.0.3/24 eth1
215.10.1.3/24 eth2
215.121.121.121/32 lo
VCFC2：
215.10.3.3/24 eth1
215.10.4.3/24 eth2
215.122.122.122/32 lo
```

end



#### SDN实际组网例子

在安装H3C VCF控制器之前，建议您根据具体需求先规划好SDN网络，配置好欲与控制器连接的SDN交换机。

**VCF控制器可以直接访问和操作SDN交换机的转发平面**，控制SDN交换机对数据报文的转发。控制器和SDN交换机之间的控制报文主要由OpenFlow协议定义。OpenFlow协议规范包括多个版本，VCF控制器目前支持标准的OpenFlow 1.3版本。

与VCF控制器建立连接的所有SDN交换机组成的区域可以称为该控制器的控制域，在规划和配置SDN网络时，请确保控制域内的SDN交换机与控制域外的交换机之间没有环路，否则会形成广播风暴。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/h3c-sdn-flow.png)

其中各对应角色及其处理为：

Ø VCFC：H3C SDN控制器。作为网络资源池的唯一控制点，VCFC控制了虚拟化网络，并且通过对虚拟网络进行抽象和编排，定义服务链特征；**VTEP和服务节点上的转发策略都由控制器下发。**

Ø 服务链流分类节点（VTEP1）：原始报文通过VTEP1接入VXLAN网络，并直接进行流分类，以确定报文是否需要进入服务链：如果需要进入服务链，则将报文做VXLAN+服务链ID的封装，转到服务链首节点处理。

Ø 服务链首节点（SF1）：进行服务处理后，将数据报文继续做服务链封装，交给服务链下一个服务节点。

Ø 服务链尾节点（SF2）：进行服务处理后，服务链尾节点需要删除服务链封装，将报文做普通VXLAN封装，并转发给目的VTEP。如果SF2不具备根据用户报文寻址能力，则需要将用户报文送到网关(VTEP3)，VTEP3再查询目的VTEP进行转发。

报文转发说明如下：

Ø  ①⑥Native以太报文，IP(src)---->IP(dst)

Ø  ② VXLAN+业务链报文，外层: IP(VTEP1)---->IP(SF1)

Ø  ③ VXLAN+业务链报文，外层: IP(SF1)---->IP(SF2)

Ø  ④ VXLAN报文，外层: IP(SF2)---->IP(VTEP3)

Ø  ⑤ VXLAN报文，外层: IP(VTEP3)---->IP(VTEP4)

具体的匹配转发流程描述如下：

Ø  VM首包上送控制器处理时，在VCFC上解析packet in报文，根据报文目的地址确定是虚拟网络内的东西向流量还是通往传统网络的南北向流量，对于南北向流量则将报文转发到网关设备，报文后续的处理由网关设备负责；

Ø  对于东西向流量，从收到的packet in报文中提取源端口，并根据源端口确定源subnet、network、router信息；并根据packet in报文的目的IP获取目的VM连接的目的端口，并根据目的端口确定目的subnet、network、router信息；

Ø  对于东西向流量，根据报文特征进行服务链匹配，首先使用源端口和目的端口的属性与服务链配置进行匹配，如果找到匹配的服务链，则下发导流流表；如果没找到匹配的服务链，就下发东西向卸载的流表项（即非服务链转发）；

Ø  如果存在匹配的服务链，确定服务链所在VTEP的VTEP IP，VCFC向VTEP和后续处理节点下发流表项，流表项格式如下：

Match：port & vlan & 五元组   普通报文进入服务链

​      或 tunnel & vni & 五元组   vxlan报文 进入服务链

Action：vni & service chain id & order counter & tunnel，指向服务链首节点所在的服务节点，order counter =1。

Ø  当匹配到多条服务链时，按照最精确匹配的原则确定实际使用的服务链配置。上述4种维度按照精确程度从低到高排序依次为：Routers, Networks, Subnets, Ports。

### NFV

NFV, Network functions virtualization

NFV即网络功能虚拟化（Network Functions Virtualization），通过借用IT的虚拟化技术虚拟化形成VM（虚拟机，Virtual Machine），然后将传统的CT业务部署到VM上。

NFV通过使用x86等通用性硬件以及虚拟化技术，来承载很多功能的软件处理。从而降低网络昂贵的设备成本。可以通过软硬件解耦及功能抽象，使网络设备功能不再依赖于专用硬件，资源可以充分灵活共享，实现新业务的快速开发和部署，并基于实际业务需求进行自动部署、弹性伸缩、故障隔离和自愈等。

> 干掉硬件F5？？，那LVS算NFV吗

NFV的目标是取代通信网络中私有、专用和封闭的网元，实现统一通用硬件平台+业务逻辑软件的开放架构。

NFV 的引入能够实现运营商在 x86 等通用硬 件服务器上，通过虚拟化技术完成软硬件解耦， 使网络设备功能不再依赖于专用硬件，资源可以 充分共享，进而实现网络的灵活、动态构建和部 署以及自动调整。

#### VNF

**虚拟网络功能（VNF）**是提供网络功能（例如文件共享、目录服务和 IP 配置）的软件应用。

virtualized network function,VNF 实现传统非云化电信网元的功能。VNF 所需资源需要分解为虚拟的计算/存储/交换资源， 由 NFVI 来承载。一个 VNF 可以部署在一个或多 个虚拟机（virtual machine，VM）上。

#### NFVi

**网络功能虚拟化基础架构（NFVi）**包含了平台上的基础架构组件（计算、存储、联网），从而支持运行网络应用所需的软件（例如像 KVM 这样的虚拟机监控程序）或容器管理平台。

NFV infrastructure，主要功能是为 VNF 的部署、管理和 执行提供资源池，NFVI 需要将物理计算、存储、 交换资源虚拟化成虚拟的计算、存储、交换资源 池。NFVI 可以跨地域部署。



#### MANO

**管理、自动化和网络编排（MANO）**提供了用于管理 NFV 基础架构和置备新 VNF 的框架。

NFV management and orchestration, NFV 管理和编排系统，主要包括 NFV 编排器 （NFV orchestrator，NFVO）、VNF 管理器（VNF manager，VNFM）和虚拟设施管理器（virtualised infrastructure manager，VIM）3 部分。

NFVO，负责全网的网络服务、物理/虚拟资 源和策略的编排和维护以及其他虚拟化系统相关 维护管理功能。实现网络服务生命周期的管理， 与 VNFM 配合实现 VNF 的生命周期管理和资源 的全局视图功能。

 VNFM，实现虚拟化网元 VNF 的生命周期管 理，包括 VNFD 的管理及处理、VNF 实例的初始 化、VNF 的扩容/缩容、VNF 实例的终止。支持 接收 NFVO 下发的弹性伸缩策略，实现 VNF 的弹 性伸缩。

 VIM，主要负责基础设施层硬件资源、虚拟 化资源的管理，监控和故障上报，面向上层 VNFM 和 NFVO 提供虚拟化资源池



#### LVS is of VNF？

网络负载均衡主要有硬件与软件两种实现方式，主流负载均衡解决方案中，硬件厂商以F5为代表，软件主要为NGINX与LVS。但是，无论硬件或软件实现，都逃不出基于四层交互技术的“报文转发”或基于七层协议的“请求代理”这两种方式。 四层的转发模式通常性能会更好，但七层的代理模式可以根据更多的信息做到更智能地分发流量。一般大规模应用中，这两种方式会同时存在。

Yep，it is.

数据中心内的VNF更多的是用户开发，现在的软件，例如haproxy，lvs，iptables，dnsmasq等够较为方便的在linux机器上，实现相应的VNF。

对于电信运营商，VNF更多是采购自相应的网络设备商。而NFV最大吸引力之一在于打破了单一供应商的锁定，使用户在未来的网络搭建中掌握更多的主动权。用户可以根据技术和商业发展来改变自身的策略，混合各家的产品，采购最合适的解决方案，并且连接到同一个管理平台。这使得买方的议价能力大大提升。

#### SDN vs. NFV

SDN 将网络转发功能与网络控制功能分开，其目标是创建可集中管理和可编程的网络。NFV 则是从硬件中抽象出网络功能。NFV 通过提供可运行 SDN 软件的基础架构来支持 SDN。

NFV 和 SDN的关系：

1. NFV不依赖与SDN，但是SDN中控制和数据转发的分离可以改善NFV网络性能。
2. SDN也可以通过使用通用硬件作为SDN的控制器和服务交换机以虚拟化形式实现。
3. **结论：以移动网络，NFV是网络演进的主要架构。在一些特定场景，将引入SDN。**

SDN与NFV对比：

| 类型     | SDN                                      | NFV                                                |
| :------- | :--------------------------------------- | :------------------------------------------------- |
| 主要主张 | 转发与控制分离，控制面集中，网络可编程化 | 将网络功能从原来专用的设备移到通用设备上。         |
|          | 校园网，数据中心、云。                   | 运营商网络                                         |
|          | 商用服务器和交换机                       | 专用服务器和交换机                                 |
|          | 云资源调度和网络                         | 路由器、防火墙、网关、CND、广域网加速器、SLA保证等 |
| 通用协议 | OpenFlow                                 | 尚没有                                             |
|          | ONF（Open Networking Forun）组织         | ETSI NFV工作组                                     |

- NFV是具体设备的虚拟化，将设备控制平面运行在服务器上，这样设备是开放的兼容的。
- SDN是一种全新的网络架构，SDN的思想是取消设备控制平面，由控制器统一计算，下发流表，SDN是全新的网络架构。
- NFV和SDN是高度互补关系，但并不互相依赖。网络功能可以在没有SDN的情况下进行虚拟化和部署，然而这两个理念和方案结合可以产生潜在的、更大的价值。
- 网络功能虚拟化（NFV）的目标是可以不用SDN机制，仅通过当前的数据中心技术去实现。但从方法上有赖于SDN提议的控制和数据转发平面的分离，可以增强性能、简化与已存在设备的兼容性、基础操作和维护流程。
- NFV可以通过提供给SDN软件运行的基础设施的方式来支持SDN。而且，NFV和SDN在都利用用基础的服务器、交换机去达成目标，这一点上是很接近

#### VRF

Virtual Routing Forwarding

Tips: Virtualization is VRF in the router, VLAN in the switch, trunk (dot1q tagging) on the Ethernet link, context or VDOM on the firewall and VM on the server.

> 虚拟化 是 VRF之于路由器， VLAN之于交换机，trunk之于以太网连接，VDOM之于防火墙，VM之于服务器 

VRF将一个路由器分隔成两个，使其用于独立的路由表，类似VLAN隔离广播域。

场景如下，可以补个图：

假设PC1与R2这一侧的网络属于一个独立的业务；PC2与R3这一侧的网络属于另一个独立的业务，由于设备资源有限或者其他方面的原因，这两个独立的业务的相关节点连接在R1上，也就是同一台设备上。那么在完成相关配置后，R1的路由表如上图所示。
现在如果PC1要发一个数据包到2.2.2.2，那么这个数据包在到达R1后，R1就会去查看自己的路由表，发现有一条2.2.2.0/24的路由匹配，因此将这个IP包从GE0/0/2口转发给192.168.100.2。这是没有问题的，然而如果PC1要访问3.3.3.0/24网络呢？也是无压力的，因为数据包到达R1后，她照样查找路由表结果发现有匹配的路由，因此将数据包转给R3。但是实际上，从业务的角度考虑，我们禁止PC1访问3.3.3.0/24网络。

VRF1及VRF2有了自己的接口，也有了自己的路由表。并且相互之间是隔离的

#### H3C VNF Manager： B/S arch, mark

VNF Manager 配置 1G以上就行--

H3C 公司推出了一系列 VNF 产品，如 VSR、vBRAS、vFW、vLB 等，通过 VNF Manager（简称 VNFM）可以对这些 VNF 产品进行管理。

用户不需要安装客户端软件，使用浏览器即可访问 VNF Manager。

H3C VNF Manager支持用户通过页面创建、复制、启动、重启、停止、删除VNF。在VNF的创建过程中，用户可以灵活选择Flavor、镜像文件、网络绑定关系，无需关注主机所使用的虚拟平台。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/h3c-vnf-manager.jpg)

#### VNF Manager Agent + SR-IOV

Agent是真正提供 VNF的机器，这个机器需要挺高配置的，内存4G以上。

两台服务器安装NFV Manager Agent，其中一台安装NFV Manager用于管理两台服务器创建VSR等，NFV Manager也可以单独安装在一台虚机或服务器上。保证NFV Manager和两台Agent的管理口地址可达。

SR-IOV(Single Root Input/Output Virtualization) 是一项硬件虚拟化技术，它的目的是将一个 PCIe 设备（例如网卡），虚拟成多个互相隔离的设备，提供给不同的使用者（例如虚拟机）使用。

SR-IOV 将设备虚拟成多个 VFs 之后，每个 VF 可以直接 pass through 给虚拟机使用，虚拟机以**独占**的方式直接操作 VF，就像直接操作物理设备一样。一般来说，使用这种方式需要启用 Intel 虚拟化技术 VT-d。



#### 基于SDN的NFV解决方案

SDN控制器北向对接云平台接收配置信息，根据云平台需求计算流表，南向下发流表至CE设备。

> CE（Customer Edge），用户边缘路由器。

SDN控制器通过NFV实现，底层为E9000/RH2288服务器，平台为SUSE Linux系统，上层为VRP通用路由平台，VRP负责计算转发表。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/h3c-sdn-overlay.jpg)



### Openstack + SDN + NFV: 实践

前置条件：

1. SDN 控制器 VCF Controller已经部署好，且物理网元、网关等已经配置
2.  NFV Manager已经安装部署
3. Openstack 使用

实际效果：

1. 在Openstack的dashboard页面创建租户tenant1网络
2. 登录VCFC集群北向地址，[虚拟网络/租户管理]页面，可以看到刚才创建的tenant1项目已经同步到控制器上。
3. 在[承载网络/NFV网元]，看到NFV资源列表中新增了2台NFV资源，分别为tenant1租户的网关及服务资源。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/h3c-vnf-manager.jpg)



和openstack连动

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/h3c-openstack-sdn-vnf.jpg)



 在该组网图中，两台VCF Controller作为SDN网络控制器组成集群，集群地址为192.168.222.240/24；OpenStack提供云管理平台，**compute1和compute2为计算节点承载虚拟机业务，虚拟机为用户实际业务承载体**；NFV-Manager用于管理NFV资源，NFV资源作为租户的VXLAN网关以及服务资源。



### Service Mesh



### 引用

1. https://www.zhaohuabing.com/post/2019-10-26-what-can-service-mesh-learn-from-sdn/
2. https://drobp.github.io/2019/08/03/%E5%8D%97%E5%90%91%E6%8E%A5%E5%8F%A3/