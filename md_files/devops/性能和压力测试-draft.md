# 性能和压力测试-draft

## 性能测试

phoronix.com 是业内一个知名的网站，其经常发布硬件性能测评以及 Linux 系统相关的性能测评， Phoronix Test Suite 为该网站旗下的 linux 平台测试套件 , Phoronix 测试套件遵循GNU GPLv3协议。

跨平台的基准测试工具。

### Phoronix Test Suite 

#### 架构

Phoronix Test Suite 性能测试套件由三部分构成：

* PHP 环境
* phoronix-test-suite 主程序
* test-profiles：N多个pts套件，不仅是cpu、内存和IO，甚至由专门软件的测试套件比如redis和mysql

整体使用流程，运行`phoronix-test-suite`命令开始下载各种测试项，测试套件，并在本机环境下编译安装，这样就能够达到跨平台的效果，比如你可以测试arm架构的cpu和x86架构的cpu.当然不仅仅测试cpu，gpu等性能，你还能够测试服务器，数据库等。

#### online-install

```bash
yum install -y gcc gcc-c++ libstdc++-devel
yum install -y php-cli php-gd php-xml
yum install -y unzip patch bzip2
yum install -y tcl autoconf popt-devel libaio-devel qt4-devel gmp-devel 

## 编译安装phoronix-test-suite
tar -zxvf phoronix-test-suite-7.6.0m1.tar.gz  
cd phoronix-test-suite 
./install-sh
## 运行此命令列出可用组件并自动生成installed-test测试目录（比较慢），需要联网
## 获取phoronix-test-suite测试套件目录
phoronix-test-suite list-tests

```

end

#### offline-install

##### 获取安装依赖

主要包含php环境

```bash
# 提前下载所需rpm包，然后离线安装
# php-common-7.2.10、php-xml-7.2.10、php-opcache-7.2.10、php-json-7.2.10、php-devel-7.2.10、php-cli-7.2.10、php-7.2.10、openssl-devel-1.1.1、openssl-1.1.1、krb5-devel-1.18.2、e2fsprogs-devel-1.45.6、keyutils-libs-devel-1.6.3、libverto-devel-0.3.1-2
$ rpm -ivh ./*.rpm
```

end

##### 编译安装

```bash
## 编译安装phoronix-test-suite
tar -zxvf phoronix-test-suite-7.6.0m1.tar.gz  
cd phoronix-test-suite 
./install-sh
```

end

##### 获取测试套件目录

这一步如果在联网环境下`phoronix-test-suite list-tests`命令可以直接生成目录

```bash
# 此zip包包含了所有的测试用例，将其上传到linux任意目录下解压
unzip test-profiles-master.zip 
# 移动解压后pts目录到phoronix-test-suite/目录下即给默认为空的pts目录加上所有组件
cd test-profiles-master
mv pts /var/lib/phoronix-test-suite/installed-tests/ 
```

end

##### 获取测试套件实例

pts/<test_name>/目录下都用`downloads.xml`，如下所示，记录了test case需要用到的tgz的下载路径

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/phoronix-test-suite.png)

离线环境下，应该预先下载好要运行的测试套件的tgz包，然后上传到对应的

`/var/lib/phoronix-test-suite/installed-tests/pts/<test_case_name>/ `目录中

此外，当test case有多个版本时，使用版本号最大的版本。

##### 实际离线运行栗子

` /usr/bin/phoronix-test-suite run pts/c-ray`或者` /usr/bin/phoronix-test-suite benchmark c-ray`

```bash
# 离线环境下运行c-ray套件测试cpu浮点计算
$ /usr/bin/phoronix-test-suite run pts/c-ray
No Internet Connectivity After Wait
No Internet Connectivity After Wait

	## 提示pts/c-ray-1.2.0 还没有安装，输入y开始安装
	## 此时如果可以连接外网，则直接开始下载
	## 离线环境，需要提前将c-ray-1.1.tar.gz
	## 上传到/var/lib/phoronix-test-suite/installed-tests/pts/c-ray-1.2.0目录中
    [PROBLEM] pts/c-ray-1.2.0 is not installed.
    Would you like to stop and install these tests now (Y/n): y
    Evaluating External Test Dependencies .................................................................................................

Phoronix Test Suite v10.8.3

    To Install:    pts/c-ray-1.2.0

    Determining File Requirements .........................................................................................................
    Searching Download Caches .............................................................................................................

    1 Test To Install
        6MB Of Disk Space Is Needed
        3 Seconds Estimated Install Time

    pts/c-ray-1.2.0:
        Test Installation 1 of 1
        1 File Needed [0.22 MB]
        File Found: c-ray-1.1.tar.gz                                                                                               [0.22MB]
        Approximate Install Size: 6.0 MB
        Estimated Install Time: 3 Secon
```

end

## I/O测试

### fio

得到各种场景下IOPS

```bash
# 8k 随机读
fio -filename= /test/fio/test_randread -direct=1 -iodepth 1 -thread -rw=randread -ioengine=libaio -bs=8k -size=4G -numjobs=10 -runtime=60 -group_reporting -name=mytest

# 已文件形式记录日志
fio -filename=/test/fio/iotest -direct=1 -iodepth 64 -thread -rw=randwrite -ioengine=libaio -bs=4k -size=20G -numjobs=64 -runtime=60 -group_reporting -name=randw_64_8k \
--output=/root/`hostname`_`date +%F`randw_64_8k.log \
--write_bw_log=fio-test-`hostname`_`date +%F`randw_64_8k \
--write_iops_log=fio-test-`hostname`_`date +%F`randw_64_8k \
--write_lat_log=fio-test-`hostname`_`date +%F`randw_64_8k

# 下次测试前清理缓存
sync
echo 1 > /proc/sys/vm/drop_caches

```



## 压力测试

测试过程需关闭防火墙，避免无法连接测试端口

压测，即压力测试，是确立系统稳定性的一种测试方法，通常在系统正常运作范围之外进行，以考察其功能极限和隐患。

主要检测服务器的承受能力，包括用户承受能力（多少用户同时玩基本不影响质量）、流量承受等。

### 网络压力测试iperf

没有依赖，可以直接rpm安装

```bash
$ rpm -ivh iperf3-devel-3.6-4 iperf3-3.6.4
```

Iperf软件通过设置Server和Client两个角色可以进行TCP/UDP流量的传输，Linux和Windows环境下命令配置一致。在Server端通过命令创建Server，缺省支持tcp和udp协议，可以通过-u参数设置为仅支持UDP。

```bash
## server
$ iperf3 -u -s

## client
$ iperf3 -c <server_IP> -t 3600 -u -b 1000M
代表以1000Mbit/s速率传输UDP报文，通常用来进行指定压力测试。

## -w/--windows 设置套接字缓冲区为指定大小。对于TCP方式，此设置为TCP窗口大小。
## 对于UDP方式，此设置为接受UDP数据包的缓冲区大小，限制可以接受数据包的最大值。
$ iperf3 -c <server_IP> -t 3600 -w 256K
```

end

#### 测试思路

https://cloud.tencent.com/developer/article/1688469

带宽性能压测通常采用udp模式，因为能测出极限带宽、时延抖动、丢包率。在进行测试时，首先以链路理论带宽作为数据发送速率进行测试，例如，从客户端到服务器之间的链路的理论带宽为100Mbps，先用`-b 100M`进行测试，然后根据测试结果（包括实际带宽，时延抖动和丢包率），再以实际带宽作为数据发送速率进行测试，会发现时延抖动和丢包率比第一次好很多，重复测试几次，就能得出稳定的实际带宽。

通过UDP发包测试不仅可以通过-b xxxM的形式测试实例的带宽性能情况，还可以通过-b xxxpps测试实例的收发包性能，这里还是选用腾讯云标准网络优化型实例（S2ne.LARGE8），官网承诺收发包量为30Wpps进行测试.

#### 优化策略

**网络收发包量的测试同时还会受到缓冲区大小的影响，默认的缓冲区比较小的话，会造成实例到达高pps丢包的现象，这里建议在测试前调整下缓冲区大小；同时由于UDP默认发包大小为1470字节，在发包量很高的情况会超出实例的带宽限制，所以这里需-l 指定发包大小**

```bash
## 调整UDP缓冲区大小
$ vi /etc/sysctl.conf
net.core.wmem_default = 8388608
net.core.rmem_default = 8388608
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216 

然后执行sysctl -p 使得参数生效(不重启刷新参数)
```

end

### CPU and Memory 压测

#### stress

```bash
## CPU压力测试
stress --cpu 8 --timeout 300
## 内存压力测试
stress --vm 10 --vm-bytes 512M

```



### 接口压测

#### ab

#### go-stress

https://github.com/link1st/go-stress-testing



### 引用

1. https://cloud.tencent.com/developer/article/1688469
2. 