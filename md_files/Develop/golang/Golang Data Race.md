## Golang Data Race





### goroutine执行顺序

```go
func main(){
	var test int

	go testNum(&test)
	test = 8
	//go testNum(&test)
	//fmt.Println("main goroutine waiting goroutine flush")
	//time.Sleep(1  * time.Second)
	fmt.Println("main goroutine test addr",&test)
	fmt.Println("main goroutine test value",test)

	time.Sleep(3 * time.Second)

}

func testNum(n *int){
	fmt.Println("goroutine origin n", *n)
	*n = 4
	fmt.Println("goroutine change n value to ", *n)
}
//output
goroutine origin n 8
main goroutine test addr 0xc0000aa058
main goroutine test value 4
goroutine change n value to  4

```

如上输出所示，goroutine中修改了`test`变量的值，但是`func main`中仍然显示`test`的值为0。

#### v1：time.Sleep

增加

```go

```



### 参考引用

1. https://www.zhihu.com/question/434964023
2. https://www.modb.pro/db/72705
