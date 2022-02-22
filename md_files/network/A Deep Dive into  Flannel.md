## A Deep Dive into  Flannel 

Overlay是再4层传输的（有应用的封包解包，所以性能不好），Calico是bgp协议在三层直接ok的。所以性能好.

Flannel.1 : vxlan vtep，由flannel创建。负责转发到pod所在的node

cni0:  node bridge，不属于pod网络，由flannel cni创建。定位到具体pod/container



### flannel cni and flannel

cni plugins中的flannel是开源网络方案flannel的“调用器”。这也是flannel网络方案适配CNI架构的一个产物。为了便于区分，以下我们称cni plugins中的flannel 为`flanenl cni`。

我们知道flannel是一个容器的网络方案，通常使用flannel时，node上会运行一个daemon进程：flanneld，这个进程会返回该node上的flannel网络、subnet，MTU等信息。并保存到本地文件中。

如果对flannel网络方案有一定的了解，会知道他在做网络接口配置时，其实干的事情和bridge组件差不多。只不过flannel网络下的bridge会跟flannel0网卡互联，而flannel0网卡上的数据会被封包（udp、vxlan下）或直接转发（host-gw）。

> VxLAN封装已经集成到 Linux Kernel了

而`flannel cni`做的事情就是：

- 执行ADD命令时，`flannel cni`会从本地文件中读取到flanneld的配置。然后根据命令的参数和文件的配置，生成一个新的cni配置文件（保存在本地，文件名包含容器id以作区分）。新的cni配置文件中会使用其他cni组件，并注入相关的配置信息。之后，`flannel cni`根据这个新的cni配置文件执行ADD命令。
- 执行DEL命令时，`flannel cni`从本地根据容器id找到之前创建的cni配置文件，根据该配置文件执行DEL命令。

也就是说`flannel cni`此处是一个flannel网络模型的委托者，falnnel网络模型委托它去调用其他cni组件，进行网络配置。通常调用的是bridge和host-local。

### Flannel vxlan 实战： mark

https://msazure.club/flannel-networking-demystify/

#### Q： 

**flanneld进程挂掉会影响pod通信吗？**

详细问题，更新Flannel配置或flannel container OOM，导致Flannel Daemon重启，会影响k8s集群 pod 正常访问吗。

#### K：

##### Veth Pair

For each K8S Pod, flannel will create a pair of veth devices. Refer to [veth](http://man7.org/linux/man-pages/man4/veth.4.html) Taking node1 for example, eth0(containern) is created in flannel network namespace, vethxxxxxxxx is created in host network namespace.

容器里面，查看

```bash
[root@cloudos2 ~]$ oc exec -n cloudos-paas                 os-app-mgmt-695fbb5778-95dgw  -- cat /sys/class/net/eth0/iflink
4519
```

host上 遍历/sys/claas/net下面的全部目录查看子目录ifindex的值和容器里面查出来的iflink值相当的veth名字，也就是：veth63a89a3。这样就找到了container 和 veth pair的关系。

```bash
[root@cloudos2 ~]$ for i in `ls /sys/class/net/*/ifindex`; do grep -l 4519 $i ; done
/sys/class/net/veth5b6bf33c/ifindex
```

或者使用`ip link`

容器里执行，ip link show eth0 命令，然后可以看到 116: eth0@if117，其中116是eth0接口的index, 117是和他pair的veth的index

```bash
~ $ ip link show eth0 116: eth0@if117: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP    link/ether 02:42:0a:00:04:08 brd ff:ff:ff:ff:ff:ff
```

在host执行下面命令可以看到对应117的vethinterface是哪一个，这样就得到了container和veth pair关系

```bash
$ ip link show | grep 117 
117: veth145042b@if116: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue master docker0 state UP
```

end

##### cni0 bridge

Actually, cni0 is created by Flannel CNI (described in the next section) when the pod that uses the pod network first is scheduled to the node. **However, cni0 belongs to the node network but not the pod network logically.**

Flannel 创建的vxlan vtep，做节点间的路由，然后通过node上pod的二层信息（cni0 FDB），访问到目的pod。

Interface vethxxxxxxxx is connected to interface cni0, cni0 is a Linux network bridge device, all veth devices will connect to this bridge, so all containers in same node can communicate with each other. cni0 has ip address 10.244.X.1,

To check cni0 details, run `ip -d link show cni0`

```bash
5: cni0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 0a:58:0a:f4:01:01 brd ff:ff:ff:ff:ff:ff promiscuity 0 
    bridge forward_delay 1500 hello_time 200 max_age 2000 ageing_time 30000 stp_state 0 priority 32768 vlan_filtering 0 vlan_protocol 802.1Q addrgenmode eui64
```

We can verify veth device is connected with cni0 by issuing command `ip -d link show veth43b57597`

```bash
...
6: veth43b57597@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master cni0 state UP mode DEFAULT group default 
    link/ether 92:e6:95:33:81:fc brd ff:ff:ff:ff:ff:ff link-netnsid 0 promiscuity 1 
    veth 
    bridge_slave state forwarding priority 32 cost 2 hairpin on guard off root_block off fastleave off learning on flood on addrgenmode eui64
```

It shows veth43b57597 is a bridge_slave and it's master is cni0.

```bash
$ bridge vlan show
port	vlan ids
docker0	 1 PVID Egress Untagged

cni0	 1 PVID Egress Untagged

veth43b57597	 1 PVID Egress Untagged

veth822966d6	 1 PVID Egress Untagged
```

end

##### vxlan device: flannel.1

flannel默认的VNI值为1，所以flannel创建的网桥叫`flannle.1`

When VXLAN backend is being used by flannel, a VXLAN device whose name is flannel.<vni> will be created, <vni> stands for VXLAN Network Identifier, by default in flannel VNI is set to 1, that means the default device name is flannel.1. `ip -d link show flannel.1` will show details about this VXALN device

```bash
4: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN mode DEFAULT group default 
    link/ether 8e:d0:f8:0a:41:19 brd ff:ff:ff:ff:ff:ff promiscuity 0 
    ## vni is 1.
    vxlan id 1 local 172.16.0.5 dev eth0 srcport 0 0 dstport 8472 nolearning ageing 300 udpcsum addrgenmode eui64 
```

As displayed from above output, vxlan id is 1, eth0 device is used in tunneling, VXLAN UDP port is 8472 and the nolearning tag that disables source-address learning meaning Multicast is not used but Unicast with static L3 entries for the peers.

VXLAN device flannel.1 is linked with physical network device eth0 to send out VXLAN traffics through physical network.

##### flanneld

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/flannel-data-flow.jpg)

 Agent `flanneld` will populate node ARP table as well as the bridge forwarding database, so flannel.1 knows how to forward traffics within physical network. When a new kubernetes node is found (either during startup or when it’s created), `flanneld` adds

- ARP entry for remote node's VXLAN device. (VXLAN device IP->VXLAN device MAC)
- VXLAN fdb entry to remote host. (VXLAN device MAC->Remote Node IP)

Sample ARP entry and FDB entry from node1

```bash
# Permanet ARP entry programmed by flanneld
ip neigh show dev flannel.1
10.244.0.0 lladdr 76:34:2f:c5:51:ec PERMANENT

# Static fdb database programmed by flanneld
bridge fdb show flannel.1
33:33:00:00:00:01 dev eth0 self permanent
...
33:33:ff:db:62:3e dev veth822966d6 self permanent
```

end

##### flannel iface-regex

 flannel 通信问题, 如果多个网卡, 且启动时未指定, flannel 会找一个缺省的网卡, 对于虚拟机来讲没有关系, 但是对于物理机, flannel 会找到 eth0 这个外网网卡, flannel 使用错误的网卡发送数据, 抓包的数据可以看出 flannel 使用了公网的网卡发送内网数据, 会被交换机丢弃, 具体图片就不贴了, IP 属于公司机密.

具体的修改方法是确保 flannel 使用了正确的网卡, 需要在启动时指定参数`--iface`与`--iface-regex`: 我们的虚拟机数量少, 物理机数量多. 除了`eth1`, 还有`bond1`这种网卡名, 因此针对虚拟机, 统一将其`eth0`改名变成`eth1`, 而后指定了`-iface-regex=eth1|bond1`这样的配置, 对于后续增加物理机更友好.

问题到这里似乎就结束了, 但是随着 flannel 经常发生 OOM 重启, 暴露了我们的设置问题

为什么一开始没出现, 但是重启又会发生呢, 问题出在了正则表达式上. K8s 在机器上启动容器时, 会创建虚拟的网卡. 这些网卡的名字类似`veth17f90f70@if3`, 这样网卡名称的也会被正则表达式匹配到, 导致 flannel 无法启动, 临时的解决方案就是把机器上的容器移走, `vethxxx`网卡会自动删除, flannel 也就自动恢复了.

当然根本的解决方案是修改正则配置: `--iface-regex="^(bond1|eth1)$"` 使 flannel 更加精准的匹配网卡名称.



#### A:

在Flannel模式的集群中，只要k8s node 节点没有新增或者删除节点，flanneld进程不会介入操作，`cni0`二层表FDB由`flannel cni`（通常是bridge and local-host）维护。







### Flannel backend: package flow

#### vxlan： L3

##### 核心：666

 flannel.1 随着flannel安装出现 但是cni0则是有pod时才出现。

即，每新增一个host(VTEP)，Flannel Daemon会修改所有节点上的RIB，增加相应的host信息。当host上启动pod时，会调用`bridge`创建响应的Linux bridge通过veth pair连通Container and Host。



flannel 的功能主要是负责机器上路由表的修改, 也就是说, 只要不增删机器, flannel 挂掉也没关系, 因为路由表不需要修改，而且VxLAN的解封装是由Linux来完成的。

In this scheme the scaling of table entries is linear to the number of remote hosts - 1 route, 1 arp entry and 1 FDB entry per host.

Kernel集成了Vxlan的模块，高效。

https://www.kernel.org/doc/Documentation/networking/vxlan.txt

https://www.slideshare.net/enakai/how-vxlan-works-on-linux

本质： 检测到vxlan header vni，支持vxlan协议的机器就会接受并对他解包。

##### 实现

**flannel默认的VNI值为1，所以flannel创建的网桥叫`flannle.1`**

VXLAN（Virtual eXtensible Local Area Network，虚拟扩展局域网），是由IETF定义的NVO3（Network Virtualization over Layer 3）标准技术之一，采用L2 over L4（MAC-in-UDP）的报文封装模式，将二层报文用三层协议进行封装，可实现二层网络在三层范围内进行扩展，同时满足数据中心大二层虚拟迁移和多租户的需求。

In this scheme the scaling of table entries is linear to the number of remote hosts - 1 route, 1 arp entry and 1 FDB entry per host.

0.0 成线性关系，每增加一个`remote host`都会增加`1 route, 1 arp and 1 FDB`

```bash
## ??? 这个路由表和host-gw有什么区别
[node3 ~]$ ip route
default via 172.18.0.1 dev eth1
10.96.0.0/16 dev eth0 scope link
10.244.0.0/24 via 10.244.0.0 dev flannel.1 onlink ## node 1
10.244.1.0/24 via 10.244.1.0 dev flannel.1 onlink ## node 2
10.244.2.0/24 dev cni0 proto kernel scope link src 10.244.2.1
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown
172.18.0.0/16 dev eth1 proto kernel scope link src 172.18.0.18
192.168.0.0/23 dev eth0 proto kernel scope link src 192.168.0.16
0.0 
## 0.0 这和上面有什么区别？？？node可以跨网段
[root@master-1 ~]# ip route
default via 21.48.5.254 dev eth0 
10.222.0.0/24 dev cni0 proto kernel scope link src 10.222.0.1 
10.222.1.0/24 via 21.48.5.36 dev eth0 
10.222.2.0/24 via 21.48.5.37 dev eth0 
21.48.5.0/24 dev eth0 proto kernel scope link src 21.48.5.35 
169.254.169.254 via 21.48.5.3 dev eth0 proto static 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 
```

end

<https://blog.csdn.net/cloudvtech/article/details/79781094>

flannel.1是一个vtep(virtual tunnel end point)设备,flannel.1是一个vtep二层设备，所以需要根据vxlan的协议标准进行二层封装转发.

流量走向： POD(container) --> container eth0 --> flannel.1 --> fdb或etcd --> UDP转发。

0.0 flannel.1（flannel配置文件自己定义） 有自己的ip所以会判断是转发还是自己接受。

```bash
## 查看fdb
$ brctl showmacs bridge_name

## flannel.1 随着flannel安装出现 但是cni0则是有pod时才出现
[node1 ~]$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1
    link/ipip 0.0.0.0 brd 0.0.0.0
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:e1:58:32:c9 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
4: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default
    link/ether de:e6:13:36:5e:2b brd ff:ff:ff:ff:ff:ff
    inet 10.244.0.0/32 scope global flannel.1
       valid_lft forever preferred_lft forever
5: cni0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default qlen 1000
    link/ether 0a:58:0a:f4:00:01 brd ff:ff:ff:ff:ff:ff
    inet 10.244.0.1/24 scope global cni0
       valid_lft forever preferred_lft forever
 ...    
 
[node2 ~]$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1
    link/ipip 0.0.0.0 brd 0.0.0.0
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:28:12:7e:9c brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
4: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default
    link/ether 06:6e:dd:49:92:4e brd ff:ff:ff:ff:ff:ff
    inet 10.244.1.0/32 scope global flannel.1
       valid_lft forever preferred_lft forever
5: cni0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default qlen 1000
    link/ether 0a:58:0a:f4:01:01 brd ff:ff:ff:ff:ff:ff
    inet 10.244.1.1/24 scope global cni0
       valid_lft forever preferred_lft forever
 ...
 
[node3 ~]$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1
    link/ipip 0.0.0.0 brd 0.0.0.0
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:ce:f8:22:bb brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
4: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default
    link/ether f2:d5:ee:7b:d9:34 brd ff:ff:ff:ff:ff:ff
    inet 10.244.2.0/32 scope global flannel.1
       valid_lft forever preferred_lft forever
5: cni0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default qlen 1000
    link/ether 0a:58:0a:f4:02:01 brd ff:ff:ff:ff:ff:ff
    inet 10.244.2.1/24 scope global cni0
       valid_lft forever preferred_lft forever
       
 $ cat /etc/cni/net.d/10-flannel.conflist
{
  "name": "cbr0", ## what??? the name 
  "plugins": [
    {
      "type": "flannel",
      "delegate": {
        "hairpinMode": true,
        "isDefaultGateway": true
      }
    },
    {
      "type": "portmap",
      "capabilities": {
        "portMappings": true
      }
    }
  ]
}
 
# node1: 10.244.0.0/24  node2: 10.244.1.0/24  node3: 10.244.2.0/24

[node3 ~]$ ip route
default via 172.18.0.1 dev eth1
10.96.0.0/16 dev eth0 scope link
10.244.0.0/24 via 10.244.0.0 dev flannel.1 onlink
10.244.1.0/24 via 10.244.1.0 dev flannel.1 onlink
10.244.2.0/24 dev cni0 proto kernel scope link src 10.244.2.1
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown
172.18.0.0/16 dev eth1 proto kernel scope link src 172.18.0.18
192.168.0.0/23 dev eth0 proto kernel scope link src 192.168.0.16

[node2 ~]$ ip route
default via 172.18.0.1 dev eth1
10.96.0.0/16 dev eth0 scope link
10.244.0.0/24 via 10.244.0.0 dev flannel.1 onlink
10.244.1.0/24 dev cni0 proto kernel scope link src 10.244.1.1
10.244.2.0/24 via 10.244.2.0 dev flannel.1 onlink
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown
172.18.0.0/16 dev eth1 proto kernel scope link src 172.18.0.9
192.168.0.0/23 dev eth0 proto kernel scope link src 192.168.0.17

[node1 ~]$ ip route
default via 172.18.0.1 dev eth1
10.244.0.0/24 dev cni0 proto kernel scope link src 10.244.0.1
10.244.1.0/24 via 10.244.1.0 dev flannel.1 onlink
10.244.2.0/24 via 10.244.2.0 dev flannel.1 onlink
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown
172.18.0.0/16 dev eth1 proto kernel scope link src 172.18.0.17
192.168.0.0/23 dev eth0 proto kernel scope link src 192.168.0.18

# vxlan 还有 DirectRouting 选项，启用时，两台宿主机位于同一个网段时，不封装通过路由直接送达。
DirectRouting (Boolean): Enable direct routes (like host-gw) when the hosts are on the same subnet. VXLAN will only be used to encapsulate packets to hosts on different subnets. Defaults to false. 
DriectRouting：能够直接路由的时候采用直接路由的方式，否则就通过vxlan。

# package flow
$ kubectl get pod -o wide
NAME          READY     STATUS    RESTARTS   AGE       IP           NODE      NOMINATED NODE
nginx-64f497f8fd-72bqr   1/1       Running   0          14s       10.244.1.2   node2     <none>
```

end

##### 数据流

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

==因为它不能再node网络上传播，所以叫他Inner Ethernet==

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

#### host-gw： L2

Use host-gw to create IP routes to subnets via remote machine IPs. Requires direct layer2 connectivity between hosts running flannel. 

wow, 只有一个`cni0` 而没有`flannel.1`

`host-gw`，通过直接路由的方式传送虚拟网络报文。

Host-gw原理和vxlan中的DirectRouting相同，但是要求所有宿主机都支持直接路由方式（即在同一个二层网络中），并全部采用采用直接路由的方式。

==同一个LAN下，可以通添加静态路由的方式实现集群外机器直接访问pod_ip==

0.0 前提是同一个LAN下哦，因为next hop 只能指向同网段的IP。 要想实现不同LAN的机器直接访问POD_IP则需要在三层路由层面或在汇聚层交换机上配置了。

以直接路由的方式联通flannel的各个子网。就是所有节点都要互相有点对点的路由记录，才能给pod提供一个flat network环境。这要求所有加入flannel网络的主机需要在同一个LAN里面（不在一个LAN，则下一跳不能直接指向node，就需要在Router上加路由规则了），每个节点上有n-1个路由，而n个节点一共有n(n-1)/2个路由。

```bash
0.0 
[root@master-1 ~]# ip route
default via 21.48.5.254 dev eth0 
10.222.0.0/24 dev cni0 proto kernel scope link src 10.222.0.1 
10.222.1.0/24 via 21.48.5.36 dev eth0 
10.222.2.0/24 via 21.48.5.37 dev eth0 
10.222.3.0/24 via 21.48.5.38 dev eth0 
10.222.4.0/24 via 21.48.5.39 dev eth0 
10.222.5.0/24 via 21.48.5.41 dev eth0 
10.222.6.0/24 via 21.48.5.42 dev eth0 
10.222.7.0/24 via 21.48.5.43 dev eth0 
10.222.8.0/24 via 21.48.5.44 dev eth0 
10.222.9.0/24 via 21.48.5.45 dev eth0 
10.222.10.0/24 via 21.48.5.46 dev eth0 
10.222.11.0/24 via 21.48.5.47 dev eth0 
10.222.13.0/24 via 21.48.5.40 dev eth0 
21.48.5.0/24 dev eth0 proto kernel scope link src 21.48.5.35 
169.254.169.254 via 21.48.5.3 dev eth0 proto static 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 


[root@master-1 ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether fa:16:3e:13:cb:4b brd ff:ff:ff:ff:ff:ff
    inet 21.48.5.35/24 brd 21.48.5.255 scope global dynamic eth0
       valid_lft 60807sec preferred_lft 60807sec
    inet6 fe80::f816:3eff:fe13:cb4b/64 scope link 
       valid_lft forever preferred_lft forever
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP 
    link/ether 02:42:ce:09:0f:ca brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:ceff:fe09:fca/64 scope link 
       valid_lft forever preferred_lft forever
4: cni0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP qlen 1000
    link/ether 0a:58:0a:de:00:01 brd ff:ff:ff:ff:ff:ff
    inet 10.222.0.1/24 scope global cni0
       valid_lft forever preferred_lft forever
    inet6 fe80::cc8b:ffff:fed8:9a3c/64 scope link 
       valid_lft forever preferred_lft forever
       
```

end



这是一种纯三层网络的方案，性能最高
![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/flannel-gw.png)

howt-gw模式的工作原理，就是将每个Flannel子网的下一跳，设置成了该子网对应的宿主机的IP地址，也就是说，宿主机（host）充当了这条容器通信路径的“网关”（Gateway），这正是host-gw的含义
所有的子网和主机的信息，都保存在Etcd中，flanneld只需要watch这些数据的变化 ，实时更新路由表就行了。
核心是IP包在封装成桢的时候，使用路由表的“下一跳”设置上的MAC地址，这样可以经过二层网络到达目的宿主机。

不过，这样node不能跨二层网段哟

```bash
## 路由很直观
$ ip r
...
192.168.1.0/24 via 21.48.0.43 dev eth0
192.168.2.0/24 via 21.48.0.44 dev eth0
192.168.3.0/24 via 21.48.0.45 dev eth0
192.168.4.0/24 via 21.48.0.46 dev eth0
...
POD Subnet一直排下去
```

而在这种模式下，容器通信的过程就免除了额外的封包和解包带来的性能损耗。根据实际的测试，host-gw 的性能损失大约在 10% 左右，而其他所有基于 VXLAN“隧道”机制的网络方案，性能损失都在 20%~30% 左右。当然，通过上面的叙述，你也应该看到，host-gw 模式能够正常工作的核心，就在于 IP 包在封装成帧发送出去的时候，会使用路由表里的“下一跳”来设置目的 MAC 地址。这样，它就会经过二层网络到达目的宿主机。

> 0.0 我想关心的是，非k8s cluster node 能否通过加路由来实现访问集群内部pod.
>
> 吆西，应该是可以的。calico就可以。

所以说，Flannel host-gw 模式必须要求集群宿主机之间是二层连通的。

##### calico and gw

RR 模式可以实现pod ip 和 svc ip 被集群外访问。

而且这个比Flannel-gw好，f-gw模式还得加N个路由，而这个bpg只需一条路由即可。

且f-gw必须要求二层互通，而calico不需要。



#### UDP: 低效

UDP 模式，是 Flannel 项目最早支持的一种方式，也是性能最差的一种模式，如今已被弃用，但是对我们理解集群的网络还是有帮助的 

==0.0== 性能差是用于内核态和用户态切换太多。

该模式的主要通信过程如下

1. 容器A需要发送一个数据包到B(用户态)
2. 首先还是会到docker0网口（**用户态->内核态**）
3. flannel在运行的时候会在宿主机创建一系列的路由规则(xx.xx.xx.xx/16 dev flannel0，前提是docker0的网桥网段范围由flannel控制，dockerd启动的时候添加参数即可，FLANNEL_SUBNET=100.96.1.1/24 dockerd --bip=$FLANNEL_SUBNET ）
4. A需要访问的网段，会走宿主机的路由，他的下一跳是flannel0（内核态）
5. flannel0这里会把数据包交给应用程序 flanneld:8125 （**内核态->用户态**）
6. flanneld 进行 UDP 封包之后重新进入内核态，将udp的包通过宿主机的发出去（**用户态->内核态**）

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/flannel-upd-1.png)

UDP模式内核、用户态示意图

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/flannel-udp-2.png)

可以看出该模式仅在ip包发出的过程中，就需要经过三次用户态与内核态之间的数据拷贝，这个代价是比较高的，所以被弃用



###  flannel vxlan

gayhub： <https://github.com/coreos/flannel/blob/master/backend/vxlan/vxlan.go>

 vxlan 要解决二层 Guest MAC 地址寻址、跨三层 Guest IP 地址寻址的问题，并实现全网高效路由分发和同步。

```go
// Some design notes and history:
// VXLAN encapsulates L2 packets (though flannel is L3 only so don't expect to be able to send L2 packets across hosts)
// The first versions of vxlan for flannel registered the flannel daemon as a handler for both "L2" and "L3" misses
// - When a container sends a packet to a new IP address on the flannel network (but on a different host) this generates
//   an L2 miss (i.e. an ARP lookup) 0.0 arp
// - The flannel daemon knows which flannel host the packet is destined for so it can supply the VTEP MAC to use.
//   This is stored in the ARP table (with a timeout) to avoid constantly looking it up.
// - The packet can then be encapsulated but the host needs to know where to send it. This creates another callout from
//   the kernal vxlan code to the flannel daemon to get the public IP that should be used for that VTEP (this gets called
//   an L3 miss). The L2/L3 miss hooks are registered when the vxlan device is created. At the same time a device route
//   is created to the whole flannel network so that non-local traffic is sent over the vxlan device.
//
// In this scheme the scaling of table entries (per host) is:
//  - 1 route (for the configured network out the vxlan device)
//  - One arp entry for each remote container that this host has recently contacted
//  - One FDB entry for each remote host
//
// The second version of flannel vxlan removed the need for the L3MISS callout. When a new remote host is found (either
// during startup or when it's created), flannel simply adds the required entries so that no further lookup/callout is required.
//
//
// The latest version of the vxlan backend  removes the need for the L2MISS too, which means that the flannel deamon is not
// listening for any netlink messages anymore. This improves reliability (no problems with timeouts if
// flannel crashes or restarts) and simplifies upgrades.
//
// How it works:
// Create the vxlan device but don't register for any L2MISS or L3MISS messages
// Then, as each remote host is discovered (either on startup or when they are added), do the following
// 1) Create routing table entry for the remote subnet. It goes via the vxlan device but also specifies a next hop (of the remote flannel host).
// 2) Create a static ARP entry for the remote flannel host IP address (and the VTEP MAC)
// 3) Create an FDB entry with the VTEP MAC and the public IP of the remote flannel daemon.
//   Forwarding Database  即MAC地址表
// In this scheme the scaling of table entries is linear to the number of remote hosts - 1 route, 1 arp entry and 1 FDB entry per host
//
// In this newest scheme, there is also the option of skipping the use of vxlan for hosts that are on the same subnet,
// this is called "directRouting"

```

end

网络测评： <http://machinezone.github.io/research/networking-solutions-for-kubernetes/>

大致意思：

1. Flannel 的第一个版本，l3miss 学习，通过查找 ARP 表 MAC 完成的。 l2miss 学习，通过获取 VTEP 上的 public ip 完成的。

2. Flannel 的第二个版本，移除了 l3miss 学习的需求，当远端主机上线，只是直接添加对应的 ARP 表项即可，不用查找学习了。

3. Flannel的最新版本，移除了 l2miss 学习的需求，不再监听 netlink 消息。

   它的工作模式：

   1. 创建 vxlan 设备，不再监听任何 l2miss 和 l3miss 事件消息
   2. 为远端的子网创建路由
   3. 为远端主机创建静态 ARP 表项
   4. 创建 FDB 转发表项，包含 VTEP MAC 和远端 Flannel 的 public IP

==就是这篇了==： <https://www.sdnlab.com/21143.html>

#### 第一代flannel vxlan 实现

l2miss 和 l3miss vxlan 实现方案

理论基础

l2miss 和 l3miss vxlan 方案的基础依赖 Kernel 内核的 vxlan DOVE 机制，当数据包无法在 FDB 转发表中找到对应的 MAC 地址转发目的地时 kernel 发出 l2miss 通知事件，当在 ARP 表中找不到对应 IP-MAC 记录时 kernel 发出 l3miss 通知事件。内核在查询 `vxlan_find_mac` FDB 转发时未命中则发送 l2miss netlink 通知，在查询 `neigh_lookup` ARP 表时未命中则发送 l3miss netlink 通知，以便有机会让用户态学习路由，这就是第一代 Flannel vxlan 的实现基础。

> **Distributed Overlay Virtual Ethernet** (**DOVE**) is a [tunneling](https://en.wikipedia.org/wiki/Tunneling_protocol) and [virtualization](https://en.wikipedia.org/wiki/Virtualization) technology for [computer networks](https://en.wikipedia.org/wiki/Computer_network), created and backed by [IBM](https://en.wikipedia.org/wiki/IBM). DOVE allows creation of network virtualization layers for deploying, controlling, and managing multiple independent and isolated network applications over a shared physical network infrastructure.



![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/flannel_vxlan_1.0_impl.png)

图中 10.20.1.4 与 10.20.1.3 通信流程：

1. 当 Guest0 (10.20.1.4)第一次发送一个目的地址 `10.20.1.3` 数据包的时候，进行二层转发，查询本地 Guest ARP 表，无记录则发送 ARP 广播 `who is 10.20.1.3`；(这就是L2miss ？？？)
2. vxlan 开启了的本地 ARP 代答 proxy、l2miss、l3miss 功能，数据包经过 vtep0 逻辑设备时，当 Host ARP 表无记录时，vxlan 触发 l2miss 事件，ARP 表是用于三层 IP 进行二层 MAC 转发的映射表，存储着 IP-MAC-NIC 记录，在二层转发过程中往往需要根据 IP 地址查询对应的 MAC 地址从而通过数据链路转发到目的接口中；
3. l2miss 事件被 Flannel 的 Daemon 进程捕捉到，Daemon 查询 Etcd 存储的路由数据库并返回 `10.20.1.3` 的 MAC 地址 `e6:4b:f9:ce:d7:7b`(Guest1) 并存储 Host ARP 表；
4. vtep0 命中 ARP 记录后回复 ARP Reply；
5. Guest0 收到 ARP Reply 后存 Guest ARP 表，开始发送数据，携带目的 `e6:4b:f9:ce:d7:7b` 地址；
6. 数据包经过 bridge 时查询 FDB（Forwarding Database entry） 转发表，询问 where `e6:4b:f9:ce:d7:7b` send to? 如未命中记录，发生 l3miss 事件，FDB 表为 2 层交换机的转发表，FDB 存储这 MAC-PORT 的映射关系，用于 MAC数据包从哪个接口出；
7. Flannel Daemon 捕捉 l3miss 事件，并向 FDB 表中加入目的 `e6:4b:f9:ce:d7:7b` 的数据包发送给对端 Host `192.168.100.3`；
8. 此时 `e6:4b:f9:ce:d7:7b` 数据包流向 vtep0 接口，vtep0 开始进行 UDP 封装，填充 VNI 号为 1，并与对端 `192.168.100.3`建立隧道，对端收到 vxlan 包进行拆分，根据 VNI 分发 vtep0 ，拆分后传回 Bridge，Bridge 根据 dst mac 地址转发到对应的 veth 接口上，此时就完成了整个数据包的转发；
9. 回程流程类似；

模拟验证：

环境：

1. 2 台 Centos7.x 机器，2 张物理网卡
2. 2 个 Bridge，2 张 vtep 网卡
3. 2 个 Namespace，2 对 veth 接口

步骤：

1. 创建 Namespace 网络隔离空间模拟 Guest 环境
2. 创建 veth 接口、Bridge 虚拟交换机、vtep 接口
3. 验证 l2miss 和 l3miss 通知事件
4. 配置路由
5. 验证连通性，Guest0 ping Guest1

准备环境

网卡配置如下：

```bash
# Host0
[root@i-7dlclo08 ~]# ip -d a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00 promiscuity 0
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 52:54:ca:9d:db:ff brd ff:ff:ff:ff:ff:ff promiscuity 0
    inet 192.168.100.2/24 brd 192.168.100.255 scope global dynamic eth0
       valid_lft 24182sec preferred_lft 24182sec
    inet6 fe80::76ef:824d:95ef:18a3/64 scope link
       valid_lft forever preferred_lft forever
48: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP qlen 1000
    link/ether 5a:5f:4f:3c:4d:a6 brd ff:ff:ff:ff:ff:ff promiscuity 0
    bridge forward_delay 1500 hello_time 200 max_age 2000
    inet 10.20.1.1/24 scope global br0
       valid_lft forever preferred_lft forever
    inet6 fe80::585f:4fff:fe3c:4da6/64 scope link
       valid_lft forever preferred_lft forever
50: veth0@if49: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br0 state UP qlen 1000
    link/ether 82:17:03:c5:a5:bf brd ff:ff:ff:ff:ff:ff link-netnsid 1 promiscuity 1
    veth
    bridge_slave
    inet6 fe80::8017:3ff:fec5:a5bf/64 scope link
       valid_lft forever preferred_lft forever
51: vtep0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UNKNOWN qlen 1000
    link/ether 52:2d:1f:cb:13:55 brd ff:ff:ff:ff:ff:ff promiscuity 1
    vxlan id 1 dev eth0 srcport 0 0 dstport 4789 nolearning proxy l2miss l3miss ageing 300
    bridge_slave
 
# Host1
[root@i-hh5ai710 ~]# ip -d a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00 promiscuity 0
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 52:54:d5:9b:94:4c brd ff:ff:ff:ff:ff:ff promiscuity 0
    inet 192.168.100.3/24 brd 192.168.100.255 scope global dynamic eth0
       valid_lft 30286sec preferred_lft 30286sec
    inet6 fe80::baef:a34c:3194:d36e/64 scope link
       valid_lft forever preferred_lft forever
87: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP qlen 1000
    link/ether d6:ca:34:af:d7:fd brd ff:ff:ff:ff:ff:ff promiscuity 0
    bridge forward_delay 1500 hello_time 200 max_age 2000
    inet 10.20.1.1/24 scope global br0
       valid_lft forever preferred_lft forever
    inet6 fe80::d4ca:34ff:feaf:d7fd/64 scope link
       valid_lft forever preferred_lft forever
89: veth0@if88: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br0 state UP qlen 1000
    link/ether 42:3b:18:50:10:d6 brd ff:ff:ff:ff:ff:ff link-netnsid 0 promiscuity 1
    veth
    bridge_slave
    inet6 fe80::403b:18ff:fe50:10d6/64 scope link
       valid_lft forever preferred_lft forever
90: vtep0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UNKNOWN qlen 1000
    link/ether 52:2f:17:7c:bc:0f brd ff:ff:ff:ff:ff:ff promiscuity 1
    vxlan id 1 dev eth0 srcport 0 0 dstport 4789 nolearning proxy l2miss l3miss ageing 300
    bridge_slave
```

end

观察L2miss和L3miss事件：

```bash
# Host0 10.20.1.4 ping 10.20.1.3
[root@i-7dlclo08 ~]# ip netns exec ns0 ping 10.20.1.3
PING 10.20.1.3 (10.20.1.3) 56(84) bytes of data.
From 10.20.1.4 icmp_seq=1 Destination Host Unreachable
From 10.20.1.4 icmp_seq=2 Destination Host Unreachable
From 10.20.1.4 icmp_seq=3 Destination Host Unreachable
 
# See l2miss & l3miss
[root@i-7dlclo08 ~]# ip monitor all
[nsid current]miss 10.20.1.3 dev vtep0  STALE
[nsid current]miss 10.20.1.3 dev vtep0  STALE
[nsid current]miss 10.20.1.3 dev vtep0  STALE
[nsid 1]10.20.1.3 dev if49  FAILED
```

当缺失 ARP 记录时触发`[nsid 1]10.20.1.3 dev if45 FAILED`，当缺失 FDB 转发记录时触发`[nsid current]miss dev vtep0 lladdr e6:4b:f9:ce:d7:7b STALE`，与预期一样，未配置路由前，网络不通。

配置Guest的路由

```bash
[root@i-7dlclo08 ~]# ip n add 10.20.1.3 lladdr e6:4b:f9:ce:d7:7b dev vtep0
[root@i-7dlclo08 ~]# bridge fdb add e6:4b:f9:ce:d7:7b dst 192.168.100.3 dev vtep0
```

回程路由类似，不赘述

连通性测试：`10.20.1.4 ping 10.20.1.3`

```bash
[root@i-7dlclo08 ~]# ip netns exec ns0 ping 10.20.1.3
PING 10.20.1.3 (10.20.1.3) 56(84) bytes of data.
64 bytes from 10.20.1.3: icmp_seq=1 ttl=64 time=1.04 ms
64 bytes from 10.20.1.3: icmp_seq=2 ttl=64 time=0.438 m
```

配置好对端 Guest 路由后，网络连通成功。通过成功配置需要互通所有 Guest 路由转发信息后，Overlay 的数据包能成功抵达最终目的 Host 的目的 Guest 接口上。这种方式存在明显的缺陷：

end

l2miss 和 l3miss 方案缺陷

1. 每一台 Host 需要配置所有需要互通 Guest 路由，路由记录会膨胀，不适合大型组网

2. 通过 netlink 通知学习路由的效率不高

   Netlink套接字是用以实现**用户进程**与**内核进程**通信的一种特殊的进程间通信(IPC) ,也是网络应用程序与内核通信的最常用的接口。

3. **Flannel Daemon 异常后无法持续维护 ARP 和 FDB 表，从而导致网络不通**

#### 三层路由 vxlan 实现方案

为了弥补 l2miss 和 l3miss 的缺陷，flannel 改用了三层路由的实现方案。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/flannel_vxlan_1.0_impl-2.png)



Flannel 在最新 vxlan 实现上完全去掉了 l2miss & l3miss 方式，**Flannel deamon 不再监听 netlink 通知**，因此也不依赖 DOVE。而改成主动给目的子网添加远端 Host 路由的新方式，同时为 vtep 和 bridge 各自分配三层 IP 地址，当数据包到达目的 Host 后，在内部进行三层转发，这样的好处就是 Host 不需要配置所有的 Guest 二层 MAC 地址，从一个二层寻址转换成三层寻址，路由数目与 Host 机器数呈线性相关，官方声称做到了同一个 VNI 下每一台 Host 主机 1 route，1 arp entry and 1 FDB entry。

模拟验证

环境：

1. 2 台 Centos7.x 机器，2张网卡
2. 2 个 Bridge，2 张 vtep 网卡
3. 2 个 Namespace，2 对 veth 接口

步骤：

1. 创建 Namespace 网络隔离空间模拟 Guest 环境
2. 创建 veth 接口、Bridge 虚拟交换机、vtep 接口
3. 配置路由
4. 验证连通性，Guest0 ping Guest1

**准备环境**

网口配置如下：

```bash
# Host0 网络配置接口
[root@i-7dlclo08 ~]# ip -d a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00 promiscuity 0
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 52:54:ca:9d:db:ff brd ff:ff:ff:ff:ff:ff promiscuity 0
    inet 192.168.100.2/24 brd 192.168.100.255 scope global dynamic eth0
       valid_lft 43081sec preferred_lft 43081sec
    inet6 fe80::76ef:824d:95ef:18a3/64 scope link
       valid_lft forever preferred_lft forever
60: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP qlen 1000
    link/ether 5a:5f:4f:3c:4d:a6 brd ff:ff:ff:ff:ff:ff promiscuity 0
    bridge forward_delay 1500 hello_time 200 max_age 2000
    inet 10.20.1.1/24 scope global br0
       valid_lft forever preferred_lft forever
    inet6 fe80::585f:4fff:fe3c:4da6/64 scope link
       valid_lft forever preferred_lft forever
62: veth0@if61: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br0 state UP qlen 1000
    link/ether 92:6d:63:de:28:d2 brd ff:ff:ff:ff:ff:ff link-netnsid 1 promiscuity 1
    veth
    bridge_slave
    inet6 fe80::906d:63ff:fede:28d2/64 scope link
       valid_lft forever preferred_lft forever
63: vtep0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN qlen 1000
    link/ether 52:2d:1f:cb:13:55 brd ff:ff:ff:ff:ff:ff promiscuity 0
    vxlan id 1 dev eth0 srcport 0 0 dstport 4789 nolearning proxy l2miss l3miss ageing 300
    inet 10.20.1.0/32 scope global vtep0
       valid_lft forever preferred_lft forever
 
# Host1 网络配置接口
[root@i-hh5ai710 ~]# ip -d a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00 promiscuity 0
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 52:54:d5:9b:94:4c brd ff:ff:ff:ff:ff:ff promiscuity 0
    inet 192.168.100.3/24 brd 192.168.100.255 scope global dynamic eth0
       valid_lft 29412sec preferred_lft 29412sec
    inet6 fe80::baef:a34c:3194:d36e/64 scope link
       valid_lft forever preferred_lft forever
95: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP qlen 1000
    link/ether d6:ca:34:af:d7:fd brd ff:ff:ff:ff:ff:ff promiscuity 0
    bridge forward_delay 1500 hello_time 200 max_age 2000
    inet 10.20.2.1/24 scope global br0
       valid_lft forever preferred_lft forever
    inet6 fe80::d4ca:34ff:feaf:d7fd/64 scope link
       valid_lft forever preferred_lft forever
97: veth0@if96: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br0 state UP qlen 1000
    link/ether a6:cc:5b:a4:54:d3 brd ff:ff:ff:ff:ff:ff link-netnsid 0 promiscuity 1
    veth
    bridge_slave
    inet6 fe80::a4cc:5bff:fea4:54d3/64 scope link
       valid_lft forever preferred_lft forever
98: vtep0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN qlen 1000
    link/ether 52:2f:17:7c:bc:0f brd ff:ff:ff:ff:ff:ff promiscuity 0
    vxlan id 1 dev eth0 srcport 0 0 dstport 4789 nolearning proxy l2miss l3miss ageing 300
    inet 10.20.2.0/32 scope global vtep0
       valid_lft forever preferred_lft forever
```

路由HOST配置

```bash
# One Route
10.20.2.0/24 via 10.20.2.0 dev vtep0 onlink
# One ARP
10.20.2.0 dev vtep0 lladdr 52:2f:17:7c:bc:0f PERMANENT
# One FDB
52:2f:17:7c:bc:0f dev vtep0 dst 192.168.100.3 self permanent
```

回程路由类似。

**测试连通性**

`10.20.1.4 ping 10.20.2.4`

```bash
[root@i-7dlclo08 ~]# ip netns exec ns0 ping 10.20.2.4
PING 10.20.2.4 (10.20.2.4) 56(84) bytes of data.
64 bytes from 10.20.2.4: icmp_seq=1 ttl=62 time=0.992 ms
64 bytes from 10.20.2.4: icmp_seq=2 ttl=62 time=0.518 ms
```

end

可以看到，通过增加一条三层路由 `10.20.2.0/24 via 10.20.2.0 dev vtep0 onlink` 使目标为 10.20.2.0/24 的目的 IP 包通过 vtep0 接口送往目的 Host1，目的 Host1 收到后，在本地 Host 做三层转发，最终送往 veth0 接口。在 Host 多个 Guest 场景下也无需额外配置 Guest 路由，从而减少路由数量，方法变得高效。

**总结**

以上就是对 Flannel 第一个版本和最新版本 vxlan overlay 实现基本原理的解析和验证，可以看到 SDN 的 Overlay 配置很灵活也很巧妙，Overlay 的数据包通过 vxlan 这种隧道技术穿透 Underlay 网络，路由配置很灵活，同时主机中迭代着两层网络配置带来了一定的复杂性，但最终无论方案如何变化都离不开二三层路由转发的基本原则。

### Flannel 网络隔离

Flannel这个项目小而精，缺少了一些功能。最直接的缺陷就是：`不支持ACL`，即Kubernetes中的Network Policy。

Flannel is a simple and easy way to configure a layer 3 network fabric designed for Kubernetes.

Flannel is focused on networking. For network policy, other projects such as [Calico](http://www.projectcalico.org/) can be used.

> Calico 推出了一个Cannal 项目可以结合Flannel的网络功能配合自己的Network policy功能。



### Flannel vxlan 缺点

#### k8s pod通信模型

当 pod 访问 K8S 外部 IP 时，pod 的报文在计算节点以 SNAT 的形式访问外部 IP，所以外部 IP 感知到的是计算节点的 IP，而非 pod 的 IP。如下例子，当报文经 SNAT 后，src IP 由 192.168.0.2 修改为 10.0.0.2；src port 由 43345 修改为 47886。

```
+--------------------------+
|  Node 10.0.0.2           |
| +-----------------+      |                 +--------------+
| | Pod 192.168.0.2 | SNAT |  ------------>  | Dst 10.0.0.3 |
| +-----------------+      |                 +--------------+
+--------------------------+
报文中的 IP／Port 信息
src: 192.168.0.2:43345                       src: 10.0.0.2:47886
dst: 10.0.0.3:80                             dst: 10.0.0.3:80
```

外部 IP 无法直接访问 K8S 中的 pod，为了解决这个问题，K8S 引入了 service 和 ingress，其中 service 为 4 层的概念，支持 UDP 和 TCP；ingress 为 7 层概念，支持 http。

虽然 service 支持 cluster IP, node port, loadbalancer 等模式，但是 cluster IP 仅限于 pod 之间的访问；node port 基于 DNAT 为外部服务访问容器提供了入口； loadbalancer 依赖云厂商；而 ingress 仅支持 7 层协议，不满足 rpc 调用等场景，所以相比而言 node port 模式更为简便可行。

K8S 分配计算节点上的某个端口，并映射到容器的某个端口，当外部服务访问容器时，只需要访问相对应的计算节点端口，该端口收到报文后，将目的 ip 和 端口替换为容器的 ip 和端口，最终将报文转发到容器。

```
                                 +--------------------------+
                                 |  Node 10.0.0.2           |
+--------------+                 |      +-----------------+ |                 
| Src 10.0.0.3 |  ------------>  | DNAT | Pod 192.168.0.2 | |
+--------------+                 |      +-----------------+ |
                                 +--------------------------+
报文 IP／Port 信息
src: 10.0.0.3:43345               src: 10.0.0.3:43345
dst: 10.0.0.2:47886               dst: 192.168.0.2:80
```

end

#### 服务发现问题

业务方接入 K8S 是一个循序渐进灰度的过程，而非一刀切。在现有的电商技术栈下，业务方之间存在大量的上下游依赖，而且依赖关系相对复杂。我们梳理在 flannel 网络模型下，业务方接入过程中极有可能遇上的问题。

服务发现是重要的中间件之一，常见的开源项目有 dubbo 和 zookeeper 等。公司自研了类似 dubbo 的组件，并且接入了大量的业务。

首先简要介绍服务发现的基本过程，provider 作为服务提供方，它启动后向注册中心注册信息，如 IP 和 port 等等；consumer 作为消费者，它首先访问注册中心获取 provider 的基本信息，然后基于该信息访问 provider。

```
                         +-----------+ 
                      _  |  注册中心  |  _
     2. get provider  /| +-----------+ |\ 1. register
        basic info   /                   \
    +----------+    /                     \    +----------+                              
    | consumer | --+  3. access provider   +-- | provider |
    +----------+      ------------------>      +----------+
```

让我们设想一种场景，当 provider 先接入 K8S 后，每当 provider 向 register 注册时，它获取本地的 IP 并把该 IP 注册到中心。consumer 访问注册中心，拿到 provider 的 IP 后，却发现这个 IP 并不可达，如此严重影响了业务方接入 K8S。

有人会想到，是否可以先接入 consumer，然后再接入注册中心和 provider 呢？事实上，电商体系的链路比较长，依赖复杂，某个应用既有可能是上游业务的 consumer，同时也是下游业务的 provider，导致业务方无法平滑接入。虽然 K8S 也实现了服务发现功能，但是和我们自研的服务发现功能差距较大，涉及到大量的改造，成本和风险过高。

#### 额外的端口管理问题

某些不需要服务发现的业务方接入到 K8S 后，需要通过 node port 暴露服务入口。那么问题来了，为了避免端口冲突，暴露的端口往往是随机的，这就要求业务方需要额外的增加端口管理，动态维护 pod 对应宿主机端口信息。此外，这也给业务方的下游增加了复杂性，因为下游可能要经常变更访问的端口号。

#### 用户体验问题

业务方常常需要 SSH 登陆到容器中定位问题。当容器运行在 flannel 网络模型下时，业务方访问容器的方式主要有三种：1. 通过 node port 访问，这种情况下需要业务方维护随机端口，增加了管理和使用的复杂性；2. 通过跳板机访问，这种方式需要引入了跳板机器；3. 通过 kube exec，这种情况下需要完善权限控制的机制，且 K8S 地升级会中断 SSH 连接。无论采用哪种方式，都带来额外的维护成本。

我们常常用 ping 包来检查目的主机是否可达，有些甚至用 ping 来作为主机是否宕机的标志。处于 flannel 下的 pod 无法被 ping 通，这也容易业务方的误解，增加沟通成本。

#### flannel 性能问题

最后，flannel 还潜在网络性能问题，主要因为：

- 运行在用户态，增加了数据拷贝流程。每次处理报文时，需要将报文从内核态拷贝到用户态处理，处理完后再拷贝到内核态。
- 基于 UDP 封装的 overlay 报文，带来额外的封包和解包操作。
- 引入 service，增长网络链路。service 默认是通过 iptables 实现的，当条目过多时，iptables 存在性能问题。

从官网的[测试结果](https://github.com/coreos/flannel/issues/738)来看，采用 flannel 容器的网络带宽降低了 50%。从博文 [networking-solutions-for-kubernetes](http://machinezone.github.io/research/networking-solutions-for-kubernetes/) 测试结果来看，flannel 有一定概率出现较大的 latency，这将给公司某些对时延敏感的业务，如交易支付链路带来致命的影响。

### 引用

1. https://mp.weixin.qq.com/s/4NEFyQ49_X4XtqVQ5JG0GA
2. 