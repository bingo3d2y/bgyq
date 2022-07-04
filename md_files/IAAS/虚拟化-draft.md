## 虚拟化

### 虚拟化 and 网络：顶呱呱

假设，网络已经高可用了，那么接入多个网络的服务器也可以实现高可用了。对服务器来说，网络高可用的代价是至少消耗两个网卡即实现bond_mode_1(active-backup)。

如下，服务网卡可以做bond，接两个交换机的网线，实现服务网络高可用。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/DRNI-1.png)

如果，没有虚拟化技术，那么每个物理设备都需要做这个的高可用，才能实现虚拟主机的网络高可用，配置和成本都很高。

在虚拟化技术的加持下，只需要在一个物理服务器上配置网络高可用，该host上所有的虚拟机都可以复用host上的网卡。即，可以实现所以vhost的网络高可用。

### 虚拟机 and 虚拟交换机 and VLAN

虚拟化平台（类似VM vSphere 和 H3C CAS）支持标准虚拟交换机及分布式虚拟交换机。

可以把虚拟交换机，当成一个"二层"可网管的交换机来使用。普通的物理交换机支持的功能与特性，虚拟交换机也支持。组成虚拟化平台物理主机的物理网卡，可以"看成"虚拟交换机与物理交换机之间的"级联线"。

根据主机物理网卡连接到的物理端口的属性（Access、Trunk、链路聚合），可以在虚拟交换机上，以实现不同的网络功能。

#### vswitch上行access

当虚拟交换机（标准交换机或分布式交换机）上行链路（指主机物理网卡）连接到交换机的Access端口时，该虚拟交换机应该与其上行链路物理交换机端口属性相同，如果该主机物理网卡连接到一个VLAN，则该虚拟交换机要配置的VLAN ID属性和物理交换机（物理网卡在一个物理交换机的端口）的VLAN ID一直。

#### vswitch上行trunk

当虚拟交换机上行链路连接到物理交换机的Trunk端口时，虚拟交换机的配置：

* VLAN：在物理交换机Trunk端口允许的vlan id范围内随便选择一个应用到虚拟机上，虚拟机可以与其他虚拟机及物理网络通讯。
* VLAN 中继(trunk)：将虚拟网卡所链接的虚拟交换机端口也设置成trunk？？？

##### 类似kvm vlan的实现吧

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/kvm-vlan.png)



* 宿主机的网卡eth0配置成了Tunk模式，用于和交换机互联，当然交换机的接入端也需要配置成trunk模式

* 子接口eth0.10和eth0.20配置成access port，vlanid分别是10和20，并接入不同的网桥。

* 虚拟机的vnet端接入不同的网桥，就实现了不同虚拟机在不同的vlan

```bash
# more /etc/sysconfig/network-scripts/ifcfg-eth0
DEVICE=bond0
TYPE=Ethernet
BOOTPROTO=static
ONBOOT=yes
# more /etc/sysconfig/network-scripts/ifcfg-eth0.10
DEVICE=eth0.10
TYPE=Ethernet
BOOTPROTO=static
ONBOOT=yes
VLAN=yes

```

end

### OVS vlan组网｛转载｝

https://www.sdnlab.com/20196.html

在OVS中，VLAN的概念和普通交换机一样，不同的是，在OVS中可以通过流表进行VLAN值的修改，这就使得VLAN在OVS中的应用更加灵活。本文通过举例两种基本的应用场景讲述OVS中的VLAN实现，其中场景一和普通交换机工作原理相同，通过access进行设置vlan tag，通过trunk进行转发vlan报文来完成vlan组网；场景二使用OVS的流表进行vlan id的转换来完成vlan的组网。

#### 传统方式设置vlan tag

如下图所示，多台PC设备分别接入不同的SDN交换机，通过vxlan隧道组成大二层网络。其中交换机中的eth1和eth2两个桥端口用于接入PC，vxlan端口通过eth0接入Internet并完成隧道封装和传输，OVS通过VLAN组网把PC1和PC2划分为VLAN 100，把PC3和PC4换分为VLAN 200，从而实现二层的网络隔离。



![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/OVS-VxLan-fig-1.png)

```bash
ovs-vsctl add-br br-eth
ovs-vsctl add-port br-eth eth0
ovs-vsctl add-port br-tap tap1
ovs-vsctl add-port br-tap tap2
## 设置vlan tag
ovs-vsctl set Port tap1 tag=100
ovs-vsctl set Port tap2 tag=200

ovs-vsctl add-port br-eth patch-eth -- set interface patch-eth type=patch options:peer=patch-tap
ovs-vsctl add-port br-tap patch-tap -- set interface patch-tap type=patch options:peer=patch-eth
```



#### OVS流表转换实现vlan组网

这种场景下，接入OVS的某些端口报文自身带有vlan，而转发出去的OVS对应端口需要转换成其它vlan值（类似于OpenStack网络服务中的VLAN网络模式，好举例），用下图所示的组网进行实验测试，其中接入eth0端口的物理链路报文分别带有vlan 100和200，对应的内部虚拟主机VM1和VM2分别使用vlan 1和2进行网络隔离，这里采用OVS完成vlan100和vlan1，vlan200和vlan2的内外部vlan tag的转换。
![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/OVS-VxLan-fig-4.png)

##### 实现原理

为了实现vlan的转换对应关系，我们用OVS建立两个桥，一个桥用于转换进入的vlan100和vlan200报文为vlan1和vlan2报文，另个一桥用于转换进入的vlan1和vlan2报文为vlan100和vlan200，两个OVS桥采用patch port进行连接，vlan的转换采用流表实现，网络结构图如下所示：
![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/OVS-VxLan-fig-5.png)

```bash
ovs-vsctl add-br br-eth
ovs-vsctl add-port br-eth eth0
ovs-vsctl add-port br-tap tap1
ovs-vsctl add-port br-tap tap2
# 设置接口vlan ID即转换后的vlan id
ovs-vsctl set Port tap1 tag=1
ovs-vsctl set Port tap2 tag=2

## 连接网桥
ovs-vsctl add-port br-eth patch-eth -- set interface patch-eth type=patch options:peer=patch-tap
ovs-vsctl add-port br-tap patch-tap -- set interface patch-tap type=patch options:peer=patch-eth

## 设置流变转换
ovs-ofctl add-flow br-eth in_port=1,priority=2,dl_vlan=100,actions=mod_vlan_vid:1,NORMAL
ovs-ofctl add-flow br-eth in_port=1,priority=2,dl_vlan=100,actions=mod_vlan_vid:1,NORMAL
ovs-ofctl add-flow br-eth in_port=1,priority=1,actions=drop

ovs-ofctl add-flow br-tap in_port=1,priority=2,dl_vlan=1,actions=mod_vlan_vid:100,NORMAL
ovs-ofctl add-flow br-tap in_port=2,priority=2,dl_vlan=2,actions=mod_vlan_vid:200,NORMAL
ovs-ofctl add-flow br-tap in_port=1,priority=1,actions=drop
ovs-ofctl add-flow br-tap in_port=2,priority=1,actions=drop
```

end

### 引用

1. https://www.jianshu.com/p/dfe605ca8ca7
1. https://zhuanlan.zhihu.com/p/37408055
1. https://www.sdnlab.com/20196.html
