## BGP--draft

初学者容易进入一个误区：喜欢从人的角度来揣摩路由器的行为，而不是以路由器的角度来看问题，所以就会觉得莫名其妙。



### next-hop

Error: The next-hop address is invalid.

怎么判断路由器下一跳是否无效

路由器是什么？是一台机器啊，**你不告诉它货物（IP包）的中转地址，它是不会知道的，所以需要明确地告诉它**，

同学们对于**多路访问链路**，比如以太网、帧中继，ATM网络概念不是很清晰，所谓多路访问，就是这个接口连接的不是一台设备，也可能是多个网段的设备，**需要明确地知道是和哪台设备做二层的通信**，所以需要明确地知道其IP地址，然后再解析为二层地址。

而**点对点链路**，如PPP、HDLC，可以理解为一根水管，货物可以顺着水管流到对端，而无需任何二层地址，所以自然也不需要对端的IP地址。

再重申一下，**为何需要下一跳的IP地址？
因为需要知道其二层地址！**

分组转发，接力棒嘛，路由器的路由算法中下一跳完全是为了灵活简化，每个端口有固定的IP，但是不一定是最优的。只能靠这种方式随时调整，只管自己不管其他的事情。

作者：车小胖
链接：https://www.zhihu.com/question/263357419/answer/268276017
来源：知乎
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。



作者：哆啦不是梦
链接：https://www.zhihu.com/question/263357419/answer/268510116
来源：知乎
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。



以太网环境下，设备之间的通信不仅仅需要知道对方的IP地址，还需要知道对方的MAC地址，如果你的静态路由是只写了下一跳IP地址，那么在设备的路由表中，你会发现对应的路由条目没有出接口，这说明什么？说明设备需要针对下一跳再查一次路由表，我该如何到达这个下一跳啊？设备发现，这个下一跳和我直连啊，因此三层没问题了，二层呢？ARP啊！！因此本地设备会直接对下一跳做ARP，获取MAC地址，封装数据包，转发，搞定，你网络就通了。

但是你要是写了个出站接口呢？那关键点还是在于路由表，这里有一个我个人觉得很重要的点，就是这个静态路由在路由表的表现形式，你发现了吗？和直连路由是一样的！！！尽管目标网段和你本地设备不直连，但是因为你静态路由写的是出站接口，设备自己认为就好像这个网段直接连接在了这个出接口上，这会发生什么事情？设备发现，既然是直连的，那我直接对这个目标网段做ARP啊，好家伙，问题来了，设备直接对一个非直连网段做了ARP请求，很显然，这个请求真的能到达目的地吗？显然不能，只会被下一跳设备收到，那么下一跳设备收到了这个ARP请求怎么办呢？设备发现这个不是找我的啊，因此按照我们的想法，设备会丢弃掉这个ARP请求，对吧，但是思科设备的以太网接口默认情况下开启了代理ARP功能，什么是代理ARP啊？就是设备收到一个不是请求自己的MAC地址的ARP请求的时候，不会立刻丢弃，而是会查找自己的路由表看自己有没有去往目标网段的路由条目，如果有，就把自己的MAC地址以ARP应答的形式发送出去，没有才丢弃这个ARP请求呢，这样以来，如果你下一跳设备的路由条目正确，设备会把自己的MAC地址响应给本地设备，你发现什么问题了吗？本地设备歪打正着的获取到了真正下一跳的MAC地址，数据也能正常转发了，可以你如果在下一跳设备上把代理ARP功能关闭掉，你就会发现，你ping不通了，这就是你为什么静态路由只写出接口，有时候能通，有时候不通的原因。



路由器路由表的nexthop，到底是本路由器自身的端口呢，还是连接自身端口的对方的端口呢



如下图，①中R1到10.0.12.0/24的nexthop是10.0.12.1，是R1自己的端口；②中R1到10.0.24.0/24的nexthop是10.0.12.2，是R2的端口，但是R1到10.0.24.0/24不是也应该先经过自己的10.0.12.1口，然后再经过R2的10.0.12.2口吗?

我理解的是路由②的nexthop应该是10.0.12.1，10.0.12.1的nexthop才是10.0.12.2。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/next-hop.png)





nexthop写地址和写interface的区别。 简单说吧，在p2p接口下，两种配法在主流路由器实现基本没啥区别。在multipoint 情况下，有区别。写nexthop 需要做next hop的L2 resolve，写interface时候，设备会认为这是direct connected，所以会对转发目标地址直接做L2 Resolve，注意是对转发包文的Destination 做L2 resolve，并不是没有L2解析。这种场景在multibind接口有使用，如果multibind接口 enable arp proxy，那么下面设备根本不需要配置gateway，直接静态指向interface就ok。国内主要有些地方做点到点以太上网专线有用这种solution。思科有个文档说的很清楚

如果出接口是点到点接口比如hdlc或ppp封装的接口是可以的。

但出接口是以太网就不行了，因为以太网接口是广播型接口，可以经过交换机或hub连接多个下一跳设备。**不指定下一跳IP的话路由器无法通过arp解析到下一跳的MAC地址，从而无法完成出向以太网帧报文头的封装。**

> 哈哈哈 ，对，不指定下一跳直接出不去了--

以上知识在Juniper或其他厂商文档里都有详细说明



#### Next Hop IP

先使用static Route设定好Next Hop IP。

```bash
# static route
[ar1]ip route-static 10.20.0.0 255.255.0.0 10.10.10.2

# Router IP
[ar1]display ip interface brief
*down: administratively down
^down: standby
(l): loopback
(s): spoofing
...

Interface                         IP Address/Mask      Physical   Protocol  
GigabitEthernet0/0/0              192.168.1.1/24       up         up        
GigabitEthernet0/0/1              10.10.10.1/16        up         up        

## Switch IP
[Huawei]display  ip interface brief
*down: administratively down
^down: standby
(l): loopback
(s): spoofing
...

Interface                         IP Address/Mask      Physical   Protocol  
...   
Vlanif2                           10.20.10.1/16        up         up       
# svi 连接 Router
Vlanif100                         10.10.10.2/16        up         up  
```

假设R1要发送Packet到10.20.10.2，根据Route Table，它会送到Next Hop 10.10.10.2。但它不知道10.10.10.2（Switch SVI）在哪里呢（不知道MAC地址），所以它要做一次Recursive Lookup，然后找到10.10.0.0/16的Next Hop Interface是G0/0/1。然后在G0/0/1上发一个ARP Request 问10.20.10.2的MAC Address，于是Switch 1回应并发出ARP Reply使得R1得知MAC Address并记录在ARP Table之中。

以后不管Destination是10.20.10.3、10.20.10.4(any IP of sub net 10.20.0.0/16)，都不需要再发ARP请求了，因为Next Hop 10.10.10.2已经在ARP Table了

#### Next Hop Interface

用Interface来当Next Hop情况稍有不同。假设R1要发送Packet到10.20.10.2，根据Route Table，发现Next Hop是Interface G0/0/1，没有Next Hop IP，Router只能在接口G0/0/1上发起查询10.20.10.2 MAC的请求，与路由器相连的Switch VLANIF IP 10.10.10.2不是10.20.10.2但是Switch会回复自己的MAC，原因就是Switch 的Proxy ARP默认是Enable，即如果Switch的Route Table中有10.20.10.2的地址会，Switch会通过Proxy ARP使用Switch Interface MAC 回复Router 1的解析。

```bash
## arp欺骗
[Huawei]display  arp
IP ADDRESS      MAC ADDRESS     EXPIRE(M) TYPE INTERFACE      VPN-INSTANCE      
                                          VLAN 
------------------------------------------------------------------------------
10.10.10.2      4c1f-cc47-269b            I -  Vlanif100
10.10.10.1      00e0-fca6-6c66  17        D-0  GE0/0/1
                                          100
10.20.10.1      4c1f-cc47-269b            I -  Vlanif2

[Huawei]display  ip routing-table 
Route Flags: R - relay, D - download to fib
...
Destination/Mask    Proto   Pre  Cost      Flags NextHop         Interface
        0.0.0.0/0   Static  60   0          RD   10.10.10.1      Vlanif100
      10.10.0.0/16  Direct  0    0           D   10.10.10.2      Vlanif100
      10.20.0.0/16  Direct  0    0           D   10.20.10.1      Vlanif2
```



但是有个问题，每个Destination IP查询ARP就会在ARP TABLE多一个条记录，导致ARP table条目激增，影响路由器效率。

```bash
[ar1]display  arp
IP ADDRESS      MAC ADDRESS     EXPIRE(M) TYPE        INTERFACE   VPN-INSTANCE 
                                          VLAN/CEVLAN PVC                      
------------------------------------------------------------------------------
192.168.1.1     00e0-fca6-6c65            I -         GE0/0/0
10.10.10.1      00e0-fca6-6c66            I -         GE0/0/1
10.10.10.2      4c1f-cc47-269b  11        D-0         GE0/0/1
10.10.10.4      4c1f-cc47-269b  11        D-0         GE0/0/1
10.10.10.7      4c1f-cc47-269b  11        D-0         GE0/0/1
10.10.10.8      4c1f-cc47-269b  11        D-0         GE0/0/1
```

且当interface Down时，基于接口的route都会消失。

#### Next Hop Interface and IP

如果必须使用Next Hop Interface来限制Packet送到某个Outgoing Interface的话，最好把Next Hop IP也一并写进去，这样可以避免ARP Record记录多个Destination IP。

```bash
# 这样G0/0/1就一条arp记录
ip route-static 10.20.0.0 255.255.0.0 G0/0/1 10.10.10.2
ip route-static 10.30.0.0 255.255.0.0 G0/0/1 10.10.10.2
ip route-static 10.40.0.0 255.255.0.0 G0/0/1 10.10.10.2
```

通过这个设定，Packet必然使用G0/0/1作为Outgoing Interface而Next Hop IP则使用10.10.10.2.但如果G0/0/1 Down了，所有基于Interface 作为出口的Route 就会消失。

end

```bash
#
bgp 100
 non-stop-routing
 group evpn internal
 peer evpn connect-interface LoopBack0
 peer 192.168.101.248 group evpn
 peer 192.168.101.252 group evpn
 #
 address-family l2vpn evpn
  undo policy vpn-target
  peer evpn enable
  peer evpn reflect-client
#
line vty 0 63
 authentication-mode scheme
 user-role network-admin
 user-role network-operator
#
 ip route-static vpn-instance mgmt 0.0.0.0 0 192.168.11.240
#
 vtep enable
#
 ssh server enable
# 
local-user admin class manage

```

