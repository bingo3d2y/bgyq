## Golang Protobuf

在Kafka中，发送的消息是字节数组，因此就需要一个方法来将消息对象序列化为字节数组，在消费者端再反序列化为对象。最常用的序列化格式就是JSON了。虽然JSON对人类非常友好，但是对于机器来说，更容易进行序列化和反序列化的格式还是二进制的格式。

Protobuf（Protocol buffers）是由Google开发的一种二进制协议，用于对结构化数据进行序列化和反序列化。这种格式占用空间更少，更加简单易于维护，同时拥有更好的性能。对于机器之间的通信，Protobuf是比XML和JSON等格式更好的一种选择。

### 目录规划

一般情况下，我们会将编译生成的 pb.go 文件生成在与 proto 文件相同的目录，这样我们就不需要再创建相同的目录层级结构来存放 pb.go 文件了。由于同一文件夹下的 pb.go 文件同属于一个 package，所以在定义 proto 文件的时候，相同文件夹下的 proto 文件也应声明为同一的 package，并且和文件夹同名，这是因为生成的 pb.go 文件的 package 是取自 proto package 的。

通常，存放proto文件的目录为`./pb/`。

### protoc安装

下载不同操作系统对应的二进制文件，然后放到系统变量PTAH中即可执行`protoc`

#### go mod vendor

could not import google.golang.org/protobuf/reflect/protoreflect (no package for import google.golang.org/protobuf/reflect/protoreflect)

solution：

I had the same problem, and it turned out that I just forgot to add protobuf to go.mod.

```bash
$ go mod vendor
go: finding module for package google.golang.org/protobuf/reflect/protoreflect
go: finding module for package google.golang.org/protobuf/runtime/protoimpl
go: found google.golang.org/protobuf/reflect/protoreflect in google.golang.org/protobuf v1.28.0
go: found google.golang.org/protobuf/runtime/protoimpl in google.golang.org/protobuf v1.28.0

# 查看go.mod的变化
module test

go 1.17

## 自动增加了 require
require google.golang.org/protobuf v1.28.0

```

end

### proto语法

https://www.cnblogs.com/shijingxiang/articles/14370775.html



在.proto文件的第一行就是syntax = "proto3";，用于声明该文件是proto3版本的。之后可以声明package用于避免命名冲突，最后就可以定义message了。

```go
syntax = "proto3";

package coredns.dns;
option go_package = "./;pb";

message DnsPacket {
	bytes msg = 1;
}

service DnsService {
	rpc Query (DnsPacket) returns (DnsPacket);
}
```

**package**用于proto,在引用时起作用;
**option go_package**用于生成的.pb.go文件,在引用时和生成go包名时起作用

#### message



#### package

Golang 中一个Directory中只能有一个package。不然会提示错误`Multiple packages in the directory: packae_name_1, packae_name_2`

You can add an optional package specifier to a .proto file to prevent name clashes between protocol message types.

在 proto 文件中添加可选包说明符组织 protocol 消息类型之间的命名冲突。

```go
package foo.bar;
message Open { ... }
```

You can then use the package specifier when defining fields of your message type:

当定义消息类型的字段时候也可以使用包识别符：

```go
message Foo {
  ...
  required foo.bar.Open open = 1;
  ...
}
```

end

#### go_package

The `go_package` option defines the import path of the package which will contain all the generated code for this file. The Go package name will be the last path component of the import path. For example: `"example.com/protos/foo;package_name"`. This usage is discouraged since the package name will be derived by default from the import path in a reasonable manner.

```go
package coredns.dns;
// go_package = ".;pb";报错：
// protoc-gen-go: invalid Go import path "." for "pb/test.proto"
// The import path must contain at least one forward slash ('/') character.

option go_package = "./;pb";
```

go_package的value包含了`;`
后面是就是生成go代码时，package名
前面是生成代码时，如果其他proto **引用** 了这个proto，会使用`;`前面的作为go包路径

Coredns 这样写是因为，所有的proto都在一个目录--

#### import

一般情况下，我们会将编译生成的 pb.go 文件生成在与 proto 文件相同的目录，同属于一个包内的 proto 文件之间的引用也需要声明 `import` ，因为每个 proto 文件都是相互独立的，这点不像 Go（**Golang定义包内所有定义均可见**）。我们的项目 user 模块下 service.proto 就需要用到 message.proto 中的 message 定义，`user/service.proto`代码是这样写的

```go
syntax = "proto3";  
package user;  // 声明所在包
option go_package = "test.com/test/pb-demo/proto/user";  // 声明生成的 go 文件所属的包
  
import "proto/user/message.proto";  // 导入同包内的其他 proto 文件
import "proto/article/message.proto";  // 导入其他包的 proto 文件
  
service User {  
    rpc GetUserInfo (UserID) returns (UserInfo);  
    rpc GetUserFavArticle (UserID) returns (article.Articles.Article);  
}
```

`package` 属于 proto 文件自身的范围定义，与生成的 go 代码无关，它不知道 go 代码的存在（但 go 代码的 `package` 名往往会取自它）。这个 proto 的 `package` 的存在是为了避免当导入其他 proto 文件时导致的文件内的命名冲突。所以，当导入非本包的 `message` 时，需要加 package 前缀，如 service.proto 文件中引用的 `Article.Articles`，点号选择符前为 package，后为 message。同包内的引用不需要加包名前缀。

`user.proto`这样定义

```go
syntax = "proto3";  
package user;  
option go_package = "test.com/test/pb-demo/proto/user";  
  
message UserID {  
    int64 ID = 1;  
}  
  
message UserInfo {  
    int64 ID = 1;  
    string Name = 2;  
    int32 Age = 3;  
    gender Gender = 4;  
    enum gender {  
        MALE = 0;  
        FEMALE = 1;  
    }  
}
```

`article/message.proto`内容

```go
syntax = "proto3";  
package article;  
option go_package = "test.com/test/pb-demo/proto/article";  
  
message Articles {  
    repeated Article Articles = 1;  
    message Article {  
        int64 ID = 1;  
        string Title = 2;  
    }  
}
```

而 `option go_package` 的声明就和生成的 go 代码相关了，它定义了生成的 go 文件所属包的完整包名，所谓完整，是指相对于该项目的完整的包路径，应以项目的 Module Name 为前缀。如果不声明这一项会怎么样？最开始我是没有加这项声明的，后来发现 **依赖这个文件的** 其他包的 proto 文件 **所生成的 go 代码** 中，引入本文件所生成的 go 包时，`import` 的路径并不是基于项目 Module 的完整路径，而是import 在执行 `protoc` 命令时相对于 `--proto_path` 的包路径，这导致在 go build 时是找不到要导入的包的。

这里听起来可能有点绕，建议大家亲自尝试一下。

### 编译proto文件

通常情况下我们编译命令是这样的（基于本项目来说，执行命令的 `pwd` 为项目根目录）：

```bash
# 通常protocol buffer 文件都在 pb 目录下
# 因为service下proto引用了其他目录下的proto文件，所以需要指定base目录
$ protoc --proto_path=./pb/base:./pb/service --go_out=./pb ./pb/base/*.proto ./pb/service/*.proto  
```

其中，`--proto_path` 或者 `-I` 参数用以指定所编译源码（包括直接编译的和被导入的 proto 文件）的搜索路径，proto 文件中使用 `import` 关键字导入的路径一定是要基于 `--proto_path` 参数所指定的路径的。该参数如果不指定，默认为 `pwd` ，也可以指定多个以包含所有所需文件。

其中，`--go_out` 参数是用来指定 protoc-gen-go 插件的工作方式 和 go 代码目录架构的生成位置，可以向 `--go_out` 传递很多参数，见 [golang/protobuf 文档](https://github.com/golang/protobuf#parameters) 。

主要的两个参数为 **plugins** 和 **paths** ，代表 生成 go 代码所使用的插件 和 生成的 go 代码的目录怎样架构。

例如：`--go_out=plugins=grpc,paths=import:.` 

* `--go_out` 参数的写法是，参数之间用逗号隔开，最后加上冒号来指定代码目录架构的生成位置。

* paths 参数有两个选项，`import` 和 `source_relative` 。默认为 `import` ，代表按照生成的 go 代码的包的全路径去创建目录层级，`source_relative` 代表按照 proto 源文件的目录层级去创建 go 代码的目录层级，如果目录已存在则不用创建。

  `import`即根据proto文件中定义的`package NAME`去生成相应的目录

在上面的示例命令中，`--go_out` 默认使用了 `paths=import` 所以，我的 go 文件都被编译到了 `./test/pb/proto/user/` 下，后来阅读 [文档](https://github.com/golang/protobuf#packages-and-input-paths) 才发现:

However, the output directory is selected in one of two ways. Let us say we have `inputs/x.proto` with a `go_package` option of `github.com/golang/protobuf/p` . The corresponding output file may be:

* Relative to the import path:

  ```bash
  $ protoc --go_out=. inputs/x.proto
  # writes ./github.com/golang/protobuf/p/x.pb.go
  ```

  ( This can work well with `--go_out=$GOPATH` )

* Relative to the input file:

  即生成的go文件在`x.proto`所在的目录

  ```bash
  $ protoc --go_out=paths=source_relative:. inputs/x.proto
  # generate ./inputs/x.pb.go
  ```

所以，我们应该将 `--go_out` 参数改为 `--go_out=paths=source_relative:.` 。

**请切记 `option go_package` 声明和 `--go_out=paths=source_relative:.` 命令行参数缺一不可** 。

> 等价于`--go-out=./ --go-out=paths=source_relative`

- `option go_package` 声明 是为了让生成的其他 go 包（依赖方）可以正确 `import` 到本包（被依赖方）
- `--go_out=paths=source_relative:.` 参数 是为了让加了 `option go_package` 声明的 proto 文件可以将 go 代码编译到与其同目录。

一般用法为了统一性，我会将所有 proto 文件中的 `import` 路径写为相对于项目根目录的路径，然后 `protoc` 的执行总是在项目根目录下进行：

`--go_out=plugins=grpc`参数来生成gRPC相关代码，如果不加`plugins=grpc`，就只生成`message`数据.

```bash
$ protoc --go_out=plugins=grpc,paths=source_relative:. ./proto/user/*.proto 
$ protoc --go_out=plugins=grpc,paths=source_relative:. ./proto/article/*.proto
```

如果你觉得每个包都需要单独编译，有些麻烦，可以执行脚本（ `**/*` 代表递归获取当前目录下所有的文件和文件夹）

```bash
# -IPATH, --proto_path=PATH   Specify the directory in which to search for
#                              imports.  May be specified multiple times;
#                              directories will be searched in order.  If not
#                              given, the current working directory is used.
#                              If not found in any of the these directories,
#                              the --descriptor_set_in descriptors will be
#                              checked for required proto file.
#
# pwd is package test dir
$ protoc --go_out=. --go_opt=paths=source_relative ./pb/*.proto
$ tree
.
├── test.pb.go
├── test.proto

```

end

#### 两种写法

```bash
$ protoc --go_out=./ --go_out=plugins=grpc,paths=source_relative:. a.proto
$ md5sum a.pb.go
4ff25245f2602a4dac0f6c067db2de49  a.pb.go

$ \rm a.pb.go
$ protoc --go_out=./ --go_opt=plugins=grpc,paths=source_relative a.proto
$ md5sum a.pb.go
4ff25245f2602a4dac0f6c067db2de49  a.pb.go
## 默认带了grpc参数？
$ protoc --go_out=./ --go_opt=paths=source_relative a.proto            
$ md5sum a.pb.go
4ff25245f2602a4dac0f6c067db2de49  a.pb.go

```

end

### gRPC

http://liuqh.icu/2022/01/20/go/rpc/03-grpc-ru-men/



### 引用

1. https://studygolang.com/articles/25743
2. https://segmentfault.com/a/1190000039767770
3. https://www.cnblogs.com/shijingxiang/articles/14370775.html
