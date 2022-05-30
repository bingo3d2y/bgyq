## Golang 包管理

### package and directory

Golang 中一个Directory中只能有一个package。不然会提示错误`Multiple packages in the directory: packae_name_1, packae_name_2`



### 从dep到go mod

在原项目中直接运行 `go mod init`。注意该命令会尝试把 *Gopkg.lock* 转成 *go.mod*，过程可能会非常慢。可以把 *Gopkg.lock* 重命名或挪到其他地方去。`init` 成功后运行 `go mod tidy` 生成依赖。然后对比 *go.mod* 和 *Gopkg.lock*中的版本，尽量修改成一致。

go mod 默认不会把依赖放在 vendor 目录，需要运行 `go mod vendor` 命令生成。

有了内部代理后，可以选择不把 vendor 同源码一起管理。

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

### go mod

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

#### 本地包：相同项目

一个项目下不同模块可以直接`import package`使用

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/go-import.jpg)



##### 本地包: 不同项目模块

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



##### 本地包: 无法下载的远程仓库包

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