## Linux command cookies

### ssh

#### `-fn`

当要批量ssh到不同的机器执行一个不能立马返回执行结果的命令时，可以使用`-fn`参数

```bash
# -f Requests ssh to go to background just before command execution.  This is useful if ssh is going to ask for passwords or passphrases, but the user wants it in the background.
# -n Redirects stdin from /dev/null (actually, prevents reading from stdin).  This must be used when ssh is run in the background.

$ ssh -fn IP "sleep 60"
$ ssh -fn IP "fio xxx"
# example
$ ssh -fn 127.0.0.1 sleep 10
$ ps -ef|grep sleep |grep -v grep
root      5085     1  0 00:40 ?        00:00:00 ssh -fn 127.0.0.1 sleep 10
root      5100  5083  0 00:40 ?        00:00:00 sleep 10

```

end

### ethtool

linux 系统接收网络报文的过程。

1. 首先网络报文通过物理网线发送到网卡
2. 网络驱动程序会把网络中的报文读出来放到 ring buffer 中，这个过程使用 DMA（Direct Memory Access），不需要 CPU 参与
3. 内核从 ring buffer 中读取报文进行处理，执行 IP 和 TCP/UDP 层的逻辑，最后把报文放到应用程序的 socket buffer 中
4. 应用程序从 socket buffer 中读取报文进行处理

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/linux-net-stack.jpg)

如果硬件或者驱动没有问题，一般网卡丢包是因为设置的缓存区（ring buffer）太小，可以使用 `ethtool` 命令查看和设置网卡的 ring buffer。

`ethtool -g` 可以查看某个网卡的 ring buffer，比如下面的例子

```bash
$ ethtool -g eth0
Ring parameters for eth0:
Pre-set maximums:
RX:        4096
RX Mini:    0
RX Jumbo:    0
TX:        4096
Current hardware settings:
RX:        256
RX Mini:    0
RX Jumbo:    0
TX:        256
```

Pre-set 表示网卡最大的 ring buffer 值，可以使用 `ethtool -G eth0 rx 8192` 设置它的值。

### 引用

1. https://cizixs.com/2018/01/13/linux-udp-packet-drop-debug/