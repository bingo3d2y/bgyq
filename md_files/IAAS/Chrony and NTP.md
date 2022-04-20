## Chrony配置使用

 chronyd is a daemon for synchronisation of the system clock. It can synchronise the clock with NTP servers, reference clocks (e.g. a GPS receiver), and manual input using wristwatch and keyboard via chronyc.

CentOS 8中已经移除了ntp和ntpdate，它们也没有集成在基础包中。

CentOS 8使用chronyd作为时间服务器，但如果只是简单做时间同步，可直接使用systemd.timesyncd组件。

### 时间同步：cpu tick

在Linux下，默认情况下，系统时间和硬件时间，并不会自动同步。在Linux运行过程中，系统时间和硬件时间以异步的方式运行，互不干扰。硬件时间的运行，是靠Bios电池来维持，而系统时间，是用CPU tick来维持的。

在系统开机的时候，会自动从Bios中取得硬件时间，设置为系统时间。

```bash
##　同步系统时间和硬件时间，可以使用hwclock命令。

## 以系统时间为基准，修改硬件时间

$ hwclock --systohc<== sys（系统时间）to（写到）hc（Hard Clock）
$ hwclock -w

## 以硬件时间为基准，修改系统时间
$ hwclock --hctosys
$ hwclock -s
```

dnd

### NTP

#### ntpupdate

`ntpdate [-nv] [NTP IP/hostname]` 从时间服务器同步时间

 但这样的同步，只是强制性的将系统时间设置为ntp服务器时间。

如果cpu tick有问题，只是治标不治本。所以，一般配合cron命令，来进行定期同步设置。比如，在crontab中添加：

```bash
00 10 * * * /usr/sbin/ntpdate -u ntp.api.bz > /dev/null 2>&1; /sbin/hwclock -w
```

#### ntpd

 使用ntpd服务，要好于ntpdate加cron的组合。因为，ntpdate同步时间，会造成时间的跳跃，对一些依赖时间的程序和服务会造成影响。比如sleep，timer等。而且，ntpd服务可以在修正时间的同时，修正cpu tick。理想的做法为，在开机的时候，使用ntpdate强制同步时间，在其他时候使用ntpd服务来同步时间。

  要注意的是，ntpd有一个自我保护设置: 如果本机与上源时间相差太大, ntpd不运行. 所以新设置的时间服务器一定要先ntpdate从上源取得时间初值, 然后启动ntpd服务。ntpd服务运行后, 先是每64秒与上源服务器同步一次, 根据每次同步时测得的误差值经复杂计算逐步调整自己的时间, 随着误差减小, 逐步增加同步的间隔. 每次跳动, 都会重复这个调整的过程.

所以，ntpd调整时间速度很慢。

ntp服务，默认只会同步系统时间。如果想要让ntp同时同步硬件时间，可以设置/etc/sysconfig/ntpd文件。

在/etc/sysconfig/ntpd文件中，添加 SYNC_HWCLOCK=yes 这样，就可以让硬件时间与系统时间一起同步。

ntpd server配置文件

```bash
$ cat /etc/ntp.conf |grep -v "^$\|^#"
driftfile /var/lib/ntp/drift
restrict default nomodify notrap nopeer noquery
restrict 127.0.0.1
restrict ::1
includefile /etc/ntp/crypto/pw
keys /etc/ntp/keys
disable monitor
server 100.5.15.214 iburst
server 127.127.1.0
fudge 127.127.1.0 stratum 8

```

end

### chronyd: 类比ntpd

chronyd 类比ntpd，且比它更优就好理解了。ntpupdate是一次性的client同步工具，ntpd和chronyd是daemon可以持续的同步时间。

`Chrony` 会根据实际时间计算修正值，并将补偿参数记录在该指令指定的文件里，默认为 `driftfile /var/lib/chrony/drift`。

客户端进程是`chronyc`

与 `ntpd` 或者 `ntpdate` 最大的区别就是，`Chrony` 的修正是连续的，通过减慢时钟或者加快时钟的方式连续的修正。而 `ntpd` 或者 `ntpdate` 搭配 `Crontab` 的校时工具是直接调整时间，会出现间断，并且相同时间可能会出现两次。因此，**请放弃使用** `ntpd`、`ntpdate` 来校时。

但chronyd 本身就会与按一定间隔同步时间吧--

```bash
## aliyun ecs
$ chronyc tracking
Reference ID    : 64643D58 (100.100.61.88)
Stratum         : 2
## 引用的 还是其他时区的时间服务器
Ref time (UTC)  : Wed Apr 20 07:00:23 2022
System time     : 0.000027242 seconds fast of NTP time
Last offset     : -0.000002264 seconds
RMS offset      : 0.000010882 seconds
Frequency       : 35.756 ppm slow
Residual freq   : +0.000 ppm
Skew            : 0.006 ppm
Root delay      : 0.000113088 seconds
Root dispersion : 0.010375380 seconds
## 上次同步时间
Update interval : 257.1 seconds
Leap status     : Normal
## 本地时区是shanghai
## Universal 是 UTC
$ timedatectl
                      Local time: Wed 2022-04-20 15:02:55 CST
                  Universal time: Wed 2022-04-20 07:02:55 UTC
                        RTC time: Wed 2022-04-20 15:02:56
                       Time zone: Asia/Shanghai (CST, +0800)
       System clock synchronized: yes
systemd-timesyncd.service active: yes
                 RTC in local TZ: yes

Warning: The system is configured to read the RTC time in the local time zone.
         This mode can not be fully supported. It will create various problems
         with time zone changes and daylight saving time adjustments. The RTC
         time is never updated, it relies on external facilities to maintain it.
         If at all possible, use RTC in UTC by calling
         'timedatectl set-local-rtc 0'.


## 
[root@cloudos03 ~]# chronyc
chrony version 3.4
chronyc> sources -v
210 Number of sources = 1

  .-- Source mode  '^' = server, '=' = peer, '#' = local clock.
 / .- Source state '*' = current synced, '+' = combined , '-' = not combined,
| /   '?' = unreachable, 'x' = time may be in error, '~' = time too variable.
||                                                 .- xxxx [ yyyy ] +/- zzzz
||      Reachability register (octal) -.           |  xxxx = adjusted offset,
||      Log2(Polling interval) --.      |          |  yyyy = measured offset,
||                                \     |          |  zzzz = estimated error.
||                                 |    |           \
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^* os-chrony-svc.default.sv>    10   0   377     0   -112us[ -115us] +/-  387us
chronyc>


$ chronyc tracking
Reference ID    : 0A6426B2 (os-chrony-svc.default.svc.cloudos)
Stratum         : 11
Ref time (UTC)  : Thu Apr 21 20:23:57 2022
System time     : 1.429598689 seconds slow of NTP time
Last offset     : +0.000003098 seconds
RMS offset      : 0.000003699 seconds
Frequency       : 100000.578 ppm slow
Residual freq   : +0.038 ppm
Skew            : 1.284 ppm
Root delay      : 0.000327190 seconds
Root dispersion : 0.000037934 seconds
#  说明最后两次更新的时间间隔是
Update interval : 1.0 seconds
Leap status     : Normal

```

end

### 阿里云的chronyd配置

```bash
$ cat /etc/chrony/chrony.conf
# Use Alibaba NTP server
# Public NTP
# Alicloud NTP


server ntp.cloud.aliyuncs.com minpoll 4 maxpoll 10 iburst
server ntp.aliyun.com minpoll 4 maxpoll 10 iburst
server ntp1.aliyun.com minpoll 4 maxpoll 10 iburst
server ntp1.cloud.aliyuncs.com minpoll 4 maxpoll 10 iburst
server ntp10.cloud.aliyuncs.com minpoll 4 maxpoll 10 iburst
server ntp11.cloud.aliyuncs.com minpoll 4 maxpoll 10 iburst
server ntp12.cloud.aliyuncs.com minpoll 4 maxpoll 10 iburst
server ntp2.aliyun.com minpoll 4 maxpoll 10 iburst
server ntp2.cloud.aliyuncs.com minpoll 4 maxpoll 10 iburst
server ntp3.aliyun.com minpoll 4 maxpoll 10 iburst
server ntp3.cloud.aliyuncs.com minpoll 4 maxpoll 10 iburst
server ntp4.aliyun.com minpoll 4 maxpoll 10 iburst
server ntp4.cloud.aliyuncs.com minpoll 4 maxpoll 10 iburst
server ntp5.aliyun.com minpoll 4 maxpoll 10 iburst
server ntp5.cloud.aliyuncs.com minpoll 4 maxpoll 10 iburst
server ntp6.aliyun.com minpoll 4 maxpoll 10 iburst
server ntp6.cloud.aliyuncs.com minpoll 4 maxpoll 10 iburst
server ntp7.cloud.aliyuncs.com minpoll 4 maxpoll 10 iburst
server ntp8.cloud.aliyuncs.com minpoll 4 maxpoll 10 iburst
server ntp9.cloud.aliyuncs.com minpoll 4 maxpoll 10 iburst

# Ignore stratum in source selection.
stratumweight 0.05

# Record the rate at which the system clock gains/losses time.
driftfile /var/lib/chrony/drift

# Enable kernel RTC synchronization.
rtcsync

# In first three updates step the system clock instead of slew
# if the adjustment is larger than 10 seconds.
makestep 10 3

# Allow NTP client access from local network.
#allow 192.168/16

# Listen for commands only on localhost.
bindcmdaddress 127.0.0.1
bindcmdaddress ::1

# Disable logging of client accesses.
noclientlog

# Send a message to syslog if a clock adjustment is larger than 0.5 seconds.
logchange 0.5

logdir /var/log/chrony
#log measurements statistics tracking
root@Dodo:~#


## 查看同步状态
chronyc sourcestats -v
210 Number of sources = 15
                             .- Number of sample points in measurement set.
                            /    .- Number of residual runs with same sign.
                           |    /    .- Length of measurement set (time).
                           |   |    /      .- Est. clock freq error (ppm).
                           |   |   |      /           .- Est. error in freq.
                           |   |   |     |           /         .- Est. offset.
                           |   |   |     |          |          |   On the -.
                           |   |   |     |          |          |   samples. \
                           |   |   |     |          |          |             |
Name/IP Address            NP  NR  Span  Frequency  Freq Skew  Offset  Std Dev
==============================================================================
100.100.61.88               6   3   21m     +0.018      0.047  -8060ns  5144ns
203.107.6.88               64  41   18h     -0.009      0.038   -113us  1775us
120.25.115.20              16   9  466m     +0.044      0.076   +573us   649us
10.143.33.49                0   0     0     +0.000   2000.000     +0ns  4000ms
100.100.3.1                13   9  207m     -0.017      0.035    +49us   107us
100.100.3.2                11   6  172m     -0.006      0.039  -7716ns    83us
100.100.3.3                 8   5  137m     -0.001      0.030   -165us    34us
10.143.33.50                0   0     0     +0.000   2000.000     +0ns  4000ms
10.143.33.51                0   0     0     +0.000   2000.000     +0ns  4000ms
10.143.0.44                 0   0     0     +0.000   2000.000     +0ns  4000ms
10.143.0.45                 0   0     0     +0.000   2000.000     +0ns  4000ms
10.143.0.46                 0   0     0     +0.000   2000.000     +0ns  4000ms
100.100.5.1                 8   5  121m     -0.011      0.073   +238us    85us
100.100.5.2                11   7  171m     -0.016      0.043   -291us    98us
100.100.5.3                 7   5  103m     +0.013      0.022   +108us    19us


## ubuntu
$ timedatectl
                      Local time: Wed 2022-04-20 15:23:57 CST
                  Universal time: Wed 2022-04-20 07:23:57 UTC
                        RTC time: Wed 2022-04-20 15:23:58
                       Time zone: Asia/Shanghai (CST, +0800)
       System clock synchronized: yes
# systemd-timesyncd服务为活动也就是开启了NTP时间同步
systemd-timesyncd.service active: yes
                 RTC in local TZ: yes

## Centos
$ timedatectl
      Local time: Sat 2022-04-23 05:24:54 CST
  Universal time: Fri 2022-04-22 21:24:54 UTC
        RTC time: Fri 2022-04-22 21:24:49
       Time zone: Asia/Shanghai (CST, +0800)
     NTP enabled: yes
NTP synchronized: yes
 RTC in local TZ: no
      DST active: n/a

```

end

### 引用

1. https://www.cnblogs.com/liushui-sky/p/9203657.html