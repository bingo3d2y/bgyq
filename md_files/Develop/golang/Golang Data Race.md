## Golang Data Race

### 核心:triangular_flag_on_post:

Don't communicate by sharing memory, share memory by communicating.

多个goroutine同时操作同一个变量（**communicate by sharing memory**），会有数据竞争的问题，尽量不要用这种方式；而推荐用传递共享方式，一个goroutine处理完了以后传递给另一个goroutine继续处理（**share memory by communicating**）

**communicate by sharing memory**需要考虑加锁避免race condition,加锁又会提高开发及调试难度。而 **share memory by communicating**则可以将对共享资源(memory)的访问串行化，于是就不用考虑race condition了。

程序中多个 goroutine 同时访问某个数据，必须保证**串行化**访问。即需要增加同步逻辑，可以使用 sync 或者 sync/atomic中的锁或原子类型来保证。另外，还可以尝试用 channel 来实现这个同步逻辑，从而实现内存的可见性。

### data race导致内存不可见性:fire:

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

### 共享变量

```go
var test *int

func main() {
	var wg sync.WaitGroup
	n := 5
	wg.Add(n)
	for i := 0; i < n; i++ {
		go func() {
			fmt.Println("i is:", i) // it is not the 'i' you are looking for.
			wg.Done()
		}()
	}
	wg.Wait()
}

// go run -race test.go
==================
WARNING: DATA RACE
Read at 0x00c0000180a8 by goroutine 7:
  main.main.func1()
      /tmp/test.go:86 +0x3d

Previous write at 0x00c0000180a8 by main goroutine:
  main.main()
      /tmp/test.go:84 +0xab

Goroutine 7 (running) created at:
  main.main()
      /tmp/test.go:85 +0x8f
==================
i is: 3
i is: 5
i is: 5
i is: 5
i is: 5
Found 1 data race(s)
exit status 66
// 每次输出结果不定
// go run test.go
i is: 5
i is: 5
i is: 5
i is: 5
i is: 5


```

原因分析

* 用于控制退出循环的变量 i, 对 i 执行自增操作, 有写操作；循环内部，起 5 个 goroutine, 每个 goroutine 均输出 i, 存在读操作；满足 data race 发生的条件
* 执行 `go run loop_counter.go` 大概率会输出 55555，而不是 01234
* 在开启数据竞态检测的情况下，每次输出的结果几乎不同，这是因为存在数据并发读写，程序行为不可预测

#### 消除竞态：内存

消除N个goroutine对一快内存的同时写操作，即将i在赋值给不同的变量，实现内存不共享。

```go
var test *int

func main() {
	var wg sync.WaitGroup
	n := 5
	wg.Add(n)
	for i := 0; i < n; i++ {
		go func(j int) {
			fmt.Println("j is:", j)
			wg.Done()
		}(i)
	}
	wg.Wait()
}

```

goroutine加个形参，实现每个goroutine print时是独立的内存。

#### 消除竞态：锁

```go
var test *int

func main() {
	var wg sync.WaitGroup
	var mux sync.Mutex
	n := 5
	wg.Add(n)
	for i := 0; i < n; i++ {
		mux.Lock()
		go func() {
            //这里 i要减一，因为for先循环才到Lock()
			fmt.Println("i", i -1 )
			wg.Done()
			mux.Unlock()

		}()
	}
	wg.Wait()
}
// output
i 0
i 1
i 2
i 3
i 4
```

通过锁可以控制竞态，但是竞态还是存在的，`go run -race`还会显示`WARNING: DATA RACE`。

### 意外共享变量

竞态原因：内层 goroutine 与 外层main  goroutine 使用同一个 err 变量，两个 goroutine 并发对 err 执行写操作

```go
func main() {
    data := []byte("hello golang")
    res := make(chan error, 2)
    f1, err := os.Create("file1")
    fmt.Printf("pointer of err from file1: %p\n", &err)
    if err != nil {
        res <- err
    } else {
        go func() {
            // This err is shared with the main goroutine, so the write races with the write below.
            _, err = f1.Write(data)
            res <- err
            f1.Close()
        }()
    }
    // receive the error message
    go func() {
        for err := range res {
            fmt.Println("err", err)
        }
    }()
    time.Sleep(1 * time.Second) // wait the goroutine to receive the err
}

```

#### 消除竞态

修改很简单，将`_, err = f1.Write(data)`改成`_, err := f1.Write(data)`，即在goroutine中声明新的变量err。

### 为保护的全局变量

代码发生竞态的条件：SetName 和 GetName 访问的是同一个 map num2Name，读写同一个map

```go
var (
        num2Name = make(map[int]string) // key: number, value: name
)

func SetName(num int, name string) {
        num2Name[num] = name
}

func GetName(num int) string {
        return num2Name[num]
}

func main() {
        wg := &sync.WaitGroup{}
        wg.Add(2)
        go func() {
                defer wg.Done()
                SetName(1, "Chandler Unicorn")

        }()
        fmt.Println("wait set goroutine flush")
        time.Sleep(time.Second)
        go func() {
                defer wg.Done()
                fmt.Println(GetName(1))
        }()
        wg.Wait()
}

root@aa628216dec3:/tmp# go run -race test.go
wait set goroutine flush
==================
WARNING: DATA RACE
Read at 0x00c00009c150 by goroutine 9:
  runtime.mapaccess1_fast64()
      /usr/local/go/src/runtime/map_fast64.go:12 +0x0
  main.GetName()
      /tmp/test.go:18 +0xb0
  main.main.func2()
      /tmp/test.go:33 +0xcd

Previous write at 0x00c00009c150 by goroutine 7:
  runtime.mapassign_fast64()
      /usr/local/go/src/runtime/map_fast64.go:92 +0x0
  main.SetName()
      /tmp/test.go:14 +0xa4
  main.main.func1()
      /tmp/test.go:26 +0x7b

Goroutine 9 (running) created at:
  main.main()
      /tmp/test.go:31 +0x190

Goroutine 7 (finished) created at:
  main.main()
      /tmp/test.go:24 +0xd0
==================
==================
WARNING: DATA RACE
Read at 0x00c000060048 by goroutine 9:
  main.GetName()
      /tmp/test.go:18 +0xbd
  main.main.func2()
      /tmp/test.go:33 +0xcd

Previous write at 0x00c000060048 by goroutine 7:
  main.SetName()
      /tmp/test.go:14 +0xb0
  main.main.func1()
      /tmp/test.go:26 +0x7b

Goroutine 9 (running) created at:
  main.main()
      /tmp/test.go:31 +0x190

Goroutine 7 (finished) created at:
  main.main()
      /tmp/test.go:24 +0xd0
==================
Chandler Unicorn
Found 2 data race(s)
```

#### 消除竞态

解决决思路：使用读写锁控制并发，访问 map 前加锁

```go
var (
	num2Name = make(map[int]string) // key: number, value: name
	lock = new(sync.RWMutex)
)

func SetName(num int, name string) {
	lock.Lock()
	defer lock.Unlock()
	num2Name[num] = name
}

func GetName(num int) string {
	lock.Lock()
	defer lock.Unlock()
	return num2Name[num]
}

func main() {
	wg := &sync.WaitGroup{}
	wg.Add(2)
	go func() {
		defer wg.Done()
		SetName(1, "Chandler Unicorn")

	}()

	go func() {
		defer wg.Done()
		fmt.Println(GetName(1))
	}()
	wg.Wait()
}
```

end

### 为保护的私有变量

KeepAlive 和 Start 函数均访问了私有变量 last，当多个 goroutine 并发调用 KeepAlive 和 Start 函数时，就会发生竞态。

```go
type Watchdog struct{ last int64 }

func (w *Watchdog) KeepAlive() {
	w.last = time.Now().UnixNano() // First conflicting access.
}

func (w *Watchdog) Start() {
	go func() {
		for {
			time.Sleep(time.Second)
			// Second conflicting access.
			if w.last < time.Now().Add(-10 * time.Second).UnixNano() {
				fmt.Println("No keepalives for 10 seconds. Dying.")
				os.Exit(1)
			} else {
				fmt.Println("time < w.last")
			}
		}
	}()
}

func main() {
	watchDog := Watchdog{}
	go watchDog.KeepAlive()
	watchDog.Start()
	time.Sleep(20 * time.Second) // wait a moment
}
```

end

#### 消除竞态：channel

常规方法：使用 channel 或者 锁。

channel 思路应该是生产者-消费者模型吧，先由`watchDog.KeepAlive()`给`last`赋值，然后`watchDog.Start()`才开始读取`last`

```go
type Watchdog struct{ last int64 }

func (w *Watchdog) KeepAlive(flag chan int) {
	w.last = time.Now().UnixNano()
	flag <- 1
}

func (w *Watchdog) Start(flag chan int) {
	go func() {
		<- flag
		for {
			if w.last < time.Now().Add(-10*time.Second).UnixNano() {
				fmt.Println("No keepalives for 10 seconds. Dying.")
				os.Exit(1)
			}
		}
	}()
}

func main() {
	flag := make(chan int)
	watchDog := Watchdog{}
	go watchDog.KeepAlive(flag)
	watchDog.Start(flag)
	time.Sleep(20 * time.Second) // wait a moment
}

// go run -race  test.go
No keepalives for 10 seconds. Dying.
exit status 1

```

end

#### 消除竞态：atomic

也可以使用 sync/automic 包来实现无锁行为.

`Mutex`由**操作系统**实现，而`atomic`包中的原子操作则由**底层硬件**直接提供支持。在 CPU 实现的指令集里，有一些指令被封装进了`atomic`包，这些指令在执行的过程中是不允许中断（interrupt）的，因此原子操作可以在`lock-free`的情况下保证并发安全，并且它的性能也能做到随 CPU 个数的增多而线性扩展。

```go
type Watchdog struct{ last int64 }

func (w *Watchdog) KeepAlive() {
	atomic.StoreInt64(&w.last, time.Now().UnixNano())
}

func (w *Watchdog) Start() {
	go func() {
		for {
			time.Sleep(time.Second)
			if atomic.LoadInt64(&w.last) < time.Now().Add(-10*time.Second).UnixNano() {
				fmt.Println("No keepalives for 10 seconds. Dying.")
				os.Exit(1)
			}
		}
	}()
}

```

end

### 异步channel写和关闭

存在对同一个 channel 同时写和关闭的操作。

According to the Go memory model, a send on a channel happens before the corresponding receive from that channel completes.

```go
package main

func main() {
	c := make(chan struct{}) // or buffered channel

	// The race detector cannot derive the happens before relation for the following send and close operations. These two operations are unsynchronized and happen concurrently.
	go func() { c <- struct{}{} }()
	close(c)
}

```

 end

#### 竞态消除

核心还是在close之前write已经完成

```go
package main

func main() {
	c := make(chan struct{}) // or buffered channel

	// The race detector cannot derive the happens before relation
	// for the following send and close operations. These two operations
	// are unsynchronized and happen concurrently.
	go func() { c <- struct{}{} }()
	<-c // receive operation can guarantee the send is done
	close(c)
}

```

读 channel 的操作能够保证 写 channel 的动作已完成

To synchronize send and close operations, use a receive operation that guarantees the send is done before the close


### 参考引用

1. https://www.zhihu.com/question/434964023
2. https://www.modb.pro/db/72705
2. https://www.zhihu.com/question/27596075/answer/593672097
2. https://www.zhihu.com/question/434964023/answer/1628814199
