# 虚拟化中KVM, Xen, Qemu的区别及Hypervisor

#### [Hypervisor][ haipə'vaizə]

0.0 完成内存、CPU、IO的虚拟化翻译工作。

Hypervisors allow a single set of physical hardware to host multiple virtual machines, and hold all of the necessary variables and information required to make those virtual machines work.  **hypervisors are software, specialized firmware, or both which allow physical hardware to be shared across multiple virtual machines.** 

> Hypervisors 允许单台物理设备管理多个虚拟机，并保存这些虚拟机工作交流所必需的变量和信息。
>
> hypervisor可以是软件，特定的固件或者一切可以共享联通多个虚拟机的物理硬件。

**Benefits of hypervisors**： kube-node3的kubelet进程cpu使用率1000%,该vm hung但是不影响其他同node的vms

Even though VMs can run on the same physical hardware, they are still logically separated from each other. This means that if one VM experiences an error, crash or a malware attack, it doesn’t extend to other VMs on the same machine, or even other machines.

> VMs虽然运行在同一个物理硬件上面，但是它们在逻辑上还是彼此分离的。这意味着，其中一个VM遭受错误，中断或者恶意软件攻击，不会影响该宿主机上的其他VMs，更不会影响其他物理机。
>
> VM的隔离性还是很强的。Eg：kube-node3 ，kubelet爆炸，只影响这个一个node。
>
> Containers就不行了因为它的所有runtime process都可以在host上面用ps查到，container不做资源限制会耗光整个host，且container若是被攻击，则很可能波及整个host。

**主要分为下面两种Hypervisor：**

**Type-1 hypervisors**： Type-1 hypervisor is direct access to the physical hardware

**Type-1 hypervisors** are installed directly to the “bare metal” – **in other words** they require no software or Operating Systems to be installed ahead of time, and install right onto the hardware you want to run them on.  The **benefit of a Type-1 hypervisor is direct access to the physical hardware**, with no operating system getting in the way of the virtual machines using that hardware.  

> Type-1 Hypervisors可以直接运行在“裸金属”上，换句话说它们不需要预先安装操作系统软件，可以直接在硬件上运行。

**Type-2 hypervisors**

**Type-2 hypervisors are designed to install onto an existing operating system**.    **Type-2 hypervisors typically cannot run more complex virtual workloads or high-[utilization][juːtɪlaɪ'zeɪʃən n.利用，使用] workloads within virtual machines.  They’re** **designed for basic development, testing, and emulation** (such as running Windows software on a Mac, or running a virtual desktop provided by a company on a personal machine).

> Type-2被设计安装在已存在的操作系统上面。
>
> 0.0 Type-2通常性能不太好？？

博观而约取

## QEMU

QEMU ： quickly emulator

**QEMU is a userland type 2 (i.e runs upon a host OS) hypervisor** for **performing hardware virtualization** (not to be confused with hardware-assisted virtualization), such as disk, network, VGA, PCI, USB, serial/parallel ports, etc. It is flexible in that it can emulate CPUs via dynamic binary translation (DBT) allowing code written for a given processor to be executed on another (i.e ARM on x86, or PPC on ARM). Though QEMU can run on its own and emulate all of the virtual machines resources, as all the emulation is performed in software it is **extremely slow**.

qemu不用使用硬件的虚拟化，全部是使用软件虚拟一切硬件（CPU和disk等）

## KVM

KVM is a Linux kernel module. It is **a type 1 hypervisor** that is a full virtualization solution for Linux on x86 hardware containing virtualization extensions (Intel VT or AMD-V). But what is full virtualization, you may ask? **When a CPU is emulated (vCPU) by the hypervisor, the hypervisor has to translate the instructions meant for the vCPU to the physical CPU. As you can imagine this has a massive performance impact.** To overcome this, modern processors support virtualization extensions, such as Intel VT-x and AMD-V. These technologies provide the ability for a slice of the physical CPU to be directly mapped to the vCPU. Therefor the the instructions meant for the vCPU can be directly executed on the physical CPU slice.

支持硬件辅助虚拟化。部分低权限的指令直接走kvm module，一些高权限指令则继续走qemu，从而提高性能

Intel CPU： `grep "vmx" /proc/cpuinfo`

AMD CPU: `grep "svm" /proc/cpuinfo`

These technologies provide the ability for a slice of the physical CPU to be directly mapped to the vCPU.

0.0 KVM结合一些支持虚拟化的CPU硬件，使一些操作指令直接CPU slice 操作而不经过翻译后传给vCPU

CPU  slice， cpu切片

## SUMMARY

As previously mentioned, QEMU can run independently, but due to the emulation being performed entirely in software it is extremely slow. To overcome this, QEMU allows you to use KVM as an [accelerator][**the pedal (= a part that you push with your foot) in a vehicle that makes it gofaster** 加速器] so that t**he physical CPU virtualization extensions can be used.** ==So to conclude== QEMU is a type 2 hypervisor that runs within user space and performs virtual hardware emulation, where as KVM is a type 1 hypervisor that runs in kernel space, that allows a user space program access to the hardware virtualization features of various processors.

kvm可以使用CPU slice 所以是type 1



## 虚拟化类型（CPU虚拟化）

### 全虚拟化（Full Virtualization)

全虚拟化也成为原始虚拟化技术，该模型使用虚拟机协调guest操作系统和原始硬件，VMM在guest操作系统和裸硬件之间用于工作协调，一些受保护指令必须由Hypervisor（虚拟机管理程序）来捕获处理。

![tech-full-virtualization](D:\data_files\MarkDown\Images\tech-full-virtualization.gif)
​								图1 全虚拟化模型

全虚拟化的运行速度要快于硬件模拟，但是性能方面不如裸机，因为Hypervisor需要占用一些资源

### 半虚拟化（Para Virtualization）

半虚拟化是另一种类似于全虚拟化的技术，它使用Hypervisor分享存取底层的硬件，但是它的guest操作系统集成了虚拟化方面的代码。该方法无需重新编译或引起陷阱，因为操作系统自身能够与虚拟进程进行很好的协作。

![tech-para-virtualization](D:\data_files\MarkDown\Images\tech-para-virtualization.gif)
​								图2 半虚拟化模型

半虚拟化需要guest操作系统做一些修改，使guest操作系统意识到自己是处于虚拟化环境的，但是半虚拟化提供了与原操作系统相近的性能。

## 虚拟化技术

### KVM(Kernel-based Virtual Machine)基于内核的虚拟机（全虚拟化）

**KVM是集成到Linux内核的Hypervisor**，是X86架构且硬件支持虚拟化技术（Intel VT或AMD-V）的Linux的**全虚拟化**解决方案。**它是Linux的一个很小的模块**，利用Linux做大量的事，如任务调度、内存管理与硬件设备交互等。

![tech-kvm-architecture](D:\data_files\MarkDown\Images\tech-kvm-architecture.jpg)
						图3 KVM虚拟化平台架构

### Xen（全虚拟化和半虚拟化）

**Xen是第一类运行再裸机上的虚拟化管理程序(Hypervisor)。**它支持**全虚拟化和半虚拟化**,Xen支持hypervisor和虚拟机互相通讯，而且提供在所有Linux版本上的免费产品，包括Red Hat Enterprise Linux和SUSE Linux Enterprise Server。**Xen最重要的优势在于半虚拟化**，此外未经修改的操作系统也可以直接在xen上运行(如Windows)，能让虚拟机有效运行而不需要仿真，因此虚拟机能感知到hypervisor，而不需要模拟虚拟硬件，从而能实现高性能。

![tech-xen-architecture](D:\data_files\MarkDown\Images\tech-xen-architecture.jpg)
​							图4 Xen虚拟化平台架构

### QEMU

QEMU是一套由Fabrice Bellard所编写的模拟处理器的自由软件。它与Bochs，PearPC近似，但其具有某些后两者所不具备的特性，如高速度及跨平台的特性。经由kqemu这个开源的加速器，QEMU能模拟至接近真实电脑的速度。

### KVM和QEMU的关系（看下）

准确来说，**KVM是Linux kernel的一个模块**。可以用命令modprobe去加载KVM模块。加载了模块后，才能进一步通过其他工具创建虚拟机。但仅有KVM模块是 远远不够的，因为用户无法直接控制内核模块去作事情,你还必须有一个运行在用户空间的工具才行。这个用户空间的工具，kvm开发者选择了已经成型的开源虚拟化软件 QEMU。说起来QEMU也是一个虚拟化软件。它的特点是可虚拟不同的CPU。比如说在x86的CPU上可虚拟一个Power的CPU，并可利用它编译出可运行在Power上的程序。KVM使用了QEMU的一部分，并稍加改造，就成了可控制KVM的用户空间工具了。所以你会看到，官方提供的KVM下载有两大部分(qemu和kvm)三个文件(KVM模块、QEMU工具以及二者的合集)。也就是说，你可以只升级KVM模块，也可以只升级QEMU工具。这就是KVM和QEMU 的关系。

![tech-kvm-and-qemu](D:\data_files\MarkDown\Images\tech-kvm-and-qemu.png)
​						图5 KVM和QEMU关系

kvm与QEME的关系和NetFilter与iptables/firewalld类似。

kerne space： kvm      NetFilter

user space:   QEMU    iptables/firewalld   



# CPU硬件辅助虚拟化（AMD和INTEL需要配置的内核参数也不一样）

##内存虚拟化

**背景**

内存用于暂存CPU将要执行的指令和数据，所有程序的运行都必须先载入到内存中才可以，内存的大小及其访问速度也直接影响整个系统性能。在平台虚拟化技术中，Guest的运行也需要依赖内存。和运行在真实物理硬件上的操作系统一样，在Guest操作系统看来，Guest可用的内存空间也是一个从零地址开始的连续的物理内存空间。为了达到这个目的，Hypervisor（即KVM）引入了一层新的地址空间，即Guest物理地址空间，这个地址空间不是真正的硬件上的地址空间，它们之间还有一层映射。所以，在虚拟化环境下，内存使用就需要两层的地址转换，即Guest应用程序可见的Guest虚拟地址（Guest Virtual Address，GVA）到客户机物理地址（Guest Physical Address，GPA）的转换，再从Guest物理地址（GPA）到宿主机物理地址（Host Physical Address，HPA）的转换。其中，前一个转换由Guest操作系统来完成，而后一个转换由Hypervisor来负责。为了解决这两次地址转换，Intel先后使用了不同的技术手段。

**影子页表**

影子页表（Shadow Page Tables）是从软件上维护了从GVA到HPA之间的映射，每一份Guest操作系统的页表也对应一份影子页表。有了影子页表，在普通的内存访问时都可实现从GVA到HPA的直接转换，从而避免了上面前面提到的两次地址转换。Hypervisor将影子页表载入到物理上的内存管理单元（Memory Management Unit，MMU）中进行地址翻译。GVA、GPA、HPA之间的转换，以及影子页表的作用如下图。

![内存虚拟化-shadow Page Tables](D:\data_files\MarkDown\Images\内存虚拟化-shadow Page Tables.png)

尽管影子页表提供了在物理MMU硬件中能使用的页表，但是其缺点也是比较明显的。

**技术复杂**

影子页表实现非常复杂，导致其开发、调试和维护都比较困难。

**物理内存开销大**

由于需要为每个客户机进程对应的页表的都维护一个影子页表，因此影子页表的内存开销非常大。

**EPT**

EPT（Extended Page Tables，扩展页表），属于Intel的第二代硬件虚拟化技术，它是针对内存管理单元（MMU）的虚拟化扩展。相对于影子页表，EPT降低了内存虚拟化的难度（，也提升了内存虚拟化的性能。从基于Intel的Nehalem架构的平台开始，EPT就作为CPU的一个特性加入到CPU硬件中去了。

Intel在CPU中使用EPT技术，AMD也提供的类似技术叫做NPT，即Nested Page Tables。都是直接在硬件上支持GVA-->GPA-->HPA的两次地址转换，从而降低内存虚拟化实现的复杂度，也进一步提升了内存虚拟化的性能。Intel EPT技术的基本原理如下图

![内存虚拟化-ept](D:\data_files\MarkDown\Images\内存虚拟化-ept.png)

CR3（控制寄存器3）将客户机程序所见的客户机虚拟地址（GVA）转化为客户机物理地址（GPA），然后在通过EPT将客户机物理地址（GPA）转化为宿主机物理地址（HPA）。这两次转换地址转换都是由CPU硬件来自动完成的，其转换效率非常高。

在使用EPT的情况下，客户机内部的Page Fault、INVLPG（使TLB]项目失效）指令、CR3寄存器的访问等都不会引起[VM-Exit](https://www.cnblogs.com/kelamoyujuzhen/p/10115167.html)，因此大大减少了VM-Exit的数量，从而提高了性能。

EPT只需要维护一张EPT页表，而不需要像“影子页表”那样为每个客户机进程的页表维护一张影子页表，从而也减少了内存的开销。

**VPID**

TLB(Translation Lookaside Buffer)转换检测缓冲区是一个内存管理单元,用于改进虚拟地址到物理地址转换速度的缓存。

VPID（VirtualProcessor Identifiers，虚拟处理器标识），是在硬件上对TLB资源管理的优化，通过在硬件上为每个TLB项增加一个标识，用于不同的虚拟处理器的地址空间，从而能够区分开Hypervisor和不同处理器的TLB。

硬件区分了不同的TLB项分别属于不同虚拟处理器，因此可以避免每次进行VM-Entry和VM-Exit时都让TLB全部失效，提高了VM切换的效率。

由于有了这些在VM切换后仍然继续存在的TLB项，硬件减少了一些不必要的页表访问，减少了内存访问次数，从而提高了Hypervisor和客户机的运行速度。

VPID也会对客户机的实时迁移（Live Migration）有很好的效率提升，会节省实时迁移的开销，会提升实时迁移的速度，降低迁移的延迟（Latency）。

VPID与EPT是一起加入到CPU中的特性，也是Intel公司在2009年推出Nehalem系列处理器上新增的与虚拟化相关的重要功能。

**Host上常看CPU对EPT、VPID支持情况**

我Host上有8颗逻辑CPU

```
[root@localhost ~]# grep "ept vpid" /proc/cpuinfo -o
ept vpid
ept vpid
ept vpid
ept vpid
ept vpid
ept vpid
ept vpid
ept vpid
```

#####**KVM虚拟化下如何开关EPT、VPID**

在加载kvm_intel模块时，可以通过设置ept和vpid参数的值来打开或关闭EPT和VPID。

当然，如果kvm_intel模块已经处于加载状态，则需要先卸载这个模块，在重新加载之时加入所需的参数设置。

```
[root@localhost ~]# cat /sys/module/kvm_intel/parameters/ept
Y
[root@localhost ~]# cat /sys/module/kvm_intel/parameters/vpid
Y
```

一般不要手动关闭EPT和VPID功能，否则会导致Guest中内存访问的性能下降。

## CPU虚拟化

针对X86系列的CPU虚拟化，Intel从2006年，AMD从2007年开始，对CPU从硬件层面进行了扩展，创造了环-1，Hypervisor运行在环-1，Guest运行在环0，这样当Guest系统中的应用发起系统调用时，还是直接向Guest内核发起请求，除了部分敏感指令（特权指令的子集）外，其余指令不在需要Hypervisor进行辅助，由于他不需要Guest修改内核，又比全虚拟化省去了捕获翻译的过程，所以成为现阶段CPU虚拟化的主流技术。

## IO虚拟化

