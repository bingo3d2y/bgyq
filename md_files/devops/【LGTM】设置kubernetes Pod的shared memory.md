# 【LGTM】设置kubernetes Pod的shared memory

by **伊布**

November 10, 2019

in [Tech](https://ieevee.com/category/tech)

- [问题描述](https://ieevee.com/tech/2019/11/10/shm.html#问题描述)
- 背景知识1：shared memory
  - [System V](https://ieevee.com/tech/2019/11/10/shm.html#system-v)
  - [POSIX](https://ieevee.com/tech/2019/11/10/shm.html#posix)
- [背景知识2：kubernetes shared memory](https://ieevee.com/tech/2019/11/10/shm.html#背景知识2kubernetes-shared-memory)
- [kubernetes empty dir](https://ieevee.com/tech/2019/11/10/shm.html#kubernetes-empty-dir)
- [使用empty dir的缺点](https://ieevee.com/tech/2019/11/10/shm.html#使用empty-dir的缺点)
- cgroup限制
  - [内存的cgroup限制](https://ieevee.com/tech/2019/11/10/shm.html#内存的cgroup限制)
  - [共享内存受cgroup限制吗？](https://ieevee.com/tech/2019/11/10/shm.html#共享内存受cgroup限制吗)
- [最终的设计](https://ieevee.com/tech/2019/11/10/shm.html#最终的设计)

### 问题描述

用户可以使用共享内存来做一些进行通信（vs golang的 “通过通信来共享内存”）。在kvm或者物理机器上，用户可以使用的共享内存大小约为总内存的一半。如下是我的pve机器的共享内存的情况，`/dev/shm`即为共享内存的大小。

```
root@pve:~# free -h
              total        used        free      shared  buff/cache   available
Mem:           47Gi        33Gi       3.4Gi       1.7Gi       9.7Gi        10Gi
Swap:            0B          0B          0B
root@pve:~# df -h
Filesystem            Size  Used Avail Use% Mounted on
udev                   24G     0   24G   0% /dev
tmpfs                  24G   54M   24G   1% /dev/shm
```

但在kubernetes上，Pod里无法使用超过 64MB的shared memory。如下是集群上一个pod的信息，可以看到，`/dev/shm`的大小是 64MB，通过dd往共享内存写数据时，写到64MB的时候就会报“No space left on device”。

```
$ kubectl run centos --rm -it --image centos:7 bash
[root@centos-7df8445565-dvpnt /]# df -h|grep shm
Filesystem      Size  Used Avail Use% Mounted on
shm              64M     0   64M   0% /dev/shm
[root@centos-7df8445565-dvpnt /]# dd if=/dev/zero of=/dev/shm/test      bs=1024 count=102400
dd: error writing '/dev/shm/test': No space left on device
65537+0 records in
65536+0 records out
67108864 bytes (67 MB) copied, 0.161447 s, 416 MB/s
[root@centos-7df8445565-dvpnt /]# ls -l -h
total 64M
-rw-r--r-- 1 root root 64M Nov 14 14:33 test
[root@centos-7df8445565-dvpnt /]# df -h|grep shm
Filesystem      Size  Used Avail Use% Mounted on
shm              64M   64M     0 100% /dev/shm
```

### 背景知识1：shared memory

shared memory(共享内存)，是Linux上一种用于进程间通信(IPC)的机制。

我们知道进程间通信可以使用管道，Socket，信号，信号量，消息队列等方式，但这些方式通常需要在用户态、内核态之间拷贝，一般认为会有4次拷贝；相比之下，共享内存将内存直接映射到用户态空间，即多个进程访问同一块内存，理论上性能更高。

共享内存通常与信号量结合使用，达到进程间的同步与互斥。

共享内存有两种使用方式，System V和POSIX。其中：

- shmget/shmat/shmdt 为System V方式
- shm_open/mmap/shm_unlink 为POSIX方式，

#### System V

System V 机制历史悠久，我们贴一个 System V的示例代码。

下面是写进程的代码。

```
#include <iostream> 
#include <sys/ipc.h> 
#include <sys/shm.h> 
#include <stdio.h> 
using namespace std; 
  
int main() 
{ 
    // ftok to generate unique key 
    key_t key = ftok("shmfile",65); 
  
    // shmget returns an identifier in shmid 
    int shmid = shmget(key,1024,0666|IPC_CREAT); 
  
    // shmat to attach to shared memory 
    char *str = (char*) shmat(shmid,(void*)0,0); 
  
    cout<<"Write Data : "; 
    gets(str); 
  
    printf("Data written in memory: %s\n",str); 
      
    //detach from shared memory  
    shmdt(str); 
  
    return 0; 
} 
```

下面是读取进程的代码。可以看到，读取进程根据`key`找到了对应的共享内存，并且读取了共享内存中的内容。

```
#include <iostream> 
#include <sys/ipc.h> 
#include <sys/shm.h> 
#include <stdio.h> 
using namespace std; 
  
int main() 
{ 
    // ftok to generate unique key 
    key_t key = ftok("shmfile",65); 
  
    // shmget returns an identifier in shmid 
    int shmid = shmget(key,1024,0666|IPC_CREAT); 
  
    // shmat to attach to shared memory 
    char *str = (char*) shmat(shmid,(void*)0,0); 
  
    printf("Data read from memory: %s\n",str); 
      
    //detach from shared memory  
    shmdt(str); 
    
    // destroy the shared memory 
    shmctl(shmid,IPC_RMID,NULL); 
     
    return 0; 
} 
```

#### POSIX

POSIX接口更方便易用一些，其中的`shm_open("abc"..)`，实际就相当于`open("/dev/shm/aaa"..)`，将共享内存转换为了文件操作；然后通过mmap将文件映射到指针，通过指针达到共享内存的寻址。

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <sys/shm.h>
#include <sys/stat.h>
#include <sys/mman.h>
#include <unistd.h>

int main()
{
    /* the size (in bytes) of shared memory object */
    long size;
    printf("Size of shm(in bytes): ");
    scanf("%ld", &size);
    printf("size %ld\r\n", size);

    /* name of the shared memory object */
    const char* name = "test";

    /* shared memory file descriptor */
    int shm_fd;

    /* pointer to shared memory obect */
    void* ptr;

    /* create the shared memory object */
    shm_fd = shm_open(name, O_CREAT | O_RDWR, 0666);

    /* configure the size of the shared memory object */
    ftruncate(shm_fd, size);

    /* memory map the shared memory object */
    ptr = mmap(0, size, PROT_WRITE, MAP_SHARED, shm_fd, 0);

    memset(ptr, 0, size);
    pause();
    close(shm_fd);
    shm_unlink(name);
    return 0;
}
```

### 背景知识2：kubernetes shared memory

kubernetes创建的Pod，其共享内存默认64MB，且不可更改。

为什么是这个值呢？其实，kubernetes本身是没有设置share memory的大小的，64MB其实是docker 默认的共享内存的大小。

docker run 的时候，可以通过`--shm-size`来设置共享内存的大小。

```
$ docker run  --rm centos:7 df -h |grep shm
shm              64M     0   64M   0% /dev/shm
$ docker run  --rm --shm-size 128M centos:7 df -h |grep shm
shm             128M     0  128M   0% /dev/shm
```

然而，kubernetes并没有提供设置shm大小的途径。在这个[issue](https://github.com/kubernetes/kubernetes/issues/28272)里社区讨论里很久是否要给shm增加一个参数，但是最终并没有形成结论，只是有一个workgroud的办法：将memory类型的emptyDir挂载到/dev/shm来解决。

### kubernetes empty dir

emptyDir volume在一些场景下非常有用，例如在Pod的不同container之间需要共享数据，可以将同一个emptyDir挂载到两个container中来达到共享的目的。此时，emtpyDir实际是一个宿主机上的目录，本质上还是磁盘（比如通过sidecar的方式来收集日志）。

kubernetes提供来一种特殊的emptyDir：medium为memory的临时存储。用户可以将memory介质的emptyDir挂到任何目录，然后将这个目录当作一个高性能的文件系统来使用。

当然，我们也可以将memory介质的empty dir，挂载到 /dev/shm 下。这样，就可以解决共享内存 /dev/shm 不够用的问题。

下面是一个示例。

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: xxx
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: xxx
  template:
    metadata:
      labels:
        app: xxx
    spec:
      containers:
      - args:
        - infinity
        command:
        - sleep
        image: centos:7
        name: centos
        volumeMounts:
        - mountPath: /dev/shm
          name: cache-volume
      volumes:
      - emptyDir:
          medium: Memory
          sizeLimit: 128Mi
        name: cache-volume
```

在kubernetes上执行后，我们进入到容器里进行一番操作，查看下`/dev/shm`是否扩容，是否不会超过`sizeLimit`到值；如果超过`sizeLimit`，会发生什么。

```
$ kubectl exec -it xxx-f6f865646-l54tq bash
[root@xxx-f6f865646-l54tq /]# df -h |grep shm
tmpfs           3.9G     0  3.9G   0% /dev/shm
[root@xxx-f6f865646-l54tq /]# dd if=/dev/zero of=/dev/shm/test bs=1024 count=204800
204800+0 records in
204800+0 records out
209715200 bytes (210 MB) copied, 0.366529 s, 572 MB/s
[root@xxx-f6f865646-l54tq /]# ls -l -h /dev/shm/test
-rw-r--r-- 1 root root 200M Dec  4 14:52 /dev/shm/test
[root@xxx-f6f865646-l54tq /]# command terminated with exit code 137
$ kubectl get pods -o wide |grep xxx
xxx-f6f865646-8ll9p                       1/1     Running   0          10s     10.244.7.110    optiplex-1   <none>           <none>
xxx-f6f865646-l54tq                       0/1     Evicted   0          4m59s   <none>          optiplex-3   <none>           <none>
```

从上面到操作记录可以看到，共享内存 `/dev/shm` 可以写超过64MB大小到文件，但是如果写的文件超过sizeLimit，则一段时间后（1-2分钟）会被kubelet evict掉。之所以不是“立即”被evict，是因为kubelet是定期进行检查的，这里会有一个时间差。

### 使用empty dir的缺点

上面使用memory介质的临时存储，可以解决共享内存最大只能64MB的问题，但是它是有缺点的。

列举如下：

- 不能及时禁止用户使用内存。虽然过1-2分钟kubelet会将Pod挤出，但是这个时间内，其实对node还是有风险的；
- 影响kubernetes调度，因为empty dir并不涉及node的resources，这样会造成Pod“偷偷”使用了node的内存，但是调度器并不知晓；
- 用户不能及时感知到内存不可用

关于第二点，如下所示，由于Deployment配置中没有设置request和limit，所以node上显示该Pod的内存Request和limits都是0，但是前面设置的1GiB 的memroy临时存储，并没有反映在这里，因此会影响调度器调度，造成调度器“高估”了该node的内存使用情况。

```
$ kubectl describe nodes optiplex-1
...
Non-terminated Pods:         (11 in total)
  Namespace                  Name                                      CPU Requests  CPU Limits  Memory Requests  Memory Limits  AGE
  ---------                  ----                                      ------------  ----------  ---------------  -------------  ---
  default                    xxx-f6f865646-8ll9p                       0 (0%)        0 (0%)      0 (0%)           0 (0%)         7m3s
```

要解决这些缺点，我们需要先回到cgroup。

### cgroup限制

#### 内存的cgroup限制

下面是一个c的小程序，代码中会申请 number 个 size 大小的内存，并将该内存memset为0。注意必须memset，否则malloc只是“宣称”申请了内存，内核并不会进行虚拟地址到物理内存地址的映射。

```
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>

int main () {
   int i, n, size;
   int *a;

   printf("Number of calloc: ");
   scanf("%d", &n);
   printf("Size of calloc(in bytes): ");
   scanf("%d", &size);

   int** x = calloc(n, sizeof(int*));
   for (i=0; i< n; i++) {
     x[i] = (int*)malloc(size);
     if (x[i] == NULL) {
       perror("malloc failed");
       return -1;
     }
     memset(x[i], 0, size);
     printf("calloc %d bytes memory.\r\n", size);
   }
   printf("done\r\n");

   printf("pause\r\n");
   pause();
}
```

我们为容器设置内存的limit，在启动后将上面编译到的二进制文件calloc拷贝到容器中去。

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: xxx-cgroup-calloc
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: xxx
  template:
    metadata:
      labels:
        app: xxx
    spec:
      containers:
      - args:
        - infinity
        command:
        - sleep
        image: centos:7
        name: centos
        resources:
          limits:
            memory: 256Mi
```

容器启动后，我们先进到容器中看一下当前的cgroup的配置。

```
[root@xxx-cgroup-calloc-5b998f5897-b7xn6 memory]# cat /sys/fs/cgroup/memory/memory.limit_in_bytes
268435456
[root@xxx-cgroup-calloc-5b998f5897-b7xn6 memory]# cat /sys/fs/cgroup/memory/memory.usage_in_bytes
2056192
```

可以看到，cgroup中配置的内存上限为`memory.limit_in_bytes`为256MB，而当前已用内存`memory.usage_in_bytes`大约为2MB，因为容器中启动了sleep进程、bash进程等，会消耗一部分内存。

我们先用上面的calloc小程序申请64MB内存。

在控制台1中申请64MB内存，并且不要退出calloc进程。

```
[root@xxx-cgroup-calloc-5b998f5897-b7xn6 memory]# /bin/calloc
Number of calloc: 1
Size of calloc(in bytes): 67108864
calloc 67108864 bytes memory.
done
pause
```

在控制台2中查看当前已用内存。

```
[root@xxx-cgroup-calloc-5b998f5897-b7xn6 memory]# cat /sys/fs/cgroup/memory/memory.usage_in_bytes
69464064
```

可以看到，已用内存增加大约67108864字节。

那么，cgroup什么时候会行使自己的权力呢？

我们在控制台1上停止当前的calloc，并再次申请256MB内存。

```
[root@xxx-cgroup-calloc-5b998f5897-b7xn6 memory]# /bin/calloc
Number of calloc: 1
Size of calloc(in bytes): 268435456
Killed
```

可以看到，calloc进程直接被杀死了。是被谁杀死的呢？

我们去pod所在的node上去看下系统日志，可以找到oom killer工作的痕迹。

```
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276093] calloc invoked oom-killer: gfp_mask=0x14000c0(GFP_KERNEL), nodemask=(null), order=0, oom_score_adj=968
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276094] calloc cpuset=beead52e53d4918e7b2820c2712151f2833e4055bc72abc9ea287a02459fc55a mems_allowed=0
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276097] CPU: 0 PID: 28285 Comm: calloc Not tainted 4.15.0-63-generic #72-Ubuntu
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276097] Hardware name: QEMU Standard PC (i440FX + PIIX, 1996), BIOS rel-1.12.1-0-ga5cab58e9a3f-prebuilt.qemu.org 04/01/2014
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276098] Call Trace:
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276103]  dump_stack+0x63/0x8e
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276104]  dump_header+0x71/0x285
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276106]  oom_kill_process+0x21f/0x420
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276107]  out_of_memory+0x2b6/0x4d0
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276108]  mem_cgroup_out_of_memory+0xbb/0xd0
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276109]  mem_cgroup_oom_synchronize+0x2e8/0x320
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276111]  ? mem_cgroup_css_online+0x40/0x40
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276112]  pagefault_out_of_memory+0x36/0x7b
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276114]  mm_fault_error+0x90/0x180
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276115]  __do_page_fault+0x46b/0x4b0
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276116]  ? __schedule+0x256/0x880
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276117]  do_page_fault+0x2e/0xe0
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276118]  ? async_page_fault+0x2f/0x50
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276120]  do_async_page_fault+0x51/0x80
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276121]  async_page_fault+0x45/0x50
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276122] RIP: 0033:0x7f129f50dfe0
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276123] RSP: 002b:00007ffcbafce968 EFLAGS: 00010206
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276124] RAX: 00007f128f47e010 RBX: 0000563b4b7df010 RCX: 00007f129f1cb000
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276124] RDX: 00007f129f47e000 RSI: 0000000000000000 RDI: 00007f128f47e010
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276125] RBP: 00007ffcbafce9a0 R08: ffffffffffffffff R09: 0000000010000000
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276125] R10: 0000000000000022 R11: 0000000000001000 R12: 0000563b49a997c0
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276125] R13: 00007ffcbafcea80 R14: 0000000000000000 R15: 0000000000000000
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276126] Task in /kubepods/burstable/pod465a9fc1-0de9-44cd-b22c-7e6b23fd75f8/beead52e53d4918e7b2820c2712151f2833e4055bc72abc9ea287a02459fc55a killed as a result of limit of /kubepods/burstable/pod465a9fc1-0de9-44cd-b22c-7e6b23fd75f8
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276129] memory: usage 262144kB, limit 262144kB, failcnt 131
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276130] memory+swap: usage 0kB, limit 9007199254740988kB, failcnt 0
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276130] kmem: usage 1704kB, limit 9007199254740988kB, failcnt 0
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276131] Memory cgroup stats for /kubepods/burstable/pod465a9fc1-0de9-44cd-b22c-7e6b23fd75f8: cache:0KB rss:0KB rss_huge:0KB shmem:0KB mapped_file:0KB dirty:0KB writeback:0KB inactive_anon:0KB active_anon:0KB inactive_file:0KB active_file:0KB unevictable:0KB
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276136] Memory cgroup stats for /kubepods/burstable/pod465a9fc1-0de9-44cd-b22c-7e6b23fd75f8/d081810d182504b5c3c38e0a28e16467385abda8553e286948f6cc5b3849ca0b: cache:0KB rss:40KB rss_huge:0KB shmem:0KB mapped_file:0KB dirty:0KB writeback:0KB inactive_anon:0KB active_anon:40KB inactive_file:0KB active_file:0KB unevictable:0KB
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276141] Memory cgroup stats for /kubepods/burstable/pod465a9fc1-0de9-44cd-b22c-7e6b23fd75f8/beead52e53d4918e7b2820c2712151f2833e4055bc72abc9ea287a02459fc55a: cache:0KB rss:260400KB rss_huge:0KB shmem:0KB mapped_file:0KB dirty:0KB writeback:0KB inactive_anon:0KB active_anon:260396KB inactive_file:0KB active_file:0KB unevictable:0KB
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276145] [ pid ]   uid  tgid total_vm      rss pgtables_bytes swapents oom_score_adj name
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276203] [ 8111]     0  8111      256        1    32768        0          -998 pause
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276204] [ 8255]     0  8255     1093      146    57344        0           968 sleep
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276205] [14119]     0 14119     2957      710    65536        0           968 bash
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276206] [21462]     0 21462     2958      690    69632        0           968 bash
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276207] [28285]     0 28285    66627    65119   577536        0           968 calloc
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276208] Memory cgroup out of memory: Kill process 28285 (calloc) score 1955 or sacrifice child
Dec  4 15:24:33 optiplex-2 kernel: [6686782.276513] Killed process 28285 (calloc) total-vm:266508kB, anon-rss:259280kB, file-rss:1196kB, shmem-rss:0kB
Dec  4 15:24:33 optiplex-2 kernel: [6686782.289037] oom_reaper: reaped process 28285 (calloc), now anon-rss:0kB, file-rss:0kB, shmem-rss:0kB
```

我们经常会在一些内存紧张的服务器上看到oom killer杀死进程，但其实oom killer是以cgroup为单位进行工作的，比如，上面的cgroup就是`/kubepods/burstable/pod465a9fc1-0de9-44cd-b22c-7e6b23fd75f8/`。在这个cgroup中，当出现缺页中断（do_page_fault）时，oom killer会在**该cgroup**中查找所有进程，并对他们进行打分，打分过程会参考oom_score_adj，它是一个优先级系数，值越小，优先级越高。

从日志中可以看到，除了pause进程（即Pod中的sandbox container，cgroup和namespace的创建者，优先级为-998，优先级最高），其他进程均为968，因此打分更多的是看进程使用的内存。

由于calloc进程使用的进程最多，所以其score最高（1955），因此oom killer会杀死该进程，也就是上面我们所看到的calloc进程被“Killed”。

这也是kubernetes的memory limit是怎么生效的原理。

#### 共享内存受cgroup限制吗？

回到我们的问题。

由于普通的内存是受cgroup限制的，那么自然会有一个问题：共享内存受cgroup限制吗？如果受限制，那么只要给Pod设置memroy limit就可以了，至于用户将内存是用于共享内存，还是用于普通用途，kubernetes层面并不需要台关心：因为共享内存并不是一个“独立”的内存，只是包含在memory limit里进行调度，因此也不会影响调度。

我们为deployment xxx增加 memory limit，继续进行测试。由于我们使用POSIX方式来操作共享内存，因此直接通过dd就可以进行测试。

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: xxx
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: xxx
  template:
    metadata:
      labels:
        app: xxx
    spec:
      containers:
      - args:
        - infinity
        command:
        - sleep
        image: centos:7
        name: centos
        resources:
          limits:
            memory: 256Mi
        volumeMounts:
        - mountPath: /dev/shm
          name: cache-volume
      volumes:
      - emptyDir:
          medium: Memory
          sizeLimit: 128Mi
        name: cache-volume
```

我们到容器中，通过dd在`/dev/shm`中创建、删除一个100MB的文件，查看`memory.usage_in_bytes`可以看到过程中共享内存的使用，是反映到cgroup中去的。

```
$ kubectl exec -it xxx-789569d8c8-8pls5 bash
[root@xxx-789569d8c8-8pls5 /]# cat /sys/fs/cgroup/memory/memory.limit_in_bytes
268435456
[root@xxx-789569d8c8-8pls5 /]# df -h |grep shm
tmpfs           3.9G     0  3.9G   0% /dev/shm
[root@xxx-789569d8c8-8pls5 /]# dd if=/dev/zero of=/dev/shm/test      bs=1024 count=102400
102400+0 records in
102400+0 records out
104857600 bytes (105 MB) copied, 0.156172 s, 671 MB/s
[root@xxx-789569d8c8-8pls5 /]# ls -l -h /dev/shm/test
-rw-r--r-- 1 root root 100M Dec  4 15:43 /dev/shm/test
[root@xxx-789569d8c8-8pls5 /]# df -h |grep sh
tmpfs           3.9G  100M  3.8G   3% /dev/shm
[root@xxx-789569d8c8-8pls5 /]# cat /sys/fs/cgroup/memory/memory.usage_in_bytes
106643456
[root@xxx-789569d8c8-8pls5 /]# rm /dev/shm/test
rm: remove regular file '/dev/shm/test'? y
[root@xxx-789569d8c8-8pls5 /]# cat /sys/fs/cgroup/memory/memory.usage_in_bytes
1654784
[root@xxx-789569d8c8-8pls5 /]# dd if=/dev/zero of=/dev/shm/test      bs=1024 count=409600
Killed
[root@xxx-789569d8c8-8pls5 /]# ls
bash: /usr/bin/ls: Argument list too long
```

需要注意的是，当最后创建一个大于 memory limit的文件的时候，会发现dd进程会被杀死，这也符合cgroup的限制，但是这时会引起一个问题：由于内存都被共享内存耗尽了，任何命令都无法执行。

不过不要担心，由于共享内存的使用情况超过了设置的`sizeLimit`，过1-2分钟，会被kubelet evict掉，重新拉起新的pod，新的pod可以正常运行。

所以，参考kvm和物理机的情况，我们可以将`sizeLimit`总是设置为memory limit的50%，即使用户使用共享内存时，超出了memory limit，也会被kubelet发现而evict掉。

### 最终的设计

综合上面的分析，基于kubernetes的平台，“共享内存”特性设计如下：

1. 将memory介质的`emptyDir`挂载到`/dev/shm/`
2. 配置container的`memory limit`
3. 配置memory介质的`emptyDir`的`sizeLimit`为 `memory limit`的50%

进一步，上述配置可以在admission controller中自动为用户配置，用户不需要显式的“请求开启共享内存”。

Ref:

- [Redhat: MEMORY](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/resource_management_guide/sec-memory)
- [POSIX shared-memory API](https://www.geeksforgeeks.org/posix-shared-memory-api/)
- [IPC through shared memory](https://www.geeksforgeeks.org/ipc-shared-memory/)
- [DOCKER基础技术：LINUX CGROUP](https://coolshell.cn/articles/17049.html)
- [两种Linux共享内存](http://blog.jqian.net/post/linux-shm.html)
- [Kubernetes resource limits and kernel cgroups](https://medium.com/@kkwriting/kubernetes-resource-limits-and-kernel-cgroups-337625bab87d)

**Tags:** [kubernetes](https://ieevee.com/tag/kubernetes)