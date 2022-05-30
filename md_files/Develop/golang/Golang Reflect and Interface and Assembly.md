## Golang Reflect and  Interface

### interface

#### Nameless params

There is no way to get the names of the parameters of a method or a function.

The reason for this is because the names are not really important for someone calling a method or a function. What matters is the types of the parameters and their order.

A Function type denotes the set of all functions with the same parameter and result types. The type of 2 functions having the same parameter and result types is identical regardless of the names of the parameters. The following code prints true:

```go
func f1(a int) {}

func f2(b int) {}

func NamelessParams(int, string) {

	fmt.Println("NamelessParams called")

}

func main()  {
	fmt.Println(reflect.TypeOf(f1) == reflect.TypeOf(f2))
	NamelessParams(1, "a")
}

//output
true
NamelessParams called
```

参数名称对于调用方法或函数的人来说并不重要，重要的是参数的类型及其顺序。

因为只有函数内部才会使用这个函数的参数去执行响应的逻辑。

什么情况下可能会用到 nameless parameters呢

##### 接口声明

如下protoc编译的grpc代码中接口声明的method就使用了nameless parameters

```go
type HelloServiceServer interface {
	// 定义方法
	SayHello(context.Context, *HelloParam) (*HelloResult, error)
	mustEmbedUnimplementedHelloServiceServer()
}

type UnimplementedHelloServiceServer struct {
}

func (UnimplementedHelloServiceServer) SayHello(context.Context, *HelloParam) (*HelloResult, error) {
	return nil, status.Errorf(codes.Unimplemented, "method SayHello not implemented")
}
```

end

##### 兼容性

你写了一个框架/库，如果你要修改或者扩展一个函数的参数列表，必然会破坏版本的兼容性，因为golang不支持函数的重载。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/golang-method-name.jpg)

所以，写框架或者库的时候应该有远见的声明一些带有其他不同类型的参数方法，但是你可能还没有想好变量名，只是确定了方法可能用到的参数类型，这时就会用到Nameless parameters

```go
// HelloServiceServer is the server API for HelloService service.
type HelloServiceServer interface {
    // nameless parameters
	SayHello(context.Context, *HelloParam) (*HelloResult, error)
	mustEmbedUnimplementedHelloServiceServer()
}
```

end

##### parameters named as `_`

方法或函数声明时，要么使用nameless parameters，要么使用named parameters，不能混合使用。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/golang-name-and-unnamed.jpg)

对于一些不得不命名但又不使用的参数名可以使用`_`来代替。

```go
func (UnimplementedHelloServiceServer) SayHello(_ context.Context, p *HelloParam) (*HelloResult, error) {
	return &HelloResult{Result: fmt.Sprintf("%s say %s",p.GetName(),p.GetContext())},nil
}
```

Unused variables are always a programming error, whereas it is common to write a function that doesn't use all of its arguments.

One could leave those arguments unnamed (using _), but then that might confuse with functions like

func foo(_ string, _ int) // what's this supposed to do?

The names, even if they're unused, provide important documentation.





#### Interface value详解

**从概念上来讲**，interface value 有两部分组成：type 部分是一个 concrete type，vlaue 部分是这个 concrete type 对应的 instance，它们分别称之为 interface value 的 dynamic type 和 dynamic value。

> concrete type , 用户自定义的，eg:`Struct` 、`type Intger int`等 ，因为static type不能绑定Methods
>
> eg，`func (a int)test()` 报错--
>
> --`cannot define new methods on non-local type int`

由于 Go 是静态类型的语言，type 是在编译阶段已经定义好的，而 interface 存储的值是动态的，在上面这个概念模型中，type 部分更准确叫法是 type descriptors，主要是提供 concrete type 的相关信息，包括 method、name 等。

下面这几个语句：

```go
// w is a interface
var w io.Writer
w = os.Stdout
w = new(bytes.Buffer)
w = nil
```

 变量 **w** 依次存储了三种不同的值，在此我们依次来看看每种不同的值的确切含义。 

 语句 **var w io.Writer** 声明并初始化了一个 interface value w，其值是 **nil**，此时 type 和 value 部分都是 nil。 

```go
w:
    type --> nil
    value --> nil
```

 interface value 是否是 nil 取决于其 dynamic type，在 nil 的 interface value 上调用会 panic 

```go
// panic
w.Write([]byte("hello")) 
```

 语句 **w = os.Stdout** 赋值 `*os.File` 类型的 value 给 w，这个赋值操作包含一个隐式的类型转换，用以把 concrete type 转换成 **interface type io.Writer(\*os.File)**，在这个转换过程中 dynamic type 被赋值为 *os.File 类型，在这里其实是它的 type descriptor，同样得，dynamic value 赋值为 **os.Stdout** 的一份 copy，一个指向 os.File 类型的指针且代表标准输出的变量。 

```go
Stdout = NewFile(uintptr(syscall.Stdout), "/dev/stdout")
func NewFile(fd uintptr, name string) *File {}

w:
    type --> *os.File
    value -------> fd int=1(stdout)
```

 在 interface value w 上调用 Write method 实际上调用的是 *os.File 类型的 Write 方法，于是输出 "hello"。 

```go
// Write writes len(b) bytes to the File.
func (f *File) Write(b []byte) (n int, err error) {}

w.Write([]byte("hello")) // "hello"
```

 由于在编译阶段，我们并不知道一个 interface value 的 dynamic type 是什么，所以 interface value 的调用必须进行 dynamic dispatch。为了能调用 dynamic value 的 Write method，compiler 必须生成相关代码以便在执行的时候通过 dynamic type 获取对应 method 的真实地址的 copy，在调用的形式上好像是我们直接调用了 dynamic value 的 Write method。 

```go
os.Stdout.Write([]byte("hello")) // "hello"
```

语句 w=new(bytes.Buffer) 赋值 *bytes.Buffer 类型的 value 作为 w 的 dynamic value，对 w 的处理是也类似的，调用 Write method 将调用 *bytes.Buffer 的 Write method。

语句 w=nil 和初始语句一样将 w 重置为 nil。

格式化输出interface value

```go
var w io.Writer
fmt.Printf("%T\n", w) // "<nil>"
w=os.Stdout
fmt.Printf("%T\n", w) // "*os.File"

w=new(bytes.Buffer)
fmt.Printf("%T\n", w) // "*bytes.Buffer"
```

##### nil Interface and nil value:thinking:

> dynamic value 是 nil 的 interface value 可能并不是 nil，因为dynamic type可能不是nil。

 Interface value （type和value都是nil）是 nil 和 interface value 包含的 dynamic value 是 nil 并不是一回事，后者不是 nil.

Interface value 是nil的意思是:  (concrete value, type descriptor)都是nil.

type descriptor指向 dynamic type，concrete value 和 dynamic value一样，即实现接口的dynamic type instance的值和Interface instance（a pair）的 值是不一样的。

```go
package main

import (
	"bytes"
	"io"
    "fmt"
    "reflect"
	
)
const debug = false

func main() {
    // buf 是一个值为nil的结构体指针
    var buf *bytes.Buffer
    if debug {
        buf = new(bytes.Buffer) // enable collection of output
    }
    if buf == nil {
        f(buf) // NOTE: subtly incorrect!
    }
    if debug {
        // ...use buf...
    }
}

// If out is non-nil, output will be written to it.

func f(out io.Writer) {
    // ...do something...
    // 这里的 out 是 (*bytes.Buffer)(nil)的接口类型数据
    // 因为out的Type不为nil 所以，这里 out != nil 成立
    if out != nil {
        out.Write([]byte("done!\n"))
    }
}
--- ----
第一个if buf == nil，是判断变量的值是不是nil
第二个if out != nil, 是判断interface是不是nil interface

// Writer是一个接口...
type Writer interface {
	Write(p []byte) (n int, err error)
}

	var w io.Writer
	var buf *bytes.Buffer
	fmt.Printf("%#v\n", w)
	w = buf
	fmt.Printf("%#v\n", w)
// output
<nil>
(*bytes.Buffer)(nil)

// buffer.go
func (b *Buffer) Write(p []byte) (n int, err error){}


	var w io.Writer
	var buf io.Writer
	fmt.Printf("%#v\n", w)
	w = buf
	fmt.Printf("%#v\n", w)
	w = os.Stdout
// output
<nil>
<nil>
```

 如果执行这段程序，回报错，我们来稍微分析一下，在`main()`中，我们首先声明了一个`buf`变量，类型是`*bytes.Buffer`指针类型，它的值为`nil`（buf == nil is true），在调用函数`f`的时候，参数`out`会被赋值为动态类型为`*bytes.Buffer`，动态值为`nil`，也就是`out`是一个包含了空指针值的非空接口。那么在`out != nil`判断时，这个等式便是成立的 、但是调用 Write 方法时，Write 的 receiver 也就是 out 的 **dynamic value** 是 nil，因而会 panic。

总结：

1. buf 是 *bytes.Buffer，它是实例实现Write() method，所以实现了io.Writer interface
2. out io.Writer 作为形参隐性赋值即out = buf
3. 由于buf和out不是相同类型，所以out能触发简单的值复制，这导致out的dynamic value是nil
4. nil.Write()就panic了 panic: runtime error: invalid memory address or nil pointer dereference 

**解决方法**是改变 main 中的 buf 为 io.Writer。 `var buf io.Writer`

由于变量buf和形参out，都是一个类型的，直接触发最简单的值复制，所以可以调用`Write()`

```go
func main() {
	var buf io.Writer
	var out io.Writer
	buf = new(bytes.Buffer)

	fmt.Printf("%#v\n", buf)
	out = buf
	fmt.Printf("%#v\n", out)

}
// output
&bytes.Buffer{buf:[]uint8(nil), off:0, lastRead:0}
&bytes.Buffer{buf:[]uint8(nil), off:0, lastRead:0}

```

end



通过汇编可以定位到interface对应的结构体是：

* iface
* eface

 interface类型的变量存储了一个元组(a pair): 一是赋值给这个变量的具体值(concrete value),一是那个值的类型描述符(type descriptor)。

concrete value就是dynamic value 

 type descriptor执行dynamic的Type-

>  Interface instance的nil是a pair，即(nil, nil) 、而其他Type的nil，是指它的value是nil。
>

```go
// io.Reader是一个接口--
func main() {

	var r io.Reader

	tty, _ := os.OpenFile("/dev/tty", os.O_RDWR, 0)
	
	r = tty

	fmt.Printf("%#v", r)
}
--- output ---
(*os.File)(nil)

```

接口变量 `r`包含一个(value, type)对，那就是`(tty, *os.File)`。 

这个Type指针指向， concrete type 。

#### interface 数据结构

接口是高级语言中的一个规约，是一组方法签名的集合。Go 的 interface 是非侵入式的，具体类型实现 interface 不需要在语法上显式的声明，只需要具体类型的方法集合是 interface 方法集合的超集，就表示该类实现了这一 interface。编译器在编译时会进行 interface 校验。interface 和具体类型不同，它不能实现具体逻辑，也不能定义字段。

在 Go 语言中，interface 和函数一样，都是“第一公民”。interface 可以用在任何使用变量的地方。可以作为结构体内的字段，可以作为函数的形参和返回值，可以作为其他 interface 定义的内嵌字段。interface 在大型项目中常常用来解耦。在层与层之间用 interface 进行抽象和解耦。由于 Go interface 非侵入的设计，使得抽象出来的代码特别简洁，这也符合 Go 语言设计之初的哲学。

* Interface 是一个定义了方法签名的集合,用来指定对象的行为，如果对象做到了 Interface 中方法集定义的行为，那就可以说实现了 Interface；
* 这些方法可以在不同的地方被不同的对象实现，这些实现可以具有不同的行为；即多态？？？
* interface 的主要工作仅是提供方法名称签名,输入参数,返回类型。最终由具体的对象来实现方法，比如 struct或`type A int`；

使用 type 关键字来申明，interface 代表类型，大括号里面定义接口的方法签名集合。 

```go
type Animal interface {
	Bark() string
	Walk() string
}
```

 Dog 实现了 Animal 接口，所以可以用 Animal 的实例去接收 Dog的实例，必须是同时实现 Bark() 和Walk() 方法，否则都不能算实现了Animal接口。 

```go
type Dog struct {
	name string
}

func (dog Dog) Bark() string {
	fmt.Println(dog.name + ":wang wang"!")
	return "wang wang"
}

func (dog Dog) Walk() string {
	fmt.Println(dog.name + ":walk to park!")
	return "walk to park"
}

func main() {
	var animal Animal

	fmt.Println("animal value is:", animal)	//animal value is: <nil>
	fmt.Printf("animal type is: %T\n", animal) //animal type is: <nil>
    // 这里就是多态了，其他的animals也定义实现了这个接口，就可以放一块
    // eg: animals := []Animal{Dog, Kitten, Horse}
	animal = Dog{"旺财"}
	animal.Bark() //旺财:wan wan wan!
	animal.Walk() //旺财:walk to park!

	fmt.Println("animal value is:", animal) //animal value is: {旺财}
	fmt.Printf("animal type is: %T\n", animal) //animal type is: main.Dog
}

```

ene

##### iface:heavy_check_mark:

非空的 interface 初始化的底层数据结构是 iface

```go
type iface struct {
	tab  *itab
	data unsafe.Pointer
}
```

tab 中存放的是类型、方法等信息。data 指针指向的 iface 绑定对象的原始数据的副本。这里同样遵循 Go 的统一规则，值传递。tab 是 itab 类型的指针。

```go
// layout of Itab known to compilers
// allocated in non-garbage-collected memory
// Needs to be in sync with
// ../cmd/compile/internal/gc/reflect.go:/^func.WriteTabs.
type itab struct {
	inter *interfacetype
	_type *_type
	hash  uint32 // copy of _type.hash. Used for type switches.
	_     [4]byte
	fun   [1]uintptr // variable sized. fun[0]==0 means _type does not implement inter.
}

type interfacetype struct {
	typ     _type
	pkgpath name
	mhdr    []imethod
}
```

itab 中包含 5 个字段。inner 存的是 interface 自己的静态类型。`_type` 存的是 interface 对应具体对象的类型。itab 中的 `_type `和 iface 中的 data 能简要描述一个变量。`_type `是这个变量对应的类型，data 是这个变量的值。这里的 hash 字段和 _type 中存的 hash 字段是完全一致的，这么做的目的是为了类型断言(下文会提到)。fun 是一个函数指针，它指向的是具体类型的函数方法。虽然这里只有一个函数指针，但是它可以调用很多方法。在这个指针对应内存地址的后面依次存储了多个方法，利用指针偏移便可以找到它们。

由于 Go 语言是强类型语言，编译时对每个变量的类型信息做强校验，所以每个类型的元信息要用一个结构体描述。再者 Go 的反射也是基于类型的元信息实现的。_type 就是所有类型最原始的元信息。

```go
// Needs to be in sync with ../cmd/link/internal/ld/decodesym.go:/^func.commonsize,
// ../cmd/compile/internal/gc/reflect.go:/^func.dcommontype and
// ../reflect/type.go:/^type.rtype.
// ../internal/reflectlite/type.go:/^type.rtype.
type _type struct {
	size       uintptr // 类型占用内存大小
	ptrdata    uintptr // 包含所有指针的内存前缀大小
	hash       uint32  // 类型 hash
	tflag      tflag   // 标记位，主要用于反射
	align      uint8   // 对齐字节信息
	fieldAlign uint8   // 当前结构字段的对齐字节数
	kind       uint8   // 基础类型枚举值
	equal func(unsafe.Pointer, unsafe.Pointer) bool // 比较两个形参对应对象的类型是否相等
	gcdata    *byte    // GC 类型的数据
	str       nameOff  // 类型名称字符串在二进制文件段中的偏移量
	ptrToThis typeOff  // 类型元信息指针在二进制文件段中的偏移量
}
```

有 3 个字段需要解释一下：

- kind，这个字段描述的是如何解析基础类型。在 Go 语言中，基础类型是一个枚举常量，有 26 个基础类型，如下。枚举值通过 kindMask 取出特殊标记位。

  ```go
  // typekind.go
  package runtime
  
  const (
  	kindBool = 1 + iota
  	kindInt
  	kindInt8
  	kindInt16
  	kindInt32
  	kindInt64
  	kindUint
  	kindUint8
  	kindUint16
  	kindUint32
  	kindUint64
  	kindUintptr
  	kindFloat32
  	kindFloat64
  	kindComplex64
  	kindComplex128
  	kindArray
  	kindChan
  	kindFunc
  	kindInterface
  	kindMap
  	kindPtr
  	kindSlice
  	kindString
  	kindStruct
  	kindUnsafePointer
  
  	kindDirectIface = 1 << 5
  	kindGCProg      = 1 << 6
  	kindMask        = (1 << 5) - 1
  )
  
  ```

  end

- str 和 ptrToThis，对应的类型是 nameoff 和 typeOff。这两个字段的值是在链接器段合并和符号重定向的时候赋值的。
  ![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/iface-1.png)
  链接器将各个 .o 文件中的段合并到输出文件，会进行段合并，有的放入 .text 段，有的放入 .data 段，有的放入 .bss 段。name 和 type 针对最终输出文件所在段内的偏移量 offset 是由 resolveNameOff 和 resolveTypeOff 函数计算出来的，然后链接器把结果保存在 str 和 ptrToThis 中。具体逻辑可以见源码中下面 2 个函数:

  ```go
  func resolveNameOff(ptrInModule unsafe.Pointer, off nameOff) name {}  
  func resolveTypeOff(ptrInModule unsafe.Pointer, off typeOff) *_type {}
  ```

  回到 _type 类型。上文谈到 _type 是所有类型原始信息的元信息。例如：

  ```go
  type arraytype struct {
  	typ   _type
  	elem  *_type
  	slice *_type
  	len   uintptr
  }
  
  type chantype struct {
  	typ  _type
  	elem *_type
  	dir  uintptr
  }
  ```

  在 arraytype 和 chantype 中保存类型的元信息就是靠 _type。同样 interface 也有类似的类型定义：

  ```go
  type imethod struct {
  	name nameOff
  	ityp typeOff
  }
  
  type interfacetype struct {
  	typ     _type     // 类型元信息
  	pkgpath name      // 包路径和描述信息等等
  	mhdr    []imethod // 方法
  }
  ```

  因为 Go 语言中函数方法是以包为单位隔离的。所以 interfacetype 除了保存 _type 还需要保存包路径等描述信息。mhdr 存的是各个 interface 函数方法在段内的偏移值 offset，知道偏移值以后才方便调用。

- end

##### eface

空的 inferface{} 是没有方法集的接口。所以不需要 itab 数据结构。它只需要存类型和类型对应的值即可。对应的数据结构如下：

```go
type eface struct {
	_type *_type
	data  unsafe.Pointer
}
```

从这个数据结构可以看出，只有当 2 个字段都为 nil，空接口才为 nil。空接口的主要目的有 2 个，一是实现“泛型”，二是使用反射。

#### interface类型转换，汇编:articulated_lorry:

##### 指针类型转换成interface

汇编解析展示

```go
package main

import "fmt"

func main() {
	var s Person = &Student{name: "halfrost"}
	s.sayHello("everyone")
}

type Person interface {
	sayHello(name string) string
	sayGoodbye(name string) string
}

type Student struct {
	name string
}

//go:noinline
func (s *Student) sayHello(name string) string {
	return fmt.Sprintf("%v: Hello %v, nice to meet you.\n", s.name, name)
}

//go:noinline
func (s *Student) sayGoodbye(name string) string {
	return fmt.Sprintf("%v: Hi %v, see you next time.\n", s.name, name)
}
```

利用 go build 和 go tool 命令将上述代码变成汇编代码：

```bash
$ go tool compile -S -N -l main.go >main.s1 2>&1
```

main 方法中有 3 个操作，重点关注后 2 个涉及到 interface 的操作：

1. 初始化 Student 对象指针
2. 将 Student 对象指针转换成 interface
3. 调用 interface 的方法

> Plan9 汇编常见寄存器含义：
> BP: 栈基，栈帧（函数的栈叫栈帧）的开始位置。
> SP: 栈顶，栈帧的结束位置。
> PC: 就是IP寄存器，存放CPU下一个执行指令的位置地址。
> TLS: 虚拟寄存器。表示的是 thread-local storage，Golang 中存放了当前正在执行的g的结构体。

先来看 Student 初始化的汇编代码：

```go
0x0021 00033 (main.go:6)	LEAQ	type."".Student(SB), AX      // 将 type."".Student 地址放入 AX 中
0x0028 00040 (main.go:6)	MOVQ	AX, (SP)                     // 将 AX 中的值存储在 SP 中
0x002c 00044 (main.go:6)	PCDATA	$1, $0
0x002c 00044 (main.go:6)	CALL	runtime.newobject(SB)        // 调用 runtime.newobject() 方法，生成 Student 对象存入 SB 中
0x0031 00049 (main.go:6)	MOVQ	8(SP), DI                    // 将生成的 Student 对象放入 DI 中
0x0036 00054 (main.go:6)	MOVQ	DI, ""..autotmp_2+40(SP)     // 编译器认为 Student 是临时变量，所以将 DI 放在栈上
0x003b 00059 (main.go:6)	MOVQ	$8, 8(DI)                    // (DI.Name).Len = 8
0x0043 00067 (main.go:6)	PCDATA	$0, $-2
0x0043 00067 (main.go:6)	CMPL	runtime.writeBarrier(SB), $0
0x004a 00074 (main.go:6)	JEQ	78
0x004c 00076 (main.go:6)	JMP	172
0x004e 00078 (main.go:6)	LEAQ	go.string."halfrost"(SB), AX  // 将 "halfrost" 字符串的地址放入 AX 中
0x0055 00085 (main.go:6)	MOVQ	AX, (DI)                      // (DI.Name).Data = &"halfrost"
0x0058 00088 (main.go:6)	JMP	90
0x005a 00090 (main.go:6)	PCDATA	$0, $-1
```

先将 *_type 放在 (SP) 栈顶。然后调用 runtime.newobject() 生成 Student 对象。(SP) 栈顶的值即是 newobject() 方法的入参。

```go
func newobject(typ *_type) unsafe.Pointer {
	return mallocgc(typ.size, typ, true)
}
```

PCDATA 用于生成 PC 表格，PCDATA 的指令用法为：PCDATA tableid, tableoffset。PCDATA有个两个参数，第一个参数为表格的类型，第二个是表格的地址。runtime.writeBarrier() 是 GC 相关的方法，感兴趣的可以研究它的源码。以下是 Student 对象临时对象 GC 的一些汇编代码逻辑，由于有 JMP 命令，代码是分开的，笔者在这里将它们汇集在一起。

```go
0x0043 00067 (main.go:6)    PCDATA  $0, $-2
0x0043 00067 (main.go:6)    CMPL    runtime.writeBarrier(SB), $0
0x004a 00074 (main.go:6)    JEQ 78
0x004c 00076 (main.go:6)    JMP 172
......
0x00ac 00172 (main.go:6)	PCDATA	$0, $-2
0x00ac 00172 (main.go:6)	LEAQ	go.string."halfrost"(SB), AX
0x00b3 00179 (main.go:6)	CALL	runtime.gcWriteBarrier(SB)
0x00b8 00184 (main.go:6)	JMP	90
0x00ba 00186 (main.go:6)	NOP
```

78 对应的十六进制是 0x004e，172 对应的十六进制是 0x00ac。先对比 runtime.writeBarrier(SB) 和 $0 存的是否一致，如果相同则 JMP 到 0x004e 行，如果不同则 JMP 到 0x00ac 行。0x004e 行和 0x00ac 行代码完全相同，都是将字符串 "halfrost" 的地址放入 AX 中，不过 0x00ac 行执行完会紧接着调用 runtime.gcWriteBarrier(SB)。执行完成以后再回到 0x005a 行。

第一步结束以后，内存中存了 3 个值。临时变量 .autotmp_2 放在 +40(SP) 的地址处，它也就是临时 Student 对象。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/interface-1.png)

接下来是第二步，将 Student 对象转换成 interface。

```go
0x005a 00090 (main.go:6)	MOVQ	""..autotmp_2+40(SP), AX
0x005f 00095 (main.go:6)	MOVQ	AX, ""..autotmp_1+48(SP)
0x0064 00100 (main.go:6)	LEAQ	go.itab.*"".Student,"".Person(SB), CX
0x006b 00107 (main.go:6)	MOVQ	CX, "".s+56(SP)
0x0070 00112 (main.go:6)	MOVQ	AX, "".s+64(SP)
```

经过上面几行汇编代码，成功的构造出了 itab 结构体。在汇编代码中可以找到 itab 的内存布局：

```go
go.itab.*"".Student,"".Person SRODATA dupok size=40
	0x0000 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0010 0c 31 79 12 00 00 00 00 00 00 00 00 00 00 00 00  .1y.............
	0x0020 00 00 00 00 00 00 00 00                          ........
	rel 0+8 t=1 type."".Person+0
	rel 8+8 t=1 type.*"".Student+0
	rel 24+8 t=1 "".(*Student).sayGoodbye+0
	rel 32+8 t=1 "".(*Student).sayHello+0
```

itab 结构体的首字节里面存的是 inter *interfacetype，此处即 Person interface。第二个字节中存的是 *_type，这里是第一步生成的，放在 (SP) 地址处的地址。第四个字节中存的是 fun [1]uintptr，对应 sayGoodbye 方法的首地址。第五个字节中存的也是 fun [1]uintptr，对应 sayHello 方法的首地址。回顾上一章节里面的 itab 数据结构：

```go
type itab struct {
    inter *interfacetype // 8 字节
    _type *_type         // 8 字节
    hash  uint32 		 // 4 字节，填充使得内存对齐
    _     [4]byte        // 4 字节
    fun   [1]uintptr     // 8 字节
}
```

现在就很明确了为什么 fun 只需要存一个函数指针。每个函数指针都是 8 个字节，如果 interface 里面包含多个函数，只需要 fun 往后顺序偏移多个字节即可。第二步结束以后，内存中存储了以下这些值：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/interface-2.png)

在第二步中新建了一个临时变量 .autotmp_1 放在 +48(SP) 地址处。并且利用第一步中生成的 Student 临时变量构造出了 *itab。值得说明的是，虽然汇编代码并没有显示调用函数生成 iface，但是此时已经生成了 iface。

```go
type iface struct {
    tab  *itab
    data unsafe.Pointer
}
```

如上图，+56(SP) 处存的是 *itab，+64(SP) 处存的是 unsafe.Pointer，这里的指针和 +8(SP) 的指针是完全一致的。接下来就是最后一步，调用 interface 的方法。

```go
0x0075 00117 (main.go:7)	MOVQ	"".s+56(SP), AX
0x007a 00122 (main.go:7)	TESTB	AL, (AX)
0x007c 00124 (main.go:7)	MOVQ	32(AX), AX
0x0080 00128 (main.go:7)	MOVQ	"".s+64(SP), CX
0x0085 00133 (main.go:7)	MOVQ	CX, (SP)
0x0089 00137 (main.go:7)	LEAQ	go.string."everyone"(SB), CX
0x0090 00144 (main.go:7)	MOVQ	CX, 8(SP)
0x0095 00149 (main.go:7)	MOVQ	$8, 16(SP)
0x009e 00158 (main.go:7)	NOP
0x00a0 00160 (main.go:7)	CALL	AX
```

先取出调用方法的真正对象，放入 (SP) 中，再依次将方法中的入参按照顺序放在 +8(SP) 之后。然后调用函数指针对应的方法。从汇编代码中可以看到，AX 直接从取出了 *itab 指针存的内存地址，然后偏移到了 +32 的位置，这里是要调用的方法 sayHello 的内存地址。最后从栈顶依次取出需要的参数，即算完成 iterface 方法调用。方法调用前一刻，内存中的状态如下，主要关注 AX 的地址以及栈顶的所有参数信息。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/interface-3.png)

栈顶依次存放的是方法的调用者，参数。调用格式可以表示为 func(reciver, param1)。

##### 非指针类型转换成interface

指针类型和结构体类型在类型转换中会有哪些区别？测试代码和指针类型大体一致，只是类型转换的时候换成了结构体，方法实现也换成了结构体，其他都没有变。

```go
package main

import "fmt"

func main() {
	var s Person = Student{name: "halfrost"}
	s.sayHello("everyone")
}

type Person interface {
	sayHello(name string) string
	sayGoodbye(name string) string
}

type Student struct {
	name string
}

//go:noinline
func (s Student) sayHello(name string) string {
	return fmt.Sprintf("%v: Hello %v, nice to meet you.\n", s.name, name)
}

//go:noinline
func (s Student) sayGoodbye(name string) string {
	return fmt.Sprintf("%v: Hi %v, see you next time.\n", s.name, name)
}
```

用同样的命令生成对应的汇编代码：

```bash
$ go tool compile -S -N -l main.go >main.s2 2>&1
```

对比相同的 3 个环节：

1. 初始化 Student 对象
2. 将 Student 对象转换成 interface
3. 调用 interface 的方法

```go
0x0021 00033 (main.go:6)	XORPS	X0, X0                       // X0 置 0
0x0024 00036 (main.go:6)	MOVUPS	X0, ""..autotmp_1+64(SP)     // 清空 +64(SP)
0x0029 00041 (main.go:6)	LEAQ	go.string."halfrost"(SB), AX
0x0030 00048 (main.go:6)	MOVQ	AX, ""..autotmp_1+64(SP)
0x0035 00053 (main.go:6)	MOVQ	$8, ""..autotmp_1+72(SP)
0x003e 00062 (main.go:6)	MOVQ	AX, (SP)
0x0042 00066 (main.go:6)	MOVQ	$8, 8(SP)
0x004b 00075 (main.go:6)	PCDATA	$1, $0
```

这段代码将 "halfrost" 放入内存相应的位置。上述代码 1-8 行，将字符串 "halfrost" 地址和长度 8 拷贝到 +0(SP)，+8(SP) 和 +64(SP)，+72(SP) 中。从这里可以了解到普通的临时变量在内存中布局是怎么样的。从上述汇编代码中可以看出，编译器发现这个变量只是临时变量，都没有调用 runtime.newobject()，仅仅是将它的每个基本类型的字段生成好放在内存中。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/interface-4.png)

```go
0x004b 00075 (main.go:6)	CALL	runtime.convTstring(SB)
0x0050 00080 (main.go:6)	MOVQ	16(SP), AX
0x0055 00085 (main.go:6)	MOVQ	AX, ""..autotmp_2+40(SP)
0x005a 00090 (main.go:6)	LEAQ	go.itab."".Student,"".Person(SB), CX
0x0061 00097 (main.go:6)	MOVQ	CX, "".s+48(SP)
0x0066 00102 (main.go:6)	MOVQ	AX, "".s+56(SP)
```

上述代码生成了 interface。第 1 行调用了 runtime.convTstring()。

```go
func convTstring(val string) (x unsafe.Pointer) {
	if val == "" {
		x = unsafe.Pointer(&zeroVal[0])
	} else {
		x = mallocgc(unsafe.Sizeof(val), stringType, true)
		*(*string)(x) = val
	}
	return
}
```

runtime.convTstring() 会从栈顶 +0(SP) 取出入参 "halfrost" 和长度 8。在栈上生成了一个字符串的变量，返回了它的指针放在 +16(SP) 中，并拷贝到 +40(SP) 里。第 4 行生成了 itab 的指针，这里和上一章里面一致，不再赘述。至此，iface 生成了，*itab 和 unsafe.Pointer 分别存在 +48(SP) 和 +56(SP) 中。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/interface-5.png)

```go
0x006b 00107 (main.go:7)	MOVQ	"".s+48(SP), AX
0x0070 00112 (main.go:7)	TESTB	AL, (AX)
0x0072 00114 (main.go:7)	MOVQ	32(AX), AX
0x0076 00118 (main.go:7)	MOVQ	"".s+56(SP), CX
0x007b 00123 (main.go:7)	MOVQ	CX, (SP)
0x007f 00127 (main.go:7)	LEAQ	go.string."everyone"(SB), CX
0x0086 00134 (main.go:7)	MOVQ	CX, 8(SP)
0x008b 00139 (main.go:7)	MOVQ	$8, 16(SP)
0x0094 00148 (main.go:7)	CALL	AX
```

最后一步是调用 interface 方法。这一步和上一节中的流程基本一致。先通过 itab 指针找到函数指针。然后将要调用的方法的入参都放在栈顶。最后调用即可。此时内存布局如下图：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/interface-6.png)

看到这里可能有读者好奇，为什么结构体类型转换里面也没有 runtime.convT2I() 方法调用呢？笔者认为这里是编译器的一些优化导致的。

```go
func convT2I(tab *itab, elem unsafe.Pointer) (i iface) {
	t := tab._type
	if raceenabled {
		raceReadObjectPC(t, elem, getcallerpc(), funcPC(convT2I))
	}
	if msanenabled {
		msanread(elem, t.size)
	}
	x := mallocgc(t.size, t, true)
	typedmemmove(t, x, elem)
	i.tab = tab
	i.data = x
	return
}
```

runtime.convT2I() 这个方法会生成一个 iface，在堆上生成 iface.data，并且会 typedmemmove()。笔者找了 2 个相关的 PR，感兴趣的可以看看。[optimize convT2I as a two-word copy when T is pointer-shaped](https://go-review.googlesource.com/c/go/+/20901/9)，[cmd/compile: optimize remaining convT2I calls](https://go-review.googlesource.com/c/go/+/20902)。这里仅仅涉及类型转换，所以在内存中构造出 *itab 和 unsafe.Pointer 就够用了。编译器觉得没有必要调用 runtime.convT2I() 再构造出 iface 多此一举。

#### Type Assertion

 Type assertion(断言)是用于 interface value 的一种操作，语法是` x.(T)`，x 是 interface type 的表达式，而 T 是 assertd type，被断言的类型。 

```go
package main
import "fmt"
func main() {
    // i is a empty interface
	var i interface{}
	i = "hello"

	s := i.(string)
	fmt.Println(s)

	s, ok := i.(string)
	fmt.Println(s, ok)

	f, ok := i.(float64)
	fmt.Println(f, ok)

	i = 100
	t, ok := i.(int)
	fmt.Println(t, ok)

	t2 := i.(string) //panic
	fmt.Println(t2)
}
--- output ---
hello
hello true
0 false
100 true
panic: interface conversion: interface {} is int, not string
```

断言的使用主要有两种情景:

**如果 asserted type 是一个 concrete type，一个实例类 type，断言会检查 x 的 dynamic type 是否和 T 相同，如果相同，断言的结果是 x 的 dynamic value**，当然 dynamic value 的 type 就是 T 了。换句话说，对 concrete type 的断言实际上是获取 x 的 dynamic value。

**如果 asserted type 是一个 interface type，断言的目的是为了检测 x 的 dynamic type 是否满足 T，如果满足，断言的结果是满足 T 的表达式，但是其 dynamic type 和 dynamic value 与 x 是一样的**。换句话说，对 interface type 的断言实际上改变了 x 的 type，通常是一个更大 method set 的 interface type，但是保留原来的 dynamic type 和 dynamic value。

这个断言interface type比较抽象，看下面例子：

```go
package main

import (
"fmt"
"io"
"os"
)

func main() {

var w io.Writer
w = os.Stdout
w.Write([]byte("hello Go!\n"))
fmt.Printf("%T\n", w)
fw := w.(*os.File)
fmt.Printf("%T\n", fw)
}
--- output --
hello Go!
*os.File
*os.File
```

在上面的代码中，w 是一个有 `Write` method 的 interface expression，其 **dynamic value 是 os.Stdout**，断言 `w.(*os.File)` 针对 concrete type `*os.File` 进行的，那么 f 就是 w 的 dynamic value `os.Stdout`。 

通常我们仅仅只是想知道 dynamic value 是哪种 concrete type ，可以借助 ok 表达式。 

```go
var w io.Writer = os.Stdout
f, ok := w.(*os.File) // success: ok, f == os.Stdout
b, ok := w.(*bytes.Buffer) // failure: !ok, b == nil
```

end

#### nil interface

interface 是一个元组 (type, value) 来唯一标识它--

官方定义：Interface values with nil underlying values:

- 只声明没赋值的interface 是nil interface，value和 type 都是 nil
- 只要赋值了，即使赋了一个值为nil类型，也不再是nil interface

```go
package main
import "fmt"
type Animal interface {
	Bark() string
	Walk() string
}
func main() {
	var a Animal
	fmt.Printf("a Type: %T, a value: %v", a,a)
}
--- output ---
a Type: <nil>, a value: <nil>
```

对象强制转换成 interface 的时候，不仅仅含有原来的对象，还会包含对象的类型信息。

```go
func main() {
	var x interface{} = nil
	var y *int = nil
	interfaceIsNil(x)
	interfaceIsNil(y)
}

func interfaceIsNil(x interface{}) {
	if x == nil {
		fmt.Println("empty interface")
		return
	}
	fmt.Println("non-empty interface")
}
// output
x is <nil>
empty interface
x is (*int)(nil)
non-empty interface

```

end

#### Empty interface

 Go 允许不带任何方法的 interface ,这种类型的 interface 叫 empty interface。所有类型都实现了 empty interface,因为任何一种类型至少实现了 0 个方法。  

空的 inferface{} 是没有方法集的接口。

 动态类型和动态值都是`nil`，为一个空接口。 

`type Any interface {}`

空接口，常做函数参数在配合反射使用。

```go
package main
import "fmt"
// i is a empty interface
func Print(i interface{}) {
	fmt.Printf("i type: %T, i value: %v\n",i,i)
}
func main() {
	var i interface{}
	fmt.Printf("init i type: %T, init i value: %v\n",i,i)
	i = "hello"
	Print(i)
	i = 100
	Print(i)
	i = 1.29
	Print(i)
}
--- output ---
init i type: <nil>, init i value: <nil>
i type: string, i value: hello
i type: int, i value: 100
i type: float64, i value: 1.29
```

end

##### any

**any就是一个interface{}的type alias，它与interface{}完全等价**



### Reflect

#### 限制

无法获取struct的private variable即小写字母开头的变量。

```go
package main

import (
	"fmt"
	"reflect"
)

type User struct {
	Id   int
	Name string
	Age  int
	//Salary int
	salary int
}


func main() {

	user := User{1, "Chandelier", 25, 20000}

	fmt.Println(reflect.ValueOf(user).Field(1).Interface())
	fmt.Println(reflect.ValueOf(user).Field(3).Interface())
}

// output
Chandelier
panic: reflect.Value.Interface: cannot return value obtained from unexported field or method
```

end

###  高玩，通过反射，调用方法:fire:

#### 核心

使用者(前端操作必然映射一个后端接口)，必须要清楚要调用的method name。然后利用反射根据传入的函数名称实现函数的参数填充和自动调用

#### 应用场景:grin:

核心：比如你写了开发框架，预留了一个list给开发者，传入list的method会按照顺序执行，就会利用到反射。

传入`Package.Method `和参数，框架自动去调用这个method

如果是静态调用，则需要框架`import all package`但是问题时，你预先不知道用户的package name。

做框架工程时，需要随意扩展方法或者给用户预留可以自定义方法的入口。

> 比如，一个两个下拉框，先选择实例，然后另一个下拉框选择要调用的方法。后台自动使用实例的属性去填充调用方法的形参。

这就需要反射了，让用户自己传入要调用的方法名称，然后通过reflect获取到函数前面，而是使用约定或者自定义的参数格式，调用方法。

eg：通过登陆态获取到User信息，然后调用User的getAge方法，就可以通过Reflect先获取User结构体的全部信息，再获取getAge的函数前面，最后实现getAge的方法调用。

#### 数据格式约定

开发过程中，要调用其他的未知方法，但是数据格式必须做好约定，这样通过Reflect获取到函数的参数时，可以更快速的填入正确的参数类型，进而动态调用函数。

比如约定，json格式内容、自定义变量类型等，类似下面的猜想

... 正则 不会写

```go

package main

import (
	"fmt"
	"reflect"
)

type nameString string
type ageInt int
type genderString string


type Person struct {
	Name nameString
	Age ageInt
	Gender genderString
}


func (p Person) Test( age ageInt, name nameString,){
	fmt.Println(age, name)
}

func (p Person) getName() nameString{
	return p.Name
}

func (p Person) getAge() ageInt{
	return p.Age
}

func (p Person) getGender() genderString{
	return p.Gender
}

func (p Person)valueMap(key string) interface{}  {
	switch key {
	case "nameString":
		return p.getName()
	case "ageInt":
		return p.getAge()
	case "genderString":
		return p.getGender()
	}
	return nil
}

func main() {
	person := Person{"Chandler",30,"Man"}
	getValue := reflect.ValueOf(person)

	methodValue := getValue.MethodByName("Test")
	fmt.Printf("Kind : %s, Type : %s\n",methodValue.Kind(),methodValue.Type())

	argNum := methodValue.Type().NumIn()

	argValues := make([]reflect.Value, 0, argNum)
	//valuesType := methodValue.Type().String()
	//fmt.Println(valuesType)
	//compileRegex := regexp.MustCompile("")
	//matchArr := compileRegex.FindStringSubmatch(valuesType)
	argtyps := []string{"ageInt", "nameString"}
	for _, i := range argtyps {
		argValues = append(argValues, reflect.ValueOf(person.valueMap(i)))
	}
	fmt.Println("get all arg values:", argValues)
	fmt.Println("call Test func")
	methodValue.Call(argValues)
}


```

end

#### demo

挺一般的demo，因为是写死了调用值，应该自动填充比较酷炫-

```go

package main

import (
	"fmt"
	"reflect"
)

type nameString string
type ageInt int
type genderString string


type Person struct {
	Name nameString
	Age ageInt
	Gender genderString
}

func (p Person)Say(msg string)  {
	fmt.Println("hello，",msg)
}
func (p Person)PrintInfo()  {
	fmt.Printf("Name：%s, Age：%d, Gender：%s\n",p.Name,p.Age,p.Gender)
}

func (p Person) Test(i,j int,s string){
	fmt.Println(i,j,s)
}


// 如何通过反射来进行方法的调用？
// 本来可以用结构体对象.方法名称()直接调用的，
// 但是如果要通过反射，那么首先要将方法注册，也就是MethodByName，然后通过反射调动mv.Call

func main() {
	person := Person{"Ruby",30,"Man"}
	// 1. 要通过反射来调用起对应的方法，必须要先通过reflect.ValueOf(interface)来获取到reflect.Value，
	// 得到“反射类型对象”后才能做下一步处理
	getValue := reflect.ValueOf(person)
	// 2.一定要指定参数为正确的方法名
	// 先看看没有参数的调用方法
	// 纳尼？你说你不知道你自己要调用的methodName或者methodID，那你调用个锤子，爬。
	methodValue1 := getValue.MethodByName("PrintInfo")
	fmt.Printf("Kind : %s, Type : %s\n",methodValue1.Kind(),methodValue1.Type())
	//没有参数，直接写nil
	fmt.Println("call PrintInfo func")
	methodValue1.Call(nil)

	args1 := make([]reflect.Value, 0) //或者创建一个空的切片也可以
	methodValue1.Call(args1)

	// 有参数的方法调用
	methodValue2 := getValue.MethodByName("Say")
	fmt.Printf("Kind : %s, Type : %s\n",methodValue2.Kind(),methodValue2.Type())
	args2 := []reflect.Value{reflect.ValueOf("Reflect")}
	fmt.Println("call Say func")
	methodValue2.Call(args2)

	methodValue3 := getValue.MethodByName("Test")
	fmt.Printf("Kind : %s, Type : %s\n",methodValue3.Kind(),methodValue3.Type())
	args3 := []reflect.Value{reflect.ValueOf(100), reflect.ValueOf(200),reflect.ValueOf("Hello")}
	fmt.Println("call Test func")
	methodValue3.Call(args3)
}

```

end



#### Java 反射

Java方面 当你做一个软件可以安装插件的功能，你连插件的类型名称都不知道，你怎么实例化这个对象呢？
因为程序是支持插件的（第三方的），在开发的时候并不知道 。所以，无法在代码中 New出来 ，但反射可以，通过反射，动态加载程序集，然后读出类，检查标记之后再实例化对象，就可以获得正确的类实例。
反射的目的就是为了扩展未知的应用。比如你写了一个程序，这个程序定义了一些接口，只要实现了这些接口的dll都可以作为插件来插入到这个程序中。那么怎么实现呢？就可以通过反射来实现。就是把dll加载进内存，然后通过反射的方式来调用dll中的方法。
很多工厂模式就是使用的反射。

#### webhook和反射接口

Webhooks是一个API概念，是微服务API的使用范式之一，也被成为反向API，即：`前端不主动发送请求，完全由后端主动向前端推送`。

举个常用例子，比如你在微信发了一条动态，后端会将这条消息推送给你所有的好友的客户端（朋友圈），这就是 Webhooks 的典型场景。

简单来说，Webhooks就是一个接收HTTP POST（或GET，PUT，DELETE）的URL。一个实现了Webhooks的API提供商就是在当事件发生的时候会向这个配置好的URL发送一条信息。与`主动请求-响应式`不同，使用Webhooks，你可以实时被动的接受到变化信


**webhook像是触发器，reflect接口则是后端框架的触发器。**

### 代码

https://www.cnblogs.com/CharmCode/p/14315474.html

### 引用

1. https://www.cnblogs.com/Csir/p/9561488.html
2. https://juejin.cn/post/7061933166717567012
2. https://halfrost.com/go_interface/
2. https://blog.csdn.net/weixin_35792468/article/details/112923590