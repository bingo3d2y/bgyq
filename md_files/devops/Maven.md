### Maven

#### 小坑

docker maven latest镜像构建编译，gayhub sprint-boot-demo无法运行，其他环境maven构建的可用运行。

草

#### Java参数

java启动参数共分为三类：

其一是**标准参数**（**-**），所有的JVM实现都必须实现这些参数的功能，而且向后兼容；

其二是**非标准参数**（**-X**），默认jvm实现这些参数的功能，但是并不保证所有jvm实现都满足，且不保证向后兼容；

其三是**非Stable参数**（**-XX**），此类参数各个jvm实现会有所不同，将来可能会随时取消，需要慎重使用；

#### no main manifest attribute

java 没指定main class入口

```bash
$ java -jar target/HelloWorld-1.0-SNAPSHOT.jar
no main manifest attribute, in target/HelloWorld-1.0-SNAPSHOT.jar
##使用 -cp参数
$  java -cp target/HelloWorld-1.0-SNAPSHOT.jar  org.darebeat.App
start connect

## 或者修改pom.xml 指定main class
<build>  
    <plugins>  
        <plugin>  
            <!-- Build an executable JAR -->  
            <groupId>org.apache.maven.plugins</groupId>  
            <artifactId>maven-jar-plugin</artifactId>  
            <version>3.1.0</version>  
            <configuration>  
                <archive>  
                ## 指定class
                    <manifest>  
                        <mainClass>org.darebeat.App</mainClass>  
                    </manifest>  
                </archive>  
            </configuration>  
        </plugin>  
    </plugins>  
</build>
## main根据代码内容修改

$  java -jar target/HelloWorld-1.0-SNAPSHOT.jar 
start connect
ckage org.darebeat;

import java.sql.*;

public class App {
    private Connection conn = null;
    public static void main(String[] args)   {
       ...
    }
}
```

end



#### 获取一个应用的全部maven依赖:mag:

1. docker run --rm -it maven bash
2. git clone http_url_git
3. mvn package -Dmaven.test.skip=true 
4. cd /root/.m2/repository/ && tar zcvf /tmp/repo.tar repository/*

网上给的这个命令不太靠谱的样子：

`mvn dependency:copy-dependencies`



#### Maven仓库优先级配置

Repository Order

Remote repository URLs are queried in the following order for artifacts until one returns a valid result:

1. effective settings:
   1. Global `settings.xml`
   2. User `settings.xml`
2. local effective build POM:
   1. Local `pom.xml`
   2. Parent POMs, recursively
   3. Super POM
3. effective POMs from dependency path to the artifact.

**核心**： 用户自定义的pom.xml中的repository优先级小于settings.xml中定义的repository优先级。



配置 **profile**（私服）为 172.16.xxx.xxx 远程仓库, 

**repository** 为 dev.xxx.wiki 远程仓库,

**mirror** 为本地 localhost 仓库，还配置了一个 **mirrorOf 为 central** 远程仓库为 maven.aliyun.com 的中央仓库,

本地仓库的优先级是最高的
maven仓库的优先级别为

本地仓库 > 私服 （profile）> 远程仓库（repository）和 镜像 （mirror） > 中央仓库 （central）

镜像是一个特殊的配置，其实镜像等同与远程仓库，没有匹配远程仓库的镜像就毫无作用（如 foo2）。
3.总结上面所说的，Maven 仓库的优先级就是 **私服和远程仓库** 的对比，没有其它的仓库类型。为什么这么说是因为，镜像等同远程，而中央其实也是 maven super xml 配置的一个repository 的一个而且。所以 maven 仓库真正的优先级为

> 本地仓库 > 私服（profile）> 远程仓库（repository）

##### Local repository

默认local repository is `~/.m2/repository/`

当 dependency artifact存在本地时，就不需要在从remote repository download，节约构建时间。

典型应用：

jenkins-slave 使用pod模式运行在k8s集群上时，可以定义一个ManyReadWrite PVC，多个pod同时使用，都挂载到`/root/.m2/repository`目录，实现本地仓库的持久化。 

#### mirrorOf and profile: mark

mirrorOf可以理解“为某个仓库（repository）的做镜像”，填写的是repostoryId。”*“ 的意思就是匹配所有的仓库（repository）。相当于一个拦截器，它会拦截maven对remote repository的相关请求，把请求里的remote repository地址，重定向到mirror里配置的地址。

即`mirrorOf *`会导致配置的其他所以repositories失效，都被拦截转到mirror URL了。

在maven中配置一个mirror时，有多种形式，例如:

* `mirrorOf=“*” `//刚才经过，mirror一切，你配置的repository不起作用了
* `mirrorOf=my-repo-id` //镜像my-repo-id，你配置的my-repo-id仓库不起作用了
* `mirrorOf=*,!my-repo-id` //!表示非运算，排除你配置的my-repo-id仓库，其他仓库都被镜像了。就是请求下载my-repo-id的仓库的jar不使用mirror的url下载，其他都是用mirror配置的url下载
* `mirrorOf=external:` //如果本地库存在就用本地库的，如果本地没有所有下载就用mirror配置的url下载



```xml
<!-- mirrorOf * 表示所有artifact获取都经过mirror配置的url -->	
   <mirror>
      <id>nexus-aliyun</id>
      <mirrorOf>*</mirrorOf>
      <name>Nexus aliyun</name>
      <url>http://maven.aliyun.com/nexus/content/groups/public</url>
    </mirror> 

<!-- 自定义私有仓库 -->
 <profiles>
    <!-- profile
     | Specifies a set of introductions to the build process, to be activated using one or more of the
     | mechanisms described above. For inheritance purposes, and to activate profiles via <activatedProfiles/>
     | or the command line, profiles have to have an ID that is unique.
     |
    -->
	<profile>
        <id>pipeline_auto_config_profile</id>
        <repositories>
            <repository>
                <id>湖北中烟依赖库</id>
                <releases>
                    <enabled>true</enabled>
                    <updatePolicy>always</updatePolicy>
                </releases>
                <snapshots>
                    <enabled>true</enabled>
                    <updatePolicy>always</updatePolicy>
                </snapshots>
                <url>http://10.156.23.49:8081/repository/maven-public/</url>
            </repository>
        </repositories>
        <pluginRepositories>
            <pluginRepository>
                <id>湖北中烟依赖库</id>
                <releases>
                    <enabled>true</enabled>
                    <updatePolicy>always</updatePolicy>
                </releases>
                <snapshots>
                    <enabled>true</enabled>
                    <updatePolicy>always</updatePolicy>
                </snapshots>
                <url>http://10.156.23.49:8081/repository/maven-public/</url>
            </pluginRepository>
        </pluginRepositories>
        <activation>
            <activeByDefault>true</activeByDefault>
        </activation>
    </profile>

  </profiles>
```

end

#### mvn and setting.xml 配置

```bash
 -P,--activate-profiles <arg>           Comma-delimited list of profiles to activate
 
  <profiles>
        <profile>
            <id>pipeline_auto_config_profile</id>
            <repositories>
                <repository>
                    <id>maven-public</id>
                    <releases>
                        <enabled>true</enabled>
                        <updatePolicy>always</updatePolicy>
                    </releases>
                    <snapshots>
                        <enabled>true</enabled>
                        <updatePolicy>always</updatePolicy>
                    </snapshots>
                    <url>http://XXX/repository/maven-public/</url>
                </repository>
            </repositories>
        </profile>
                <profile>
            <id>pipeline_auto_config_profile</id>
            <repositories>
                <repository>
                    <id>maven-public</id>
                    <releases>
                        <enabled>true</enabled>
                        <updatePolicy>always</updatePolicy>
                    </releases>
                    <snapshots>
                        <enabled>true</enabled>
                        <updatePolicy>always</updatePolicy>
                    </snapshots>
                    <url>http://xxx/repository/maven-test/</url>
                </repository>
            </repositories>
        </profile>
    </profiles>
 # 可以指定某个仓库来获取编译依赖 mvn -P<prifile_name> clean package 
 $ mvn -Ppipeline_auto_config_profil clean package 
```

end

#### mvn and  pom.xml配置

mvn执行构建命令时，默认读取的是当前目录下的`pom.xml`文件，可以通过`-f`参数指定其他的pom文件。

eg： 项目下有两个pom.xml，作用不一

* pom.xml : 正常构建jar包
* pom-build.xml: 构建docker image



##### 修改pom文件定义jar版本

修改pom文件，定义artifac 版本

目标将：

 http://10.156.23.49:8081/repository/maven-releases/com/oracle/ojdbc6/11.2.0.4/ojdbc6-11.2.0.4.jar
 http://10.156.23.49:8081/repository/maven-public/com/oracle/ojdbc6/11.2.0.4/ojdbc6-11.2.0.4.pom

修改成：

 http://10.156.23.49:8081/repository/maven-public/com/oracle/ojdbc6/11.2.04/ojdbc6-11.2.04.pom

操作：

jar包不变，修改pom.xml 中的version

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" 
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
	
	<modelVersion>4.0.0</modelVersion>
	<groupId>com.oracle</groupId>
	<artifactId>ojdbc6</artifactId>
   <!-- 将11.2.0.4 改成11.2.04 -->
	<version>11.2.04</version>

	<name>Oracle JDBC Driver</name>
	....
	
</project>
```

end

##### \<scope>



##### modules

maven聚合项目查看那个是root module，可以在项目中搜索`modules`

```xml
  <!-- pom 文件 module决定了谁是root -->
    <modules>
        <module>app</module>
        <module>data-structure-conversion</module>
    </modules>

```

end

##### 跳过plugin

跳过pom中定义的某个插件执行`<skip>true</skip>`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>nti-datatrans-service</artifactId>
        <groupId>com.nti56</groupId>
        <version>1.0-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>
    <packaging>jar</packaging>

    <artifactId>datatrans-app</artifactId>
    <properties>
        ....
        <server.port>9000</server.port>

    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        ...
    </dependencies>

    <build>
        <plugins>
            <!--跳过使用build image插件-->
            <plugin>
                <groupId>com.google.cloud.tools</groupId>
                <artifactId>jib-maven-plugin</artifactId>
                <version>3.0.0</version>
                <configuration>
                    ...
                </configuration>

                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>build</goal>
                        </goals>
                    </execution>
                </executions>
                <skip>true</skip>
            </plugin>
            ...
        </plugins>
    </build>

</project>

```

end

##### 将所有依赖打入jar:cry:

不将项目依赖的jar打入制品jar，会导致制品jar换个机器或者环境就无法运行了。但是这样做的代价是jar包很大。

```xml

<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-shade-plugin</artifactId>
      <version>3.0.0</version>
      <executions>
        <execution>
          <phase>package</phase>
          <goals>
            <goal>shade</goal>
          </goals>
        </execution>
      </executions>
    </plugin>
    <plugin>
    <!-- Build an executable JAR -->
       <groupId>org.apache.maven.plugins</groupId>
       <artifactId>maven-jar-plugin</artifactId>
       <version>3.1.0</version>
        <configuration>
           <archive>
              <manifest>
                  <mainClass>org.darebeat.App</mainClass>
              </manifest>
            </archive>
        </configuration>
   </plugin>
  </plugins>
</build>

```

springboot肯定包含了这些，就这一个build-plugin，至少包含了两步

* 将jar所有依赖打到制品jar中，在其他机器上可以直接`java -jar`运行
* 指定了入口manifest class，使得可以直接`java -jar`运行

```xml
<build>
  <plugins>
  <!--spring boot maven插件-->
     <plugin>                         <groupId>org.springframework.boot</groupId>
 <artifactId>spring-boot-maven-plugin</artifactId>
      </plugin>
  </plugins>
</build>

```

end

#### mvn 命令

mvn clean:清理项目建的临时文件,一般是模块下的target目录

mvn clean package依次执行了clean、resources、compile、testResources、testCompile、test、jar(打包)等７个阶段。
mvn clean install依次执行了clean、resources、compile、testResources、testCompile、test、jar(打包)、install等8个阶段。
mvn clean deploy依次执行了clean、resources、compile、testResources、testCompile、test、jar(打包)、install、deploy等９个阶段。
由上面的分析可知主要区别如下，

**package**命令完成了项目编译、单元测试、打包功能，**但没有**把打好的可执行jar包（war包或其它形式的包）布署到本地maven仓库和远程maven私服仓库，仅仅放在<outputDirectory>目录中，默认在`./target/`
**install**命令完成了项目编译、单元测试、打包功能，同时把打好的可执行jar包（war包或其它形式的包）布署到本地maven仓库，但没有布署到远程maven私服仓库
**deploy**命令完成了项目编译、单元测试、打包功能，同时把打好的可执行jar包（war包或其它形式的包）布署到本地maven仓库和远程maven私服仓库

##### maven 构建声明周期

构建生命周期中的每一个由构建阶段的不同列表定义，其中构建阶段表示生命周期中的阶段。

例如，**默认（default）**的生命周期包括以下阶段（注意：这里是简化的阶段，用于生命周期阶段的完整列表，请参阅下方**生命周期参考**）：

- **`验证（validate）`** - 验证项目是否正确，所有必要的信息可用
- **`编译（compile）`** - 编译项目的源代码
- **`测试（test）`** - 使用合适的单元测试框架测试编译的源代码。这些测试不应该要求代码被打包或部署
- **`打包（package）`** - 采用编译的代码，并以其可分配格式（如JAR）进行打包。
- **`验证（verify）`** - 对集成测试的结果执行任何检查，以确保满足质量标准
- **`安装（install）`** - 将软件包安装到本地存储库中，用作本地其他项目的依赖项
- **`部署（deploy）`** - 在构建环境中完成，将最终的包复制到远程存储库以与其他开发人员和项目共享。

##### skip test

`mvn -Dmaven.test.skip=true clean package`

```bash
 -D,--define <arg>                      Define a system property
 mvn -Dmaven.test.skip=true clean package
```

避免，docker image 改造时，由于容器环境和vm环境不同导致单元测试执行失败

```bash
-------------------------------------------------------
 T E S T S
-------------------------------------------------------
Running com.nti.test.ifs.sap.Enum2DataTest
09:09:07.790 [background-preinit] INFO  o.h.v.i.util.Version - [<clinit>,21] - HV000001: Hibernate Validator 6.0.22.Final
Application Version: ${nti.version}
Spring Boot Version: 2.2.13.RELEASE
////////////////////////////////////////////////////////////////////
//                          _ooOoo_                               //
//                         o8888888o                              //
//                         88" . "88                              //
//                         (| ^_^ |)                              //
//                         O\  =  /O                              //
//                      ____/`---'\____                           //
//                    .'  \\|     |//  `.                         //
//                   /  \\|||  :  |||//  \                        //
//                  /  _||||| -:- |||||-  \                       //
//                  |   | \\\  -  /// |   |                       //
//                  | \_|  ''\---/''  |   |                       //
//                  \  .-\__  `-`  ___/-. /                       //
//                ___`. .'  /--.--\  `. . ___                     //
//              ."" '<  `.___\_<|>_/___.'  >'"".                  //
//            | | :  `- \`.;`\ _ /`;.`/ - ` : | |                 //
//            \  \ `-.   \_ __\ /__ _/   .-` /  /                 //
//      ========`-.____`-.___\_____/___.-`____.-'========         //
//                           `=---='                              //
//      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^        //
//             ????       ????      ??BUG               //
////////////////////////////////////////////////////////////////////
09:09:08.358 [main] INFO  c.a.n.c.c.i.LocalConfigInfoProcessor - [<clinit>,212] - LOCAL_SNAPSHOT_PATH:/root/nacos/config
09:09:08.401 [main] INFO  c.a.n.c.c.i.Limiter - [<clinit>,54] - limitTime:5.0
09:09:09.424 [main] ERROR c.a.n.c.c.h.ServerHttpAgent - [httpGet,112] - [NACOS SocketTimeoutException httpGet] currentServerAddr:http://139.159.194.101:28848? err : connect timed out
09:09:10.427 [main] ERROR c.a.n.c.c.h.ServerHttpAgent - [httpGet,112] - [NACOS SocketTimeoutException httpGet] currentServerAddr:http://139.159.194.101:28848? err : connect timed out
## 连接 Nacos失败
09:09:11.430 [main] ERROR c.a.n.c.c.h.ServerHttpAgent - [httpGet,112] - [NACOS SocketTimeoutException httpGet] currentServerAddr:http://139.159.194.101:28848? err : connect timed out
09:09:11.431 [main] ERROR c.a.n.c.c.h.ServerHttpAgent - [httpGet,133] - no available server
```

end

#### Maven force update-snapshots ：mark

snapshot版本

```bash
# -U --update-snapshots    Forces a check for missing  releases and updated snapshots on remote repositories

$ mvn clean install -e -U
```

end

#### 私有maven仓库 账号密码加密

https://blog.csdn.net/u013648164/article/details/81005876

其实，实践中使用环境变量代替了

```xml
    <servers>
        <server>
            <id>releases</id>
            <username>${env.NEXUS_USERNAME}</username>
            <password>${env.NEXUS_PASSWORD}</password>
        </server>
        <server>
            <id>snapshots</id>
            <username>${env.NEXUS_USERNAME}</username>
            <password>${env.NEXUS_PASSWORD}</password>
        </server>
    </servers>

```

end

#### Maven顶级元素

```xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
                          https://maven.apache.org/xsd/settings-1.0.0.xsd">
  <localRepository/>
  <interactiveMode/>
  <usePluginRegistry/>
  <offline/>
  <pluginGroups/>
  <servers/>
  <mirrors/>
  <proxies/>
  <profiles/>
  <activeProfiles/>
</settings>
```

end

#### Maven 聚合打包目录

https://www.cnblogs.com/pu20065226/p/11044386.html

理解聚合项目：

西瓜:
你用go或者py写程序的时候不是也要用到一些库引用，你需要import 导入。
java也是一样的，这个common，还有poxy另外三个工程就是他写的java代码打包成一个jar作为库给server使用的。



maven所有编译、打包生成的文件都放在`target`目录里。这些就是一个Maven项目的标准目录结构。

如果在`pom.xml`中，通过`<outputDirectory>`标签指定打包路径，则编译jar包会在指定目录，否则会在默认的`target/`目录下。

但是Maven聚合项目要注意，如下目录结构：

```bash
$ tree /tmp/diap-parent/
/tmp/diap-parent/
├── daip-common
│   ├── pom.xml
│   └── src
├── daip-proxy
│   ├── pom.xml
│   └── src
├── daip-server
│   ├── pom.xml
│   └── src
└── pom.xml

```

从项目的目录结构来看daip-parent是一个聚合工程，每个子模块都会在自己的target目录下生成jar包。

所以daip-parent模块编译构建的jar目录不在默认路径daip-parent/target/下面，而在主模块daip-server的targt目录下。

如何，确定哪个是主模块，IDE可以方便的看出来，自己查看pom文件也行。

如下，很明显：daip-server --> daip-proxy --> daip-common

daip-server是食物链的顶端。

```bash
$  cat ./daip-proxy/pom.xml
...
    <dependencies>

        <!--平台公共模块-->
        <dependency>
            <groupId>com.wiseda.daip</groupId>
            <artifactId>daip-common</artifactId>
        </dependency>
    </dependencies>
...

$ cat ./daip-server/pom.xml
...
    <dependencies>

        <!--平台代理模块-->
        <dependency>
            <groupId>com.wiseda.daip</groupId>
            <artifactId>daip-proxy</artifactId>
        </dependency>
     </dependencies>
 ...
```



 聚合工程优势：

1.统一maven操作。可以在一个maven工程管理多个子工程（每个子工程可单独打包，重启，调试。也可通过聚合工程一起管理）。

2.统一管理依赖版本。可以借助父工程（dependencyManagement）来管理依赖包的版本，子工程就直接引用包而不用添加版本信息。

3.统一引入公共依赖，而不需要每个子项目都去重复引入。

4.防止pom.xml过长。



#### Maven 3.8.+ default https

maven3.8.1 为了解决CVE-2021-26291 这个漏洞，将http 的仓库禁用了。。。而且默认配置了一个http://0.0.0.0 的mirrors，如果自己项目的pom.xml里面配置的是http的仓库，那就会造成拉不到包。。。
 解决方法就是把nexus仓库配置成https，然后把pom.xml里面的http地址改成https的就行了。

上面再扯犊子，下面解决方法很便捷

修改全局的settings.xml文件（一般在系统路径下，比如mac就在`/usr/local/Cellar/maven/3.8.2/libexec/conf/settings.xml`）,删除如下部分即可：

```xml
<!-- windows D:\Program Files\apache-maven-3.8.3\conf ->

<mirror>
    <id>maven-default-http-blocker</id>
    <mirrorOf>external:http:*</mirrorOf>
    <name>Pseudo repository to mirror external repositories initially using HTTP.</name>
    <url>http://0.0.0.0/</url>
    <blocked>true</blocked>
</mirror>
```

然后就可以继续正常使用Maven了。

```bash
mvn help:effective-settings

## 默认走0.0.0.0
$ mvn clean install
[INFO] Scanning for projects...
Downloading from maven-default-http-blocker: http://0.0.0.0/com/wiseda/daip/daip-parent/1.0.0-SNAPSHOT/maven-metadata.xml
[WARNING] Could not transfer metadata com.wiseda.daip:daip-parent:1.0.0-SNAPSHOT/maven-metadata.xml from/to maven-default-http-blocker (http://0.0.0.0/): transfer failed for http://0.0.0.0/com/wiseda/daip/daip-parent/1.0.0-SNAPSHOT/
maven-metadata.xml
[ERROR] [ERROR] Some problems were encountered while processing the POMs:
[FATAL] Non-resolvable parent POM for com.wiseda.daip:daip-job:1.0.0-SNAPSHOT: Could not transfer artifact com.wiseda.daip:daip-parent:pom:1.0.0-SNAPSHOT from/to maven-default-http-blocker (http://0.0.0.0/): Blocked mirror for repos
itories: [maven-public (http://10.156.23.49:8081/repository/maven-public/, default, releases+snapshots)] and 'parent.relativePath' points at wrong local POM @ line 14, column 13
 @

```

end

#### CloudOS and Maven

配置maven私有仓库，cloudOS的jenkins-slave直接根据cloudos上的配置生成`settings.xml`

很明显了，页面上配置几个Maven仓库就对应解析成几个`<repository>`标签就行了。

```bash
bash-4.2# cat /root/.m2/settings.xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<settings xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://maven.apache.org/SETTINGS/1.0.0">
    <servers>
        <server>
            <id>maven-public</id>
            <password>hbzy@123</password>
            <username>admin</username>
        </server>
    </servers>
    <mirrors/>
    <profiles>
        <profile>
            <id>pipeline_auto_config_profile</id>
            <repositories>
                <repository>
                    <id>maven-public</id>
                    <releases>
                        <enabled>true</enabled>
                        <updatePolicy>always</updatePolicy>
                    </releases>
                    <snapshots>
                        <enabled>true</enabled>
                        <updatePolicy>always</updatePolicy>
                    </snapshots>
                    <url>http://10.156.23.49:8081/repository/maven-public/</url>
                </repository>
            </repositories>
            <pluginRepositories>
                <pluginRepository>
                    <id>maven-public</id>
                    <releases>
                        <enabled>true</enabled>
                        <updatePolicy>always</updatePolicy>
                    </releases>
                    <snapshots>
                        <enabled>true</enabled>
                        <updatePolicy>always</updatePolicy>
                    </snapshots>
                    <url>http://10.156.23.49:8081/repository/maven-public/</url>
                </pluginRepository>
            </pluginRepositories>
            <activation>
                <activeByDefault>true</activeByDefault>
            </activation>
        </profile>
    </profiles>
</settings>
```

end

### IntelliJ Idea

##### 配置maven

file  --> settings --> build,exection,Deployment --> Maven

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/intenlliJ-maven-1.jpg)

添加maven仓库配置

repositories 名为仓库，解决项目依赖的第三方jar包存放位置，以及多个项目公用第三方jar包。    

pluginRepositories 名为插件仓库，存放maven插件的仓库，告诉项目您使用的插件应该去什么地方下载。

```xml
  <servers>
	<server>
      <id>湖北中烟依赖库</id>
      <password>admin/hbzy@123</password>
       <username>admin</username>
    </server>
 </servers>
...
	<profile>
        <id>pipeline_auto_config_profile</id>
        <repositories>
            <repository>
                <id>湖北中烟依赖库</id>
                <releases>
                    <enabled>true</enabled>
                    <updatePolicy>always</updatePolicy>
                </releases>
                <snapshots>
                    <enabled>true</enabled>
                    <updatePolicy>always</updatePolicy>
                </snapshots>
                <url>http://10.156.23.49:8081/repository/maven-public/</url>
            </repository>
        </repositories>
        <pluginRepositories>
            <pluginRepository>
                <id>湖北中烟依赖库</id>
                <releases>
                    <enabled>true</enabled>
                    <updatePolicy>always</updatePolicy>
                </releases>
                <snapshots>
                    <enabled>true</enabled>
                    <updatePolicy>always</updatePolicy>
                </snapshots>
                <url>http://10.156.23.49:8081/repository/maven-public/</url>
            </pluginRepository>
        </pluginRepositories>
        <activation>
            <activeByDefault>true</activeByDefault>
        </activation>
    </profile>

  </profiles>
...
```

end

##### 利用maven图形化界面调试

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/intenlliJ-maven-2.png)

### 实际排错

#### Not found class FQN

如下，错误，必然是缺少TransmittableThreadLocal class 即缺少对应的jar

java.lang.NoClassDefFoundError: com/alibaba/ttl/TransmittableThreadLocal

解决:

```xml
<dependency>
  <groupId>com.alibaba</groupId>
  <artifactId>transmittable-thread-local</artifactId>
  <version>2.12.0</version>
</dependency>

```

end

#### mvn  miss `file` parameters

The parameters 'file' for goal org.apache.maven.plugins:maven-deploy-plugin:2.7:deploy-file are missing or invalid -> [Help 1]

```bash
$ mvn deploy:deploy-file -Dpackaging=pom  -DpomFile=nti-commons-1.0.1-SNAPSHOT.pom   -DFile=nti-commons-1.0.1-SNAPSHOT.pom -DrepositoryId=maven-snapshots -Durl=http://10.156.23.49:8081/repository/maven-snapshots/
...
[ERROR] Failed to execute goal org.apache.maven.plugins:maven-deploy-plugin:2.7:deploy-file (default-cli) on project standalone-pom: The parameters 'file' for goal org.apache.maven.plugins:maven-deploy-plugin:2.7:deploy-file are missing or invalid -> [Help 1]

## 解决
## 它提示了缺 file这个参数...我就换成 小写file了--
$ mvn deploy:deploy-file -Dpackaging=pom  -DpomFile=nti-commons-1.0.1-SNAPSHOT.pom   -Dfile=nti-commons-1.0.1-SNAPSHOT.pom -DrepositoryId=maven-snapshots -Durl=http://10.156.23.49:8081/repository/maven-snapshots/


```

end



### 引用

1. https://blog.csdn.net/zhaojianting/article/details/80324533
2. https://maven.apache.org/guides/mini/guide-multiple-repositories.html 
2. https://www.jianshu.com/p/c4f02c5bdfc7