## 交换机-Switch

1. 交换机类型
2. 交换机概念
3. 交换机协议



### Port

#### Trunk

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

优点

1.可以在不同的交换机之间连接多个VLAN，可以将VLAN扩展到整个网络中；

2.Trunk可以捆绑任何相关的端口，也可以随时取消设置，这样提供了很高的灵活性；

3.Trunk可以提供负载均衡能力以及系统容错。由于Trunk实时平衡各个交换机端口和服务器接口的流量，一旦某个端口出现故障，它会自动把故障端口从Trunk组中撤消，进而重新分配各个Trunk端口的流量，从而实现系统容错；

#### Channel

