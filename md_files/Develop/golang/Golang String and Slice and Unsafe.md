## Golang String and Slice and array  and Unsafe

### UTF-8

编码：按约定把二进制数字转成字符

Unicode编码统一是4个字节，存一个字母太浪费了。

utf-8通过约定定界符实现可变长的编码字节

一字节：以`0`开头即`0XXXXXXX`

两字节：`110`开头，第二个字节以`10`开头，即`110XXXXX 10XXXXXX`

三字节：`1110`开头，第二个字节以`10`开头，第三个字节以`10`开头即`1110XXXX 10XXXXXX 10XXXXXX`

四字节：`11110`开头，第二个字节以`10`开头，第三个字节以`10`开头，第四个以`10`开头

### String

字符串是由字符组成的数组，C 语言中的字符串使用字符数组 `char[]` 表示。数组会占用一片连续的内存空间，而内存空间存储的字节共同组成了字符串，且C语言中字符串各个字符可以按下标随意修改，**但Go 语言中的字符串只是一个只读的字节数组,**string的底层数据结构就是byte数组。



#### Stringheader

在运行中, Go `string` 值传递的是 [reflect.StringHeader](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fgolang%2Fgo%2Fblob%2Fgo1.16.7%2Fsrc%2Freflect%2Fvalue.go%23L1976), 内存大小是16字节(byte).

即**再长的字符串传递的都是16字节。**

```go
// reflect/value.go
// StringHeader is the runtime representation of a string.
// It cannot be used safely or portably and its representation may
// change in a later release.
// Moreover, the Data field is not sufficient to guarantee the data
// it references will not be garbage collected, so programs must keep
// a separate, correctly typed pointer to the underlying data.
type StringHeader struct {
	Data uintptr
	Len  int
}


func main()  {
	var str string
	fmt.Printf("string str size of byte num is: %d\n", unsafe.Sizeof(str))
}
// output
string str size of byte num is: 16
```



### Slice

`slice`是一个引用类型，是一个动态的指向数组切片的指针。

`slice`是一个不定长的，总是指向底层的数组`array`的数据结构

```go
// reflect/value.go
// slice结构体data字段存了指向的内存地址
// 直接创建slice变量时，系统默认给你创建了。
// slice指向的data内存地址，不够时会字段扩容
// so,slice 有bug
// slice_row  
// slice_s1 = slice_row
// extend slice_row to slice data memory address change
// slice_s2 = slice_row
// Now, slice_s1 不等于 slice_s2
type SliceHeader struct {
	Data uintptr
	Len  int
	Cap  int
}
```

Cap:  表示slice的长度

Len:  表示slice的容量

#### slice cap

**slice的capability是由它的起始位决定的即[n;m]中`;`前的n决定的和m没关系。**

比如，`s1 := s[:0]`，这表示`s1`的capability是`cap(s)`，而`len(s1)`是0。

```go
func main()  {
	str := make([]int,11,11)
	for i:=0;i<len(str);i++{
		str[i] = i
	}
	fmt.Printf("str is %v\t,len(str) is %d\n",str, len(str))
	s1 := str[1:3]
	s2 := str[8:10]

	fmt.Printf("s1 is %v\t,cap(s1) is %d\tlen(s1) is %d\n",s1, cap(s1),len(s1))
	fmt.Printf("s2 is %v\t,cap(s2) is %d\tlen(s2) is %d\n", s2,cap(s2),len(s2))
}

// output
// length 都是2 即 1,2 和 8,9 都是两个元素
str is [0 1 2 3 4 5 6 7 8 9 10]	,len(str) is 11
s1 is [1 2]	,cap(s1) is 10	len(s1) is 2
s2 is [8 9]	,cap(s2) is 3	len(s2) is 2
```

end

#### slice 扩容 and 内存规格

扩容原则如下

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/slice-cap-scale.jpg)

按照上面的规则，capability为21的slice，扩容后应该是42，但是呢下面代码运行的结果是44.

```go
func main()  {
	str := make([]string, 21, 21)
	fmt.Printf("str is %v\t,cap(str) is %d\tlen(str) is %d\n",str,cap(str), len(str))

	str = append(str, "Chandler")
	fmt.Printf("After append: str is %v\t,cap(str) is %d\tlen(str) is %d\n",str,cap(str), len(str))
}

// output
str is [                    ]	,cap(str) is 21	len(str) is 21
After append: str is [                     Chandler]	,cap(str) is 44	len(str) is 22
```

`44 ≠ 21 * 2`，这是什么原因呢，答案是**内存规格**

扩容预估cap为42，容量乘以元素大小 16 bytes，得到扩容后内存规格为`42 * 16 = 672`

但Golang 预设的内存规格Size Class共有67种，这个是写死在代码里的

```go
// runtime/sizeclass.go
var class_to_size = [_NumSizeClasses]uint16{0, 8, 16, 24, 32, 48, 64, 80, 96, 112, 128, 144, 160, 176, 192, 208, 224, 240, 256, 288, 320, 352, 384, 416, 448, 480, 512, 576, 640, 704, 768, 896, 1024, 1152, 1280, 1408, 1536, 1792, 2048, 2304, 2688, 3072, 3200, 3456, 4096, 4864, 5376, 6144, 6528, 6784, 6912, 8192, 9472, 9728, 10240, 10880, 12288, 13568, 14336, 16384, 18432, 19072, 20480, 21760, 24576, 27264, 28672, 32768}

```

根本没有672这个规格，所以往上匹配到了704，`704/16=44`。

所以扩容后的capability大小为44。

#### len为0的slice

但是有个问题哈，哪个煞笔会把slice前面初始化为0，而且不修改直接追加，这不是导致数据错误吗

当slice长得不确定时，会进行re-slice申请新的大容量slice，但是这样不断的re-allocate会增加内存消耗。

```go
func main()  {
	vals := make([]int, 4)
	fmt.Println("init vals:", vals)
	fmt.Println("init len:",len(vals), "init cap:", cap(vals))
	for i := 0; i < 6; i++ {
		vals = append(vals, 9)
	}
	fmt.Println("vals:", vals)
	fmt.Println("len:",len(vals), "init cap:", cap(vals))
}

// output
init vals: [0 0 0 0]
init len: 4 init cap: 4
vals: [0 0 0 0 9 9 9 9 9 9]
// cap 扩大了四倍
len: 10 init cap: 16
```

 将slice设置长度为 `0`，仅指定一个预估的 Capacity， 可以减少re-allocate的次数

```go
func main()  {
	vals := make([]int, 0,4)
	fmt.Println("init vals:", vals)
	fmt.Println("init len:",len(vals), "init cap:", cap(vals))
	for i := 0; i < 6; i++ {
		vals = append(vals, 9)
	}
	fmt.Println("vals:", vals)
	fmt.Println("len:",len(vals), "init cap:", cap(vals))
}
init vals: []
init len: 0 init cap: 4
vals: [9 9 9 9 9 9]
// --
len: 6 init cap: 8

```

当然我们初始化时指定的容量5很有可能比最终切片长度小，循环过程中还是会发生 re-allocate，但已经也减少了 re-allocate 次数。

#### 深拷贝

```go
func main() {
	slice1 := []int{1,2,3,4,5}
	slice2 := make([]int,5,5)
	copy(slice2,slice1)
	slice2[1]=100  //
	fmt.Println(slice1)
	fmt.Println(slice2)
	fmt.Println("-----")
	slice3 := slice1
	slice3[1] = 100
	fmt.Println(slice1)
	fmt.Println(slice2)

}

```

大同小异

### Array

在数组中由于长度固定不可变，因此`len(arr)`和`cap(arr)`的输出永远相同

```go
a := [3]int{}
fmt.Println(len(a), cap(a))
// output
3 3
```



### Unsafe：待补充

string 类型数据的字节数可以通过`unsafe.Sizeo()`查看值是16.

slice类似的数据字节数是24，因为比string多了一个capability为8个字节，所以是24

通过汇编来看，golang中string的声明赋值都会调用一个`convTstring()`

```go

func main()  {
	str := "test"
	fmt.Println(str)
}
// go tool compile -S -N -l main.go 
// Assembly code display to invoke convTstring function 
    	0x0018 00024 (main.go:8)        FUNCDATA        $2, "".main.stkobj(SB)
        0x0018 00024 (main.go:9)        LEAQ    go.string."test"(SB), CX
        0x001f 00031 (main.go:9)        MOVQ    CX, "".str+40(SP)
        0x0024 00036 (main.go:9)        MOVQ    $4, "".str+48(SP)
        0x002d 00045 (main.go:10)       MOVUPS  X15, ""..autotmp_1+56(SP)
        0x0033 00051 (main.go:10)       LEAQ    ""..autotmp_1+56(SP), CX
        0x0038 00056 (main.go:10)       MOVQ    CX, ""..autotmp_3+32(SP)
        0x003d 00061 (main.go:10)       MOVQ    "".str+40(SP), AX
        0x0042 00066 (main.go:10)       MOVQ    "".str+48(SP), BX
        0x0047 00071 (main.go:10)       PCDATA  $1, $1
		// here
        0x0047 00071 (main.go:10)       CALL    runtime.convTstring(SB)
        0x004c 00076 (main.go:10)       MOVQ    AX, ""..autotmp_4+24(SP)
        0x0051 00081 (main.go:10)       MOVQ    ""..autotmp_3+32(SP), CX
        0x0056 00086 (main.go:10)       TESTB   AL, (CX)
        0x0058 00088 (main.go:10)       LEAQ    type.string(SB), DX
        0x005f 00095 (main.go:10)       MOVQ    DX, (CX)
        0x0062 00098 (main.go:10)       LEAQ    8(CX), DI
        0x0066 00102 (main.go:10)       PCDATA  $0, $-2
        0x0066 00102 (main.go:10)       CMPL    runtime.writeBarrier(SB), $0

```

然后我就很好奇这个`runtime.convTstring()`是干嘛的...

```go

// runtime/iface.go
func convTstring(val string) (x unsafe.Pointer) {
	if val == "" {
		x = unsafe.Pointer(&zeroVal[0])
	} else {
		x = mallocgc(unsafe.Sizeof(val), stringType, true)
		*(*string)(x) = val
	}
	return
}

func convTslice(val []byte) (x unsafe.Pointer) {
	// Note: this must work for any element type, not just byte.
	if (*slice)(unsafe.Pointer(&val)).array == nil {
		x = unsafe.Pointer(&zeroVal[0])
	} else {
		x = mallocgc(unsafe.Sizeof(val), sliceType, true)
		*(*[]byte)(x) = val
	}
	return
}

```

runtime.convTstring和runtime.convTslice都返回了一个`unsafe.Pointer`类型数据，然后再看`Sizeof（）`方法

```go
// Sizeof takes an expression x of any type and returns the size in bytes
// of a hypothetical variable v as if v was declared via var v = x.
// The size does not include any memory possibly referenced by x.
// For instance, if x is a slice, Sizeof returns the size of the slice
// descriptor, not the size of the memory referenced by the slice.
// The return value of Sizeof is a Go constant.
func Sizeof(x ArbitraryType) uintptr
```

