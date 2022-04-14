## 网络实操-draft

### 从CAS看VM vlan ID：

#### 问题

CAS虚拟化平台上创建了一个VM，选了`vSwitch-172-200-50`但是配置好IP后，VM无法ping同网关`172.200.50.1`.

#### 解决

在论坛看到了如下提示:

虚拟机网络虚拟机网段vlan是多少 需要虚拟交换机对应相应的vlan 才能访问的

在查看该网端下其他可以ping通网关的VM，他们的VLAN ID是140

而我创建的VM VLAN ID是1，从而导致了无法ping通同网段网关的现象。

#### 收获:star2:

**VM Vlan id不对（和物理机NIC 的VLAN ID不同），port直接丢掉了VM的包，所以根本到不了网关呢**

一个物理机有N个网卡，做完bond聚合后，按不同功能的网络（业务、存储和管理）划分网卡。

eg：

eth2 & eth3 作为管理网卡，vlan ID为140，网段为172.200.50.0/24

eth7 & eth8 作为存储外网，vlan ID为110，网段为172.200.20.0/24

CAS虚拟化平台，先创建vSwitch，配置vSwitch使用物理机的哪个网卡，但是此时并没有指定VM使用哪个VLAN，默认就是VLAN 1

然后CAS平台提供了“网络策略模版”的功能，网络策略中配置了使用的VLAN，可以附加到vSwitch上，最终决定了平台虚拟机的网络。

vSwitch选择了物理机网卡作为出口，但是物理网卡本身被划分了VLAN ID，当你的虚拟机配置的VLAN ID和物理机网卡出口VLAN ID不一致时，就回导致虚拟机ping不通网关。

##### 查看 VM VLAN

```bash
 $ virsh edit test-vm
 ...
  <interface type='bridge'>
      <mac address='0c:da:41:1d:39:6c'/>
      <source bridge='test_vs' mode='bridge'/>
      <vlan>
        <tag id='1'/>
      </vlan>
      <model type='virtio'/>
      <driver name='vhost'/>
      <priority type='low'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </interface>
...

```

end

##### tcpdump抓vlan包

```bash
## eth7 抓到很多vlan 包--
$ tcpdump -ev -i eth7 vlan
tcpdump: listening on eth7, link-type EN10MB (Ethernet), capture size 262144 bytes
18:10:35.093761 2e:99:88:a8:a6:a2 (oui Unknown) > Broadcast, ethertype 802.1Q (0x8100), length 64: vlan 110, p 0, ethertype ARP, Ethernet (len 6), IPv4 (len 4), Request who-has 172.200.20.3 (Broadcast) tell 172.200.20.3, length 46
18:10:35.093796 2e:99:88:a8:a6:a2 (oui Unknown) > 01:00:5e:00:00:12 (oui Unknown), ethertype 802.1Q (0x8100), length 64: vlan 110, p 0, ethertype IPv4, (tos 0xc0, ttl 255, id 12932, offset 0, flags [none], proto VRRP (112), length 40)
    172.200.20.11 > 224.0.0.18: vrrp 172.200.20.11 > 224.0.0.18: VRRPv2, Advertisement, vrid 2, prio 240, authtype simple, intvl 3s, length 20, addrs: 172.200.20.3 auth "1111^@^@^@^@"
18:10:35.140821 fa:16:3e:a2:bf:34 (oui Unknown) > Broadcast, ethertype 802.1Q (0x8100), length 64: vlan 2078, p 0, ethertype ARP, Ethernet (len 6), IPv4 (len 4), Request who-has 10.200.90.13 tell 10.200.90.11, length 46
18:10:35.141494 fa:16:3e:a2:bf:34 (oui Unknown) > Broadcast, ethertype 802.1Q (0x8100), length 64: vlan 2078, p 0, ethertype ARP, Ethernet (len 6), IPv4 (len 4), Request who-has 10.200.90.15 tell 10.200.90.11, length 46
18:10:35.181382 fa:16:3e:86:a8:4d (oui Unknown) > fa:16:3e:b8:77:8f (oui Unknown), ethertype 802.1Q (0x8100), length 70: vlan 2099, p 0, ethertype IPv4, (tos 0x0, ttl 64, id 987, offset 0, flags [DF], proto TCP (6), length 52)
    10.200.60.183.44815 > 10.200.60.182.l2c-data: Flags [.], cksum 0x8f23 (incorrect -> 0x21e3), ack 339504112, win 1418, options [nop,nop,TS val 527943880 ecr 963015109], length 0
18:10:35.474073 fa:16:3e:86:a8:4d (oui Unknown) > fa:16:3e:c9:66:48 (oui Unknown), ethertype 802.1Q (0x8100), length 70: vlan 2099, p 0, ethertype IPv4, (tos 0x0, ttl 64, id 9328, offset 0, flags [DF], proto TCP (6), length 52)
    10.200.60.183.57506 > 10.200.60.181.bgp: Flags [.], cksum 0x3965 (correct), ack 4079796666, win 229, options [nop,nop,TS val 527944172 ecr 527942633], length 0
18:10:35.476154 74:3a:20:1f:cf:e2 (oui Unknown) > 74:3a:20:1f:cd:12 (oui Unknown), ethertype 802.1Q (0x8100), length 64: vlan 3, p 0, ethertype ARP, Ethernet (len 6), IPv4 (len 4), Request who-has cvknode3 tell 3.3.3.24, length 46
18:10:35.476354 74:3a:20:1f:cd:12 (oui Unknown) > 74:3a:20:1f:cf:e2 (oui Unknown), 

## eth2 网卡抓到的是vlan 140的包
$ tcpdump -e -i eth2 vlan |head -n3
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode

18:50:01.964338 0c:da:41:1d:38:b4 (oui Unknown) > 0c:da:41:1d:03:73 (oui Unknown), ethertype 802.1Q (0x8100), length 64: vlan 140, p 0, ethertype IPv4, 172.200.50.101.42964 > 172.200.50.221.ssh: Flags [.], ack 1336051340, win 12313, length 0
18:50:01.971567 00:be:d5:e9:ce:a6 (oui Unknown) > 00:be:d5:e9:ce:5e (oui Unknown), ethertype 802.1Q (0x8100), length 70: vlan 140, p 0, ethertype IPv4, cvknode3.35534 > cvknode1.ssh: Flags [S], seq 2367578562, win 29200, options [mss 1460,nop,nop,sackOK,nop,wscale 7], length 0
18:50:01.971637 00:be:d5:e9:ce:5e (oui Unknown) > 00:be:d5:e9:ce:a6 (oui Unknown), ethertype 802.1Q (0x8100), length 70: vlan 140, p 0, ethertype IPv4, cvknode1.ssh > cvknode3.35534: Flags [S.], seq 2655920071, ack 2367578563, win 29200, options [mss 1460,nop,nop,sackOK,nop,wscale 7], length 0

```

end

##### DHCP分配IP标识

```bash
# 配置了静态IP然后发现有两个IP地址
$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 0c:da:41:1d:94:48 brd ff:ff:ff:ff:ff:ff
    inet 172.200.50.220/24 brd 172.200.50.255 scope global noprefixroute dynamic eth0
       valid_lft 3564sec preferred_lft 3564sec
    inet 172.200.50.223/24 brd 172.200.50.255 scope global secondary noprefixroute eth0
       valid_lft forever preferred_lft forever
## 将BOOTPROTO="dhcp"改成BOOTPROTO="none"
$ vi /etc/sysconfig/network-scripts/ifcfg-eth0
## 重启网络，dhcp分配的IP地址消失
$ systemctl restart network
$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 0c:da:41:1d:94:48 brd ff:ff:ff:ff:ff:ff
    inet 172.200.50.223/24 brd 172.200.50.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft foreve
```

end

### 云平台下发Mysql集群

#### 现象

客户下发Mysql集群失败，远程排查故障

#### 思路

cloudos平台部署mysql服务逻辑很简单，选择计算可用域和网络，指定规格创建和配置虚机即可。

计算域没有问题，而且恰巧我知道有两个网络可以选择：

* VPC：af-xwzx，可用
* 经典网络：af-jyz，不可用

所以，查看客户创建mysql时的配置信息，如下：

```bash
{
    "type":"mysql",
    "name":"afnetmysql",
    "is_share":false,
    "cluster_mode":"0",
    "administrator":"admin",
    "administrator_password":"admin@123",
    "administrator_confirm_password":"admin@123",
    "description":"123",
    ...
    "availability_zone":{
        "id":"3b46ff60-4c07-49d1-a5b1-23dd442f6ffe",
        "name":"cas2",
        "virtType":"CAS"
    },
    // 网络配置在这里--
    "neutron_management_network_id":"0e91456d-c3b9-4ded-9e97-1bb86a289fff",
    "keypair_name":"test",
    "manager_virtual_ip":"",
    
    ...
    "database_init_password":"admin@123",
    "database_init_port":3306,
    "db_schema":"standalone",
    "sync_count":0
}
```

在查看经典网络af-jyz的信息，判断是否吻合

```bash
{
    "data":[
        {
            "name":"AF_JYZ",
            // ID 和 mysql创建时现在的network_id一致
            "id":"0e91456d-c3b9-4ded-9e97-1bb86a289fff",
            "tenantId":"30d50b11bda14b209d1ab25603c6b8d6",
            ...
            "aliasZoneIds":[
                "3b46ff60-4c07-49d1-a5b1-23dd442f6ffe"
            ],
            "preGateway":null,
            "tenantName":"生产域-安防集成平台",
            "routerName":"-----",
            "alias_zone_names":"cas2",
            "subnetNameAndCidr":null,
            "subnetSharedModify":true,
            "base64ProviderPhysicalNetwork":null
...
}
```

结论：创建mysql时选择了错误的经典网络。

### host连不上存储网络: lldp

lldp neighbor 查看



现象：一个机器上rbd全部down了

定位：cvk-node-5节点ping 本机上存储外网网卡失败，然后断定是网卡问题

查看交换机配置表：

| XG1/0/17 | cvknode5-slot1-port1（eth6） | Trunk | 110-120 | 11   |
| -------- | ---------------------------- | ----- | ------- | ---- |
| XG1/0/20 | cvknode5-slot3-port1（eth0） | Trunk | 110-120 | 11   |

连上交换机查看交换机bound是否端口聚合是否出错

果然：发现本来该是port 20 和 port 17聚合的，结果是port 16 和 port 27了



难点：

1. 我怎么快速定位是因为node连不上存储外网IP呢
2. 我怎么快速在交换机上定位port配置问题呢
3. 我怎么快速对比计划配置和现存配置的区别呢

### centos 添加静态路由

问题：

租管互通的管理节点无法访问一个业务网段的vm

原因：

租管互通的管理网节点node上部署了k8s，k8s service ip的路由覆盖了vm路由导致无法访问。

```bash

$ ip r
10.13.50.128/25 dev bond0 proto kernel scope link src 
...
## k8s service 路由
10.100.0.0/16 dev tun0
10.240.0.0/12 dev tun0 scope link
172.17.0.0/16 dev docker0 proto kernel scope link src 
...
```

解决办法，添加某个租户业务网段所在的路由-

创建`etc/sysconfig/network-scripts/route-eth0`文件，eth0根据实际网卡名称替换

```bash
# 在 `/etc/sysconfig/network-scripts/` 目录下创建名为 route-eth0 的文件
vi /etc/sysconfig/network-scripts/route-eth0
# 在此文件添加如下格式的内容
10.100.180.0/24 via 172.100.50.1
# 重启网络验证有效
systemctl restart network
```

