## Golang Context

### 野生goroutine场景

Go  http 服务端，每个进入的请求会被其所属 goroutine 处理，常会启动额外的的 goroutine 进行数据查询或 PRC 调用等。而当请求返回时，这些额外创建的 goroutine 需要及时回收。一个请求对应一组请求域内的数据可能会被该请求调用链条内的各 goroutine 所需要。

例如，在如下代码中，当请求进来时，Handler 会创建一个监控 goroutine，其会每隔 1s 打印一句“req is processing”。

```go
func main() {
	ser := &http.Server{
		Addr: "127.0.0.1:8080",

	}
	http.HandleFunc("/echo", func(w http.ResponseWriter, r *http.Request) {
		// monitor
		go func() {
			for range time.Tick(time.Second) {
				fmt.Println("req is processing")
			}
		}()

		// assume req processing takes 3s
		time.Sleep(3 * time.Second)
		w.Write([]byte("hello"))
		ser.Close()
	})

	ser.ListenAndServe()
	log.Println(ser.ListenAndServe())
	time.Sleep(3 * time.Second)
	os.Exit(-2)
}

// output
req is processing
req is processing
req is processing
2022/05/16 12:02:19 http: Server closed
// request is quit, but goroutine is running.
req is processing
req is processing
req is processing

Process finished with the exit code -2
```

假定请求需耗时 3s，即请求在 3s 后返回，我们期望监控 goroutine 在打印 3 次“req is processing”后即停止。但运行发现，监控 goroutine 打印 3 次后，其仍不会结束，而会继续打印下去直到main goroutine退出。

问题就出在创建监控 goroutine 后，未对其生命周期作控制。

#### context demo

为了控制request的goroutine，下面使用 context 作控制，即监控程序打印前需检测`r.Context()`是否已经结束，若结束则退出循环，即结束生命周期。

```go
func main() {
	ser := &http.Server{
		Addr: "127.0.0.1:8080",

	}
	http.HandleFunc("/echo", func(w http.ResponseWriter, r *http.Request) {
		// monitor
		go func() {
			for range time.Tick(time.Second) {
				select {
				case <- r.Context().Done():
					fmt.Println("req is outgoing")
					return
				default:
					fmt.Println("req is processing")
				}

			}
		}()

		// assume req processing takes 3s
		time.Sleep(3 * time.Second)
		w.Write([]byte("hello"))
		ser.Close()
	})

	ser.ListenAndServe()
	log.Println(ser.ListenAndServe())
	time.Sleep(3 * time.Second)
	os.Exit(-2)
}
// output
req is processing
req is processing
req is outgoing
2022/05/16 14:09:16 http: Server closed
```

http.Request结构体需要包含`ctx.Context`字段

### Context类型

源码定义的很简单

```go
type Context interface {
	// Deadline returns the time when work done on behalf of this context should be canceled. Deadline returns ok==false when no deadline is set. Successive calls to Deadline return the same results.
	Deadline() (deadline time.Time, ok bool)

	// Done returns a channel that's closed when work done on behalf of this context should be canceled. Done may return nil if this context can never be canceled. Successive calls to Done return the same value.

	Done() <-chan struct{}

	// If Done is not yet closed, Err returns nil.
    // If Done is closed, Err returns a non-nil error explaining why:
	// Canceled if the context was canceled
	// or DeadlineExceeded if the context's deadline passed.
	// After Err returns a non-nil error, successive calls to Err return the same error.
	Err() error

	// Value returns the value associated with this context for key, or nil if no value is associated with key. Successive calls to Value with the same key returns the same result.
    // 获取key之前设置的value
	Value(key interface{}) interface{}
}
```

Done 方法返回一个 channel，当 Context 取消或到达截止时间时，该 channel 即会关闭。

Err 方法返回 Context 取消的原因。

**Context 自己没有 Cancel 方法，而且 Done channel 仅用来接收信号（只能<- chan struct{}）**：接收取消信号的函数不应同时是发送取消信号的函数。父 goroutine 启动子 goroutine 来做一些子操作，而子 goroutine 不应用来取消父 goroutine。

若有截止时间，Deadline() 方法可以返回该 Context 的取消时间。

Value()允许 Context 携带请求域内的数据，该数据访问必须保障多个 goroutine 同时访问的安全性。

**Context 是安全的，可被多个 goroutine 同时使用。一个 Context 可以传给多个 goroutine，而且可以给所有这些 goroutine 发取消信号。**

#### Done(): lazily

https://zhuanlan.zhihu.com/p/68792989

先来看 `Done()` 方法的实现：

```go
func (c *cancelCtx) Done() <-chan struct{} {
	d := c.done.Load()
	if d != nil {
		return d.(chan struct{})
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	d = c.done.Load()
	if d == nil {
		d = make(chan struct{})
		c.done.Store(d)
	}
	return d.(chan struct{})
}
// A cancelCtx can be canceled. When canceled, it also cancels any children
// that implement canceler.
type cancelCtx struct {
	Context

	mu       sync.Mutex            // protects following fields
	done     atomic.Value          // of chan struct{}, created lazily, closed by first cancel call
	children map[canceler]struct{} // set to nil by the first cancel call
	err      error                 // set to non-nil by the first cancel call
}



// Load returns the value set by the most recent Store.
// It returns nil if there has been no call to Store for this Value.
func (v *Value) Load() (val interface{}) {
	vp := (*ifaceWords)(unsafe.Pointer(v))
	typ := LoadPointer(&vp.typ)
	if typ == nil || uintptr(typ) == ^uintptr(0) {
		// First store not yet completed.
		return nil
	}
	data := LoadPointer(&vp.data)
	vlp := (*ifaceWords)(unsafe.Pointer(&val))
	vlp.typ = typ
	vlp.data = data
	return
}
```

`Done()` 返回一个 channel，可以表示 context 被取消的信号：当这个 channel 被关闭时，说明 context 被取消了。注意，这是一个只读的channel。 我们又知道，读一个关闭的 channel 会读出相应类型的零值。并且源码里没有地方会向这个 channel 里面塞入值。换句话说，这是一个 `receive-only` 的 channel。因此在子协程里读这个 channel，除非被关闭，否则读不出来任何东西。也正是利用了这一点，子协程从 channel 里读出了值（零值）后，就可以做一些收尾工作，尽快退出。



c.done 是“懒汉式”创建，只有调用了 Done() 方法的时候才会被创建。再次说明，函数返回的是一个只读的 channel，而且没有地方向这个 channel 里面写数据。所以，直接调用读这个 channel，协程会被 block 住。一般通过搭配 select 来使用。一旦关闭，就会立即读出零值。

`Err()` 和 `String()` 方法比较简单，不多说。推荐看源码，非常简单



### Context衍生类

```go
// WithCancel returns a copy of parent with a new Done channel. The returned context's Done channel is closed when the returned cancel function is called or when the parent context's Done channel is closed, whichever happens first.

func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {}

// WithDeadline returns a copy of the parent context with the deadline adjusted
// to be no later than d. If the parent's deadline is already earlier than d,
// WithDeadline(parent, d) is semantically equivalent to parent. The returned
// context's Done channel is closed when the deadline expires, when the returned
// cancel function is called, or when the parent context's Done channel is
// closed, whichever happens first.
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {}


// WithTimeout returns WithDeadline(parent, time.Now().Add(timeout)).
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
	return WithDeadline(parent, time.Now().Add(timeout))
}

// WithValue returns a copy of parent in which the value associated with key is val.
func WithValue(parent Context, key, val interface{}) Context {}
```

context 包提供从已有 Context 衍生新的 Context 的能力。这样即可形成一个 Context 树，**当父 Context 取消时，所有从其衍生出来的子 Context 亦会被取消**。

**Background 是所有 Context 树的根，其永远不会被取消。**

使用 WithCancel 及 WithTimeout 可以创建衍生的 Context，WithCancel 可用来取消一组从其衍生的 goroutine，WithTimeout 可用来设置截止时间来取消其衍生goroutine。

WithValue 提供给 Context 赋予请求域数据的能力。

#### withCannel

main 函数使用 WithCancel 创建一个基于 Background 的 ctx，根据传入的参数循环输出`monitor woring...`，然后手动取消context，发现子goroutine也停止了输出。

```go
func main() {
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()
	// monitor
	go testContext(ctx, 3)
	fmt.Println("before cancel...")
	time.Sleep(4 * time.Second)
	cancel()
	fmt.Println("after cancel...")
	time.Sleep(4 * time.Second)
}

func testContext(ctx context.Context, count int)  {
	for  i:=0;i< count; i++ {
		time.Sleep(1 * time.Second)
		select {
		case <-ctx.Done():
			return
		default:
			fmt.Println("monitor woring...")
		}
	}
	fmt.Println("for is over, waiting cancel()...")
	time.Sleep(4 * time.Second)
}
```

end

#### withTimeout

如下代码中使用 WithTimeout 创建一个基于 Background 的 ctx，其会在 3s 后取消。

注意，虽然到截止时间会自动 cancel，但 cancel 代码仍建议加上。

到截止时间而被取消还是被 cancel 代码所取消，取决于哪个信号发送的早。

```go
func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()

    select {
    case <-time.After(4 * time.Second):
        fmt.Println("overslept")
    case <-ctx.Done():
        fmt.Println(ctx.Err())
    }
}
```

WithDeadline 的使用与 WithTimeout 相似。

没想好 Context 的具体使用，可以使用 TODO 来占位，也便于工具作正确性检查。

#### withValue

基于 Background 创建一个带值的 ctx，然后可以根据 key 来取值。

**注意**：避免多个包同时使用 context 而带来冲突，key 不建议使用 string 或其他内置类型，而建议自定义 key 类型。

```go
type ctxKey string

func main() {
    // func WithValue(parent Context, key, val interface{}) Context {
    // return &valueCtx{parent, key, val} }
	ctx := context.WithValue(context.Background(), ctxKey("a"), "a")

	get := func(ctx context.Context, k ctxKey) {
		if v, ok := ctx.Value(k).(string); ok {
			fmt.Println(v)
		}
	}
	get(ctx, ctxKey("a"))
	get(ctx, ctxKey("b"))
}

// A valueCtx carries a key-value pair. It implements Value for that key and
// delegates all other calls to the embedded Context.
type valueCtx struct {
	Context
	key, val interface{}
}

func (c *valueCtx) Value(key interface{}) interface{} {
	if c.key == key {
		log.Println("nnn")
		return c.val
	}
	log.Println("aaa")
    // parent Context???
	return c.Context.Value(key)
}

// output
a
2022/05/16 15:36:42 nnn
2022/05/16 15:36:42 aaa

Process finished with the exit code 0
```

end

### Context使用规则

 Context 使用规则：

- 勿将 Context 作为 struct 的字段使用，而是对每个使用其的函数分别作参数使用，其需定义为函数或方法的第一个参数，一般叫作 ctx；
- 勿对 Context 参数传 nil，未想好的使用那个 Context，请传`context.TODO`；
- 使用 context 传值仅可用作请求域的数据，其它类型数据请不要滥用；
- 同一个 Context 可以传给使用其的多个 goroutine，且 Context 可被多个 goroutine 同时安全访问。

在官方博客里，对于使用 context 提出了几点建议：

1. Do not store Contexts inside a struct type; instead, pass a Context explicitly to each function that needs it. The Context should be the first parameter, typically named ctx.
2. Do not pass a nil Context, even if a function permits it. Pass context.TODO if you are unsure about which Context to use.
3. Use context Values only for request-scoped data that transits processes and APIs, not for passing optional parameters to functions.
4. The same Context may be passed to functions running in different goroutines; Contexts are safe for simultaneous use by multiple goroutines.

通常使用思路

1. main goroutine创建 background context， 它是一个空的context不能被取消，没有值，也没有超时时间。
2. 有了根节点 context，又提供了四个函数创建子节点 context：withXXX()

context 会在函数传递间传递。只需要在适当的时间调用 cancel 函数向 goroutines 发出取消信号或者调用 Value 函数取出 context 中的值。

### 参考引用

1. https://leileiluoluo.com/posts/golang-context.html
2. https://zhuanlan.zhihu.com/p/68792989

