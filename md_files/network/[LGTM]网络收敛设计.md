## [LGTM]网络收敛设计

### 流量收敛

数据报文的流量收敛，是指数据报文在网络转发过程中由于架构、设备等非故障原因而不能实现线速无丢包转发。在流量收敛时，网络设备会有部分端口拥塞，进而丢弃部分报文。为了能够描述不同的收敛程度，我们通常用一个系统所有南向（下行）接口的总带宽比上这个系统所有北向（上行）接口总带宽的数值来表示，这个数值称为这个系统的收敛比。

举个例子，假设你有10台服务器，每台服务器通过10GE的接口连接到一个接入交换机，那我们一共就有100G（10×10G=100G）的南向带宽。假设这台交换机还有2个40GE的接口可以用于接入到更高一层的汇聚交换机，那我们一共就有80G（2×40G=80G）的北向带宽。此时，我们得到的收敛比则是1.25：1（100G÷80G=1.25）。

需要说明的是，造成网络流量收敛的原因并不总是上述描述的这个例子，不过总的来说，可以将流量收敛的原因分为两类：

- 交换机不支持线速转发，在交换机内部可能形成流量收敛；

  交换机背板转发能力（也就是整个交换机的线速转发能力）等于或大于全部端口转发能力之和，我们可以认为此交换机具备全线速转发无丢包。

- 网络架构设计的原因，无论交换机是否线速，转发报文时也会存在流量收敛。

#### 线速转发问题

某交换机只具有8Gbps线速转发的交换能力，某时刻从交换机前12个接口向后12个接口同时转发流量，当每个接口流量均达到1Gbps时，在交换机内部一定会有拥塞，此时便形成了转发的收敛。实际每秒交换机接收流量为12Gbps，但转发出去的报文只有8Gbps，收敛比为输入带宽(12Gbps)÷输出带宽(8Gbps)=1.5：1。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/switch线速.png)

#### 架构设计问题

4台服务器分别通过10GE链路连接接入交换机，接入交换机通过1条25GE链路连接核心交换机。即接入交换机的下行带宽为40Gbps，接入交换机的上行带宽为25Gbps。下上行链路收敛比为下行带宽（40Gbps）÷上行带宽（25Gbps）=1.6：1。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/switch-架构.png)



#### 理想收敛比

最理想的收敛比是1：1。但低收敛比的设计意味着选用更高上行端口带宽的设备，这意味着更多的投入；如果在不计成本的情况下，1：1的收敛比的期望是能实现的。另外一方面，由于服务器也不是每时每刻都工作在高负荷下，占用100%的带宽，这意味着即使不是1：1的收敛比，也不是就一定会出现数据报文因拥塞丢包，业务仍可以正常运行。因此，找到这两者之间的平衡，找到最适合的收敛比，就十分有必要。

### Border Leaf-Spine-Leaf 收敛设计

Spine交换机具有高吞吐量、低延迟且端口密集，它们与每个Leaf交换机都有直接的高速 (40-400Gbps) 连接。

Leaf交换机与传统TOR交换机非常相似，它们通常是 24 或 48 端口 1、10 或 40Gbps的接入层连接。但是，它们增加了到每个Spine交换机的 40、100 或 400Gbps 上行链路的能力。



以高密机架服务器场景为例，如下图所示，每机架部署了 10 台机架式服务器（通常会部署 8-12 台服务器），每排共有 10 个机架（通常每排会部署 8-12 个机架）部署了服务器，构成高密度机架服务器部署。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/top接入交换机.png)



#### server接入leaf(3:1)

计算单个 leaf下行和上行

为了保证可靠性每台服务器采用 M-LAG 的方式接入网络，相邻的两个机架组成一组 M-LAG 的系统，这样一个机架上面的 TOR 交换机需要接入 200G（10G × 10 × 2 = 200G）带宽的流量。根据经验值，我们在接入层的收敛比一般控制在 3:1 左右，这主要取决于我们将为接入交换机设计多大的上行带宽。

> M-LAG（Multichassis Link Aggregation Group）提供一种跨设备链路聚合的技术。

目前来说，接入交换机的单个上行接口可以达到 40G 的带宽，理论上通过 4 个 40G 的上行接口，我们就可以大致将收敛比做到 1:1。但是此时我们至少需要为该 Spine 设置 4 台汇聚交换机，且每增加一个上行接口就需要增加一台汇聚交换机，因此这个开销还是十分可观的。

在实际部署中，我们一般设置两台汇聚交换机，接入交换机通过 2 个 40GE 接口接入汇聚交换机，提供 80G（40G × 2 = 80G）的上行带宽。这样我们就可以得到 2.5:1（200G ÷ 80G = 2.5）的收敛比，很好的将其控制在 3:1 以内。

#### Leaf 接入Spine

计算整个Spine下行和上行

每排机架上共有 10 台 TOR 交换机需要连接到我们的汇聚交换机，每台 TOR 交换机通过 2 个 40GE 接口接入到 Spine 层设备，一共是 80 个（2 × 10 × 4 = 80）接口，3200G（80 × 40G = 3200G）的带宽。

> 总共4排机架row 1 ~ row 4，每个机架有10个机柜。
>
> 也可以这么算，单个ToR switch提供80G的上行带宽，总共4 row * 10 * 80G = 3200G即Spine层下行带宽为3200G 

为了避免的设备的单点故障引起网络问题，我们每个 Spine 节点都至少有 2 台 Spine 层交换机。

##### 一个Spine交换机

这些 TOR 交换机作为一个 Spine 接入，那么这个 Spine 接入的接口数为 80 （4 rows * 2 ports  * 10 ToR switch）个接口，3200G 的带宽，如下图所示。在这种情况下，如果通过 100G 的**上行链路**链接到 Border Leaf 设备（一般为两台，北向连接到出口路由器），则可以提供 400G（4 × 100G = 400G）的带宽，此时收敛比为 8:1（3200G ÷ 400G = 8）。

> 400 G = 2 switch * 2 port * 100G ？？？

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/one-spine-top.png)

##### 两个Spine交换机

两个Spine时，每个Spine的下行带宽为2 row * 10 * 80G=1600G，然后每个Spine连接2个Border，则上行带宽为

`1600G / 2 / 2 = 400G `

在这个场景下，我们采用了两个 Spine 接入的方式，即将 Row1 和 Row2 共用一个 Spine，Row3 和 Row4 共用一个Spine。此时我们一般选用框式交换机或高性能的盒式灵活插卡交换机来作为 Spine 设备。

将这些 TOR 交换机分为两个 Spine 接入，那么每个 Spine 接入的接口数为 40 个接口，1600G 的带宽。在这种情况下，我们如果通过 100G 的**上行链路**链接到 Border Leaf 设备，则可以提供 400G（4 × 100G = 400G）的带宽，此时收敛比为 4:1（1600G ÷ 400G = 4）。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/two-spine-top.png)

在数据中心网络中 Spine 的划分，收敛比并不是唯一的依据，更主要的是根据业务和功能的分区来划分的。另外受限于交换机本身路由、ARP 等规格的限制，再加上现在虚拟机的大规模应用（虚拟比达到 1:30 或更高，对规格要求更高），一个 Spine 的规模也不会太大。

#### Spine 接入 Border Leaf

计算Border下行和上行

Border Leaf 北向主要是连接 Data Center Border Router（DCBRs，DC 边界/出口路由器），南向连接不同的 Spine，承担 Spine 间东西向流量的转发。Border Leaf 的设计很重要的一个是考虑客户所购买的出口路由器的端口。这些端口相比较于下层的网络设备比较贵，一般情况下都是 10GE/40GE 的接口。这也意味着我们在 Border Leaf 的收敛比会比较大

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/border-leaf-spine.png)

如果按照 4 (2 switches * 2 ports)个 40GE 的出口接口来计算，我们将有 160G 的带宽，此时收敛比是 5:1（800G ÷ 160G = 5）。

> 800G是Spine上行带宽，怎么算出来的？？？
>
> 2 Spine Switch * 2 * 2 *100G =800G
>
> 两个Spine交换机两个端口做冗余且一个冗余端口连接两个不同的Border

但是，根据目前统计约 75% 的流量都是发生在数据中心的内部，即东西向的流量。那么剩下的 25% 的流量，即南北向的流量大概只有 200G（800G × 25% = 200G）。如果按照这个来估算，我们的收敛比为 1.25:1（200G ÷ 160G = 1.25），属于可接受的范围。



#### 网络收敛设计的逻辑图

最终得到网络收敛设计的逻辑图

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/border-leaf-spine-leaf-leaf.png)



### 引用

1. https://support.huawei.com/enterprise/zh/doc/EDOC1100023543?section=j00m