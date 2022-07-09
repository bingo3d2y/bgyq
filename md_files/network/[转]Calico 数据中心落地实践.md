## [转]Calico 数据中心落地实践:heavy_check_mark:

### cni抉择

2020 年的 CNI 性能测试报告《Benchmark results of Kubernetes network plugins (CNI) over 10Gbit/s network》 

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/k8s-cni-benchmark-2020.png)

在对我司机房新区域的网络选型上，我们最终选择了 Calico 作为云端网络解决方案，在这里我们简单阐述下为什么选择 Calico 的几点原因：

- 支持 BGP 广播，Calico 通过 BGP 协议广播路由信息，且架构非常简单。在 kubernetes 可以很容易的实现 mesh to mesh 或者 RR 模式，在后期如果要实现容器跨集群网络通信时，实现也很容易。
- Calico配置简单，且配置都是通过 Kubernetes 中的 CRD 来进行集中式的管理，通过操作 CR 资源，我们可以直接对集群内的容器进行组网和配置的实时变更
- 丰富的功能及其兼容性，考虑到集群内需要与三方应用兼容，例如配置多租户网络、固定容器 IP 、网络策略等功能又或者与 Istio、MetalLB、Cilium 等组件的兼容，Calico 的的表现都非常不错
- 高性能， Calico 的数据面采用 HostGW 的方式，由于是一个纯三方的数据通信，所以在实际使用下性能和主机资源占用方面不会太差，至少也能排在第一梯队

结合我司机房新区域采购的是 H3C S9 系列的交换机，**支持直接在接入层的交换机侧开启路由反射器。**所以最终我们选择Calico 并以 BGP RR 的模式作为 Kubernetes 的 CNI 组件便水到渠成。

### calico bgp with ToR

在不改变 IDC 机房内部网络拓扑的情况下，接入层交换机和核心层交换机建立 BGP 连接，借助于机房内部已有的路由策略实现。针对 Node 所处的物理位置分配 PodCIDR，每个节点将 PodCIDR 通过 BGP 协议宣告给接入层交换机，实现全网通信的能力。





![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/calico-bgp-with-ToR.webp)

1. 每个接入层交换机与其管理的 Node 二层联通，共同构成一个 AS。每个节点上跑 BGP 服务，用于宣告本节点路由信息。
2. 核心层交换机和接入层交换机之间的每个路由器单独占用一个 AS，物理直连，跑 BGP 协议。核心层交换机可以感知到全网的路由信息，接入层交换机可以感知与自己直连的 Node 上的路由信息。
3. 每个 Node 上只有一条默认路由，直接指向接入层交换机。同一个接入层交换机下的 Node 通信下一跳指向对端。

#### 邻居发现

在 BGP 实现的集群网络中，经常存在节点新增和删除的情形，如果采用静态配置对等体的方式，需要频繁的操作交换机进行对等体增删的操作，维护工作量很大，不利于集群的横向扩展。为了避免手动对交换机进行操作，我们支持基于配置接入层交换机和软件层面实现的路由反射器这两种模式来动态发现 BGP 邻居。

##### 通过接入层交换机实现动态邻居发现

接入层交换机充当边界路由器，并开启 [Dynamic Neighbors](https://link.segmentfault.com/?enc=uarxWA9Zv1jlrYiRNbMeFA%3D%3D.P6is9tAKS6c5Yq7uNdsuYokIliyGGxjGHl7l34%2B4Lm9%2Bhc9l6iYRrhV%2FBh0uPwRD%2BqJfkSUga%2B8XIzC3PUEfna64zk8WM1JftOPa9%2F%2B8%2FVKJyGfXJAnuN6H3eJvCl7ailbhg9eJB26G0BJQxU5BBKMp9DgrScIG1VhmKfE0Yy9I%3D) 功能，H3C、Cisco以及华为的路由器具体开启 Dynamic Neighbors 配置请参考官方文档。Node上的 BGP 服务主动与接入层交换机建立 iBGP 连接，并宣告本地路由，接入层交换机将学习到的路由宣告给整个数据机房内部。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/ToR-dynamic-bgp.webp)

##### 通过RR实现动态邻居发现

物理交换机或者 Node 节点充当反射路由器 RR，反射路由器与接入层交换机建立 iBGP 连接，Node 节点上的 BGP 服务与反射路由器建立连接。Node上的BGP服务将本地路由宣告给 RR，RR 反射到接入层交换机，接入层交换机接着宣告给整个数据机房内部。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/RR-ToR-dynamic-bgp.webp)

#### 下一跳

每个 Node 上跑 BGP 服务，将本节点的上的 PodCIDR 宣告给接入层交换机，每个接入层交换机可以感知到直连的所有 Node 上的 PodCIDR。接入层交换机下的 Node 之间相互学习路由下发到本地，流量经过接入层交换机二层转发。**跨接入层交换机的节点之间通信下一跳指向接入层交换机，同一个接入层交换机下的节点之间通信下一跳指向对端节点。**下图展示了同一个接入层交换机下以及跨接入层交换机下节点的路由学习情况，可以直观的根据路由表判定下一跳地址。

- 同一个接入层交换机下通信链路：10.2.0.2节点与10.2.0.3节点处在同一个接入层交换机下，具备二层连通，报文经过封装后不经过三层转发直接被送到对端。
- 不同接入层交换机之间通信链路：10.2.0.2节点与10.3.0.3节点处在不同的接入层交换机下，报文需要经过接入层交换机和核心交换机路由后才能到达对端。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/ToR-bgp-next.webp)

#### 优雅重启

BGP 是基于 TCP 实现的路由协议，TCP 连接异常断开后，开启 Graceful Restart 功能的交换机不会删除 RIB 和 FIB，仍然按照原有的转发表项转发报文，并启动 RIB 路由老化定时器。BGP Peer 需要两端同时开启 Graceful Restart 功能才能生效，Graceful Restart可以有效防止BGP链路震荡，提升底层网络的可用性。



### metaLB and calico

https://mp.weixin.qq.com/s/tk_Zlt_zVAOX9IhDuGkYFA

**k8s集群外请求：**

1. 通过 NodePort 在主机上以 nat 方式将流量转发给容器，优点配置简单且能提供简单的负载均衡功能， 缺点也很明显下游应用只能通过主机地址+端口来做寻址
2. 通过 Ingress-nginx 做应用层的 7 层转发，优点是路由规则灵活，且流量只经过一层代理便直达容器，效率较高。缺点是 ingress-nginx 本身的服务还是需要通过 `NodePort` 或者 `HostNetwork` 来支持

可以看到在没有外部负载均衡器的引入之前，应用部署在 kubernetes 集群内，它对南北向流量的地址寻址仍然不太友好。

`MetalLB` 就是在裸金属服务器下为 Kubernetes 集群诞生的一个负载均衡器项目。

简单来说，MetalLB包含了两个组件，`Controler`用于操作 Service 资源的变更以及IPAM。`Speaker`用于外广播地址以及 BGP 的连接。它支持两种流模式模式即：`layer2` 和 `BGP`。

通过上述的介绍，你可能发现了一个问题：在MetaLB  BGP 模式的场景下，Calico 和 MetalLB 都需要运行一个 DaemonSet 的 bgp 客户端在主机上与上层路由器建立 bgp peer，在 Calico 中是 Bird ，MetalLB 中是 Speaker。这就会引出来它们使用上的一些问题。

BGP 只允许每个节点建立一个会话，如果使用 Calico 并建立了 BGP 路由器会话，MetalLB 无法建立自己的会话。因为这条 BGP会话会被路由器认为是相冲突而拒绝连接

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/calico-conflict-with-metalb.jpg)



事实上我们传统的 Fabric 网络在运用上述方案也遇到此问题，MetalLB 社区也给了 3 个方案来解决：

- BGP 与 Tor 交换机连接

  此方案即 MetalLB 放弃在 Node 节点上部署 Speaker 服务，关于主机上 BGP 路由的广播统一交给 Calico Bird 处理。这也是 Calico 社区建议采取的方案。

  ![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/metalb-shutdown-bgp.jpg)

- BGP 与 Spine 交换机连接

  此方案让 MetalLB Speaker 的 BGP Peer 绕过 Tor 路由，直达上层核心路由器。虽然解决了 BGP 连接问题，但是额外带来了配置的复杂性，以及损失了BGP 连接的扩展性，在大型的数据中心是不被认可的！

  ![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/metalb-bgp-with-spine.jpg)

- 开启 VRF-虚拟路由转发

  如果你的网络硬件支持 VRF（虚拟路由转发），那就可以将通过虚拟化的方式分别为 Calico Bird 和 MetalLB Speaker 创建独立的路由表，并建立 BGP 连接。然后再在两个 VRF 内部之间进行路由

  ![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vrf-with-metalb.jpg)

​		此方案理论上可行，但笔者的数据中心并没有支持 VRF 功能的路由器，且受限于不同网络设备厂家的实现方式不同而带来的操作差异也不可控。所以具体的实现还需每个用户自行决定。



### 引用

1. https://segmentfault.com/a/1190000040269867
2. https://mp.weixin.qq.com/s/tk_Zlt_zVAOX9IhDuGkYFA