## CPU soft lookup--draft

### what is soft lookup

所谓lockup，是指某段内核代码占着CPU不放。Lockup严重的情况下会导致整个系统失去响应。Lockup有几个特点：

- 首先只有内核代码才能引起lockup，因为用户代码是可以被抢占的，不可能形成lockup（只有一种情况例外，就是SCHED_FIFO优先级为99的实时进程即使在用户态也可能使[watchdog/x]内核线程抢不到CPU而形成soft lock，参见《[Real-Time进程会导致系统Lockup吗？](http://linuxperf.com/?p=197)》）
- 其次内核代码必须处于禁止内核抢占的状态(preemption disabled)，因为Linux是可抢占式的内核，只在某些特定的代码区才禁止抢占，在这些代码区才有可能形成lockup。

Lockup分为两种：soft lockup 和 hard lockup，它们的区别是 hard lockup 发生在CPU屏蔽中断的情况下。

- Soft lockup是指CPU被内核代码占据，以至于无法执行其它进程。检测soft lockup的原理是给每个CPU分配一个定时执行的内核线程[watchdog/x]，如果该线程在设定的期限内没有得到执行的话就意味着发生了soft lockup，[watchdog/x]是SCHED_FIFO实时进程，优先级为最高的99，拥有优先运行的特权。
- Hard lockup比soft lockup更加严重，CPU不仅无法执行其它进程，而且不再响应中断。检测hard lockup的原理利用了PMU的NMI perf event，因为NMI中断是不可屏蔽的，在CPU不再响应中断的情况下仍然可以得到执行，它再去检查时钟中断的计数器hrtimer_interrupts是否在保持递增，如果停滞就意味着时钟中断未得到响应，也就是发生了hard lockup。

Linux kernel设计了一个检测lockup的机制，称为**NMI Watchdog**，是利用NMI中断实现的，用NMI是因为lockup有可能发生在中断被屏蔽的状态下，这时唯一能把CPU抢下来的方法就是通过NMI，因为NMI中断是不可屏蔽的。

### resolution soft lookup

This 'soft lockup' can happen if the kernel is busy, working on a huge amount of objects which need to be scanned, freed, or allocated, respectively.
The stack traces of those tasks can give a first idea about what the tasks were doing. However, to be able to examine the cause behind the messages, a kernel dump would be needed.

While these messages cannot be disabled entirely, in some situations, increasing the time before these

soft lockups are fired can relax the situation.

 

To do so, increase the following sysctl parameter: `kernel.watchdog_thresh`

 

Default value for this parameter is 10 and to double the value might be a good start.

 ```bash
 $ echo "kernel.watchdog_thresh=20" > /etc/sysctl.d/99-watchdog_thresh.conf
 $ sysctl -p  /etc/sysctl.d/99-watchdog_thresh.conf
 
 ```

A 'soft lockup' is defined as a bug that causes the kernel to loop in kernel mode for more than 20 seconds without giving other tasks a chance to run.
The watchdog daemon will send an non-maskable interrupt (NMI) to all CPUs in the system who, in turn, print the stack traces of their currently running tasks.

### 脑洞

可以使用eBPF起个hook打印soft lookup的信息么

### 引用：

1. http://linuxperf.com/?p=83
2. https://www.suse.com/support/kb/doc/?id=000018705
3. end