## 剑来 -- golang

### 快捷键

`Ctrl+\`： navigate back 这个不赖

### Slice: `:`

比较简单，直接用`初始化表达式`创建。

```golang
package main

import "fmt"

func main() {
    // before : is index, after : is value
	s1 := []int{0, 1, 2, 3, 8: 100}
	fmt.Println(s1, len(s1), cap(s1))
}

// 下面代表什么意思
var data = []int{0: -10, 1: -5, 2: 0, 3: 1, 4: 2, 5: 3, 6: 5, 7: 7, 8: 11, 9: 100, 10: 100, 11: 100, 12: 1000, 13: 10000}
// 淦,output
[-10 -5 0 1 2 3 5 7 11 100 100 100 1000 10000]

// 懂了
var data = []int{0: -10, 10: 100, 11: 100, 12: 1000, 13: 10000}
// output
[-10 0 0 0 0 0 0 0 0 0 100 100 1000 10000]
```

运行结果：

```shell
[0 1 2 3 0 0 0 0 100] 9 9
```

唯一值得注意的是上面的代码例子中使用了索引号，直接赋值，这样，其他未注明的元素则默认 `0 值`。

### return

#### return value

return value要和Signature (functions) 保持一致、

如下报错，是因为函数签名中声明的返回值是`error interface`, 但是`test()`的返回值确是一个`func()`

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/golang-return.png)

改成下面这样的就行了

```go
func test() string{
	return func() string{ return "hi" }()
}

func main()  {
	a := test()
	fmt.Print(a)
}

// output
hi
```

再看毛剑给的例子

```go
// error interface definition 
type error interface {
	Error() string
}

// function signature
func serve(addr string, handler http.Handler, stop <-chan struct{}) error  {
	s := http.Server{
		Addr: addr,
		Handler: handler,
	}
	go func() {
		<-stop
		s.Shutdown(context.Background())
	}()
    // 返回error，通过收到error，去执行s.shutdonw()
	return s.ListenAndServe()
}

// http.Server.ListenAndServe()
func (srv *Server) ListenAndServe() error {
	...
	return srv.Serve(ln)
}
// 出错了返回error 不出错一直accept处理请求
func (srv *Server) Serve(l net.Listener) error {
	...

	ctx := context.WithValue(baseCtx, ServerContextKey, srv)
	for {
		rw, err := l.Accept()
		if err != nil {
			...
			return err
		}
		...
	}
}
```

end

### chan

#### close()

send-only

```go
// The close built-in function closes a channel, which must be either
// bidirectional or send-only. It should be executed only by the sender,
// never the receiver, and has the effect of shutting down the channel after
// the last sent value is received. After the last value has been received
// from a closed channel c, any receive from c will succeed without
// blocking, returning the zero value for the channel element. The form
//	x, ok := <-c
// will also set ok to false for a closed channel.
func close(c chan<- Type)
```

如下，close()发送chan Type_zero_value

```go
func test(stop <-chan struct{}){
	go func() {
		fmt.Println(<-stop)
		fmt.Print("get something")
	}()
	select {
	}
}

func main(){
	stop := make(chan struct{})
	go test(stop)
	time.Sleep(2 * time.Second)
	close(stop)
	time.Sleep(1 * time.Second)
}
// output
{}
get something
```

end

### goroutine

#### 几个原则

1. 只有调用者才可以去创建goroutine!

   下面这个示例是错误的示例，因为调用者main根本就不知道serveApp启动了一个goroutine，更没法对这个goroutine负责了。

   Never start a goroutine without knowing when it will stop

   ```go
   
   func main(){
       serveApp()
   }
   
   func serveApp(){
       go func()
   }
   ```

   

2. end

#### demo

https://github.com/da440dil/go-workgroup/blob/master/server.go

#### custom http.Server

https://studygolang.com/articles/15283

