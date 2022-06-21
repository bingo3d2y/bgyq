## go modules

### main

一般 `main()`函数写在最前面，然后按函数的调用，开始按顺序声明函数--

`main()`一定要简洁清晰--

### GOPATH[0]

Python 到 Golang，数据结构的熟悉和最基本的modules管理即类似python 的包管理，不能引用sdk和其他modules等于直接废了。

**GO11MODULE将所有的modules都放在GOPATH[0]中，统一管理。**类似maven的local repository

GOPATH可以设置多个路径，但go get 默认安装到第一个GOPATH路径。

即：

```bash
# 所有modules都在 /tmp/aaa/pkg下
$ echo $GOPATH
/tmp/aaa:/go

```

几个简单env：

GOROOT and GOPATH

```bash
# GOROOT: go 二进制文件的安装目录
$ go env|grep GOROOT
GOROOT="/usr/local/go"
$ which go
/usr/local/go/bin/go

# GOPATH: 存放go 代码依赖的地方
$ go env|grep GOPATH
GOPATH="/home/go"
$ ls /home/go/ -l
total 0
drwxr-xr-x.  2 dodo root 160 Mar  9 06:28 bin
drwxr-xr-x.  5 dodo root  41 May 20  2020 pkg
drwxr-xr-x. 12 dodo root 260 Mar 10 03:12 src
$ ls /home/go/pkg/mod/ -l
...
drwxr-xr-x.  31 dodo root 4096 Mar  9 06:17 k8s.io
drwxr-xr-x.   3 dodo root   32 May 21  2020 rsc.io
drwxr-xr-x.  14 dodo root 4096 Mar  9 06:17 sigs.k8s.io

```

end

### golang modules：download

使用`go get` 命令来拉取modules。

几个不错的代理：https://goproxy.cn,direct 和 https://mirrors.aliyun.com/goproxy/,direct

#### 常见公共pkg仓库

代码大多数在github或者google的仓库，所有需要翻墙或配置代理才行。

```bash
# 设置的方式非常简单，只需要设置如下环境变量即可，
export GOPROXY=https://mirrors.aliyun.com/goproxy/
go env -w GOSUMDB=sum.golang.google.cn
# go1.13 加入了 mirror 机制，可以通过 go env -w 设置 mirror
# 逗号后面可以增加多个 proxy，最后的 direct 则是在所有 proxy 都找不到的时候，直接访问，代理访问不到的私有仓库就可以正常使用了。
# 可以在不用用户下设置自己的goproxy不用使用全局的
# 可以定义多个 proxy address
$ go env -w GOPROXY=https://mirrors.aliyun.com/goproxy/,direct;https://goproxy.cn,direct
```

核心思想，就是配置国内大厂的代理、

#### 使用公司内部pkg仓库

```bash
## 即mod的包管理，需要先将mod升到 1.13
# Linux/Mac 用户
# 添加环境变量配置公司的go proxy
go env -w GOPROXY=goproxy.your_dns.com,direct
go env -w GOSUMDB=sum.golang.google.cn
go env -w GONOSUMDB=gopkg.your_dns.com

## 执行升级命令即可： go get -u package_name
go get -u gopkg.your_dns.com/your_package
## 指定版本升级
## 升级到最新的master版本--
$ go get -u gopkg.your_dns.com/your_package@master
go: gopkg.your_dns.com/your_package master => v0.15.1-0.20200824075759-de18f05a14bc
go: downloading gopkg.your_dns.com/your_package v0.15.1-0.20200824075759-de18f05a14bc
```

end

#### packge 校验：关闭或跳过

实践：

* 公网的package使用国内代理校验，避免风险
* 私网的package可以关闭checksum 的校验

Go Checksum Database: https://sum.golang.google.cn/

As of Go 1.13, the go command by default downloads and authenticates modules using the Go checksum database. 

```bash
# Get "https://sum.golang.google.cn/lookup/github.com/!data!dog/zstd@v1.3.5": dial tcp 74.125.203.94:443: connect: connection refused

go env -w GOSUMDB=sum.golang.google.cn
go env -w GONOSUMDB=gopkg.your_dns.com

```

end

#### go get 拉取指定版本

* `-v`: 显示操作流程的日志及信息，方便检查错误
* `-u`: 下载丢失的包，但不会更新已经存在的包
* `-d`: 只下载，不安装

```bash
# @ + version 即可
$ go get k8s.io/klog@v1.0.0
...

# 可以先列出所有版本
$ go list -m -versions gopkg.your_dns.com/your_package
gopkg.your_dns.com/your_package v0.0.1 v0.0.2 v0.0.3 v0.0.4 v0.0.5 v0.0.6 v0.0.7 v0.0.8 v0.0.9 v0.0.10 v0.0.11 v0.0.12 v0.0.13 v0.0.14 v0.0.15 v0.0.16 v0.1.0 v0.1.1 v0.1.2 v0.1.3 v0.1.4 v0.1.5 v0.1.6 v0.1.7 v0.1.8 v0.1.9 v0.1.10 v0.1.11 v0.1.12 v0.1.13 v0.1.14 v0.1.15 v0.2.0 v0.3.0 v0.4.0 v0.5.0 v0.6.0 v0.7.0 v0.8.0 v0.9.0 v0.10.0 v0.11.0 v0.12.0 v0.13.0 v0.13.1 v0.14.0 v0.15.0 v0.15.1 v0.15.2 v0.16.0 v0.16.1 v0.17.0 v0.17.1 v0.18.0 v0.18.1 v0.18.2

# 直接获取最新版本
$ go get gopkg.your_dns.com/your_package@latest
```

end

#### go get github pack

今天，拉取github上的包，报错了。因为获取github的包默认是ssh协议，但是你默认走的是https，所以需要输入密码。

It complains because it needs to use `ssh` instead of `https` but your git is still configured with https. so basically as others mentioned previously you need to either enable prompts or to configure git to use `ssh` instead of `https`. 

```bash
$ go get  "github.com/EDDYCJY/go-pprof-example/data"
# cd .; git clone https://github.com/EDDYCJY/go-pprof-example /root/go/src/github.com/EDDYCJY/go-pprof-example
Cloning into '/root/go/src/github.com/EDDYCJY/go-pprof-example'...
fatal: could not read Username for 'https://github.com': terminal prompts disabled

```

解决方法

①手动输入账号密码，

go get disables the "terminal prompt" by default. This can be changed by setting an environment variable of git:

```bash
$ env GIT_TERMINAL_PROMPT=1 go get  "github.com/EDDYCJY/go-pprof-example/data"
Username for 'https://github.com': dodo
...
```

②配置git使用ssh替代https

```bash
$ git config --global --add url."git@github.com:".insteadOf "https://github.com/"
$ go get  "github.com/EDDYCJY/go-pprof-example/data"
# cd .; git clone https://github.com/EDDYCJY/go-pprof-example /root/go/src/github.com/EDDYCJY/go-pprof-example
Cloning into '/root/go/src/github.com/EDDYCJY/go-pprof-example'...
Warning: Permanently added the RSA host key for IP address '13.250.177.223' to the list of known hosts.
Permission denied (publickey).
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.
package github.com/EDDYCJY/go-pprof-example/data: exit status 128

```

用法解释： 所有git命令都将git://替换为https://

```bash
git config --global url."https://".insteadOf git://
```

end

### golang modules：manage

`go modules` 是 golang 1.11 新加的特性。

#### version 1： 单纯GOPATH，必看

**最大缺陷：N个项目都依赖K module，则本地需要存储N份同样的module，且更新很不方便。**

其次，GOPATH没有版本概念，默认拉取最新版本package

最开始 Go 官方根本没有提供什么包管理机制，需要每个项目做一个 GOPATH，这样当前编译或运行当前项目时，回去当前目录下pkg目录下找它的依赖包。

```bash
# 声明多个 GOPATH和添加项目的bin目录到 PATH
$ export GOPATH=~/gopath1:~/gopath2:~/gopath3
$ export PATH=$PATH:/data/gopath1/bin:/data/gopath2/bin
```

每个项目一个GOPATH很不灵活，因为`go get` 时会将module下载到第一个`GOPATH`的value中，如下：

```bash
# 额外新加一个GOPATH
$ export GOPATH=/tmp/aaa:$GOPATH
$ mkdir /tmp/aaa
$ go env|grep -i gopath
GOPATH="/tmp/aaa:/go"

# go get 下载一个module，观察存放在哪个目录
$ go get github.com/davyxu/cellnet
# 很明显，没在后面的 GOPATH目录，因为连 pkg包都没有--
root@943e74637215:/go# ls /go/
bin/ src/
# 如下，果然在GOPATH[0]
root@943e74637215:/go# ls /tmp/aaa/
pkg/ src/
root@943e74637215:/go# ls /tmp/aaa/pkg/
linux_amd64
root@943e74637215:/go# ls /tmp/aaa/pkg/linux_amd64/github.com/davyxu/
cellnet.a

```

当机器上有N个GOPATH时，这没得玩了，每次你要将要拉取依赖的PROJECT_GOPATH挪到GOPATH[0] ，这样才能保证，module会保存到期望的项目中。

#### version 2：Godep && Vendor等等

空白？？？哈哈哈

#### version 3：go mod:+1:

`Go Modules`解决方式很像是`Java`看到`Maven`的做法，将第三方库储存在本地的空间，并且给项目代码去引用。

在 Go Module 模式下，`go get` 的行为会发生改变。在 GOPATH 模式下，`go get` 会把代码下载到 `GOPATH` 下对应的 `src` 目录下。而在 Go Module 模式下，`go get` 会给当前的项目下载最新版本的依赖，并写入 `go.mod` 和 `go.sum`，依赖同一下载存储到`GOPATH[0]`

如果下载的依赖没有被项目的代码直接 import，则会在 `go.mod` 的对应项目中自动添加注释 `// indirect`

```bash
$ go mod --help
download    download modules to local cache (下载依赖的module到本地cache))
edit        edit go.mod from tools or scripts (编辑go.mod文件)
graph       print module requirement graph (打印模块依赖图))
init        initialize new module in current directory (再当前文件夹下初始化一个新的module, 创建go.mod文件))
tidy        add missing and remove unused modules (增加丢失的module，去掉未用的module)
vendor      make vendored copy of dependencies (将依赖复制到vendor下)
verify      verify dependencies have expected content (校验依赖)
why         explain why packages or modules are needed (解释为什么需要依赖)
```

end

### GO11MODULE： 概述

将所有的modules都放在GOPATH[0]中，统一管理。

>  说实话，这命名真...  由于go 1.11 引入的modules机制，所有这个env叫`GO11MODULE`

在 Go Module 模式下，`go get` 的行为会发生改变。在 GOPATH 模式下，`go get` 会把代码下载到 `GOPATH` 下对应的 `src` 目录下。而在 Go Module 模式下，`go get` 会给当前的项目下载最新版本的依赖，并写入 `go.mod` 和 `go.sum`

如果下载的依赖没有被项目的代码直接 import，则会在 `go.mod` 的对应项目中自动添加注释 `// indirect`

go module可以使得项目可以在GOPATH路径外，使用`go mod`命令管理依赖，很方便

> GOPATH目录外的项目，执行`go mod tidy`就可以将go code中的依赖下载到`GOPATH[0]/pkg/`而且记录到当前的go.mod文件中。

Golang 提供一个环境变量 GO111MODULE 来设置是否使用mod，它有3个可选值，分别是off, on, auto（默认值），具体含义如下：

* off: **GOPATH mode**，查找vendor和GOPATH目录

  off就是使用按GOPATH的优先级，去发现modules了。

* on：module-aware mode，使用 go module，忽略GOPATH目录

  即，将N个项目的modules统一管理，避免冗余

* auto：如果当前目录不在$GOPATH 并且 当前目录（或者父目录）下有go.mod文件，则使用 GO111MODULE， 否则仍旧使用 GOPATH mode。

#### go mod init

##### 在GOPTAH下执行go mod init

`go mod` 命令在 `$GOPATH` 里默认是执行不了的，因为 `GO111MODULE` 的默认值是 `auto`。默认在`$GOPATH` 里是不会执行， 如果一定要强制执行，就设置环境变量为 `on`。

```bash
## 如下 auto 模式下，无法在GOPATH执行 go mod init
root@943e74637215:/tmp/aaa# echo $GOPATH
/tmp/aaa:/go
root@943e74637215:/tmp/aaa# cd /tmp/aaa/
root@943e74637215:/tmp/aaa# go mod int
root@943e74637215:/tmp/aaa# go mod init
go: cannot determine module path for source directory /tmp/aaa (outside GOPATH, module path must be specified)

Example usage:
        'go mod init example.com/m' to initialize a v0 or v1 module
        'go mod init example.com/m/v2' to initialize a v2 module

Run 'go help mod init' for more information.
root@943e74637215:/tmp/aaa# cd /go/
root@943e74637215:/go# go mod init
go: cannot determine module path for source directory /go (outside GOPATH, module path must be specified)

Example usage:
        'go mod init example.com/m' to initialize a v0 or v1 module
        'go mod init example.com/m/v2' to initialize a v2 module

Run 'go help mod init' for more information.

```

end

##### 在GOPATH外执行go mod init

在GOPATH外的路径使用`go mod init`时，需要指定mod name.

```bash
$ go mod init
go: cannot determine module path for source directory E:\golang\test_mod_init (outside GOPATH, module path must be specified)

Example usage:
        'go mod init example.com/m' to initialize a v0 or v1 module
        'go mod init example.com/m/v2' to initialize a v2 module

Run 'go help mod init' for more information.

pc MINGW64 /e/golang/test_mod_init
$ go mod init test_mod_init
go: creating new go.mod: module test_mod_init

pc MINGW64 /e/golang/test_mod_init
$ cat go.mod
module test_mod_init

go 1.16

```

end

#### 对与先引用package后使用go mod int

```bash
# awesomeproject下已经有imprt了
$ cat test.go
package main

import (
	"fmt"
	"sigs.k8s.io/controller-runtime/pkg/client"
)

func main()  {
	a := client.Client
	fmt.Println(a)
}
# 使用mod 管理package
$ go mod init awesomeProject
go: creating new go.mod: module awesomeProject
go: to add module requirements and sums:
        go mod tidy

# 会有提示哦
$ go mod tidy
go: finding module for package sigs.k8s.io/controller-runtime/pkg/client

## 变更了 package，无论是引用了新的还是更新了版本，都需要是 go mod tidy 更新go.mod
$ go mod tidy
...
噼里啪啦 一顿下载
...
$  cat go.mod
module test

go 1.14

require sigs.k8s.io/controller-runtime v0.8.3
```

end

#### go mod graph

go modules 版本冲突怎么办？

这就要梳理版本了，是最没有捷径的。一个比较简单的办法是把所有 `go.mod` 里不需要指定版本的包全部删掉，仅指定必要的包版本，然后通过 `go build` 让项目自动构建依赖包的版本。

`go mod graph`可以显示出模块见的依赖关系

```bash
$ go mod graph|grep -v ^github.com/envoyproxy/go-control-plane |grep github.com/envoyproxy/go-control-plane
google.golang.org/grpc@v1.28.0 github.com/envoyproxy/go-control-plane@v0.9.4
google.golang.org/grpc@v1.26.0 github.com/envoyproxy/go-control-plane@v0.9.1-0.20191026205805-5f8ba28d4473
google.golang.org/grpc@v1.25.1 github.com/envoyproxy/go-control-plane@v0.9.0
google.golang.org/grpc@v1.27.1 github.com/envoyproxy/go-control-plane@v0.9.1-0.20191026205805-5f8ba28d4473
google.golang.org/grpc@v1.27.0 github.com/envoyproxy/go-control-plane@v0.9.1-0.20191026205805-5f8ba28d4473
github.com/hashicorp/consul@v1.8.0 github.com/envoyproxy/go-control-plane@v0.8.0
google.golang.org/grpc@v1.20.0 github.com/envoyproxy/go-control-plane@v0.6.9
mosn.io/mosn@v0.21.0 github.com/envoyproxy/go-control-plane@v0.9.4

```

左边是项目包，右边是被依赖的包和版本。

如果确实存在两个需要指定版本的包互相冲突，那就要做取舍，修改代码，升级或降级某个包了。 

#### go mod vendor

`go mod vendor` 会复制modules下载到你的go project的vendor目录中。

```bash
$ ls vendor/gopkg.your_dns.com/ -l
total 0
drwxrwxr-x.  3 gitlab-runner gitlab-runner  20 Mar 11 18:19 account
drwxrwxr-x.  3 gitlab-runner gitlab-runner  17 Mar 11 18:19 plat
drwxrwxr-x. 16 gitlab-runner gitlab-runner 211 Mar 11 18:19 your_package
$ cat go.mod
module account/address

go 1.13

require (
        github.com/360EntSecGroup-Skylar/excelize/v2 v2.3.1
        github.com/golang/protobuf v1.4.3
        github.com/minio/minio-go/v7 v7.0.6
        github.com/pkg/errors v0.9.1 // indirect
        go.uber.org/zap v1.10.0
        google.golang.org/grpc v1.28.1
        gopkg.your_dns.com/account/common v1.1.1-0.20200421140853-1c517ff8cf22
        gopkg.your_dns.com/plat/kit v0.0.0-20210223082909-7eb04e276634
        gopkg.your_dns.com/your_package v0.18.1
)

```

end

#### replace

replace顾名思义，就是用新的package去替换另一个package，他们可以是不同的package，也可以是同一个package的不同版本.

```bash
$ go mod edit -replace=old[@v]=new[@v]
# 或者直接修改go.mod 文件
$ vim go.mod 
..
replace (
	google.golang.org/grpc => google.golang.org/grpc v1.26.0
)
..
```

go mod可能都N个模块依赖一个包事，为了避免冲突，go mod 帮我们选择了一个连间接依赖都算不上的外部库指定的版本来进行更新。但这可能导致，出现问题，最好使用 replace 替换指定版本。

对 `github.com/envoyproxy/go-control-plane` 有依赖的也就只有 mosn，只要 replace 就好

#### go mod tidy：重要

构建过程会更新go.mod文件(在执行go get、go build、 go test、 go list等命令时都会根据需要的依赖自动和更新require语句)

**核心： go get 后使用`go tidy`更新go.mod 和 go.sum 文件。**

go.mod文件一旦创建后，它的内容将会被go toolchain全面掌控。go toolchain会在各类命令执行时，比如go get、go build、go mod等修改和维护go.mod文件。 			

使用命令 `go list -m -u all` 来检查可以升级的package，使用`go get -u need-upgrade-package` 升级后会将新的依赖版本更新到go.mod * 也可以使用 `go get -u` 升级所有依赖

```bash
## 升级到最新的master版本--
$ go get -u gopkg.your_dns.com/your_package@master
go: gopkg.your_dns.com/your_package master => v0.15.1-0.20200824075759-de18f05a14bc
go: downloading gopkg.your_dns.com/your_package v0.15.1-0.20200824075759-de18f05a14bc


## 升级完成后，修改go.sum
## go.mod 和 go.sum文件会被修改，然后git push 更新下仓库
$ go mod tidy
## git 更新--
$git add . && git commit && git push

```

end

#### 离线安装module

当goproxy拉不到module时使用下面办法

1.下载源码:

到github上去下载zip包解压 

```
https://github.com/golang/text
```

或者git拉取

```
git clone https://github.com/golang/text.git   
```

2.编译安装源码

1.${gopath}下一般会有 src , pkg , bin 三个目录, 将下载text包放在 ${gopath}/src/golang.org/x 目录下

2.在 ${gopath}/src 目录下执行

```bash
go install -x  golang.org/x/text
```

这样就会在pkg目录下生成一个text.a的包文件

注意:

go install的执行路径为 ${gopath}/src/  加上go install命令后面跟的目录

#### 从dep到go mod

在原项目中直接运行 `go mod init`。注意该命令会尝试把 *Gopkg.lock* 转成 *go.mod*，过程可能会非常慢。可以把 *Gopkg.lock* 重命名或挪到其他地方去。`init` 成功后运行 `go mod tidy` 生成依赖。然后对比 *go.mod* 和 *Gopkg.lock*中的版本，尽量修改成一致。

go mod 默认不会把依赖放在 vendor 目录，需要运行 `go mod vendor` 命令生成。

有了内部代理后，可以选择不把 vendor 同源码一起管理。

### package and directory

Golang 中一个Directory中只能有一个package。不然会提示错误`Multiple packages in the directory: packae_name_1, packae_name_2`

### 部署项目到新机器

```bash
# generate pb.go
# 根据Makefile定义执行不同的命令
$ make pb
# download dependencies
$ go mod init & go mod tidy
# build
$ make pb
$ make
# run service
$ ./exxx serve
```

类似下面的Makefile文件

```makefile
pb: ## Copy mcube protobuf files to common/pb
	@mkdir -pv common/pb/github.com/infraboard/mcube/pb
	@cp -r ${MCUBE_PKG_PATH}/pb/* common/pb/github.com/infraboard/mcube/pb
	@sudo rm -rf common/pb/github.com/infraboard/mcube/pb/*/*.go
```

end

### go mod实践

#### 配置代理

```bash
## 即mod的包管理，需要先将mod升到 1.13
# Linux/Mac 用户
# 添加环境变量配置公司的go proxy
go env -w GOPROXY=goproxy.test.com,direct
go env -w GOSUMDB=sum.golang.google.cn
go env -w GONOSUMDB=gopkg.test.com

```

end

#### 初始化go.mod

适用场景：将代码中引用的第三方依赖，自动写入到go.mod文件

could not import google.golang.org/protobuf/reflect/protoreflect (no package for import google.golang.org/protobuf/reflect/protoreflect)

solution：

I had the same problem, and it turned out that I just forgot to add protobuf to go.mod.

如何添加呢，运行`go mod tidy`

```bash
$ go mod tidy
D:\DoDo\gopath\src\test>go mod tidy
go: finding module for package google.golang.org/grpc/status
go: finding module for package google.golang.org/grpc
go: finding module for package google.golang.org/grpc/codes
go: downloading google.golang.org/grpc v1.46.2
...
go: downloading golang.org/x/text v0.3.3



# 查看go.mod的变化
module test

go 1.17

## 自动增加了 require中的内容
require (
	github.com/golang/protobuf v1.5.2 // indirect
	golang.org/x/net v0.0.0-20201021035429-f5854403a974 // indirect
	golang.org/x/sys v0.0.0-20210119212857-b64e53b001e4 // indirect
	golang.org/x/text v0.3.3 // indirect
	google.golang.org/genproto v0.0.0-20200526211855-cb27e3aa2013 // indirect
)

```

对于一个新创建的项目，我们可以在项目文件夹下按照以下步骤操作：

1. 执行`go mod init 项目名`命令，在当前项目文件夹下创建一个`go.mod`文件。
2. 手动编辑`go.mod`中的require依赖项或执行`go get`自动发现、维护依赖

#### 更新go.mod中依赖版本

tidy        add missing and remove unused modules

```bash
## 执行升级命令即可： go get -u package_name
go get -u gopkg.test.com/test
## 指定版本升级
## 升级到最新的master版本--
$ go get -u gopkg.test.com/test@master
go: gopkg.test.com/test master => v0.15.1-0.20200824075759-de18f05a14bc
go: downloading gopkg.test.com/test v0.15.1-0.20200824075759-de18f05a14b

## 升级完成后，修改go.sum
## go.mod 和 go.sum文件会被修改，然后git push 更新下仓库
go mod tidy
git add . && git commit && git push
```

end

#### require:shamrock:

##### 远程仓库依赖包

`require`用来定义依赖包及版本， go编译时回去远程仓库下载这些指定的依赖包

```go
module test

go 1.17

require (
	github.com/coredns/coredns v1.9.2
	google.golang.org/protobuf v1.28.0
    ...
)
```

end

#### 引用本地包：相同项目

一个项目下不同模块可以直接`import package`使用

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/go-import.jpg)

#### 引用本地包: 不同项目模块

当两个包不在同一个项目路径下，你想要导入本地包，并且这些包也没有发布到远程的github或其他代码仓库地址。这个时候我们就需要在`go.mod`文件中使用`replace`指令。

```go
go 1.17

require (
	google.golang.org/protobuf v1.28.0  
	test-server v0.0.0-00010101000000-000000000000 // 这个会版本在你执行 go build 之后自动在该文件添加
)

replace test-server => ../test-server // 指定需要的包目录去后面这个路径中寻找

```

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/go-mod-replace.jpg)

#### 引用本地包: 无法下载的远程仓库包

这事 replace的变种应用，当你依赖某个第三方库且因为网络原因无法下载时，你可以先从别的途径下载到本地，然后通过replace实现无法拉取依赖问题。



#### go mod vendor and download：第三方库

go mod vendor将引入的第三方库自动加到go.mod，并将依赖拷贝到当前的vendor目录。

##### vendor

一般第三方依赖库（包括公司内网gitlab上的依赖库），其源码都不被包含在我们的项目内部，而是在编译的时候go连接公网、内网下载到本地GOPATH，然后编译。

但是当由于网络原因、项目部署便捷性或者依赖库版本丢失等问题，导致下载编译失败时，可以使用`go vendor`命令将项目依赖拷贝到本地目录，直接进行编译避免下载问题。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/go-mod-vendor.jpg)

go mod vendor 执行完成后，查看go.mod 被添加了很多

vendor      make vendored copy of dependencies

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/go-mod-vendor-2.jpg)

修改了go.mod后，必须再执行go mod vendor更新vendor目录的缓存

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/go-mod-vendor-4.png)



##### download

如下还有部分依赖没有下载，根据最新go.mod依赖执行go mod download

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/go-mod-vendor-2.jpg)

download    download modules to local cache

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/go-mod-vendor-3.jpg)

end



### go sum

go.sum是一个构建状态跟踪文件。它会记录当前module所有的顶层和间接依赖，以及这些依赖的校验和，从而提供一个可以100%复现的构建过程并对构建对象提供安全性的保证。

go.sum同时还会保留过去使用的包的版本信息，以便日后可能的版本回退，这一点也与普通的锁文件不同。所以go.sum并不是包管理器的锁文件。

因此我们应该把go.sum和go.mod一同添加进版本控制工具的跟踪列表，同时需要随着你的模块一起发布。如果你发布的模块中不包含此文件，使用者在构建时会报错，同时还可能出现安全风险（go.sum提供了安全性的校验）。

`go mod tidy` updates your current `go.mod` to include the dependencies needed for tests in your module — if a test fails, we must know which dependencies were used in order to reproduce the failure.

`go mod tidy`会将`go.mod`记录的依赖包的版本的hash值记录下来，这样就可以保证编译过程的准确行。

此外执行上面的命令会把`go.mod`的`latest`版本换成实际的最新的版本，并且会生成一个`go.sum`记录每个依赖库的版本和哈希值。

> go.sum 起到了很强的校验作用。

官方文档，很香

https://github.com/golang/go/wiki/Modules#faqs--gomod-and-gosum

`go.sum` contains the expected cryptographic checksums of the content of specific module versions.