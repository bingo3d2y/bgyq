## Golang Pointer

### nil pointer dereference

nil 最好不要作为参数传给函数--

```go
func main(){
	var n *int
	*n = 4
}

// panic: runtime error: invalid memory address or nil pointer dereference
n 刚初始化的值是<nil>，无法被访问
// 校正
func main(){
	var n *int
	fmt.Println(n)
	num := 3
	n = &num
	fmt.Println(*n)

}
// output
<nil>
3
```

#### 指针和指向指针的指针

通常指针做形参，是不是不应该直接修改形参的指向，而是应该只修改指针指向的内容。

```go
func main(){
	var test *int	
	go testNum(test)
	fmt.Println("main goroutine waiting goroutine flush")
	time.Sleep(8  * time.Second)
	fmt.Println("main goroutine test addr",test)
	fmt.Println("main goroutine test value",*test)

	time.Sleep(3 * time.Second)

}

func testNum(n *int){
	fmt.Println("goroutine origin n", n)
	num := 5
    // 这里n替换了原来执行的指针，即此时n已经不指向main中的test了
	n = &num
	fmt.Println("goroutine init n", n)
	fmt.Println("goroutine change n value to ", *n)
	time.Sleep(4 * time.Second)
}

// output 
main goroutine waiting goroutine flush
goroutine origin n <nil>
goroutine init n 0xc00008c000
goroutine change n value to  5
// test 仅初始化声明了，没有赋值所以为<nil>
main goroutine test addr <nil>
panic: runtime error: invalid memory address or nil pointer dereference
```

修正版

```go
var test *int
func main(){

	go testNum(test)
	fmt.Println("main goroutine waiting goroutine flush")
	time.Sleep(8  * time.Second)
	fmt.Println("main goroutine test addr",test)
	fmt.Println("main goroutine test value",*test)

	time.Sleep(3 * time.Second)

}

func testNum(n *int){
	fmt.Println("goroutine origin n", n)
	num := 5
	n = &num
    // 或者不要形参n，直接给全局变量test赋值
	test = n
	fmt.Println("goroutine init n", n)
	fmt.Println("goroutine change n value to ", *n)
	time.Sleep(4 * time.Second)
}
```

end