## Link-Aggregation-draft

### 基本概念

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/link-aggregation.png)

将多个以太网接口捆绑在一起所形成的组合称为聚合组，而这些被捆绑在一起的以太网接口就称为该聚合组的成员端口。每个聚合组唯一对应着一个逻辑接口，我们称之为聚合接口。聚合组/聚合接口分为以下两种类型：

*  二层聚合组/二层聚合接口：二层聚合组的成员端口全部为二层以太网接口，其对应的聚合接口称为二层聚合接口（Bridge-aggregation Interface，BAGG）。
* 三层聚合组/三层聚合接口：三层聚合组的成员端口全部为三层以太网接口，其对应的聚合接口称为三层聚合接口（Route-aggregation Interface，RAGG）。

#### LAG

链路聚合组LAG（Link Aggregation Group）是指将若干条以太链路捆绑在一起形成一条逻辑链路，也称Eth-Trunk链路。每个聚合组对应一个链路聚合接口或Eth-Trunk接口，组成Eth-Trunk接口的各个物理接口称为成员接口，成员接口对应的链路称为成员链路。链路聚合接口可以作为普通的以太网接口来使用，与普通以太网接口的差别在于：转发的时候链路聚合组需要从成员接口中选择一个或多个接口来进行数据转发。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/link-aggregation-group.png)

*链路聚合组与链路聚合接口、成员接口和成员链路关系示意图*

LAG是一种链路聚合技术，当在两台交换机之间并行连接多个端口并将它们配置为LAG时，链路聚合组就会形成，而LACP是一种自动建立LAG的控制协议，用于启用LAG自动配置网络交换机端口、分离链路故障和激活故障切换。

LAG主要有两种模式，分别是手工模式和LACP模式。

- 手工模式：指LAG不启用任何链路聚合协议，Eth-Trunk的建立、成员接口的加入由手工配置。
- LACP模式：指LAG启用LACP链路聚合协议，Eth-Trunk的建立、成员接口的加入基于LACP协议协商完成。

部分设备支持手工模式，但不支持LACP模式，LACP模式需要本端和对端设备同时启用LACP协议，所选择的活动接口必须保持一致，才能建立LAG，如果对端设备未启用LACP协议，本端LAG会尝试将数据包传输到远程单个接口，可能导致通信失败。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/手工模式与lacp模式.png)



#### LACP

LACP（Link Aggregation Control Protocol，链路聚合控制协议）是一种基于IEEE802.3ad标准的实现链路动态聚合与解聚合的协议，它是链路聚合中常用的一种协议。链路聚合组中启用了LACP协议的成员端口通过发送LACPDU报文进行交互，双方对哪些端口能够发送和接收报文达成一致，确定承担业务流量的链路。此外，当聚合条件发生变化时，如某个链路发生故障，LACP模式会自动调整聚合组中的链路，组内其他可用成员链路接替故障链路维持负载平衡。这样在不进行硬件升级的情况下，可以增加设备之间的逻辑带宽，提高网络的可靠性。

##### LACP模式对数据传输更加稳定和可靠

手工模式下，所有链路都是活动链路，所有活动链路均参与数据转发，平均分担流量。如果某条活动链路故障，链路聚合组自动在剩余的活动链路中平均分担流量。

LACP模式下，由LACP确定聚合组中的活动和非活动链路，又称为M:N模式，即M条活动链路与N条备份链路的模式。这种模式提供了更高的链路可靠性，并且可以在M条链路中实现不同方式的负载均衡。

如下图所示，两台设备间有M+N条链路，在聚合链路上转发流量时在M条链路上分担负载，即活动链路，不在另外的N条链路转发流量，这N条链路提供备份功能，即备份链路。此时链路的实际带宽为M条链路的总和，但是能提供的最大带宽为M+N条链路的总和。当M条链路中有一条链路故障时，LACP会从N条备份链路中找出一条优先级高的可用链路替换故障链路。此时链路的实际带宽还是M条链路的总和，但是能提供的最大带宽就变为M+N-1条链路的总和。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/lacp-m-n.png)

##### LACP模式对聚合链路组的故障检测更加准确和有效

手工模式只能检测到同一聚合组内的成员链路有断路等有限故障，LACP模式不仅能够检测到同一聚合组内的成员链路有断路等有限故障，还可以检测到链路故障、链路错连等故障。

如下图所示，DeviceA与DeviceB之间创建Eth-Trunk，需要将DeviceA上的四个接口与DeviceB捆绑成一个Eth-Trunk。由于错将DeviceA上的一个接口与DeviceC相连，这将会导致DeviceA向DeviceB传输数据时可能会将本应该发到DeviceB的数据发送到DeviceC上。

手工模式的Eth-trunk不能及时检测到该故障，如果在DeviceA和DeviceB上都启用LACP协议，经过协商后，Eth-Trunk就会选择正确连接的链路作为活动链路来转发数据，从而DeviceA发送的数据能够正确到达DeviceB。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/lacp故障检测.png)

##### LACP工作原理

LACP为数据交换设备提供一种标准的协商方式，系统根据自身配置自动形成聚合链路，并启动聚合链路收发数据。LACP通过链路聚合控制协议数据单元LACPDU（Link Aggregation Control Protocol Data Unit）与对端交互信息，LACPDU报文中包含设备的系统优先级、MAC地址、接口优先级、接口号和操作Key等信息，对端接收到这些信息后，将这些信息与其它端口所保存的信息比较以选择能够汇聚的端口，双方对端口加入或退出某个动态聚合组达成一致，确定承担业务流量的链路。

LACP主要工作主要包含互发LACPDU报文、确定主动端、确定活动链路、链路切换。

在对接的两台设备上创建Eth-Trunk并配置为LACP模式，然后向Eth-Trunk中手工加入成员接口。此时成员接口上便启用了LACP协议，两端互发LACPDU报文，LACPDU报文中包含设备的系统优先级、MAC地址、接口优先级、接口号和操作Key等信息。

#### PAgP

LACP和PAgP（Port Aggregation Protocol，端口汇聚协议）是链路聚合中使用最广泛的两种协商协议。LACP和PAgP的功能类似，都是通过捆绑链路并协商成员链路之间的流量提高网络的可用性和稳定性。LACP和PAgP数据包在交换机之间通过支持以太网通道的端口交换。

它们之间最大的区别是支持的供应商不同，LACP是开放标准，可以在大多数交换机上运行，如华为S5700系列交换机，而PAgP是Cisco专有协议，只能在Cisco或支持PAgP的第三方交换机上运行。



### Bridge-aggregation：二层聚合

链路聚合分为静态聚合和动态聚合两种模式，它们各自的优点如下所示：

* 静态聚合模式：一旦配置好后，端口的选中/非选中状态就不会受网络环境的影响，比较稳定。
* 动态聚合模式：能够根据对端和本端的信息调整端口的选中/非选中状态，比较灵活。

处于静态聚合模式下的聚合组称为静态聚合组，处于动态聚合模式下的聚合组称为动态聚合组。

#### 配置静态聚合组

```bash
<H3C>system-view
System View: return to User View with Ctrl+Z.
## 创建aggregation group
[H3C]interface Bridge-Aggregation 101
## 添加描述信息
[H3C-Bridge-Aggregation101]description text static-bg-101
[H3C-Bridge-Aggregation101]quit
## 查看aggregation信息
[H3C]display interface Bridge-Aggregation brief
Brief information on interfaces in bridge mode:
Link: ADM - administratively down; Stby - standby
Speed: (a) - auto
Duplex: (a)/A - auto; H - half; F - full
Type: A - access; T - trunk; H - hybrid
Interface            Link Speed   Duplex Type PVID Description
BAGG101              DOWN auto    A      A    1    text static-bg-101

```

end

#### 配置动态聚合组：lacp

LACP工作模式分为ACTIVE和PASSIVE两种。

如果动态聚合组内成员端口的LACP工作模式为PASSIVE，且对端的LACP工作模式也为PASSIVE时，两端将不能发送LACPDU。如果两端中任何一端的LACP工作模式为ACTIVE时，两端将可以发送LACPDU。

```bash
<H3C>system-view
System View: return to User View with Ctrl+Z.
## 创建aggregation group
[H3C]interface Bridge-Aggregation 301
## 添加描述信息
[H3C-Bridge-Aggregation301]description text dynamic-bg-301
## 配置聚合组工作在动态聚合模式下，缺省情况下，聚合组工作在静态聚合模式下
[H3C-Bridge-Aggregation301]link-aggregation mode dynamic
[H3C-Bridge-Aggregation301]quit
## 查看aggregation信息
[H3C]display interface Bridge-Aggregation brief
Brief information on interfaces in bridge mode:
Link: ADM - administratively down; Stby - standby
Speed: (a) - auto
Duplex: (a)/A - auto; H - half; F - full
Type: A - access; T - trunk; H - hybrid
Interface            Link Speed   Duplex Type PVID Description
BAGG1                DOWN auto    A      A    1    text
BAGG2                DOWN auto    A      A    1
BAGG3                DOWN auto    A      A    1
BAGG101              DOWN auto    A      A    1    text static-bg-101
BAGG301              DOWN auto    A      A    1    text dynamic-bg-301

## 可选
## 缺省情况下，端口的LACP优先级为32768
## 改变端口的LACP优先级，将会影响到动态聚合组成员端口的选中/非选中状态
lacp port-priority port-priority
```

end

#### ports加入聚合组

多次执行此步骤可将多个二层以太网接口加入聚合组

```bash
## 将G1/0/4加入 aggegation group 401
[H3C]interface GigabitEthernet1/0/4
[H3C-GigabitEthernet1/0/4]port link-aggregation group 401
[H3C-GigabitEthernet1/0/4]quit
## 将G1/0/5加入 aggegation group 401
[H3C]interface GigabitEthernet1/0/5
[H3C-GigabitEthernet1/0/5]port link-aggregation group 401
[H3C-GigabitEthernet1/0/4]quit
```

end

#### 查看维护

```bash
## 显示聚合信息
[H3C]display  interface Bridge-Aggregation
...
Bridge-Aggregation2
Current state: UP
Line protocol state: UP
IP packet frame type: Ethernet II, hardware address: 703a-a671-a073
Description: to-Server1-YeWu
Bandwidth: 20000000 kbps
20Gbps-speed mode, full-duplex mode
Link speed type is autonegotiation, link duplex type is autonegotiation
PVID: 1        
Port link-type: Trunk
 VLAN Passing:   2-1999, 2001-2007, 2009-2016, 2018, 2020-2021, 2024-2026
                 2028-2039, 2041-2044, 2046-2049, 2051, 2053, 2055-2058
                 2060-2073, 2076-2079, 2081-2083, 2085-2097, 2099-2100
                 2102-2221, 2223-2299, 2301-4094
 VLAN permitted: 2-1999, 2001-2007, 2009-2016, 2018, 2020-2021, 2024-2026
                 2028-2039, 2041-2044, 2046-2049, 2051, 2053, 2055-2058
                 2060-2073, 2076-2079, 2081-2083, 2085-2097, 2099-2100
                 2102-2221, 2223-2299, 2301-4094
 Trunk port encapsulation: IEEE 802.1q
Last clearing of counters: Never
Last 300 seconds input:  17 packets/sec 3016 bytes/sec 0%
Last 300 seconds output:  62 packets/sec 64173 bytes/sec 0%
Input (total):  12347068 packets, 2070923086 bytes
        11963655 unicasts, 192 broadcasts, 383221 multicasts, 0 pauses
Input (normal):  12347068 packets, - bytes
        11963655 unicasts, 192 broadcasts, 383221 multicasts, 0 pauses
Input:  0 input errors, 0 runts, 0 giants, 0 throttles
        0 CRC, 0 frame, - overruns, 0 aborts
        - ignored, - parity errors
Output (total): 69920914 packets, 59922317108 bytes
        55834727 unicasts, 7228857 broadcasts, 6857330 multicasts, 0 pauses
Output (normal): 69920914 packets, - bytes
        55834727 unicasts, 7228857 broadcasts, 6857330 multicasts, 0 pauses
Output: 0 output errors, - underruns, - buffer failures
        0 aborts, 0 deferred, 0 collisions, 0 late collisions
        0 lost carrier, - no carrier

..

## 显示哪些端口在哪个聚合组下
[H3C]display link-aggregation member-port
Flags: A -- LACP_Activity, B -- LACP_Timeout, C -- Aggregation,
       D -- Synchronization, E -- Collecting, F -- Distributing,
       G -- Defaulted, H -- Expired

GigabitEthernet1/0/1:
Aggregate Interface: Bridge-Aggregation1
Port Number: 2
Port Priority: 32768
Oper-Key: 1

GigabitEthernet1/0/2:
Aggregate Interface: Bridge-Aggregation1
Port Number: 3
Port Priority: 32768
Oper-Key: 1

GigabitEthernet1/0/3:
Aggregate Interface: Bridge-Aggregation3
Port Number: 4
Port Priority: 32768
Oper-Key: 2
[H3C]
Inactive timeout reached, logging out.

GigabitEthernet1/0/4:
Aggregate Interface: Bridge-Aggregation401
Local:
    Port Number: 5
    Port Priority: 32768
    Oper-Key: 3
    Flag: {ACG}
Remote:
    System ID: 0x8000, 0000-0000-0000
    Port Number: 0
    Port Priority: 32768
    Oper-Key: 0
    Flag: {EF}
Received LACP Packets: 0 packet(s)
Illegal: 0 packet(s)
## 确实和static aggregation不一样
Sent LACP Packets: 0 packet(s)

GigabitEthernet1/0/5:
Aggregate Interface: Bridge-Aggregation401
Local:
    Port Number: 6
    Port Priority: 32768
    Oper-Key: 3
    Flag: {ACG}
Remote:
    System ID: 0x8000, 0000-0000-0000
    Port Number: 0
    Port Priority: 32768
    Oper-Key: 0
    Flag: {EF}
Received LACP Packets: 0 packet(s)
Illegal: 0 packet(s)
Sent LACP Packets: 0 packet(s)


```

end

### route-aggregation：三层聚合

#### 静态聚合组

组网要求

* Device A与Device B通过各自的三层以太网接口GigabitEthernet1/0/1～GigabitEthernet1/0/3相互连接。

* 在Device A和Device B上分别配置三层静态链路聚合组，并为对应的三层聚合接口配置IP地址和子网掩码。

```bash
## 配置device A
# 创建三层聚合接口1，并为该接口配置IP地址和子网掩码。

<DeviceA> system-view

[DeviceA] interface route-aggregation 1

[DeviceA-Route-Aggregation1] ip address 192.168.1.1 24

[DeviceA-Route-Aggregation1] quit

# 分别将接口GigabitEthernet1/0/1至GigabitEthernet1/0/3加入到聚合组1中。

[DeviceA] interface gigabitethernet 1/0/1

[DeviceA-GigabitEthernet1/0/1] port link-aggregation group 1

[DeviceA-GigabitEthernet1/0/1] quit

[DeviceA] interface gigabitethernet 1/0/2

[DeviceA-GigabitEthernet1/0/2] port link-aggregation group 1

[DeviceA-GigabitEthernet1/0/2] quit

[DeviceA] interface gigabitethernet 1/0/3

[DeviceA-GigabitEthernet1/0/3] port link-aggregation group 1

[DeviceA-GigabitEthernet1/0/3] quit

## 配置Device B

Device B的配置与Device A相似，配置过程略。
```

验证配置结果

```bash
# 查看Device A上所有聚合组的详细信息。

[DeviceA] display link-aggregation verbose

Loadsharing Type: Shar -- Loadsharing, NonS -- Non-Loadsharing

Port Status: S -- Selected, U -- Unselected, I -- Individual

Port: A -- Auto port, M -- Management port, R -- Reference port

Flags:  A -- LACP_Activity, B -- LACP_Timeout, C -- Aggregation,

        D -- Synchronization, E -- Collecting, F -- Distributing,

        G -- Defaulted, H -- Expired

 

Aggregate Interface: Route-Aggregation1

Aggregation Mode: Static

Loadsharing Type: Shar

Management VLANs: None

  Port             Status  Priority Oper-Key

  GE1/0/1(R)       S       32768    1

  GE1/0/2          S       32768    1

  GE1/0/3          S       32768    1

以上信息表明，聚合组1为负载分担类型的三层静态聚合组，包含有三个选中端口。
```

end

#### 动态聚合组

好简洁...

```bash
# 配置Device A
# 创建三层聚合接口1，配置该接口为动态聚合模式，并为其配置IP地址和子网掩码。

<DeviceA> system-view

[DeviceA] interface route-aggregation 1

[DeviceA-Route-Aggregation1] link-aggregation mode dynamic

[DeviceA-Route-Aggregation1] ip address 192.168.1.1 24

[DeviceA-Route-Aggregation1] quit

# 分别将接口GigabitEthernet1/0/1至GigabitEthernet1/0/3加入到聚合组1中。

[DeviceA] interface gigabitethernet 1/0/1

[DeviceA-GigabitEthernet1/0/1] port link-aggregation group 1

[DeviceA-GigabitEthernet1/0/1] quit

[DeviceA] interface gigabitethernet 1/0/2

[DeviceA-GigabitEthernet1/0/2] port link-aggregation group 1

[DeviceA-GigabitEthernet1/0/2] quit

[DeviceA] interface gigabitethernet 1/0/3

[DeviceA-GigabitEthernet1/0/3] port link-aggregation group 1

[DeviceA-GigabitEthernet1/0/3] quit

## 配置Device B

Device B的配置与Device A相似，配置过程略。
```

end

### 引用

1. https://info.support.huawei.com/info-finder/encyclopedia/zh/LACP.html
2. https://www.h3c.com/cn/d_201912/1252416_30005_0.htm
