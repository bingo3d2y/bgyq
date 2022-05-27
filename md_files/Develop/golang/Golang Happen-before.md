## Golang Happen-before

### 并发： Happen-before原则

#### 问题： goroutine无法修改全局变量

代码逻辑如下，期望结果：

第一个goroutine，先启动输出running的值，两秒后，第二个goroutine启动，将running的值改完0，使得goroutine1 停止输出。

实际结果：

goroutine 1 一直在输出。。。

```go
package main

import (
	"fmt"
	"time"
)

func main()  {
	var running int32 = 1

	go func() {
		fmt.Println("Start goroutine 1")
		for running == 1 {
			fmt.Printf("Goroutine 1 is running %d\n", running)
			time.Sleep(time.Second)
		}
	}()

	go func() {
		time.Sleep(time.Second * 2)
		fmt.Println("Start goroutine 2")
		for {
			running = 0
		}
	}()
	time.Sleep(time.Hour)
}


// 一直输出...
Start goroutine 1
Goroutine 1 is running 1
Goroutine 1 is running 1
Start goroutine 2
Goroutine 1 is running 1
Goroutine 1 is running 1

...
```

end

#### 原因：Happen-before原则，保证可见性

可见性是指当一个线程修改了共享变量的值，其它线程能够适时得知这个修改。

作者：周晓军
链接：https://www.zhihu.com/question/434964023/answer/1635454280

这个应该是**内存可见性**（Memory Visibility）的问题，按照《**Java并发编程实战**》书中的所说：是指一个线程修改了对象状态后，其他线程能够看到发生的状态变化。那么在 Go 中如何来保证呢？推荐题主阅读《[The Go Memory Model](https://link.zhihu.com/?target=https%3A//golang.org/ref/mem)》或者中文版《[Go 内存模型](https://link.zhihu.com/?target=https%3A//studygolang.com/articles/30706)》，它详细介绍了在多个 goroutine 情况下，如何保证共享变量的可见性以及 **Happens Before** 原则。为了保证共享变量的可见性，该文档中的建议如下：

> Programs that modify data being simultaneously accessed by multiple goroutines must serialize such access[[3\]](#ref_3)

程序中多个 goroutine 同时访问某个数据，必须保证**串行化**访问。即需要增加同步逻辑，可以使用 [sync](https://link.zhihu.com/?target=https%3A//golang.org/pkg/sync/) 或者 [sync/atomic](https://link.zhihu.com/?target=https%3A//golang.org/pkg/sync/atomic/) 中的锁或原子类型来保证。另外，题主还可以尝试用 channel [[4\]](#ref_4) 来实现这个同步逻辑，示例代码请查看 [codewalk-sharemem](https://link.zhihu.com/?target=https%3A//golang.org/doc/codewalk/sharemem/)。通过这个代码来理解一下 Go 语言中的一个经典说法：Do not communicate by sharing memory; instead, share memory by communicating。



#### 背后： 编译器和汇编

https://www.zhihu.com/question/434964023/answer/1635454280?utm_source=wechat_session&utm_medium=social&utm_oi=1156972076551303168&utm_content=group2_Answer&utm_campaign=shareopn



#### golang：happen-before

https://zuoqy.com/2018/11/26/Golang-Memory-Model/

指令重排序对goroutine内部的读写顺序可能有影响，但不会影响代码定义的当前goroutine整体的行为。
例如在goroutine1中有代码`a = 1; b = 2;`，在其他goroutine中可能观察到b先于a赋值，但对于goroutine1，其行为不会因为指令重排序发生变化。

Go语言中的Happens-Before：Happens-Before定义了内存操作的顺序。如果e1 happens-before e2, 则 e2 一定在e1发生之后才发生。如果e1 e2 没有happens-before的关系限制，则e1 e2是并发发生的。在单个goroutine内部， happens-before 次序就是程序定义的顺序。

**hb(a,b) presents “a happens before b”**

读取操作r对变量v 可以观察到写入操作w对变量v的写入，必须同时满足如下两个条件：

1. r不能先于w发生
2. 没有其他的对变量v的写入操作w‘ 晚于 w 但 先于r发生

读取操作保证能观察到写入操作w对变量v的写入，必须同时满足如下两个条件：

1. w 先于 r发生
2. 任何其他对变量v的写入操作，要么先于w发生，要么晚于r发生。

初始化变量v为其对应类型的零值，在此内存模型视为写入操作。读取或者写入长于一个机器字长的值，会被分为多个机器字长、执行顺序不固定的读取或者写入操作.

常见的Happens-before：

* The go statement that starts a new goroutine **happens before** the goroutine’s execution begins.

  goroutine前的go语句，会`hb(go statement, new goroutine)`

  ```go
  // 在未来的某个时刻执行f() 可能是在hello()或者main() 返回之后
  package main
  
  import (
  	"fmt"
  	"time"
  )
  
  func f(a string) {
      // 故意制造延时，观察goroutine是否先执行
      time.Sleep(time.Second * 1)
  	fmt.Println(a)
  }
  
  func hello(a string)  {
  	go f(a)
  }
  
  func main()  {
  	var a string = "test"
  	hello(a)
  	time.Sleep(time.Second * 3) // 不加 sleep，主程序退出，goroutine无法打印输出。
  }
  
  ```

* A send on a channel **happens before** the corresponding receive from that channel completes.

  即，hb(send to channel, receive from channel)，channel一定是先发，再取。

  `<- ch`可以单独调用获取通道的（下一个）值，当前值会被丢弃.

  `int2 = <- ch` 表示：变量 int2 从通道 ch（一元运算的前缀操作符，前缀 = 接收）接收数据（获取新值）

  `ch <- int1` 表示：给通道 ch 发送变量 int1的值（双目运算符，中缀 = 发送）

  无缓冲的与有缓冲channel有着重大差别，那就是一个是同步的 一个是非同步的。

  比如

  c1:=make(chan int)     无缓冲

  c2:=make(chan int,1)   有缓冲

  c1<-1               

  无缓冲： 不仅仅是向 c1 通道放 1，而是一直要等有别的携程 <-c1 接手了这个参数，那么c1<-1才会继续下去，要不然就一直阻塞着。

  有缓冲： c2<-1 则不会阻塞，因为缓冲大小是1(其实是缓冲大小为0)，只有当放第二个值的时候，第一个还没被人拿走，这时候才会阻塞。

  ```go
  package main
  
  import (
  	"fmt"
  	"time"
  )
  
  var c = make(chan int, 10)
  var a string 
  
  func f() {
  	a = "hi, dodo"
  	for i:=1; i<4; i++{
  		time.Sleep(time.Second * 2)
  		c <- i
  		fmt.Printf("len(c): %d\n",len(c))
  	}
  	//c <- 0 // or close(c)
  }
  
  func main()  {
  	fmt.Printf("goroutine before: %s\n",a)
  	go f()
      // hb(<-c, c<-i)
  	fmt.Printf("<-c: %d\n", <-c)
  	fmt.Println("This go statement is after <-c")
  	fmt.Printf("goroutine after: %s\n",a)
  	time.Sleep(time.Second * 10)
  }
  
  
  
  $ ./happenBefore
  goroutine before:
  len(c): 0
  <-c: 1
  This go statement is after <-c
  goroutine after: hi, dodo
  len(c): 1
  len(c): 2
  
  
  ```

  end

* The closing of a channel **happens before** a receive that returns a zero value because the channel is closed.

  关闭channel先于从对应channel中接收数据，下面的程序保证能打印a的值。

  ```go
  package main
  
  import (
  	"fmt"
  )
  
  var c = make(chan int, 1)
  var a string
  //var c = make(chan int)
  
  func f() {
      // 如果使用非阻塞的channel，则这里需要先send
  	//c <- 5 
  	a = "hi, dodo"
  	close(c)
  }
  
  func main()  {
  	go f()
  	fmt.Printf("<-c: %d\n", <-c)
  	fmt.Println(a)
  }
  
  # ./happenBefore
  <-c: 0
  hi, dodo
  
  ```

  基于上面的理论逆推下，下面代码保证a一定不会输出。

  因为同一个goroutine中golang 语句（go statement）是按顺序执行的即有happen-before的保证。

  close channel后，然后进行sleep，此时a还没有赋值，所有 main goroutine print时，是空白。

  ```go
  package main
  
  import (
  	"fmt"
  	"time"
  )
  
  var c = make(chan int, 1)
  var a string
  //var c = make(chan int)
  
  func f() {
  	//c <- 5
  	//a = "hi, dodo"
  	close(c)
  	time.Sleep(time.Second * 2)
  	a = "hi, dodo"
  }
  
  func main()  {
  	//c <- 5
  	go f()
  	fmt.Printf("<-c: %d\n", <-c)
  	fmt.Println(a)
  }
  // 输出空白
  #./happenBefore
  <-c: 0
  
  [root@dodo awesomeProject]#
  
  ```

  end

  

* A receive from an unbuffered channel **happens before** the send on that channel completes.

  从非缓冲channel中接收数据先于向channel发送数据发生，违背直觉，但下面的程序会保证输出hello world

  When you send on an unbuffered channel, the **sender blocks** until the receiver has taken the value. This means the **receive happens before the send completes**.

  A buffered channel is different, because the value has somewhere to go. If you're still confused, an example might help:

  Say I want to leave a package at your house:

  - If the channel is buffered, you have somewhere for me to leave the package - perhaps a mailbox. This means I can complete my task of giving you the package (sending on the channel) before you receive it (when you check your mailbox).
  - If the channel is not buffered, I have to wait by your front door until you come and take the package off me. You receive the package before I am done with my task of delivering it to you

  ```go
  package main
  
  import (
  	"fmt"
  	"time"
  )
  
  var a string
  var c = make(chan int)
  
  func f() {
      // 故意设置延时，因为同一个goroutine这go statement按顺序执行
      // so，先sleep，再赋值a，最后是 <-c
      // 由于 <-c happen-before c <- 5, 所有，a肯定有值。
  	time.Sleep(time.Second * 2)
  	a = "hi, dodo"
  	<-c
  }
  
  func main()  {
  	go f()
  	c <- 5
  	fmt.Println(a)
  }
  
  ```

  end

* The kth receive on a channel with capacity C **happens before** the k+Cth send from that channel completes.

  下面这个程序定义了信号量为3，因此同时最多只有3个worker在工作

  ？？？ kth是什么

* For any `sync.Mutex` or `sync.RWMutex` variable `l` and `n < m`, call n of `l.Unlock()` happens before call m of `l.Lock()` returns.
  下面的程序中，f()函数中的unlock（1）先于Lock（2）执行

  这个好理解啊，解锁前不允许其他goroutine加锁。

  ```go
  package main
  
  import (
  	"fmt"
  	"time"
  	"sync"
  )
  
  var l sync.Mutex
  var a string
  var c = make(chan int)
  
  func f() {
  	time.Sleep(time.Second * 2)
  	a = "hi, dodo"
  	l.Unlock() //1
  }
  
  func main()  {
  	l.Lock()
  	go f()
  	l.Lock() //2
  	fmt.Println(a)
  }
  
  ```

  end

* For any call to `l.RLock` on a `sync.RWMutex` variable `l`, there is an n such that the `l.RLock` **happens (returns) after** call n to `l.Unlock` and the matching `l.RUnlock` **happens before** call n+1 to `l.Lock`.

  

* A single call of f() from once.Do(f) happens (returns) before any call of once.Do(f) returns.
  `f()`函数先于`once.Do()`返回
  下面的程序只会执行setup函数一次

  [`Once.Do()`](https://golang.org/pkg/sync/#Once.Do) does not return until `f()` has been executed once. Which means if multiple goroutines call `Once.Do()` concurrently, `f()` will be executed once of course, but all calls will wait until `f()`completes (they will be blocked).

  Your proposed solution does not have this very important property! Yours only guarantees that `f()`will be executed only once, but if called from multiple goroutines concurrently, subsequent calls will return immediately, even if `f()` is still running.

  When we use `sync.Once`, we rely on this behavior, we rely on `f()` being completed after `Once.Do()`returns, so we can use all variables that `f()` initialized safely, without having a race condition.

  ```go
  package main
  
  import (
  	"fmt"
  	"time"
  	"sync"
  )
  
  var once sync.Once
  var a string
  var i int
  
  func setup(){
  	i = 3 + i
  	fmt.Println(i)
  	time.Sleep(time.Second * 2)
  	a = "hi, dodo"
  }
  
  func doPrint() {
  	once.Do(setup)
  	//setup()
  	fmt.Println(a)
  }
  
  func main()  {
  	i = 1
  	go doPrint()
  	go doPrint()
  	time.Sleep(time.Second * 3)
  }
  
  // setup 仅执行了一次
  $ ./happenBefore
  4
  hi, dodo
  hi, dodo
  
  // 将once.Do(setup)换成setup()
  $ ./happenBefore
  4
  7
  hi, dodo
  hi, dodo
  
  ```

  end



**1) 单线程**

**2) Init 函数**

如果包P1中导入了包P2，则P2中的init函数Happens Before 所有P1中的操作 main函数Happens After 所有的init函数

 **3) Goroutine**

Goroutine的创建Happens Before所有此Goroutine中的操作 Goroutine的销毁Happens After所有此Goroutine中的操作 

**4) Channel**

对一个元素的send操作Happens Before对应的receive 完成操作  ,  [先发后接] 对channel的close操作Happens Before receive 端的收到关闭通知操作  [先关后接,接到零值] 对于无缓冲channel(unbuffered Channel)，对一个元素的receive 操作Happens Before对应的send完成操作  [先接后发] 对于Buffered Channel，假设Channel 的buffer 大小为C，那么对第k个元素的receive操作，Happens Before第k+C个send完成操作。可以看出上一条Unbuffered Channel规则就是这条规则C=0时的特例  [先接后发] 

**5) Lock**

Go里面有Mutex和RWMutex两种锁，RWMutex除了支持互斥的Lock/Unlock，还支持共享的RLock/RUnlock。

对于一个Mutex/RWMutex，设n < m，则第n个Unlock操作Happens Before第m个Lock操作。 对于一个RWMutex，存在数值n，RLock操作Happens After 第n个UnLock，其对应的RUnLock Happens Before 第n+1个Lock操作。 简单理解就是这一次的Lock总是Happens After上一次的Unlock，读写锁的RLock HappensAfter上一次的UnLock，其对应的RUnlock Happens Before 下一次的Lock。