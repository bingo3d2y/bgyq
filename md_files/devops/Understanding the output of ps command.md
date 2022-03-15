## Understanding the output of ps command

### Overview:

When troubleshooting a system, the commands I regularly use are top and ps. They tell you a lot about the state of the system, active processes and provide information about CPU utilization for a process. This information is a good place to start when troubleshooting a system related issue.

### Understanding a process and its Lifecycle:

*A running program is called a process.* It is an instance of a program that is being executed. Every process goes through a lifecycle that involves multiple stages/states broadly.

- When a process is created, it enters a *new* or *created* state and waits to be promoted into the ready state
- When the process is loaded into the main memory, it enters the ready state. It is generally added to a queue or processes that are awaiting execution on a CPU.
- When a process is chosen by a CPU to serve, the process state is updated to running. The instructions of the process are executed by one of the CPUs. At any given time there is at most one running process on a CPU
- If the process needs to access I/O or network resources or is waiting for user input, it enters into a blocked state and the processor serves a different process, if any until the current process received what it was waiting for.
- A process is terminated either when it is done with its execution or is explicitly killed. A process that is gracefully terminated would no longer be in the process table.

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/process-lifecycle.png)

### ps command

ps stands for process status. It reports a snapshot of current processes. It gets the information being displayed from the virtual files in /proc filesystem.

The output of ps command is as follows

```
$ ps
PID        TTY     STAT   TIME          CMD 
5140     pts/4    Ss        00:00:00     bash 
61244    pts/4    R+        00:00:00     ps
```

**PID:** Every process is assigned a PID (Process Identifier) which is a unique identifier that is associated with a running process in the system.

**TTY:** Controlling terminal associated with the process.

**STAT:** Process State Code

**TIME:** Total time of CPU Usage

**CMD:** The command that is executed by the process.

I prefer using the ps command with the afx parameters (*ps afx )* as I like the tree layout that comes with using the *f* parameter. `ps aux `also comes in handy as *u* option shows more details about the process. The below details are also provided when the ps command is executed with the option.

- **RSS:** Resident set size, the non-swapped physical memory that is used by a process
- **VSZ:** Virtual memory usage of the process
- **%CPU:** CPU time used divided by the process run time.
- **%MEM:** Ratio of the process’s resident set size to the physical memory on the machine
- **START**: Start time of the process

### Process state as displayed by ps or top commands:

A Linux process can be in many states. It’s easy to observe it in the output of the`ps` command in the column named `S`. The man page of ps describes the possible values:

```bash
PROCESS STATE CODES 
  Here are the different values that the s, stat and state output specifiers (header "STAT" or "S") will display to describe the state of a process:D    uninterruptible sleep (usually IO)
I    Idle kernel thread
R    running or runnable (on run queue)
S    interruptible sleep (waiting for an event to complete)
T    stopped by job control signal
t    stopped by debugger during the tracing
W    paging (not valid since the 2.6.xx kernel)
X    dead (should never be seen)
Z    defunct ("zombie") process, terminated but not reaped by its parent

## ps aux
   For BSD formats and when the stat keyword is used, additional characters may be displayed:

               <    high-priority (not nice to other users)
               N    low-priority (nice to other users)
               L    has pages locked into memory (for real-time and custom IO)
               s    is a session leader
               l    is multi-threaded (using CLONE_THREAD, like NPTL pthreads do)
               +    is in the foreground process group

```

**Uninterruptible sleep (D):** Processes that cannot be killed or interrupted with a signal. A process enters such a state if it deals with I/O. If a process, when waiting for an I/O receives a signal to terminate, it will terminate before having the chance to handle the requested data. Since the process is blocked performing a system call, the process cannot be interrupted until the system call completes and the result is handled. When a process enters this state, there is no trivial way to kill the process except for rebooting. A process can be in this state when it is in the waiting stage in the process lifecycle.

**Idle (I):** The idle state is mainly to reduce energy consumption. In Linux, one idle task is created for every processor by default and is locked to that processor, when there’s no other process to run on a CPU, the idle task is scheduled.

**Running /Runnable(R):** Running process is currently the process that is being executed on the CPU. The runnable process is waiting in the queue and is in the ready state, meaning it has all it needs to be run and is waiting for a CPU core to execute it. The state corresponds to the running stage in the process life cycle.

**Interruptable Sleep (S):** When a process enters this state, it will wake up to handle any signals. A process usually enters this state when it is waiting for an event to complete, for instance, a user input. The event that is waiting for is not a system call if that is the case, the process enters an uninterruptable sleep state. A process can be in this state when it is in the waiting stage in the process lifecycle.

**Stopped (T):** A process enters this state when it is stopped or suspended manually. I will discuss include more details about the types of signals in the second part. This state corresponds to the terminated stage in the process life cycle.

**Defunct/Zombie (Z):** It is a process that is done executing, but still has an entry in the process table. If the child process has completed execution and is in a terminated state but the parent hasn’t read the exit status with the wait system call yet, then the child process enters the zombie state. An entry for the child still exits in the process table to facilitate the read of the exit status by its parent. Once the parent reads the exit status, the process is reaped and the child is no longer a zombie. A process can be in this state when it is in the terminated stage in the process lifecycle.

Processes that stay zombies for a long time are generally an error and they can cause a resource leak.

### idle state

idle 进程，也就是0号进程，不参与schedule机制，当系统中没有任何进程可以调度（就绪队列为空），cpu会进入该进程。多cpu系统中每个cpu一个idle。

系统空闲进程，一般优先级最低，系统没事干的时候就执行它。实际上在某些系统会在idle中处理一些内存回收之类的事情。其存在的原因是为了让调度器有事情做，要不然所有的进程全挂起了调度器会很郁闷的。

### 引用

一个系列的文章： https://medium.com/100-days-of-linux

1. https://medium.com/100-days-of-linux/understanding-the-output-of-ps-commands-e9e270a418f9
2. 