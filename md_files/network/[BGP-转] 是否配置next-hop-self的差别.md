# [BGP-转] 是否配置next-hop-self的差别

原文：https://zhuanlan.zhihu.com/p/203030604

## `next-hop-self`的使用背景

使用`next-hop-self`的根源，在于BGP的协议设计里的一个限制或者说路由通告时的原则，在RFC4271中的描述如下——

> When sending a message to an internal peer, if the route is not locally originated, the BGP speaker SHOULD NOT modify the NEXT_HOP attribute unless it has been explicitly configured to announce its own IP address as the NEXT_HOP

将非本地起源的路由通告给iBGP邻居的时候，BGP不应该(**'SHOULD NOT'**)修改`NEXT_HOP`属性，除非明确配置使用它自身的IP地址作为`NEXT-HOP`。

从这段描述可以看出，这个特性并不是针对eBGP的，而是针对所有`route is not locally originated`的情况，显然，从eBGP收到的路由符合这个情况。

典型的iBGP & eBGP都存在的拓扑如下所示

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/iBGP-eBGP-topo.png)

R01和R05在一个AS内，跑一个iBGP，并且通过OSPF通告了环回口路由，确保iBGP的正常递归。R01和R02分属两个不同的AS，之间通过直连IP地址跑eBGP。

由RFC的说法我们可知，R01将R02收到的路由发给R05的时候，不会修改`NEXT_HOP`属性，这就导致对于R05来说，R02的地址是不可达的，递归路由失败。为了解决这个问题，我们就有了两个做法——

- 将R01和R02之间的直连路由通过BGP或者IGP宣告给R05
- 在R01上设置对R05的`next-hop-self`，当R01将路由通告给R05的时候，会修改下一跳为自身。由于R01的IP地址在AS100内通过IGP可达，就消除了下一跳递归失败的问题。

## 使用`next-hop-self`和发布直连路由的对比

发布直连路由也有多种做法——

- 重分发直连路由到BGP
- 重分发直连路由到IGP
- 将R01连接R02的接口直接宣告进IGP

### **重分发直连路由到BGP**

这个方案我们已经在前面那篇测试过了，是最差的方案，由R02发布的路由，在R05上需要经过三次递归才行。

配置示例

```bash
route-map red permit 10 
 match interface GigabitEthernet1
!
router bgp 100
 bgp router-id 1.1.1.1
 bgp log-neighbor-changes
 timers bgp 10 30
 neighbor 5.5.5.5 remote-as 100
 neighbor 5.5.5.5 update-source Loopback0
 neighbor 123.1.1.2 remote-as 200
 !
 address-family ipv4
  redistribute connected route-map red
  neighbor 5.5.5.5 activate
  neighbor 123.1.1.2 activate
 exit-address-family
```

检查R01的路由是否有`123.1.1.0/24`的直连路由

```bash
Network          Next Hop            Metric LocPrf Weight Path
 *>i  10.15.1.0/24     5.5.5.5                  0    100      0 i
 *>i  10.15.2.0/24     5.5.5.5                  0    100      0 i
 *>i  10.15.3.0/24     5.5.5.5                  0    100      0 i
 *>i  10.15.4.0/24     5.5.5.5                  0    100      0 i
 *>i  10.15.5.0/24     5.5.5.5                  0    100      0 i
 *>   10.234.1.0/24    123.1.1.2                              0 200 i
 *>   10.234.2.0/24    123.1.1.2                              0 200 i
 *>   10.234.3.0/24    123.1.1.2                              0 200 i
 *>   10.234.4.0/24    123.1.1.2                              0 200 i
 *>   10.234.5.0/24    123.1.1.2                              0 200 i
 *>   123.1.1.0/24     0.0.0.0                  0         32768 ?
```

检查R05是否收到了该路由

```bash
Network          Next Hop            Metric LocPrf Weight Path
 *>   10.15.1.0/24     0.0.0.0                  0         32768 i
 *>   10.15.2.0/24     0.0.0.0                  0         32768 i
 *>   10.15.3.0/24     0.0.0.0                  0         32768 i
 *>   10.15.4.0/24     0.0.0.0                  0         32768 i
 *>   10.15.5.0/24     0.0.0.0                  0         32768 i
 *>i  10.234.1.0/24    123.1.1.2                0    100      0 200 i
 *>i  10.234.2.0/24    123.1.1.2                0    100      0 200 i
 *>i  10.234.3.0/24    123.1.1.2                0    100      0 200 i
 *>i  10.234.4.0/24    123.1.1.2                0    100      0 200 i
 *>i  10.234.5.0/24    123.1.1.2                0    100      0 200 i
 *>i  123.1.1.0/24     1.1.1.1                  0    100      0 ?
```

可以看到R5不仅收到`123.1.1.0/24`，来自AS200的五条路由也已经被BGP注入了路由表。

有什么问题呢？通过CEF能看出来——

```bash
R05#sh ip cef 10.234.1.0 detail
10.234.1.0/24, epoch 2, flags [rib only nolabel, rib defined all labels]
  recursive via 123.1.1.2
    recursive via 123.1.1.0/24
      recursive via 1.1.1.1
        nexthop 15.1.1.1 GigabitEthernet1
```

从IP地址递归到直连路由，再递归到通告该路由的R01的环回口地址。

### **重分发直连路由到IGP**

这是我上次漏掉的一个场景，实际上大部分时候我们应该是重分发到IGP而不是BGP

```text
route-map red permit 10 
 match interface GigabitEthernet1
router ospf 1
 router-id 1.1.1.1
 redistribute connected subnets route-map red
```

R05上的递归结果如下

```text
R05#sh ip cef 10.234.1.0 detail
10.234.1.0/24, epoch 2, flags [rib only nolabel, rib defined all labels]
  recursive via 123.1.1.2
    recursive via 123.1.1.0/24
      nexthop 15.1.1.1 GigabitEthernet1
```

由于直连被放进IGP，所以只需要从`NEXT_HOP`递归到直连网段即可。

**将R01连接R02的接口直接宣告进IGP**

```text
router ospf 1
 router-id 1.1.1.1
interface GigabitEthernet1
 ip address 123.1.1.1 255.255.255.0
 ip ospf 1 area 0
```

R05上的递归结果如下

```text
R05#sh ip cef 10.234.1.0 detail
10.234.1.0/24, epoch 2, flags [rib only nolabel, rib defined all labels]
  recursive via 123.1.1.2
    recursive via 123.1.1.0/24
      nexthop 15.1.1.1 GigabitEthernet1
```

和重分发没有什么不同。

### **配置`next-hop-self`**

R01配置

```text
R01#sh run | s r b
router bgp 100
 bgp router-id 1.1.1.1
 bgp log-neighbor-changes
 timers bgp 10 30
 neighbor 5.5.5.5 remote-as 100
 neighbor 5.5.5.5 update-source Loopback0
 neighbor 123.1.1.2 remote-as 200
 !
 address-family ipv4
  neighbor 5.5.5.5 activate
  neighbor 5.5.5.5 next-hop-self
  neighbor 123.1.1.2 activate
 exit-address-family
```

R05上的路由递归结果

```text
R05#sh ip cef 10.234.1.0 detail
10.234.1.0/24, epoch 2, flags [rib only nolabel, rib defined all labels]
  recursive via 1.1.1.1
    nexthop 15.1.1.1 GigabitEthernet1
```

只递归一层，因为`1.1.1.1`原本就可以通过IGP可达。

## 结论

从R05的递归结果上来看，一定是**`next-hop-self` > IGP发布外部直连 > BGP发布外部直连**。从管理边界角度考虑，如果外部直连并非由本地AS分配，那么一定是选择`next-hop-self`，原则上不应该让不在自己管理范围内的互连网段进入内部网络，污染内部路由。

从收敛角度则很难看出差别，当直连发生故障时，BGP和OSPF的反应时间差不太多，也不存在谁收敛更快的情况，而且对于MSTP这种存在链路up而三层可能挂掉了的链路，几乎只能等待BGP的Hold Time超时。

## 关于`next-hop-self`碎碎念

有人问过我一个问题，既然`next-hop-self`这个功能这么常用，为什么协议或者厂商实现时不把它作为一个默认动作，当时我回答的不好，现在的话，我想我大概能够给出个答案——

- 首先，从RFC的描述中，我们可以看出来，不修改下一跳，这件事并不是针对eBGP的，而是针对所有非本地起源路由的，只是恰好在从eBGP发路由到iBGP的时候遇到了问题，而在一个iBGP环境里，可能是没有问题的。协议设计者不可能针对所有的场景都去做一些exception的行为，提供一个 next-hop-self 这样的开关才是更合理的做法。

- 其次，协议本身是发展着的，实际上，对于`NEXT_HOP`这个属性，RFC4271相对于发布于1995年的RFC1771，有不少变化。举个例子，RFC1771是这么写的`When a BGP speaker advertises the route to a BGP speaker located in its own autonomous system, the advertising speaker shall not modify the NEXT_HOP attribute associated with the route`，用词是**'shall not'**，而到了RFC4271，类似的描述就变成了**'SHOULD NOT'**，并且指明了`unless it has been explicitly configured to announce its own IP address as the NEXT_HOP`。说明这并不是一个一开始就存在的东西。

- 向后兼容性问题，协议在发展，但是应该考虑向后的兼容性，如果`next-hop-self`被做成一个默认行为，这对网络来说未必是一个好事，遵循新标准和旧标准的设备可能会存在互操作和兼容上的问题。

- 最后，厂家为什么不做成一个默认行为？这点我朋友

   

  [@如月千早](https://www.zhihu.com/people/0dc263021fa274673b9a7c8866956574)

   

  的说法我非常认同——厂家压根就不应该做这样的事，这是在越俎代庖，无论什么样的功能，厂家一定是提供给客户一个开关好于做成一个默认行为。而且从遵循标准这条原则上来说，厂家也不应该做这种事。没有标准的协议和方案最终就变成了彻头彻尾的商业游戏，无法大规模应用，也给客户带去很多困扰，在这一点上，现在的VXLAN/EVPN是个血淋淋的例子。