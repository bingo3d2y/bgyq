## Golang Memory Model

### Memory Visibility

Memory Visibility(内存可见性)，是指一个线程修改了对象状态后，其他线程能够看到发生的状态变化。那么在 Go 中如何来保证呢？《The Go Memory Model》，它详细介绍了在多个 goroutine 情况下，如何保证共享变量的可见性以及 **Happens Before** 原则。为了保证共享变量的可见性，该文档中的建议如下：

Programs that modify data being simultaneously accessed by multiple goroutines must serialize such access

程序中多个 goroutine 同时访问某个数据，必须保证**串行化**访问。即需要增加同步逻辑，可以使用 sync 或者 sync/atomic中的锁或原子类型来保证。另外还可以尝试用 channel 来实现这个同步逻辑。



### 引用

1. https://www.zhihu.com/question/434964023/answer/1635454280
2. 