### Spring boot项目容器部署实践-- H3C

Spring Boot makes it easy to create stand-alone, production-grade Spring based Applications that you can "just run".

Spring Boot是个脚手架帮你快速生成Spring框架项目所需的配置。

### 官方初始化springboot工具

https://start.spring.io/

不赖~~

### cannot load configuration class

现象：docker maven 中 `mvn package`一个spring boot demo，但是运行jar时报错

提示:

```bash
2022-05-09 16:17:08.777 ERROR 1026 --- [           main] o.s.boot.SpringApplication               : Application startup failed

java.lang.IllegalStateException: Cannot load configuration class: Demo_Class_name

```

解决，替换Pom.xml中springframework.boot的版本

```xml
 <parent>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-parent</artifactId>
                <version>2.2.4.RELEASE</version>
                <relativePath/> <!-- lookup parent from repository -->
        </parent>

```

end

### 面临问题

Pod IP不固定，没有spring cloud的Eureka注册中心，且Spring boot依赖配置文件指定其他服务模块访问地址（IP＋ Port）。

### 解决：Spring Boot Profile + Inject env to pod

Profile是Spring对不同环境提供不同配置功能的支持，可以通过激活、指定参数等方式快速切换环境

多Profile文件格式：`application-{profile}.yml`

> Spring boot 安装自己定义的格式，去读取配置文件。
>
> test环境配置就是`application-test.yml`

利用spring框架可以读本机环境变量的功能，只要传进去指定的环境变量，框架自带的功能(估计是个jar)可以读取环境变量并自动渲染即可。

使用环境变量方式替代spring boot 配置文件中调用其他服务使用固定IP+端口的配置。

四个yaml文件，开发、测试和生产各一个配置文件，一个`application.yaml`引用其中某个环境变量。

只需给Pod注入环境变量即可。

如下：

````bash
## application.yml
spring:
  profiles:
    # 指定读取SPRING_PROFILE_ACTIVE环境变量，并一个缺省值为pro即使用application-pro.yml配置
    active: ${SPRING_PROFILE_ACTIVE:pro}
  jackson:
    #时区必须要设置
    time-zone: ${JACKSON_TIME_ZONE:GMT+8}
    #日期格式
    date-format: yyyy-MM-dd HH:mm:ss
    #ALWAYS的意思是即时属性为null，仍然也会输出这个key
    default-property-inclusion: ALWAYS
    
##  application-pro.yml、
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
    druid:
      stat-view-servlet:
        enabled: ${DRUID_STAT_ENABLED:true}
        loginUsername: ${DRUID_USERNAME:admin}
        loginPassword: ${DRUID_PASSWORD:wiseda@123456}
        allow:
      web-stat-filter:
        enabled: true
    dynamic:
      druid: # 全局druid参数，绝大部分值和默认保持一致。
        # 连接池的配置信息
        # 初始化大小，最小，最大
        initial-size: ${DRUID_INITIAL_SIZE:5}
        min-idle: ${DRUID_MIN_IDLE:5}
        maxActive: ${DRUID_MAX_ACTIVE:20}
        # 配置获取连接等待超时的时间
        maxWait: 60000
        # 配置间隔多久才进行一次检测，检测需要关闭的空闲连接，单位是毫秒
        timeBetweenEvictionRunsMillis: 60000
        # 配置一个连接在池中最小生存的时间，单位是毫秒
        minEvictableIdleTimeMillis: 300000
        validationQuery: SELECT 1 FROM DUAL
        testWhileIdle: true
        testOnBorrow: false
        testOnReturn: false
        # 打开PSCache，并且指定每个连接上PSCache的大小
        poolPreparedStatements: true
        maxPoolPreparedStatementPerConnectionSize: 20
        # 配置监控统计拦截的filters，去掉后监控界面sql无法统计，'wall'用于防火墙
        filters: stat,wall,slf4j
        # 通过connectProperties属性来打开mergeSql功能；慢SQL记录
        connectionProperties: druid.stat.mergeSql\=true;druid.stat.slowSqlMillis\=5000
      datasource:
        master:
          url: ${DB_URL:jdbc:mysql://daip-mysql:3306/daip?generateSimpleParameterMetadata=true&useUnicode=true&characterEncoding=utf8&useSSL=false&serverTimezone=Asia/Shanghai}
          username: ${DB_USER:daip}
          password: ${DB_PASS:daip@2020}
          driver-class-name: ${DB_DRIVER:com.mysql.cj.jdbc.Driver}
  # redis 缓存配置
  redis:
    # Redis服务器地址（本地测试，可配置host）
    host: ${REDIS_HOST:daip-redis-master}
    # nodeport映射端口
    port: ${REDIS_PORT:6379}
    # Redis数据库索引（默认为0）
    database: ${REDIS_DB_NUMBER:2}
    # Redis服务器连接密码（默认为空）
    password: ${REDIS_PASSWORD:s(l(YBaEv1m<DaWY}
    # 连接超时时间（毫秒）
    timeout: 10000
    # Lettuce
    lettuce:
      pool:
        # 连接池最大连接数（使用负值表示没有限制）
        max-active: ${REDIS_MAX_ACTIVE:8}
        # 连接池中的最小空闲连接
        min-idle: ${REDIS_MIN_IDLE:1}
        # 连接池中的最大空闲连接
        max-idle: ${REDIS_MAX_IDLE:5}
        # 连接池最大阻塞等待时间（使用负值表示没有限制）
        max-wait: ${REDIS_MAX_WAIT:-1}
#mybatis plus 设置
mybatis-plus:
  #  mapper-locations: classpath*:com/wiseda/**/xml/*Mapper.xml
  global-config:
    # 关闭MP3.0自带的banner
    banner: false
    db-config:
      #主键类型  AUTO:"数据库ID自增",NONE:"该类型为未设置主键类型", INPUT:"用户输入ID",ASSIGN_ID:"全局唯一ID (数字类型唯一ID)", ASSIGN_UUID:"全局唯一ID UUID"";
      id-type:  ASSIGN_UUID
  configuration:
    # 返回类型为Map,显示null对应的字段
    call-setters-on-nulls: true
# 日志级别配置
logging:
  level:
    root: info
    com:
      wiseda: debug
# XXL-JOB 控制器注册配置
xxl:
  job:
    admin:
      addresses: ${XXL_ADDRESSES:http://daip-job-admin:8300/job-admin}
    executor:
      appname: ${XXL_APPNAME:daip-job-executor}
      port: ${XXL_PORT:9998}
      logpath: ${XXL_LOGPATH:/data/applogs/daip-job/jobhandler}
waf:
  server:
    # token 默认30分钟
    token-timeout: ${WAF_TOKEN_TIMEOUT:30}
  api:
    # 创新应用平台rest服务地址
    server-rest-url: ${SERVER_REST_URL:http://d.wiseda.cn/daip-server}
    # 全文检索rest服务地址
    search-rest-url: ${SEARCH_REST_URL:http://d.wiseda.cn/daip-search}
    # 消息中心rest服务地址
    message-rest-url:  ${MESSAGE_REST_URL:http://d.wiseda.cn/daip-message}

````

end

### Eureka注册

eureka的client注册到server时默认是使用hostname而不是ip,这就导致client在多台机器时，服务间相互调用时也会使用hostname进行调用，从而调用失败。
这时候就需要使用ip来服务到eureka-server上，需要在eureka的client增加配置：

`eureka.instance.prefer-ip-address=true`将instacne hostname解析成IP

然后增加`eureka.instance.instance-id`属性来配置services的注册instance

```bash
$ cat application.properties

server.port=8903
spring.application.name=service-client
eureka.client.service-url.defaultZone=${EUREKA_URL:http://springcloud-discover.springcloudtiizxr06:8761}
eureka.instance.prefer-ip-address=true
eureka.instance.instance-id=${spring.cloud.client.ip-address}:${server.port}

```

end

### Servlet and Tomcat 

在JavaEE平台上，处理TCP连接，解析HTTP协议这些底层工作统统扔给现成的Web服务器去做，我们只需要把自己的应用程序跑在Web服务器上。为了实现这一目的，JavaEE提供了Servlet API，我们使用Servlet API编写自己的Servlet来处理HTTP请求，Web服务器实现Servlet API接口，实现底层功能：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/servlet.jpg)

#### Servelet  and tomcat

**servlet就是一个Java接口**，interface! 打开idea，ctrl + shift + n，搜索servlet，就可以看到是一个只有5个方法的interface!

**网络协议、http什么的，servlet根本不管！也管不着！**

**那servlet是干嘛的？很简单，接口的作用是什么？规范呗！**

servlet接口定义的是一套处理网络请求的规范，所有实现servlet的类，都需要实现它那五个方法，其中最主要的是两个生命周期方法 init()和destroy()，还有一个处理请求的service()，也就是说，所有实现servlet接口的类，或者说，所有想要处理网络请求的类，都需要回答这三个问题：

- 你初始化时要做什么
- 你销毁时要做什么
- 你接受到请求时要做什么

servlet是一个规范，那实现了servlet的类，就能处理请求了吗？

**答案是，不能。**

你可以随便谷歌一个servlet的hello world教程，里面都会让你写一个servlet，相信我，**你从来不会在servlet中写什么监听8080端口的代码，servlet不会直接和客户端打交道！**

那请求怎么来到servlet呢？答案是servlet容器，比如我们最常用的tomcat，同样，你可以随便谷歌一个servlet的hello world教程，里面肯定会让你把servlet部署到一个容器中，不然你的servlet压根不会起作用。

**tomcat才是与客户端直接打交道的家伙**，他监听了端口，请求过来后，根据url等信息，确定要将请求交给哪个servlet去处理，然后调用那个servlet的service方法，service方法返回一个response对象，tomcat再把这个response返回给客户端。



作者：柳树
链接：https://www.zhihu.com/question/21416727/answer/339012081
来源：知乎
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

#### servelet API

现一个最简单的Servlet：

```java
// WebServlet注解表示这是一个Servlet，并映射到地址/:
@WebServlet(urlPatterns = "/")
public class HelloServlet extends HttpServlet {
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {
        // 设置响应类型:
        resp.setContentType("text/html");
        // 获取输出流:
        PrintWriter pw = resp.getWriter();
        // 写入响应:
        pw.write("<h1>Hello, world!</h1>");
        // 最后不要忘记flush强制输出:
        pw.flush();
    }
}
```

一个Servlet总是继承自`HttpServlet`，然后覆写`doGet()`或`doPost()`方法。注意到`doGet()`方法传入了`HttpServletRequest`和`HttpServletResponse`两个对象，分别代表HTTP请求和响应。我们使用Servlet API时，并不直接与底层TCP交互，也不需要解析HTTP协议，因为`HttpServletRequest`和`HttpServletResponse`就已经封装好了请求和响应。以发送响应为例，我们只需要设置正确的响应类型，然后获取`PrintWriter`，写入响应即可。

现在问题来了：Servlet API是谁提供？

Servlet API是一个jar包，我们需要通过Maven来引入它，才能正常编译。编写`pom.xml`文件如下：

```
<project xmlns="http://maven.apache.org/POM/4.0.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.itranswarp.learnjava</groupId>
    <artifactId>web-servlet-hello</artifactId>
    <packaging>war</packaging>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <maven.compiler.source>11</maven.compiler.source>
        <maven.compiler.target>11</maven.compiler.target>
        <java.version>11</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>javax.servlet</groupId>
            <artifactId>javax.servlet-api</artifactId>
            <version>4.0.0</version>
            <scope>provided</scope>
        </dependency>
    </dependencies>

    <build>
        <finalName>hello</finalName>
    </build>
</project>
```

注意到这个`pom.xml`与前面我们讲到的普通Java程序有个区别，打包类型不是`jar`，而是`war`，表示Java Web Application Archive：

```
<packaging>war</packaging>
```

引入的Servlet API如下：

```
<dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>javax.servlet-api</artifactId>
    <version>4.0.0</version>
    <scope>provided</scope>
</dependency>
```

注意到`<scope>`指定为`provided`，表示编译时使用，但不会打包到`.war`文件中，因为运行期Web服务器本身已经提供了Servlet API相关的jar包。

我们还需要在工程目录下创建一个`web.xml`描述文件，放到`src/main/webapp/WEB-INF`目录下（固定目录结构，不要修改路径，注意大小写）。文件内容可以固定如下：

```
<!DOCTYPE web-app PUBLIC
 "-//Sun Microsystems, Inc.//DTD Web Application 2.3//EN"
 "http://java.sun.com/dtd/web-app_2_3.dtd">
<web-app>
  <display-name>Archetype Created Web Application</display-name>
</web-app>
```

运行Maven命令`mvn clean package`，在`target`目录下得到一个`hello.war`文件，这个文件就是我们编译打包后的Web应用程序。

#### 运行war

我们应该如何运行这个`war`文件？

普通的Java程序是通过启动JVM，然后执行`main()`方法开始运行。但是Web应用程序有所不同，我们无法直接运行`war`文件，必须先启动Web服务器，再由Web服务器加载我们编写的`HelloServlet`，这样就可以让`HelloServlet`处理浏览器发送的请求。

因此，我们首先要找一个支持Servlet API的Web服务器。常用的服务器有：

- [Tomcat](https://tomcat.apache.org/)：由Apache开发的开源免费服务器；
- [Jetty](https://www.eclipse.org/jetty/)：由Eclipse开发的开源免费服务器；
- [GlassFish](https://javaee.github.io/glassfish/)：一个开源的全功能JavaEE服务器。

还有一些收费的商用服务器，如Oracle的[WebLogic](https://www.oracle.com/middleware/weblogic/)，IBM的[WebSphere](https://www.ibm.com/cloud/websphere-application-platform/)。

无论使用哪个服务器，只要它支持Servlet API 4.0（因为我们引入的Servlet版本是4.0），我们的war包都可以在上面运行。

#### Tomcat and Sprint boot

正常情况下，我们开发 SpringBoot 项目，由于内置了Tomcat，所以项目可以直接启动，部署到服务器的时候，直接打成 jar 包，就可以运行了 (使用内置 Tomcat 的话，可以在 application.yml 中进行相关配置)

有时我们会需要打包成 war 包，放入外置的 Tomcat 中进行运行，步骤如下 (此处我用的 SpringBoot 版本为 2.1.1，Tomcat 的版本为 8.0)


##### change jar to war

https://spring.io/blog/2014/03/07/deploying-spring-boot-applications

将spring boot jar 变成war，五步

* 排除spring boot内置 Tomcat
* 将打包方式更改为 war
* 修改启动类，使启动类继承 SpringBootServletInitializer 类。
*  SpringBootServletInitializer 类需要用到 servlet-api 的相关 jar 包，所以需要添加依赖。
* 配置外置 Tomcat

But, I imagine you wondering, “how do I deploy it to an *existing* Tomcat installation, or to the classic Java EE application servers (some of which cost a lot of money!) like WebSphere, WebLogic, or JBoss?” Easy! It’s still just Spring, after all, so very little else is required. You’ll need to make three intuitive changes: [move from a `jar` build to a `war` build in Maven](https://spring.io/guides/gs/convert-jar-to-war/): comment out the declaration of the `spring-boot-maven-plugin` plugin in your `pom.xml` file, then change the Maven `packaging` type to `war`. Finally, add a web entry point into your application. Spring configures almost everything for you using Servlet 3 Java configuration. You just need to give it the opportunity. Modify your `Application` entry-point class thusly:

```java
package demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.EnableAutoConfiguration;
import org.springframework.boot.builder.SpringApplicationBuilder;
import org.springframework.boot.context.web.SpringBootServletInitializer;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@Configuration
@ComponentScan
@EnableAutoConfiguration
public class Application extends SpringBootServletInitializer {

    public static void main(String[] args) {
        SpringApplication.run(applicationClass, args);
    }

    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
        return application.sources(applicationClass);
    }

    private static Class<Application> applicationClass = Application.class;
}


@RestController
class GreetingController {

    @RequestMapping("/hello/{name}")
    String hello(@PathVariable String name) {
        return "Hello, " + name + "!";
    }
} COPY
```

This new base class - `SpringBootServletInitializer` - taps into a Servlet 3 style Java configuration API which lets you describe in code what you could only describe in `web.xml` before. Such configuration classes are discovered and invoked at application startup. This gives Spring Boot a chance to tell the web server about the application, including the reqired `Servlet`s, `Filter`s and `Listener`s typically required for the various Spring projects.

This new class can now be used to run the application using embeddedd Jetty or Tomcat, internally, and it can be deployed to any Servlet 3 container. You may experience issues if you have classes that conflict with those that ship as parter of a larger application server. In this case, use your build tool’s facilities for excluding or making `optional` the relevant APIs. Here are the changes to the Maven build that I had to make to get the starter Spring Boot REST service up and running on JBoss WildFly (the AS formerly known as JBoss AS):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>org.demo</groupId>
	<artifactId>demo</artifactId>
	<version>0.0.1-SNAPSHOT</version>

    <packaging>war</packaging>
	<description>Demo project</description>

	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>1.0.0.BUILD-SNAPSHOT</version>
	</parent>

	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter-tomcat</artifactId>
                </exclusion>
            </exclusions>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
		</dependency>
	</dependencies>

	<properties>
        <start-class>demo.Application</start-class>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.7</java.version>
	</properties>
	<repositories>
		<repository>
			<id>spring-snapshots</id>
			<name>Spring Snapshots</name>
			<url>http://repo.spring.io/snapshot</url>
			<snapshots>
				<enabled>true</enabled>
			</snapshots>
		</repository>
		<repository>
			<id>spring-milestones</id>
			<name>Spring Milestones</name>
			<url>http://repo.spring.io/milestone</url>
			<snapshots>
				<enabled>false</enabled>
			</snapshots>
		</repository>
	</repositories>
	<pluginRepositories>
		<pluginRepository>
			<id>spring-snapshots</id>
			<name>Spring Snapshots</name>
			<url>http://repo.spring.io/snapshot</url>
			<snapshots>
				<enabled>true</enabled>
			</snapshots>
		</pluginRepository>
		<pluginRepository>
			<id>spring-milestones</id>
			<name>Spring Milestones</name>
			<url>http://repo.spring.io/milestone</url>
			<snapshots>
				<enabled>false</enabled>
			</snapshots>
		</pluginRepository>
	</pluginRepositories>
</project>
```

I was then able to re-run the build and `cp` the built `.war` to the `$WILDFLY_HOME/standalone/deployments` directory.

Start the application server if it’s not already running, and you should then be able to bring the application up at `http://localhost:8080/$YOURAPP/hello/World`. Again, I’ve substituted `$YOURAPP` for the name of your application, as built.



### SpringBoot

SpringBoot框架, 会从这几个环境变量中读取数据库的连接信息：`DB_URL, DB_USER and DB_PASS`

所以，SpringBoot应用在容器里运行时，需要通过环境变量声明DB信息。

```bash
 ## 如下，定义了环境变量名和缺省值。
 datasource:
        master:
          url: ${DB_URL:jdbc:mysql://daip-mysql:3306/daip?generateSimpleParameterMetadata=true&useUnicode=true&characterEncoding=utf8&useSSL=false&serverTimezone=Asia/Shanghai}
          username: ${DB_USER:daip}
          password: ${DB_PASS:daip@2020}
          driver-class-name: ${DB_DRIVER:com.mysql.cj.jdbc.Driver}
		  
```

end



### vue访问后端配置：vue仅处理DOM，

项目是vue + spring boot组成

**vue-cli3 脚手架搭建完成后，项目目录中没有 vue.config.js 文件，需要手动创建**

#### `.env` and `.env.XXX`

https://cli.vuejs.org/zh/guide/mode-and-env.html

**模式**是 Vue CLI 项目中一个重要的概念。默认情况下，一个 Vue CLI 项目有三个模式：

- `development` 模式用于 `vue-cli-service serve`
- `test` 模式用于 `vue-cli-service test:unit`
- `production` 模式用于 `vue-cli-service build` 和 `vue-cli-service test:e2e`

注意：.env文件无论是开发还是生成都会加载的公用文件

根据启动命令vue会自动加载对应的环境，vue是根据文件名进行加载的，比如执行npm run serve命令，会自动加载.env.development文件

属性名必须以`VUE_APP_`开头，比如`VUE_APP_URL` 、 `VUE_APP_XXX`

```bash
$ ls .env.*
.env .env.development .env.production
$ cat .env.production
#生产环境
NODE_ENV="production"

# api 数据是否加密
VUE_APP_API_ENCRYPT = true

# 因webseal没有开启websocket代理，这里采用企业微信地址代理
VUE_APP_MESSAGE_API = '/daip-message'

# qiankun应用入口
VUE_APP_QIANKUN_APIWEB_ENTRY = '//d.wiseda.cn/waf-apiweb/'

```

end

#### vue + axios + vue proxy：重要

解决: 将vue要掉的后端接口通过proxy指向测试环境的后端地址。

https://www.cnblogs.com/code-duck/p/13414099.html

Axios 是一个基于 promise 的 HTTP 库，可以用在浏览器和 node.js 中。

axios前端通信框架，因为vue的边界很明确，就是为了处理DOM，所以并不具备通信功能，此时就需要额外使用一个通信框架与服务器交互；当然也可以使用jQuery提供的AJAX通信功能。

Axios特性：

1、可以在浏览器中发送 XMLHttpRequests
2、可以在 node.js 发送 http 请求
3、支持 Promise API
4、拦截请求和响应
5、转换请求数据和响应数据
6、能够取消请求
7、自动转换 JSON 数据
8、客户端支持保护安全免受 XSRF 攻击

Vue-Axios实现跨域请求

`.env`配置文件

```js
VUE_APP_BASE_API=/server
```

`request.js`

```js
import axios from 'axios'
const test = axios.create({
  baseURL: process.env.VUE_APP_BASE_API, // api 的 base_url
  timeout: 50000, // request timeout
  headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
  withCredentials: true
})

export default test
```

创建api请求接口

```js
import request from './request.js'

/**
 * 查询所有用户信息
 */
export function list () {
  return request({
    url: '/all/users',
    method: 'get'
  })
}
```

**配置`vue.config.js`代理请求**

```js
module.exports = {
  devServer: {
    port: 8000, // 改端口号
    open: true,
    proxy: {
      // 以server开头的请求才会使用到该代理，即http://localhost:8000/server/query/users.
      '/server': {
        target: 'http://localhost:8081/', // 服务器地址
        changeOrigin: true, // 开启跨域
        pathRewrite: {
           // 当请求以/server开头时，将其置为空 则请求最终为http://localhost:8081/query/users
          '/server': '' 
        }
      }
    }
  }
}
```

end

**前端访问地址为：**http://localhost:8000/server/query/users

**会被代理解析为：**http://localhost:8081/query/users 访问到服务器端获取数据

即本地项目运行端口为8000，访问`IP:8000/server`会代理到`IP/8081/`.

0.0 Vue自带的反向代理。

#### vue + axios + kong/nginx：总结

生成环境： vue定义后端接口的url，然后通过网关kong或者nginx实现后端的访问。

1. 定义后端接口url即`VUE_APP_XXX=xxx`

2. 创建axios访问后端接口即当前IP_Port+url

3. 根据后端url去访问接口即要保证vue监听的IP/域名加上vue定义的url可以访问到后端接口。

   到这里就简单了，这里可以是kong网关上注册的接口，可以是nginx配置的反向代理，甚至可以是vue proxy自己指定的代理。

k8s + kong环境是：先将vue要调用的后端服务在kong上配置，然后就可以访问了。因为vue代码里指明了要访问的url。

kong负责了将解析代理vue要访问的后端url。

1. vue在kong上注册了`/daip-server`，且代码定义了要访问`/daip-job`
2. daip-job组件在kong上注册了`/daip-job`服务，即可

nginx：

1. nginx 代理了vue 服务`location /daip-server`
2. nginx 代理了daip-job 服务`location /daip-job`

#### vue 构建

小野哥，通过docker build env 传入不同的env，使用不同的配置文件，构建出测试和生产不同的镜像。

主要是，数据库和调用其他组件的配置信息不同。



### 编程命名规则

编程中最难问题之一是：**如何命名？**

命名的合理性 影响着 项目的可维护性、代码阅读速度。最理想的命名应该是 **“一眼就能看出这个变量代表着什么”** 。

合理命名的关键问题又在于，**如何“分隔多个单词”？** 有的通过大小写来分隔，有的用下划线/中划线分隔。 主流是使用大小写（骆驼(camel)、帕斯卡（pascal）），C/C++的偏向全小写加下划线，少量情况下会有使用连字符（中线命名法，烤串命名法 (kebab-case)）、全大写加下划线（常用于全局常量）。

大小写是主流的写法，其主要规则是：

- 类名 首字母大写 `class CpuTemperature {...}`
- 变量 首字母小写 `int cupTemperature = ....`
- 接口 首字母以大写I开关，后接大写开头的名称。 `interface ICpuTemperatuer {...}`
- 当出现 大写缩写的单词，两个字母的大写，两个字母以上的按单词算。（比如IO就全大写，`IOException`，CPU就被写成首字母大写,`CpuTemperature`）

全小写加下划线的写法：`cpu_temperature` ，这种写法识别速度最快，在c/c++，mysql字段命名里比较常见，所以常见一些工具用于 大小写与下划线 写法相互转换的工具和代码库。

全大写加下划线的写法：`CPU_TEMPERATURE`，常用于常量声明

连字符写法：`cpu-temperature`，常见于对大小写不敏感的环境，常见于HTML、CSS。

特别地，部分环境对大小写不敏感（即不能区分大小写，如`Cpu`与`CPU`会被解析成一样的，比如HTML就不敏感），所以就不会使用主流的大小写的写法，而更倾向于使用连字符写法。

或许有人问，连字符写法与下划线写法有什么区别？

| 对比项 | 下划线               | 连字符                                                       |
| ------ | -------------------- | ------------------------------------------------------------ |
| 写法   | cpu_temperature      | cpu-temerature                                               |
| 双击   | 是一个单词，双击全选 | 是多个单词，双击只能选中一个单词，Google 搜索引擎也会以此为依据区分是单个单词还是多个。 |
| 输入   | 需要多按一个shift    | 略                                                           |
| 问题   | 略                   | 连字符对应代码里的减号，不能直接使用，在一些环境中使用连字符作为文件名会导致异常。 |

end

### CloudOS 发布流程

基础镜像：必须保证一致。



发布流程：
测试环境流水线传入构建版本参数，触发构建和发布，然后进行测试验证。
测试环境验证ok后，说明代码分支没有问题，然后切换账号，到生产环境组织，开始触发流水线。流水线到构建出镜像为止。
生产环境仍使用手动升级部署进行迭代发布。

流水线：
1. 每个模块单独配置一个流水线，降低耦合性。 再加两个整体操作的，一个调用其他8个流水线，一个全套的
2. 测试环境流水线包含： 代码检测、构建和发布
   生产环境流水线仅包含：代码检测和构建。
   PS：暂时不添加其他质量门禁

### 前端好玩的

如何显示hover事件...

即不用鼠标悬停也能显示出提示语....

### Spring Cloud

Spring Cloud 应用pod 比Spring boot 应用Pod多了两个自动注入的环境变量`EUREKA_URL`and`CONFIG_URL`。

