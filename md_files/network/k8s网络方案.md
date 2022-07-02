## k8s网络方案

因为容器的网络发展复杂性就在于它其实是寄生在 Host 网络之上的。从这个角度讲，可以把容器网络方案大体分为 Underlay/Overlay 两大派别：

- Underlay 的标准是它与 Host 网络是同层的，从外在可见的一个特征就是它是不是使用了 Host 网络同样的网段、输入输出基础设备、容器的 IP 地址是不是需要与 Host 网络取得协同（来自同一个中心分配或统一划分）。这就是 Underlay；
- Overlay 不一样的地方就在于它并不需要从 Host 网络的 IPM 的管理的组件去申请 IP，一般来说，它只需要跟 Host 网络不冲突，这个 IP 可以自由分配的。

 举例：

Underlay：Calico-bgp 、Flannel-gw、macvlan、ipvlan等等

其中Calico-bgp 和 Flannel-gw不是完全依赖host 网络，它们是通过路由方式实现pod互连，要求Host必须在同一个二层网络即可以下一跳直达。

macvlan和ipvlan应该只有路由可达即可...

Overlay：Flannel-Vxlan/UDP 和Calico-ipip

### OSI和TCP/IP基础

上层到下层：增加协议头

下层到上层：去除协议头

 每一层的协议构成： 
 Packet = Protocol Header + Payload
 Payload 指传入这一层的数据内容，比如：
 TCP Segment = TCP header + HTTP data

IP Header 永远在 TCP Header 前面

L2的以太网帧：Ethernet frames的Payload包含了IP header和TCP header等上层协议数据。

在802.3标准里，规定了一个以太帧的数据部分(Payload)的最大长度是1500个字节，这个数也是你经常在网络设备里看到的MTU。

我惯性错误是： Payload包含的下层的data。正确的答案是： Payload永远承载的上层的协议的全部数据。

所以，才会有： MTU = MSS + TCP Header + UDP Header

**MTU**：maximum transmission unit，最大传输单元，由硬件规定，如以太网的MTU为1500字节。

**MSS**：maximum segment size，最大分节大小，为TCP数据包每次传输的最大数据分段大小，一般由发送端向对端TCP通知对端在每个分节中能发送的最大TCP数据。MSS值为MTU值减去IPv4 Header（20 Byte）和TCP header（20 Byte）得到即1460。

这个错误的想法，是我没有真正理解OSI模型划分。这导致了我很多概念无法弄清比如：Overlay的L2-in-L4.

### k8s固定IP方案

#### 静态保持

cni插件自动分配，静态保持，应用（deployment）创建出来自动分配一个IP，这个应用升级迭代IP不会改变，但是扩容缩容IP会波动。

#### 手动指定

手动固定IP，创建应用（deployment）时，手动输入指定的容器IP，应用升级过程中IP保持不变。

扩容IP怎么指定呢，需要设计来实现

### 网络收敛

1. 网络依赖小：仅需NVE间IP可达
2. 环路避免：隧道间水平分割，IP overlay TTL
3. 高效转发：基于IP路由 SPF，ECMP
4. 快速收敛：网路变化实时侦听，全网拓扑毫秒收敛
5. 虚拟化：overlay + VNI， 容量16M
6. 部署灵活：物理设备和vswitch均部署

- VxLAN overlay层转发与VLAN一样



### Overlay: 四层转发

Overlay 的核心是L2-in-L4,即通过SDN技术建立在另一网络之上的计算机网络。

> bgp的ipip也是overlay.

注意哟，L2 Ethernet Frames包喊了L3 和 L4等上层的全部数据哟，其他MTU为1500。

无论是UDP还是Vxlan封包模式，它都是通过UDP协议在flannel节点间进行转发的。

创建容器网络的流程就是：kubelet ——> flannel ——> flanneld。

工作原理
很容易混淆几个东西。我们通常说的Flannel（coreos/flannel），其实说的是flanneld。大家都知道Kubernetes是通过CNI标准对接网络插件的，但是当你去看Flannel（coreos/flannel）的代码时，并没有发现它实现了CNI的接口。如果你玩过其他CNI插件，你会知道还有一个二进制文件用来供kubele调用，并且会调用后端的网络插件。对于Flannel（coreos/flannel）来说，这个二进制文件是什么呢？git repo在哪里呢？

这个二进制文件就对应宿主机的/etc/cni/net.d/flannel，它的代码地址是https://github.com/containernetworking/plugins，最可恨的它的名字就叫做flannel，为啥不类似contiv netplugin对应的contivk8s一样，取名flannelk8s之类的。

#### UDP实现方式

数据流：container -（用户到内核）-> docker0 --> flannel0 -（内核到用户）-> flaneld -（用户到内核）->eth0

L2-in-L4的数据包封装由flanneld在用户态完成，所以基于UDP实现的Overlay性能极差。

UDP 模式，是 Flannel 项目最早支持的一种方式，也是性能最差的一种模式，如今已被弃用，但是对我们理解集群的网络还是有帮助的 

性能差是用于内核态和用户态切换太多。

该模式的主要通信过程如下

1. 容器A需要发送一个数据包到B(用户态)
2. 首先还是会到docker0网口（**用户态->内核态**）
3. flannel在运行的时候会在宿主机创建一系列的路由规则(xx.xx.xx.xx/16 dev flannel0，前提是docker0的网桥网段范围由flannel控制，dockerd启动的时候添加参数即可，FLANNEL_SUBNET=100.96.1.1/24 dockerd --bip=$FLANNEL_SUBNET ）
4. A需要访问的网段，会走宿主机的路由，他的下一跳是flannel0（内核态）
5. flannel0这里会把数据包交给应用程序 flanneld:8125 （**内核态->用户态**）
6. flanneld 进行 UDP 封包之后重新进入内核态，将udp的包通过宿主机的发出去（**用户态->内核态**）

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/flannel-upd-1.png)

UDP模式内核、用户态示意图（

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/flannel-udp-2.png)

可以看出该模式仅在ip包发出的过程中，就需要经过三次用户态与内核态之间的数据拷贝，这个代价是比较高的，所以被弃用

#### VxLan实现方式

```bash
#在master虚拟机中执行
sudo ip link add vxlan1 type vxlan id 88 remote 192.168.88.8 dstport 4789 dev eth1
sudo ip link set vxlan1 up
sudo ip addr add 10.0.0.106/24 dev vxlan1
#下面的两条命令到是我了查看配置
#路由表多了vxlan1，并通过vxlan1转发
[vagrant@master ~]$ ip route | grep vxlan1
10.0.0.0/24 dev vxlan1 proto kernel scope link src 10.0.0.106
#fdb多了vxlan1,dst的vtep是192.168.88.9
[vagrant@master ~]$ bridge fdb | grep vxlan1
00:00:00:00:00:00 dev dev vxlan1 dst 192.168.88.9 via eth1 self permanent
#在slave1虚拟机中执行
sudo ip link add vxlan1 type vxlan id 88 remote 192.168.88.9 dstport 4789 dev eth1
sudo ip link set vxlan1 up
sudo ip addr add 10.0.0.107/24 dev vxlan1
```

直接借助linux内核就完成了vxlan的封装。

数据流向：  containers-（用户到内核）->docker0-->flannel.1-->eth0

Kernel集成了Vxlan的模块，高效。

https://www.kernel.org/doc/Documentation/networking/vxlan.txt

https://www.slideshare.net/enakai/how-vxlan-works-on-linux

本质： 检测到vxlan header vni，支持vxlan协议的机器就会接受并对他解包。



VxLAN 全称是 Visual eXtensible Local Area Network（虚拟扩展本地局域网），从名字上就知道，这是一个 VLAN 的扩展协议。

VxLAN 本质上是一种隧道封装技术。它使用 TCP/IP 协议栈的惯用手法——封装/解封装技术，将 L2 的以太网帧（Ethernet frames）封装成 L4 的 UDP 数据报（datagrams），然后在 L3 的网络中传输，效果就像 L2 的以太网帧在一个广播域中传输一样，实际上是跨越了 L3 网络，但却感知不到 L3 网络的存在。

如下图所示，左右两边是 L2 广播域，中间跨越一个 L3 网络，VTEP 是 VxLAN 隧道端点(VxLAN Tunnel Point)，当 L2 以太网帧到达 VTEP 的时候，通过 VxLAN 的封装，跨越 L3 层网络完成通信，由于 VxLAN 的封装"屏蔽"了 L3 网络的存在，所以整个过程就像在同一个 L2 广播域中传输一样。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/L2-over-L3.png)

下图是 vxlan 协议的报文，白色的部分是虚拟机发送报文（二层帧，包含了 MAC 头部、IP 头部和传输层头部的报文），前面加了 vxlan 头部用来专门保存 vxlan 相关的内容，在前面是标准的 UDP 协议头部（UDP 头部、IP 头部和 MAC 头部）用来在底层网路上传输报文。

**Original Packet** and **Outer Packet**

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vnet-vxlan.png)

从这个报文中可以看到三个部分：

1. 最外层的 UDP 协议报文用来在底层网络上传输，也就是 vtep
   之间互相通信的基础
2. 中间是 VXLAN 头部，vtep 接受到报文之后，去除前面的 UDP 协议部分，根据这部分来处理 vxlan 的逻辑，主要是根据 VNI 发送到最终的虚拟机
3. 最里面是原始的报文，也就是虚拟机看到的报文内容

报文各个部分的意义如下：

- VXLAN header：vxlan 协议相关的部分，一共 8 个字节

  - VXLAN flags：标志位

    在 Flannel 中，VNI 的默认值是 1，这也是为何，宿主机上的 VTEP 设备都叫作 flannel.1 的原因，这里的“1”，其实就是 VNI 的值。

  - Reserved：保留位

  - VNID：24 位的 VNI 字段，这也是 vxlan 能支持千万租户的地方

  - Reserved：保留字段

- UDP 头部，8 个字节

  - UDP 应用通信双方是 vtep 应用，其中目的端口就是接收方 vtep 使用的端口，IANA 分配的端口是 4789

- IP 头部：20 字节

  - 主机之间通信的地址，可能是主机的网卡 IP 地址，也可能是多播 IP 地址

- MAC 头部：14 字节

  - 主机之间通信的 MAC 地址，源 MAC 地址为主机 MAC 地址，目的 MAC 地址为下一跳设备的 MAC 地址

可以看出 vxlan 协议比原始报文多 50 字节的内容，这会降低网络链路传输有效数据的比例。vxlan 头部最重要的是 VNID 字段，其他的保留字段主要是为了未来的扩展，目前留给不同的厂商用这些字段添加自己的功能。

在这种模式下，数据包的流转依然需要经过flanneld，**但是处理机制不同了  containers->docker0->flannel.1->eth0** 



基于VTEP设备进行通信的示意图：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/flannel-vxlan-1.png)

1. 图中每台宿主机上名叫 flannel.1 的设备，就是 VXLAN 所需的 VTEP 设备，它既有 IP 地址，也有 MAC 地址。

2. 现在，我们的 container-1 的 IP 地址是 10.1.15.2，要访问的 container-2 的 IP 地址是 10.1.16.3。 

3. flannel在运行的时候会在宿主机创建一系列的路由规则(xx.xx.xx.xx/24 dev flannel.1）

4. 为了能够将“原始 IP 包”封装并且发送到正确的宿主机，VXLAN 就需要找到这条“隧道”的出口，即：目的宿主机的 VTEP 设备->给这个原始数据包加上目的MAC（二层封装）

5. 那么这个目的MAC的地址是什么呢，前面说道VTEP设备有ip和MAC，那么有ip想知道MAC就是ARP实现的东西了，在VTEP创建好后，会在自己的ARP表中记录，ip neigh show dev flannel.1 可以看到

   > -/- 每个flannel.1 的 MAC通过ectd 下发到各个宿主机上？？？
   >
   > ```bash
   > ning@ubuntu-k8s-master:~$ ip r
   > default via 192.168.10.2 dev ens33 onlink
   > 10.244.0.0/24 dev cni0  proto kernel  scope link  src 10.244.0.1
   > 10.244.1.0/24 via 10.244.1.0 dev flannel.1 onlink
   > 10.244.2.0/24 via 10.244.2.0 dev flannel.1 onlink
   > 192.168.10.0/24 dev ens33  proto kernel  scope link  src 192.168.10.10
   > ##
   > ning@ubuntu-k8s-master:~$ ip neigh show
   > 192.168.10.1 dev ens33 lladdr 00:50:56:c0:00:08 REACHABLE
   > 192.168.10.2 dev ens33 lladdr 00:50:56:e6:25:ce STALE
   > 192.168.10.20 dev ens33 lladdr 00:0c:29:a0:73:8c REACHABLE
   > 192.168.10.30 dev ens33 lladdr 00:0c:29:b8:32:df REACHABLE
   > 10.244.2.0 dev flannel.1 lladdr 16:09:80:5c:db:cc PERMANENT
   > 10.244.0.2 dev cni0 lladdr 06:6b:d3:3e:76:18 REACHABLE
   > 
   > ## fdb
   > ning@ubuntu-k8s-master:~$ bridge fdb show flannel.1 | grep 16:09:80:5c:db:cc
   > 16:09:80:5c:db:cc dev flannel.1 dst 192.168.10.30 self permanent
   > ```
   >
   > end

初步封包如下：

vxlan报文是在原有报文的基础上再次进行封装，以实现三层传输二层的目的。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vxlan-data.png)

如上图所示，原有封装后的报文成为vxlan的data部分，vxlan header为vni，ip层header为源和目的vtep地址，链路层header为源vtep的mac地址和到目的vtep的下一个设备mac地址。

> 检测到vxlan header vni，支持vxlan协议的机器就会接受并对他解包。

**因为它不能再node网络上传播，所以叫他Inner Etherne**t

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/flannel-vxlan-2.png)

6. 但是，**上面提到的这些 VTEP 设备的 MAC 地址，对于宿主机网络来说并没有什么实际意义**。所以上面封装出来的这个数据帧，并不能在我们的宿主机二层网络里传输。为了方便叙述，我们把它称为“内部数据帧”（Inner Ethernet Frame）。

   **为什么说这个vtep设备的mac在宿主机网络是无效的。**

   因为，宿主机的物理网络没有这个IP地址啊，不会从网卡出去。自然不会包有在物理网络中流通的下一步：即将目标mac会被替换成下一跳网关地址。

   卧槽，原来如此，IP不变，mac一直在变，即物流的卡车或者人一直在换，目的地会一直保证不变。

   > vtep全称vxlan tunnel endpoint，vxlan可以抽象的理解为在三层网络中打通了一条条隧道，起点和终点的两端就是vetp。
   >
   > ![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vxlan-vtep.png)
   >
   > end

7. 所以接下来，Linux 内核还需要再把“内部数据帧”进一步封装成为宿主机网络里的一个普通的数据帧，好让它“载着”“内部数据帧”，通过宿主机的 eth0 网卡进行传输。

8. 我们把这次要封装出来的、宿主机对应的数据帧称为“外部数据帧”（Outer Ethernet Frame）。

9. Linux 内核会在“内部数据帧”前面，加上一个特殊的 VXLAN 头，用来表示这个实际上是一个 VXLAN 要使用的数据帧。

10. 而这个 VXLAN 头里有一个重要的标志叫作VNI,，它是 VTEP 设备识别某个数据帧是不是应该归自己处理的重要标识。而在 Flannel 中，VNI 的默认值是 1，这也是为何，宿主机上的 VTEP 设备都叫作 flannel.1 的原因，这里的“1”，其实就是 VNI 的值。

11. 然后，Linux 内核会把这个数据帧封装进一个 UDP 包里发出去。

12. 不过，一个 flannel.1 设备只知道另一端的 flannel.1 设备的 MAC 地址，却不知道对应的宿主机地址是什么。

13. 在这种场景下，flannel.1 设备实际上要扮演一个“网桥”的角色，在二层网络进行 UDP 包的转发。而在 Linux 内核里面，“网桥”设备进行转发的依据，来自于一个叫作 FDB（Forwarding Database）的转发数据库。

14. 这个 flannel.1“网桥”对应的 FDB 信息，也是 flanneld 进程负责维护的。它的内容可以通过 bridge fdb 命令查看到，如下所示：

```bash
# 在 Node 1 上，使用“目的 VTEP 设备”的 MAC 地址进行查询
$ bridge fdb show flannel.1 | grep 5e:f8:4f:00:e3:37
5e:f8:4f:00:e3:37 dev flannel.1 dst 10.168.0.3 self permanent
```

这样udp的目的地就有了

15. UDP 包是一个四层数据包，所以 Linux 内核会在它前面加上一个 IP 头，即原理图中的 Outer IP Header，组成一个 IP 包。并且，在这个 IP 头里，会填上前面通过 FDB 查询出来的目的主机的 IP 地址，即 Node 2 的 IP 地址 10.168.0.3。

16. 最后，Linux 内核再在这个 IP 包前面加上二层数据帧头，即原理图中的 Outer Ethernet Header，并把 Node 2 的 MAC 地址填进去。这个 MAC 地址本身，是 Node 1 的 ARP 表要学习的内容，无需 Flannel 维护。这时候，我们封装出来的“外部数据帧”的格式，如下所示（张磊图）：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/flannel-vxlan-3.png)

另外一个示意图：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/flannel-vxlan-4.gif)

17. 接下来，Node 1 上的 flannel.1 设备就可以把这个数据帧从 Node 1 的 eth0 网卡发出去。显然，这个帧会经过宿主机网络来到 Node 2 的 eth0 网卡。这时候，Node 2 的内核网络栈会发现这个数据帧里有 VXLAN Header，并且 VNI=1。所以 Linux 内核会对它进行拆包，拿到里面的内部数据帧，然后根据 VNI 的值，把它交给 Node 2 上的 flannel.1 设备。而 flannel.1 设备则会进一步拆包，取出“原始 IP 包”。接下来就回到了我在前面提到的单机容器网络的处理流程。最终，IP 包就进入到了 container-2 容器的 Network Namespace 里。

### hostgateway: underlay

host-gw，也就是修改主机路由的方式实现容器间的跨主机通信，此时每个Node都相当于一个路由器，作为容器的网关，负责容器的路由转发。

host-gw的方式相对overlay由于少了vxlan的封包拆包过程，直接路由数据包到目的地址，因此性能相对要好。不过正是由于它是通过添加静态路由的方式实现，每个宿主机相当于是容器的网关，因此每个宿主机必须在同一个子网内，否则跨子网由于链路层不通导致无法实现路由。

### flannel cni and flannel

cni plugins中的flannel是开源网络方案flannel的“调用器”。这也是flannel网络方案适配CNI架构的一个产物。为了便于区分，以下我们称cni plugins中的flannel 为`flanenl cni`。

我们知道flannel是一个容器的网络方案，通常使用flannel时，node上会运行一个daemon进程：flanneld，这个进程会返回该node上的flannel网络、subnet，MTU等信息。并保存到本地文件中。

如果对flannel网络方案有一定的了解，会知道他在做网络接口配置时，其实干的事情和bridge组件差不多。只不过flannel网络下的bridge会跟flannel0网卡互联，而flannel0网卡上的数据会被封包（udp、vxlan下）或直接转发（host-gw）。

而`flannel cni`做的事情就是：

- 执行ADD命令时，`flannel cni`会从本地文件中读取到flanneld的配置。然后根据命令的参数和文件的配置，生成一个新的cni配置文件（保存在本地，文件名包含容器id以作区分）。新的cni配置文件中会使用其他cni组件，并注入相关的配置信息。之后，`flannel cni`根据这个新的cni配置文件执行ADD命令。
- 执行DEL命令时，`flannel cni`从本地根据容器id找到之前创建的cni配置文件，根据该配置文件执行DEL命令。

**也就是说`flannel cni`此处是一个flannel网络模型的委托者，falnnel网络模型委托它去调用其他cni组件，进行网络配置。通常调用的是bridge和host-local。**

