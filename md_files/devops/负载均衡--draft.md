## 负载均衡--draft

### 四层负载and七层负载

所谓的7层代理的意思就是收到包之后会按照7层协议重新封装报文。

tcp和http负载不是一个维度，设成tcp的代理只是不能识别7层协议而已、

七层负载的好处就是：负载设备可以通过修改http协议报文实现更多元的负载策略。

比如，通过负载设备加header或者拦截内容或者加xff透传客户端ip等等

负载均衡器通常称为四层交换机或七层交换机。四层交换机主要分析IP层及TCP/UDP层，实现四层流量负载均衡。七层交换机除了支持四层负载均衡以外，还有分析应用层的信息，如HTTP协议URI或Cookie信息。

负载均衡分为L4 switch（四层交换），即在OSI第4层工作，也就是TCP层。四层负载均衡器（如：LVS，F5）不关心应用协议（如HTTP/FTP/MySQL等）。

L7 switch（七层交换），OSI的最高层，应用层。七层负载均衡器（如： HAProxy，MySQL Proxy）能理解应用协议。

#### 应用场景

七层负载均衡器可以是使整个网络更智能化，比如可以通过七层代理将图片类、静态文件类（JS、CSS）请求转发到特定的服务器，利用缓存技术达到更好的性能。

从技术原理上，可以对客户端的请求和服务器的响应进行任意意义上的修改，极大的提升了应用系统在网络层的灵活性。例如Nginx或者Apache上部署的功能可以前移到负载均衡设备上，例如客户请求中的Header重写，服务器响应中的关键字过滤或者内容插入等功能。

针对SYN Flood攻击，四层模型下攻击请求会被转发到后端服务器上，而七层模式下则可在负载均衡器上进行拦截，不影响后台服务器正常运营。针对SQL注入等情况，也可以通过在七层代理设置策略进行特定报文的过滤，从应用层面进一步提高系统整体安全。

总之，七层负载均衡，主要着重于应用HTTP协议，所以其应用范围主要是众多的网站或者内部信息平台等基于B/S开发的系统。 四层负载均衡则对应其他TCP应用，例如基于C/S开发的ERP等系统。

虽然四层代理性能比七层高很多，但目前像Nginx这类负载均衡器已经可以满足大多数场景的需求，应用最多的也是像Nginx这类七层负载均衡器。

https://segmentfault.com/a/1190000041317227

### 会话保持

应用如果做了基于session共享的会话识别前置代理就不用做会话保持了。

#### k8s session会话保持

通过 kubernetes 部署了 tomcat+mysql 服务，设置 tomcat 多副本时发现首页登陆无法跳转的情况，经排查是由于 session 问题引起的。

kubernetes 上可以多实例（pod）高负载运行，但是如果应用如果没有做 session 同步的话，就会导致 session 不一致。
kubernetes 有 session 亲和性的功能（每个 client 每次访问，都会匹配到对应 session 的后端）。

**解决方案**

此时，在 service 的配置文件中加入 `sessionAffinity: ClientIP`，功能是选择与请求来源 ip 更接近的 pod，这样就会固定同一个 session。

#### Long-lived connections don't scale out of the box in Kubernetes

原文：https://learnk8s.io/kubernetes-long-lived-connections

从前端到后端启动每个HTTP请求时，都会打开和关闭一个新的TCP连接。

如果前端每秒向后端发出100个HTTP请求，则在该秒内将打开和关闭100个不同的TCP连接。

如果打开TCP连接并将其重新用于任何后续HTTP请求，则可以改善延迟并节省资源。

HTTP协议具有称为HTTP保持活动或HTTP连接重用的功能，该功能使用单个TCP连接发送和接收多个HTTP请求和响应。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/http-with-keepalived.gif)

它不是现成的;您的服务器和客户端应该配置为使用它，这种改变本身很简单，而且在大多数语言和框架中都可以使用。

但是当在Kubernetes服务中使用keep-alive时会发生什么?

让我们假设前端和后端都支持keep-alive，您有一个前端实例和三个后端副本，前端向后端发出第一个请求并打开TCP连接，请求到达服务，其中一个Pod被选择为目的地，后端Pod响应，前端接收响应。但是，它没有关闭TCP连接，而是为后续的HTTP请求保持打开状态。

当前端发出更多请求时会发生什么?
他们被送到同一个Pod。

难道iptables不应该分配流量吗?
**事实上、它打开了一个TCP连接，第一次调用了iptables规则，三个Pod中的一个被选为目的地，由于所有后续请求都通过同一个TCP连接进行传输，因此iptables不再被调用。**

即，k8s使用了keepalive会导致流量不均匀。

#### server-to-server长连接

长连接是会话保持的基础，如果换了访问节点，会话肯定无法保持了。

#####  基于golang实现

https://its401.com/article/kdpujie/73177179

**一，问题起因**

​    线上server to server的服务，出现大量的TIME_WAIT。用netstat发现，不断的有连接在建立，没有保持住连接。抓TCP包确认request和response中的keepalive都已经设置，但是每个TCP连接处理6次左右的http请求后，就被关闭。

​    就处理该问题的过程中，查看了一下http client的部分源码。

**二，HTTP Client简单结构**

**1，**简单HTTP client定义

```cpp
httpClient := &http.Client{
		Transport: trans,
		Timeout:   config.Client_Timeout * time.Millisecond,
	}
```

Timeout：从发起请求到整个报文响应结束的超时时间。

Transport：为http.RoundTripper接口，定义功能为负责http的请求分发。实际功能由结构体net/http/transport.go中的Transport struct继承并实现，除了请求发分还实现了对空闲连接的管理。如果创建client时不定义，就用系统默认配置。

  **2，**DefaultTransport定义

```cpp
// DefaultTransport is the default implementation of Transport and is
// used by DefaultClient. It establishes network connections as needed
// and caches them for reuse by subsequent calls. It uses HTTP proxies
// as directed by the $HTTP_PROXY and $NO_PROXY (or $http_proxy and
// $no_proxy) environment variables.
var DefaultTransport RoundTripper = &Transport{
	Proxy: ProxyFromEnvironment,
	DialContext: (&net.Dialer{
		Timeout:   30 * time.Second,
		KeepAlive: 30 * time.Second,
		DualStack: true,
	}).DialContext,
	MaxIdleConns:          100,
	IdleConnTimeout:       90 * time.Second,
	TLSHandshakeTimeout:   10 * time.Second,
	ExpectContinueTimeout: 1 * time.Second,
}
```

 net.Dialer.Timeout： 连接超时时间。



net.Dialer.KeepAlive：开启长连接（说明默认http client是默认开启长连接的）。

http.Transport.TLSHandshakeTimeout：限制TLS握手使用的时间。
 http.Transport.ExpectContinueTimeout：限制客户端在发送一个包含：100-continue的http报文头后，等待收到一个go-ahead响应报文所用的时间。

http.Transport.MaxIdleConns：最大空闲连接数。(the maximum number of idle (keep-alive) connections across all hosts. Zero means no limit.)

http.Transport.IdleConnTimeout：连接最大空闲时间，超过这个时间就会被关闭。

**三，问题跟踪-keepAlive设置**

**1.** 按照DefaultTransport自定义Transport后，怎么调整参数，线上问题依旧没有得到解决。怀疑是对keepAlive参数的理解不到位，所以继续看源码中对keepAlive参数的使用。

**2.** net包中net/dial.go中， 使用方法func (d *Dialer) DialContext()创建新连接，有代码片段如下：

```cpp
	if tc, ok := c.(*TCPConn); ok && d.KeepAlive > 0 {
		setKeepAlive(tc.fd, true)
		setKeepAlivePeriod(tc.fd, d.KeepAlive)
		testHookSetKeepAlive()
	}
```

   在Dialer中设置的一个keepalive参数，被分解成了两个分支，一是开关，二是keepalive周期。再继续往下跟踪源码的时候，就开始系统调用了，提取出关键代码如下：

```cpp
setKeepAlive():
syscall.SetsockoptInt(fd.sysfd, syscall.SOL_SOCKET, syscall.SO_KEEPALIVE, boolint(keepalive))
setKeepAlivePeriod():  
syscall.SetsockoptInt(fd.sysfd, syscall.IPPROTO_TCP, sysTCP_KEEPINTVL, secs)  
syscall.SetsockoptInt(fd.sysfd, syscall.IPPROTO_TCP, syscall.TCP_KEEPALIVE, secs) 
```

   大致意思是，首先开启系统socket的SOL_SOCKET设置；然后TCP_KEEPINTVL和TCP_KEEPALIVE用的同一个时间来设置。

**3.** 可以查看linux系统中TCP关于keepalive的三个参数，执行man 7 tcp 命令可以找到以下三个参数：

```plain
 tcp_keepalive_intvl (integer; default: 75; since Linux 2.4)
     The number of seconds between TCP keep-alive probes.

 tcp_keepalive_probes (integer; default: 9; since Linux 2.2)
     The  maximum number of TCP keep-alive probes to send before giving up and killing the connection if no response is obtained
     from the other end.

tcp_keepalive_time (integer; default: 7200; since Linux 2.2)
     The number of seconds a connection needs to be idle before TCP begins sending out keep-alive probes.  Keep-alives are  only
     sent  when  the SO_KEEPALIVE socket option is enabled.  The default value is 7200 seconds (2 hours).  An idle connection is
     terminated after approximately an additional 11 minutes (9 probes an interval of  75  seconds  apart)  when  keep-alive  is
     enabled.

     Note that underlying connection tracking mechanisms and application timeouts may be much shorter.
```

 大意为：要想使用keepalive机制，首先得开启SO_KEEPALIVE设置；然后系统会在connection空闲keepalive_time时间后发起探针，连续keepalive_probes个探针失败时，系统将关闭连接。keepalive_intvl为两次探针的间隔时间。

明白go的keepalive后，理论上应用中的设置是没问题的，实际经过调大调小该参数，也是没有解决保持不住长连接的问题。无奈从Client开始继续看源码...

**四，Transport**

client发起请求一般是由Do(req *Request) (*Response, error)方法开始，而真正处理请求分发的是transport的RoundTrip(*Request) (*Response, error)方法，Transport定义如下：

```cpp
// Transport is an implementation of RoundTripper that supports HTTP,
// HTTPS, and HTTP proxies (for either HTTP or HTTPS with CONNECT).
//
// By default, Transport caches connections for future re-use.
// This may leave many open connections when accessing many hosts.
// This behavior can be managed using Transport's CloseIdleConnections method
// and the MaxIdleConnsPerHost and DisableKeepAlives fields.
//
// Transports should be reused instead of created as needed.
// Transports are safe for concurrent use by multiple goroutines.
//
// A Transport is a low-level primitive for making HTTP and HTTPS requests.
// For high-level functionality, such as cookies and redirects, see Client.
//
// Transport uses HTTP/1.1 for HTTP URLs and either HTTP/1.1 or HTTP/2
// for HTTPS URLs, depending on whether the server supports HTTP/2,
// and how the Transport is configured. The DefaultTransport supports HTTP/2.
// To explicitly enable HTTP/2 on a transport, use golang.org/x/net/http2
// and call ConfigureTransport. See the package docs for more about HTTP/2.
type Transport struct {
	idleMu     sync.Mutex
	wantIdle   bool                                // user has requested to close all idle conns
	idleConn   map[connectMethodKey][]*persistConn // most recently used at end
	idleConnCh map[connectMethodKey]chan *persistConn
	idleLRU    connLRU

	reqMu       sync.Mutex
	reqCanceler map[*Request]func(error)

	altMu    sync.Mutex   // guards changing altProto only
	altProto atomic.Value // of nil or map[string]RoundTripper, key is URI scheme

	// Proxy specifies a function to return a proxy for a given
	// Request. If the function returns a non-nil error, the
	// request is aborted with the provided error.
	// If Proxy is nil or returns a nil *URL, no proxy is used.
	Proxy func(*Request) (*url.URL, error)

	// DialContext specifies the dial function for creating unencrypted TCP connections.
	// If DialContext is nil (and the deprecated Dial below is also nil),
	// then the transport dials using package net.
	DialContext func(ctx context.Context, network, addr string) (net.Conn, error)

	// Dial specifies the dial function for creating unencrypted TCP connections.
	//
	// Deprecated: Use DialContext instead, which allows the transport
	// to cancel dials as soon as they are no longer needed.
	// If both are set, DialContext takes priority.
	Dial func(network, addr string) (net.Conn, error)

	// DialTLS specifies an optional dial function for creating
	// TLS connections for non-proxied HTTPS requests.
	//
	// If DialTLS is nil, Dial and TLSClientConfig are used.
	//
	// If DialTLS is set, the Dial hook is not used for HTTPS
	// requests and the TLSClientConfig and TLSHandshakeTimeout
	// are ignored. The returned net.Conn is assumed to already be
	// past the TLS handshake.
	DialTLS func(network, addr string) (net.Conn, error)

	// TLSClientConfig specifies the TLS configuration to use with
	// tls.Client.
	// If nil, the default configuration is used.
	// If non-nil, HTTP/2 support may not be enabled by default.
	TLSClientConfig *tls.Config

	// TLSHandshakeTimeout specifies the maximum amount of time waiting to
	// wait for a TLS handshake. Zero means no timeout.
	TLSHandshakeTimeout time.Duration

	// DisableKeepAlives, if true, prevents re-use of TCP connections
	// between different HTTP requests.
	DisableKeepAlives bool

	// DisableCompression, if true, prevents the Transport from
	// requesting compression with an "Accept-Encoding: gzip"
	// request header when the Request contains no existing
	// Accept-Encoding value. If the Transport requests gzip on
	// its own and gets a gzipped response, it's transparently
	// decoded in the Response.Body. However, if the user
	// explicitly requested gzip it is not automatically
	// uncompressed.
	DisableCompression bool

	// MaxIdleConns controls the maximum number of idle (keep-alive)
	// connections across all hosts. Zero means no limit.
	MaxIdleConns int

	// MaxIdleConnsPerHost, if non-zero, controls the maximum idle
	// (keep-alive) connections to keep per-host. If zero,
	// DefaultMaxIdleConnsPerHost is used.
	MaxIdleConnsPerHost int

	// IdleConnTimeout is the maximum amount of time an idle
	// (keep-alive) connection will remain idle before closing
	// itself.
	// Zero means no limit.
	IdleConnTimeout time.Duration

	// ResponseHeaderTimeout, if non-zero, specifies the amount of
	// time to wait for a server's response headers after fully
	// writing the request (including its body, if any). This
	// time does not include the time to read the response body.
	ResponseHeaderTimeout time.Duration

	// ExpectContinueTimeout, if non-zero, specifies the amount of
	// time to wait for a server's first response headers after fully
	// writing the request headers if the request has an
	// "Expect: 100-continue" header. Zero means no timeout and
	// causes the body to be sent immediately, without
	// waiting for the server to approve.
	// This time does not include the time to send the request header.
	ExpectContinueTimeout time.Duration

	// TLSNextProto specifies how the Transport switches to an
	// alternate protocol (such as HTTP/2) after a TLS NPN/ALPN
	// protocol negotiation. If Transport dials an TLS connection
	// with a non-empty protocol name and TLSNextProto contains a
	// map entry for that key (such as "h2"), then the func is
	// called with the request's authority (such as "example.com"
	// or "example.com:1234") and the TLS connection. The function
	// must return a RoundTripper that then handles the request.
	// If TLSNextProto is not nil, HTTP/2 support is not enabled
	// automatically.
	TLSNextProto map[string]func(authority string, c *tls.Conn) RoundTripper

	// ProxyConnectHeader optionally specifies headers to send to
	// proxies during CONNECT requests.
	ProxyConnectHeader Header

	// MaxResponseHeaderBytes specifies a limit on how many
	// response bytes are allowed in the server's response
	// header.
	//
	// Zero means to use a default limit.
	MaxResponseHeaderBytes int64

	// nextProtoOnce guards initialization of TLSNextProto and
	// h2transport (via onceSetNextProtoDefaults)
	nextProtoOnce sync.Once
	h2transport   *http2Transport // non-nil if http2 wired up

	// TODO: tunable on max per-host TCP dials in flight (Issue 13957)
}
```

 大意为Transport是一个支持HTTP、HTTPS、HTTP Proxies的RoundTripper，是协程安全的，并默认支持连接池。

从源码能看到，当获取一个IdleConn处理完request后，会调用tryPutIdleConn方法回放conn，代码有这样一个逻辑：

```cpp
	idles := t.idleConn[key]
	if len(idles) >= t.maxIdleConnsPerHost() {
		return errTooManyIdleHost
	}
```

 也就是说IdleConn不仅受到MaxIdleConn的限制，也受到MaxIdleConnsPerHost的限制，DefaultTranspor中是没有设置该参数的，而默认的参数为2.

由于我们业务为server to server，所以是定点访问，经过该参数的调整，服务器上已经保持住稳定的长连接了。



## Client-side load balancing

HTTP isn't the only protocol that can benefit from long-lived TCP connections.

If your app uses a database, the connection isn't opened and closed every time you wish to retrieve a record or a document.

Instead, the TCP connection is established once and kept open.

If your database is deployed in Kubernetes using a Service, you might experience the same issues as the previous example.

There's one replica in your database that is utilised more than the others.

Kube-proxy and Kubernetes don't help to balance persistent connections.

Instead, you should take care of load balancing the requests to your database.

Depending on the library that you use to connect to the database, you might have different options.

The following example is from a clustered MySQL database called from Node.js:

index.js

```js
var mysql = require('mysql');
var poolCluster = mysql.createPoolCluster();

var endpoints = /* retrieve endpoints from the Service */

for (var [index, endpoint] of endpoints) {
  poolCluster.add(`mysql-replica-${index}`, endpoint);
}

// Make queries to the clustered MySQL database
```

As you can imagine, several other protocols work over long-lived TCP connections.

Here you can read a few examples:

- Websockets and secured WebSockets
- HTTP/2
- gRPC
- RSockets
- AMQP

You might recognise most of the protocols above.

*So if these protocols are so popular, why isn't there a standard answer to load balancing?*

*Why does the logic have to be moved into the client?*

*Is there a native solution in Kubernetes?*

Kube-proxy and iptables are designed to cover the most popular use cases of deployments in a Kubernetes cluster.

But they are mostly there for convenience.

If you're using a web service that exposes a REST API, then you're in luck — this use case usually doesn't reuse TCP connections, and you can use any Kubernetes Service.

But as soon as you start using persistent TCP connections, you should look into how you can evenly distribute the load to your backends.

Kubernetes doesn't cover that specific use case out of the box.

However, there's something that could help.

## Load balancing long-lived connections in Kubernetes

k8s没有现成的长连接解决方式--

Kubernetes has four different kinds of Services:

1. ClusterIP
2. NodePort
3. LoadBalancer
4. Headless

The first three Services have a virtual IP address that is used by kube-proxy to create iptables rules.

But the fundamental building block of all kinds of the Services is the Headless Service.

The headless Service doesn't have an assigned IP address and is only a mechanism to collect a list of Pod IP addresses and ports (also called endpoints).

Every other Service is built on top of the Headless Service.

The ClusterIP Service is a Headless Service with some extra features:

- the control plane assigns it an IP address
- kube-proxy iterates through all the IP addresses and creates iptables rules

So you could ignore kube-proxy all together and always use the list of endpoints collected by the Headless Service to load balance requests client-side.

*But can you imagine adding that logic to all apps deployed in the cluster?*

If you have an existing fleet of applications, this might sound like an impossible task.

But there's an alternative.

## Service meshes to the rescue

You probably already noticed that the client-side load balancing strategy is quite standard.

When the app starts, it should

- retrieve a list of IP addresses from the Service
- open and maintain a pool of connections
- periodically refresh the pool by adding and removing endpoints

As soon as it wishes to make a request, it should:

- pick one of the available connections using a predefined logic such as round-robin
- issue the request

The steps above are valid for WebSockets connections as well as gRPC and AMQP.

You could extract that logic in a separate library and share it with all apps.

Instead of writing a library from scratch, you could use a Service mesh such as **Istio** or Linkerd.

Service meshes augment your app with a new process that:

- automatically discovers IP addresses Services
- inspects connections such as WebSockets and gRPC
- load-balances requests using the right protocol

Service meshes can help you to manage the traffic inside your cluster, but they aren't exactly lightweight.

Other options include using a library such as Netflix Ribbon, a programmable proxy such as **Envoy** or just ignore it.

*What happens if you ignore it?*

You can ignore the load balancing and still don't notice any change.

There are a couple of scenarios that you should consider.

1. If you have more clients than servers, there should be limited issues.
2. Imagine you have five clients opening persistent connections to two servers.

Even if there's no load balancing, both servers likely utilised.

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/more-client-than-server.svg)

The connections might not be distributed evenly (perhaps four ended up connecting to the same server), but overall there's a good chance that both servers are utilised.

What's more problematic is the opposite scenario.

If you have fewer clients and more servers, you might have some underutilised resources and a potential bottleneck.

Imagine having two clients and five servers.

At best, two persistent connections to two servers are opened.

The remaining servers are not used at all.

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/more-servers-than-clients.svg)

If the two servers can't handle the traffic generated by the clients, horizontal scaling won't help.

## Summary

Kubernetes Services are designed to cover most common uses for web applications.

However, as soon as you start working with application protocols that use persistent TCP connections, such as databases, gRPC, or WebSockets, they fall apart.

**Kubernetes doesn't offer any built-in mechanism to load balance long-lived TCP connections.**

Kubernetes不提供任何内置机制来平衡长期存在的TCP连接的负载。