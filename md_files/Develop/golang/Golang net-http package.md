## Golang  net-http package

一般 `main()`函数写在最前面，然后按函数的调用，开始按顺序声明函数--

`main()`一定要简洁清晰--

### 实现http服务

#### V1：simple

直接使用http.HandleFunc(partern,function(http.ResponseWriter,
*http.Request){})

HandleFunc接受两个参数，第一个为路由地址，第二个为处理方法。

```go
// A Request represents an HTTP request received by a server
// or to be sent by a client.
// A Request represents an HTTP request received by a server
// or to be sent by a client.

// The handler is typically nil, in which case the DefaultServeMux is used.
//
// ListenAndServe always returns a non-nil error.
func ListenAndServe(addr string, handler Handler) error {
	server := &Server{Addr: addr, Handler: handler}
	return server.ListenAndServe()
}

//v1
func main() {
	http.HandleFunc(
		"/",
		func(w http.ResponseWriter, r *http.Request) {
			w.Write([]byte("httpserver v1"))
		},
	)
	http.HandleFunc("/bye", sayBye)
	log.Println("Starting v1 server ...")
	log.Fatal(http.ListenAndServe(":1210", nil))
}

func sayBye(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("bye bye ,this is v1 httpServer"))
}
```



#### V2：custom handler

查看标准库源码，v1版本实际上是调用了handle方法,传入的HandlerFunc实现了Handler的ServeHTTP方法，实际上是ServeHTTP在做http请求处理。

```go
// HandleFunc registers the handler function for the given pattern in the DefaultServeMux.
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	DefaultServeMux.HandleFunc(pattern, handler)
}

// HandleFunc registers the handler function for the given pattern.
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	if handler == nil {
		panic("http: nil handler")
	}
	mux.Handle(pattern, HandlerFunc(handler))
}

// Handle function signature 
// Handle method belongs to ServeMux struct. 
func (mux *ServeMux) Handle(pattern string, handler Handler) {}

// HandleFunc传入的function最终实现了ServeHTTP interface
type Handler interface {
    // 自定义的handler必须实现这个接口
	ServeHTTP(ResponseWriter, *Request)
}

// The handler is typically nil, in which case the DefaultServeMux is used.
//
// ListenAndServe always returns a non-nil error.
func ListenAndServe(addr string, handler Handler) error {
	server := &Server{Addr: addr, Handler: handler}
	return server.ListenAndServe()
}
```

由此我们可以自定义自己的Handler，v2版本代码如下:

```go
// v2
func main() {
	mux := http.NewServeMux()
	mux.Handle("/", &myHandler{})
	mux.HandleFunc("/bye", sayBye)


	log.Println("Starting v2 server ...")
	log.Fatal(http.ListenAndServe(":1210", mux))
}

type myHandler struct {}
// Write writes the data to the connection as part of an HTTP reply.
// Interface signature: Write([]byte) (int, error)
func (*myHandler) ServeHTTP(w http.ResponseWriter, r * http.Request){
	w.Write([]byte("this is version 2." + "url_path is" + r.URL.Path))
}

func sayBye(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("bye bye ,this is v2 httpServer"))
}
```

需要启动http listen时，使用自动的myHandler替代默认handler

#### V3： custom http server

继续查看ListenAndServe的源码

```go
// ListenAndServe always returns a non-nil error.
func ListenAndServe(addr string, handler Handler) error {
	server := &Server{Addr: addr, Handler: handler}
	return server.ListenAndServe()
}
```

这里可以自定义http服务器配置，都在Server这个结构体中,这个对象能配置监听地址端口，配置读写超时时间，配置handler,配置请求头最大字节数...，所有稍微改造一下v2的程序得到v3版:

```go
var server *http.Server
func main() {
	mux := http.NewServeMux()
	mux.Handle("/", &myHandler{})
	mux.HandleFunc("/bye", sayBye)

	server := &http.Server{
		Addr:     ":1210",
		WriteTimeout: time.Second * 3,      //设置3秒的写超时
		Handler:   mux,                     // 使用自定义handler
	}
	log.Println("Starting v3 httpserver")
	log.Fatal(server.ListenAndServe())
}

type myHandler struct{}

func (*myHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("this is version 3"))
}

func sayBye(w http.ResponseWriter, r *http.Request) {
	// 睡眠4秒 上面配置了3秒写超时，
	//所以访问 “/bye“路由会出现没有响应的现象
	time.Sleep(4 * time.Second)
	w.Write([]byte("bye bye ,this is v3 httpServer"))
}
```

end

#### V4：shutdown

HTTP是的一问一答协议. 每个连接有一个业务状态位, 在HTTP请求读取/构造完成后标记为 `StateActive`, 请求处理完毕后标记为 `StateIdle`, 并阻塞等待下一个请求读取上.

Close的操作:

1. 关掉所有监听的Listener, 确保不会有新的连接产生
2. 遍历连接字典, 直接关闭TCP连接.

Shutdown:

1. 同Close, 卸掉监听端口
2. 创建一个定时器轮询连接字典, 并关闭掉处于 `StateIdle` 状态的连接, 直到连接全部关闭, 或者超时.

如果关闭的时候没有在途请求处理, 那么`Close`和`Shutdown`是等效的, 这在QPS比较低的时候会比较普遍.

如果 `Shutdown` 由于超时失败, 安全点的做法感觉还是要调用下 `Close` 确保所有客户端连接都被关掉.

实现由于涉及多线程的问题, 代码处理的时候用了很多channel, mutex, 以及atomic操作, 看上去会比较啰嗦.

Shutdown 将无中断的关闭正在活跃的连接，然后平滑的停止服务。处理流程如下：

- 首先关闭所有的监听;
- 然后关闭所有的空闲连接;
- 然后无限期等待连接处理完毕转为空闲，并关闭;
- 如果提供了 带有超时的Context，将在服务关闭前返回 Context的超时错误;

栗子思路：

发起两个请求`/`和`/bye`，`/`故意延时30s使连接不断开，然后请求`/bye`去执行shutdown，这时可以发下，执行`/bye`时不会里面停止httpServer而是在`/`请求结束后，再停止httpserver。

优雅，优雅！

```go
// Shutdown gracefully shuts down the server without interrupting any active connections.

var server *http.Server
func main() {

	// 一个通知退出的chan
	quit := make(chan os.Signal)
	signal.Notify(quit, os.Interrupt)

	mux := http.NewServeMux()
	mux.Handle("/", &myHandler{})
	mux.HandleFunc("/bye", sayBye)

	server = &http.Server{
		Addr:     ":1210",
		WriteTimeout: time.Second * 40,
		Handler:   mux,
	}

	go func() {
		// 接收退出信号
		<-quit
		log.Println("get quit signal")
		if err := server.Close(); err != nil {
			log.Fatal("Close server:", err)
		}
	}()

	log.Println("Starting v3 httpserver")
	err := server.ListenAndServe()
	if err != nil {
		// 正常退出
		if err == http.ErrServerClosed {
			log.Println("Server closed under request")
		} else {
			log.Println("Server closed unexpected", err)
		}
	}
	log.Println("Server exited")
	time.Sleep( 40 * time.Second)
}

type myHandler struct{}

func (*myHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("this is version 3"))
	time.Sleep( 30 * time.Second)
}

// 关闭http
func sayBye(w http.ResponseWriter, r *http.Request) {
	// --这里为什么没有输出--，request被终止了？
	w.Write([]byte("bye bye ,shutdown the server"))   // 没有输出
	ctx := context.TODO()
	err := server.Shutdown(ctx)
	if err != nil {
		log.Println([]byte("shutdown the server err"))
	}
}
```

end

### keep-alive

#### http default setting

按照HTTP/1.1的规范，Go http包的http server和client的实现默认将所有连接视为长连接，无论这些连接上的初始请求是否带有**Connection: keep-alive**。

#### disable keep-alive

核心：在建立http连接时，添加**Connection:[close]**的http header

##### client

修改http client默认的http Transport，将keepalive给关掉

```go
// Transport is an implementation of RoundTripper that supports HTTP,
// HTTPS, and HTTP proxies (for either HTTP or HTTPS with CONNECT).
//
// By default, Transport caches connections for future re-use.
// This may leave many open connections when accessing many hosts.
// This behavior can be managed using Transport's CloseIdleConnections method
// and the MaxIdleConnsPerHost and DisableKeepAlives fields.	
	tr := &http.Transport{
		DisableKeepAlives: true,
	}
// A Client is an HTTP client. Its zero value (DefaultClient) is a usable client that uses DefaultTransport.
	c := &http.Client{
		Transport: tr,
	}
	req, err := http.NewRequest("get", "http://127.0.0.1:8080", nil)
	if err != nil{
		panic(err)
	}
	resp, err := c.Do(req)
	fmt.Println(resp)
```

http.Client底层的数据连接建立和维护是由http.Transport实现的，http.Transport结构有一个DisableKeepAlives字段，其默认值为false，即启动keep-alive。这里我们将其置为false，即关闭keep-alive，然后将该Transport实例作为初值，赋值给http Client实例的Transport字段。

##### server

自定义http server关闭keepalive功能

```go
	server = &http.Server{
		Addr:     ":1210",
		WriteTimeout: time.Second * 40,
		Handler:   mux,
	}
	server.SetKeepAlivesEnabled(false)
```

调用了http.Server的SetKeepAlivesEnabled方法，并传入false参数，这样我们就在全局层面关闭了该Server对keep-alive连接的支持.

##### 长连接闲时自动关闭

直接选择关闭和开启keepalive都太极端了，通过Idletime参数http server即可以对高密度传输数据的连接保持keep-alive，又可以及时清理掉那些长时间没有数据传输的idle连接。

```go
	server = &http.Server{
		Addr:     ":1210",
		WriteTimeout: time.Second * 40,
		IdleTimeout: 5 * time.Second,
		Handler:   mux,
	}
```

源码中idle tomeout参数的注释

```go
	// IdleTimeout is the maximum amount of time to wait for the
	// next request when keep-alives are enabled. If IdleTimeout
	// is zero, the value of ReadTimeout is used. If both are
	// zero, there is no timeout.
	IdleTimeout time.Duration
```

end

### mutli-port-http:happy:

一个应用监听两个端口，一个端口用于提供服务，另一个端口用来提供debug信息

```go
package main

import (
	"context"
	"fmt"
	"net/http"
	"time"
)

var serverApp *http.Server
var serverDebug  *http.Server

func main()  {

	done := make(chan error, 2)
	stop := make(chan struct{})

	serAppMux := http.NewServeMux()
	SerDebugMux := http.NewServeMux()

	serAppMux.HandleFunc("/", app)
	SerDebugMux.HandleFunc("/", debug)

	serAppMux.HandleFunc("/shutdown", appShutdown)
	SerDebugMux.HandleFunc("/shutdown", debugShutdown)

	go func() {
		done <- serveAppHTTP("127.0.0.1:8080",serAppMux,stop)
	}()
	go func() {
		done <- serveDebugHTTP("127.0.0.1:8082",SerDebugMux,stop)
	}()
	var stoped bool

	for i:=0; i < cap(done); i++ {
		if err := <-done; err != nil {
			fmt.Println("error: ", err)
		}
		// close(stop)放到这里，当任意一个http server退出时，整个application都会退出
		//if !stoped{
		//	stoped = true
		//	fmt.Println("main goroutine close stop channel")
		//	close(stop)
		//}
	}
	// close(stop)放到这里，app server 和 debug server,退出一个不会影响另一个的运行
	if !stoped{
		stoped = true
		fmt.Println("main goroutine close stop channel")
		close(stop)
	}
	time.Sleep(time.Second)
}

func serveAppHTTP(addr string, handler http.Handler, stop <-chan struct{}) error  {
	serverApp = &http.Server{
		Addr: addr,
		Handler: handler,
	}
	go func() {
		<-stop
		fmt.Println("stop serveApp http server")
		serverApp.Close()
	}()
	return serverApp.ListenAndServe()
}

func serveDebugHTTP(addr string, handler http.Handler, stop <-chan struct{}) error  {
	serverDebug = &http.Server{
		Addr: addr,
		Handler: handler,
	}
	go func() {
		<-stop
		fmt.Println("stop serveDebug http server")
		serverDebug.Close()
	}()
	return serverDebug.ListenAndServe()
}

func debug(w http.ResponseWriter, r *http.Request)  {
	w.Write([]byte("debug info"))
}

func debugShutdown(w http.ResponseWriter, r *http.Request)  {
	w.Write([]byte("debug server shutdown"))
	ctx := context.TODO()
	serverDebug.Shutdown(ctx)
}

func app(w http.ResponseWriter, r *http.Request)  {
	w.Write([]byte("app info"))
}

func appShutdown(w http.ResponseWriter, r *http.Request)  {
	w.Write([]byte("app shutdown"))
	serverApp.Shutdown(context.TODO())

}

```

end

#### 代码优化：抽象公共serve

但是，这样怎么调用shutdown呢？？？

```go
func main() {
	done := make(chan error, 2)
	stop := make(chan struct{})
	// 将这两个serve返回值给 done 后，这两个任何有一个退出都可以在done中发现。
	// 同时通过stop来控制退出
	go func() {
		done <- serveDebug(stop)
	}()
	go func() {
		done <- serveApp(stop)
	}()

	var stopped bool
	for i := 0; i < cap(done); i++ {
		if err := <-done; err != nil {
			fmt.Println("error: %v", err) // 若deug或app中任何一个退出都会打印退出的错误
		}
		// 同时只要有退出，就将stop关闭，那么根据抽象模型可以知道，
		// stop被关闭后deug和app都会唤醒其内部的goroutine，都会取调用shutdown而退出整个serve
		if !stopped {
			stopped = true
			close(stop)
		}
	}
}

func serveApp(stop chan struct{}) error {
	mux := http.NewServeMux()
	mux.HandleFunc("/", func(resp http.ResponseWriter, req *http.Request) {
		fmt.Fprintln(resp, "hello,CQ")
	})
	return serve("0.0.0.0:8080", mux, stop)

}

func serveDebug(stop chan struct{}) error{
	mux := http.NewServeMux()
	mux.HandleFunc("/", func(resp http.ResponseWriter, req *http.Request) {
		fmt.Fprintln(resp, "hello,debug info")
	})
	return serve("127.0.0.1:8082", mux,stop)
}



// 接受一个监听地址，接收一个http.Handler,接收一个stop信号量
func serve(addr string, handler http.Handler, stop <-chan struct{}) error {
	s := &http.Server{
		Addr:    addr,
		Handler: handler,
	}
	// 后台启用一个 goroutine，能够接收一个stop信号，之后会调用shutdown方法
	go func() {
		<-stop // wait for stop signal
		// 调用了shutdown方法之后 ListenAndServe 也会被退出，
		// 因此这个 goroutine 能够决定外面的 goroutine的声明周期
		s.Shutdown(context.Background())
	}()
	return s.ListenAndServe()
}
```

end

### 引用

1. https://www.qetool.com/scripts/view/4589.html  
1. https://tonybai.com/2021/01/08/understand-how-http-package-deal-with-keep-alive-connection/
