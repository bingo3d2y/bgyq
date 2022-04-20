## MPLS VPN-draft

### VPN实现地址重叠

现实情况下，每个VPN在PE上，有以下独立的内容：

- 独立的路由转发表
- CE与PE相连的端口
- 与PE进行路由交换的进程，比如IGP
- 与VPN相关的路由器变量

除了最后一点，前三点就是一个常规路由器应该做的工作。

因此，在PE上，对应每一个VPN，都有一个virtual router与之连接。另一方面，PE还需要与服务提供商网络的P路由器连接，以提供MPLS Tunnel。因此，PE路由器还有一个非虚拟的路由器，其实就是原来的PE。



### What is MPLS?

MPLS全称是Multi-Protocol Label Switching，直译过来就是多协议标签交换技术。

传统的路由决策，路由器需要对网络数据包进行解包，再根据目的IP地址计算归属的FEC。而MPLS提出，当网络数据包进入MPLS网络时，对网络数据包进行解包，计算归属的FEC，生成标签（Label）。当网络数据包在MPLS网络中传输时，路由决策都是基于Label，路由器不再需要对网络数据包进行解包。并且Label是个整数，以整数作为key，可以达到O(1)的查找时间。大大减少了路由决策的时间。这里的Label就是MPLS里面的L。需要注意的是Label在MPLS网络里面，是作为网络数据包的一部分，随着网络数据包传输的。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/MPLS-label.jpg)

也就是说，在MPLS网络里面，数据被封装在了盒子里，上面贴了标签，每个经手的人只需要读标签就知道盒子该送到哪。而传统的路由网络里面，每个经手的人都需要打开盒子，看看里面的内容，再决定送往哪。

MPLS网络，是一个由相连的，支持MPLS的设备组成的网络。打上MPLS标签的数据可以在这个网络里面传输。MPLS的核心就是，**一旦进入了MPLS网络，那么网络数据包的内容就不再重要，路由决策（包括FEC归属的计算，next hop的查找）都是基于Label来进行的。**

在MPLS VPN中，CE、PE、P的概念:

* CE（Customer Edge）设备：用户网络边缘设备，有接口直接与SP相连。CE可以是路由器或交换机，也可以是一台主机。CE“感知”不到VPN的存在，也不需要必须支持MPLS。
* PE（Provider Edge）路由器：服务提供商边缘路由器，是服务提供商网络的边缘设备，与用户的CE直接相连。在MPLS网络中，对VPN的所有处理都发生在PE上。
* P（Provider）路由器：服务提供商网络中的骨干路由器，不与CE直接相连。P设备只需要具备基本MPLS转发能力。
* VC：Virtual circuit，服务提供商（Service provider）向用户提供模拟的线路，用户基于模拟线路来实现跨地域互连。

* 外层标签（称为Tunnel标签）用于将报文从一个PE传递到另一个PE；
* 内层标签（称为VC标签）用于区分不同VPN中的不同连接；
* 接收方PE根据VC标签决定将报文转发给哪个CE。

### Why is MPLS？

MPLS的提出是为了实现快速路由交换，但随着设备的发展，这方面的优势已经不存在了。真正使MPLS的流行却是别的原因。

为什么SP这么钟爱MPLS？

- 统一的MPLS骨干网提供多种L2，L3服务。SP只需要一个MPLS骨干网，即可以提供L3 VPN，又可以提供兼容ATM，Frame Relay，Ethernet的L2 VPN。
- 向前兼容。用户支持Frame Relay或者ATM的旧设备，仍然能在MPLS网络上使用，这得益于VPWS/AToM。
- 支持peer-to-peer模式的L3 VPN。提供稳定，安全，易扩展的VPN连接。
- 核心设备负载轻。从MPLS L2/L3 VPN的实现来看，不论是Targeted LDP，还是MP-BGP，都是运行在PE之间，P路由器不需要运行这些路由信息交换进程。核心设备，也就是P路由器，只需要运行一些IGP和LDP即可，而这些IGP和LDP本身带来的负载也较小。
- MPLS Traffic engineering。这比基于IP 的Traffic engineering效率更高。
- 较好的QoS支持。



MPLS L2VPN提供基于MPLS（Multiprotocol Label Switching，多协议标签交换）网络的二层VPN服务，使运营商可以在统一的MPLS网络上提供基于不同数据链路层的二层VPN，包括ATM、FR、VLAN、Ethernet、PPP等。

简单来说，MPLS L2VPN就是在MPLS网络上透明传输用户二层数据。从用户的角度来看，MPLS网络是一个二层交换网络，可以在不同节点间建立二层连接。

以ATM为例，每个用户边缘设备（Customer Edge，CE）配置一条ATM虚电路（Virtual Circuit，VC），通过MPLS网络与远端CE相连，这与通过ATM网络实现互联类似。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/MPLS-L2VPN.png)

由于MPLS VPN由单个网络服务提供商拥有和运营，因此提供商可以管理网络，以便您的流量按照您的策略和优先级进行操作。

VPN、Site、VPN之间的关系如下：

* VPN是多个Site的组合。一个Site可以属于多个VPN。
* 每一个Site在PE上都关联一个VPN实例。VPN实例综合了它所关联的Site的VPN成员关系和路由规则。多个Site根据VPN实例的规则组合成一个VPN
* VPN实例与VPN不是一一对应的关系，VPN实例与Site之间存在一一对应的关系。

VPN实例也称为VPN路由转发表VRF（VPN Routing and Forwarding table）。PE上存在多个路由转发表，包括一个公网路由转发表，以及一个或多个VPN路由转发表。









#### MPLS VPN和传统VPN

http://net.zhiding.cn/network_security_zone/2009/0105/1302035.shtml

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vpn运营模式分类.jpg)

IPSec VPN来说，配置将成为问题，供应商必须配置好每个IPSec隧道。配置单一的一个IPSec隧道不成问题。但网络结点数量增大时，问题就来了。在建立全网状的布局时，情况最糟，上例中，配置一个由21个结点组成的网络需费时数天。对于服务供应商来说，日常维护的难度也很大。

### MPLS L3VPN

MPLS VPN uses L3 on underlay network.

MPLS VPN carrier grade technical skills are required to configure thorough CLI while VXLAN in SDN is used to automate configuration.

MPLS VPN技术，它本身是可以解决不同VPN的地址重叠问题。

MPLS L3 VPN又称为BGP/MPLS IP VPN，由[RFC4364](https://link.zhihu.com/?target=https%3A//tools.ietf.org/html/rfc4364)定义。这不是某个特定的而技术或者协议，而是一些技术的集合应用。

MPLS L3VPN是服务提供商VPN解决方案中一种基于PE的L3VPN技术，它使用BGP在服务提供商骨干网上发布VPN路由，使用MPLS在服务提供商骨干网上转发VPN报文。

和其他VPN实现就多了一个多了一个MP-BGP。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vpn-mpls-L3-arc.png)

VPN提供了在公共基础设施上进行私有网络服务的传播，即就是CE1要能访问到CE2，CE3要能访问到CE4，同时CE1不能访问CE3，CE4（类似的对CE2，3，4都只能访问同一个VPN内的私网）。

#### 控制层面/Control plane

所谓控制层面，就是路由数据怎么传递的？例如，CE2里面是怎么就有了CE1网络的路由，并且能正常工作。

#### VRFs:+1:

全称VPN Routing and Forwarding tables。

MPLS L3 VPN引以为傲的一个地方就是支持不同VPN中IP地址重叠。为了实现这个功能，MPLS L3 VPN里面做了很多工作。VRFs就是其中之一。

一个PE可能同时连着多个VPN，当多个VPN中存在重复IP地址的时候，且目的地是重复IP地址的网络流量到达PE，PE怎么区分该网络流量属于哪个VPN？PE通过管理多个路由转发表来解决，每个VPN对应一个路由转发表。这样可以对重复的地址指定不同的转发出口。

现实情况下，每个VPN在PE上，有以下独立的内容：

- 独立的路由转发表
- CE与PE相连的端口
- 与PE进行路由交换的进程，比如IGP
- 与VPN相关的路由器变量

除了最后一点，前三点就是一个常规路由器应该做的工作。因此，在PE上，对应每一个VPN，都有一个virtual router与之连接。另一方面，PE还需要与服务提供商网络的P路由器连接，以提供MPLS Tunnel。因此，PE路由器还有一个非虚拟的路由器，其实就是原来的PE。

### MPLS L2VPN

MPLS L2VPN提供基于MPLS（Multiprotocol Label Switching，多协议标签交换）网络的二层VPN服务，使运营商可以在统一的MPLS网络上提供基于不同数据链路层的二层VPN，包括ATM、FR、VLAN、Ethernet、PPP等。

简单来说，MPLS L2VPN就是在MPLS网络上透明传输用户二层数据。从用户的角度来看，MPLS网络是一个二层交换网络，可以在不同节点间建立二层连接。

#### VLL

VLL（ Virtual Leased Line，虚拟租用线，也被称为VPWS）是对传统租用线业务的仿真VLL技术是一种点到点的虚拟专线技术，能够支持几乎所有的链路层协议。VLL简单的理解就好比在客户之间拉了一条网线。分为Kompella和Martini两种实现方式，但Kompella工作组已经解散，Martini也更新为PWE3技术。

#### PWE3

PWE3（Pseudo-Wire Emulation Edge to Edge）是端到端仿真业务，PWE3是VLL的一种实现方式，是对Martini协议的扩展。PWE3扩展了新的信令，减少了信令的开销，规定了多跳的协商方式，使得组网方式更加灵活。在IPRAN网络大量使用PWE3来承载TDM等二层业务。

#### VPLS

VPLS（Virtual Private LAN Service）是一种基于MPLS和以太网技术的二层VPN技术。具体分为Juniper主导的Kompella方式和Cisco主导的Martini方式，是将公网模拟为二层交换机。适合于点到多点的业务，简单的说VPLS就是一个超级交换机。

VPLS提供虚拟局域网服务。从本质上来说，VPLS是Ethernet over MPLS的多点接入版本。 而Ethernet over MPLS本质上又是AToM的一种，只是对应的L2协议就是Ethernet。

在VPLS中，MPLS骨干网将模拟一个Ethernet交换机。这样看起来，尽管各个CE可能相隔了几百甚至几千公里，但是都像是接入了一个巨大的交换机。而每个PE面向用户的接口，就像是一个交换机的端口。由于模拟了一个交换机，那自然也会完成一些交换机的工作，例如MAC learning， Aging，Flood等等。

VPLS需要一个full mesh的结构。具体的说就是所有PE router之间需要full mesh的pseudowire连接。虽然对用户来说，不用关心这个，并且使用简单，但是对于SP来说，运维困难，负担较重。由RFC4762定义的基于LDP的VPLS，要求手工配置full mesh，SP的管理员估计要不开心了，RFC4761定义的基于BGP的VPLS，带有自动发现，能够节省full mesh的配置。

刚刚说了VPLS模拟交换机工作，在VPLS下，BUM（Broadcast, Unknown unicast, Multicast）会广播出来。对于某一个具体的PE router，会这么处理BUM frame，首先会广播到与当前VPLS instance相关的所有CE接口（Attachment Circuit）上，同时也会广播到与当前VPLS instance关联的所有pseudowire上，通过pseudowire发送到远端的PE，远端PE再发送到对应的CE接口。

控制层面和转发层面与之前介绍AToM是一样。所以，VPLS，就是一个full mesh版的Ethernet over MPLS。VPLS的架构图如下所示，下图同时也是一个VPLS Flood Unknown Frame的示意图：



![img](https://pic3.zhimg.com/80/v2-bc1cc8db462e1a48586f1d6e712b4956_1440w.png)

### 引用

1. https://zhuanlan.zhihu.com/p/27232535
2. http://www.h3c.com/cn/d_201212/922115_30005_0.htm