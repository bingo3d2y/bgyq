## Heap and Stack

### Summary

**堆区用于分配程序员申请的内存空间。**

> so 一般是堆溢出-

**Stack：可以想象下，程序的调用是通过stack来完成的，所以是由运行时系统kernal控制的，因此不是程序员可以自主申请的。**

linux用户空间虚拟地址空间主要包括以下几个方面：

1. stack区(进程栈，从上往下分配，大小上限一般为128M)。
2. mmap区(用于各种映射：private，share，anonymous，是文件映射的主要区域)。
3. heap区(用于anonymous匿名映射，是用户内存分配的主要区域)。
4. 数据段、代码段之类的，不做详细介绍。



\1. 堆没有方向之说，每个堆都是散落的

\2. 堆和栈之间没有谁地址高之说，看操作系统实现

### Stack

栈又称堆栈，由编译器自动分配释放，行为类似数据结构中的栈(先进后出)。堆栈主要有三个用途：

- 为函数内部声明的非静态局部变量(C语言中称“自动变量”)提供存储空间。

- 记录函数调用过程相关的维护性信息，称为栈帧(Stack Frame)或过程活动记录(Procedure Activation Record)。它包括函数返回地址，不适合装入寄存器的函数参数及一些寄存器值的保存。除递归调用外，堆栈并非必需。因为编译时可获知局部变量，参数和返回地址所需空间，并将其分配于BSS段。

- 临时存储区，用于暂存长算术表达式部分计算结果或alloca()函数分配的栈内内存。

  持续地重用栈空间有助于使活跃的栈内存保持在CPU缓存中，从而加速访问。进程中的每个线程都有属于自己的栈。向栈中不断压入数据时，若超出其容a量就会耗尽栈对应的内存区域，从而触发一个页错误。此时若栈的大小低于堆栈最大值RLIMIT_STACK(通常是8M)，则栈会动态增长，程序继续运行。映射的栈区扩展到所需大小后，不再收缩。

  Linux中ulimit -s命令可查看和设置堆栈最大值，当程序使用的堆栈超过该值时, 发生栈溢出(Stack Overflow)，程序收到一个段错误(Segmentation Fault)。注意，调高堆栈容量可能会增加内存开销和启动时间。

  堆栈既可向下增长(向内存低地址)也可向上增长, 这依赖于具体的实现。本文所述堆栈向下增长。

  栈的大小在运行时由内核动态调整。

#### caller and callee

caller： **a** visitor ，函数的调用者

A() 调用 B()， 则在B()中 B.caller 指向A()

callee：被调用的函数

函数调用栈是指程序运行时内存一段连续的区域，用来保存函数运行时的状态信息，包括函数参数与局部变量等。称之为“栈”是因为发生函数调用时，调用函数（caller）的状态被保存在栈内，被调用函数（callee）的状态被压入调用栈的栈顶；在函数调用结束时，栈顶的函数（callee）状态被弹出，栈顶恢复到调用函数（caller）的状态。Linux i386架构的机器函数调用栈在内存中从高地址向低地址生长，所以栈顶对应的内存地址在压栈时变小，退栈时变大。

##### 入参顺序

stack frame入参顺序是从右到左，即`swap(a, b int)`的入参顺序是，先入参`b`再入参`a`。

因为这样方便寻址`sp+8`就是`a`，`sp+16`就是`b`。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/func-stack-frame.jpg)

#### 闭包closure 

**核心：就是保持捕获变量在外层函数和闭包函数中的一致性。**

每个函数都有一个function value存储了callee的地址，有func作为返回值的函数返回函数地址时，会将执行函数地址的function value变量赋值给函数变量(function value)。这样当callee函数中有捕获列表（即闭包）时，既可以通过偏移量获取到捕获变量的值。

下图的stack frame比较简单，因为不涉及到闭包变量的修改

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/golang-closure.jpg)

每次有捕获列表(caption)的callee调用时，总会访问自己的独立的caption value，所以，闭包又被称作有状态的函数。那么怎么保障caller和callee操作的是一个变量呢？

答案其实很简单，就是利用指针：

* 在stack frame声明一个指针变量，指向caption value
* 在heap上声明caption value

这就是闭包导致的变量堆分配，也是变量逃逸的一种。

**Go逃逸分析最基本的原则是：如果一个函数返回对一个变量的引用，那么它就会发生逃逸。**

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/golang-closure-2.jpg)



```go
func main() {
	a := avrage()
	a(4)
	a(3)
	r := a(4)
	fmt.Println(r)

}

func avrage() func(tmp float32) float32{
	var num float32
	num = 0.0
	f := func(tmp float32) float32{
		num = (num + tmp) /2
		return  num
	}
	return f
}
```

end

##### caption is args

如果，闭包捕获的变量是参数，caller正常入参，然后闭包的外层函数将栈上的arg拷贝到heap上一份，然后callee和funticon value(闭包函数)都使用heap上的arg。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/内存逃逸-assembly-2.jpg)

##### caption is ret

如果被捕获的是返回值，则caller栈帧上依然回分配返回值ret空间。但是闭包的外层函数会在堆上也分配一个ret空间，闭包和闭包外层函数都使用heap上的ret，最后在外部函数返回前会将ret of heap拷贝到function frame上一份。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/内存逃逸-assembly-3.jpg)

#### stack frame

（stack frame）又常被称为帧（frame）.一个栈是由很多帧构成的,它描述了函数之间的调用关系.每一帧就对应了一次尚未返回的函数调用,帧本身也是以栈的形式存放数据的.

#### 栈工作原理

栈存在的意义是什么？从数据结构的角度来看，栈是一种先进后出的效果，这样的逻辑特性与函数调用之间的关系是有相似之处的。假设有函数A，B，C，调用关系为A->B->C，假如我们用一个栈来记录这个调用过程的话，栈底当然是函数A，而栈顶则是函数C了。这只是一个很粗略的想法，函数当然需要参数，调用方要传入*实参*，被调用方要用形参来接收；除此之外我们还需要记载一下调用方调用函数时，代码执行到了什么位置等等。这些所有的信息，都需要使用栈来进行记录，而操作栈的则是一些**寄存器**和**汇编指令**。

 其中最重要的两个寄 存器是ebp和esp(32位)，ebp(extended base pointer)，即扩展的基址指针寄存器，听名字就知道是干啥了的，esp(extended stack pointer)，即扩展的栈顶指针寄存器。这两个寄存器是配对使用的，esp在扩展空间后会减小(栈是逆向生长的哈)，但是ebp会始终指向栈的底部，为啥要指在底部不动？因为esp是变化的量，如果你要访问栈中的数据，当然是使用一个不变的量**esp+偏移地址**要方便的多。

 当你要初始化一个局部变量的时候，esp指针就会向下开辟空间，再将你的数据移入，之后的访问会使用ebp+变量的大小进行访问。而当你要调用一个函数时，会进行以下步骤：

- 先将你的实参以**从右到左**的方式压入栈中
- 将下一条语句的地址压入栈中
- 将当前ebp压入栈中

 而当子函数执行完毕后，就会将子函数的esp重新指向子函数的ebp，再弹出ebp恢复父函数的ebp，接着弹出返回地址到eip继续执行父函数的代码。最终的结构大概是这样的(main函数中调用子函数foo(3, 4))：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/func-and-stackframe.png)



#### 栈溢出的危害

假如在上图的子函数中存在需要用户输入的变量而没有加以控制其大小，那么就是危险的。首先用户输入到栈中的变量是从低地址到高地址生长的，也就是下面图这样：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/stack-overflow.png)

 假如这个name数组的大小为10字节，如果你输入的是一个超过10字节的变量呢？在没有任何保护机制的情况下，就会向高地址生长，覆盖掉其他变量，覆盖掉ebp，甚至返回地址的值，这样就可以实现**篡改其他只读变量**或者**函数返回地址来控制程序流程**的目的，与之相关的技术层出不穷，当然现有的防护机制也是有的，如DEP保护，Canary金丝雀机制，ASLR机制，影子栈等等。

##### Golang处理

再编译阶段就能确定好整个stack

Golang的处理是当函数需要更大的栈帧时，不会就地扩容，而是重新申请一块更大的内存地址，再将旧的栈帧数据拷贝到新的栈帧。

### Heap

在程序运行过程中，**堆可以提供动态分配的内存**，允许程序申请大小未知的内存。堆其实就是程序虚拟地址空间的一块连续的线性区域，它由低地址向高地址方向增长。我们一般称管理堆的那部分程序为堆管理器。

开发时可使用 malloc(), calloc(), realloc() 进行内存的分配，并使用 free() 进行内存的销毁。



### 协程：究极应用

协程，用线程的堆内存模拟栈内存，再通过协程栈和线程栈的不断切换，实现多协程操作。

需要了解下栈内存空间的布局，再来Linux系统下栈的生长方向是从高地址往低地址。我们只要找到栈的栈顶和栈底的地址，就可以找到整个栈内存空间了。

在coroutine中，因为协程的运行时栈的内存空间是自己分配的。在coroutine_resume阶段设置了

`C->ctx.uc_stack.ss_sp = S.S->stack`。根据以上理论，栈的生长方向是高地址到低地址，因此栈底的就是内存地址最大的位置，即`S->stack + STACK_SIZE`就是栈底位置。

这里特意用到了一个dummy变量，这个dummy的作用非常关键也非常巧妙，大家可以细细体会下。因为dummy变量是刚刚分配到栈上的，此时就位于**栈的最顶部位置**。整个内存布局如下图所示：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/stack-frame.jpg)

因此整个栈的大小就是从栈底到栈顶，`S->stack + STACK_SIZE - &dummy`。

这个dummy是在要切出协程栈的时候才分配的变量，肯定指向 模拟的stack frame栈底。

这就是最简单的coroutinue的实现

```c
struct coroutine {
    coroutine_func func; // 协程所用的函数
    void *ud;  // 协程参数
    ucontext_t ctx; // 协程上下文
    struct schedule * sch; // 该协程所属的调度器
    ptrdiff_t cap;   // 已经分配的内存大小
    ptrdiff_t size; // 当前协程运行时栈，保存起来后的大小
    int status; // 协程当前的状态
    char *stack; // 当前协程的保存起来的运行时栈
};

static void
_save_stack(struct coroutine *C, char *top) {
    // 这个dummy很关键，是求取整个栈的关键.以为dummy刚入栈，所以肯定在栈底。
    // 这个非常经典，涉及到linux的内存分布，栈是从高地址向低地址扩展，因此
    // S->stack + STACK_SIZE就是运行时栈的栈底
    // dummy，此时在栈中，肯定是位于最底的位置的，即栈顶
    // top - &dummy 即整个栈的容量
    char dummy = 0;
    assert(top - &dummy <= STACK_SIZE);
    // 如果已分配内存小于当前栈的大小，则释放内存重新分配
    if (C->cap < top - &dummy) {
        free(C->stack);
        C->cap = top-&dummy;
        C->stack = malloc(C->cap);
    }
    C->size = top - &dummy;
    // 从dummy拷贝size内存到C->stack
    memcpy(C->stack, &dummy, C->size);
}
```

end

### 引用

1. 幼麟实验室
2. 