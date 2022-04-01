## cloud-init

保证使用私有镜像创建的新云服务器可以自定义配置的工具

看：

https://docs.openstack.org/nova/rocky/user/metadata-service.html

--

https://docs.openstack.org/nova/rocky/admin/networking-nova.html#metadata-service-deploy



https://zhangchenchen.github.io/2017/01/13/openstack-init-instance-password/

### metadata

This metadata is made available via either a *config drive* or the *metadata service* and can be somewhat customised by the user using the *user data* feature. 

这些配置信息被分成两类：metadata 和 user data。Metadata 主要包括虚拟机自身的一些常用属性，如 hostname、网络配置信息、SSH 登陆秘钥等，主要的形式为键值对。而 user data 主要包括一些命令、脚本等。User data 通过文件传递，并支持多种文件格式，包括 gzip 压缩文件、shell 脚本、cloud-init 配置文件等。这些信息被获取到后被cloudinit 利用并实现各自目的。虽然 metadata 和 user data 并不相同，但是 OpenStack 向虚拟机提供这两种信息的机制是一致的，只是虚拟机在获取到信息后，对两者的处理方式不同罢了

#### User data

*User data* is a blob of data that the user can specify when they launch an instance. The instance can access this data through the metadata service or config drive. Commonly used to pass a shell script that the instance runs on boot.

You can place user data in a local file and pass it through the `--user-data <user-data-file>` parameter at instance creation.

```bash
$ openstack server create --image ubuntu-cloudimage --flavor 1 --user-data mydata.file VM_INSTANCE
```

end

#### Nova metadata

metadata service or config drive ???

As noted previously, nova provides its metadata in two formats: OpenStack format and EC2-compatible format.

OpenStack format metadata:

Metadata from the OpenStack API is distributed in JSON format. There are two files provided for each version: `meta_data.json` and `network_data.json`. The `meta_data.json` file contains nova-specific information, while the `network_data.json` file contains information retrieved from neutron. 



EC2-compatible metadata:

The EC2 API exposes a separate URL for each metadata element. Retrieve a listing of these elements by making a GET query to `http://169.254.169.254/2009-04-04/meta-data/`. 

#### Vendor data

**Where configured**, instances can retrieve vendor-specific data from the metadata service or config drive. To access it via the metadata service, make a GET request to either `http://169.254.169.254/openstack/{version}/vendor_data.json` or `http://169.254.169.254/openstack/{version}/vendor_data2.json`, depending on the deployment. For example:

```bash
# metadata service
$ curl http://169.254.169.254/openstack/2018-08-27/vendor_data2.json

# config drive
$ ls /dev/disk/by-label/config-2
```

end

### metadata service:eyeglasses:

The *metadata service* provides a way for instances to retrieve instance-specific data via a REST API. Instances access this service at `169.254.169.254` or at `fe80::a9fe:a9fe`. All types of metadata, be it user-, nova- or vendor-provided, can be accessed via this service.

在虚拟机通过169.254.169.254的方式来获取metadata的方式。由以下三个组件来完成这项工作：

- Nova-api-metadata：运行在计算节点，启动restful服务，真正负责处理虚拟机发送来的rest 请求。从请求头中获取租户，虚拟机ID，再去数据库中查询相应的metadata信息并返回结果。
- Neutron-metadata-agent：运行在网络节点，负责将接收到的获取 metadata 的请求转发给 nova-api-metadata。
- Neutron-ns-metadata-proxy：由于虚拟机获取 metadata 的请求都是以路由和 DHCP 服务器作为网络出口，所以需要通过 neutron-ns-metadata-proxy 联通不同的网络命名空间，将请求在网络命名空间之间转发

#### 大佬

https://lk668.github.io/2020/10/12/2020-10-12-neutron-metadata/

整个的工作流程如下图所示。经过neutron-metadata-agent进行转发的原因是，nova-api-metadata服务运行在管理网络，而虚拟机是运行在业务网络，两者可能不能通信。而在neutron-metadata-agent之前又嫁接一层neutron-ns-metadata-proxy的原因是在openstack中，不同的网络可以配置相同的CIDR，nova-api-metadata无法根据虚拟机的ip来判定虚拟机的id，所以需要在neutron-ns-metadata-agent服务添加network-id后者router-id。这样就可以根据instance ip，network-id/router-id来获取到唯一的虚拟机id。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/metadata-1.png)



OpenStack的Metadata服务主要是为虚拟机提供配置信息。在虚拟机启动的时候，会向Metadata服务请求自己的metadata信息，然后执行cloud-init的时候完成个性化配置。

流程请求如下：

1. 虚拟机启动的时候会配置默认路由
   虚拟机启动的时候，会进行默认路由的配置，配置访问169.254.169.254的路由信息
2. 虚拟机请求169.254.169.254
   请求到达`qrouter-<network-id>`或者`qdhcp-<network-id>`的网络命名空间。如果该网络没有配置router，是到达`qdhcp-<network-id>`的网络命名空间，如果配有router，会到达`qrouter-<network-id>`的网络命名空间。
3. 请求到达`qrouter-<network-id>`或者`qdhcp-<network-id>`网络命名空间
   请求到达网络命名空间以后，经由neutron-ns-metadata-proxy，添加router-id或者network-id，然后通过socket转发给neutron-metadata-agent。
4. 请求通过unix socket转发给neutron-metadata-agent服务
   neutron-metadata-agent通过请求头部的network-id/router-id以及请求的ip地址，获取到instance id，将请求转发给nova-api-metadata。
5. neutron-metadata-agent服务将请求转发给nova-api-metadata。
   nova-api-metadata根据instance id回复metadata response。

#### 基于`qrouter-<network-id>`命名空间的metadata服务实现

当openstack中的网络连接router时，neutron-metadata-agent通过`qrouter-<network-id>`网络命名空间来提供metadata服务。

此时，登录虚拟机，看一下路由规则。可以看到在请求169.254.169.254时，通过gateway进行转发，而gateway是配置在`qrouter-<network-id>` 网络命名空间下的

```bash
#虚拟机路由
[root@test ~]# ip route
default via 10.0.0.1 dev eth0
10.0.0.0/24 dev eth0 proto kernel scope link src 10.0.0.14
169.254.169.254 via 10.0.0.1 dev eth0 proto static

#qrouter-网络命名空间信息
[root@tstack-con01 ~]# ip netns exec qrouter-f8156a50-cf9e-430c-a8a6-64a87e41fd01 ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
31: qr-6a921526-09: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether fa:16:3e:6f:53:5d brd ff:ff:ff:ff:ff:ff
    inet 10.1.0.1/24 brd 10.1.0.255 scope global qr-6a921526-09
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fe6f:535d/64 scope link
       valid_lft forever preferred_lft forever
32: qr-853fe99d-e7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether fa:16:3e:5d:07:11 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.1/24 brd 10.0.0.255 scope global qr-853fe99d-e7
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fe5d:711/64 scope link
       valid_lft forever preferred_lft forever
```

那么请求在`qrouter-<network-id>`的命名空间是如何转发的呢？答案是通过iptables

```bash
[root@tstack-con01 ~]# ip netns exec qrouter-f8156a50-cf9e-430c-a8a6-64a87e41fd01 iptables-save|grep 169
-A neutron-l3-agent-PREROUTING -d 169.254.169.254/32 -i qr-+ -p tcp -m tcp --dport 80 -j REDIRECT --to-ports 9697
-A neutron-l3-agent-PREROUTING -d 169.254.169.254/32 -i qr-+ -p tcp -m tcp --dport 80 -j MARK --set-xmark 0x1/0xffff
```

会把对169.254.169.254，80端口的请求转发到9697端口。而9697端口就运行着neutron-ns-metadata-proxy服务，该服务实际上是一个haproxy进行。

```bash
[root@tstack-con01 ~]# ip netns exec qrouter-f8156a50-cf9e-430c-a8a6-64a87e41fd01 lsof -i:9697
COMMAND     PID    USER   FD   TYPE   DEVICE SIZE/OFF NODE NAME
haproxy 1592727 neutron    4u  IPv4 27760235      0t0  TCP *:9697 (LISTEN)
[root@tstack-con01 ~]# ip netns exec qrouter-f8156a50-cf9e-430c-a8a6-64a87e41fd01 ps -aux|grep 1592727
neutron  1592727  0.0  0.0  47940  1196 ?        Ss   Feb01   0:01 haproxy -f /var/lib/neutron/ns-metadata-proxy/f8156a50-cf9e-430c-a8a6-64a87e41fd01.conf
root     1856176  0.0  0.0 112708   952 pts/1    S+   00:34   0:00 grep --color=auto 1592727
[root@tstack-con01 ~]#
```

查看一下配置文件/var/lib/neutron/ns-metadata-proxy/

```bash
[root@tstack-con01 ~]# cat /var/lib/neutron/ns-metadata-proxy/f8156a50-cf9e-430c-a8a6-64a87e41fd01.conf

global
    log         /dev/log local0 debug
    log-tag     haproxy-metadata-proxy-f8156a50-cf9e-430c-a8a6-64a87e41fd01
    user        neutron
    group       neutron
    maxconn     1024
    pidfile     /var/lib/neutron/external/pids/f8156a50-cf9e-430c-a8a6-64a87e41fd01.pid
    daemon

defaults
    log global
    mode http
    option httplog
    option dontlognull
    option http-server-close
    option forwardfor
    retries                 3
    timeout http-request    30s
    timeout connect         30s
    timeout client          32s
    timeout server          32s
    timeout http-keep-alive 30s

listen listener
    bind 0.0.0.0:9697
    server metadata /var/lib/neutron/metadata_proxy
    http-request add-header X-Neutron-Router-ID f8156a50-cf9e-430c-a8a6-64a87e41fd01
```

实际上neutron-ns-metadata-proxy用过socket连接neutron-metadata-agent，然后在请求头部添加router-id，neutorn-metadata-agent就会根据请求的ipd地址和router-id获取到虚拟机的id。然后将请求发送给nova-api-metadata服务。

#### 基于`qdhcp-<network-id>`命名空间的metadata服务实现

如果网络没有绑定router，那么neutron通过`qdhcp-<network-id>`网络命名空间来处理metadata请求。

此时的实现方式，会在`qdhcp-<network-id>`网络命名空间添加一个169.254.169.254的网卡。

```bash
[root@openstack-con01 ~]# ip netns exec qdhcp-35a559be-9f45-4d4e-849f-6c58b7e8ddd1 ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
10: tap768dfac8-23: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether fa:16:3e:fd:4d:8e brd ff:ff:ff:ff:ff:ff
    inet 172.16.0.2/24 brd 172.16.0.255 scope global tap768dfac8-23
       valid_lft forever preferred_lft forever
    inet 169.254.169.254/16 brd 169.254.255.255 scope global tap768dfac8-23
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fefd:4d8e/64 scope link
       valid_lft forever preferred_lft forever
```

neutron-ns-metadata-proxy运行在80端口，实际就是一个haproxy服务。这样对169.254.169.254的80端口的访问都会直接到达该服务。

```bash
[root@openstack-con01 ~]# ip netns exec qdhcp-35a559be-9f45-4d4e-849f-6c58b7e8ddd1 lsof -i:80
COMMAND  PID    USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
haproxy 5443 neutron    4u  IPv4  41711      0t0  TCP *:http (LISTEN)
[root@openstack-con01 ~]# ip netns exec qdhcp-35a559be-9f45-4d4e-849f-6c58b7e8ddd1 ps -aux|grep 5443
neutron     5443  0.0  0.0  47936   868 ?        Ss    2020  15:39 haproxy -f /var/lib/neutron/ns-metadata-proxy/35a559be-9f45-4d4e-849f-6c58b7e8ddd1.conf
root     3203881  0.0  0.0 112708   968 pts/0    S+   18:20   0:00 grep --color=auto 5443
```

看一下haproxy的配置文件，实际上通过socket连接neutron-metadata-agent，然后在请求的头部添加network id。这样neutron-metadata-agent就可以通过虚拟机的ip和network的id来获取instance id，然后想nova-api-metadata请求对应instance的metadata信息。

```bash
[root@openstack-con01 ~]# cat /var/lib/neutron/ns-metadata-proxy/35a559be-9f45-4d4e-849f-6c58b7e8ddd1.conf

global
    log         /dev/log local0 info
    log-tag     haproxy-metadata-proxy-35a559be-9f45-4d4e-849f-6c58b7e8ddd1
    user        neutron
    group       neutron
    maxconn     1024
    pidfile     /var/lib/neutron/external/pids/35a559be-9f45-4d4e-849f-6c58b7e8ddd1.pid
    daemon

defaults
    log global
    mode http
    option httplog
    option dontlognull
    option http-server-close
    option forwardfor
    retries                 3
    timeout http-request    30s
    timeout connect         30s
    timeout client          32s
    timeout server          32s
    timeout http-keep-alive 30s

listen listener
    bind 0.0.0.0:80
    server metadata /var/lib/neutron/metadata_proxy
    http-request add-header X-Neutron-Network-ID 35a559be-9f45-4d4e-849f-6c58b7e8ddd1
```

end

### config drive:eyes:

*Config drives* are special drives that are attached to an instance when it boots. The instance can mount this drive and read files from it to get information that is normally available through the metadata service.

默认drive name `/dev/dsik/by-label/config-2`

所谓 config drive的方式就是将metadata信息写入虚拟机的一个特殊配置设备中，虚拟机启动的时候会挂载并读取metadata信息。要想实现这个功能，还需要宿主机和镜像满足一些条件：

1. 宿主机满足条件：
   - 虚拟化方式为以下几种：libvirt,xen,hyper-v,vmware.
   - 对于 libvirt, xen, vmware的虚拟化方式，需要安装genisoimage.并且需要设置mkisofs为 程序的安装目录，如果是跟nova-compute安装在一个目录内就不用了。
   - 如果是 hyper-v的虚拟化方式，需要用mkisofs的命令行指定mkisofs.exe 程序的绝对安装路径，且在hyperv的配置文件中设置qemu_img_cmd 为qemu-img 的命令行安装路径。
2. 镜像满足条件：
   - 安装了cloud-init，且最好是0.7.1+ 的版本。如果是没有安装的话，就需要自己去定制脚本了。
   - 如果使用xen 虚拟化，需要配置 xenapi_disable_agent 为true。

在openstack中，在使用命令行创建虚拟机的时候指定 –config-drive true 即可使用config drive



### cloud-init初始化vm密码

Cloud-Init 是一个用来自动配置虚拟机的初始设置（如主机名，网卡和密钥）的工具（支持Linux,对于Windows系列有一个类似的工具，[cloudbase-init](https://github.com/cloudbase/cloudbase-init)）。它可以在使用模板部署虚拟机时使用，从而达到避免网络冲突的目的。在使用这个工具前，cloud-init 软件包必须在虚拟机上被安装。安装后，Cloud-Init 服务会在系统启动时搜索如何配置系统的信息。您可以使用只运行一次窗口来提供只需要配置一次的设置信息；或在新建虚拟机、编辑虚拟机和编辑模板窗口中输入虚拟机每次启动都需要的配置信息。（以上内容直接引用自red hat 官网）

cloud-init 的配置数据有两种：

1. userdata:文件的形式，常见的如yaml文件，shell scripts，cloud config file。（cloud-init会自动识别以 “#!” 或 “#cloud-config” 开头的文件。）
2. metedata:键值对的形式，常见的如：server name, instance id, display name and other cloud specific details.

cloud-init 的数据源：即cloud-init会从以下几种方式获取userdata或metedata：

1. EC2方式：顾名思义，就是AWS的那一套，其实本质是创建一个http server,虚拟机通过这个http server获取到instance的userdata和metadata。这个http server的IP 通常为169.254.169.254。
2. Config Drive方式：主要是openstack提供的一种方式，本质是把data写入一个特殊的配置设备中，然后在虚拟机启动时，自动挂载并读取 metadata 信息。
3. Alt cloud：通常用于 RHEVm 和 vSphere中获取userdata。
4. vSphere 用于为了向VMWare的vSphere虚拟机中注入userdata。
5. No Cloud 允许用户在没有网络的情况下提供user-data和medatata给instances

https://zhangchenchen.github.io/2017/01/13/openstack-init-instance-password/

因为目前大部分私有云平台基本是KVM虚拟化，后台存储为ceph，所以只选适合这种架构的方案，早期的[inject-password](http://niusmallnan.com/_build/html/_templates/openstack/inject_passwd.html#inject),貌似只支持QCOW2，不支持RAW镜像(没有验证，但在ceph的文档里确实把这个功能的相关配置默认为false,而且推荐使用cloud-init)以及只支持XEN的clear-password ， root-password等API不再考虑。

利用user-data注入, 用法很简单

cloud-init数据源是采用最广泛的EC2模式，EC2的restAPI相关知识[check this](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html#instancedata-add-user-data)。关于metadata server的配置不在描述，主要是Nova，和Neutron 的相关配置（默认配置即可）

###  配置Cloud-Init工具

1. 用户可以根据需要根据用户类型配置登录云服务器的用户权限。使用root帐户登录，需要开启root用户的ssh权限，并开启密码远程登录。
    o 若用户选择注入密码，则通过自己注入的密码进行远程SSH或noVNC登录。
    o 若用户选择注入密钥，则通过自己注入的密钥进行远程SSH登录。

**配置ssh文件**

```bash
vim /etc/ssh/sshd_config
PasswordAuthentication yes
```

**在需要免密码登录的机器上执行下面命令，产生密钥对(controller节点上)**

```bash
ssh-keygen -t rsa -f ~/.ssh/id_rsa -P ''
[root@controller ~]# cat ~/.ssh/id_rsa.pub 
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDCzU9H5oJxgPpWhtXPwETEpFx425UdirDrTBRt6K/kfx7T6AcnuxzIQ6T64TmGhi72FTz7KtfYUtoIzq7specBthD8X8DTA8C9A6nIEwuHgAh12i0OugmdMmzs8g1QvKxZknFOiX32dlWQBZqU1z9V0NmaMRAxvgvYK6YR5GrHFr5QfSr3rPCMOE1qmZ7QQwfxcEFcmwz1xK65LK1VZh1TPzj7DmW97Clr0F/a11a86ekw8uXsMeowSbdGH546y9FbJlopAPZyprS7kEcwUOaj9VcfF+1LxNjjeHuPsfoquYPElgd52GIC0HE+sW5f+9otbZt2GKZUFBeuehC7g05f root@controller
```

**编辑配置文件`/etc/cloud/cloud.cfg`**

> 设置开放root密码远程登录并开启root用户的ssh权限。配置文件中的disable_root字段为1表示为禁用，为0表示不禁用。设置disable_root值为0，ssh_pwauth为1，lock_passwd设置为false，false表示不锁住用户密码。

```bash
[root@centos7-2 ~]# cat /etc/cloud/cloud.cfg
users:
  - name: root
    ssh_authorized_keys:
      - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDCzU9H5oJxgPpWhtXPwETEpFx425UdirDrTBRt6K/kfx7T6AcnuxzIQ6T64TmGhi72FTz7KtfYUtoIzq7specBthD8X8DTA8C9A6nIEwuHgAh12i0OugmdMmzs8g1QvKxZknFOiX32dlWQBZqU1z9V0NmaMRAxvgvYK6YR5GrHFr5QfSr3rPCMOE1qmZ7QQwfxcEFcmwz1xK65LK1VZh1TPzj7DmW97Clr0F/a11a86ekw8uXsMeowSbdGH546y9FbJlopAPZyprS7kEcwUOaj9VcfF+1LxNjjeHuPsfoquYPElgd52GIC0HE+sW5f+9otbZt2GKZUFBeuehC7g05f root@controller

disable_root: 0
ssh_pwauth: 1

#datasource_list: [ 'OpenStack' ]
#datasource:
#  OpenStack:
#    metadata_urls: ["http://169.254.169.254"]
#    timeout: 5
#    max_wait: 60

preserve_hostname: flase
manage_etc_hosts: true

#建议提前配置好网卡配置文件为dhcp获取，否则在私有云上创建的实例可能会导致获取不到IP地址；
#原因是cloud-init中的自动配置网卡文件可能会导致mac地址不一致
network:
  config: disabled

runcmd:
 - [ sh, -c, echo "=========Welcome To OpenStack'=========" > /root/runcmd.log ]

cloud_init_modules:
 - ssh
 - migrator
 - bootcmd
 - write-files
 - growpart
 - resizefs
 - set_hostname
 - update_hostname
 - update_etc_hosts
 - rsyslog
 - users-groups

cloud_config_modules:
 - mounts
 - locale
 - set-passwords
 - yum-add-repo
 - package-update-upgrade-install
 - timezone
 - puppet
 - chef
 - salt-minion
 - mcollective
 - disable-ec2-metadata
 - runcmd
 - ntp-conf

cloud_final_modules:
 - rightscale_userdata
 - scripts-per-once
 - scripts-per-boot
 - scripts-per-instance
 - scripts-user
 - ssh-authkey-fingerprints
 - keys-to-console
 - phone-home
 - final-message
 - power-state-change

system_info:
   distro: centos
   paths:
      cloud_dir: /var/lib/cloud/
      templates_dir: /etc/cloud/templates/
   ssh_svcname: sshd

# vim:syntax=yaml
chpasswd:
   list: |
     root:XXXX@2019  #设置要修改的密码
   expire: False
```

**用微秒生成随机密码的命令**

```bash
date +%N|sha512sum| head -c8
```

如果希望能够修改 instance 的 hostname（默认 instance 每次重启后 cloud-init 都会重新将 hostname 恢复成初始值），将`cloud_init_modules` 列表中下面两项删除或注释掉：

```bash
# - set_hostname
# - update_hostname
```

**(可选配置)在`/etc/cloud/cloud.cfg`文件中自定义网络配置**

在cloud.cfg文件增加该配置之后，cloud-init不会管理/etc/sysconfig/network-scripts/下网络配置，需要自行管理。
 建议提前配置好网卡配置文件为dhcp获取，否则在私有云上创建的实例可能会导致获取不到IP地址；原因是cloud-init中的自动配置网卡文件可能会导致mac地址不一致。

```yaml
network:
  config: disabled
```

**(可选配置)设置root用户密码**

```yaml
chpasswd:
  list: |
    root:123456
  expire: False  
```

**修改以下配置使得镜像创建的云服务器主机名不带.novalocal后缀且主机名称中可以带点号。**
 a. 执行如下命令，修改`__init__.py`文件

```python
# vim /usr/lib/python2.7/site-packages/cloudinit/sources/__init__.py
if toks: 
    toks = str(toks).split('.')
else :
    toks = ["ip-%s" % lhost.replace(".", "-")]
else :
    toks = lhost.split(".novalocal") 
    
if len(toks) > 1: 
    hostname = toks[0]
    # domain = '.'.join(toks[1: ])
else :
    hostname = toks[0]
if fqdn and domain != defdomain: 
    return "%s.%s" % (hostname, domain)
else :
    return hostname
```

> ![img](https:////upload-images.jianshu.io/upload_images/16952149-cb70083515593022.png?imageMogr2/auto-orient/strip|imageView2/2/w/954/format/webp)

**执行如下命令进入cloudinit/sources文件夹。**

```shell
cd /usr/lib/python2.7/site-packages/cloudinit/sources/
#执行如下命令，删除__init__.pyc文件和优化编译后的__init__.pyo文件
rm -rf init.pyc
rm -rf init.pyo
#执行如下命令，清理日志信息。
rm -rf /var/lib/cloud/*
rm -rf /var/log/cloud-init*
```

**执行以下命令编辑Cloud-Init日志输出路径配置文件，设置日志处理方式handlers**

```bash
#建议配置为cloudLogHandler。
vim /etc/cloud/cloud.cfg.d/05_logging.cfg
...
[logger_cloudinit]
level=DEBUG
qualname=cloudinit
handlers=cloudLogHandler
propagate=1
```

> ![img](https:////upload-images.jianshu.io/upload_images/16952149-7af6915bb2bf987e.png?imageMogr2/auto-orient/strip|imageView2/2/w/565/format/webp)

**检查Cloud-Init工具相关配置是否成功**

执行以下命令，无错误发生，说明Cloud-Init配置成功

```bash
[root@centos7 ~]# cloud-init init --local
Cloud-init v. 18.5 running 'init-local' at Wed, 27 May 2020 01:42:00 +0000. Up 1977.24 seconds.
```

**设置完成后关闭虚拟机，准备下一阶段生成镜像**

```bash
history -c
shutdown -h now
```

end

作者：Linux丶晨星
链接：https://www.jianshu.com/p/5c2a63220027
。

### mkfs.fat

mkfs.fat is used to create a FAT filesystem on a device or in an image file.  DEVICE is the special file corresponding to the device (e.g. /dev/sdXX) or the image file (which does not need to exist when the option -C is given).



### instance metedata and config drive

https://zhangchenchen.github.io/page/6/

所谓 config drive的方式就是将metadata信息写入虚拟机的一个特殊配置设备中，虚拟机启动的时候会挂载并读取metadata信息。要想实现这个功能，还需要宿主机和镜像满足一些条件：

1. 宿主机满足条件：
   - 虚拟化方式为以下几种：libvirt,xen,hyper-v,vmware.
   - 对于 libvirt, xen, vmware的虚拟化方式，需要安装genisoimage.并且需要设置mkisofs为 程序的安装目录，如果是跟nova-compute安装在一个目录内就不用了。
   - 如果是 hyper-v的虚拟化方式，需要用mkisofs的命令行指定mkisofs.exe 程序的绝对安装路径，且在hyperv的配置文件中设置qemu_img_cmd 为qemu-img 的命令行安装路径。
2. 镜像满足条件：
   - 安装了cloud-init，且最好是0.7.1+ 的版本。如果是没有安装的话，就需要自己去定制脚本了。
   - 如果使用xen 虚拟化，需要配置 xenapi_disable_agent 为true。

```bash
$ ls -l /dev/disk/by-label/config-2
lrwxrwxrwx. 1 root root 10 Mar  6 10:43 /dev/disk/by-label/config-2 -> ../../vdb1
$ ls -l /dev/vdb
vdb   vdb1
$ ls -l /dev/vdb1
brw-rw----. 1 root disk 253, 17 Mar  6 10:43 /dev/vdb1
$ mkdir /tmp/meta
$ mount  /dev/vdb1  /tmp/meta/
$ cd /tmp/meta/
# metedata
$ ls
ec2  openstack
$ cat ec2/latest/meta-data.json |jq
{
    "reservation-id":"r-8izy3khj",
    "hostname":"kafkatwo-kafka-master-1-8edf9-0.novalocal",
    "security-groups":[
        "default"
    ],
    "public-ipv4":"",
    "ami-manifest-path":"FIXME",
    "instance-type":"4*8*100",
    "instance-id":"i-00000022",
    "local-ipv4":"171.251.48.130",
    "local-hostname":"kafkatwo-kafka-master-1-8edf9-0.novalocal",
    "placement":{
        "availability-zone":"sdswzx"
    },
    "ami-launch-index":0,
    "public-hostname":"kafkatwo-kafka-master-1-8edf9-0.novalocal",
    "public-keys":{
        "0":{
            "openssh-key":"ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCgLoLx3FjwUikfvlnornBOUcSEI0oHpNw2RCrBtSaQv5ZmXOt/q8NH56p5/ITHIt72HGGUELUhyJgFF8lUZE38L3pdXIRA3jj7kIKR1ezYk0YBjJ3UVGWwCRa5y+Px50mASXL1sKPyzrGyL9oUgUXP8bQ5aK0Ko5fdycILI9wjoewGId/rFfXpwnVFywuYAuzcddTWQeTL3EQ8MyDLIT1i1KdN9xgjkI8SWuYTaXnw2+/ZS+qpMFHB7SIBFo5+9AD+t471aUIYOMI42CZmNcDTe/7nRYcVRpCUFDFdva27S1wgbohW7bfcZdvRf7UGoMNltyAo+dY/l/zz+4VurjEn Generated-by-Nova",
            "_name":"0=PaaS"
        }
    },
    "ami-id":"ami-00000001",
    "instance-action":"none",
    "block-device-mapping":{
        "ami":"vda",
        "root":"/dev/vda"
    }
}

```

end

### 169.254.169.254

https://support.huaweicloud.com/vpc_faq/vpc_faq_0084.html

各个云厂商定义的metadata file path都不一样

```bash
curl http://169.254.169.254/meta_file_path
```

华为：

**curl http://169.254.169.254/openstack/latest/meta_data.json**

华三

**curl http://169.254.169.254/2009-04-04/meta-data/instance-id**

### 云主机初始化模式

该计算节点的云主机初始化模式，包括标准模式和专业模式。

- 标准模式：云主机利用虚拟化平台提供的接口进行初始化操作，支持下发主机名、超级用户密码、网络地址等初始化数据。

- 专业模式：云主机利用开机运行的cloud-init工具进行初始化，除支持下发“标准模式”的初始化数据外，还支持下发主机路由等用户提供的数据。

 包括VFAT与ISO两种格式，选择VFAT格式时云主机将挂载一块64MB磁盘，选择ISO格式时云主机将挂载CD驱动器。云主机初始化模式为“专业模式”时可进行此项配置。



### k8s + openstack

#### metadata.go

代码逻辑，从openstack cloud-init挂载的metadata中读取元数据信息

1. 定义从哪读取数据

   configdrive：https://github.com/kubernetes/kubernetes/blob/c1e69551be1a72f0f8db6778f20658199d3a686d/staging/src/k8s.io/legacy-cloud-providers/openstack/metadata.go#L108

   即："/dev/disk/by-label/config-2" 

   metadataservice：https://github.com/kubernetes/kubernetes/blob/c1e69551be1a72f0f8db6778f20658199d3a686d/staging/src/k8s.io/legacy-cloud-providers/openstack/metadata.go#L98:6

   即169.254.169.254

2. 遍历可以读取metadata的方式

   https://github.com/kubernetes/kubernetes/blob/c1e69551be1a72f0f8db6778f20658199d3a686d/staging/src/k8s.io/legacy-cloud-providers/openstack/metadata.go#L179

当虚拟机无法从configdrive和metadataservice获取数据时就好报错。



#### 配置

```bash
[root@bbbbbbbbbbbb-17057 ~]# cat  /etc/kubernetes/kubelet.env
KUBE_LOGTOSTDERR="--logtostderr=true"
KUBE_LOG_LEVEL="--v=2"
KUBELET_ADDRESS="--node-ip=10.200.180.117"
KUBELET_HOSTNAME="--hostname-override=bbbbbbbbbbbb-17057"



KUBELET_ARGS="--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf \
--config=/etc/kubernetes/kubelet-config.yaml \
--kubeconfig=/etc/kubernetes/kubelet.conf \
--pod-infra-container-image=os-harbor-svc.default.svc.cloudos:11443/helm/k8s.gcr.io/pause:3.1 \
--runtime-cgroups=/systemd/system.slice \
   "
KUBELET_NETWORK_PLUGIN="--network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin"
KUBELET_CLOUDPROVIDER="--cloud-provider=openstack --cloud-config=/etc/kubernetes/cloud_config"

PATH=/usr/local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
[root@bbbbbbbbbbbb-17057 ~]# ls /dev/disk/by-label/CONFIG-2/
ls: cannot access /dev/disk/by-label/CONFIG-2/: Not a directory
[root@bbbbbbbbbbbb-17057 ~]# cat /etc/kubernetes/cloud_config
# openstack配置文件，该文文件会被放在K8S集群环境中
[Global]
auth-url="http://172.200.50.100:35357/v3"
username="安防集成平台_kaas"
password="kaas"
region="RegionOne"
tenant-id="30d50b11bda14b209d1ab25603c6b8d6"
tenant-name="安防集成平台"
domain-name="default"

[BlockStorage]
bs-version=v2

```

end
