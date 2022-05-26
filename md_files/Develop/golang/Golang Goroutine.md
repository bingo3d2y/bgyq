## Golang Goroutine

### Summary

在Golang中，goroutine是有三个主要的陷阱：

1. goroutine leaks 
2. data race
3. incomplete work 

对于1和3的情况：Never start a goroutine without knowing how it will stop.

对于2：Don't communicate by sharing memory, share memory by communicating.

遵循以上能尽量避免以上三个问题的发生。

再直白一点就是：

1. 每当使用go启动一个goroutine时一定要注意它是否能正常结束
2. 多个goroutine同时操作同一个变量（**communicate by sharing memory**），会有数据竞争的问题，尽量不要用这种方式；而推荐用传递共享方式，一个goroutine处理完了以后传递给另一个goroutine继续处理（**share memory by communicating**，channel？）

### Goroutine leaks

`leak`的函数创建了一个channel，通过这个channel把数据传递给后面启动的Goroutine。然后这个Goroutine在第被阻塞以等待接收channel上的来的数据。当这个Goroutine等待的时候，启动它的函数`leak`就执行完返回了，**这时候程序中的没有其他任何地方可以给这个channel发送数据**，这就导致Goroutine被永久的阻塞在`<-ch`，从而导致`fmt.Println`就永远不会执行

```go

func main()  {
	go leak()
	time.Sleep(2 * time.Second)
}

// leak is a buggy function. It launches a goroutine that
// blocks receiving from a channel. Nothing will ever be
// sent on that channel and the channel is never closed so
// that goroutine will be blocked forever.
func leak() {
    ch := make(chan int)

    go func() {
        val := <-ch
        fmt.Println("We received a value:", val)
    }()
}
```

#### demo

Any time you start a Goroutine you must ask yourself:

* **When will it terminate?**

* **What could prevent it from terminating?**

##### v1:无goroutine

`search` 函数是一个模拟实现，用于模拟长时间运行的操作，如数据库查询或 rpc 调用。

```go
// search simulates a function that finds a record based
// on a search term. It takes 200ms to perform this work.
func search(term string) (string, error) {
	time.Sleep(200 * time.Millisecond)
	return "some value", nil
}

// process is the work for the program. It finds a record
// then prints it.
func process(term string) error {
	record, err := search(term)
	if err != nil {
		return err
	}
	fmt.Println("Received:", record)
	return nil
}
```

对于一些应用而言，串行的调用`search`带来的时延或许无法接受。假设`search`无法更快的运行了，`processs`函数可以修改成不必完全消耗在等待`search`返回结果。即通过goroutine并发执行`search()`

##### v2: goroutine leak

`process()`使用一个Goroutine来等待`search()`执行的问题，可以避免串行调用问题，但是这个goroutine带来了潜在的Goroutine泄漏。

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"time"
)

//result wraps the return values from search. It allows us
// to pass both values across a single channel.
type result struct {
	record string
	err    error
}

func main()  {
	process("sss")
	time.Sleep(time.Second*2)
}
// process is the work for the program. It finds a record
// then prints it. It fails if it takes more than 100ms.
func process(term string) error {

	// Create a context that will be canceled in 100ms.
	ctx, cancel := context.WithTimeout(context.Background(), 100*time.Millisecond)
	defer cancel()

	// Make a channel for the goroutine to report its result.
	ch := make(chan result)

	// Launch a goroutine to find the record. Create a result
	// from the returned values to send through the channel.
	go func() {
		record, err := search(term)
		ch <- result{record, err}
	}()

	// Block waiting to either receive from the goroutine's
	// channel or for the context to be canceled.
	select {
	case <-ctx.Done():
		return errors.New("search canceled")
	case result := <-ch:
		if result.err != nil {
			return result.err
		}
		fmt.Println("Received:", result.record)
		return nil
	}
}

func search(term string) (string, error) {
	time.Sleep(200 * time.Millisecond)
	return "some value", nil
}
```

泄漏分析：

1. 在`process`函数中设定了一个等待`search`完成的最长时间，也就是100ms。
2. goroutine执行search，不影响`process()`的继续执行
3. goroutine会一直阻塞等待channel获取到数据即`ch <- result{record, err}`
4. 当`process`()等到100ms超时，但是`search()`还在200 ms的阻塞中，触发`case: ctx.Done()`，此时select数据块执行完成没有执行`result := <-ch`语句的机会了
5. 当`search()`在阻塞200 ms结束后，`ch <- result{record, err}`成功执行，但是没有goroutine消费这个channel了，此时`goroutine search()`就阻塞了，从而造成了内存泄漏。

##### v3：解决泄漏

最简单的解决方式就是 - 把原先的非缓冲（阻塞）channel修改为容量为1的缓冲（非阻塞）channel。即`goroutine search()`发送完数据就退出，不用等待是否有goroutine从这个channel中消费数据。

> 这也是function 低耦合，高内聚的体现吧。one function, one thing.

```go
// Make a channel for the goroutine to report its result.
// Give it capacity so sending doesn't block.
ch := make(chan result, 1)

// demo
func process(term string) error {
	// 定义一个 chan 去接受结果
	ch := make(chan result, 1)
	// 开一个goroutine 让其异步取执行
	go func() {
		record, err := search(term)
		ch <- result{record, err}
	}()

	select {
	// 在一个select进来时，若ctx.Done()先执行则直接 return 了，则意味则 chan 内容没人去获取，
	// 而这个 chan 若不是一个 buffer channel，就是一个make(chan result)，则这个 goroutine就会永
	// 远阻塞住，所以需要保证 chan 是一个buffer chan
	case <-ctx.Done():
		return errors.New("search canceled")
	case result := <-ch:
		if result.err != nil {
			return result.err
		}
		fmt.Println("Received:", result.record)
		return nil
	}
}

```

现在在超时的那个分支，即使由于超时导致`select`块结束等待往下进行（`case <-ctx.Done()`分支)，启动`search`的那个Goroutine最终也会得到搜索结果并发送到channel进而执行完成返回而不阻塞。最终Goroutine以及channel占用的内存都会被释放掉。所有事情自然而然的都解决了。

### Incomplete work

https://icode.best/i/50968040357920

```go
 func main() {
     fmt.Println("Hello")
     go fmt.Println("Goodbye")
}
```

程序运行时，只能看到输出`hello`但是不会显示`Goodbye`，因为随着func main 结束，程序直接退出，不会等待其他非main goroutine结束。

当你启动一个Goroutine去做一些很重要的工作时，程序这种退出的行为就造成问题了，因为main函数不知道要等待Goroutine完成。这种情景会导致完整性问题，比如数据库或者文件系统错误，或者数据丢失。

这个和Goroutine leaks好像....

### Data race

Memory Visibility ，内存可见性

```go
func main()  {
	var running int32 = 1

	go func() {
		fmt.Println("Start goroutine 1")
		for running == 1 {
			fmt.Println(time.Now())
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
// go run -race  test.go
Start goroutine 1
2022-05-19 02:22:09.577354271 +0000 UTC m=+0.000573314
Goroutine 1 is running 1
2022-05-19 02:22:10.579441599 +0000 UTC m=+1.002660785
Goroutine 1 is running 1
Start goroutine 2
==================
WARNING: DATA RACE
Read at 0x00c0000b800c by goroutine 7:
  main.main.func1()
      /tmp/test.go:35 +0x16e

Previous write at 0x00c0000b800c by goroutine 8:
  main.main.func2()
      /tmp/test.go:46 +0x88

Goroutine 7 (running) created at:
  main.main()
      /tmp/test.go:33 +0xb0

Goroutine 8 (running) created at:
  main.main()
      /tmp/test.go:42 +0x118
==================
// code is hanging.
^Csignal: interrupt
root@aa628216dec3:/tmp#


// output
Start goroutine 1
2022-05-19 10:18:46.4872681 +0800 CST m=+0.002037901
Goroutine 1 is running 1
2022-05-19 10:18:47.4962548 +0800 CST m=+1.011024901
Goroutine 1 is running 1
Start goroutine 2
2022-05-19 10:18:48.4970897 +0800 CST m=+2.011860101
Goroutine 1 is running 1
...
//一直输出
2022-05-19 10:18:49.4972948 +0800 CST m=+3.012065501
Goroutine 1 is running 1
```

goroutine操作里面使用 running = 0的修改变量，其实也可以达到你的效果，只是因为for循环太快了，需要不停的修改running的值，既然我还在不停的写，goroutine觉得没必要写回去，反正写回去之后，里面修改了又要写会，就没必要了。来不及把修改之后的running值写回到内核态去，不写到内核态去那么第一个goroutine就拿不到修改之后的值。你可以试试在第二个goroutine里面加 time.sleep(time.second),你会发现已经达到你的效果了。所以就是一个不可以预知的行为。

程序中多个 goroutine 同时访问某个数据，必须保证**串行化**访问。即需要增加同步逻辑，可以使用 sync 或者 sync/atomic中的锁或原子类型来保证。另外，题主还可以尝试用 channel 来实现这个同步逻辑。



### 引用

1. https://www.zhihu.com/question/27596075/answer/593672097
2. https://icode.best/i/50968040357920