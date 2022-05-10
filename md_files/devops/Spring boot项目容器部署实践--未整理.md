### Spring boot项目容器部署实践-- H3C

Spring Boot makes it easy to create stand-alone, production-grade Spring based Applications that you can "just run".

Spring Boot是个脚手架帮你快速生成Spring框架项目所需的配置。

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

