## vxlan组网例子

### vxlan外部连接方式

https://blog.csdn.net/qq_43691045/article/details/103598254



### 华三官网例子：作业

http://www.h3c.com/cn/d_201711/1046721_30005_0.htm#_Toc499551672

#### 头端复制





### test

#### 1.  VxLAN内大二层互通：缺配置

https://blog.csdn.net/weixin_40228200/article/details/119303394

类似，传统VLAN 二层互通转发，实现PC_1 和 PC_2的通讯。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vxlan-basic.png)

缺了配置，Route 和 CE Switch怎么配置的

1、VxLAN业务配置

要配置VXLAN业务，就必须先配置BD域，BD域使用不同的数字进行区分，每一个BD域只能关联一个VNI，并且一个VNI也只能被一个BD域所关联。但是BD域具体的数字并没有太大的意义，关于VXLAN业务首先需要配置BD域以及VNI，相关配置如下

```bash
bridge-domain 10
vxlan vni 10
```

end

2、VXLAN数据包归属配置

在完成上述配置后，还必须告诉CE交换机，什么样的数据包要进行VXLAN封装，在这里我们将CE下行交换机的接口设置为Trunk类型，并且在CE交换机上设置二层子接口，并在子接口上配置关联，这样，所有属于VLAN 10的流量就和VXLAN相关联了。相关配置如下

```bash
interface GE1/0/1.10 mode l2
encapsulation dot1q vid 10
bridge-domain 10
```

end

3、VxLAN隧道配置

在完成上述配置后，就要配置一条VXLAN的隧道，在这里，默认情况下以Loopbake接口作为VXLAN的端点口，相关配置如下：

```bash
interface Nve1
 source 1.1.1.1
 vni 10 head-end peer-list 2.2.2.2
```

end

#### 2. 相同子网互通

某企业在不同的数据中心中都拥有自己的VM，服务器1上的VM1属于VLAN10，服务器2上的VM1属于VLAN20，且位于同网段。现需要通过VXLAN隧道实现不同数据中心同网段VM的互通。

**拓扑**

Device_1 连接数据中心1

Device_3 连接数据中心2

Device_2 连接Device_1 和 Device_3

**配置思路：**

1. 分别在Device1、Device2和Device3上配置路由协议，保证网络三层互通。
2. 分别在Device1和Device3上配置业务接入点实现区分业务流量。
3. 分别在Device1和Device3上配置VXLAN隧道实现转发业务流量。

**数据准备：**

1. VM所属的VLAN ID分别是VLAN10和VLAN20。
2. 网络中设备互连的接口IP地址。
3. 网络中使用的路由类型是OSPF（Open Shortest Path First）。
4. 广播域BD ID是BD10。
5. VXLAN网络标识VNI ID是VNI 5010。
6. #

#### 3. 不同子网互通（集中式）

**Spine-Leaf拓扑**

两个Spine交换机 Spine_1 and Spine_2 做高可用

Leaf_1、Leaf_2和Leaf_3代表三个子网。

Leaf 1-3分别连接 Spine 1-2.

**组网需求：**

该数据中心流量主要为虚拟机迁移等形成的东西向流量，因此，客户要求在现有基础承载网络上构建一个大二层网络，并且要求网络中的Spine设备同时作为网关设备，可以对Leaf设备发送过来的报文进行负载分担。

如图2-53所示，某企业数据中心内部网络为“Spine-Leaf”两层结构：

- Spine1和Spine2为基础承载网络中的骨干节点，位于汇聚层；
- Leaf1～Leaf3为基础承载网络中的叶子节点，位于接入层；
- Leaf设备与Spine设备之间全连接，构成ECMP，保证网络的高可用性；Spine节点之间互不连接，Leaf节点之间也不互联。

**配置思路：**采用如下思路配置集中式多活网关：

1. 在Leaf1～Leaf3、Spine1～Spine2上配置OSPF，保证网络三层互通。
2. 在Leaf1～Leaf3、Spine1～Spine2上配置VXLAN隧道，实现在基础三层网络上创建虚拟大二层VXLAN网络。
3. 在Leaf1～Leaf3上配置业务接入点，区分出服务器的流量并转发至VXLAN网络。
4. 在Spine1～Spine2上配置VXLAN三层网关实现VXLAN与非VXLAN网络、不同网段VXLAN之间的通信。

注意：如果Spine1/Spine2接入上行网络的链路出现故障，当用户流量到达Spine1/Spine2时，由于没有可用的上行出接口，Spine1/Spine2会将用户流量全部丢弃。此时，可以配置monitor-link关联Spine1/Spine2的上行接口和下行接口，当Spine1/Spine2的上行出接口DOWN时，下行接口状态也会变为DOWN，这样用户侧流量就不会通过Spine1/Spine2进行转发，从而防止流量被丢弃。monitor-link的配置可以参见配置Monitor Link组的上行接口和下行接口。

**数据准备：**

1. 网络中设备互连的接口IP地址。
2. 网络中使用的路由类型是OSPF（Open Shortest Path First）。
3. VM所属的VLAN ID分别是VLAN10、VLAN20和VLAN30。
4. 广播域BD ID分别是BD10、BD20和BD30。
5. VXLAN网络标识VNI ID分别是VNI5000、VNI5001和VNI5002。

**配置步骤：**

**配置OSPF(略)**

1. 在Leaf1～Leaf3、Spine1～Spine2配置OSPF。
2. 配置OSPF时，注意需要发布32位Loopback接口地址。
3. OSPF成功配置后，设备之间可通过OSPF协议发现对方的Loopback接口的IP地址，并能互相ping通。

**配置隧道模式并使能NVO3的ACL扩展功能**

```bash
# 配置Leaf1。Spine2、Leaf1、Leaf2和Leaf3的配置与Leaf1类似，这里不再赘述。
[~Spine1] ip tunnel mode vxlan
[*Spine1] assign forward nvo3 acl extend enable
[*Spine1] commit

注意：
1.    从V100R005C10版本开始，支持使能NVO3的ACL扩展功能。
2.    配置隧道模式、使能NVO3的ACL扩展功能后，需要保存配置并重启设备才能生效，您可以选择立即重启或完成所有配置后再重启。
```

**配置VXLAN业务接入点**

```bash
在Leaf1～Leaf3配置VXLAN业务接入点。
配置Leaf1。
[~Leaf1] bridge-domain 10
[*Leaf1-bd10] quit
[~Leaf1] interface 10ge 1/0/3.1 mode l2
[*Leaf1-10GE1/0/3.1] encapsulation dot1q vid 10
[*Leaf1-10GE1/0/3.1] bridge-domain 10
[*Leaf1-10GE1/0/3.1] quit
[*Leaf1] commit

配置Leaf2。
[~Leaf2] bridge-domain 20
[*Leaf2-bd10] quit
[~Leaf2] interface 10ge 1/0/3.1 mode l2
[*Leaf2-10GE1/0/3.1] encapsulation dot1q vid 20
[*Leaf2-10GE1/0/3.1] bridge-domain 20
[*Leaf2-10GE1/0/3.1] quit
[*Leaf2] commit

配置Leaf3。
[~Leaf3] bridge-domain 30
[*Leaf3-bd10] quit
[~Leaf3] interface 10ge 1/0/3.1 mode l2
[*Leaf3-10GE1/0/3.1] encapsulation dot1q vid 30
[*Leaf3-10GE1/0/3.1] bridge-domain 30
[*Leaf3-10GE1/0/3.1] quit
[*Leaf3] commit
```

**配置VXLAN隧道**

三个Leaf的VxLAN对端都是Spine 1 and Spine 2的 vip 10.10.10.1

```bash
在Leaf1～Leaf3、Spine1～Spine2上配置VXLAN隧道，组建VXLAN网络

配置Leaf1。
[~Leaf1] bridge-domain 10
[*Leaf1-bd10] vxlan vni 5000
[*Leaf1-bd10] quit
[*Leaf1] interface nve 1
[*Leaf1-Nve1] source 10.10.10.3
# //指定对端VTEP地址，并调用VNI
[*Leaf1-Nve1] vni 5000 head-end peer-list 10.10.10.1
[*Leaf1-Nve1] quit
[*Leaf1] commit

配置Leaf2。
[~Leaf2] bridge-domain 20
[*Leaf2-bd10] vxlan vni 5001
[*Leaf2-bd10] quit
[*Leaf2] interface nve 1
[*Leaf2-Nve1] source 10.10.10.4
# //指定对端VTEP地址，并调用VNI
[*Leaf2-Nve1] vni 5001 head-end peer-list 10.10.10.1
[*Leaf2-Nve1] quit
[*Leaf2] commit

配置Leaf3。
[~Leaf3] bridge-domain 30
[*Leaf3-bd10] vxlan vni 5002
[*Leaf3-bd10] quit
[*Leaf3] interface nve 1
[*Leaf3-Nve1] source 10.10.10.5
# //指定对端VTEP地址，并调用VNI
[*Leaf3-Nve1] vni 5002 head-end peer-list 10.10.10.1
[*Leaf3-Nve1] quit
[*Leaf3] commit

# 配置Spine1。Spine2的配置与Spine1一样，此处不再赘述。
[~Spine1] bridge-domain 10
[*Spine1-bd10] vxlan vni 5000
[*Spine1-bd10] quit
[*Spine1] bridge-domain 20
[*Spine1-bd20] vxlan vni 5001
[*Spine1-bd20] quit
[*Spine1] bridge-domain 30
[*Spine1-bd30] vxlan vni 5002
[*Spine1-bd30] quit

# 10.10.10.1 就是 Spine Switch IP
[*Spine1] interface nve 1
[*Spine1-Nve1] source 10.10.10.1
[*Spine1-Nve1] vni 5000 head-end peer-list 10.10.10.3
[*Spine1-Nve1] vni 5001 head-end peer-list 10.10.10.4
[*Spine1-Nve1] vni 5002 head-end peer-list 10.10.10.5
[*Spine1-Nve1] quit
[*Spine1] commit
```

**上述配置成功后，在Leaf1～Leaf3、Spine1～Spine2上执行display vxlan vni命令可查看到VNI的状态是up；执行display vxlan tunnel命令可查看到VXLAN隧道的信息。以Spine1显示为例。**

```bash
[~Spine1] display vxlan vni
Number of vxlan vni : 3
VNI BD-ID State
---------------------------------------
5000 10 up
5001 20 up
5002 30 up

[~Spine1] display vxlan tunnel
Number of vxlan tunnel : 3
Tunnel ID Source Destination State Type
--------------------------------------------------------------
4026531842 10.10.10.1 10.10.10.3 up static
4026531843 10.10.10.1 10.10.10.4 up static
4026531844 10.10.10.1 10.10.10.5 up static
```

**配置VXLAN三层网关**

```bash
在Spine1～Spine2上配置VXLAN三层网关
# 配置Spine1。Spine2的配置与Spine1一样，此处不再赘述。
[~Spine1] interface vbdif 10
[*Spine1-Vbdif10] ip address 192.168.10.1 24
[*Spine1-Vbdif10] mac-address 0000-5e00-0101
[*Spine1-Vbdif10] quit
[*Spine1] interface vbdif 20
[*Spine1-Vbdif20] ip address 192.168.20.1 24
[*Spine1-Vbdif20] mac-address 0000-5e00-0102
[*Spine1-Vbdif20] quit
[*Spine1] interface vbdif 30
[*Spine1-Vbdif30] ip address 192.168.30.1 24
[*Spine1-Vbdif30] mac-address 0000-5e00-0103
[*Spine1-Vbdif30] quit
[*Spine1] commit

注意：
由于Spine1和Spine2作为多活网关设备，请确保这两台设备上配置的NVE接口的IP地址、BDIF接口的IP地址和MAC地址相同。
```

**配置VXLAN三层网关多活**

```bash
在Spine1～Spine2上配置多活网关
# 配置Spine1。Spine2的配置与Spine1类似，此处不再赘述。
[~Spine1] dfs-group 1
[*Spine1-dfs-group-1] source ip 10.10.10.10
[*Spine1-dfs-group-1] active-active-gateway
[*Spine1-dfs-group-1-active-active-gateway] peer 10.10.10.20
[*Spine1-dfs-group-1-active-active-gateway] quit
[*Spine1-dfs-group-1] quit
[*Spine1] commit
```

检查配置结果，上述配置成功后，在Spine1、Spine2上执行命令display dfs-group 1 active-activegateway，可以查看到DFS Group多活网关的信息。以Spine1显示为例。

```bash
[~Spine1] display dfs-group 1 active-active-gateway
A:Active I:Inactive
-------------------------------------------------------------------
Peer System name State Duration
10.10.10.20 Spine2 A 0:0:8
```

end