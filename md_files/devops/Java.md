## Java

### java code 运行过程

接下来我们看下一个正确的示例：

```bash
java -Xmx100m com.acme.example.ListUsers fred joe bert
```

在用以上命令行调用java程序时，实际上java虚拟机会完成以下工作：

1. 在编译后的class文件中搜索名为com.acme.example.ListUsers的class
2. 加载这个class
3. 校验这个class的main方法签名以及返回值类型等
4. 调用这个方法并传进(“fred”, “joe”, “bert”) 数组作为参数

### java class

**JVM上运行的本身也不是java文件，而是class文件（字节码）。**

而能够编译转化为class文件的，并不只有java一种。

class文件的本质都是一组以 8 位字节为基础单位的2进制流。记住，是2进制。

#### class FQN

即 FQN必须包含`.` or `/` ？？？

FQN fully qulified name，一个完整的java class（FQN fully qulified name）的名称应该如下所示，由于java版本的关系，Java命令行也允许使用`/`作为包名的分隔符。

```bash
# packagename.packagename2.packagename3.ClassName
com.nti56.datatrans.DataTransApplication

# packagename/packagename2/packagename3/ClassName
com/nti56/datatrans/DataTransApplication
```

这个包路径看着很像是一个文件的名字，**但是它不是文件名！！！** 如果你是使用class文件的文件名调用的，就会报错。

#### class-path：mark

-cp 和 -classpath 一样，是指定类运行所依赖其他类的路径，通常是类库，jar包之类，需要全路径到jar包，window上分号`;` 分隔，linux上是分号`:`分隔。不支持通配符，需要列出所有jar包，用一点“.”代表当前路径

class-path的作用是为了告诉java应用，去哪个路径下找应用程序需要的class，所以，如果class-path设置不对，也会导致java虚拟机找不到需要加载的FQN，进而导致could not find or load main class。

class-path需要你整个应用所有的依赖的class，也就是为了主类加载正确，JVM需要找到：

- 主类本身；
- 所有父亲类以及接口；
- 所有声明变量的类以及调用的方法等。

下面给出一个例子

- 你想要运行的FQN为com.acme.example.Foon
- class文件的完整路径为/usr/local/acme/classes/com/acme/example/Foon.class
- 你当前的工作路径为/usr/local/acme/classes/com/acme/example/

```bash
# wrong, FQN is needed，66
java Foon

# wrong, there is no `com/acme/example` folder in the current working directory
java com.acme.example.Foon

# wrong, similar to above
java -classpath . com.acme.example.Foon

# fine; relative classpath set
java -classpath ../../.. com.acme.example.Foon

# fine; absolute classpath set
java -classpath /usr/local/acme/classes com.acme.example.Foon

```



### 运行jar app

执行`java`命令传入参数，**传入的是main函数所在的类的名字，而不是class文件；java会根据类名自动去找class文件**。

#### `java -cp`

`-cp` <class search path of directories and zip/jar files>

```bash
$ java -cp  /opt/nti-server/resources:/opt/nti-server/classes:/opt/nti-server/libs/* com.nti56.datatrans.DataTransApplication
```

执行该命令时，会用到目录META-INF\MANIFEST.MF文件，在该文件中，有一个叫Main－Class的参数，它说明了java -jar命令执行的类。java -jar方式不可以指定附加依赖jar包。



#### `java -jar`

一个可执行的 JAR 必须通过 menifest 文件的头引用它所需要的所有其他从属 JAR。**如果使用了 -jar选项，那么环境变量 CLASSPATH 和在命令行中指定的所有类路径都被 JVM 所忽略。**

即一个可执行的jar必须包含运行该jar的全部依赖--

java -jar 运行一个jar的时候并没有指定运行的mian类，但是也是可以运行的，这个是因为在打包的时候，打包生成jar里面有文件指定了main类，所以，java -jar是可以直接的运行的。

**要想jar包能直接通过`java -jar xxx.jar`运行，需要满足：**

1、在jar包中的META-INF/MANIFEST.MF中指定Main-Class，这样才能确定程序的入口在哪里；

2、要能加载到依赖包。

这是一个可以单独运行的jar

```bash
$ ls extract_jar/
BOOT-INF  META-INF  org
$ ls -l extract_jar/BOOT-INF/
total 0
drwxr-xr-x. 6 root root  200 Mar 16 04:07 classes
drwxr-xr-x. 2 root root 6500 Mar 16 04:02 lib
# 一排控制文件
$ ls BOOT-INF/classes/
application-dev.yml    application-local.yml  application-prod.yml   application.yml        com/                   db/                    mapper/                org/

$ ls -l extract_jar/META-INF/
total 8
-rw-r--r--. 1 root root 350 Mar 16 04:02 MANIFEST.MF
drwxr-xr-x. 3 root root  60 Mar 16 04:02 maven
-rw-r--r--. 1 root root 922 Mar 16 04:02 spring-configuration-metadata.json
# main class
$ cat extract_jar/META-INF/MANIFEST.MF
Manifest-Version: 1.0
Archiver-Version: Plexus Archiver
Built-By: root
Start-Class: com.nti56.datatrans.DataTransApplication
Spring-Boot-Classes: BOOT-INF/classes/
Spring-Boot-Lib: BOOT-INF/lib/
Spring-Boot-Version: 2.2.6.RELEASE
Created-By: Apache Maven 3.3.9
Build-Jdk: 1.8.0_222
Main-Class: org.springframework.boot.loader.JarLauncher
```

end

##### 支持通配符

```bash
# 正常执行
$ java -cp  /opt/nti-server/resources:/opt/nti-server/classes:/opt/nti-server/libs/* com.nti56.datatrans.DataTransApplication

# 无法运行
$ java -cp  /opt/nti-server/resources:/opt/nti-server/classes:/opt/nti-server/libs/ com.nti56.datatrans.DataTransApplication
```

end



#### 区别：

1. 打包时指定了主类（应用入口），可以直接用java -jar {xxx.jar}。
2. 打包时没有指定主类，可以用java -cp {xxx.jar} {主类名称（FQN）}。
3. 要引用其他的jar包，可以用java -{[classpath|cp]} {$CLASSPATH}:{xxxx.jar} {主类名称（FQN）}。



### pom.xml: mark

配置pom文件生成可以单独运行的jar artifact

#### pom.xml： properties？

```xml
   <properties>

		...
        <!--项目启动类，各子系统根据项目实际更改 -->
        <app.main.class>com.nti.NTIApplication</app.main.class>
        <!--项目启动端口，各子系统根据项目实际更改；尽量都以8080为启动端口，docker或k8s配置代理端口统一-->
        <server.port>8080</server.port>
    </properties>

```

项目定义了main class，所以可以直接`java -jar`启动项目。

#### pom.xml：main class

指定manifest

好多不同的插件可以定义main class

* `maven-jar-plugin`

  maven-jar-plugin用于生成META-INF/MANIFEST.MF文件的部分内容，<mainClass>com.xxg.Main</mainClass>指定MANIFEST.MF中的Main-Class，<addClasspath>true</addClasspath>会在MANIFEST.MF加上Class-Path项并配置依赖包，<classpathPrefix>lib/</classpathPrefix>指定依赖包所在目录。

  例如下面是一个通过maven-jar-plugin插件生成的MANIFEST.MF文件片段：

  Class-Path: lib/commons-logging-1.2.jar lib/commons-io-2.4.jar
  Main-Class: com.xxg.Main

  只是生成MANIFEST.MF文件还不够，maven-dependency-plugin插件用于将依赖包拷贝到<outputDirectory>${project.build.directory}/lib</outputDirectory>指定的位置，即lib目录下。

  配置完成后，通过mvn package指令打包，会在target目录下生成jar包，并将依赖包拷贝到target/lib目录下，目录结构如下：

  ```bash
  target/
   |
   |_lib/
     |
     |_ a.jar
   |
   |_test.jar
  ```

  指定了Main-Class，有了依赖包，那么就可以直接通过java -jar xxx.jar运行jar包。

  这种方式生成jar包有个缺点，就是生成的jar包太多不便于管理，下面两种方式只生成一个jar文件，包含项目本身的代码、资源以及所有的依赖包。

  ```xml
  <build>
          <plugins>
              <plugin>
                  <groupId>org.apache.maven.plugins</groupId>
                  <artifactId>maven-jar-plugin</artifactId>
                  <version>3.0.2</version>
                      
                  <configuration>
                      <archive>
                          <manifest>
                              <mainClass>com.ramki.project.Main</mainClass>
                          </manifest>
                      </archive>
                  </configuration>
              </plugin>
          </plugins>
  </build>
  ```

  end

* `assembly:single`

  ```xml
  <build>
  	<plugins>
  
  		<plugin>
  			<groupId>org.apache.maven.plugins</groupId>
  			<artifactId>maven-assembly-plugin</artifactId>
  			<version>2.5.5</version>
  			<configuration>
  				<archive>
  					<manifest>
  						<mainClass>com.xxg.Main</mainClass>
  					</manifest>
  				</archive>
  				<descriptorRefs>
  					<descriptorRef>jar-with-dependencies</descriptorRef>
  				</descriptorRefs>
  			</configuration>
  			<executions>
  				<execution>
  					<id>make-assembly</id>
  					<phase>package</phase>
  					<goals>
  						<goal>single</goal>
  					</goals>
  				</execution>
  			</executions>
  		</plugin>
  
  	</plugins>
  </build>
  ```

  其中`<phase>package</phase>`、`<goal>single</goal>`表示在执行package打包时，执行`assembly:single`，所以可以直接使用`mvn package`打包。

  > 不加`package and single`，构建命令为了`mvn package assembly:single`

  不过，如果项目中用到Spring Framework，用这种方式打出来的包运行时会出现读取XML schema文件异常等奇怪的错误.

* `maven-shade-plugin`

  ```xml
  <build>
  	<plugins>
  
  		<plugin>
  			<groupId>org.apache.maven.plugins</groupId>
  			<artifactId>maven-shade-plugin</artifactId>
  			<version>2.4.1</version>
  			<executions>
  				<execution>
  					<phase>package</phase>
  					<goals>
  						<goal>shade</goal>
  					</goals>
  					<configuration>
  						<transformers>
  							<transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
  								<mainClass>com.xxg.Main</mainClass>
  							</transformer>
  						</transformers>
  					</configuration>
  				</execution>
  			</executions>
  		</plugin>
  
  	</plugins>
  </build>
  ```

  配置完成后，执行`mvn package`即可打包。在target目录下会生成两个jar包，注意不是original-xxx.jar文件，而是另外一个。和maven-assembly-plugin一样，生成的jar文件包含了所有依赖，所以可以直接运行。

  不过如果项目中用到了Spring Framework，将依赖打到一个jar包中，运行时会出现读取XML schema文件出错。原因是Spring Framework的多个jar包中包含相同的文件spring.handlers和spring.schemas，如果生成单个jar包会互相覆盖。为了避免互相影响，可以使用`AppendingTransformer`来对文件内容追加合并。

* end

#### maven-shade-plugin and spring boot将项目全部依赖打入jar包:cry:

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

### 引用：

1. https://blog.csdn.net/zdash21/article/details/101310736
2. https://www.jianshu.com/p/d67a81a0cb88
3. https://xxgblog.com/2015/08/07/maven-create-executable-jar/
4. https://blog.csdn.net/xiao__gui/article/details/47341385

