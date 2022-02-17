## VLAN

Virtual Local Area Network， 虚拟局域网。 

Creating a Vlan means you are dividing up the BROADCAST DOMAINS. 

> VLAN 优点隔离广播域 BROADCAST DOMAINS

VLANs use software to emulate separate physical LANs. Each VLAN is thus a separate broadcast domain and a separate network.



### 通透世界

1. tag 和 untag 是针对 vlan ID，一个port 在 Trunk 或Hybird模式下，可以属于多个VLAN。所以，一个Port可以有N个 tag vlan 和 M个untag vlan。

   > Access port 只能属于一个 VLAN

2. PVID是根据 physical port来说的，一个物理端口只要一个PVID。且Access和Trunk、Hybird模式端口配置PVID命令不一致。

3. 交换机如果互联用access模式，不同vlan配置同网段ip是能通讯的，因为Access 模式发送数据包时会去除tag标签。

### VLAN and IP Subnetwork：666

通常，一个vlan对应一个Subnetwork~~~

穷举一下，VLAN和IP子网，各种划分，会有4种情形

1、A和B相同IP子网，相同VLAN：可以正常通讯，单播和广播通讯都会送达。

2、A和B不同IP子网，相同VLAN：需要路由器才能单播通讯，但是会有广播跨IP子网互相干扰。

3、A和B相同IP子网，不同VLAN：完全无法通讯，即使有路由器也不行。因为IP协议认为是发给近邻直连网络，数据不会路由给网关，会进行ARP请求广播，企图直接与目的主机通讯，可是由于B在另一个VLAN，网关不予转发ARP请求广播，B收不到ARP请求。结局是网络层ARP无法解析获得数据链路层需要的目的MAC，通讯失败。（除非：路由器上对两个VLAN之间进行桥接；或者路由器上进行静态NAT，若生成树设置不当容易把交换机搞死千万别作）

> 不同vlan分配不同子网，方便路由管理，避免额外手工配置。

4、A和B不同IP子网，不同VLAN：需要路由器才能进行单播通讯，不会有广播跨子网干扰。



情形1是常见的小型局域网；

情形2不应出现在正确配置的网络里，因为花了钱买路由器（三层交换机）但是钱白花了，没能隔离广播风暴；

情形3是常见的家庭WiFi路由器配置故障，一些运营商进户线路经过NAT是192.168.1.0/24的地址段，家用WiFi路由器恰好也用192.168.1.0/24这个地址段作为Lan口默认地址，路由器Lan端和WAN端冲突，傻掉了（可以这样理解：家用路由器的Lan端和WAN端分别处于两个不同的VLAN）；

情形4是常见的企业局域网。

三层交换机用在什么地方？三层交换机最主要最擅长的应用就是进行VLAN之间的数据路由转发，这种应用场合下它的转发速度要远胜过专业路由器，它可以做到以二层帧交换的速度进行跨VLAN三层路由转发作业。但是通常来说它NAT效率不如专业路由器或防火墙，不能跨二层协议处理数据包，通常把它用在局域网网络核心节点。在局域网网络末节，比如说Internet接入处，通常都会再专设一台路由器或防火墙来进行链路层协议转换和NAT转换。可以这么简单的理解，三层交换机是一种只有以太网接口，只能处理以太网协议的路由器。（难道除了以太网还有别的？当然有，比如串行、PPP）虽然三层交换机和路由器在内部运作机制上不一样，对于用户而言，数据路由转发这个功能它俩都具备。

### VLAN 好处：666

在underlay环境下不同网络的设备需要连接至不同的交换机下，如果要改变设备所属的网络，则要调整设备的连线。引入vlan后，调整设备所属网络只需要将设备加入目标vlan下，避免了设备的连线调整。

### VLAN tag 

IEEE 802.1q，是VLAN的正式标准，对Ethernet帧格式进行修改，在源MAC地址字段和协议类型字段之间加入4字节的802.1q Tag。

802.1Q Tag包含4个字段，其含义如下：

* Type

长度为2字节，表示帧类型。取值为0x8100时表示802.1Q Tag帧。如果不支持802.1Q的设备收到这样的帧，会将其丢弃。

* PRI

Priority，长度为3比特，表示帧的优先级，取值范围为0～7，值越大优先级越高。用于当交换机阻塞时，优先发送优先级高的数据包。

* CFI

Canonical Format Indicator，长度为1比特，表示MAC地址是否是经典格式。CFI为0说明是经典格式，CFI为1表示为非经典格式。用于区分以太网帧、FDDI（Fiber Distributed Digital Interface）帧和令牌环网帧。在以太网中，CFI的值为0。

* VID

VLAN ID，长度为12比特，表示该帧所属的VLAN。在VRP中，可配置 的VLAN ID取值范围为0～4095，但是0和4095协议中规定为保留的VLAN ID，不能给用户使用。



### 交换机不同类型端口数据流: mark，背诵全文

交换机的端口有三种模式，分别是access，trunk，hybrid。（port link-type命令设置端口类型）

配置：

如果二层以太网接口直接与计算机连接，则该接口需要配置成Access接口或Hybrid接口。

端口类型配置为Access后，该端口不支持命令**port trunk allow-pass vlan**。

如果二层以太网接口与另一台交换机设备的接口连接，则该接口需要配置成Trunk接口或Hybrid接口。

端口类型配置为Trunk后，该端口不支持命令**port default vlan**。

Access端口只属于1个VLAN，所以它的缺省VLAN就是它所在的VLAN，不用设置;

Hybrid端口和Trunk端口属于多个VLAN，所以需要设置缺省VLAN ID。

缺省情况下，Hybrid端口和Trunk端口的缺省VLAN为VLAN 1

#### Access

执行命令**port default vlan** *vlan-id*，将端口加入到指定的VLAN中。

> 因为一个access port 只能属于一个vlan，必须使用`port default vlan`命令。

在交换机内部传输时会保留tag标签，所以在同一个交换机的不用vlan是不能互通的。

发送数据包时会去除tag标签，所以交换机互联用access模式，不同vlan配置同网段ip是能通讯的

1.access端口接收帧时：

①如果接收的帧有vlan tag时，该帧的vlan ID和access端口的PVID相同时，将改帧送入交换机；该帧的vlan ID和access端口的PVID不同时，丢弃帧。

②如果接收的帧没有vlan tag时，access端口会将该帧打上vlan tag，vlan ID即为本端口的PVID，送入交换机。

2.access端口发送帧时：

access端口只能发送vlan ID和端口PVID相同的帧，发送出去时会剥掉vlan tag。

```bash
[Huawei-GigabitEthernet0/0/3]port link-type access
[Huawei-GigabitEthernet0/0/3]port default vlan 10

```

此时该端口为access端口，PVID为10。

该端口只能发送vlan ID为10的帧，发送出来的帧没有vlan tag。

该端口可以接收vlan ID为10的帧（如交换机传出的帧）；也可以接收没有vlan tag的帧（如PC传出的帧），此帧将打上vlan tag（vlan ID=10）传入交换机。

end

#### Trunk

Trunk端口加入 VLAN：

1. 执行命令**port trunk allow-pass vlan** { { *vlan-id1* [ **to** *vlan-id2* ] } &<1-10> | **all** }，将端口加入到指定的VLAN中。
2. 执行命令**commit**，提交配置。

设置PVID：

**port trunk pvid vlan** *vlan-id*

1.trunk端口接收帧时：

①接收没有vlan tag的帧（untag帧），trunk端口将帧打上vlan tag，vlan ID和本端口的PVID相同，若该PVID在trunk端口的放行vlan中，送入交换机，若PVID不在trunk端口的放行vlan中，丢弃该帧。

②接收有vlan tag的帧，若帧的vlan ID在trunk端口的放行vlan中，送入交换机，若vlan ID不在trunk端口的放行vlan中，丢弃该帧。

2.trunk端口发送帧时：

trunk端口只能发送放行vlan中的帧，若该帧的vlan ID和trunk的PVID相同，则剥掉vlan tag发送；若该帧的vlan ID和trunk的PVID不同，则保留原有vlan tag发送。

即：

vid和pvid不相等，直接原样发送
vid和pvid相等，去除tag标签发送

```bash
[Huawei-GigabitEthernet0/0/4]port link-type trunk
[Huawei-GigabitEthernet0/0/4]port trunk pvid vlan 5
[Huawei-GigabitEthernet0/0/4]port trunk allow-pass vlan 10 20 30
```

此时该端口为trunk端口，PVID为5，放行vlan为10，20，30。

该端口可以发送vlan ID为10，20，30的帧，发送出去的帧时有vlan tag的。

该端口可以接收vlan ID为10，20，30的帧。

```bash
[Huawei-GigabitEthernet0/0/5]port link-type trunk
[Huawei-GigabitEthernet0/0/5]port trunk pvid vlan 5
[Huawei-GigabitEthernet0/0/5]port trunk allow-pass vlan 5 10 20 30
```

此时该端口为trunk端口，PVID为5，放行vlan为5，10，20，30。

该端口可以发送vlan ID，5，10，20，30的帧，发送vlan ID为10、20、30的帧，帧是有vlan tag的；发送vlan ID为5的帧，帧时没有vlan tag的。

该端口可以接收vlan ID为5，10，20，30的帧,，也可以接收没有vlan tag的帧。

end

#### Hybrid: tag and untag

**总结**：hybird tagged + hybird untagged = trunk allow-pass

Hybrid端口加入 VLAN：

1. 命令**port hybrid tagged vlan** { { *vlan-id1* [ **to** *vlan-id2* ] } &<1-10> | **all** }，将端口加入到指定的VLAN中。

   > 使用port hybrid tagged vlan 命令配置以Tagged形式将Hybrid类型接口加入VLAN后，接口在发送帧时不将帧中的VLAN Tag去掉。

2. port hybrid untagged vlan vlan-id 或者 undo port hybrid vlan vlan-id，删除Hybrid port加入的VLAN

   > port hybrid untagged vlan vlan-id命令用来配置Hybrid类型接口加入的VLAN，这些VLAN的帧以Untagged方式通过接口。

缺省情况下，Hybrid端口以Untagged方式加入VLAN1。

> display port vlan Ethernet0/0/1 active 可以显示

1.hybrid端口接收帧时：

①接收没有vlan tag的帧，hybrid端口将帧打上vlan tag，vlan ID和本端口的PVID相同，若该PVID在hybrid端口的放行vlan中，送入交换机，若PVID不在hybrid端口的放行vlan中，丢弃该帧。

②接收有vlan tag的帧，若帧的vlan ID在hybrid端口的放行vlan中，送入交换机，若vlan ID不在hybrid端口的放行vlan中，丢弃该帧。

2.hybrid端口发送帧时：

①判断该VLAN在本端口的属性（disp interface 即可看到该端口对哪些VLAN是untag， 哪些VLAN是tag）
②如果是untag则剥离VLAN信息，再发送，如果是tag则直接发送

hybrid端口只能发送放行vlan中的帧，可以通过命令来控制发送时是否携带vlan tag。

```bash
[Huawei-GigabitEthernet0/0/6]port hybrid pvid vlan 10
[Huawei-GigabitEthernet0/0/6]port hybrid tagged vlan 10 20 30
## vlan 100 200 300需要untag在发送
[Huawei-GigabitEthernet0/0/6]port hybrid untagged vlan 100 200 300
```

hybrid端口，PVID为10，放行的vlan有10、20、30、100、200、300。(untagged + tagged = allow-pass)

端口接收帧时同trunk是一样。

端口发送帧时，vlan ID为10、20、30的帧时有vlan tag的；vlan ID为100、200、300的帧时没有vlan tag的。

end

#### 查看交换机端口类型

执行命令**display port vlan** *interface-type interface-number* **active**，查看设备上已创建成功的VLAN中包含的指定接口类型和编号的接口信息。

```bash
<Huawei>display port vlan Ethernet0/0/1 active
T=TAG U=UNTAG
-------------------------------------------------------------------------------
Port                Link Type    PVID    VLAN List
-------------------------------------------------------------------------------
Eth0/0/1            hybrid       1       U: 1
```

end

### Switch Port and Link-type

以太网端口有三种链路类型：

- Access：只能属于一个 VLAN，一般用于连接计算机的端口

  只允许默认vlan的以太网帧，Access端口在收到以太网帧后打上vlan标签，转发时在剥离vlan标签。

  配合交换机VLAN PVID端口。

- Trunk：可以属于多个 VLAN，可以接收和发送多个 VLAN 的报文，一般用于交换机之间连接的接口。

  可以允许多个vlan通过，可以接受并转发多个vlan的报文一般作用于交换机之间连接的端口，在网络的分层结构方面，trunk被解释为"端口聚合"，就是把多个物理端口捆绑在一起作为一个逻辑端口使用，作用可以扩展带宽和做链路的备份。

  > 可以在，trunk口配置允许通过的指定VLAN，默认是所有VLAN都可以通过。
  >
  > 类似,openvswitch中：`ovs-vsctl set port patch_to_vswitch trunk=222`，则只允许`VLAN 222` 的帧通过。

- Hybrid：属于多个 VLAN，可以接收和发送多个 VLAN 报文，既可以用于交换机之间的连接，也可以 用户连接用户的计算机。 Hybrid 端口和 Trunk 端口的不同之处在于 Hybrid 端口可以允许多个 VLAN 的报文发送时不打标签，而 Trunk 端口只允许缺省 VLAN 的报文发送时不打标签。

 vlan的链路类型可以分为接入链路和干道链路。

（1）接入链路（access link）指的交换机到用户设备的链路，即是接入到户，可以理解为由交换机向用户的链路。由于大多数电脑不能发送带vlan tag的帧，所以这段链路可以理解为不带vlan tag的链路。

（2）干道链路（trunk link）指的交换机到上层设备如路由器的链路，可以理解为向广域网走的链路。这段链路由于要靠vlan来区分用户或者服务，所以一般都带有vlan tag。

#### PVID and Trunk/Hibird

Port-base VLAN ID，

PVID所在的端口必然是untag port，保障进去交换机内部的Frame是没有tag的，才能转给其他不同vlan tag的端口，从PVID端口出现的包，自动打上Port-base VLAN ID，用于在三层交换机上路由传播。

> 一个交换机上可能有N个VLAN所以需要untaged保障来自不同vlan的包可以进入到该交换机。
>
> eg： 一个交换机 16 个端口，8个在VLAN_10，另外8个在VLAN_20，1-16全都属于 vlan 300
>
> VLAN_1和VLAN_2需要通信，则可以将端口设置为untagged + tagged +PVID，保障不同VID数据包可以进入该port。
>
> 1-8 port
>
> port hybrid pvid vlan 10
>
> port hybrid tagged vlan 10 
>
> port hybrid untagged vlan 300（发送vlan_300的package时，去掉vlan tag）
>
> port 同时属于vlan 10(tag) 和 vlan 300(untag)，当vlan 10 和 vlan 20通讯时，借助vlan 300 即可。



Port VLAN ID，代表端口缺省VLAN ID.

PVID并不是加在帧头的标记，而是端口的属性，用来标识端口接收到的未标记的帧。也就是说，当端口收到一个未标记的帧时，则把该帧转发到VID和本端口PVID相等的VLAN中去。

The PVID of a port is the VLAN id that will be assigned to any untagged frames entering the switch on that port (assuming the switch is using port-based VLAN classification). This is a concept that is defined in IEEE 802.1Q.

例如: 此port的PVID = 1，代表此port可以转发VLAN1的封包，因为Untag的封包进入port後，会被标上VID1。

　PVID英文解释为Port-base VLAN ID，是基于端口的VLAN ID，一个端口可以属于多个vlan，但是只能有一个PVID，收到一个不带tag头的数据包时，会打上PVID所表示的vlan号，视同该vlan的数据包处理。
　　一个物理端口只能拥有一个PVID，当一个物理端口拥有了一个PVID的时候，必定会拥有和PVID相等的VID，而且在这个VID上，这个物理端口必定是Untagged Port。

> 若有 Tag 的封包进入 switch，则其经过 untagged port 时，Tag 将被去除 。

　　**PVID的作用只是在交换机从外部接受到可以接受Untagged 数据帧的时候给数据帧添加TAG标记用的，在交换机内部转发数据的时候PVID不起任何作用。**

Hybrid类型接口的PVID是可以手动更改的。
华为设备通过命令 port hybrid pvid vlan xx;

Access类型接口将接口化进某一个VLAN时，接口的PVID也随之改变。

Trunk类型的接口的PVID也是可以手动改的。
华为设备通过命令 port trunk pvid vlan xx;

**VID**：VLAN ID，表示端口所处的VLAN编号，是**VLAN属性**。

**PVID**：Port vlan ID，表示缺省端口的VLAN标号，是**端口属性**。

在设置PVID和VID时，要保持PVID和VID的一致。譬如：一个端口属于几个VLAN，那么这个端口就会具有好几个VID，但是只能有一个PVID，并且PVID号应该是VID号中的一个，否则交换机不识别。

##### Trunk Port

```bash
[SW1] interface gigabitEthernet0/0/24
[SW1-GigabitEthernet0/0/24] port link-type trunk
[SW1-GigabitEthernet0/0/24] port trunk allow-pass vlan 10 20
```

由于VLAN Trunk可以传输多个VLAN数据，Ethernet Frame在VLAN Trunk上传输时，需要带上802.1Q定义的VLAN tag，这样交换机才能区分Ethernet Frame到底属于哪个VLAN。VLAN Trunk的具体使用过程如下：

◆    当最左侧服务器想要访问最右侧服务器，最左侧服务器将Ethernet Frame发送到左侧交换机

◆    左侧交换机本地没有目的MAC对应的转发信息，因此Ethernet Frame发送到了左侧交换机的VLAN Trunk port

◆    由于这是来自VLAN100的Ethernet Frame，交换机给Ethernet Frame打上VLAN 100的Tag，从自己的VLAN Trunk port发出，发送到右侧交换机的VLAN Trunk port

◆    右侧的VLAN Trunk port收到VLAN 100的Ethernet Frame，去除VLAN Tag，再在本地的VLAN 100端口中查找MAC转发

◆    找到对应的MAC/端口记录，最终Ethernet Frame发送到最右侧的服务器

##### Linux Trunk Port

Windows也支持Trunk port，与Linux类似，这里只说明Linux系统下的Trunk port。

Linux系统可以通过内核模块8021q支持VLAN Trunk。这里有什么不一样？在一般情况下，主机是不感知VLAN Tag的，也就是说主机发送的网络数据都是不带VLAN Tag，所有VLAN Tag操作都是由交换机完成。但是实际上交换机也不知道自己连接的是什么，所以，如果在主机完成VLAN Tag的操作，再发送到交换机，交换机也能处理。基于这个前提，Linux能够将一块以太网卡配置成一个支持802.1q的Trunk port，使得这块网卡跟前面描述的交换机上的Trunk port一样，能够收发多个VLAN的网络数据包。并且通过配置Linux主机的子网卡，可以使得Linux主机内部完成VLAN Tag的操作（打上VLAN Tag，去除VLAN Tag）。有不止一种方法可以配置Linux VLAN Trunk，这里以ip命令为例，为eth0添加名为eth0.102的子网卡，其VLAN ID为102。

```bash
$ sudo ip link add link eth0 name eth0.102 type vlan id 102
```

就这么简单，之后可以看到一个新的网卡在系统的网卡列表中。Linux VLAN Trunk工作过程如下：

◆    外界传入的带VLAN Tag 102的包，到达了eth0，VLAN Tag 102被去除，然后不带VLAN Tag的Ethernet Frame被重新发往操作系统网络栈，再发送到eth0.102。

◆    从eth0.102发送出来的帧，被打上VLAN102的标签，再从eth0传出。



应用层面，收发的还是不带VLAN Tag的数据，只是经过Linux VLAN Trunk的处理，进出主机的的数据是带VLAN Tag的数据。

由于eth0配置成了VLAN Trunk port。为了让带有VLAN标签的Ethernet Frame能够在网络上传输。eth0所接入的网络必须是一个Trunk network。否则无法传输任意带VLAN Tag的Ethernet Frame。也就是说，需要将主机的Trunk port连接到红框内。

#### VLANID

Tag和Untag，tag是指vlan的标签，即vlan的id，用于指名数据包属于那个vlan，untag指数据包不属于任何vlan，没有vlan标记。

VLAN TAG包的VLAN ID号，有效范围是1-4094，0和4095都为协议保留值.

#### PVID and VID

PVID与VID范例 :

当Port1同时属于VLAN1、VLAN2和VLAN3时，而它的PVID为1，那么Port1可以接收到VLAN1，2，3的封包，但发出的封包只能发到VLAN1中。

当 port 的 PVID = 1时，代表此 port 可以转发 VLAN1 的封包

##### tag port ： 针对VID

tag报文结构的变化是在源mac地址和目的mac地址之后，加上了4bytes的vlan信息，也就是vlan tag头；一般来说这样的报文普通PC机的网卡是不能识别的。

从此port 转发出的封包上都将有Tag (tagged)。若有非Tag的封包进入Switch，则其经过tagged port时，Tag将被加上。

将使用在ingress(流入)端口上的pvid设定作为Tag的vlan id。(用于交换机与交换机之间传输)。

##### untag port：针对VID

untag就是普通的ethernet报文，普通PC机的网卡是可以识别这样的报文进行通讯；

此port**转发出**的封包上都没有Tag (untagged)。若有Tag的封包进入switch，则其经过untagged port时，Tag将被去除。(用于一般设备、电脑)

所谓的untagged Port和tagged Port不是讲述物理端口的状态，而是将是物理端口所拥有的某一个VID的状态，所以一个物理端口可以在某一个VID上是untagged Port，在另一个VID上是tagged Port。 

untag port和tag port是针对VID来说的，和PVID没有什么关系。比如有一个交换机的端口设置成untag port，但是从这个端口进入交换机的网络包如果没有vlan tag的话，就会被打上该端口的PVID，**不要以为它是untag port就不会被打上vlan tag。**

#### 收发报文：？

这个没看懂，这里设置hybird port干嘛，直接access port就行了。

因为可以借助layer 3 switch 实现 路由，因为 pc连接 e0/1 和 e0/2时，也要配置网关和路由呢。

下面的例子难道是不借助 Layer 3 switch？？？

PVID的作用只是在交换机从外部接受到可以接受Untagged 数据帧的时候给数据帧添加TAG标记用的.

华为交换机 hybird模式配置实例

拓扑： PC_1 连接 inter0/1 , PC_2 连接 inter0/2

```bash
[Switch-Ethernet0/1]int e0/1
[Switch-Ethernet0/1]port link-type hybrid
[Switch-Ethernet0/1]port hybrid pvid vlan 10
## [Huawei-GigabitEthernet0/0/1]port hybrid untagged vlan 100 200
[Switch-Ethernet0/1]port hybrid vlan 10 20 untagged
## 0.0
[Switch-Ethernet0/1] int e0/2
[Switch-Ethernet0/2]port link-type hybrid
[Switch-Ethernet0/2]port hybrid pvid vlan 20
## [Huawei-GigabitEthernet0/0/1]port hybrid untagged vlan 100 200
[Switch-Ethernet0/2]port hybrid vlan 10 20 untagged
-----------------------------------
```

此时inter e0/1和inter e0/2下的所接的PC是可以互通的，但互通时数据所走的往返vlan是不同的。
以下以inter e0/1下的所接的pc1访问inter e0/2下的所接的pc2为例进行说明

##### PC_1 --> PC_2

pc1所发出的数据，由inter0/1所在的pvid vlan10封装vlan10的标记后送入交换机，交换机发现inter e0/2允许vlan 10的数据通过，于是数据被转发到inter e0/2上，由于inter e0/2上vlan 10是untagged的，于是交换机此时去除数据包上vlan10的标记，以普通包的形式发给pc2，此时pc1->p2走的是vlan10

##### PC_2--> PC_1

分析pc2给pc1回包的过程，

pc2所发出的数据，由inter0/2所在的pvid vlan20封装vlan20的标记后送入交换机，交换机发现inter e0/1允许vlan 20的数据通过，于是数据被转发到inter e0/1上，由于inter e0/1上vlan 20是untagged的，于是交换机此时去除数据包上vlan20的标记，以普通包的形式发给pc1，此时pc2->pc1走的是vlan20

交换机端口的tag与untag
https://blog.51cto.com/centaurs1987/1437083



**收报文：**

Access端口：
1、收到一个报文；
2、判断是否有VLAN信息；如果没有则转到第3步，否则转到第4步；
3、打上端口的PVID，并进行交换转发；
4、直接丢弃(缺省)；
Trunk端口：
1、收到一个报文；
2、判断是否有VLAN信息；如果没有则转到第3步，否则转到第4步；
3、打上端口的PVID，并进行交换转发；
4、判断该trunk端口是否允许该VLAN的数据进入；如果可以则转发，否则丢弃；
Hybrid端口：
1、收到一个报文；
2、判断是否有VLAN信息；如果没有则转到第3步，否则转到第4步；
3、打上端口的PVID，并进行交换转发；

4、判断该Hybrid端口是否允许该VLAN的数据进入：如果可以则转发，否则丢弃；

 **发报文：**

Access端口：
1、将报文的VLAN信息剥离，直接发送出去；
Trunk端口：
1、比较端口的PVID和将要发送报文的VLAN信息；
2、如果两者相等则转到第3步，否则转到第4步；
3、剥离VLAN信息，再发送；
4、直接发送；
Hybrid端口：
1：判断该VLAN在本端口的属性(displayinterface即可看到该端口对哪些VLAN是untag,哪些VLAN是tag。)
2、如果是untag则转到第3步，如果是tag则转到第4步；
3、剥离VLAN信息，再发送；
4、直接发送；

### VLANIF：6

vlanif , vlan interface只是个虚拟接口，一般在支持802.1q的三层交换机，准确来说，它叫做SVI（Switch Virtual Interface），是一个三层接口，只会在Layer 3交换机上出现，通常用来为某个VLAN提供最后的路由（网关）服务。因为这个接口在交换机上不真实地存在，所以叫做虚接口。

```bash
[Huawei]interface vlanif 111
Error: The VLAN does not exist.
```

以三层交换机为例，假设24个端口一半是vlan 1,一半是vlan2。然后网关又要设在三层交换机上，你说网关应该设在哪个端口上呢？下面的接口全是二层接口，是没有办法设置ip地址的。这个时候有一个虚拟的接口这个事就好办了，虚拟接口1作为vlan1的三层网关接口，虚拟接口2作为vlan2的三层网关接口，这个虚拟接口就叫vlan interface.





### 交换机转发VLAN帧

#### 基础帧转发

二层交换机最基本的功能包括：

- MAC 地址学习：当交换机从它的某个端口收到数据帧时，它将端口的 ID 和帧的源 MAC 地址保存到它的内部MAC表中。这样，当将来它收到一个要转发到该 MAC 地址的帧时，它就知道直接从该端口转发出去了。
- 数据帧转发：交换机在将从某个端口收到数据帧，再将其从某个端口转发出去之前，它会做一些逻辑判断：
  - 如果帧的目的 MAC 地址是广播或者多播地址的话，将其从交换机的所有端口（除了传入端口）上转发。
  - 如果帧的目的MAC地址在它的内部MAC表中能找到对应的输出端口的话（MAC 地址学习过程中保存的），将其从该端口上转发出去。
  - 对其它情况，将其从交换机的所有端口（除了传入端口）上转发。
- 加 VLAN 标签/去 VLAN 标签：
  - 帧接收：从 trunk port 上收到的数据帧必须是加了标签的。从 access port 上收到的数据帧必须是没有加标签的，否则该帧将会被抛弃。
  - 帧处理：根据上述转发流程决定其发出的端口。
  - 帧发出：从 trunk port 发出的帧是加了标签的。从 access port 上发出的帧必须是没加标签的。

#### VLAN Frame 转发

 默认情况下，交换机的所有端口都处于VLAN 1 中，也就相当于没有配置 VLAN。该机制说明如下：

1. PC_A 发一个帧到交换机的 1 端口，其目的MAC地址为 PC_B 的 MAC。
2. 交换机比较其目的 MAC 地址和它的内部 MAC Table，发现它不存在（此时表为空）。在决定**泛洪**之前，它把端口 1 和 PC_A 的 MAC 地址存进它的 MAC Table。
3. 交换机将帧拷贝多份，分别从2和3端口发出。
4. PC_B 收到该帧以后，发现其目的 MAC 地址和他自己的 MAC 地址相同。它发出一个回复帧进入端口3。
5. 交换机将 PC_B 的 MAC地址和端口3 存在它的 MAC 表中。
6. 因为该帧的目的地址为PC_A 的 MAC 地址它已经在 MAC 表中，交换机直接将它转发到端口1，达到PC_A。

配置了 VLAN 的交换机的该机制类似，只不过：

（1）MAC 表格中每一行有不同的 VLAN ID。做比较的时候，拿传入帧的目的 MAC 地址和 VLAN ID 和此表中的行数据相比较。如果都相同，则选择其 Ports 作为转发出口端口。

（2）如果没有吻合的表项，则将此帧从所有有同样 VLAN ID 的 Access ports 和 Trunk ports 转发出去。

#### Untaged frames

what about untagged frames entering the switch from the PC or printer (They’ll be untagged because the PC or printer doesn’t know about VLAN). This is where PVID comes in. PVID tells the switch what to do with those untagged incoming frames. 

VLAN隔离个广播域，

> 常见的广播通信：
>
> - ARP请求：建立IP地址和MAC地址的映射关系。
> - RIP：一种路由协议。
> - DHCP：用于自动设定IP地址的协议。
>
> end

从Access port进入的包大多数是untag frame...

### vlan间访问控制

一台Switch 24 port 分三个VLAN，第port 24连到firewall，各个VLAN之间不能互通，但全透过port 24上网。

设定方式

1.分别设定VLAN1 : 1-8&24、VLAN2 : 9-16&24、VLAN3 :17-23&24、VLAN4 :1-24

2.设定PVID及VID 如下图所示

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vlan-untaged-port.jpg)

说明:

PC1接在Port1，PC1发送一个Untag的封包从Port1进入，Port1的PVID=1，此Untag的封包则变成带VID=1的tag封包，在switch内移动，可移动范围为VID=1的port，所以Port 1-8、24可通行也可透过port 24 连接到firewall上网。

而防火墙透过port 24进入的Untag的封包，因Port 24 的PVID=4

此Untag的封包则变成带VID=4的tag封包，在switch内移动，可移动范围为VID=4的port，所以Port 1-24 都可通行，所以port 24可以跟port 1 通。

备注说明 : untagged port & tagged port

PC1接在Port1，PC2接在Port5，因为这两个port都是untagged port，所以PC1的Untag的封包从Port1的进入后带上VID1的tag，要从Port5出来时，因为untagged port所以会将Tag移掉，再把Untag的封包交给PC2。

tagged port则不会将Tag移掉，会将封包交到正确的VLAN。





### VLAN的不足

#### VLAN 数量

VLAN 使用 12-bit 的 VLAN ID，因此第一个不足之处就是最多只支持 4096 个 VLAN 网络，无法满足虚拟化和云计算时代的用户需求。eg：公有云100W个用户，每个用户一个或N个租户。

##### QinQ

QinQ是为了扩大VLAN ID的数量而提出的技术（IEEE 802.1ad），外层tag称为Service Tag，而内层tag则称为Customer Tag。

通过将一层 VLAN Tag 封装到私网报文上，使其携带两层 VLAN Tag 穿越运营商的骨干网络(又称公网)，从而使运营商能够利用一个 VLAN 为包含多个 VLAN 的用户网络提供服务。

QinQ报文在运营商网络中传输时带有双层VLAN Tag：

- 内层 VLAN Tag：为用户的私网 VLAN Tag，Customer VLAN Tag (简称 CVLAN)。设备依靠该 Tag 在私网中传送报文。
- 外层 VLAN Tag：为运营商分配给用户的公网 VLAN Tag， Service VLAN Tag(简称 SVLAN)。设备依靠该 Tag 在公网中传送 QinQ 报文。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/QinQ.jpeg)

在公网的传输过程中，设备只根据外层 VLAN Tag 转发报文，而内层 VLAN Tag 将被当作报文的数据部分进行传输。

#### VLAN间通讯互联

VLAN在分割了二层广播域，也严格地隔离了各个VLAN之间的任何二层流量，属于不同VLAN的用户之间不能直接进行二层通信。需要借助”三层设备“实现不同VLAN之间的通讯。



##### 方法1：普通路由器互联

在路由器上为每个VLAN分配一个单独的接口，并使用一条物理链路连接到二层交换机上。

当VLAN间的主机需要通信时，数据会经由路由器进行三层路由，并被转发到目的VLAN内的主机，这样就可以实现VLAN之间的相互通信。

这种方式弊端很明显：

1. 路由器端口有限
2. 端口利用率低，比如某些VLAN之间的主机可能不需要频繁进行通信

现实中没人这么干

##### 方法2：单臂路由，router-on-a-stick

单臂路由（router-on-a-stick）是指在路由器的一个接口上通过配置子接口（或“逻辑接口”，并不存在真正物理接口）的方式，实现原来相互隔离的不同VLAN（虚拟局域网）之间的互联互通。

在交换机和路由器之间仅使用一条物理链路连接。在交换机上，把连接到路由器的端口配置成Trunk类型的端口，并允许相关VLAN的帧通过。在路由器上需要创建子接口，逻辑上把连接路由器的物理链路分成了多条。一个子接口代表了一条归属于某个VLAN的逻辑链路。

> 需要子接口的原因：
>
> 交换机的Trunk类型端口，只允许带有VLAN tag的帧通过，也就是说交换机Trunk端口出去的帧不做区分。
>
> 路由器的子接口，分别对接从交换机Trunk端口进入的VLAN 帧并根据不同路由子接口允许进入不同的VID帧，进行VLANs间的数据路由。

单臂路由缺点：

- “单臂”为网络骨干链路，容易形成网络瓶颈（带宽低，转发率不高）
- 子接口依然依托于物理接口，应用不灵活
- VLAN间转发需要查看路由表，严重浪费设备资源



![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/router-on-a-stick.png)



##### 方法3：三层交换机，VLANIF

三层交换机在原有的二层交换机上增加了路由功能，因为数据没有像单臂路由那样经过物理线路进行路由，很好解决了带宽瓶颈的问题。

在三层交换机上配置VLANIF接口来实现VLAN间路由。如果网络上有多个VLAN，则需要给每个VLAN配置一个VLANIF接口，并给每个VLANIF接口配置一个IP地址。用户设置的缺省网关就是三层交换机中VLANIF接口的IP地址。

```bash
<Huawei>sys
Enter system view, return user view with Ctrl+Z.
## system-view 模式是 []
## [HUAWEI] sysname SW2//修改设备的名称为SW2，便于识别
## 创建vlan
[HUAWEI] vlan 100
[HUAWEI] interface gigabitethernet 0/0/1
[HUAWEI-GigabitEthernet1/0/1] port link-type access
[HUAWEI-GigabitEthernet1/0/1] port default vlan 100
[HUAWEI-GigabitEthernet1/0/1] quit
[HUAWEI] display port vlan

### 添加vlanif
<Huawei>sys
Enter system view, return user view with Ctrl+Z.
[Huawei]interface vlanif 100
[Huawei-Vlanif100]ip address 192.168.1.254 24
[Huawei-Vlanif100]

```



![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/layer-3-switch.jpg)

##### 方法3.5：Hybrid/Trunk Port： layer 2 switch

这个例子应该是在layer 2 switch上实现vlan互通的例子

拓扑： 

* PC_1 连接 inter0/1，port 属于vlan 100
*  PC_2 连接 inter0/2， port属于vlan 200
* PC_4连接 inter0/3 ， port属于vlan 100

```bash
[Switch-Ethernet0/1]int e0/1
[Switch-Ethernet0/1]port link-type hybrid
[Switch-Ethernet0/1]port hybrid pvid vlan 10
## [Huawei-GigabitEthernet0/0/1]port hybrid untagged vlan 100 200
[Switch-Ethernet0/1]port hybrid vlan 10 20 untagged
## 0.0
[Switch-Ethernet0/1] int e0/2
[Switch-Ethernet0/2]port link-type hybrid
[Switch-Ethernet0/2]port hybrid pvid vlan 20
## [Huawei-GigabitEthernet0/0/1]port hybrid untagged vlan 100 200
[Switch-Ethernet0/2]port hybrid vlan 10 20 untagged
-----------------------------------
```

此时inter e0/1和inter e0/2下的所接的PC是可以互通的，但互通时数据所走的往返vlan是不同的。
以下以inter e0/1下的所接的pc1访问inter e0/2下的所接的pc2为例进行说明

PC_1/pc4 --> PC_2

pc1所发出的数据，由inter0/1所在的pvid vlan10封装vlan10的标记后送入交换机，交换机发现inter e0/2允许vlan 10的数据通过，于是数据被转发到inter e0/2上，由于inter e0/2上vlan 10是untagged的，于是交换机此时去除数据包上vlan10的标记，以普通包的形式发给pc2，此时pc1->p2走的是vlan10

PC_2--> PC_1

分析pc2给pc1回包的过程，

pc2所发出的数据，由inter0/2所在的pvid vlan20封装vlan20的标记后送入交换机，交换机发现inter e0/1允许vlan 20的数据通过，于是数据被转发到inter e0/1上，由于inter e0/1上vlan 20是untagged的，于是交换机此时去除数据包上vlan20的标记，以普通包的形式发给pc1，此时pc2->pc1走的是vlan20



#### 不同VLAN中IP(subnet)重复：大问题

公有云提供商的业务要求将实体网络租借给多个不同的用户，这些用户对于网络的要求有所不同，而不同用户租借的网络有很大的可能会出现IP地址、MAC地址的重叠，传统的VLAN仅仅解决了同一链路层网络广播域隔离的问题，而并没有涉及到网络地址重叠的问题，因此需要一种新的技术来保证在多个租户网络中存在地址重叠的情况下依旧能有效通信的技术。

#### MAC表数量限制

TOR（Top Of Rack）交换机的MAC表大小限制。

这个MAC地址表是通过交换机的flood-learn学习并记录在交换机的内存。交换机的内存比较宝贵，所以MAC地址表的大小通常是有限的。现在因为虚拟化，整个数据中心的MAC地址多了几十倍，那相应的交换机里面的MAC地址表也需要扩大几十倍。如果交换机不支持这么大的MAC地址表，那么就会导致MAC地址表溢出。溢出之后，交换机不能将新的MAC地址学习到自己的MAC地址表。如果交换机收到这些MAC地址的数据帧，因为不能通过查表转发，会flood到所有的端口。这不但增加了交换机的负担，还增加了网络中其他设备的负担。为了避免这个问题，可以用一些更大容量的交换机，但是相应的成本也要上升，而且还不能从根本上解决这个问题。



### Do VLANs require different subnet？: 有意思

https://community.spiceworks.com/topic/1116440-do-vlans-require-different-subnets

No, VLANs don't require different subnets. Different subnets require different subnet addresses if they ever need to be able to route and/or talk to each other) and by extension if one VLAN wants to talk to a different VLAN it must use different addresses so we can make a routing decision to the right place. **Most people make some relationship between the subnet address assignment and the VLAN ID (similart to your example) because it's convenient, otherwise they would be constantly consulting a table of what subnet addressing is used on what VLAN**.

Just take the V out of VLAN, different VLANs are effectively different LANs but thanks to the V can use the same switches, routers and trunks or to put it another way, if you want a separate LAN you can use a VLAN and not have to buy two of everything to build a separate LAN entirely.

In one instance the customer is already using the addresses I need to use but their network never needs to talk to mine so I just use a separate VLAN on the same switch which uses the same subnet address they do and because they never need to communicate, the separate VLAN keeps another LAN with the same address scheme on the same switch totally separate.  This was a VoIP installation and their data network was already using the same IP address block I needed to use but it didn't matter, in some cases the two separate subnets with overlapping address space are carried on the same port (e.g. switchport voice VLAN x) and they don't interfere or have duplicate IPs (even though they do have duplicates) because they are on separate LANs (or VLANs).

#### Multiple VLANs in the same subnet

I want to avoid having many Subnets (and vNIC's in TMG) just to isolate sets of a few hosts.

```
IP: 10.0.0.1         (TMG server)       VLAN:1 ~ 3

IP: 10.0.0.11 ~ 20   (Hosts group 1)    VLAN:1

IP: 10.0.0.21 ~ 30   (Hosts group 2)    VLAN:2

IP: 10.0.0.31 ~ 40   (Hosts group 3)    VLAN:3
```

Note that I don't want them to connect to each other, so ARP/inter-vlan routing (within the subnet) is not required.

Of course you can do that, but it is not the recommended way.

VLANs use software to emulate separate physical LANs. Each VLAN is thus a separate broadcast domain and a separate network.

As you have identified, routing between these VLANs would be difficult, because they are the same subnet. If the addresses are all different it is possible to route traffic using a very large number of rules which don't correspond to the actual subnet configuration and will confuse anyone who inherits this from you. However, it is completely permissible to use the same RFC1918 subnets on different physical networks. You could likely even make all the addresses the same.

#### Example

Let's say I have four PCs connected to one CISCO switch.

- **PC 1** has IP address `192.168.10.1` and is connected to the access port - *vlan ID 10*.
- **PC 2** has IP address `192.168.10.2` and is connected to the access port - *vlan ID 10*.
- **PC3** has IP address `192.168.10.1` (same IP address as PC1) and is connected to the access port - *vlan ID 20*
- **PC4** has IP address `192.168.10.2` ( same IP address as PC2) and is connected to the access port - *vlan ID 20*

Can PC1 ping PC2?
Can PC3 ping PC4?

### Openstack + VLAN：mark

VLAN网络依赖三层交换机VLAN技术，物理主机的管理流量与虚拟机的业务流量使用不同物理网口，并分处不同网络。 因此物理主机需要有两个网口，物理主机管理流量使用第一个网口，虚拟机业务流量使用第二个网口。

> 这里简化了，其实是需要三个网卡的，因为还有存储网络。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/openstack-vlan.png)

三层交换机需要进行VLAN划分，IP配置及上行链路（物理主机网口接入交换机的端口）放行操作。其中，

#### 管理网口

物理主机管理网口接入交换机的端口链路类型可以使用access，或者hybrid类型。
配置示例：
假设项目规划物理主机管理流量使用VLAN 10，其网络IP规划为192.168.10.0/24，物理主机业务网口连接在交换机GigabitEthernet0/0/1端口上。则交换机（华为）配置如下：

```bash
interface Vlanif10
 	  ip address 192.168.10.254 255.255.255.0

	interface GigabitEthernet0/0/1
 	  port link-type hybrid          # 端口链路类型为hybrid
      port hybrid pvid vlan 10      # 设置默认vlan 10
###	或者
	interface GigabitEthernet0/0/1
 	  port link-type access        # 端口链路类型为access
      port default vlan 10          # 设置默认vlan 10


```

#### 业务网口-1

物理主机业务网口接入交换机的端口链路类型可以使用hybrid，或者trunk类型。
配置示例：
假设项目规划虚拟机业务流量使用VLAN 100，其网络IP规划为192.168.100.0/24，物理主机业务网口连接在交换机GigabitEthernet0/0/3端口上。则交换机（华为）配置如下：

```bash
interface Vlanif100
 	  ip address 192.168.100.254 255.255.255.0

	interface GigabitEthernet0/0/3
 	  port link-type hybrid     # 端口链路类型为hybrid
      port hybrid tagged vlan 100 # 放行tagged vlan 100
	### 或者
	interface GigabitEthernet0/0/3
 	  port link-type trunk       端口链路类型为trunk
      port trunk allow-pass vlan 100    # 放行tagged vlan 100


```

end

#### 业务网口-N

此处仅举例一个vlan配置情况，当业务网络规划多个vlan网络时，重复上述操作即可。此处配置需要牢记，管理网络信息在配置物理主机操作系统网卡时需要，业务网络信息在OpenStack搭建完成之后，创建虚拟机网络需要。

### OpenvSwitch（OVS）+VLAN组网

https://www.1024sou.com/article/60783.html



-----------------------------------


### 引用

0. https://support.huawei.com/enterprise/zh/doc/EDOC1100059351/a64bb5c0

1. https://blog.csdn.net/weixin_52122271/article/details/112383249
2. https://blog.51cto.com/u_13212728/2516078
3. https://blog.51cto.com/centaurs1987/1437083
3. https://www.1024sou.com/article/60783.html
3. https://blog.csdn.net/bunny_nini/article/details/104545978
3. https://blog.51cto.com/u_15127681/2818076
3. https://blog.csdn.net/luuJa_IQ/article/details/104134312
3. https://blog.51cto.com/sevenot/2133771

