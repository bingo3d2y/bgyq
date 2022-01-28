## VPN

VPN具有以下两个基本特征：

- 专用（Private）：对于VPN用户，使用VPN与使用传统专网没有区别。VPN与底层承载网络之间保持资源独立，即VPN资源不被网络中非该VPN的用户所使用；且VPN能够提供足够的安全保证，确保VPN内部信息不受外部侵扰。
- 虚拟（Virtual）：VPN用户内部的通信是通过公共网络进行的，而这个公共网络同时也可以被其他非VPN用户使用，VPN用户获得的只是一个逻辑意义上的专网。这个公共网络称为VPN骨干网（VPN Backbone）。

利用VPN的专用和虚拟的特征，可以把现有的IP网络分解成逻辑上隔离的网络。这种逻辑隔离的网络应用丰富：可以用在解决企业内部的互连、相同或不同办事部门的互连；也可以用来提供新的业务，如为IP电话业务专门开辟一个VPN，以此解决IP网络地址不足、QoS保证、以及开展新的增值服务等问题。

在解决企业互连和提供各种新业务方面，VPN，尤其是MPLS VPN，越来越被运营商看好，成为运营商在IP网络提供增值业务的重要手段。

##### MPLS

MPLS全称Multi-Protocol Label Switching，目前来讲运营商网络中MPLS VPN被广泛的应用。所以我们现在通常讲的L3VPN什么的一般来讲都是基于MPLS的L3VPN。

一般VPN技术分为传统的VPN技术与虚拟的VPN技术；传统的VPN技术一般可以理解为基于静态的电路域的链接方式；虚拟的VPN技术，一般是指MPLS VPN，是一种基于MPLS技术的虚拟解决方案，可以实现底层标签的自动分配，在业务的提供上比传统的VPN技术更廉价，更快

##### L2 vs L3

L3VPN与L2VPN的本质区别就是运营商边界设备PE是否参与客户的路由；通俗来讲就是L3VPN对于用户来讲像一个大的路由器，需要L3VPN网络提供路由，而L2VPN对于用户则像一个交换机。



##### L2 VPN

With L2 VPN, you can stretch multiple logical networks (both VLAN and VXLAN) across geographical sites. In addition, you can configure multiple sites on an L2 VPN server.

###### QinQ

QinQ是 802.1Q in 802.1Q 的简称，是基于 IEEE 802.1Q 技术的一种比较简单的二层 VPN 协议。

通过将一层 VLAN Tag 封装到私网报文上，使其携带两层 VLAN Tag 穿越运营商的骨干网络(又称公网)，从而使运营商能够利用一个 VLAN 为包含多个 VLAN 的用户网络提供服务。

QinQ报文在运营商网络中传输时带有双层VLAN Tag：

- 内层 VLAN Tag：为用户的私网 VLAN Tag，Customer VLAN Tag (简称 CVLAN)。设备依靠该 Tag 在私网中传送报文。
- 外层 VLAN Tag：为运营商分配给用户的公网 VLAN Tag， Service VLAN Tag(简称 SVLAN)。设备依靠该 Tag 在公网中传送 QinQ 报文。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/QinQ.jpeg)

在公网的传输过程中，设备只根据外层 VLAN Tag 转发报文，而内层 VLAN Tag 将被当作报文的数据部分进行传输。

##### L3 VPN

L3VPN为切入点，对VPN进行一个由点到面的介绍。

L3VPN中部署最为广泛的就是MPLS BGP VPN了，MPLS提供公网隧道的转发，BGP提供私网路由的扩散。下图为MPLS BGP VPN的网络结构。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/142611g1gx7lglbxzwksnp.png)

这里VPN的实际目的就很明确，两侧VPNA的网络用户需要互联互通，而同侧的VPNA和VPNB网络需要相互隔离。简单介绍一下各个网元概念，CE指的是用户边缘设备，通常是用户侧的交换机、路由器或者是台主机；PE指运营商边缘设备，主要负责与CE通过路由协议进行对接，将CE侧的路由存入自己的VPN路由表中，并且将用户数据传入网络侧的隧道中去。P指的是运营商核心设备，主要负责公网的隧道转发。从各个设备的主要功能，就可以基本了解到这个L3VPN网络的基本工作原理了，CE与PE通过路由协议直接连接，PE设备存储CE的VPN与路由信息，入口PE到出口PE通过P设备则通过MPLS进行公网的转发传输，通过BGP进行路由的扩散；报文到达出口PE后通过扩散得到的路由信息发送至相应的出口CE这样就完成了VPN报文的转发；具体的报文实现上通常是对VPN报文进行两层标签的封装，外层的标签用于报文在公网上的传输，而内层的标签用于指定报文送达哪个VPN网络。也正是由于这两层标签的封装，实际讲用户的标识丰富成为了IP+VPN的二维信息，这样就解决了不同的用户私网中可能存在的地址重叠的问题。下图为VPN报文转发的简图。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/142632z6c8he4zhfvrlsuh.png)

图中，10.1.5.1这一层表示的为报文的IP层；15362这一层表示的是2层VPN标签封装中的内侧标签，实际意义为VPNA的标志；1024与3这一层表示的是2层VPN标签封装中的外层标签，用于PEB到PEA通过P传输时的MPLS标签；报文到达PEA后我们可以得到VPNA的信息与报文的IP信息，这样就可以精确的送达到VPNA网络中的目的地，完成报文的传输。