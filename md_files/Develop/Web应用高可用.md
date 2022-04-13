## Web应用高可用

### 应用登录态保存

登录态，mihoyo，让我发懵的词-

应用服务器的高可用架构设计主要基于服务无状态这一特性，但是事实上，业务总
是有状态的，

- 在交易类的电子商务网站，需要有购物车记录用户的购买信息，用户每次
  购买请求都是向购物车中增加商品
- 在社交类的网站中，需要记录用户的当前登录状态、最新发布的消息及好友状态等，用户每次刷新页面都需要更新这些信息

Web 应用中将这些多次请求修改使用的上下文对象称作会话(Session)
单机情况下,Session 可由部署在服务器上的Web 容器( 如Tomcat) 管理
但在使用负载均衡的集群环境中，由于负载均衡服务器可能会将请求分发到集群中的任何一台应用服务器上，所以保证每次请求依然能够获得正确的Session比单机时要复杂很多

### Session绑定/粘滞sticky：依赖负载

可以利用负载均衡的源地址Hash算法，纯负载均衡设备实现

负载均衡服务器总是将来源于同一IP的请求分发到同一台服务器上(也可以根据Cookie信息将同一个户的请求总是分发到同一台服务器上,当然这时负载均衡服务器必须工作在HTTP 协议层)
这样在整个会话期间,用户所有的请求都在同一台服务器上处理,即Session绑定在某台特定服务器上,保证Session总能在这台服务器上获取。

#### pros and cons

pros：

不需要依赖应用作额外开发和支持，依赖前面负载设备实现，比如Nginx、HAproxy、F5、LVS等

cons：

一旦某台服务器宕机,那么该机器上的Session也就不复存在了,用户请求切换到其他机器后因为没有Session而无法完成业务处理

因此虽然大部分负载均衡服务器都提供源地址负载均衡算法,但很少有网站利用这个算法进行Session管理。



### Session复制/共享：依赖架构

在访问系统的会话过程中，用户登录系统后，不管访问系统的任何资源地址都不需要重复登录，这里面servlet容易保存了该用户的会话(session)。如果两个tomcat(A、B)提供集群服务时候，用户在A-tomcat上登录，接下来的请求web服务器根据策略分发到B-tomcat，因为B-tomcat没有保存用户的会话(session)信息，不知道其已经登录，会跳转到登录界面。
这时候需要让B-tomcat也保存有A-tomcat的会话，我们可以使用tomcat的session复制实现或者通过其他手段让session共享。

#### 基于jvm共享

terracotta是jvm级别的session共享

它基本原理是对于集群间共享的数据，当在一个节点发生变化的时候，Terracotta只把变化的部分发送给Terracotta服务器，然后由服务器把它转发给真正需要这个数据的节点，并且共享的数据对象不需要序列化。

#### 基于memory共享

通过memcached-session-manager（msm）插件，通过tomcat上一定的配置，即可实现把session存储到memcached服务器上。注意：tomcat支持tomcat6+，并且memcached可以支持分布式内存，msm同时支持黏性session（sticky sessions）或者非黏性session（non-sticky sessions）两种模式，在memcached内存中共享的对象需要序列化。

#### pros and cons：

pros：



cons:

- 当集群规模较大时,集群服务器间需要大量的通信进行Session复制,占用服务器和网络的大量资源,系统不堪负担
- 而且由于所有用户的Session信息在每台服务器上都有备份,在大量用户访问的情况下,甚至会出现服务器内存不够Session使用的情况
- 而大型网站的核心应用集群就是数千台服务器,同时在线用户可达千万,因此并不适用这种方案

### Cookie记录session: 依赖开发

将存在服务端的信息，转存到用户端即浏览器

早期系统使用C/S架构,一种管理Session的方式是将Session记录在客户端,每次请求服务器的时候,将Session放在请求中发送给服务器,服务器处理完请求后再将修改过的Session响应给客户端
如今的B/S架构,网站没有客户端,但是可以利用刘览器支持的Cookie记录Session。

#### pros and cons

由于Cookie的

- 简单易用
- 可用性高
- 支持应用服务器的线性伸缩
- 而大部分应用需要记录的Session 信息又比较小

因此，许多网站都或多或少地使用Cookie记录Session。

缺点如下

- 受Cookie大小限制,能记录的信息有限
- 每次请求响应都需要传输Cookie,影响性能
- 如果用户关闭Cookie,访问就会不正常
- 加大了数据库的负载，使数据库成为集群的瓶颈。

### Session服务器:eagle:

利用Session服务器共享Session

这种方案事实上是将应服务器的状态分离,分为

- 无状态的应用服务器
- 有状态的Session服务器

然后针对这两种服务器的不同特性分别设计其架构

对于有状态的Session服务器,一种比较简单的方法是利用

- 分布式缓存
  即使用cacheDB存取session信息，应用服务器接受新请求将session信息保存在cache DB中，当应用服务器发生故障时，web服务器会遍历寻找可用节点，分发请求，当应用服务器发现session不在本机内存时，则去cache DB中查找，如果找到则复制到本机，这样实现session共享和高可用。

  redis这些都可以吧

- 数据库等

在这些产品的基础上进行包装,使其符合Session 的存储和访问要求。如果业务场景对Session 管理有比较高的要求，比如利用Session 服务集成单点登录(SSO)、用户服务等功能，则需要开发专门的Session服务管理平台。

### 引用

1. https://jishusuishouji.github.io/2017/04/07/web/%E6%B5%85%E8%B0%88web%E5%BA%94%E7%94%A8%E7%9A%84%E8%B4%9F%E8%BD%BD%E5%9D%87%E8%A1%A1%E3%80%81%E9%9B%86%E7%BE%A4%E3%80%81%E9%AB%98%E5%8F%AF%E7%94%A8_HA_%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88/
2. https://developer.aliyun.com/article/636075