## Nexus 

### 流程控制

在开发，阶段就应该指定ISV从哪几个公网地址下拉取依赖，不然太被动了-- 它们各种依赖--

或者甲方可以很便捷的给公共库配置proxy repository.

### Nexus 配置仓库

#### Nexus默认仓库

默认仓库一下：

1. maven-central：maven中央库，默认从https://repo1.maven.org/maven2/拉取jar
2. maven-releases：私库发行版jar，初次安装请将`Deployment policy`设置为`Allow redeploy`
3. maven-snapshots：私库快照（调试版本）jar
4. maven-public：仓库分组，把上面三个仓库组合在一起对外提供服务，在本地maven基础配置`settings.xml`中使用。

##### allow redeploy

由于一些内部的Bug,需要升级maven,但是影响面太广,希望在不改变的版本号的情况下发布，只需要在打包服务器上删除m2的缓存就能解决问题。

但是在重复deploy的时候出现如下错误：

> ```
> Failed to transfer file: http://xxxxx:8081/repository/maven-releases/xxx/xxx/xxx/xxx.jar. Return code is: 400, ReasonPhrase: Repository does not allow updating assets: maven-releases. -> [Help 1]
> ```

**因为仓库默认不允许重复上传同一个版本。**

解决办法：

> 登录nexus管理界面->设置–>Repository–>Repositories–>maven-releases–>Hosted–>请选择‘Allow redeploy’，然后保存。在重新上传即可成功

这个setting真的隐蔽...

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/nexus-redeploy.png)

maven-releases和maven-snapshots仓库的区别：

Hosted

Deployment policy:

Controls if deployments of and updates to artifacts are allowed

* maven-releases： disable redeploy
* maven-snapshots: allow redeploy



#### Nexus仓库类型

1. group(仓库组类型)：又叫组仓库，用于方便开发人员自己设定的仓库；
2. hosted(宿主类型)：内部项目的发布仓库（内部开发人员，发布上去存放的仓库）；
3. proxy(代理类型)：从远程中央仓库中寻找数据的仓库（可以点击对应的仓库的Configuration页签下Remote Storage属性的值即被代理的远程仓库的路径）；
4. virtual(虚拟类型)：虚拟仓库（这个基本用不到，重点关注上面三个仓库的使用）；

##### Raw Repositories

在Nexus 3中支持一种新的方式：raw repositories。利用这种方式，任何文件都可以像Maven管理对象文件那样被管理起来，对所有的artifacts进行统一集成管理。

#### 部署 Nexus：配置优先级

执行`tar -zxf nexus-unix.tar.gz`命令，会解压出两个目录:

* `nexus-<version>`程序目录，包含了 Nexus 运行所需要的文件。是 Nexus 运行必须的

* `sonatype-work` - 仓库目录。包含了 Nexus 生成的配置文件、日志文件、仓库文件等。

  当我们需要备份 Nexus 的时候默认备份此目录即可。

配置文件优先级如下：

* /opt/sonatype/sonatype-work/nexus3/etc/nexus.properties
* /nexus-data/etc/nexus.properties 
*  /opt/sonatype/nexus/etc/nexus-default.properties：优先级最低

当多个配置文件共存时，注意配置文件生效优先级--

##### 容器部署

容器部署时，直接挂载一个卷即可，而且默认账密为`admin/admin123`



#### Nexus仓库实践

一个 group type（组类型，能够组合多个仓库为一个地址提供服务）的maven应该包含以下几个仓库：

* maven-releases： hosted type，本地存储。像官方仓库一样提供本地私库功能
* maven-snapshots： hosted type，用于支持snapshot jar/pom
* maven-central：proxy type,提供代理其它仓库的类型

##### proxy and hosted and group

npm(proxy): 可配置代理的仓库，当此仓库没有相应包时 会从配置的第三方仓库拉取 并缓存到本地proxy仓库

npm(hosted)：开发自己的包推送到此仓库，需登录才能推送

npm(group)： 可配置包含上面两种仓库，这样用户只需要配置npm(group) 这个地址即可 ，避免配置npm(proxy) 和npm(hosted) 两个地址



##### Nexus： 配置代理仓库

##### maven 命令行区分大小写：windows下也区分



##### Nexus：命令行上传artifact到私有仓库, 区分大小写mark

nexus3 web端不提供直接上传快照的页面

一个打包版本中引用了SNAPSHOT快照jar。

snapshot jar 需要使用mvn命令行手动上传

修改maven settings.xml

```xml
<servers>
   <server>
       <!-- maven -DrepositoryId -->
      <id>maven-snapshots</id>
      <username>admin</username>
      <password>hbzy@123</password>
    </server>
</servers>
```

命令行格式

If you're using `-DpomFile=` you don't need to use groupID and artifactID; those will be inferred from the pom. 

究极体：

```bash
mvn deploy:deploy-file  -DgroupId=com.nti56  -Dversion=<Version>  -Dfile=<File_path> -DpomFile=<Pomfile_path> -Durl=http://10.156.23.49:8081/repository/maven-snapshots/ -DrepositoryId=maven-snapshots
```

end

```bash
## only pom
mvn deploy:deploy-file -DgroupId=<GroupID> -DartifactId=<ArtifactId> -Dversion=<Pom_Version> -Dpackaging=pom -Dfile=<Pom_file_path> -Durl=http://<Nexus_IP>/repository/maven-snapshots/ -DrepositoryId=<Repo_ID> -X
## pom and jar
mvn deploy:deploy-file -DgroupId=<GroupID>  -DartifactId=<ArtifactId>  -Dversion=<Jar_Version> -Dpackaging=jar -Dfile=<Jar_file_path> -DpomFile=<Pom_file_path> -Durl=http://<Nexus_IP>/repository/maven-snapshots/ -DrepositoryId=<Repo_ID> -X
```



405 put 问题： As indicated in the comments, you can only deploy to Maven Hosted Repositories, not Proxies. This is by design.

```bash
## 仅pom
mvn deploy:deploy-file -DgroupId=com.wiseda.daip -DartifactId=daip-parent -Dversion=1.0.0-SNAPSHOT -Dpackaging=pom -Dfile=daip-parent-1.0.0-SNAPSHOT.pom -Durl=http://10.156.23.49:8081/repository/maven-snapshots/ -DrepositoryId=<Repo_ID> -X

## 上传 snapshot jar
mvn deploy:deploy-file -DgroupId=com.wiseda.daip -DartifactId=daip-proxy -Dversion=1.0.0-SNAPSHOT -Dpackaging=jar -Dfile=daip-common-1.0.0-SNAPSHOT.jar -DpomFile=daip-common-1.0.0-SNAPSHOT.pom -Durl=http://10.156.23.49:8081/repository/maven-snapshots/ -DrepositoryId=<Reop_ID> -X

```

end

##### Nexus 界面上传artifact

upload 使用匹配文件结尾：

test.1.2.4.release.pom

要改成下面命名，不然识别不到--

test.1.2.4.pom

##### 400 error code

400 Bad Request will be returned if you attempt to:

Deploy a snapshot artifact (or version) ending in -SNAPSHOT to a release repository
Deploy a release artifact (version not ending in -SNAPSHOT) to a snapshot repository
Deploy the same version of a release artifact more than once to a release repository（redeploy可以解决）

### Snapshot and Release jar

maven包的版本有两类，一类是 SNAPSHOT，一类是 RELEASE。

这两类有个重要的区别，RELEASE 的包需要改 pom.xml 中的 <version> 的时候才会引入其他版本（如新版本），但是 SNAPSHOT 允许不改 <version> 而引入新版本（自动通过时间戳判断）

那 SNAPSHOT 是怎么做到的呢? SNAPSHOT 就是为了应对 “被依赖者频繁改版本号导致依赖者需要频繁修改pom.xml的版本” 的问题。例如A依赖B，B在开发过程中，所以B肯定是经常变动的，每次变动时都得通知A修改版本，会疯掉的。如果B是SNAPSHOT版本，B更改后，A并不需要修改pom.xml中的版本就可以不断拉取到B的更新。

- snapshot 版本代表不稳定、尚处于开发中的版本，即快照版本
- release 版本代表功能趋于稳定、当前更新停止，可以用于发行的版

#### 使用场景

- 依赖库中的 jar 包若处于不断更新，更准确的说是不停 deploy 时，deploy会发布到私服，则使用snapshot

  - 格式：<version>1.0-snapshot</version>

  - 特点

    - 不停更新/deploy 时，**版本号1.0不需更改**，私服中会**自动追加后缀时间**为版本名

    - 其他系统使用时，会自动load**时间最近也即最新的版本**

      或者使用`mvn -U`强制更新snapshot jar

- 当第三方 jar 包功能确定时，可以提供一个release版本

  - 格式：<version>1.0</version>，**去掉-snapshot即可**
  - 特点
    - 其他系统使用时，版本号不变，依赖包则不变，**不会自动load最新版本**
    - 上述有两个意思
      - 假设第三方对 1.0 version 更新了，但本地有旧的 1.0 version，其他系统**不会更新引入私服中最新的1.0**，与snapshot的区别
      - 第三方 升级了2.0，其他系统必须**手动更新**依赖的version为 2.0，否则不能引入最新版本，**这也是相对snapshot比较麻烦的地方**



### jar and war: mark

jar：即Java Archive，Java的包，Java编译好之后生成class文件，但如果直接发布这些class文件的话会很不方便，所以就把许多的class文件打包成一个jar，jar中除了class文件还可以包括一些资源和配置文件，通常一个jar包就是一个java程序或者一个java库。

解压：`jar -xf jar_file_name`

war：Web application Archive，与jar基本相同，但它通常表示这是一个Java的Web应用程序的包，依赖tomcat这种Servlet容器才能运行war包

使用：`mvn clean package`进行打包

#### war 内置Servlet容器

以前，war包的运行依赖外部的Servlet容器，即要将war包部署在Tomcat Server在配置好server.xml才能正常运行war包应用。

现在，在轻量级和微服务的影响下，web框架（eg：springboot）生成的war包通常已经包含了内置的tomcat servlet容器，可以直接使用`java -jar *.war`运行war包。

当war内置了tomcat servlet而且部署到外部的Tomcat Server环境下，可能导致war无法正常启动运行。

如下例子：

 项目在intellij idea里配置tomcat可以启动访问, 打成war包丢到tomcat webapps下能启动却访问不了相关的接口
  这个问题是因为idea会默认将项目以ROOT为目录的文件
  而丢到tomcat的webapps下面则是解压成你项目名称为目录的文件，和ROOT是同级的
  为保障springboot war可以在Tomcat webapps目录运行，可以有以下几种解决方案：
  一：将你的war名称改成作为ROOT.war
  二：在tomcat的server.xml文件的Host标签内配置\<Context path="/" docBase="project_path" reloadable="true"/>
  三：用tomcat发布时，将前端请求的路径加上你的项目名称

#### java import and package

import 可以引用 代码源文件也可以是 jar

这个job 编译的时候找不到common包了

pom.xml 定义了deps 各种依赖--

package 找不到，package是什么--

#### 启动jar包：Unable to access jarfile

在docker中指定，jar运行:

* `java -jar /opt/springboot/*.jar` : 报错，提示Unable to access jarfile
* `java -jar *.jar` : 报错，提示Unable to access jarfile
* `java -jar /opt/springboot/hello.jar` : 正确

离谱，不能使用通配符，想想也对，当目录下有N个符合条件的jar时，你到底运行哪一个呢？

 An archive is a modular jar if a module descriptor, 'module-info.class', is located in the root of the given directories, or the root of the jar archive itself. 



### 引用

1. https://www.jianshu.com/p/084fd2408d9a
1. https://www.cxybb.com/article/huofuman960209/105489978
