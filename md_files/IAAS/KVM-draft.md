## KVM-draft

KVM 全称是 Kernel-Based Virtual Machine。也就是说 KVM 是基于 Linux 内核实现的。 KVM有一个内核模块叫 kvm.ko，只用于管理虚拟 CPU 和内存。



KVM (for Kernel-based Virtual Machine) is a full virtualization solution for Linux on x86 hardware containing virtualization extensions (Intel VT or AMD-V). It consists of a loadable kernel module, kvm.ko, that provides the core virtualization infrastructure and a processor specific module, kvm-intel.ko or kvm-amd.ko.

### 总结模块思路

1. libvirt and libvirt-client
2. xml配置
3. bridge + vlan



### Libvirt

Libvirt aims to support building and executing on multiple host OS platforms, as well as working with multiple hypervisors. This document outlines which platforms are targeted for each of these areas.

libvirt是一套免费、开源的支持Linux下主流虚拟化管理程序的C函数库，其旨在为包括KVM在内的各种虚拟化管理程序提供一套方便、可靠的编程接口。
当前主流Linux平台上默认的虚拟化管理工具virt-manager,virsh等都是基于libvirt开发。

![libvirt-arc](D:\temp_files\download\libvirt-arc.jpg)

```bash
root@11-10:~# /usr/sbin/libvirtd  -h
/usr/sbin/libvirtd: invalid option -- 'h'

Usage:
  /usr/sbin/libvirtd [options]

Options:
  -v | --verbose         Verbose messages.
  -d | --daemon          Run as a daemon & write PID file.
  -l | --listen          Listen for TCP/IP connections.
  -t | --timeout <secs>  Exit after timeout period.
  -f | --config <file>   Configuration file.
     | --version         Display version information.
  -p | --pid-file <file> Change name of PID file.

libvirt management daemon:

  Default paths:

    Configuration file (unless overridden by -f):
      /etc/libvirt/libvirtd.conf

    Sockets:
      /var/run/libvirt/libvirt-sock
      /var/run/libvirt/libvirt-sock-ro

    TLS:
      CA certificate:     /etc/pki/CA/caert.pem
      Server certificate: /etc/pki/libvirt/servercert.pem
      Server private key: /etc/pki/libvirt/private/serverkey.pem

    PID file (unless overridden by -p):
      /var/run/libvirtd.pid

```

end

#### Hypervisor driver

The hypervisor drivers currently supported by libvirt are:

- [LXC](https://libvirt.org/drvlxc.html) - Linux Containers
- [OpenVZ](https://libvirt.org/drvopenvz.html)
- [QEMU](https://libvirt.org/drvqemu.html)
- [Test](https://libvirt.org/drvtest.html) - Used for testing
- [VirtualBox](https://libvirt.org/drvvbox.html)
- [VMware ESX](https://libvirt.org/drvesx.html)
- [VMware Workstation/Player](https://libvirt.org/drvvmware.html)
- [Xen](https://libvirt.org/drvxen.html)
- [Microsoft Hyper-V](https://libvirt.org/drvhyperv.html)
- [Virtuozzo](https://libvirt.org/drvvirtuozzo.html)
- [Bhyve](https://libvirt.org/drvbhyve.html) - The BSD Hypervisor
- [Cloud Hypervisor](https://libvirt.org/drvch.html)

#### Libvirt XML Format

Objects in the libvirt API are configured using XML documents to allow for ease of extension in future releases.

##### Domains：guest xml

配置都在`/etc/libvirt/qemu/`，每个虚机一个xml文件。

The final set of XML elements are all used to describe devices provided to the guest domain. 

To specify the VLAN tag configuration settings, use a management tool to make the following changes to the domain XML:

如下配置，可以指定guest(VM)的Vlan ID，但是底层是实现还是靠KVM.

```xml
  ...
  <devices>
    <interface type='bridge'>
      <vlan>
        <tag id='42'/>
      </vlan>
      <source bridge='ovsbr0'/>
      <virtualport type='openvswitch'>
        <parameters interfaceid='09b11c53-8b5c-4eeb-8f00-d84eaa0aaa4f'/>
      </virtualport>
    </interface>
  <devices>
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

```

The vhost attribute can override the default vhost device path (/dev/vhost-net) for devices with virtio model. The tap attribute overrides the tun/tap device path (default: /dev/net/tun) for network and bridge interfaces. This does not work in session mode

###### stetting vlan tag

```xml
<devices>
  <interface type='bridge'>
    <vlan>
      <tag id='42'/>
    </vlan>
    <source bridge='ovsbr0'/>
    <virtualport type='openvswitch'>
      <parameters interfaceid='09b11c53-8b5c-4eeb-8f00-d84eaa0aaa4f'/>
    </virtualport>
  </interface>
  <interface type='bridge'>
    <vlan trunk='yes'>
      <tag id='42'/>
      <tag id='123' nativeMode='untagged'/>
    </vlan>
    ...
  </interface>
</devices>
```

Network connections that support guest-transparent VLAN tagging include 1) type='bridge' interfaces connected to an Open vSwitch bridge **Since 0.10.0** .

 For VLAN trunking of multiple tags (which is supported only on Open vSwitch connections), multiple <tag> subelements can be specified, which implies that the user wants to do VLAN trunking on the interface for all the specified tags. In the case that VLAN trunking of a single tag is desired, the optional attribute trunk='yes' can be added to the toplevel <vlan> element to differentiate trunking of a single tag from normal tagging.

For network connections using Open vSwitch it is also possible to configure 'native-tagged' and 'native-untagged' VLAN modes **Since 1.1.0.** This is done with the optional nativeMode attribute on the <tag> subelement: nativeMode may be set to 'tagged' or 'untagged'. The id attribute of the <tag> subelement containing nativeMode sets which VLAN is considered to be the "native" VLAN for this interface, and the nativeMode attribute determines whether or not traffic for that VLAN will be tagged.

###### interface type

type有很多：

* hostdev：PCI network

  https://libvirt.org/formatdomain.html#pci-passthrough

  a type='hostdev' interface can have an optional driver sub-element with a name attribute set to "vfio". 

* vdpa：

  A vDPA network device can be used to provide wire speed network performance within a domain. 

  This creates a vDPA char device (e.g. /dev/vhost-vdpa-0) that can be used to assign the device to a libvirt domain. 

  https://libvirt.org/formatdomain.html#vdpa-devices

* bridge

* 等等很多

##### Networks

这个是干嘛的

https://linuxconfig.org/how-to-use-bridged-networking-with-libvirt-and-kvm

使用例子

###### The “default” network

When **libvirt** is in use and the **libvirtd** daemon is running, a default network is created. We can verify that this network exists by using the `virsh` utility, which on the majority of Linux distribution usually comes with the `libvirt-client` package. To invoke the utility so that it displays all the available virtual networks, we should include the `net-list` subcommand:

```bash
$ sudo virsh net-list --all
```

In the example above we used the `--all` option to make sure also the **inactive** networks are included in the result, which should normally correspond to the one displayed below:

```bash
Name      State    Autostart   Persistent
--------------------------------------------
 default   active   yes         yes
```

To obtain detailed information about the network, and eventually modify it, we can invoke virsh with the `edit` subcommand instead, providing the network name as argument:

```bash
$ sudo virsh net-edit default
```

A temporary file containing the **xml** network definition will be opened in our favorite text editor. In this case the result is the following:

```bash
<network>
  <name>default</name>
  <uuid>168f6909-715c-4333-a34b-f74584d26328</uuid>
  <forward mode='nat'/>
  <bridge name='virbr0' stp='on' delay='0'/>
  <mac address='52:54:00:48:3f:0c'/>
  <ip address='192.168.122.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.122.2' end='192.168.122.254'/>
    </dhcp>
  </ip>
</network>
```

As we can see, the default network is based on the use of the `virbr0` virtual bridge, and uses **NAT** based connectivity to connect the virtual machines which are part of the network to the outside world. We can verify that the bridge exists using the `ip` command:

```bash
$ ip link show type bridge
```

In our case the command above returns the following output:

```bash
5: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default qlen 1000
    link/ether 52:54:00:48:3f:0c brd ff:ff:ff:ff:ff:ff
```

To show the interfaces which are part of the bridge, we can use the `ip` command and query only for interfaces which have the `virbr0` bridge as master:

```bash
$ ip link show master virbr0
```

The result of running the command is:

```bash
6: virbr0-nic: <BROADCAST,MULTICAST> mtu 1500 qdisc fq_codel master virbr0 state DOWN mode DEFAULT group default qlen 1000
    link/ether 52:54:00:48:3f:0c brd ff:ff:ff:ff:ff:ff
```

As we can see, there is only one interface currently attached to the bridge, `virbr0-nic`. The `virbr0-nic` interface is a virtual ethernet interface: it is created and added to the bridge automatically, and its purpose is just to provide a stable **MAC** address (52:54:00:48:3f:0c in this case) for the bridge.

Other virtual interfaces will be added to the bridge when we create and launch virtual machines. For the sake of this tutorial I created and launched a Debian (Buster) virtual machine; if we re-launch the command we used above to display the bridge slave interfaces, we can see a new one was added, `vnet0`:

```bash
$ ip link show master virbr0
6: virbr0-nic: <BROADCAST,MULTICAST> mtu 1500 qdisc fq_codel master virbr0 state DOWN mode DEFAULT group default qlen 1000
    link/ether 52:54:00:48:3f:0c brd ff:ff:ff:ff:ff:ff
7: vnet0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master virbr0 state UNKNOWN mode DEFAULT group default qlen 1000
    link/ether fe:54:00:e2:fe:7b brd ff:ff:ff:ff:ff:ff
```

No physical interfaces should ever be added to the `virbr0` bridge, since it uses **NAT** to provide connectivity.

###### Use bridged networking for virtual machines

The default network provides a very straightforward way to achieve connectivity when creating virtual machines: everything is “ready” and works out of the box. Sometimes, however, we want to achieve a **full bridgining** connection, where the guest devices are connected to the host **LAN**, without using **NAT**, we should create a new bridge and share one of the host physical ethernet interfaces. Let’s see how to do this step by step.

###### Creating a new bridge

To create a new bridge, we can still use the `ip` command. Let’s say we want to name this bridge `br0`; we would run the following command:

```bash
$ sudo ip link add br0 type bridge
```

To verify the bridge is created we do as before:

```bash
$ sudo ip link show type bridge
5: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default qlen 1000
    link/ether 52:54:00:48:3f:0c brd ff:ff:ff:ff:ff:ff
8: br0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 26:d2:80:7c:55:dd brd ff:ff:ff:ff:ff:ff
```

As expected, the new bridge, `br0` was created and is now included in the output of the command above. Now that the new bridge is created, we can proceed and add the physical interface to it.

###### Adding a physical ethernet interface to the bridge

In this step we will add a host physical interface to the bridge. Notice that you can’t use your main ethernet interface in this case, since as soon as it is added to the bridge you would loose connectivity, since it will loose its IP address. In this case we will use an additional interface, `enp0s29u1u1`: this is an interface provided by an ethernet to usb adapter attached to my machine.

First we make sure the interface state is UP:

```bash
$ sudo ip link set enp0s29u1u1 up
```

To add the interface to bridge, the command to run is the following:

```bash
$ sudo ip link set enp0s29u1u1 master br0
```

To verify the interface was added to the bridge, instead:

```bash
$ sudo ip link show master br0
3: enp0s29u1u1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master br0 state UP mode DEFAULT group default qlen 1000
    link/ether 18:a6:f7:0e:06:64 brd ff:ff:ff:ff:ff:ff
```

###### Assigning a static IP address to the bridge

At this point we can assign a static IP address to the bridge. Let’s say we want to use `192.168.0.90/24`; we would run:

```bash
$ sudo ip address add dev br0 192.168.0.90/24
```

To very that the address was added to the interface, we run:

```bash
$ ip addr show br0
9: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 26:d2:80:7c:55:dd brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.90/24 scope global br0
       valid_lft forever preferred_lft forever
   [...]
```

###### Making the configuration persistent

Our bridge configuration is ready, however, as it is, it will not survive a machine reboot. To make our configuration persistent we must edit some configuration files, depending on the distribution we use.

###### Debian and derivatives

On the Debian family of distributions we must be sure that the `bridge-utils` package is installed:

```bash
$ sudo apt-get install bridge-utils
```

Once the package is installed, we should modify the content of the `/etc/network/interfaces` file:

```bash
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

# The loopback network interface
auto lo
iface lo inet loopback

# Specify that the physical interface that should be connected to the bridge
# should be configured manually, to avoid conflicts with NetworkManager
iface enp0s29u1u1 inet manual

# The br0 bridge settings
auto br0
iface br0 inet static
    bridge_ports enp0s29u1u1
        address 192.168.0.90
        broadcast 192.168.0.255
        netmask 255.255.255.0
        gateway 192.168.0.1
```

###### Red Hat family of distributions

On the Red Hat family of distributions, Fedora included, we must manipulate network scripts inside the `/etc/sysconfig/network-scripts` directory. If we want the bridge **not** to be managed by NetworkManager, or we are using an older distribution with an older version of NetworkManager not capable of managing network switches, we need to install the `network-scripts` package:

```bash
$ sudo dnf install network-scripts
```

Once the package is installed, we need to create the file which will configure the `br0` bridge: `/etc/sysconfig/network-scripts/ifcfg-br0`. Inside the file we place the following content:

```bash
DEVICE=br0
TYPE=Bridge
BOOTPROTO=none
IPADDR=192.168.0.90
GATEWAY=192.168.0.1
NETMASK=255.255.255.0
ONBOOT=yes
DELAY=0
NM_CONTROLLED=0
```

Than, we modify or create the file used to configure the physical interface we will connect to the bridge, in this case `/etc/sysconfig/network-scripts/ifcfg-enp0s29u1u1`:

```bash
TYPE=ethernet
BOOTPROTO=none
NAME=enp0s29u1u1
DEVICE=enp0s29u1u1
ONBOOT=yes
BRIDGE=br0
DELAY=0
NM_CONTROLLED=0
```

With our configurations ready, we can start the `network` service, and enable it at boot:

```bash
$ sudo systemctl enable --now network
```

###### Disabling netfilter for the bridge

To allow all traffic to be forwarded to the bridge, and therefore to the virtual machines connected to it, we need to disable netfilter. This is necessary, for example, for DNS resolution to work in the guest machines attached to the bridge. To do this we can create a file with the `.conf` extension inside the `/etc/sysctl.d` directory, let’s call it `99-netfilter-bridge.conf`. Inside of it we write the following content:

```bash
net.bridge.bridge-nf-call-ip6tables = 0
net.bridge.bridge-nf-call-iptables = 0
net.bridge.bridge-nf-call-arptables = 0
```

To load the settings written in the file, fist we ensure that the `br_netfilter` module is loaded:

```bash
$ sudo modprobe br_netfilter
```

To load the module automatically at boot, let’s create the `/etc/modules-load.d/br_netfilter.conf` file: it should contain only the name of the module itself:

```
br_netfilter
```

Once the module is loaded, to load the settings we stored in the `99-netfilter-bridge.conf` file, we can run:

```bash
$ sudo sysctl -p /etc/sysctl.d/99-netfilter-bridge.conf
```

###### Creating a new virtual network

At this point we should define a new “network” to be used by our virtual machines. We open a file with our favorite editor and paste the following content inside of it, than save it as `bridged-network.xml`:

```bash
<network>
    <name>bridged-network</name>
    <forward mode="bridge" />
    <bridge name="br0" />
</network>
```

Once the file is ready we pass its position as argument to the `net-define` `virsh` subcommand:

```bash
$ sudo virsh net-define bridged-network.xml
```

To activate the new network and make so that it is auto-started, we should run:

```bash
$ sudo virsh net-start bridged-network
$ sudo virsh net-autostart bridged-network
```

We can verify the network has been activated by running the `virsh net-list`
command, again:

```bash
$ sudo virsh net-list
Name              State    Autostart   Persistent
----------------------------------------------------
bridged-network   active   yes         yes
default           active   yes         yes
```

We can now select the network by name when using the `--network` option:

```bash
$ sudo virt-install \
  --vcpus=1 \
  --memory=1024 \
  --cdrom=debian-10.8.0-amd64-DVD-1.iso \
  --disk size=7 \
  --os-variant=debian10 \
  --network network=bridged-network
```

end

###### Conclusions

In this tutorial we saw how to create a virtual bridge on linux and connect a physical ethernet interface to it in order to create a new “network” to be used in virtual machines managed with libvirt. When using the latter a default network is provided for convenience: it provides connectivity by using NAT. When a using a bridged network as the one we configure in this tutorial, we will improve performance and make the virtual machines part of the same subnet of the host.



### virsh

--

```bash
# /etc/libvirt/qemu/
$ ls -ld /etc/libvirt/qemu/*
-rw------- 1 root root 4396 Mar  2  2021 /etc/libvirt/qemu/AI_yshj_caozhaohui.xml
drwxr-xr-x 2 root root 4096 Nov 10  2018 /etc/libvirt/qemu/autostart.bak
drwxr-xr-x 2 root root 4096 Mar 31 11:30 /etc/libvirt/qemu/domainProfile
-rw------- 1 root root 4560 Mar  2  2021 /etc/libvirt/qemu/HDP-zyf.xml
drwxr-xr-x 3 root root 4096 Nov 10  2018 /etc/libvirt/qemu/networks
drwxr-xr-x 2 root root 4096 Mar 31 11:30 /etc/libvirt/qemu/profile
-rw------- 1 root root 4617 Mar 31 11:30 /etc/libvirt/qemu/test-vs.xml
drwxr-xr-x 2 root root 4096 Mar 31 11:36 /etc/libvirt/qemu/vswitch

```

end

#### network xml

https://libvirt.org/formatnetwork.html

To specify the VLAN tag configuration settings, use a management tool to make the following changes to the domain XML:

```xml
  ...
  <devices>
    <interface type='bridge'>
      <vlan>
        <tag id='42'/>
      </vlan>
      <source bridge='ovsbr0'/>
      <virtualport type='openvswitch'>
        <parameters interfaceid='09b11c53-8b5c-4eeb-8f00-d84eaa0aaa4f'/>
      </virtualport>
    </interface>
  <devices>
  ...
```

**Figure 20.52. Setting VLAN tag (on supported network types only)**

Setting VLAN tag (on supported network types only) 

```xml
<network>
  <name>ovs-net</name>
  <forward mode='bridge'/>
  <bridge name='ovsbr0'/>
  <virtualport type='openvswitch'>
    <parameters interfaceid='09b11c53-8b5c-4eeb-8f00-d84eaa0aaa4f'/>
  </virtualport>
  <vlan trunk='yes'>
    <tag id='42' nativeMode='untagged'/>
    <tag id='47'/>
  </vlan>
  <portgroup name='dontpanic'>
    <vlan>
      <tag id='42'/>
    </vlan>
  </portgroup>
</network>
```

If (and only if) the network connection used by the guest supports VLAN tagging transparent to the guest, an optional `<vlan>` element can specify one or more VLAN tags to apply to the guest's network traffic **Since 0.10.0**. Network connections that support guest-transparent VLAN tagging include 1) type='bridge' interfaces connected to an Open vSwitch bridge **Since 0.10.0**, 2) SRIOV Virtual Functions (VF) used via type='hostdev' (direct device assignment) **Since 0.10.0**, and 3) SRIOV VFs used via type='direct' with mode='passthrough' (macvtap "passthru" mode) **Since 1.3.5**. All other connection types, including standard linux bridges and libvirt's own virtual networks, **do not** support it. 802.1Qbh (vn-link) and 802.1Qbg (VEPA) switches provide their own way (outside of libvirt) to tag guest traffic onto a specific VLAN. Each tag is given in a separate `<tag>` subelement of `<vlan>` (for example: `<tag id='42'/>`). For VLAN trunking of multiple tags (which is supported only on Open vSwitch connections), multiple `<tag>` subelements can be specified, which implies that the user wants to do VLAN trunking on the interface for all the specified tags. In the case that VLAN trunking of a single tag is desired, the optional attribute `trunk='yes'` can be added to the toplevel `<vlan>` element to differentiate trunking of a single tag from normal tagging.

For network connections using Open vSwitch it is also possible to configure 'native-tagged' and 'native-untagged' VLAN modes **Since 1.1.0.** This is done with the optional `nativeMode` attribute on the `<tag>` subelement: `nativeMode` may be set to 'tagged' or 'untagged'. The `id` attribute of the `<tag>` subelement containing `nativeMode` sets which VLAN is considered to be the "native" VLAN for this interface, and the `nativeMode` attribute determines whether or not traffic for that VLAN will be tagged.

`<vlan>` elements can also be specified in a `<portgroup>` element, as well as directly in a domain's `<interface>` element. In the case that a vlan tag is specified in multiple locations, the setting in `<interface>` takes precedence, followed by the setting in the `<portgroup>` selected by the interface config. The `<vlan>` in `<network>` will be selected only if none is given in `<portgroup>` or `<interface>`.

### Bridge and VLAN

在使用Vlan之前，需要加载`8021q`
`modprobe 8021q`

如果系统已经加载可以不用执行以上步骤
`lsmod | grep 8021q`

Virtualization, Cloud, OpenStack, and Docker. These technologies are getting increasingly important and popular. But behind them, there are two indispensable features: Bridge and VLAN.

A bridge is a way to connect two Ethernet segments together in a protocol independent way. Packets are forwarded based on Ethernet address, rather than IP address (as a router would do). Since forwarding is done at Layer 2, all protocols can go transparently through the bridge. The Linux bridge is actually a virtual switch and widely used with KVM/QEMU hypervisor, namespaces, etc.

VLAN is another very important function in virtualization. It allows network administrators to group hosts together even if the hosts are not on the same physical network switch. This can greatly simplify network design and deployment. Also, it can separate hosts/guests under the same switch/bridge into different subnets.

These two important features used to be considered as two completely distinct features and we needed to configure complex topologies to combine them together. In this blog, we will show the new bridge feature, **VLAN filter**, which hopefully makes life easier.

#### vlan filtering

https://developers.redhat.com/blog/2017/09/14/vlan-filter-support-on-bridge#bridge_and_vlan

#### vlan manager

https://github.com/nesanton/kvmvlan

Let's refer to the following diagram as an example setup. The shape in the middle is our virtual switch or vswitch. Rectangles attached to it are ports of this switch. eth0 is the only physical port. It can also be a bonding of several links

There are two VMs. First one will have one adapter which it is likely to detect as eth0 inside its OS. The other will have two interfaces supposedly detected as eth0 and eth1 inside its OS.

Names vethX were made up to refer to switch ports allocated to VMs from the host. It is important to understand the difference between vethX on the host and ethX inside a VM. These names are actually coming from VMs' xml configs.

br0 serves two functions: on one hand it is a name for the switch. On the other hand it is a port thru which the host itself is connected to outside. Host's ip settings can be set here and, yes, it can also have its PVID and extra VIDs

All the physical (eth0) and virtual (vethX) ports are attached to the bridge br0 on the configuration files level. Former thru /etc/sysconfig/network-scripts/ files and latter thru VM's XML configuration found in /etc/libvirt/qemu/. So there is no need in specifying the bridge name in commands "bridge vlan add ...".

```bash
                                ____________________
                          _____|                    |         
                         |     | pvid=1             |
 Physical switch --------|eth0 | vids = 100,200,300 |
                         |_____|                    |_____
                               |       pvid=100     |veth0|------VM1(eth0)
                               |       vids=200,300 |_____|    
                               |                    |_____
                               |       pvid=1       |veth1|------VM2(eth0)
                               |       vids=100,200 |_____|    
                               |                    |_____
                               |       pvid=300     |veth2|------VM2(eth1)
                               |       vids= N/A    |_____|    
                               |                    | 
                               |                    |
                               |   pvid=1           | 
                               |___vids=200_________|
                                     | br0  |
                                     |______|
                                         |
                                         |------ to KVM host
```

end

0.0 

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

### host gw and vm gw

这个host是cas纳管的机器，这个host也是vm？俄罗斯套娃？！

gateway是host的gw，虚拟机使用某个bridge时，vm gw is bridge ip.

```bash
## host routing-table
root@11-10:~# ip r
default via 1.1.0.254 dev vswitch0  metric 100
1.1.0.0/16 dev vswitch0  proto kernel  scope link  src 1.1.11.10
2.2.2.0/24 dev test_vs  proto kernel  scope link  src 2.2.2.2

```

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vm-gw-with-linux-bridge.jpg)

```bash
# cat /etc/network/interfaces
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto vswitch0
iface vswitch0 inet static
        address 1.1.11.10
        netmask 255.255.0.0
        network 1.1.0.0
        broadcast 1.1.255.255
        gateway 1.1.0.254
auto eth0
iface eth0 inet static
        address 0.0.0.0
        netmask 0.0.0.0
auto eth1
iface eth1 inet static
        address 0.0.0.0
        netmask 0.0.0.0
auto eth3
iface eth3 inet static
        address 0.0.0.0
        netmask 0.0.0.0
auto eth5
iface eth5 inet static
        address 0.0.0.0
        netmask 0.0.0.0
auto eth4
iface eth4 inet static
        address 0.0.0.0
        netmask 0.0.0.0
auto dataswitch
iface dataswitch inet static
        address 0.0.0.0
        netmask 0.0.0.0
auto eth2
iface eth2 inet static
        address 0.0.0.0
        netmask 0.0.0.0
auto test_vs
iface test_vs inet static
        address 2.2.2.2
        netmask 255.255.255.0

root@11-10:~#


```

end

### Qemu

Qemu是一个模拟器，它向Guest OS模拟CPU和其他硬件，Guest OS认为自己和硬件直接打交道，其实是同Qemu模拟出来的硬件打交道，Qemu将这些指令转译给真正的硬件。

由于所有的指令都要从Qemu里面过一手，因而性能较差。

#### qemu-kvm

Qemu将KVM整合进来，通过ioctl调用/dev/kvm接口，将有关CPU指令的部分交由内核模块来做。kvm负责cpu虚拟化+内存虚拟化，实现了cpu和内存的虚拟化，但kvm不能模拟其他设备。qemu模拟IO设备（网卡，磁盘等），kvm加上qemu之后就能实现真正意义上服务器虚拟化。因为用到了上面两个东西，所以称之为qemu-kvm。

Qemu模拟其他的硬件，如Network, Disk，同样会影响这些设备的性能，于是又产生了pass through半虚拟化设备virtio_blk, virtio_net，提高设备性能。



### 虚拟机流量转发

https://raymii.org/s/tutorials/KVM_with_bonding_and_VLAN_tagging_setup_on_Ubuntu_12.04.html

转发模式：虚拟交换机的转发模式，包括VEB和VXLAN(SDN)，默认为VEB。

- VEB：Virtual Ethernet Bridge，虚拟以太网桥。在该模式下，虚拟机与虚拟机之间的流量通过纯软件的方式进行转发。

  VEB是指vSwitch相对全面的网络转发功能，此模式下，同一VLAN的数据直接通过vSwitch转发，不需要通过外部网络。如果是不同的VLAN间通信，或者同一虚拟机属于VLAN但是在不同的vSwitch上，则需要通过外部网路。这样做的好处是可以减少外部网路的流量，减轻网络维护的难度，内部交换的速度也更快,只与CPU性能和内存带宽总线有关，而且对现有网络的兼容性较好。但是缺点是需要耗费更多的CPU，转发性能受制于CPU和网卡IO架构，也缺乏对流量的监控和安全控制。

- VXLAN(SDN)：部署了SDN控制器和云计算管理平台的VXLAN解决方案对应的虚拟交换机转发模式。

- VLAN ID：虚拟交换机上连接主机网络模块的内核端口的VLAN ID。

1.2 VEPA模式：

VEPA是对VEB的一种修改，无论二层三层流量，统一要先发往外部网络，由物理交换机再回发到vSwitch。这种方式简化了服务器的vSwitch功能，使得内外网络相关联，服务器内部网络相当于外部网络的扩展和延伸。对于传统的交换机来说，从一个端口收到的报文是不能再从此端口发出的，这时候为了支持VEPA，交换机必须做一些修改，允许数据绕回，这种方式称为RR模式。如果是组播/广播流量，先绕回vSwitch，在服务器内部再进行数据的复制。VEPA模式可以借助于外部网络，很方便的对VM流量进行监控和安全控制管理，也减少了CPU的消耗，但是却额外增加了外部网络的流量和延迟。VEPA模式中用到了VSI（虚拟站点接口），可以看成是虚拟机网卡在交换机上的一个逻辑接口。VSI涉及到如下几个术语：

VSI管理ID：指VSI管理者的ID号，用于识别出可以访问并获取VSI类型的数据库，用IPv6地址来标识。

VSI类型ID：用来描述一个VSI的类型，对每个VSI管理者ID来说，只有唯一一个VSI类型ID。

VSI类型版本：一个整数标识符，允许VSI管理者数据库可以包含一个给定的VSI类型的多个版本。

1.3 多通道模式：

多通道模式是将交换机端口或网卡划分为多个逻辑通道,称为S通道，并且各通道间逻辑隔离。每个逻辑通道可由用户根据需要定义成VEB或者VEPA。每个逻辑通道作为一个独立的到外部网络的通道进行处理。多通道借用了QINQ标准，另外增加了一个VLAN Tag，用channel ID和VLAN ID来对不同的通道进行标识，分别称为S-channel ID和S-VLAN ID，外层的Tag只会在服务器和接入交换机之间存在，而在VEPA模式下则不需要另外添加VLAN Tag。可以说，多通道模式是VEB和VEPA的结合。在这种模式下，组播/广播流量直接在物理交换机上进行数据的复制，然后通过各个通道再发给虚拟机。

### 引用

1. https://blog.51cto.com/changfei/1672147