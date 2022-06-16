### SpringCloud＋Nacos

Spring是一个体系，Spring Framework是Spring生态体系的基石。

SpringBoot是基于Spring Framework的脚手架，Spring Cloud则是基于Spring Boot应用的微服务架构。

### 资料：

rickiyang： https://www.cnblogs.com/rickiyang/p/11802487.html

https://www.cnblogs.com/rickiyang/p/11802487.html

客户端访问接口由统一流量入口 SpringCloud-Gateway 接收请求、响应结果，网关与微服务基于异步IO Netty通信，微服务获取配置文件启动后通过 Nacos 完成服务注册与发现，微服务之间的相互调用基于http协议的 FeignClient客户端。

### 写的真好：大佬

https://www.cnblogs.com/rickiyang/p/12153070.html



### 实践项目栗子

https://github.com/Zealon159/light-reading-cloud

### Spring cloud + k8s 服务发现:heavy_check_mark:

https://cloud.tencent.com/developer/article/1900990

发现K8S有服务发现功能(Service),而Spring Cloud也有相应的服务发现(Eureka)功能,这两者的功能是不兼容的,你只能二选一.
原文: https://www.lixin.help/2021/01/01/K8S-MicroService-Integrate.html





pod 注册到spring cloud的地址是pod IP的话，集群外的vm怎么访问这个pod注册的服务。

如果，多副本pod注册的是nodeIP+nodeport，vm是可以访问 pod service，但是多副本pod注册时，会不会出现重复注册呢？？？

so，最简单还是要上都上k8s



#### 推荐方案

发现K8S有服务发现功能(Service),而Spring Cloud也有相应的服务发现(Eureka)功能,这两者的功能是不兼容的,你只能二选一.

Ingress + Spring Cloud Gateway + Eureka(Zookeeper/Nacos) + 业务微服务:

1. eureka 和 业务微服务,不使用Service(NodePort)功能(也就是不暴露服务,仅内部访问).
2. 业务微服务 和 Gateway 都向 eureka注册,这时在注册中心的元数据是:PodIP:port.
3. 而Spring Cloud Gateway通过Service(NodePort)暴露出IP和端口.
4. Ingress接受请求,全部转发到:Gateway上,再由:Gateway进行分发到业务微服务上.
5. 优势:不是所有的微服务都要使用Service(NodPort)功能.访方案:占用Node(宿主机)的端口也比较少,对开发比较透明,在开发环境无须依赖K8S,且学习成本比较低.
6. 原理:相当于Gateway往Nginx的upstream中注册,而Gateway和Pod是在同一网段,自然也能相互访问



原文: https://www.lixin.help/2021/01/01/K8S-MicroService-Integrate.html

#### consule: 重复注册

http://www.liuhaihua.cn/archives/522516.html

**Spring Cloud中每个实例启动时都会产生一个唯一的InstanceID,**并且通过InstanceID向Consul中进行注册。不同的InstanceID对于Consul而言就代表着不同的服务实例。但是由于目前将Pod网络方式设置成为了HostNetwork。因此只要是相同主机上启动的服务，其访问地址一定是相同的。 但是反复启动时，Spring Cloud注册会生成不同的InstanceID。 这些对于Consul而言不同的Instance。 实际指向了一个相同的Pod网络（都是hostIP）。 在应用反复升级/部署之后，会发现Consul中存在大量的服务实例，而这些服务实例指向的地址都是相同的。对于这些服务Consul会定期调用Health Check去检查服务可用性。 大量的检查项导致Consul性能下降。为此需要一个简单的解决方案，确保在相同主机上运行的Pod实例，在部署/重启/升级后的InstanceID保持一致。

解决：使用

一个简单的方案就是在InstanceID中取出随机值，并且使用当前Pod的IP地址作为标识，例如： service-192-168-1-2。 这种方式可以确保Pod在默认容器网络或者Host网络下可以保持一致的行为

这里通过将当前Pod的IP地址注入到容器实例的环境变量变量中，并且覆盖默认的–spring.cloud.consul.discover.instanceId来确保注册到Consul中的ServieID唯一。

除了podIP以外，K8S还支持从spec，metadata中获取Pod相关的信息



### Spring  and Spring Framework

Spring是一个体系，Spring Framework是Spring生态体系的基石。

Spring是一个生态体系（也可以说是技术体系），是集大成者，它包含了Spring Framework、Spring Boot、Spring Cloud等（还包括Spring Cloud data flow、spring data、spring integration、spring batch、spring security、spring hateoas。

Spring Framework是整个spring生态的基石，它可是硬生生的消灭了Java官方主推的企业级开发标准EJB，从而实现一统天下。Spring官方对Spring Framework简短描述：为依赖注入、事务管理、WEB应用、数据访问等提供了核心的支持。Spring Framework专注于企业级应用程序的“管道”，以便开发团队可以专注于应用程序的业务逻辑

### SpringBoot and Spring Cloud

SpringBoot是基于Spring Framework的脚手架，Spring Cloud则是基于Spring Boot应用的微服务架构。

Spring Boot这家伙简直就是对Java企业级应用开发进行了一场浩浩荡荡的革命。以前的Java Web开发模式：Tomcat + WAR包。WEB项目基于spring framework，项目目录一定要是标准的WEB-INF + classes + lib，而且大量的xml配置。如果说，以前搭建一个SSH架构的Web项目需要1个小时，那么现在应该10分钟就可以了。

Spring Boot能够让你非常容易的创建一个单机版本、生产级别的基于spring framework的应用。然后，"just run"即可。Spring Boot默认集成了很多第三方包，以便你能以最小的代价开始一个项目。


Spring Boot是build anything。Anything包含很多，其中就包含右侧的Spring Cloud和再右侧的Spring Cloud Data Flow。 

Spring Cloud是Coordinate Anything。下面写的：Built directly on Spring Boot's innovative approach to enterprise Java。

Spring+Cloud包含API+Gateway、Config+Server，Service+Registry，多个MicroServices.

```java
//写一个Config Server还是springboot + 注入其他脚手架
@RestController
@SpringBootApplication
public class GatewaySampleApplication {
		@Value("${remote.home}")
		private URI home;
		@GetMapping("/test")
	public ResponseEntity<?> proxy(ProxyExchange<byte[]> proxy) throws Exception {
			return proxy.uri(home.toString() + "/image/png").get();
	}
}

```

end

### Springboot：配置文件

With file-based (git, svn, and native) repositories, resources with file names in `application*` (`application.properties`, `application.yml`, `application-*.properties`, and so on) are shared between all client applications. You can use resources with these file names to configure global defaults and have them be overridden by application-specific files as necessary.

```yaml
## application.yml
spring:
    application.name: beer-api-producer
    cloud.stream.bindings.output-out-0:
        
        destination: verifications
        
server.port: ${PORT:8080}
logging:
  level:
    org.springframework.cloud: debug
    
## application.properties 
spring.application.name=spring-cloud-producer
server.port=9000

```

#### Use of @Value Annotation：引用变量

The @Value annotation is used to read the environment or application property value in Java code.

实践，在`application.yaml`定义变量

```yaml
server:
  port: 8081
  servlet:
    context-path: /daip-server
  tomcat:
    uri-encoding: UTF-8

# 自定义
waf:
  swagger:
    # 生产环境禁用swagger文档（生产环境启用swagger 需自行评估风险）
    enable: ${SWAGGER_ENABLE:false}
  api:
    encrypt:
      # 是否开启api 接口数据加密（默认为false)
      enable: ${ENCRYPT_ENABLE:true}
    # 消息中心rest服务地址
    message-rest-url: ${MESSAGE_REST_URL:http://daip-message-service:8010/daip-message}
    # 调度中心rest服务地址
    job-rest-url: ${JOB_REST_URL:http://daip-job-service:8011/daip-job}
    # 执行器配置ID
    jobGroupId: ${JOB_GROUP_ID:2}
    # 全文检索rest服务地址
    search-rest-url: ${SEARCH_REST_URL:http://daip-search-service:8013/daip-search}
  server:
    # token 默认30分钟
    token-timeout: ${WAF_TOKEN_TIMEOUT:30}
    # shiro匿名访问的地址配置,多个以逗号分隔
    exclusions: ${WAF_SERVER_EXCLUSIONS:/oauth/**}
  web:
    # 第三方登录成功后，重定向前端的地址
    url: ${WAF_WEB_URL:/hbwebchat/daip-mobile-web/}
...
```

代码中应用自定义的`waf.api.message-rest-url`

```java
// *.java
//    private ISysMessagesService sysMessagesService;
    @Value("${waf.api.message-rest-url:}")
```

0.0

The @Value annotation is used to read the environment or application property value in Java code. The syntax to read the property value is shown below −

```java
@Value("${property_key_name}")
```

Look at the following example that shows the syntax to read the **spring.application.name** property value in Java variable by using @Value annotation.

```java
@Value("${spring.application.name}")
```

Observe the code given below for a better understanding −



#### SpringBoot + Eureka + Feign + Ribbon + Hystrix

SpringCloud全家桶，SpringBoot app定义好service_name/app_name注册到Eureka，为其他组件提供服务。

服务者，提供一个服务

```java
@RestController
public class HelloController {
	
    @RequestMapping("/hello")
    public String index(@RequestParam String name) {
        return "hello "+name+"，this is first messge";
    }
}
```

Feign，中转者.

> Feign是一个声明式Web Service客户端。使用Feign能让编写Web Service客户端更加简单, 它的使用方法是定义一个接口，然后在上面添加注解，同时也支持JAX-RS标准的注解。Feign也支持可拔插式的编码器和解码器。Spring Cloud对Feign进行了封装，使其支持了Spring MVC标准注解和HttpMessageConverters。Feign可以与Eureka和Ribbon组合使用以支持负载均衡。

Feign的原理就是使用的动态代理，通过@FeignClient@RequestsMapping@PathValue或者@RequestBody将请求发送到目标HandleMapping上。

Ribbon就是来告诉Feign具体发送到那个Ip上的一个负载均衡组件。

name:远程服务名即spring.application.name配置的名称

> 用FeignClient注解申明要调用的服务是哪个，该服务中的方法都有我们常见的Restful方式的API来声明，这种方式大家是不是感觉像是在写Restful接口一样。

```java
@FeignClient(name= "spring-cloud-producer")
public interface HelloRemote {
    @RequestMapping(value = "/hello")
    public String hello(@RequestParam(value = "name") String name);
}
```

消费者，调用服务。Web层调用远程服务。

将HelloRemote注入到controller层，像普通方法一样去调用即可。

> 如果，不经过Feign，直接后端调用呢。

```java
@RestController
public class ConsumerController {

    @Autowired
    HelloRemote HelloRemote;
	
    @RequestMapping("/hello/{name}")
    public String index(@PathVariable("name") String name) {
        return HelloRemote.hello(name);
    }

}
```

end

##### Feign

Feign是一个声明式的REST客户端，它的目的就是让REST调用更加简单。通过提供HTTP请求模板，让Ribbon请求的书写更加简单和便捷。另外，在Feign中整合了Ribbon，从而不需要显式的声明Ribbon的jar包。

使用Ribbon+RestTemplate时，利用RestTemplate对http请求的封装处理，形成了一套模版化的调用方法。但是在实际开发中，由于对服务依赖的调用可能不止一处，往往一个接口会被多处调用，所以通常都会针对每个微服务自行封装一些客户端类来包装这些依赖服务的调用。所以，Feign在此基础上做了进一步封装，由他来帮助我们定义和实现依赖服务接口的定义。在Feign的实现下，我们只需创建一个接口并使用注解的方式来配置它(以前是Dao接口上面标注Mapper注解,现在是一个微服务接口上面标注一个Feign注解即可)，即可完成对服务提供方的接口绑定，简化了使用Spring Cloud Ribbon时，自动封装服务调用客户端的开发量。

```java
@FeignClient(name= "eureka-client")
public interface HelloRemote {
}
```

Eureka 分为 Server 端和 Client 端，Client 端作为服务的提供者，将自己注册到 Server 端，Client端高可用的方式是使用多机部署然后注册到Server，Server端为了保证服务的高可用，也可以使用多机部署的方式。

Ribbon使用： 大概就是先定义配置文件`RestTemplate`，在应用到Rest client

使用以下方式配置的策略表示对该项目中调用的所有服务生效。

```java
Copy@Configuration
public class MyConfiguration{
    @Bean
    public IRule ribbonRule(){
        return new RandomRule();
    }
    
    //定义一个负载均衡的RestTemplate
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate(){
        return new RestTemplate();
    }

}
```

上面的配置表示：

1. 定义了一个随机方式的服务调用方式，即随即调用某个服务的提供者；
2. 定义一个负载均衡的 RestTemplate，使用了 @LoadBalanced注解，该注解配合覆盖均衡策略一起使用 RestTemplate 发出的请求才能生效。

RestTemplate是 Spring 提供的用于访问Rest服务的客户端模板工具集，Ribbon并没有创建新轮子，基于此通过负载均衡配置发出HTTP请求。

##### Hystrix

`Spring Cloud Gateway`中如何使用`Hystrix`，主要包括内置的`Hystrix`过滤器和定制过滤器结合`Hystrix`实现我们想要的功能。

分布式系统中一个服务可能依赖着很多其他服务，在高并发的场景下，如何保证依赖的某些服务如果出了问题不会导致主服务宕机这个问题就会变得异常重要。

针对这个问题直观想到的解决方案就是做依赖隔离。将不同的依赖分配到不同的调用链中，某一条链发生失败不会影响别的链。今天要说的Hystrix就提供了这样的功能。Hystrix的作用就是处理服务依赖，帮助我们做服务治理和服务监控。

那么Hystrix是如何解决依赖隔离呢？从官网上看到这样一段：

1. Hystrix使用命令模式`HystrixCommand`(Command)包装依赖调用逻辑，每个命令在单独线程中/信号授权下执行。
2. 可配置依赖调用超时时间,超时时间一般设为比99.5%平均时间略高即可.当调用超时时，直接返回或执行fallback逻辑。
3. 为每个依赖提供一个小的线程池（或信号），如果线程池已满调用将被立即拒绝，默认不采用排队，加速失败判定时间。
4. 依赖调用结果分:成功，失败（抛出异常），超时，线程拒绝，短路。 请求失败(异常，拒绝，超时，短路)时执行fallback(降级)逻辑。
5. 提供熔断器组件,可以自动运行或手动调用,停止当前依赖一段时间(10秒)，熔断器默认错误率阈值为50%,超过将自动运行。

Hystrix现在已经停止更新，意味着你在生产环境如果想使用的话就要考虑现有功能是否能够满足需求。另外开源界现在也有别的更优秀的服务治理组件：Resilience4j 和 Sentinel，如果你有需要可以去看一下它们现在的使用情况。当然这里并不影响我们继续学习Hystrix，毕竟作为分布式依赖隔离的鼻祖，它的设计思想还是需要吃透的。

Hystrix 为每个依赖项维护一个小线程池（或信号量）;如果它们达到设定值（触发隔离），则发往该依赖项的请求将立即被拒绝，执行失败回退逻辑（Fallback），而不是排队。

熔断机制是一种保护性机制，当系统中某个服务失败率过高时，将开启熔断器，对该服务的后续调用，直接拒绝，进行Fallback操作。

熔断所依靠的数据即是Metrics中的HealthCount所统计的错误率。

如何判断是否应该开启熔断器？

必须同时满足两个条件：

1. 请求数达到设定的阀值；
2. 请求的失败数 / 总请求数 > 错误占比阀值%。



Hystrix，熔断。内部使用？？？

[Hystrix](https://github.com/Netflix/Hystrix) is a library from Netflix that implements the [circuit breaker pattern](https://martinfowler.com/bliki/CircuitBreaker.html). The Hystrix GatewayFilter allows you to introduce circuit breakers to your gateway routes, protecting your services from cascading failures and allowing you to provide fallback responses in the event of downstream failures.



#### SpringBoot + k8s

Spring Cloud通过一些列组件实现服务间的调用。

### 网关：Spring Cloud Gateway  and Kong and Zuul

网关作用：

* 流量管控：服务降级、熔断、路由
* 安全防护：风控、白名单/黑名单
* 统一接入：
* 协议适配：长短连接支持、Http和RPC协议转换

http://www.ityouknow.com/springcloud/2018/12/12/spring-cloud-gateway-start.html

### 配置中心/注册中心 - Alibaba-Nacos

#### 配置标识

DataID + Group 

http://www.wenyoulong.com/articles/2021/09/09/1631175873768.html

https://blog.csdn.net/hhhhhhhr/article/details/90447821

##### Data ID 规则

data id=${spring.cloud.nacos.config.prefix}-${spring.profiles.active}.${spring.cloud.nacos.config.file-extension}

${spring.cloud.nacos.config.prefix}默认=${spring.application.name}

${spring.profiles.active}为空则
data id=${spring.cloud.nacos.config.prefix}.${spring.cloud.nacos.config.file-extension}

#### 使用方法

一般情况下在application.yml或application.properties中配置项目的配置信息。

除了application的配置文件，还有一个bootstrap的配置文件，bootstrap由父ApplicationContext加载，在application之前被加载，且属性不能被覆盖，主要用于从额外的资源加载配置信息。

使用配置中心，需要在bootstrap文件中配置spring.application.name和配置中心的相关配置，以便于从配置中心获取配置信息。

```xml
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
        </dependency>
  ....
    </dependencies>


```

end



#### 配置中心

从上面的架构图中我们可以得知，几乎所有的工程都要从配置中心获取配置信息。其目的是用来统一管理配置，配置中心可以在微服务等场景下极大地减轻配置管理的工作量，增强配置管理的服务能力。

> 单体项目的时候，我们把配置信息放到 `.yml` 或 `.properties` 文件中，随着项目走的，一个项目可能有几个配置文件。当请求量随着增大，项目可能要部署多个节点了，这时候维护起来会越来越麻烦，也容易出错。发布的工作降低了整体的工作效率，为了能够提升工作效率，配置中心应运而生了，我们可以将配置统一存放在配置中心来进行管理。

目前主流的配置中心有 Apollo、SpringCloud-Config、Nacos 等开源产品，每款配置中心都能满足统一管理配置的需求，本项目的1.0版本中使用 SpringCloud-Config 作为配置中心、Eureka为注册中心，2.0使用了 Nacos，因为它除了可以做配置中心，还可以做服务注册发现，替代了 Eureka 和 SpringCloud-Config 两个产品。

#### 注册中心

注册中心，是一个独立的服务组件，核心功能是服务治理，集中存储、监控、我们的服务信息。

工作过程简单来说，首先服务提供者启动时，向注册中心提供自己的服务信息，然后消费者服务要请求某个接口时，不是直接去请求具体的服务地址，而是在注册中心拉取得到要请求的服务地址，最后再通过这个地址、端口信息远程调用服务。大体过程如下图：

[![img](https://camo.githubusercontent.com/52e49c9ec85708387400f7c3ffd2e9f91fe0b395ee9225421ca80ae02d405c71/687474703a2f2f72656164696e672e7a65616c6f6e2e636e2f72656769737465722e6a7067)](https://camo.githubusercontent.com/52e49c9ec85708387400f7c3ffd2e9f91fe0b395ee9225421ca80ae02d405c71/687474703a2f2f72656164696e672e7a65616c6f6e2e636e2f72656769737465722e6a7067)

当然服务注册与服务发现的过程并不仅仅只有注册和拉取这两个动作，还有一些其他相关的动作。如注册中心存储数据的缓存更新、提供者服务故障处理、消费者心跳检测等等。



### Spring Boot

#### 通过环境变量使用不同yaml

如下，通过环境变量`SPRING_PROFILE_ACTIVE`决定调用哪个文件

* SPRING_PROFILE_ACTIVE值为pro，则使用`application-pro.yml`
* SPRING_PROFILE_ACTIVE值为dev，则使用`application-dev.yml`

```yaml
## application.yaml
spring:
  profiles:
    # SPRING_PROFILE_ACTIVE可自定义环境变量，默认为pro环境
    active: ${SPRING_PROFILE_ACTIVE:csdev}
  jackson:
    #时区必须要设置
    time-zone: ${JACKSON_TIME_ZONE:GMT+8}
    # 日期格式
    date-format: yyyy-MM-dd HH:mm:ss
    # 日期格式都按timestamps来序列化（不能与 date-format并存，会格式化错误）
#    serialization:
#      write-dates-as-timestamps: true
    #NON_NULL的意思是属性为null时，不输出这个key
    default-property-inclusion: NON_NULL

## 但是实际哪个环境变量生效还得看传入的值
## application.yaml DB_URL的值决定了 db的值
server:
  port: 8011
  servlet:
    context-path: /daip-job
  tomcat:
    uri-encoding: UTF-8

spring:
  autoconfigure:
    # 数据源采用DruidDataSourceAutoConfigure进行自动配置
    exclude: com.alibaba.druid.spring.boot.autoconfigure.DruidDataSourceAutoConfigure
  datasource:
    #  数据连接池类型
    ...
    dynamic:
      druid: # 全局druid参数，绝大部分值和默认保持一致。
        ...
        connectionProperties: druid.stat.mergeSql\=true;druid.stat.slowSqlMillis\=5000
      datasource:
        master:
          url: ${DB_URL:jdbc:mysql://daip-mysql:3306/daip?generateSimpleParameterMetadata=true&useUnicode=true&characterEncoding=utf8&useSSL=false&serverTimezone=Asia/Shanghai}
          username: ${DB_USER:daip}
          password: ${DB_PASS:daip@2020}
          

```

end

#### Spring Boot Define configuration

##### YAML Configuration

As well as Java properties files, we can also use [YAML](https://www.baeldung.com/spring-yaml)-based configuration files in our Spring Boot application. **YAML is a convenient format for specifying hierarchical configuration data.**

Now let's take the same example from our properties file and convert it to YAML:

```yaml
spring:
    datasource:
        password: password
        url: jdbc:h2:dev
        username: SA
```

This can be more readable than its property file alternative since it does not contain repeated prefixes.

YAML has a more concise format for expressing lists:

```java
application:
    servers:
    -   ip: '127.0.0.1'
        path: '/path1'
    -   ip: '127.0.0.2'
        path: '/path2'
    -   ip: '127.0.0.3'
        path: '/path3'
```

end



#### Spring Boot Usage configurations

Now that we've defined our configurations, let's see how to access them.

##### *Value* Annotation

We can inject the values of our properties using the *@Value* annotation:

```java
@Value("${key.something}")
private String injectedProperty;
```

Here the property *key.something* is injected via field injection into one of our objects.

#####  *Environment* Abstraction

We can also obtain the value of a property using the *Environment* API:

```java
@Autowired
private Environment env;

public String getSomeKey(){
    return env.getProperty("key.something");
}
```

##### *ConfigurationProperties* Annotation

Finally, we can also use the *@ConfigurationProperties* annotation to bind our properties to type-safe structured objects:

```java
@ConfigurationProperties(prefix = "mail")
public class ConfigProperties {
    String name;
    String description;
...
```

end