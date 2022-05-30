## Golang Protobuf

在Kafka中，发送的消息是字节数组，因此就需要一个方法来将消息对象序列化为字节数组，在消费者端再反序列化为对象。最常用的序列化格式就是JSON了。虽然JSON对人类非常友好，但是对于机器来说，更容易进行序列化和反序列化的格式还是二进制的格式。

Protobuf（Protocol buffers）是由Google开发的一种二进制协议，用于对结构化数据进行序列化和反序列化。这种格式占用空间更少，更加简单易于维护，同时拥有更好的性能。对于机器之间的通信，Protobuf是比XML和JSON等格式更好的一种选择。

### 目录规划

一般情况下，我们会将编译生成的 pb.go 文件生成在与 proto 文件相同的目录，这样我们就不需要再创建相同的目录层级结构来存放 pb.go 文件了。由于同一文件夹下的 pb.go 文件同属于一个 package，所以在定义 proto 文件的时候，相同文件夹下的 proto 文件也应声明为同一的 package，并且和文件夹同名，这是因为生成的 pb.go 文件的 package 是取自 proto package 的。

> 为保证proto文件和pb.go文件在一个目录要使用`protoc --go_opt=paths=source_relative`参数

通常，存放proto文件的目录为`./pb/`。

### protoc安装

下载不同操作系统对应的二进制文件，然后放到系统变量PTAH中即可执行`protoc`

### protoc 语法

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

`;`前面是go包的路径，即`protoc --go_out=plugins=grpc,paths=import`编译命令回自动创建`go_package`生成的目录。

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

##### example

不加`;`，go code package_name就是`/`最后的部分，这里应该`protoc`通过正则匹配提取了。

怪不得要求： The import path must contain at least one forward slash ('/') character.

```go
option go_package = "aaa/grpc/servers";
// 生成的go代码，pakcage如下
package servers

```



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

### Message

#### 定义语法

defining a message type即声明和定义一个用于序列化传输的数据。

```go
// message关键字，像go中的结构体
message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 result_per_page = 3;
}
```

定义字段语法: `类型 字段名 标识号`

每个字段都有唯一的一个数字标识符，一旦开始使用就不能够再改变。

[1, 15]之内的标识号在编码的时候会占用一个字节。[16, 2047]之内的标识号则占用2个字节。

#### 保留标识符(reserved)

reserved 标记的标识号、字段名，都不能在当前消息中使用。

```go
syntax = "proto3";
package demo;

// 在这个消息中标记
message DemoMsg {
  // 标示号：1，2，10，11，12，13 都不能用
  reserved 1,2, 10 to 13;
  // 字段名 test、name 不能用
  reserved "test","name";
  // 不能使用字段名，提示:Field name 'name' is reserved
  string name = 3;
  // 不能使用标示号,提示:Field 'id' uses reserved number 11
  int32 id = 11;
}

// 另外一个消息还是可以正常使用
message Demo2Msg {
  // 标示号可以正常使用
  int32 id = 1;
  // 字段名可以正常使用
  string name = 2;
}

// build
protoc --go_out=. --go_opt=paths=source_relative ./pb/*.proto
pb/msg.proto:8:10: Field name "name" is reserved.
pb/msg.proto: Field "id" uses reserved number 11.
pb/msg.proto: Suggested field numbers for git.test.demo.DemoMsg: 4

```

end

#### enum

```go
syntax = "proto3";
package demo;
// 声明生成Go代码，包路径
option go_package ="server/demo";
// 枚举消息
message DemoEnumMsg {
  enum Gender{
    // 枚举字段标识符,必须从0开始
    UnKnown = 0;
    Body = 1;
    Girl = 2;
  }
  // 使用自定义的枚举类型
  Gender Chandler = 2;
}
// 在枚举信息中，重复使用标识符
message DemoTwoMsg{
  enum Animal {
    // 开启允许重复使用 标示符
    option allow_alias = true;
    Other = 0;
    Cat = 1;
    Dog = 2;
    // 白猫也是猫，标示符也用1
    // 不开启allow_alias，会报错： Enum value number 1 has already been used by value 'Cat'
    WhiteCat = 1;
  }
}
```

每个枚举类型必须将其第一个类型映射为0, 原因有两个：1.必须有个默认值为0； 2.为了兼容proto2语法，枚举类的第一个值总是默认值.

#### Oneof

```go
// 定义入参消息
message HelloParam{
  string name = 1;
  string context = 2;
  // oneof 最多只能设置其中一个字段
  oneof option {
    int32 age= 3;
    string gender= 4;
  }
}
```

实际应用,生成`Go`代码后，入参只能设置其中一个值，如下

```go
// 实例化客户端
	client := server.NewHelloServiceClient(dial)
	// 定义参数
	reqParam := &server.HelloParam{
		Name:    "Unicorn",
		Context: "hello word!",
	}
	// 只能设置其中一个
	reqParam.Option = &server.HelloParam_Age{Age: 19}
	// 这个会替代上一个值
	//reqParam.Option = &server.HelloParam_Gender{Gender: "man"}
	// 发起请求
	result, err := client.SayHello(context.TODO(), reqParam)
```

end

#### 嵌套message

```go
syntax = "proto3";
option go_package = "server/nested";
// 学员信息
message UserInfo {
  int32 userId = 1;
  string userName = 2;
}
message Common {
  // 班级信息
  message CLassInfo{
    int32 classId = 1;
    string className = 2;
  }
}
// 嵌套信息
message NestedDemoMsg {
  // 学员信息 (直接使用消息类型)
  UserInfo userInfo = 1;
  // 班级信息 (通过Parent.Type，调某个消息类型的子类型)
  Common.CLassInfo classInfo =2;
}
```

end

#### map

```go
syntax = "proto3";
option go_package = "server/demo";

// map消息
message DemoMapMsg {
  int32 userId = 1;
  map<string,string> like =2;
}
```

protoc 编译后生成的go代码

```go
// map消息
type DemoMapMsg struct {
	state         protoimpl.MessageState
	sizeCache     protoimpl.SizeCache
	unknownFields protoimpl.UnknownFields

	UserId int32             `protobuf:"varint,1,opt,name=userId,proto3" json:"userId,omitempty"`
	Like   map[string]string `protobuf:"bytes,2,rep,name=like,proto3" json:"like,omitempty" protobuf_key:"bytes,1,opt,name=key,proto3" protobuf_val:"bytes,2,opt,name=value,proto3"`
}
```

#### 切片slice

```go
syntax = "proto3";
option go_package = "server/demo";

// repeated允许字段重复，对于Go语言来说，它会编译成数组(slice of type)类型的格式
message DemoSliceMsg {
  // 会生成 []int32
  repeated int32 id = 1;
  // 会生成 []string
  repeated string name = 2;
  // 会生成 []float32
  repeated float price = 3;
  // 会生成 []float64
  repeated double money = 4;
}
```

生成go代码

```go
// repeated允许字段重复，对于Go语言来说，它会编译成数组(slice of type)类型的格式
type DemoSliceMsg struct {
	state         protoimpl.MessageState
	sizeCache     protoimpl.SizeCache
	unknownFields protoimpl.UnknownFields

	// 会生成 []int32
	Id []int32 `protobuf:"varint,1,rep,packed,name=id,proto3" json:"id,omitempty"`
	// 会生成 []string
	Name []string `protobuf:"bytes,2,rep,name=name,proto3" json:"name,omitempty"`
	// 会生成 []float32
	Price []float32 `protobuf:"fixed32,3,rep,packed,name=price,proto3" json:"price,omitempty"`
	Money []float64 `protobuf:"fixed64,4,rep,packed,name=money,proto3" json:"money,omitempty"`
}
```

end

### service



#### 定义gRPC service

```go
syntax = "proto3";

option go_package = "grpc/server";

// 定义入参消息
message HelloParam{
  string name = 1;
  string context = 2;
}

// 定义出参消息
message HelloResult {
  string result = 1;
}

// 定义service
service HelloService{
  // 定义方法 
  rpc SayHello(HelloParam) returns (HelloResult);
}
```

编译后生成的go代码

```go 
// HelloServiceClient is the client API for HelloService service.
type HelloServiceClient interface {
	// 定义方法
	SayHello(ctx context.Context, in *HelloParam, opts ...grpc.CallOption) (*HelloResult, error)
}

type helloServiceClient struct {
	cc grpc.ClientConnInterface
}

....
// HelloServiceServer is the server API for HelloService service.
// All implementations must embed UnimplementedHelloServiceServer
// for forward compatibility
type HelloServiceServer interface {
	// 定义方法
	SayHello(context.Context, *HelloParam) (*HelloResult, error)
	mustEmbedUnimplementedHelloServiceServer()
}
// UnimplementedHelloServiceServer must be embedded to have forward compatible implementations.
type UnimplementedHelloServiceServer struct {
}

...
```

end

#### 修改server实现接口

下面是编译后的`SayHello method`并没有具体的代码实现，go调用时会出错，所以需要修改。

```go
func (UnimplementedHelloServiceServer) SayHello(context.Context, *HelloParam) (*HelloResult, error) {
	return nil, status.Errorf(codes.Unimplemented, "method SayHello not implemented")
}

// 添加形参名和详细的实现逻辑
func (UnimplementedHelloServiceServer) SayHello(ctx context.Context, p *HelloParam) (*HelloResult, error) {
	return &HelloResult{Result: fmt.Sprintf("%s say %s",p.GetName(),p.GetContext())},nil
}
```

end





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

#### gRPC service编译

如果想要将消息类型用在RPC(远程方法调用)系统中，需要使用关键字(`service`)定义一个RPC服务接口，使用`rpc`定义具体方法，而消息类型则充当方法的参数和返回值。

然后通过`protoc --go-grpc_out`编译输出grpc代码

```bash
$ protoc --go_out=. --go_opt=paths=source_relative --go-grpc_out=./ --go-grpc_opt=paths=source_relative ./pb/*.proto

```



### gRPC demo

gRPC是一个高性能、开源、通用的`RPC`框架，由`Google`推出，基于HTTP2协议标准设计开发，默认采用Protocol Buffers数据序列化协议，支持多种开发语言。`gRPC`提供了一种简单的方法来精确的定义服务，并且为客户端和服务端自动生成可靠的功能库。

最底层为`TCP`或`Unix`套接字协议，在此之上是`HTTP/2`协议的实现，然后在`HTTP/2`协议之上又构建了针对`Go`语言的`gRPC`核心库（`gRPC`内核+解释器）。应用程序通过`gRPC`插件生成的`Stub`代码和`gRPC`核心库通信，也可以直接和`gRPC`核心库通信。

```go
syntax = "proto3";

option go_package = "./;pb";

// 定义入参消息
message HelloParam{
  string name = 1;
  string context = 2;
}

// 定义出参消息
message HelloResult {
  string result = 1;
}

// 定义service
service HelloService{
  // 定义方法
  rpc SayHello(HelloParam) returns (HelloResult);
}

```

`protoc`工具编译proto文件，并修改`SayHello()`实现

```go
...
// 实现SayHello接口
func (UnimplementedHelloServiceServer) SayHello(_ context.Context, p *HelloParam) (*HelloResult, error) {
	return &HelloResult{Result: fmt.Sprintf("%s say %s",p.GetName(),p.GetContext())},nil
}
...
```

client and server code

```go
// server.go
package main

import (
	"fmt"
	"google.golang.org/grpc"
	"log"
	"net"
	"test/pb"
)

func main()  {

	rpcServer := grpc.NewServer()
	pb.RegisterHelloServiceServer(rpcServer, new(pb.UnimplementedHelloServiceServer))
	listen, err := net.Listen("tcp", ":8083")
	if err != nil{
		log.Fatalln("server is error", err)
	}
	fmt.Println("r")
	rpcServer.Serve(listen)
}

// client.go
package main

import (
	"context"
	"fmt"
	"google.golang.org/grpc"
	"test/pb"
)

// 客户端代码
func main() {
	// 建立链接
	dial, err := grpc.Dial("127.0.0.1:8083", grpc.WithInsecure())
	if err != nil {
		fmt.Println("Dial Error ", err)
		return
	}
	// 延迟关闭链接
	defer dial.Close()
	// 实例化客户端
	client := pb.NewHelloServiceClient(dial)
	// 发起请求
	result, err := client.SayHello(context.TODO(), &pb.HelloParam{
		Name:    "Chandler",
		Context: "hello word!",
	})
	if err != nil {
		fmt.Println("请求失败:", err)
		return
	}
	// 打印返回信息
	fmt.Printf("%+v\n", result)
}
```

end

### 引用

1. https://studygolang.com/articles/25743
2. https://segmentfault.com/a/1190000039767770
3. https://www.cnblogs.com/shijingxiang/articles/14370775.html
3. http://liuqh.icu/2022/01/20/go/rpc/03-grpc-ru-men/
