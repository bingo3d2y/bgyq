## Golang 汇编

### callee and caller

caller： **a** visitor ，函数的调用者

A() 调用 B()， 则在B()中 B.caller 指向A()

callee：被调用的函数

“栈”是函数调用栈，是以“栈帧”（stack frame）为单位的。

每一次函数调用会在栈上分配一个新的栈帧，在这次函数调用结束时释放其空间。

函数调用栈是指程序运行时内存一段连续的区域，用来保存函数运行时的状态信息，包括函数参数与局部变量等。称之为“栈”是因为发生函数调用时，调用函数（caller）的状态被保存在栈内，被调用函数（callee）的状态被压入调用栈的栈顶；在函数调用结束时，栈顶的函数（callee）状态被弹出，栈顶恢复到调用函数（caller）的状态。Linux i386架构的机器函数调用栈在内存中从高地址向低地址生长，所以栈顶对应的内存地址在压栈时变小，退栈时变大。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/golang-callee-and-caller.jpg)

下图更详细

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/golang-caller-and-callee.webp)

return addr 也是在 caller 的栈上的，不过往栈上插 return addr 的过程是由 CALL 指令完成的（在分析汇编时，是看不到关于 addr 相关空间信息的。在分配栈空间时，addr 所占用空间大小不包含在栈帧大小内）。

### Go 伪寄存器

go 汇编中有 4 个核心的伪寄存器，这 4 个寄存器是编译器用来维护上下文、特殊标识等作用的：

| 寄存器                  | 说明                 |
| ----------------------- | -------------------- |
| SB(Static base pointer) | global symbols       |
| FP(Frame pointer)       | arguments and locals |
| PC(Program counter)     | jumps and branches   |
| SP(Stack pointer)       | top of stack         |

- FP: 使用如 `symbol+offset(FP)`的方式，引用 callee 函数的入参参数。例如 `arg0+0(FP)，arg1+8(FP)`，使用 FP 必须加 symbol ，否则无法通过编译(从汇编层面来看，symbol 没有什么用，加 symbol 主要是为了提升代码可读性)。另外，需要注意的是：往往在编写 go 汇编代码时，要站在 callee 的角度来看(FP)，在 callee 看来，(FP)指向的是 caller 调用 callee 时传递的第一个参数的位置。假如当前的 callee 函数是 add，在 add 的代码中引用 FP，该 FP 指向的位置不在 callee 的 stack frame 之内。而是在 caller 的 stack frame 上，指向调用 add 函数时传递的第一个参数的位置，经常在 callee 中用`symbol+offset(FP)`来获取入参的参数值。
- SB: 全局静态基指针，一般用在声明函数、全局变量中。
- SP: 该寄存器也是最具有迷惑性的寄存器，因为会有伪 SP 寄存器和硬件 SP 寄存器之分。plan9 的这个伪 SP 寄存器指向当前栈帧第一个局部变量的结束位置(为什么说是结束位置，可以看下面寄存器内存布局图)，使用形如 symbol+offset(SP) 的方式，引用函数的局部变量。offset 的合法取值是 [-framesize, 0)，注意是个左闭右开的区间。假如局部变量都是 8 字节，那么第一个局部变量就可以用 localvar0-8(SP) 来表示。与硬件寄存器 SP 是两个不同的东西，在栈帧 size 为 0 的情况下，伪寄存器 SP 和硬件寄存器 SP 指向同一位置。手写汇编代码时，如果是 symbol+offset(SP)形式，则表示伪寄存器 SP。如果是 offset(SP)则表示硬件寄存器 SP。`务必注意`：对于编译输出(go tool compile -S / go tool objdump)的代码来讲，所有的 SP 都是硬件 SP 寄存器，无论是否带 symbol（这一点非常具有迷惑性，需要慢慢理解。往往在分析编译输出的汇编时，看到的就是硬件 SP 寄存器）。
- PC: 实际上就是在体系结构的知识中常见的 pc 寄存器，在 x86 平台下对应 ip 寄存器，amd64 上则是 rip。除了个别跳转之外，手写 plan9 汇编代码时，很少用到 PC 寄存器。
  通过上面的讲解，想必已经对 4 个核心寄存器的区别有了一定的认识（或者是更加的迷惑、一头雾水）。那么，需要留意的是：如果是在分析编译输出的汇编代码时，要重点看 SP、SB 寄存器（FP 寄存器在这里是看不到的）。如果是，在手写汇编代码，那么要重点看 FP、SP 寄存器。

### 程序内存布局

- 代码段:存储程序指令，位于内存最低端
- 数据段（初始化数据段）：全局变量或者静态变量，数据段分只读区和读写区。
- BSS段（未初始化数据段）：未初始化全局变量
- 栈：一种LIFO结构，从高地址向低地址增长。
- 堆：动态分布内存区，从低向高增长。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/func-mem-layout.png)

### Plan9 assembly

Go 使用了plan9 汇编。

intel 或 AT&T 汇编提供了 push 和 pop 指令族，plan9 中没有 push 和 pop，栈的调整是通过对硬件 SP 寄存器进行运算来实现的，例如:

```go
SUBQ $0x18, SP // 对 SP 做减法，为函数分配函数栈帧
...               // 省略无用代码
ADDQ $0x18, SP // 对 SP 做加法，清除函数栈帧
```

plan9 的汇编的操作数的方向是和 intel 汇编相反的，与 AT&T 类似。

```go
MOVQ $0x10, AX ===== mov rax, 0x10
       |    |------------|      |
       |------------------------|
```

不过凡事总有例外，如果

通用的指令和 IA64 平台差不多，下面是通用通用寄存器的名字在 IA64 和 plan9 中的对应关系:

| IA64  | rax  | rbx  | rcx  | rdx  | rdi  | rsi  | rbp  | rsp  | r8   | r9   | r10  | r11  | r12  | r13  | r14  | rip  |
| ----- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| Plan9 | AX   | BX   | CX   | DX   | DI   | SI   | BP   | SP   | R8   | R9   | R10  | R11  | R12  | R13  | R14  | PC   |

#### 常用指令

下面列出了常用的几个汇编指令（指令后缀`Q` 说明是 64 位上的汇编指令）

| 助记符  | 指令种类 | 用途           | 示例                                     |
| ------- | -------- | -------------- | ---------------------------------------- |
| `MOVQ`  | 传送     | 数据传送       | MOVQ 48, AX // 把 48 传送到 AX           |
| `LEAQ`  | 传送     | 地址传送       | LEAQ AX, BX // 把 AX 有效地址传送到 BX   |
| `PUSHQ` | 传送     | 栈压入         | PUSHQ AX // 将 AX 内容送入栈顶位置       |
| `POPQ`  | 传送     | 栈弹出         | POPQ AX // 弹出栈顶数据后修改栈顶指针    |
| `ADDQ`  | 运算     | 相加并赋值     | ADDQ BX, AX // 等价于 AX+=BX             |
| `SUBQ`  | 运算     | 相减并赋值     | SUBQ BX, AX // 等价于 AX-=BX             |
| `CMPQ`  | 运算     | 比较大小       | CMPQ SI CX // 比较 SI 和 CX 的大小       |
| `CALL`  | 转移     | 调用函数       | CALL runtime.printnl(SB) // 发起调用     |
| `JMP`   | 转移     | 无条件转移指令 | JMP 0x0185 //无条件转至 0x0185 地址处    |
| `JLS`   | 转移     | 条件转移指令   | JLS 0x0185 //左边小于右边，则跳到 0x0185 |

### 神奇的编译器优化...

```go
// v1 
package main

type Student struct {
	Class int
}

func main() {
	var a = &Student{1}
	println(a)
}

// v2
package main

type Student struct {
	Class int
}

func main() {
	var a = Student{1}
	var b = &a
	println(b)
}
```

这两段代码的执行效率竟然是一样的....

```bash
# 不能添加 -N
# -N    disable optimizations，禁止编译器优化
$ go tool compile -S v1.go
...
"".main STEXT size=80 args=0x0 locals=0x18 funcid=0x0
        0x0000 00000 (main.go:7)        TEXT    "".main(SB), ABIInternal, $24-0
        0x0000 00000 (main.go:7)        CMPQ    SP, 16(R14)
        0x0004 00004 (main.go:7)        PCDATA  $0, $-2
        0x0004 00004 (main.go:7)        JLS     73
        0x0006 00006 (main.go:7)        PCDATA  $0, $-1
        0x0006 00006 (main.go:7)        SUBQ    $24, SP
        0x000a 00010 (main.go:7)        MOVQ    BP, 16(SP)
        0x000f 00015 (main.go:7)        LEAQ    16(SP), BP
        0x0014 00020 (main.go:7)        FUNCDATA        $0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x0014 00020 (main.go:7)        FUNCDATA        $1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x0014 00020 (main.go:8)        MOVQ    $0, ""..autotmp_2+8(SP)
        0x001d 00029 (main.go:8)        MOVQ    $1, ""..autotmp_2+8(SP)
        0x0026 00038 (main.go:9)        PCDATA  $1, $0
        0x0026 00038 (main.go:9)        CALL    runtime.printlock(SB)
        0x002b 00043 (main.go:9)        LEAQ    ""..autotmp_2+8(SP), AX
        0x0030 00048 (main.go:9)        CALL    runtime.printpointer(SB)
        0x0035 00053 (main.go:9)        CALL    runtime.printnl(SB)
        0x003a 00058 (main.go:9)        CALL    runtime.printunlock(SB)
        0x003f 00063 (main.go:10)       MOVQ    16(SP), BP
        0x0044 00068 (main.go:10)       ADDQ    $24, SP
        0x0048 00072 (main.go:10)       RET
        0x0049 00073 (main.go:10)       NOP
        0x0049 00073 (main.go:7)        PCDATA  $1, $-1
        0x0049 00073 (main.go:7)        PCDATA  $0, $-2
        0x0049 00073 (main.go:7)        CALL    runtime.morestack_noctxt(SB)
        0x004e 00078 (main.go:7)        PCDATA  $0, $-1
        0x004e 00078 (main.go:7)        JMP     0

...
$ go tool compile -S v2.go
"".main STEXT size=80 args=0x0 locals=0x18 funcid=0x0
        0x0000 00000 (main.go:7)        TEXT    "".main(SB), ABIInternal, $24-0
        0x0000 00000 (main.go:7)        CMPQ    SP, 16(R14)
        0x0004 00004 (main.go:7)        PCDATA  $0, $-2
        0x0004 00004 (main.go:7)        JLS     73
        0x0006 00006 (main.go:7)        PCDATA  $0, $-1
        0x0006 00006 (main.go:7)        SUBQ    $24, SP
        0x000a 00010 (main.go:7)        MOVQ    BP, 16(SP)
        0x000f 00015 (main.go:7)        LEAQ    16(SP), BP
        0x0014 00020 (main.go:7)        FUNCDATA        $0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x0014 00020 (main.go:7)        FUNCDATA        $1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x0014 00020 (main.go:8)        MOVQ    $0, "".a+8(SP)
        0x001d 00029 (main.go:8)        MOVQ    $1, "".a+8(SP)
        0x0026 00038 (main.go:10)       PCDATA  $1, $0
        0x0026 00038 (main.go:10)       CALL    runtime.printlock(SB)
        0x002b 00043 (main.go:10)       LEAQ    "".a+8(SP), AX
        0x0030 00048 (main.go:10)       CALL    runtime.printpointer(SB)
        0x0035 00053 (main.go:10)       CALL    runtime.printnl(SB)
        0x003a 00058 (main.go:10)       CALL    runtime.printunlock(SB)
        0x003f 00063 (main.go:11)       MOVQ    16(SP), BP
        0x0044 00068 (main.go:11)       ADDQ    $24, SP
        0x0048 00072 (main.go:11)       RET
        0x0049 00073 (main.go:11)       NOP
        0x0049 00073 (main.go:7)        PCDATA  $1, $-1
        0x0049 00073 (main.go:7)        PCDATA  $0, $-2
        0x0049 00073 (main.go:7)        CALL    runtime.morestack_noctxt(SB)
        0x004e 00078 (main.go:7)        PCDATA  $0, $-1
        0x004e 00078 (main.go:7)        JMP     0
 ...
```

对比编译的结果，竟然如此相似....j简直一模一样

### 大佬的汇编demo

```go
package main

import "fmt"

const s = "Go101.org"
//len(s)==1
//1 << 9 == 512
// 512/128 =4
var num_a byte = 1 << len(s) / 128
var num_b byte = 1 << len(s[:]) / 128

func main() {
	fmt.Println(num_a, num_b)
}
```

下面看下神奇的执行结果

```bash
$ go run main.go
4 0
```

如果你不懂go汇编,你很难知道到底发生了什么? 你懂go汇编的话,你看容易在汇编代码中找到端倪.

```bash
# -N    disable optimizations
# -S    print assembly listing
# -l    disable inlining
# -l 禁止内联 -N 编译时，禁止优化 -S 输出汇编代码
$ go tool compile -l -S -N main.go
```

在汇编代码中看到

```bash
go.string."Go101.org" SRODATA dupok size=9
        0x0000 47 6f 31 30 31 2e 6f 72 67                       Go101.org
# .data段
"".num_a SNOPTRDATA size=1
        0x0000 04                                               .
# .bss段
"".num_b SNOPTRBSS size=1
```

`var num_a byte = 1 << len(s) / 128` 这段代码被go编译器(go 1.14)直接优化成num_a初始化为4的变量,num_b这没被初始化,num_b的赋值在main package的init函数中

```bash
"".init STEXT nosplit size=62 args=0x0 locals=0x20 funcid=0x0	
        ...
        0x0015 00021 (main.go:10)       MOVQ    AX, ""..autotmp_0+8(SP)
        0x001a 00026 (main.go:10)       MOVQ    $9, ""..autotmp_0+16(SP)
        0x0023 00035 (main.go:10)       MOVQ    $9, ""..autotmp_1(SP)
        0x002b 00043 (main.go:10)       JMP     45
        0x002d 00045 (main.go:10)       MOVB    $0, "".num_b(SB)
        ...
```

上面的代码很有趣,num_b被赋值为0,至于0从哪里冒出来的,完全看不到. 

我们很有理由怀疑go的编译器出bug了,bug不是因为b的结果不对,而是应该对len(s)和len(s[:])用同一套规则,假定 `1 << len(s[:])`已经 让byte类型溢出了其结果为0,那么应该`1 << len(s) `这个也应该是一样的结果.

但是编译器认为s是常量s[:]是变量,所以`len(s)`还是常量，`len(s[:])`是变量。

常量在运算过程中有隐式类型转换,`1 << len(s)`，1会变成int类型`var a byte = int(1) << len(s) / 128`。

而变量则没有，所以b为`uint8(1) << len(s[:])`的结果为0.

#### 疑惑

我使用package refelect 检测`len(s)`和`len(s[:])`的类型怎么都是int？

### 引用

0. https://xargin.com/plan9-assembly/

1. https://github.com/widaT/learning-go/blob/master/go%E6%B1%87%E7%BC%96/go%E6%B1%87%E7%BC%96%E8%BF%90%E7%94%A8.md