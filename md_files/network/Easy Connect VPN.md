### Easy Connect VPN

虚拟专用网VPN(virt ual private network)是在公共网络中建立的安全网络连接，这个网络连接和普通意义上的网络连接不同之处在于，它采用了专有的隧道协议，实现了数据的加密和完整性的检验、用户的身份认证，从而保证了信息在传输中不被偷看、篡改、复制，从网络连接的安全性角度来看，就类似于再公共网络中建立了一个专线网络一样，只补过这个专线网络是逻辑上的而不是物理的所以称为虚拟专用网。

原理：

在client上起一个虚拟网络设备完成 到目标网络的隧道？？？

安装后在windows电脑的网络适配上多了一个 “计算机--管理--设备管理器--网络适配器”Sangfor SSL VPN CS Support System VNIC

遇到问题：

easy connect显示 虚拟IP地址 ：未分配--

感觉是它导致我无法通过VPN方式其他资源--即资源组中的资源一直不可用

都没有分配IP怎么允许访问--

论坛答案：

虚拟ip不能获取， 虚拟网卡不正常， 要么就是驱动被拦截了额或是加载项不成功

“网络更改网络适配器”---> 显示easy connect网卡被拔出

正常的easy connect显示“无法识别网络”但easy connect client的连接信息显示有IP。

> cmd --> ipconfig/all 也能显示IP

检测工具也提示：获取虚拟网卡信息失败

https://www.jianshu.com/p/30a77db87182

解决：

启动easy connect后，easy connect的虚拟网卡显示，网络电缆被拔出（就是启动失败了），右键点击诊断，查看详细信息，又是显示“windows无法自动将IP协议堆栈绑定到网络适配器”。

好办勒，右键网络适配器，点击属性，把Npcap Package Driver协议勾选掉就行了---



OpenVPN安装后生成新的网络设备：

 TAP-Windows Adapter V9 for OpenVPN Connect

#### windos网络协议冲突

错误现象：wifi无法连接，无线网络不能使用--

错误信息：windows无法自动将IP协议堆栈绑定到网络适配器

错误原因：安装了gns3给网络适配器上加了许多网络协议冲突了

解决方法：更改适配器设置–网络连接，查看以太网属性（右键属性，显示网络协议）
除了Microsoft网络客户端、Microsoft网络的文件和打印机共享、IPV4、IPV6， 以及后面两个链路层选项，其他的都取消勾选。

主要是勾选掉Npcap两个协议。

nonono，勾选掉Npcap，wireshark 无法抓包了，因为接口上需要安装Npcap协议才能捕获接口信息。

#### Npcap

Npcap Packet Driver （NPCAP）
是Nmap 项目的 Windows 数据包嗅探（和发送）库。它基于已停产的 WinPcap 库，但提高了速度、可移植性、安全性和效率。
Npcap 能够使用 Windows 筛选平台 （WFP） 来嗅探环回数据包,环回数据包注入：Npcap 还能够使用 Winsock 内核 （WSK） 技术发送环回数据包