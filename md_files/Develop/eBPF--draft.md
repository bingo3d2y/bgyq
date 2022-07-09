## eBPF--draft

### 学习资料

http://arthurchiao.art/blog/ebpf-and-k8s-zh/

### demo code

看着demo感觉是eBPF可以在linux kernel代码函数中添加任意hook来进行操作

```bash
## 安装bbc及依赖
apt-get install bpfcc-tools linux-headers-$(uname -r)
# add key server
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys D4284CDD
# add iovisor to repo
echo "deb https://repo.iovisor.org/apt/bionic bionic main" | sudo tee /etc/apt/sources.list.d/iovisor.list
# update the repo
sudo apt-get update
# install libbcc
sudo apt-get install libbcc
# install python-bcc
sudo apt-get install python-bcc
```

python demo code

```python
from bcc import BPF

# define BPF program
prog = """
int hello(void *ctx) {
    bpf_trace_printk("my bcc program\\n");
    return 0;
}
"""

# load BPF program
b = BPF(text=prog)
b.attach_kprobe(event=b.get_syscall_fnname("clone"), fn_name="hello")

# header
print("%-18s %-16s %-6s %s" % ("TIME(s)", "COMM", "PID", "MESSAGE"))

# format output
while 1:
    try:
        (task, pid, cpu, flags, ts, msg) = b.trace_fields()
    except ValueError:
        continue
    print("%-18.9f %-16s %-6d %s" % (ts, task, pid, msg))


```

程序分析：

1. `prog = """ xxx ""`此处通过变量声明了一个 C 程序源码，其中xxx是可以换行的 C 程序。
2. `int hello() { xxx }` 声明了一个 C 语言函数，未使用上个例子中 kprobe__ 开头的快捷方式。BPF 程序中的任何 C 函数都需要在一个探针上执行，因此我们必须将 pt_reg* ctx 这样的 ctx 变量放在第一个参数。如果需要声明一些不在探针上执行的辅助函数，则需要定义成 static inline 以便编译器内联编译。有时候可能需要添加 _always_inline 函数属性。
3. `b.attach_kprobe(event=b.get_syscall_fnname("clone"), fn_name="hello")` 这里建立了一个内核探针，内核系统出现 clone 操作时执行 hello() 这个函数。可以多次调用 `attch_kprobe()` ，这样就可以用 C 语言函数跟踪多个内核函数。
4. `b.trace_fields()` 这里从 `trace_pipe` 返回一个混合数据，这对于黑客测试很方便，但是实际工具开发中需要使用 `BPF_PERF_OUTPUT()` 

还有一个一个简单的例子，代码如下：

```python
from bcc import BPF
BPF(text='int kprobe__sys_clone(void *ctx) { bpf_trace_printk("Hello, World!\\n"); return 0; }').trace_print()
```

可以看到，这段代码是在 python 中内嵌 C 语言程序的，请注意六点：

1. `text='...'` 这里定义了一个内联的、C 语言写的 BPF 程序。
2. `kprobe__sys_clone()` 这是一个通过内核探针（kprobe）进行内核动态跟踪的快捷方式。如果一个 C 函数名开头为 kprobe__ ，则后面的部分实际为设备的内核函数名，这里是 sys_clone() 。
3. void *ctx 这里的 ctx 实际上有一些参数，不过这里我们用不到，暂时转为 void * 。
4. `bpf_trace_printk()`这是一个简单的内核工具，用于 printf() 到 trace_pipe（译者注：可以理解为 BPF C 代码中的 printf()）。它一般来快速调试一些东西，不过有一些限制：最多有三个参数，一个%s ，并且 trace_pipe 是全局共享的，所以会导致并发程序的输出冲突，因而 BPF_PERF_OUTPUT() 是一个更棒的方案，我们后面会提到。
5. return 0 这是一个必须的部分。
6. `trace_print()` 一个 bcc 实例会通过这个读取 trace_pipe 并打印出来。

### eBPF工具

当linux 出现 soft lookup时，能不能通过eBPF起个hook，打印出进程、内存和内核信息。

This 'soft lockup' can happen if the kernel is busy, working on a huge amount of objects which need to be scanned, freed, or allocated, respectively.
The stack traces of those tasks can give a first idea about what the tasks were doing. However, to be able to examine the cause behind the messages, a kernel dump would be needed.

### 引用

1. http://kerneltravel.net/blog/2020/ebpf_ljr_no1/
2. 