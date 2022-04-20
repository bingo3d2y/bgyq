## Link-Aggregation-draft

### 基本概念

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/link-aggregation.png)

将多个以太网接口捆绑在一起所形成的组合称为聚合组，而这些被捆绑在一起的以太网接口就称为该聚合组的成员端口。每个聚合组唯一对应着一个逻辑接口，我们称之为聚合接口。聚合组/聚合接口分为以下两种类型：

*  二层聚合组/二层聚合接口：二层聚合组的成员端口全部为二层以太网接口，其对应的聚合接口称为二层聚合接口（Bridge-aggregation Interface，BAGG）。
* 三层聚合组/三层聚合接口：三层聚合组的成员端口全部为三层以太网接口，其对应的聚合接口称为三层聚合接口（Route-aggregation Interface，RAGG）。

### Bridge-aggregation

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

#### 配置动态聚合组

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

