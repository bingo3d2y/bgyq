## 路由交换原理：Frame和Packet

哈哈哈， 卧槽，原来如此，IP不变，MAC一直在变，即物流的卡车或者人一直在换，目的地会一直保证不变。

0.0 MAC是具体的物理设备，一跳跳的实体，IP的作用是寻址（引导mac跳的方向性），Port的作用是复用，TCP的作用是保障传输。

> 目标mac会被替换成下一跳网关地址，但是网关的IP是不会出现在IP Header中的

 **二层的PDU叫做Frame;    IP的叫做Packet;    TCP的叫做Segment；    UDP的叫做Datagram。**

#### 网关

基础知识：

* 网关的作用：将网关MAC替换成数据包中src的mac地址，且网关的IP不会出现在任何网络包头中。

* 网关的IP地址毫无意义，它的作用就是引流。

  在传统物理网络中，网关负责将流量引流到接入层交换机，然后使得数据包可以被路由。

  在Calico方案中，网关将数据包引流至主机的caliXXX设备中，从而将二三层流量全部转换层三层流量，借助host网络拓扑开始路由。

#### RIB 表/路由表

Routing Info Base，RIB或者叫Routing Table.

==RIB作为控制平面（接受所有路由协议产生的路由信息），只告诉数据往哪个方向跳；而FIB作为转发层面，则告诉数据包具体怎么跳（从哪个）==

RIB 存储所有的路由信息。

当 IP 路由从 RIB 拷贝到 FIB 时，它们的下一跳信息被明确地分析出来，包括下一跳的具体端口，以及如果到下一跳有多条路径时，每条路径的具体端口。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/路由协议-rib-fib.jpg)

这种 RIB 加 FIB 的结构，使用控制平面的 RIB 和转发平面的 FIB 分离。这种分离使路由器的性能更加有连续性。

因此，以后再衡量路由器或三层交换机时，一定要检查路由表和 FIB 表的大小。

RIB，全；FIB，快。

####  FIB表

Forwarding Information Base，FIB表中每条转发表项都**指明到达某网段或主机的报文应该通过路由器的哪个物理接口或逻辑口发送，然后就可到达该路径的下一个路由器**，或者不再经过别的路由器而传送到直接相连的网络中的目的主机。

> 流弊，流弊。这个真正告诉了data flow 走哪个物理口。

FIB，二层转发，指定data从哪个物理口传输。

```bash
$ man route
...
 -F     operate on the kernel's FIB (Forwarding Information Base) routing table.  This is the default.
...
root@Dodo:~# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.21.175.253  0.0.0.0         UG    100    0        0 eth0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
172.18.0.0      0.0.0.0         255.255.0.0     U     0      0        0 br-0a3f6ee27391
172.21.160.0    0.0.0.0         255.255.240.0   U     0      0        0 eth0
172.21.175.253  0.0.0.0         255.255.255.255 UH    100    0        0 eth0

```

end

#### ARP and FDB表

ARP表：IP和MAC的对应关系；

FDB表(Forwarding Data Base)：MAC+VLAN和PORT的对应关系；

两个最大的区别在于ARP是三层转发，FDB是用于二层转发。也就是说，就算两个设备不在一个网段或者压根没配IP，只要两者之间的链路层是连通的，就可以通过FDB表进行数据的转发！

FDB表的最主要的作用就是在于交换机二层选路，试想，如果仅仅有ARP表，没有FDB表，就好像只知道地名和方位，而不知道经过哪条路才能到达目的地，设备是无法正常工作的。FDB表的作用就在于告诉设备从某个端口出去就可以到某个目的MAC。

那么FDB表是怎么形成的呢？很简单，交换机会在收到数据帧时，提取数据帧中的源MAC、VLAN和接收数据帧的端口等组成FDB表的条目。当下次有到该VLAN中的该MAC的报文就直接从该端口丢出去就OK了。

原文链接：https://blog.csdn.net/lanlicen/article/details/6333694

#### ADJ表

**邻接表信息取至于ARP表**，其中FIB中的每一个下一跳的2层报头重写信息，如果网络中两个节点之间距离一跳之内，那么就称为邻接节点，**邻接表维护所有FIB条目的2层下一跳地址和链路层报头信息，每发现一个邻接节点。路由器就会填充邻接表，每次创建一个链接条目，路由器就会预先算出邻接节点的链路层报头并将其储存在邻接表中**

每次，ARP解析到mac都会在ADJ表中记录。

https://www.bilibili.com/read/cv575583/

https://www.bilibili.com/read/cv575583/

https://www.bilibili.com/read/cv575583/

0.0 

#### ARP 

##### ARP 同网段

例如：

A的地址为：IP：192.168.10.1 MAC: AA-AA-AA-AA-AA-AA
B的地址为：IP：192.168.10.2 MAC: BB-BB-BB-BB-BB-BB

根据上面的所讲的原理，我们简单说明这个过程：A要和B通讯，A就需要知道B的以太网地址，于是A发送一个ARP请求广播（谁是192.168.10.2 ，请告诉192.168.10.1），当B收到该广播，就检查自己，结果发现和自己的一致，然后就向A发送一个ARP**单播应答**（192.168.10.2 在BB-BB-BB-BB-BB-BB）。

> 注意，arp是广播询问，单播应答。

##### ARP 跨网段

   不同网段的主机通信时，主机会封装网关（通常是路由器）的mac地址，然后主机将数据发送给路由器，后续路由进行路由转发，通过arp解析目标地址的mac地址，然后将数据包送达目的地。具体过程分析如下：

 图片： Host_A（3.1）  <----> （3.X）Router（4.X） <----> Host_B（4.1）

Router有两个ip和mac，对应两个接口

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/arp-different-network.png)



如上图，主机A、B通过路由器连接，属于两个不同的网段子网掩码24（255.255.255.0）

1、主机A有数据发往主机B，数据封装IP之后发现没有主机B的mac地址；然后查询ARP，ARP回应：“我在192.168.3.0/24网段，目标地址在192.168.4.0/24,不属于同一网段，**需要使用默认网关**”；ARP发现默认网关是192.168.3.2，但是没有网关mac地址，需要先进行查询；

> 这里注意默认网关和默认路由

2、主机将数据包先放到缓存中，然后发送ARP查询报文：封装自己的mac地址为源mac，目标mac地址写全F的广播地址，请求网关192.168.3.2的mac地址。然后以广播方式发送出去；

> 广播地址（Broadcast Address）：这是一个特殊地址，当数据包可以寻址到该地址时，可以帮助所有设备打开和处理信息。例如，MAC 地址，格式为 0xFFFFFFFF 是一种广播地址；IP 地址255.255.255.255是通用广播地址。任何设备都将打开寻址到广播地址的信息，并将它们传送到下一个工作站。
>
> 0.0 MAC的广播地址就是0xFFFFFFFF

3、路由器收到广播数据包，首先将原192.168.3.1添加到自己的mac地址表中，对应mac地址为0800.0222.2222。路由发现是请求自己的mac地址，然后路由回复一个ARP应答：封装自己的IP地址为源IP自己的mac地址为源mac，主机A的IP为目的IP主机A的mac为目的mac，发送一个单播应答“我是192.168.3.2.我的mac地址为0800.0333.2222”；

4、主机收到应答后，将网关mac地址对应192.168.4.2（跨网关通信，其他网段IP地址的mac地址均为网关mac），然后将缓存中的数据包，封装网关mac地址进行发送；

5、路由收到数据包，检查目的IP地址，发现不是给自己的，决定要进行路由，然后查询路由表，需要发往192.168.4.0网段中的192.168.4.2地址。路由准备从相应接口上发出去，然后查询mac地址表，发现没有主机B的映射。路由器发送arp请求查询主机B的mac地址（原理同2、3步，主机B收到请求后首先会添加网关的mac地址，然后单播回复arp请求）；

6、路由器收到主机B的mac地址后，将其添加到路由mac地址表中，然后将缓存中的数据2层帧头去掉，封装自己的mac地址为源mac，主机B的mac地址为目的mac（源和目的IP地址不变），加上二层帧头及校验，发送给主机B；

7、主机B收到数据之后，进行处理，发送过程结束；

8、如果主机B收到数据后进行回复，主机B会进行地址判断，不在同一网段，然后决定将数据发送给网关，主机B查询mac地址表获得网关mac地址，将数据封装后发送（arp地址解析的过程不再需要了，mac地址表条目有一定的有效时间），网关收到数据后直接查询mac表，将二层帧mac地址更改为A的mac发送出去。如此，主机A收到主机B的回复；

综上在跨网段通信过程中有以下过程：
1、判断地址是否同一网段
2、查询目的IP地址的mac（发送arp请求）

#### ARP代理（Proxy ARP）

https://blog.51cto.com/chenxinjie/1961255

卧槽，上面链接，最牛逼的是下面的评论。

Proxy ARP is a technique by which a proxy device on a given network answers the ARP queries for an IP address that is not on that network.The proxy is aware of the location of the traffic's destination, and offers its own [MAC address](https://en.wikipedia.org/wiki/MAC_address) as the (ostensibly final) destination.[[1\]](https://en.wikipedia.org/wiki/Proxy_ARP#cite_note-1) The traffic directed to the proxy address is then typically routed by the proxy to the intended destination via **another interface** or via a [tunnel](https://en.wikipedia.org/wiki/Tunneling_protocol).

0.0 在一个没网络的环境里，会用到Proxy ARP

卧槽，calico，calico，calico就是这样通过Proxy ARP将pod的流量都转到三层的virtual Interface上的(calicoxxxxx)

然后，Calico的其他组件来管理这些interface.

==arp代理是说没有路由且目的主机不在同一子网的情况下，才会用到这个功能。==

平时PC访问不同网段IP时，例如(8.8.8.8）是有路由，直接查询FIB表找到出接口，查询ADJ表找到二层mac封装，就可以直接扔出去，此时不需要Proxy ARP.

> Proxy ARP and ARP是互补的关系。

代理ARP是一个"善意的欺骗"，当电脑要跨网段访问外网设备时，网关设备用自己的MAC返回；

##### 典型应用：calico

0.0 利用Proxy ARP 完成 pod流量的引流。因为，pod中的默认网关/路由是不存的IP。



任意选择 k8s 集群中的一个节点作为实验节点，进入容器 A，查看容器 A 的 IP 地址：

```bash
$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
3: eth0@if771: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1440 qdisc noqueue state UP
    link/ether 66:fb:34:db:c9:b4 brd ff:ff:ff:ff:ff:ff
    inet 172.17.8.2/32 scope global eth0
       valid_lft forever preferred_lft forever
       
```

这里容器获取的是 /32 位主机地址，表示将容器 A 作为一个单点的局域网。

瞄一眼容器 A 的默认路由：

```bash
$ ip route
default via 169.254.1.1 dev eth0
169.254.1.1 dev eth0 scope link
```

看到这个默认路由，符合Proxy ARP(孤岛网络环境),因为这个默认路由地址就不存在(或者说不可以)。

现在问题来了，从路由表可以知道 `169.254.1.1` 是容器的默认网关，但却找不到任何一张网卡对应这个 IP 地址，这是个什么鬼？

莫慌，先回忆一下，当一个数据包的目的地址不是本机时，就会查询路由表，从路由表中查到网关后，它首先会通过 `ARP` 获得网关的 MAC 地址，然后在发出的网络数据包中将目标 MAC 改为网关的 MAC，而网关的 IP 地址不会出现在任何网络包头中。也就是说，没有人在乎这个 IP 地址究竟是什么，只要能找到对应的 MAC 地址，能响应 ARP 就行了。

想到这里，我们就可以继续往下进行了，可以通过 `ip neigh` 命令查看一下本地的 ARP 缓存：

```bash
$ ip neigh
169.254.1.1 dev eth0 lladdr ee:ee:ee:ee:ee:ee REACHABLE

### 这个ee:...:ee 正好就行node上的calico interface
$ ip addr
...
771: calicba2f87f6bb@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1440 qdisc noqueue state UP group default
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 14
    inet6 fe80::ecee:eeff:feee:eeee/64 scope link
       valid_lft forever preferred_lft forever
...
## 呐，pod的8.2 ip 由 calicba2f87f6bb 处理
$ ip route 
...
172.17.8.2 dev calicba2f87f6bb scope link
...
```

这个 MAC 地址应该是 Calico 硬塞进去的，而且还能响应 ARP。但它究竟是怎么实现的呢？

我们先来回想一下正常情况，内核会对外发送 ARP 请求，询问整个二层网络中谁拥有 `169.254.1.1` 这个 IP 地址，拥有这个 IP 地址的设备会将自己的 MAC
地址返回给对方。但现在的情况比较尴尬，容器和主机都没有这个 IP 地址，甚至连主机上的端口 `calicba2f87f6bb`，MAC 地址也是一个无用的 `ee:ee:ee:ee:ee:ee`。按道理容器和主机网络根本就无法通信才对呀！所以 Calico 是怎么做到的呢？

这里我就不绕弯子了，实际上 Calico 利用了网卡的代理 ARP 功能。代理 ARP 是 ARP 协议的一个变种，当 ARP 请求目标跨网段时，网关设备收到此 ARP 请求，会用自己的 MAC 地址返回给请求者，这便是代理 ARP（Proxy ARP）。

查看是否开启代理 ARP：

```bash
## 在node上看还是pod里
$ cat /proc/sys/net/ipv4/conf/calicba2f87f6bb/proxy_arp
1
```

如果还不放心，可以通过 tcpdump 抓包验证一下：

```bash
$ tcpdump -i calicba2f87f6bb -e -nn
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on calicba2f87f6bb, link-type EN10MB (Ethernet), capture size 262144 bytes


14:27:13.565539 ee:ee:ee:ee:ee:ee > 0a:58:ac:1c:ce:12, ethertype IPv4 (0x0800), length 4191: 10.96.0.1.443 > 172.17.8.2.36180: Flags [P.], seq 403862039:403866164, ack 2023703985, win 990, options [nop,nop,TS val 331780572 ecr 603755526], length 4125
14:27:13.565613 0a:58:ac:1c:ce:12 > ee:ee:ee:ee:ee:ee, ethertype IPv4 (0x0800), length 66: 172.17.8.2.36180 > 10.96.0.1.443: Flags [.], ack 4125, win 2465, options [nop,nop,TS val 603758497 ecr 331780572], length 0
```

总结：

1. Calico 通过一个巧妙的方法将 workload 的所有流量引导到一个特殊的网关 169.254.1.1，从而引流到主机的 calixxx 网络设备上，最终将二三层流量全部转换成三层流量来转发。

   0.0 利用Proxy ARP 完成 pod流量的引流。

2. 在主机上通过开启代理 ARP 功能来实现 ARP 应答，使得 ARP 广播被抑制在主机上，抑制了广播风暴，也不会有 ARP 表膨胀的问题。

##### calico组网验证

cnblogs.com/ryanyangcs/p/11273040.html

先在 Host0 上执行以下命令：

```bash
$ ip link add veth0 type veth peer name eth0
$ ip netns add ns0
$ ip link set eth0 netns ns0
$ ip netns exec ns0 ip a add 10.20.1.2/24 dev eth0
$ ip netns exec ns0 ip link set eth0 up
$ ip netns exec ns0 ip route add 169.254.1.1 dev eth0 scope link
$ ip netns exec ns0 ip route add default via 169.254.1.1 dev eth0
$ ip link set veth0 up
$ ip route add 10.20.1.2 dev veth0 scope link
$ ip route add 10.20.1.3 via 192.168.1.16 dev ens192
$ echo 1 > /proc/sys/net/ipv4/conf/veth0/proxy_arp
```

在 Host1 上执行以下命令：

```bash
$ ip link add veth0 type veth peer name eth0
$ ip netns add ns1
$ ip link set eth0 netns ns1
$ ip netns exec ns1 ip a add 10.20.1.3/24 dev eth0
$ ip netns exec ns1 ip link set eth0 up
$ ip netns exec ns1 ip route add 169.254.1.1 dev eth0 scope link
$ ip netns exec ns1 ip route add default via 169.254.1.1 dev eth0
$ ip link set veth0 up
$ ip route add 10.20.1.3 dev veth0 scope link
$ ip route add 10.20.1.2 via 192.168.1.32 dev ens192
$ echo 1 > /proc/sys/net/ipv4/conf/veth0/proxy_arp
```

网络连通性测试：

```bash
# Host0
$ ip netns exec ns1 ping 10.20.1.3
PING 10.20.1.3 (10.20.1.3) 56(84) bytes of data.
64 bytes from 10.20.1.3: icmp_seq=1 ttl=62 time=0.303 ms
64 bytes from 10.20.1.3: icmp_seq=2 ttl=62 time=0.334 ms
```

实验成功！

具体的转发过程如下：

1. ns0 网络空间的所有数据包都转发到一个虚拟的 IP 地址 169.254.1.1，发送 ARP 请求。
2. Host0 的 veth 端收到 ARP 请求时通过开启网卡的代理 ARP 功能直接把自己的 MAC 地址返回给 ns0。
3. ns0 发送目的地址为 ns1 的 IP 数据包。
4. 因为使用了 169.254.1.1 这样的地址，Host 判断为三层路由转发，查询本地路由 `10.20.1.3 via 192.168.1.16 dev ens192` 发送给对端 Host1，如果配置了 BGP，这里就会看到 proto 协议为 BIRD。
5. 当 Host1 收到 10.20.1.3 的数据包时，匹配本地的路由表 `10.20.1.3 dev veth0 scope link`，将数据包转发到对应的 veth0 端，从而到达 ns1。
6. 回程类似

通过这个实验，我们可以很清晰地掌握 Calico 网络的数据转发流程，首先需要给所有的 ns 配置一条特殊的路由，并利用 veth 的代理 ARP 功能让 ns 出来的所有转发都变成三层路由转发，然后再利用主机的路由进行转发。这种方式不仅实现了同主机的二三层转发，也能实现跨主机的转发。

#### 默认网关和默认路由:fire:

default-gateway for layer2

default route for layer3

--

we use ip default-gateway when ip routing is disabled to provide a default gateway to a device like a L2 LAN switch

we use ip route 0.0.0.0 0.0.0.0 Gw-ipaddress when ip routing is enabled

if ip routing is enabled ip default-gateway command is ignored

if ip routing is disabled ip route 0.0.0.0 0.0.0.0 is not effective



下面是思科的牛逼解释

Default route which is also known as the gateway of last resort, is used in forwarding packets whose destination address does not match any route in the routing table. In IPv4 the CIDR notation for a default route is 0.0.0.0/0 and ::/0 in IPv6. Now since the both the host/network portion and the prefix length is zero a default route is the shortest possible match. In previous lessons in which we discussed basics of IP Routing we know that a router when performing a route lookup will select a route with longest possible match based on CIDR specifications, however if packet does not match any route in the routing table it will match a default route, the shortest possible route, if it exists in the routing table. **A default route is very useful in network where learning all the more specific routes is not desirable such as in case of stub networks.** A default is immensely useful when a router is connected to the Internet as without a default route the router must have the routing entry for all networks on the Internet which are in numbers of several hundred thousand, however with single route configured as the default the router will only need to know the destinations internal to the administrative domain and will forward IP packets for any other address towards the Internet using the default route.

**The default gateway is a device such as a router that serves as the edge devices providing an access point to other networks and is used to forward IP packets which does not match any routes in the routing table.** We usually encounter the concept of default gateways in our daily computer life. The LAN configuration in our windows requires us to specify the IP address, Subnet Mask and the Default Gateway to access the Internet. The default gateway IP address is the IP address of the CPE or Internet modems which provide the connectivity to the Internet, now since the Internet has several hundred thousand routes which we cannot install in our table, we simply tell our computer to forward all packets destined to the Internet to this device. Again the CPE will itself have a default route and gateway configured which will point to the ISP access device.



Router R1 connects two internal networks of an organization and provides connectivity between them, it also connects to the Internet via an ISP CPE device to provide Internet access to the users. On R1 a default route is only needed to provide Internet connectivity which is configured as shown below.

Router(config)#**ip route 0.0.0.0 0.0.0.0 192.168.1.1**

All packets not matching a specific match in the routing table will be matched using the default route and forwarded to 192.168.1.2 which is the default gateway for R1. Similarly on every computer connected to either of the organization’s LAN will have the IP address of the router configured as the default gateway.

Now at the packet level what happens is that when a device attached to the network needs to communicate with another device it will first check whether the other device is on the same network or another network by comparing the IP address of the other device with subnet mask assigned to itself. If it is on another network the it will create a packet with the source IP of itself and destination of of the other device. However the layer 2 frame will have the source of address of the device itself while the destination address will be the layer 2 address of the gateway. As the packet is routed by intermediate routers the packet will remain the same i.e no change in source and destination IP addresses while the layer 2 frame will change as it crosses networks i.e the source and destination mac addresses will be changed after every layer 3 hop.

This brings us to end of this lesson in which we discussed the concept of default routes and default gateways. Both of these concepts are very essential as network administrator you will encounter both of these in the daily job life.

整理下:arp 代理，默认网关（请求目的ip不在奔网段时，使用默认网关地址作为目标mac。他需要去网关进行转发），不在同一网段的arp，arp广播查询，单播响应
网关ip不会出现在任何ip header中。
arp代理和默认网关的区别
嗷嗷，arp代理更灵活些，可以指定任意的Mac
https://www.cnblogs.com/ryanyangcs/p/11273040.html
https://blog.csdn.net/jazzsoldier/article/details/52635744



####数据如何在网络中传输（重点）

wow，还有PoE(Power over Ethernet)功能的设备,网线捎带通电功能。

就TCP/IP网络模型来说：

- 链路层：负责封装和解封装IP报文，发送和接受ARP/RARP报文等。
- 网络层：负责路由以及把分组报文发送给目标网络或主机。
- 传输层：负责对报文进行分组和重组，并以TCP或UDP协议格式封装报文。
- 应用层：负责向用户提供应用程序，比如HTTP、FTP、Telnet、DNS、SMTP等

当数据数据链路层传播的时候通常叫“帧”，当一个帧被接收并交由第二层处理：剥开帧头帧尾，获得数据包（对于第二层来说它只认识帧头和帧尾，其它包括包头等都是帧承载的普通数据）；完了这个包（packet）被交给第三层：它能识别包头，得到被包在里面的信息（这信息就包含第四层比如tcp数据报头，但对于第三层来说报头也只是它所承载的普通数据），第三层完事之后把去掉包头的数据给第四层，这坨数据就是数据报（英文datagram / segment），那么第四层就认得报头然后干该自己干的事了。

0.0 下面给一个数据包传输的图，**数据包真实传输是在链路层上**，网络层负责指路，传输层负责保障数据不丢失，应用层负责使用数据。所以啊，**mac在链路层上会被一直修改，而IP地址除了额外设置NAT是不会改变IP的**。

网络拓扑：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/packet-1.jpg)

起始帧

目标mac会被替换成下一跳网关地址。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/packet-2.jpg)

经过一个路由器转发后，变化的帧。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/packet-3.jpg)

第二次变化：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/packet-4.jpg)

MAC地址如果不开proxy，只在本地网段内使用
A跨网段发给B的时候，A发出的数据包目的MAC地址写的是A的网关MAC

> 通过arp获取到网关mac，然后封包.

A的网关转发这个包的时候目的MAC写的是路由表中下一跳next hoop的MAC地址



0.0 在不考虑代理的情况下，数据包每到一个三层，就把源MAC替换成自己的出接口MAC，目标MAC替换成下一跳MAC，不跨三层不替换。



#### 路由器工作原理:honeybee:

传统的路由决策，路由器需要对网络数据包进行解包，再根据目的IP地址计算归属的FEC。

传统的路由网络里面，当一个（无状态的）网络层协议数据包（例如IP协议报文）在路由器之间游荡时，每个路由器都是独立的对这个数据包做出路由决策。**路由决策就是路由器决定数据包如何路由转发的过程。**在这里指，每个路由器都需要分析包头，根据网络协议层的数据进行运算，再基于这些分析和运算，独立的为数据包选择下一跳（next hop），最后通过next hop将数据包发送出去。以IP协议报文为例，路由决策是基于目的IP地址，路由器根据目的IP地址，选择路由条目，再做转发。路由决策可以认为是由两部分组成：

- 分类，将特定的数据包归属为一个等价转发类（Forwarding Equivalence Classes，FECs）
- 查找，查找FEC对应的next hop

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/路由器-route-machine-1.png)

对于同一个路由器来说，同一个FEC必然对应同一个next hop，那么属于同一个FEC的所有网络数据包必然会走同一条路径转发出去。（注，在多链路负载均衡的情况下，一个FEC也可能对应一组next hop，但是逻辑上还是能看成是一个next hop，因为殊途同归！)

具体到IP协议报文，当多个IP协议报文的目的地址都对应路由器的一条路由，且这条路由是所有路由里面最长匹配（longest match）的路由，那么对于这个路由器来说，就会认为这两个IP协议报文属于一个FEC。因此，这两个数据包就会走同一条路径出这个路由器。这就是我们最常见到的路由转发。



需要注意的是，这里的FEC是针对一个路由器的，而不是全局的。举个例子，目的地址为192.168.31.1和192.168.31.100的两个IP协议报文，第一个路由器具有192.168.31.0/24这条路由，那么在第一个路由器它们属于同一个FEC，都会被转发到第二个路由器。第二个路由器具有192.168.31.0/26和192.168.31.0/24两条路由，并且两条路由的next hop不一样。因为192.168.31.0/26能更精确的匹配192.168.31.1，所以192.168.31.1匹配第一条路由，而192.168.31.100匹配第二条路由，最终，这两个IP协议报文在第二个路由器被认为是不同的FEC，从不同的路径出去。这就是每个路由器都需要独立的做路由决策的原因之一。

路由器的工作原理如下图所示：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/路由器-route-machine-2.png)


由于每个路由器都需要独立的路由决策（虽然会有这样那样的缓存机制加速决策），而**路由器的收发队列一旦满了，就会丢包。所以在一个高流量，高容量的网络里面，无疑对每个路由器的要求都很高（否则就会丢包了！）**



#### 路由与多网卡环境：包转发流程

 **一个机器上只能有一个默认路由规则**`ip r add default IP dev if_name` 

but ,`/etc/sysconfig/network-scripts/ifcfg-bond2` 网卡设备配置文件中在`GATEWAY` 可以指定网卡的网关，此外还可以通过`ip route add default` 再加一个真正的全局默认网关（only one ）--

默认路由：Default route，是对IP中的目的地址找不到存在的其他路由时，路由器所选择的路由。

路由：通过互联的网络把信息从源地址传输到目的地址的活动。

路由缺失：路由是双向的请注意，多网卡环境容易出现路由缺失的情况，单网卡几乎体会不到，路由是双向的和路由缺失的特性，因为它就一个网卡，配合上默认路由，轻松搞定。只有网关默认路由能正常寻址就能保证数据回去。

* 一块网卡

  这种情况简单，没什么好说的，一个默认路由网关搞定一切。

  注意哟，默认网关只能设置一个。因为只有一块网卡，所以出口就一个嘛，正好去默认路由了

  `route add default gw 224.224.224.224 eth0`

* 两块或多块网卡

  极其容易出现**路由缺失**的情况。

  多块网卡，默认路由只有一条（只能指向一个网卡所在的网关），你要是不在给希望被访问的网卡加上路由则会导致你访问目的网段`21.49.22.0/24`时，访问不了。

  因为，就是你能路由到`21.49.22.0/24`,但是它回不去啊...没有路由指路它只能走默认路由了，但是默认路由的网关又是`172.29.33.254` 它一看...卧槽，不是我的网段的啊，就转发了... 但是前面防火墙没开端口，GG

  nono，从172出去，防火墙不允许了。。。。

  > 全公司这么多网段，我得一个个加？？？。
  >
  > 目前三个网卡的作用：
  >
  > bond0： 90 存储
  >
  > bond1： 21 业务，对外访问
  >
  > bond2： 172 云管，用于管理员连接
  >
  > 最终配置：
  >
  > master1-3，默认路由设置成172网段
  >
  > node1-3，默认路由设置成21网段。
  >
  > 不能在node上添加静态路由：`21.48.8.143`(我的主机) 到172网段，因为静态路由优先级大于默认路由，
  >
  > 这样导致，我的机器还是无法访问业务网段。--因为走到172网段通信时被防火墙拦截了。
  >
  > 0.0 
  
  当主机A发向主机B的数据流在网络层封装成IP数据包，IP数据包的首部包含了源地址和目标地址。主机A会用本机配置的24位IP网络掩码255.255.255.0与目标地址进行与运算，得出目标网络地址与本机的网络地址是不是在同一个网段中。如果不是将IP数据包转发到网关。

  在发往网关前主机A还会通过ARP的请求获得默认网关的MAC地址。在主机A数据链路层IP数据包封装成以太网数据帧，然后才发住到网关……也就是路由器上的一个端口。

  当网关路由器接收到以太网数据帧时，发现数据帧中的目标MAC地址是自己的某一个端口的物理地址，这时路由器会把以太网数据帧的封装去掉。路由器认为这个IP数据包是要通过自己进行转发，接着它就在匹配路由表。匹配到路由项后，它就将包发往下一条地址。

  路由器转发数据包就是这样，所以它始终是不会改IP地址的。只会改MAC.

  ```bash
  $ ip r
  default via 172.29.33.254 dev bond2 proto static metric 302 
  21.49.22.0/24 dev bond1 proto kernel scope link src 21.49.22.7 metric 303 
  90.16.1.0/24 dev bond0 proto kernel scope link src 90.16.1.31 metric 300 
  172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 
  172.29.33.0/24 dev bond2 proto kernel scope link src 172.29.33.4 metric 302 
  
  
  
  只要在pod 所在的node上加上路由就行了
  
  $ route add -net 22.48.8.0/24 gw 21.49.22.254
  或者
  $ ip r add 22.48.8.0/24 via 21.49.22.254 dev bond1
  
  [root@cs1-k8s-n1 ~]# ip r
  default via 172.29.33.254 dev bond2 proto static metric 302 
  21.49.22.0/24 dev bond1 proto kernel scope link src 21.49.22.7 metric 303 
  22.48.8.0/24 via 21.49.22.254 dev bond1 
  90.16.1.0/24 dev bond0 proto kernel scope link src 90.16.1.31 metric 300 
  172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 
  172.29.33.0/24 dev bond2 proto kernel scope link src 172.29.33.4 metric 302 
  
  0.0 emmm 添加默认路由
  $ ip r add default via 22.49.22.254 dev bond1
  0.0 
  ```
  
  end

#### 交换机基础知识

将Switch和linux Server来对比下，理解更快。

都要OS、核心处理引擎（CPU/背板带宽）和 网卡（吞吐量/包转发率）

#### Switch 几个指标

##### 背板带宽（相当于CPU的处理速度）

**背板带宽（交换带宽）表示的是我们的接口处理器或者接口卡和核心交换引擎之间的速度**。也就是说，交换机内部数据传输的速度。举个栗子，52口千兆全双工交换机，那么，你的交换机背板带宽就应该大于或者等于：`52*2*1 Gbps = 104 Gbps`。这里，52表示52个网口，2表示双工传输，1 Gbps表示千兆交换机。这里，假设52个口同时在接收和发送数据，并且都是满速的，那么同一个网口，数据进来要有1Gbps，出去也是1Gps。

1000Mps = 1Gps

##### 交换容量

交换容量和背板带宽的概念很类似，交换容量是指所有的口都在收、发数据，此时，交换机整体的带宽。52口千兆全双工交换机，应该也满足 52 * 2 * 1Gbps = 104Gbps，如果小于这个值，那么肯定有人在传输的时候，速度是小于 1 Gbps的。

##### 包转发率（即端口吞吐量，一般交换机达不到--）

交换机的包转发率，也称为端口吞吐量，是交换机在某个端口进行数据包转发的能力，单位通常为pps，叫包每秒，即每秒钟内所转发数据包的个数。

这里补充一个网络小常识：网络数据传输通过数据包，数据包的构成是传输的数据+帧头+帧间隙。网络中规定一个数据包最小为64字节，这里的64字节就是单纯的数据，加上8字节帧头( 前导符和帧开始符)和12字节帧间隙**，那么网络中最小的包就是84字节。**

那么一个全双工的千兆接口达到线速时包转发率就要=1000Mbps/((64+8+12)*8bit)＝1.488Mpps。

两者的关系：

交换机背板带宽代表了交换机总的数据交换能力，也是决定包转发率的一个重要指标。所以背板可以理解成电脑总线，背板越高处理数据能力越强，也就是包转发率越高。



这里，交换机的pps能力越强，越能搞定64Byte的小包。对于一个52口的全千兆全双工交换机，所有的52个网口都在发送64 Byte的小包，那么，我们交换机需要：

1.488 Mpps * 2 * 52 = 153.92 Mpps。显然，一般的交换机时达不到这样的能力的。

例如，思科的SG300-52，其pps最大为77.38 Mpps，也就是说，所有人都在发送64Byte的小包时，它是不能够处理的。

emm, 但是常见的网络场景也没有这么高的传输率--

#### 以太网 几个概念

车小胖： https://zhuanlan.zhihu.com/p/21318925

网络设计为了更加灵活而采用了各种隧道技术，加上了各种各样的头部封装，让原来可以正常通行的IP packet，因为加上新的头部信息臃肿（变长）而无法正常通行，需要做瘦身手术（分片），这个过程我们称之为IP Fragment，到达目的地再把这些IP fragmented packet重组成一个完整IP packet，这个过程我们称之为重组IP Reassemble。

让我们来罗列一下有哪些协议让包变长：PPPoE， 802.1q，QinQ ， MPLS， L2TP， GRE，IP Security，OTV ，VxLAN等，对于这些协议先不展开，先来了解一些影响MTU的因素。



##### 以太网帧：Ethernet Frame

标准的以太网帧，我们经常说的以太网帧长度是从图中 Destination MAC开始，FCS结束。网卡对网络层数据的操作是加以太网帧头、以太网帧尾FCS，很显然上层需要提供目的MAC地址，否则接口无从完成以太网帧的封装。这需要IP层需要事先完成和ARP的交互，解析出目的IP对应的目的MAC，这显然不能由网卡来完成。

网卡对物理层接收到的二进制流成帧处理，校验FCS，去掉以太网帧头，把载荷Payload 放在接收缓存，等待网络层取走。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/etherframe.jpg)

标准的以太网帧最大可以发送长度1518字节，指的就是这个。去掉以太网头14个字节，再去掉尾部的校验和FCS 4个字节，留给**上层协议**也就是([1518-14-4](tel:1518-14-4))=1500个字节，这个就是MTU的由来。上层协议加黑的原因是要引起大家的注意，这个上层协议如果是IP，那么就是IP MTU，如果是MPLS，就是MPLS MTU，如果是IPv6，那就是IPv6 MTU。

PS： 由于IPv6要求链路层所支持的最小MTU为1280，所以PMTU的值必须大于1280。
IPV6为何这么叼--



##### Ether Type: **以太网协议**

网卡用来分辨封装的是什么协议，然后再通知不同的协议模块来取走数据。



##### Payload: **载荷**

这个允许负荷的最大长度对应的就是负荷的最大传输单元，即MTU，标准的以太网帧，允许的最大负荷长度为1500字节，所以如果上层协议为IPv4，那就是IPv4 MTU=1500，所以经常看到主机的MTU为1500字节。



##### FCS: **校验码**

为了防止在传输过程中发生错误，数据发送方的网卡会计算一个校验码，覆盖整个以太网帧，并放在以太网帧尾部，发送出去，接收网卡需要对其进行校验，来决定是否接收。而如果不校验，一个错误的帧可能要到TCP、UDP才能被发现出来，这样的话会浪费很多CPU资源。CPU会说：屁大点的事都搞不定，还要劳烦朕，可以去自宫了。而如果网卡来进行校验，错了就默默地丢弃，不惊动高层，高层肯定偷偷乐开了花。

太强了，网卡层面直接校验保证的正确性... 666



##### IP层能发现IP包的损坏吗？

**IP头的校验码只覆盖IP头，保证关键信息如目的IP在传输过程没有差错，可以到达目的地，至于里面封装内容则由目的地主机负责校验，可以减少路由器的处理时间，提高转发效率。**

建议背诵全文...

##### 以太网帧长度上下限

标准以太网帧长度下限为：64 字节

标准以太网帧长度上限为：1518 字节

所有标准以太网的数据部分限制为： 46字节 ~ 1500字节

太网帧的上下限为：64-1518，这是业界标准；但有的厂家支持40-9214，这个企业标准就包含了业界标准，并不矛盾

最早的以太网工作方式：载波多路复用/冲突检测CSMA/CD，因为网络是共享的，即任何一个节点发送数据之前，先要侦听线路上是否有数据在传输，如果有，需要等待，如果线路可用，才可以发送。

假设A发出第一个bit位，到达B，而B也正在传输第一个bit位，于是产生冲突，冲突信号得让A在完成最后一个bit位之前到达A，这个一来一回的时间间隙slot time是57.6μs.

因为太网帧最小长度为576个bits，从而让最极端的碰撞都能够被检测到。这个576bit换算一下就是72个字节，去掉8个字节的前导符和帧开始符，以太网帧的最小长度为64字节。

在10Mbps的网络中，在57.6μs的时间内，能够传输576个bit，所有slot time为57.6μs.

> 57.6μs怎么计算的：
>
> 最初协议标志定义的Ethernet Frame的最小值：Preamble + SFD + MAC destination + MAC source + Ethertype + Payload + CRC = 7 + 1 + 6 + 6 + 2 + 46 + 4 = 72 Bytes（8 + 60 + 4） = 576 bit，而10Mb/s = 10bit/μs, 所以在链路层bit steam从A到B最小时间为 576 / (10bit/μs) = 57.6μs ，冲突信号要在A传输完最后一个bit前到达A，所以slot time是57.6μs 

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/etherframe-slot.jpg)

如果说以太网帧的最小长度64byte是由CSMA/CD限制所致，那最大长度1500byte又是处于什么考虑的呢？

IP头total length为两个byte，理论上IP packet可以有65535 byte，加上Ethernet Frame头和尾，可以有65535 +14 + 4 = 65553 byte。如果在10Mbps以太网上，将会占用共享链路长达50ms,这将严重影响其它主机的通信，特别是对延迟敏感的应用是无法接受的。

由于线路质量差而引起的丢包，发生在大包的概率也比小包概率大得多，所以大包在丢包率较高的线路上不是一个好的选择。

但是如果选择一个比较小的长度，传输效率又不高，拿TCP应用来说，如果选择以太网长度为218byte，TCP payload = 218 - Ethernet Header -IP Header - TCP Header=[218-18 - 20](tel:218-18 - 20)-20= 160 byte

那有效传输效率=160/218= **73%**

而如果以太网长度为1518，那有效传输效率=1460/1518=**96%**

通过比较，选择较大的帧长度，有效传输效率更高，而更大的帧长度同时也会造成上述的问题，于是最终选择一个折衷的长度：1518 byte ! 对应的IP packet 就是 1500 byte，这就是最大传输单元MTU的由来。

##### **Jumbo Frame**

以一个标准的 Ethernet 帧为例，如果想让 payload 里的数据包达到 1500 bytes （即三层 MTU）那么二层 MTU 的值应该设置为 1514 bytes (1500+14)。

最早的以太网是通过Hub或集线器来工作的，在任意时刻只能有一台主机发送，这种共享方式发送效率很低，而现代高速交换机则让每个连接交换机的主机工作在独占模式，带宽独享，可以同时收发，而且现在早已不是早期的10Mbps的带宽，而是1000M、10000M，即使发送大包也不会影响别的主机，影响的只是交换机的接收和发送队列，既然发送大包效率要比小包效率搞，而且特定的应用也有发大包的需求，比如NFS文件系统，那为什么不把接口MTU提高一些，再高一些呢？这是一个好主意，于是网卡、交换机、路由器网络接口可以实现更大的MTU，可以达到>9000字节的大小，我们称这种远大于标准以太帧尺寸的帧为巨型帧Jumbo Frame 。

于是网络接口提供可以修改MTU的配置命令，比如缺省为1500，可以修改为1508以支持QinQ，或者1512以支持802.1q Mpls label，这样既可以支持终端用户标准1500 字节IP packet，又可以避免分片。

有一点需要说明，二层交换机的接口，我们可以看成一块普通的网卡，网卡工作在数据链路层，所以分片不是它的职责，如果一个帧需要从交换机一个接口发送出去，而帧的长度>接口MTU，怎么办？丢弃！会发什么消息告诉源主机吧？不会的，默默地丢，当什么否没有发生，这种情况最难以排查，如果traceroute可以看到端对端使通的，而发送数据就是会失败。所以切记，一台交换机要保证接口MTU的一致性。如果在一个VLAN上、或整个交换机都采用同样的MTU，就不会发生上述情况。而如果入接口是9000字节，而出接口是1500，就会发生上述问题。

如果一条物理链路的两端MTU不一致，则会发生什么情况，比如一侧是1500，一侧是9000，1500一侧发出来的数据肯定没有问题，但是如果从9000侧发给1500呢？数据也背默默地丢了。为什么呢？我们来谈另外一个很少提及的词汇：**MRU**，最大接收单元。

因为普遍是ipv4 ,且一开始网络带宽不高10M 所有MTU就为1518了。

##### 最大接收单元MRU

我们一直谈的最大传输单元MTU是关于出方向的流量处理，而MRU恰恰相反，是关于入方向的流量处理。

一般情况下MTU = MRU，比如9000侧的数据到达1500，由于9000>MRU ，所以直接默默丢弃。

所以在配置链路时，要确保两侧的设备MTU要匹配，无论各家厂商对MTU理解如何、实现如何，一定要保证两端匹配，即各自**允许在以太网线上发送、接收的数据流，即以太网帧的最大长度一样**！

##### Why the MTU size is 1500

作者：知乎用户
链接：https://www.zhihu.com/question/21524257/answer/18501433

There is an obvious reason why the frame payload size was chosen to be 1500 bytes. A frame size of 1500 bytes, offers, maximum efficiency or throughput.
 对于选择帧的负荷量为1500字节，很明显有他的理由的。一个1500字节的帧，具有有最大的发送和接受效率。
As you know, ethernet frame has 8 byte preamble, 6 byte source and 6 byte destination mac address, mac type of 2 bytes, and 4 bytes CRC. Assuming the MTU payload to be 1500 the total number of bytes comes to 1500 + 8 + 6 + 6 + 2 + 4 = 1526 bytes. Now between each frame there is a inter frame gap of 12 bytes which constitues 9.6micro seconds gap between each frame. This is essential so that frames dont mix up. So the total size of each frame going out of a host is 1538 bytes.
 如你所知，以太网帧有8个字节报头，6字节的源和6字节的目的MAC地址，MAC类型2个字节，4字节CRC。假设MTU的有效载荷是1500字节，则总的字节数1500+ 8 +6+6+ 2 + 4=1526字节。现在每帧之间有一个跨12字节的帧间隙(在每个字节中有9.6micro秒?没翻译过来，大致就是两个字节中有间隔）。这对于帧不混合到一起非常重要。因此，主机发出的每一帧的总大小为1538个字节
So at 10 Mbps rate, the frame rate is 10 Mbps / 1538 bytes = 812.74 frames / second.
 因此，以10 Mbps的速率，帧速率是10Mbps/1538字节=812.74帧/秒。
Now we can find the throughput or efficiency  of link, to transmit 1500 bytes of payload. by multiplying the frame rate with the number of bytes of the payload.
 现在我们能够得到链路传输1500字节的吞吐量效率，通过使用帧速率和字节数的乘积。
So efficiency = 812.74 * 1500 * 8  = 9752925.xxxxx bps which is 97.5 percent efficient ( comparing with 10 MBps)
 所以效率=812.74* 1500 *8=9752925.   xxxxx bps效率97.5％（10 MBPS）
I guess I have gone too much with mathematics of Ethernet, but the interesting thing to notice is that, as the number of bytes in the payload increases, the frame rate is decreasing. See that for an MTU of 1500 bytes on payload, the frame rate has reduced to 812 frames per second. If you increase it above 1500, frame rate would  become less than 812.
 我想我说了太多关于因特网的数学的内容，但是我们会发现一个有趣的事情，随着字节负载的增加，帧速率在降低。你已经看到MTU为1500字节的时候，帧速率已经下降到812帧/S。如果你将帧增加超过1500字节，帧速率会小于812帧/S。
Also there is a minimum limit for the MTU which is actually 46 bytes. If you calculate the size of the frame for a 46 byte payload it would come to 12+8+6+6+2+46+4 = 84 bytes. Now calculating the frame rate we get it as =
10mbps/ (84 * 8 bytes) = 14880 frames per second. We could have gone to a frame size even lesser than this, which could increase the frame rate even more, but I guess during those times, when IEEE made the standards, the routers didnt have that much frame forwarding capability.
 当然也有最小的MTU的值，是46字节。如果你计算46字节负载的时候，总字节为12+8+6+6+2+46+4 = 84字节，现在计算一下帧速率，我们得到10mbps/ (84 * 8 bytes) = 14880帧/S，它能够提升帧速率或者增加的更多，但是我想在IEEE定制标准的时候，路由器还没有那么强大的转发帧的能力。
So I think due to above reasons, and considering maximum efficiency, IEEE would have fixed the min and max size of payload as 46 bytes and 1500 bytes.
所以，我想基于以上的云因，考虑最大的效率，IEEE修订了最小和最大的字节传输量为46字节和1500字节。

##### MSS： Maximum Segment Size

MSS的设计初衷是想约束TCP，以避免发送大包给IP，造成不必要的分片。 

既然MSS是一个TCP选项，各个TCP协议栈可以支持，也可以不支持（老的版本）。如果遇到对方不支持MSS选项，那该如何是好呢？ 

那就采用最保守的方法，如果TCP握手发现对方MSS为空，默认对方不支持MSS，采用default MSS = 536 来建立TCP连接。

 Default MSS 536 如何得来？ 

MSS= Maximum IP -IP Header - TCP Header = 576 -20 -20 = 536 那为何选择 **Maximum IP 为576**？ 

常用的各种物理接口MTU (1500常见)> 1000，所以选择576这个值非常安全，可以避免极端情况下的分片。

 576最早来源于UDP应用

最早的操作系统UDP应用程序使用512 Byte作为UDP Datagram，加上8 byte UDP Header，再加上 20 byte IP Header ，就会变成540 byte 的IP包。后来TCP标准化就使用了IP Maximum Size =576。

如果底层物理接口MTU= 1500 byte，则 为了不产生分片的前提下，MSS = 1500- 20(IP Header) -20 (TCP Header) = 1460 byte，如果application 有2000 byte发送，需要两个segment才可以完成发送，第一个TCP segment = 1460，第二个TCP segment = 540。

> 这里其实还要减去帧间隔12 bytes吧--

A (MTU 1500) <——Internet ——> (MTU 1492) B
见上图，TCP SYN消息，A 发送给B 的MSS= 1460，告诉B，B发给A最大segment 为1460 byte
TCP SYN消息，B发送给A的MSS= 1452，告诉A，A发给B最大segment 为1452 byte 。
但A最终能一次发给B多大的字节的segment呢？？？我们给它取名为：A_Send_MSS，取决于两个值，一个是B的通告MSS= 1452；另一个是本地物理接口MTU的限制：1500-20-20= 1460。取这两者较小的一个值，则
A_Send_MSS = minimum ( 1452, 1460) = 1452

> 因为 MTU 通常和 MRU值相同，如果发了大于MRU的frame会被丢弃--

同理可得
B_Send_MSS = minimum ( 1460，1452）=1452

MSS的值是在TCP三次握手建立连接的过程中，经通信双方协商确定的。

链路层使用以太网的话，IP层的MTU是1500 byte，这样去掉IP数据报首部（20 byte），在去掉TCP首部（20 byte）后为1460 byte，此时在默认情况下TCP“选项”字段的MSS值为1460 byte = 1500 - 20 - 20。在 Internet 标准中，IP层的MTU是576 byte，那么此时TCP“选项”字段的MSS值为536 byte = 576 - 20 - 20。

> 所以，MSS的默认值是536

**==MSS值只会出现在SYN报文中==，即SYN=1时，才会有MSS字段值。当客户端想要以TCP方式从服务器端下载数据时，**

（1）首先客户端会发送一个SYN请求报文，这个SYN报文的“选项”字段中会有MSS值（MSS = MUT - IP首部长度 - TCP首部长度）。该MSS值是为了告知对方最大的发送数据大小。

（2）当服务器端收到SYN报文后，会向请求端返回SYN+ACK（同步确认报文）报文，其中的“选项”字段也会有MSS值。

（3）通信双方选择SYN和SYN+ACK报文中最小的MSS最为此次TCP连接的MSS，从而达到通信双发协商MSS的效果。

**综上，可以回答开始时的问题。在第二次握手后就可以确定TCP中最大传输报文（MSS）大小。**

