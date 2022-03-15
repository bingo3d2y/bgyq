## Sonar

https://www.cnblogs.com/Uni-Hoang/p/15344228.html

### Sonar安装配置

https://se-2020.github.io/resources/sonarqube-tutorial-v1.pdf

### sonar-scanner

sonar-scanner提供本地的代码扫描，客户端可以安装在与服务端相同的服务器，也可以装在不同的服务器（也可以安装在开发人员本地电脑）。

使用浏览器打开刚刚装好的SonarQube网页，新建project时根据指引，可以找到对应OS的sonar-scanner客户端下载地址以及使用命令。

sonar-scanner配置文件

```bash
# 修改配置
$ vi /data/sonar-scanner-4.6.2.2472-linux/conf/sonar-scanner.properties
# 打开下面两项配置
sonar.host.url=http://IP:9003 # 因为是在本地，不用修改了
sonar.sourceEncoding=UTF-8
# 并添加两行
sonar.login=admin
sonar.password=yourpassword
```

end

### gitlab-runner集成sonar

```bash
$ more .gitlab-ci.yml
stages:
  - test
job_name_sonar:
  stage: test
  only:
    - master
  script:
    - /data/sonar-scanner-4.6.2.2472-linux/bin/sonar-scanner -Dsonar.host.url="http://***.***.***.***:9003" -Dsonar.login=admin -Dsonar.projectKey=TEST_PROJECT2 -Dsonar.sources=.
  tags:
    - dev
```

end

### Maven集成sonar

首先起好sonar的服务

配置maven的`setting.xml`文件

```xml
<profile>
		  <id>sonar</id>
		  <activation>
			  <activeByDefault>true</activeByDefault>
		  </activation>
		  <properties>
			  <!-- 平台登录的账号的用户名,格式：姓全拼+名第一个字母 -->
			  <sonar.login>admin</sonar.login>
			  <!-- SonarQube平台登录的账号的密码，格式：姓全拼+名第一个字母 -->
			  <sonar.password>admin</sonar.password>
			  <!-- SonarQube访问地址 -->
			  <sonar.host.url>http://localhost:9000</sonar.host.url>
			  <!-- 代码分析包括哪些文件需要分析，英文逗号分隔  -->
			  <sonar.inclusions>**/*.java,**/*.xml</sonar.inclusions>
		  </properties>
	  </profile>

```

activeProfiles中加入以下配置，用以让sonar配置生效

```xml
<activeProfile>sonar</activeProfile>
```

打开项目根目录，使用命令进行代码分析

```bash
mvn clean install sonar:sonar
```

end

### Sonar启动失败

Native memory allocation (mmap) failed to map 134217728 bytes（约为128MB） for committing reserved memory.

Java应用启动通病：启用时默认要求分配NGB的内存，由于免费服务器性能限制，出现内存分配的异常

解决：`-Xmx and -Xms`指定配置合适是参数

