## A Deep Dive into Calico

why do you use Calico？

完全利用路由规则实现动态组网，通过 BGP 协议通告路由。

一个目的： 使POD像VM一样有一个可在集群外独立访问的IP.

实现手段：Calico RR（Router Reflector）将 pod subnet 和 集群外网络打通。

### 学习点：

1. bgp and ipip-patch-bgp
2. RR and mesh-mesh
3. network policy



### calico 架构

calico 的好处是 endpoints 组成的网络是单纯的三层网络，报文的流向完全通过路由规则控制，没有 overlay 等额外开销。

calico 的 endpoint 可以漂移，并且实现了 acl。

calico 的缺点是路由的数目与容器数目相同，非常容易超过路由器、三层交换、甚至 node 的处理能力，从而限制了整个网络的扩张。

calico 的每个 node 上会设置大量（海量) 的 iptables 规则、路由，运维、排障难度大。

calico 的原理决定了它不可能支持 VPC，容器只能从 calico 设置的网段中获取 ip。

> 无法支持vpc自定义容器网络？？？

calico 目前的实现没有流量控制的功能，会出现少数容器抢占 node 多数带宽的情况。

calico 的网络规模受到 BGP 网络规模的限制



![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/calico-arch.png)

- Felix：是calico agent，运行在每台需要运行workload的节点上，主要负责配置路由及ACL2等信息来确保Endpoint的连通状态。
- etcd：分布式键值存储，主要负责网络元数据一致性，确保calico网络状态的准确性。
- BGP Client（BIRD）：主要负责把 Felix 写入 Linux 内核的路由信息分发到当前calico网络，确保workload间通信的有效性。
- BGP Route Reflector（BIRD）：大规模部署时使用，摒弃所有节点互联的网格模式，通过一个或者多个BGP Route Reflector来完成集中式的路由分发。

#### 核心

Felix 整理汇总host上的pod各种信息，并同步到etcd

Bird 实现 Felix agent所在节点信息互通，实现路由。



#### Felix

Felix是一个守护程序，它在每个提供endpoints资源的计算机上运行。在大多数情况下，这意味着它需要在托管容器或VM的宿主机节点上运行。 Felix 负责编制路由和ACL规则以及在该主机上所需的任何其他内容，以便为该主机上的endpoints资源正常运行提供所需的网络连接

根据特定的编排环境，Felix负责以下任务：

- 管理网络接口，Felix将有关接口的一些信息编程到内核中，以使内核能够正确处理该endpoint发出的流量。 特别是，它将确保主机正确响应来自每个工作负载的ARP请求，并将为其管理的接口启用IP转发支持。它还监视网络接口的出现和消失，以便确保针对这些接口的编程得到了正确的应用。
- 编写路由，Felix负责将到其主机上endpoints的路由编写到Linux内核FIB（转发信息库）中。 这可以确保那些发往目标主机的endpoints的数据包被正确地转发。
- 编写ACLs，Felix还负责将ACLs编程到Linux内核中。 这些ACLs用于确保只能在endpoints之间发送有效的网络流量，并确保endpoints无法绕过Calico的安全措施。
- 报告状态，Felix负责提供有关网络健康状况的数据。 特别是，它将报告配置其主机时发生的错误和问题。 该数据会被写入etcd，以使其对网络中的其他组件和操作才可见。

#### BIRD

BIRD是布拉格查理大学数学与物理学院的一个学校项目，项目名是BIRD Internet Routing Daemon的缩写

BGP客户端负责执行以下任务：

- 路由信息分发，当Felix将路由插入Linux内核FIB时，BGP客户端将接收它们并将它们分发到集群中的其他工作节点

BGP Route Reflector负责以下任务：

- 集中式的路由信息分发，当Calico BGP客户端将路由从其FIB通告到Route Reflector时，Route Reflector会将这些路由通告给部署集群中的其他节点。

### Link Local Address

Link Local地址也被称为：链路本地地址（link local address），是设备在本地网络中通讯时用的地址，网段为169.254.0.1～169.254.254.255

* LLA(Link Local Address), 链路本地地址, 是设备在本地网络中通讯时用的地址. 网段为`169.254.0.0/16`

- LLA是本地链路的地址, 是在本地网络通讯的, **不通过路由器转发**, 因此网关为0.0.0.0.
- LLA在分配时的具体流程: PROBING -> ANNOUNCING -> BOUND
- `169.254.0.0/16`属于B类网络



### calico网络通讯模型

在主机网络拓扑的组织上，calcio是在主机上启动虚拟机路由器，将每个主机作为路由器使用，组成互联互通的网络拓扑。

每个主机上都部署了 calico node 作为虚拟路由器，并且可以通过 calico 将宿主机组织成任意的拓扑集群。当集群中的容器需要与外界通信时，就可以通过BGP协议将网关物理路由器加入到集群中，使外界可以直接访问容器IP，而不需要做任何NAT之类的复杂操作。 

注意事项：

1、部署 calcio workload 的各节点机的 hostname 应该全部不同，否则会导致写入 etcd 的数据发生覆盖，但是可以通过在启动 calcio node 时，指明 calico node 的名称来避免。

2、calcio workload 的各节点机的 eth0 网卡的 IP 段和 calcio 的网段必须不同，可通过修改 calcio 网段解决。

### CALICO_NETWORKING_BACKEND

The networking backend to use. In `bird` mode, Calico will provide BGP networking using the BIRD BGP daemon; VXLAN networking can also be used. In `vxlan` mode, only VXLAN networking is provided; BIRD and BGP are disabled. If set to `none` (also known as policy-only mode), both BIRD and VXLAN are disabled. [Default: `bird`]

### calico Plugins

CNI Plugins分为: ipam、main和meta三部分。

Calico网络方案，自己实现`ipam plugin: calico-ipam`和`main plugin: calico`，调用了cni原生的`meta plugin: portmap`

```yaml
   {
      "name": "k8s-pod-network",
      "cniVersion": "0.3.0",
      "plugins": [
        {
          "type": "calico",
          "etcd_endpoints": "__ETCD_ENDPOINTS__",
          "etcd_key_file": "__ETCD_KEY_FILE__",
          "etcd_cert_file": "__ETCD_CERT_FILE__",
          "etcd_ca_cert_file": "__ETCD_CA_CERT_FILE__",
          "log_level": "info",
          "mtu": 1500,
          "ipam": {
              "type": "calico-ipam"
          },
          "policy": {
              "type": "k8s"
          },
          "kubernetes": {
              "kubeconfig": "__KUBECONFIG_FILEPATH__"
          }
        },
        {
          "type": "portmap",
          "snat": true,
          "capabilities": {"portMappings": true}
        }
      ]
    }
```

### calico cni 工作原理

Calico 通过一个巧妙的方法将 workload 的所有流量引导到一个特殊的网关 169.254.1.1，从而引流到主机的 calixxx 网络设备上，最终将二三层流量全部转换成三层流量来转发。

在主机上通过开启代理 ARP 功能来实现 ARP 应答，使得 ARP 广播被抑制在主机上，抑制了广播风暴，也不会有 ARP 表膨胀的问题。

**first**， net_ns + veth pair 实现container and host的连通和隔离

**second**，local link address + gateway + arp_proxy 实现容器内数据的引流，确保它能从eth0发送到它在host上是caliXXX对端（veth pair）

#### gateway

```bash
## 先看pod 奇怪的路由
## 所有的pod的默认网关是169.254.1.1
$ ip r
default via 169.254.1.1 dev eth0
169.254.1.1 dev eth0 scope link
## 再看arp信息
## 更奇怪全部都是ee:ee:ee:ee:ee:ee
## 2.13是本机ip地址,是 pod IP?
$ ip nei
192.168.2.13 dev eth0 lladdr ee:ee:ee:ee:ee:ee STALE
169.254.1.1 dev eth0 lladdr ee:ee:ee:ee:ee:ee STALE
$ ip nei del 192.168.2.13 dev eth0

```

#### local link address

```bash
## 尝试添加169.254.1.2
$ ip r add  default via 169.254.1.2
RTNETLINK answers: Network is unreachable
$ ip r del 169.254.1.1
## 169.254.1.1 怎么添加default route的很明显应该是是 network is nureachable
$ ip r add  default via 169.254.1.1
$ ip r
default via 169.254.1.1 dev eth0
169.254.1.1 dev eth0 scope link

## 我猜测是169.254.1.1 dev eth0 scope link 这条的作用
## 删出路由信息和scope link
$ ip r del 169.254.1.1
$ ip r delete 169.254.1.1 dev eth0 scope link
## 再次添加169.254.1.1作为默认路由
##  关键就在这个169.254.1.1 dev eth0 scope link
$ ip r add  default via 169.254.1.1
RTNETLINK answers: Network is unreachable
## scope link : 允许本机连接
$ ip r add 169.254.1.1 dev eth0 scope link
$ ip r
169.254.1.1 dev eth0 scope link
## 再次添加成功
$ ip r add  default via 169.254.1.1
## 1.2 仍是unreachable
$ ip r add  default via 169.254.1.2
RTNETLINK answers: Network is unreachable
## error is chaneged.
$ ip r add 169.254.1.2 dev eth0 scope link
$ ip r add  default via 169.254.1.2
RTNETLINK answers: File exists
```

#### arp_proxy

是否开启arp代理，开启arp代理的话则会以自己的mac地址回复arp请求，0为不开启，1则开启。

**在开启了proxy_arp的情况下，如果请求中的ip地址不是本机网卡接口的地址：但是有该地址的路由，则会以自己的mac地址进行回复；如果没有该地址的路由，不回复**

```bash
## without IPIP mode
$ cat /proc/sys/net/ipv4/conf/cali4158485edce/proxy_arp
1

## IPIP mode
## 而且没有/proc/sys/net/ipv4/conf/cali4158485edce/proxy_arp 接口配置
$ cat /proc/sys/net/ipv4/conf/tunl0/proxy_arp
0
```

ipip就是tunnel的一种实现，Point-to-Point通信。避免中间经过没有k8s cluster bgp信息的路由（bird node或者Router）,所以要开启arp_proxy。

**last**，对bgp的简单利用实现pod的路由。

详细分析：

* 普通网关的作用：将网关MAC替换成数据包中src的mac地址，且网关的IP不会出现在任何网络包头中。

  在传统物理网络中，网关负责将流量引流到接入层交换机，然后使得数据包可以被路由。

* Calico 网络模型下容器网关的IP地址毫无意义，它的作用就是引流。

  在Calico方案中，网关将数据包引流至主机的caliXXX设备中，从而将二三层流量全部转换层三层流量，借助host网络拓扑开始路由。只要pod IP能获得网关的mac地址就可以将数据包发到veth peer了即caliXXX.

  问题来了，calico怎么把这个`169.254.1.1`给配置上去的。

  经过下面的验证，更加证明`169.254.1.1`的作用了就是引流的过程...你不喜欢这个ip了可以改改calico的代码换个别的IP。

  ```bash
  ## 抓包分析
  ## 先清空 pod中的arp表，然后 在host上抓包
  [root@dodo ~]# tcpdump  -i cali60321f46a69 -e -n
  # 10.244.210.140 ping -c 3 10.244.210.139
  # 先清空 pod中的arp表，然后 在host上抓包
  $ tcpdump  -e -n -i cali60321f46a69
  tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
  listening on cali60321f46a69, link-type EN10MB (Ethernet), capture size 262144 bytes
  09:45:24.167575 a6:dd:fe:8b:80:0d > Broadcast, ethertype ARP (0x0806), length 42: Request
  ## default gateway is 169.254.1.1
  who-has 169.254.1.1 tell 10.244.210.140, length 28
  ## 10.244.210.140 获得到网关的mac地址，然后就可将包发出去了。即发给veth peer.
  09:45:24.167597 ee:ee:ee:ee:ee:ee > a6:dd:fe:8b:80:0d, ethertype ARP (0x0806), length 42: Reply 169.254.1.1 is-at ee:ee:ee:ee:ee:ee, length 28
  09:45:24.167602 a6:dd:fe:8b:80:0d > ee:ee:ee:ee:ee:ee, ethertype IPv4 (0x0800), length 98: 10.244.210.140 > 10.244.210.139: ICMP echo request, id 21, seq 1, length 64
  09:45:24.167715 ee:ee:ee:ee:ee:ee > a6:dd:fe:8b:80:0d, ethertype IPv4 (0x0800), length 98: 10.244.210.139 > 10.244.210.140: ICMP echo reply, id 21, seq 1, length 64
  09:45:25.167779 a6:dd:fe:8b:80:0d > ee:ee:ee:ee:ee:ee, ethertype IPv4 (0x0800), length 98: 10.244.210.140 > 10.244.210.139: ICMP echo request, id 21, seq 2, length 64
  09:45:25.167862 ee:ee:ee:ee:ee:ee > a6:dd:fe:8b:80:0d, ethertype IPv4 (0x0800), length 98: 10.244.210.139 > 10.244.210.140: ICMP echo reply, id 21, seq 2, length 64
  09:45:26.167757 a6:dd:fe:8b:80:0d > ee:ee:ee:ee:ee:ee, ethertype IPv4 (0x0800), length 98: 10.244.210.140 > 10.244.210.139: ICMP echo request, id 21, seq 3, length 64
  09:45:26.167803 ee:ee:ee:ee:ee:ee > a6:dd:fe:8b:80:0d, ethertype IPv4 (0x0800), length 98: 10.244.210.139 > 10.244.210.140: ICMP echo reply, id 21, seq 3, length 64
  09:45:29.184741 ee:ee:ee:ee:ee:ee > a6:dd:fe:8b:80:0d, ethertype ARP (0x0806), length 42: Request who-has 10.244.210.140 tell 192.168.2.13, length 28
  09:45:29.184797 a6:dd:fe:8b:80:0d > ee:ee:ee:ee:ee:ee, ethertype ARP (0x0806), length 42: Reply 10.244.210.140 is-at a6:dd:fe:8b:80:0d, length 28
  
  
  
  ```
  

end

#### 信息维护

每一个calico-node启动后都直接接入到etcd中，它们通过etcd感知彼此的变化。

calicoctl也是从etcd中读取系统的状态信息，指令是通过改写etcd中的数据下发。

指令的执行者有两个，一个是calio-node上的felix，它就是一个agent，负责设置iptable等。 另一个执行者是cni插件，kubelet创建Pod的时候会调用cni插件，cni插件负责为Pod准备 workloadendpoint。

> `calicoctl get workloadendpoint --workload=<NAMESPACE>.<PODNAME> -o yaml`
>
> 0.0 nice
>
> 这个排查思路 ，真的牛逼
>
> <https://www.lijiaocn.com/%E6%96%B9%E6%B3%95/2017/08/18/calico-network-problem-resove.html>
>
> end

此外还有一个名为bird的小软件，它和felix位于同一个镜像中，负责BGP通信，向外通告路由、设置本地路由。 





### Calico IPIP Mode：666

IP-ip-IP相当于起的Tunnel，完成两个calico-node（bird）的直连，因为如果L 2不可达，那么下一跳的Router可能没有Calico的bgp信息，就导致无法通信了。

**IP in IP** is an [IP tunneling](https://en.wikipedia.org/wiki/IP_tunnel) protocol that encapsulates one [IP](https://en.wikipedia.org/wiki/Internet_Protocol) packet in another IP packet. 

Calico控制平面的设计要求物理网络得是L2 Fabric，这样vRouter间都是直接可达的为了支持L3 Fabric，Calico推出了IPinIP Mode。

为了支持L3 Fabric，Calico推出了IPinIP的选项。

* ALICO_IPV4POOL_IPIP：是否启用IPIP模式，启用IPIP模式时，Calico将在node上创建一个tunl0的虚拟隧道。
* FELIX_LOGSEVERITYSCREEN： 日志级别。
* FELIX_IPV6SUPPORT ： 是否启用IPV6。
* IPIP是一种将各Node的路由之间做一个tunnel，再把两个网络连接起来的模式，启用IPIP模式时，Calico将在各Node上创建一个名为"tunl0"的虚拟网络接口

If your network fabric performs source/destination address checks and drops traffic when those addresses are not recognized, it may be necessary to enable IP-in-IP encapsulation of the inter-workload traffic.

tunnel0 只是用来建立了tunnel 隧道。

从192.168.70.42（pod1） ping 192.168.169.196（pod2）的时候，在node10.39.0.110（node1）上抓取报文可以看到:

> 可以清晰的看到IP-in-IP封装的外出IP Header是，pod所在node的IP。

```bash
# 注意，抓包要在node上抓，不能再pod里抓，因为pod里已经是经过tunl0 解封后包。
$ tcpdump -n -i eth0 host 10.39.3.75
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
16:41:06.884185 IP 10.39.0.110 > 10.39.3.75: IP 192.168.70.42 > 192.168.169.196: ICMP echo request, id 20142, seq 12, length 64 (ipip-proto-4)
16:41:07.884165 IP 10.39.0.110 > 10.39.3.75: IP 192.168.70.42 > 192.168.169.196: ICMP echo request, id 20142, seq 13, length 64 (ipip-proto-4)
16:41:08.884128 IP 10.39.0.110 > 10.39.3.75: IP 192.168.70.42 > 192.168.169.196: ICMP echo request, id 20142, seq 14, le
```

Host（bird agent）如果可以直连，那么通过bgp peer可以交换路由信息，但是如果两个host中间隔了N个Router，导致Host无法直连交换路由信息，就无法实现pod跨node通讯。emmm，甚至 Bird 和 Router都无法建立bgp peer。所以，出现了IPIP mode，将三层可达的Host 变成可以直达（Next-hop）的host

Calico 将node作为Router实现路由交换(BGP)

新增加的node与原先的node不属于同一个网段，发现新的node中pod无法与原先的node中的pod通信。

node`10.39.0.110`上的pod的地址为192.168.70.42，node`10.39.3.75`(新加node不在一个网段上)上的pod的地址为192.168.169.196。

从node10.39.0.110上无法ping通192.168.169.196，查看10.39.0.110上的路由：

```bash
# ip route
default via 10.39.0.1 dev eth0  proto static  metric 100
192.168.169.192/26 via 10.39.0.1 dev eth0  proto bird
...
```

end

从路由表中发现到192.168.169.196的路由是通过网关`10.39.0.1`送出。

calico没有网关进行路由交换，网关10.39.0.1并不知道192.168.169.192的存在。

#### BGP Patch for IPIP：mark，666

Calico uses BIRD for BGP routing between the hosts in various cloud orchestration systems (Kubernetes, OpenStack etc.), to distribute routes to the pods/VMs/containers in those systems, each of which has its own IP.  If all the hosts are directly connected to each other, this is straightforward, but sometimes they are not.  For example GCE instances are not directly connected to each other: there is at least one router between them, that knows about routing GCE addresses, and to/from the Internet, and we cannot peer with it or otherwise tell it how to route pod/VM/container IPs.  So if we use GCE to create e.g. OpenStack compute hosts, with Calico networking, we need to do something extra to allow VM-addressed data to pass between the compute hosts.

One of our solutions is to use IP-in-IP; it works as shown by this diagram:

    >        10.65.0.3 via 10.240.0.5 dev tunl0 onlink
    >        default via 10.240.0.1
    >                |
    >              +-|----------+                             +------------+
    >              | o          |                             |            |
    >              |   Host A   |         +--------+          |   Host B   |
    >              |            |---------| Router |----------|            |
    >              | 10.240.0.4 |         +--------+          | 10.240.0.5 |
    >              |            |---.                         |            |
    >              +------------+    |                        +------------+
    >                ^       ^   +---v---+                                |
    >  src 10.65.0.2 |       |   | tunl0 |                                |
    >  dst 10.65.0.3 |       |   +-------+                                |
    >                |        \      |                                    v
    >          +-----------+   '----'                               +-----------+
    >          |   Pod A   |      src 10.240.0.4                    |   Pod B   |
    >          | 10.65.0.2 |      dst 10.240.0.5                    | 10.65.0.3 |
    >          +-----------+          ------                        +-----------+
    >                              src 10.65.0.2
    >                              dst 10.65.0.3

Can't you just use a tunnel between Host A and Host B and run BGP on top
of this tunnel?  It would seem to be cleaner than hacking multi-hop BGP to
obtain appriopriate next-hop values, unless I am missing something.

It would look something like this:

             +-|----------+                             +------------+
             | o Host A   |                             |   Host B   |
             |            |         +--------+          |            |
             |  10.240.0.4|---------| Router |----------|10.240.0.5  |
             |            |         +--------+          |            |
             |   10.65.0.4|--.  +-------+   +-------+ .->10.65.0.5   |
             +------------+   `>| tunlA |-->| tunlB |-  +------------+
                                +-------+   +-------+


The BGP session would be established between 10.65.0.4 (IP of host A on
tunlA) and 10.65.0.5 (IP of host B on tunlB), so that the routes learnt
via BGP would be immediately correct.

Basically, it's a simple overlay network.



### Calico实践：6

#### 开启NAT

默认情况下，calico访问集群外网络是通过SNAT成宿主机ip方式。

如果关闭NAT会导致复杂网络环境下（比如国网私有OpenStack and 其他物理机）pod 无法与集群外的node交流，因为网络环境负责有防火墙策略阻挡，只允许k8s cluster node 网段访问 集群外的node。

#### 关闭NAT

在一些金融客户环境中为了能实现防火墙规则，需要直接针对POD ip进行进行规则配置，所以需要关闭natOutgoing

calico 关闭`SNAT`，即将 `natOutgoing: true`修改为`natOutgoing: false`此时，calico网络访问集群外的ip，源ip POD_IP就不会snat成宿主机的ip地址

```bash
$ calicoctl get ipPool -o yaml|sed 's/natOutgoing: true/natOutgoing: false/g'|calicoctl apply -f -
```

end

#### 固定POD IP

固定单个ip

```
apiVersion: apps/v1 
kind: Deployment
metadata:
  name: nginx-test
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1 # tells deployment to run 1 pods matching the template
  template:
    metadata:
      labels:
        app: nginx
      annotations:
        "cni.projectcalico.org/ipAddrs": "[\"10.42.210.135\"]"
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

固定多个ip，只能通过ippool的方式

```
cat ippool1.yaml
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: pool-1
spec:
  blockSize: 31
  cidr: 10.21.0.0/31
  ipipMode: Never
  natOutgoing: true
apiVersion: apps/v1 
kind: Deployment
metadata:
  name: nginx-test
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1 # tells deployment to run 1 pods matching the template
  template:
    metadata:
      labels:
        app: nginx
      annotations:
        "cni.projectcalico.org/ipv4pools": "[\"pool-1\"]"
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

end

#### **mutli-interfaces**: mark

默认情况下，calico 在启动时会自动选择一块网卡。当主机上有多块网卡时，为了保证路由的正确性，需要手动指定 calico 使用哪块物理网卡！

```
- name: IP_AUTODETECTION_METHOD
              value: "interface=eth.*
```

end

`calico-kube-controller` 报错：`calico Readiness probe failed: Error reaching apiserver: <nil>`

0.0 

For an HTTP probe, the kubelet sends an HTTP request to the specified path and port to perform the check. The kubelet sends the probe to the pod’s IP address, unless the address is overridden by the optional `host` field in `httpGet`. If `scheme` field is set to `HTTPS`, the kubelet sends an HTTPS request skipping the certificate verification. In most scenarios, you do not want to set the `host` field. Here’s one scenario where you would set it. Suppose the Container listens on 127.0.0.1 and the Pod’s `hostNetwork`field is true. Then `host`, under `httpGet`, should be set to 127.0.0.1. If your pod relies on virtual hosts, which is probably the more common case, you should not use `host`, but rather set the `Host` header in `httpHeaders`

changing-ip-pools

0.0 烦人，记录到etcd不会随着`kubectl apply -f calico.yaml`修改，除非通过`calicoctl`或直接修改etcd数据。

<https://docs.projectcalico.org/v3.6/networking/changing-ip-pools>



#### reinstall

`modprobe -r ipip`   should delete tunl0 iface

#### Peer scopes

BGP Peers can exist at either global or node-specific scope. A peer’s scope determines which `calico/node`s will attempt to establish a BGP session with that peer.

#### Global peer

To assign a BGP peer a global scope, omit the `node` and `nodeSelector` fields. All nodes in the cluster will attempt to establish BGP connections with it

#### Node-specific peer

A BGP peer can also be node-specific. When the `node` field is included, only the specified node will peer with it. When the `nodeSelector` field is included, the nodes with labels that match that selector will peer with it.

```yaml
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: some.name
spec:
  node: rack1-host1 ## 决定了scope is node specific
  peerIP: 192.168.1.1
  asNumber: 63400
  
  
  
  #IPv4 BGP status
#+---------------+---------------+-------+----------+-------------+
#| PEER ADDRESS  |   PEER TYPE   | STATE |  SINCE   |    INFO     |
#+---------------+---------------+-------+----------+-------------+
#| 10.199.152.53 | node specific | up    | 08:42:48 | Established |
#|
```

end



### calico组网验证

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

```
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





### NetworkPolicy（k8s和calico两套）:eyes:

先抛出问题来，对某个pod设置NetworkPolicy后，实现了ACL但是也导致了node（除了该pod所在的node其余的node）均无法访问该POD

calico networkpolicy的几个知识点：

* profile： https://docs.projectcalico.org/v3.9/reference/resources/profile

  0.0 不太好理解

  ```bash
  apiVersion: projectcalico.org/v3
  kind: Profile
  metadata:
    name: dev-apps
  spec:
    ingress:
    - action: Deny
      source:
        nets:
        - 10.0.20.0/24
    - action: Allow
      source:
        selector: stage == 'development'
    egress:
    - action: Allow
    # An optional set of labels to apply to each endpoint in this profile (in addition to the endpoint’s own labels)
    labelsToApply:
      stage: development
  ```

  The following sample profile allows all traffic from endpoints that have the label `stage: development` (i.e. endpoints that reference this profile), except that *all* traffic from 10.0.20.0/24 is denied.

  阿西吧，这个太重要了

* hostendpoint： A host endpoint resource (`HostEndpoint`) represents one or more real or virtual interfaces attached to a host that is running Calico。

  就是将host 交由calico纳管

  calico中的endpoint，分为：workloadendpoint  和 hostendpoint.

  workloadendpoint对应在k8s中对应的就是分配给pod的接口，hostendpoint对应的是node的接口。

* workendpoint

* networkpolicy： This sample policy allows TCP traffic from `frontend` endpoints to port 6379 on `database` endpoints.

  ```yaml
  apiVersion: projectcalico.org/v3
  kind: NetworkPolicy
  metadata:
    name: allow-tcp-6379
    namespace: production
  spec:
    selector: role == 'database'
    types:
    - Ingress
    - Egress
    ingress:
    - action: Allow
      protocol: TCP
      source:
        selector: role == 'frontend'
      destination:
        ports:
        - 6379
    egress:
    - action: Allow
  ```

  end

* end

#### 经测试GlobalNetworkPolicy优先级大于NetworkPolicy： 很好看

测试思路很简洁，A和B，分别取相反值，同时应用判断优先级。

为了避免有顺序优先的可能(即那个在前哪个优先)，尝试了先A后B和先B后A，还是global优先。

```bash
### k8s native NetworkPolicy
$ cat net-policy-test.yaml
apiVersion: extensions/v1beta1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      run: source-ip-app
  ingress:
  - from:
  # 将node所在的网段加入ingress
    - ipBlock:
        cidr: 21.49.22.0/24
    - podSelector:
        matchLabels:
          run: test
          
 $ kubectl apply -f net-policy-test.yaml
 0.0 测试满足，需求
 
 
 ## calico NetworkPolicy
 思路：
 1. 所有的pod都是workloadEndpoint
 2. 将node的interface 已hostEndpoint方式加入calico object，避免node无法访问pod
 3. 这样所有的interface都是endpoint对象了，肆意编排
 
## 为node 设置hostEndpoint
## 因为利用了all.allow profile所有此时的hostendpoint是全通的
$ cat m1-hep.yaml  && calicoctl apply -f m1-hep.yaml
kind: HostEndpoint
metadata:
  labels:
    calico: vip
  name: cs1-k8s-m1
spec:
# The name of the node where this HostEndpoint resides.
# 这个node后面的名字？？
  node: cs1-k8s-m1.zpepc.com.cn
  expectedIPs:
  - 21.49.22.4
  interfaceName: bond1
  ## 注意没有这个profiles直接暴毙，calico通过iptables是你的node 任何都访问不了
  profiles:
  - all.allow
  
# all.allow当然是我自己创建的了 默认设置全通--
 $ calicactl get profiles all.allow
 apiVersion: projectcalico.org/v3
kind: Profile
metadata:
  name: all.allow
spec:
  egress:
  - action: Allow
    destination: {}
    source: {}
  ingress:
  - action: Allow
    destination: {}
    source: {}

## workloadendpoint 长这个样子
- apiVersion: projectcalico.org/v3  
  kind: WorkloadEndpoint            
  metadata:                         
    creationTimestamp: 2019-11-21T02:02:07Z
    generateName: source-ip-app-68b64dc7bc-
    labels:                                
      pod-template-hash: 68b64dc7bc        
      projectcalico.org/namespace: default 
      projectcalico.org/orchestrator: k8s  
      projectcalico.org/serviceaccount: default
      run: source-ip-app                       
      workloadID_ingress-d188feee46ef994debda00f09923a0f5: "true"  
    name: cs1--k8s--n1.zpepc.com.cn-k8s-source--ip--app--68b64dc7bc--46dzt-eth0
    namespace: default                                                         
    resourceVersion: "28487445"                                                
    uid: ec2edbb3-0c02-11ea-9d7b-e04f43ddacb0                                  
  spec:                                                                          
    containerID: 8d2adbfe5a602864ced8e8e98636b196b85a9ec4b64506cb72bcd74de144fb98
    endpoint: eth0                                                               
    interfaceName: cali6bae7ed2e34                                               
    ipNetworks:                                                                  
    - 192.168.118.87/32                                                          
    mac: 4e:0f:f9:25:13:70                                                       
    node: cs1-k8s-n1.zpepc.com.cn                                                
    orchestrator: k8s             
    pod: source-ip-app-68b64dc7bc-46dzt
    ## 每个namespace和serviceaccount都有一个默认的profile 
    profiles:                          
    - kns.default                      
    - ksa.default.default     
 ，，，
 
## networkpolicy
## 卧槽失败了，因为node 无法访问 
## 麻痹哟，NetworkPolicy is a namespaced resource. NetworkPolicy in a specific namespace only applies to workload endpoint resources in that namespace.
0.0 
$ calicoctl apply -f acl.yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  selctor:   run == 'source-ip-app'
  ingress:
  - action: Allow
    source: 
    ## node m1 是vip啊不行草
    ##因为是 NetwPolicy so，无法匹配hostendpoint
      selector: role == 'vip'

## GlobalNetworkPolicy
## GlobalNetworkPolicy is not a namespaced resource. GlobalNetworkPolicy applies to workload endpoint resources in all namespaces, and to host endpoint resources.
## 经测试其他namespace下的pod如果带有role=vip的label都可以访问目标pod：source-ip-app
$ calicoctl apply -f global-acl.yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: test-global-network-policy
spec:
  selctor:   run == 'source-ip-app'
  ingress:
  - action: Allow
    source: 
      selector: role == 'vip'
      
      
 ## 但是hostendpoint还是不行，卧槽
 ## 0.0 
 先放放吧，NetworkPolicy不会影响kubectl exec 应为它走的路线是0.0 
 kubectl -- kubelet -- OCI --containerd
 0.0 
 ## 哈哈哈解决了，修改下hostendpoint，我忘了加expectIP
 ## 准确来说，我记错单词了 expect和except...
 ## shoot
apiVersion: projectcalico.org/v3
kind: HostEndpoint
metadata:
  # 宿主机端点名称
  name: some.name
  # 标签，用于应用对应的策略
  labels:
    role: vip
spec:
  # 端点对应的网络接口
  interfaceName: eth0
  # 端点所在的宿主机
  node: myhost
  # 期望和此网络接口对应的IP地址
  expectedIPs:
  - 192.168.0.1
  - 192.168.0.2
  # 配置，用于应用对应的策略
  profiles:
  - profile1
  - profile2
  # 命名端口，可以在Policy Rule中引用
  ports:
  - name: some-port
    port: 1234
    protocol: TCP
  - name: another-port
    port: 5432
    protocol: UDP
```

end

### BGP Speaker RR模式：无敌

RR 模式可以实现pod ip 和 svc ip 被集群外访问。

**因为，RR模式指定的节点，包含了整个集群的路由信息。**

**而且这个必Flannel-gw好，f-gw模式还得加N个路由，而这个bpg只需一条路由即可。**

https://blog.csdn.net/nexus124/article/details/105005240

```bash
## 呐，重头戏了。在外部机器上添加路由即可实现
## svc ip也可以真的野啊
## 192.168 是定义的 k8s pod网段
[root@cs1-harbor-1 ~]# ip r add 192.168.0.0/16 via 21.49.22.7

[root@cs1-harbor-1 ~]# curl  192.168.1.161:8080 -I
HTTP/1.1 200 OK
Server: nginx/1.10.0
Date: Sun, 29 Dec 2019 08:59:33 GMT
Content-Type: text/plain
Connection: keep-alive

## 添加node-1 或 node-2 都行
## 10.96是定义的k8s svc网段
[root@cs1-harbor-1 ~]# ip r add 10.96.0.0/12 via 21.49.22.8
[root@cs1-harbor-1 ~]# curl  10.102.102.13:8080 -I
HTTP/1.1 200 OK
Server: nginx/1.10.0
Date: Sun, 29 Dec 2019 09:07:08 GMT
Content-Type: text/plain
Connection: keep-alive

## 当然最方便的肯定是将这一步在开发网络的路由上做，设置完成后开发网络就可以直连集群内的 Pod IP 和 Service IP 了；至于想直接访问 Service Name 只需要调整上游 DNS 解析既可
https://blog.foxsar.black/?p=246
```

end

[https://cshihong.github.io/2018/01/23/BGP%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86/](https://cshihong.github.io/2018/01/23/BGP基础知识/)

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/calico-rr.jpg)

Calico方案只是把"hostnetwork"（这个是加引号的hostnetwork）方案配置自动化了。由于还是通过物理设备进行虚拟设备的管理和通信，所以整个网络依然受物理设备的限制影响；另外因为项目及设备数量的增加，内网IP也面临耗尽问题。

> Host Network”将游戏服务器的虚拟IP也配置成实体IP，并注入到主路由中。但这使得主路由的设备条目数比设计容量多了三倍。
>
> ![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/pod-hostnet.jpg)
>
> 0.0

0.0 也说明了这个问题，当使用bgp将网络打通后podIP也充斥着整个网络拓扑，私有ip可能不足，，，这得多少节点。

牛逼： <https://www.lijiaocn.com/%E9%A1%B9%E7%9B%AE/2017/04/11/calico-usage.html#bgp-speaker-%E5%85%A8%E4%BA%92%E8%81%94%E6%A8%A1%E5%BC%8Fnode-to-node-mesh>

牛逼： <https://www.troyying.xyz/index.php/IT/5.html>

-- 

Calico 在每一个计算节点利用 Linux kernel 实现了一个高效的 vRouter 来负责数据转发 而每个 vRouter 通过 BGP 协议负责把自己上运行的 workload 的路由信息像整个 Calico 网络内传播 － 小规模部署可以直接互联，大规模下可通过指定的 BGP route reflector 来完成。

由于BGP就是为了替换EGP而创建，它的地位与EGP相似。但是BGP也可以应用在一个AS内部。因此BGP又可以分为IBGP（Interior BGP ：同一个AS之间的连接）和EBGP（Exterior BGP：不同AS之间的BGP连接）。

tunnel0 只是用来建立了tunnel 隧道。

==有意思==，在这种模式下，是不是就算node不在一个二层网络离也不需要开`IPIP Mode`就可以保证跨网段的节点的间的BGP了。

> ​       --------- Router ----------
>
> ​				|
>
> ​         NodeA  ----  | --- NodeB
>
> 0.0 It was guessed that  can connect A with B even if it isn't a subnet and IPIP mode.

RR模式，就是在网络中指定一个或多个BGP Speaker作为Router Reflection，RR与所有的BGP Speaker建立BGP连接。

每个**BGP Speaker**只需要与RR交换路由信息，就可以得到全网路由信息。

RR则必须与所有的BGP Speaker建立BGP连接，以保证能够得到全网路由信息。

在calico中可以通过Global Peer实现RR模式。

Global Peer是一个BGP Speaker，需要手动在calico中创建，所有的node都会与Global peer建立BGP连接。

```
A global BGP peer is a BGP agent that peers with every calico node in the network. 
A typical use case for a global peer might be a mid-scale deployment where all of
the calico nodes are on the same L2 network and are each peering with the same Route
Reflector (or set of Route Reflectors).
```

关闭了**全互联模式（node-to-node mesh）**后，再将RR作为Global Peers添加到calico中，calico网络就切换到了RR模式，可以支撑容纳更多的node。

calico中也可以通过node Peer手动构建BGP Speaker（也就是node）之间的BGP连接。

node Peer就是手动创建的BGP Speaker，只有指定的node会与其建立连接。

```
A BGP peer can also be added at the node scope, meaning only a single specified node 
will peer with it. BGP peer resources of this nature must specify a node to inform 
calico which node this peer is targeting.
```

因此，可以为每一个node指定不同的BGP Peer，实现更精细的规划。

例如当集群规模进一步扩大的时候，可以使用[AS Per Pack model](http://docs.projectcalico.org/v2.1/reference/private-cloud/l3-interconnect-fabric#the-as-per-rack-model):

```
每个机架是一个AS
node只与所在机架TOR交换机建立BGP连接
TOR交换机之间作为各自的ebgp全互联
```

end



### BGP Speaker 全互联模式(node-to-node mesh)

全互联模式，就是一个BGP Speaker需要与其它所有的BGP Speaker建立bgp连接(形成一个bgp mesh)。

网络中bgp总连接数是按照O(n^2)增长的，有太多的BGP Speaker时，会消耗大量的连接。

calico默认使用全互联的方式，扩展性比较差，只能支持小规模集群:

```
say 50 nodes - although this limit is not set in stone and 
Calico has been deployed with over 100 nodes in a full mesh topology
```

可以打开/关闭全互联模式：

```
calicoctl config set nodeTonodeMesh off
calicoctl config set nodeTonodeMesh on
```

end



In Project Calico, broadcast production IP by BGP working on **BIRD containers** (OSS routing software) launched on each nodes of Kubernetes. By default, it broadcast in cluster only. By setting peering routers outside of clusters, it makes it possible to access a Pod from outside of the clusters. 



![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/What-is-iBGP-and-eBGP.jpg)

calico how it works

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/calico-how it works.jpg)





Calico 简介

> `tcpdump -e` 看到mac

Calico 是一个基于BGP协议的网络互联解决方案。它是一个纯3层的方法，使用路由来实现报文寻址和传输。 
相比 flannel, ovs等SDN解决方案，Calico 避免了层叠网络带来的性能损耗。将节点当做 router ，位于节点上的 container 被当做 router 的直连设备。利用 Kernel 来实现高效的路由转发。 节点间的路由信息通过 BGP 协议在整个 Calico 网络中传播。 具有以下特点： 

1. 在 calico 中的数据包不需要进行封包和解封。 
2. 基于三层网络通信，troubleshoot 会更方便。 
3. 网络安全策略使用 ACL 定义，基于 iptables 实现，比起 overlay 方案中的复杂机制更只管和容易操作。

Environment

server	ip	mac	gw mac
walker-1	172.16.6.47	fa:16:3e:02:8b:17	00:23:89:8C:E8:31
walker-2	172.16.6.249	fa:16:3e:8c:21:13	00:23:89:8C:E8:31
demi-1	172.16.199.114	fa:16:3e:d9:a0:5e	00:23:89:8C:E8:31
busybox-1	192.168.187.211	3a:1d:1e:91:f5:9e	66:39:fa:e7:9f:a9
busybox-2	192.168.135.74	de:16:fc:1c:44:35	5a:4a:df:5e:c9:6c
busybox-3	192.168.121.2	de:16:fc:1c:44:35	8e:6b:fa:f7:5d:3b
node2 单独位于vlan 199 中,和master以及node1间的通信需要经过网关（Router）转发。

使用 IP-in-IP
**ipip enable=false**

```bash
$ calicoctl apply -f - << EOF
apiVersion: v1
kind: ipPool
metadata:
  cidr: 192.168.0.0/16
spec:
  nat-outgoing: true
EOF
```

end

ipip 模式禁用时，busybox-3 和 busybox-{1,2} 之间无法通信。

分析如下：

主机路由：

```bash
root@walker-1 manifests]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.16.6.254    0.0.0.0         UG    0      0        0 eth0
169.254.169.254 172.16.6.87     255.255.255.255 UGH   0      0        0 eth0
172.16.6.0      0.0.0.0         255.255.255.0   U     0      0        0 eth0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
192.168.121.0   172.16.199.114  255.255.255.192 UG    0      0        0 eth0
192.168.135.64  172.16.6.249    255.255.255.192 UG    0      0        0 eth0
192.168.187.192 0.0.0.0         255.255.255.192 U     0      0        0 *
192.168.187.209 0.0.0.0         255.255.255.255 UH    0      0        0 calic6611247c43
192.168.187.211 0.0.0.0         255.255.255.255 UH    0      0        0 calie50081a277c
```

end

从busybox-1 发往 busybox-3 的报文头部如下所示：

```text
src max | dst mac | src ip | dst ip 
---     | ---     | ---    | ---    
3a:1d:1e:91:f5:9e | 66:39:fa:e7:9f:a9  | 192.168.187.211 | 192.168.121.2

```

end

根据宿主机路由，报文会从eth0 发往 172.16.199.114。

由于二者位于不通广播域，需要通过网关转发。因此报文的 dst mac 会被修改为 172.16.6.254(gw) 对应的mac。

```text
src max | dst mac | src ip | dst ip | enc src IP | enc dst IP 
---     | ---     | ---    | ---    | --- | ---
fa:16:3e:02:8b:17 | 00:23:89:8C:E8:31  | 192.168.187.211 | 192.168.121.2

```

end

gw 收到该报文后，会比对本地路由条目。由于 router 中并没有到 192.168.121.0\26 网段的路由，因此报文被丢弃。

**ipip enable=true**

```bash
$ calicoctl apply -f - << EOF
apiVersion: v1
kind: ipPool
metadata:
  cidr: 192.168.0.0/16
spec:
  ipip:
    enabled: true
    mode: always
  nat-outgoing: true
EOF
```

end
这种模式下，可实现跨网段节点上容器的互通。

```bash
[root@walker-1 manifests]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.16.6.254    0.0.0.0         UG    0      0        0 eth0
169.254.169.254 172.16.6.87     255.255.255.255 UGH   0      0        0 eth0
172.16.6.0      0.0.0.0         255.255.255.0   U     0      0        0 eth0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
192.168.121.0   172.16.199.114  255.255.255.192 UG    0      0        0 tunl0
192.168.135.64  172.16.6.249    255.255.255.192 UG    0      0        0 tunl0
192.168.187.192 0.0.0.0         255.255.255.192 U     0      0        0 *
192.168.187.209 0.0.0.0         255.255.255.255 UH    0      0        0 calic6611247c43
192.168.187.211 0.0.0.0         255.255.255.255 UH    0      0        0 calie50081a277c
```

end
busybox-1 发送报文至 busybox-3 时，根据 master 上路由，会经过 tunl0 设备。tunl0 会为原来的IP报文在封装一层IP协议头。过程如下：

1.busybox-1 向 busybox-3 发送 icmp 报文（略去容器至 calico 网卡步骤）。

src max	dst mac	src ip	dst ip
3a:1d:1e:91:f5:9e	66:39:fa:e7:9f:a9	192.168.187.211	192.168.121.2

2.报文被node 截获，查询本机路由后由 tunl0 设备将报文发出。

src max	dst mac	src ip	dst ip	enc src IP	enc dst IP
fa:16:3e:02:8b:17	00:23:89:8C:E8:31	192.168.187.211	192.168.121.2	172.16.6.47	172.16.199.114

3.报文发送至网关（router), 根据目的IP将报文路由至 172.16.199.114（略去ARP等步骤）。

src max	dst mac	src ip	dst ip	enc src IP	enc dst IP
00:23:89:8C:E8:31	fa:16:3e:d9:a0:5e	192.168.187.211	192.168.121.2	172.16.6.47	172.16.199.114

4.到达 demi-1 后，根据 demi-1 上的路由策略，将报文发送至 busybox-3（略去容器至 calico 网卡步骤） 。

src max	dst mac	src ip	dst ip
8e:6b:fa:f7:5d:3b	de:16:fc:1c:44:35	192.168.187.211	192.168.121.2

Note: ==容器的 gw 为和该容器对应的 calico 网卡。==

虽然实现了 calico 跨网段通信，但对于 busybox-{1,2} 间的通信来说，IP-in-IP 就有点多余了，因为2者宿主机处于同一广播域，2层互通，直接走主机路由即可。

**calico cross-subnet**

```bash
$ calicoctl apply -f - << EOF
apiVersion: v1
kind: ipPool
metadata:
  cidr: 192.168.0.0/16
spec:
  ipip:
    enabled: true
    mode: cross-subnet
  nat-outgoing: true
EOF
```

end
为了解决IP-in-IP 模式下，同网段封装报文的问题，calico 提供了 cross-subnet 的配置选项

[root@walker-1 k8s]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.16.6.254    0.0.0.0         UG    0      0        0 eth0
169.254.169.254 172.16.6.87     255.255.255.255 UGH   0      0        0 eth0
172.16.6.0      0.0.0.0         255.255.255.0   U     0      0        0 eth0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
192.168.121.0   172.16.199.114  255.255.255.192 UG    0      0        0 tunl0
192.168.135.64  172.16.6.249    255.255.255.192 UG    0      0        0 eth0
192.168.187.192 0.0.0.0         255.255.255.192 U     0      0        0 *
192.168.187.209 0.0.0.0         255.255.255.255 UH    0      0        0 calic6611247c43
192.168.187.211 0.0.0.0         255.255.255.255 UH    0      0        0 calie50081a277c

从主机路由可看出，对于同一网段中的路由，直接走 eth0 网卡。



### 引用

1. https://xdhuxc.github.io/posts/mixed/calico/calico-theory-and-architecture/
2. https://juejin.im/post/5d40f34d5188255d6f4faa87
3. https://imliuda.com/post/1015
4. https://www.v2k8s.com/kubernetes/t/205
5. https://www.bladewan.com/2020/11/18/calico_ops/