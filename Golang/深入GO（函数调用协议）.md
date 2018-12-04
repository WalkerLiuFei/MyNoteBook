# 深入GO（函数调用协议）

## Go 中调用 C

1. C文件中必须要包含`runtime.h`头文件 ，且函数的开头需要时hi

## GO 中的多值返回

被调函数被定义为下面形式， 在函数中会修改ret1和ret2。 对指针参数所指向的内容的修改会被返回到调用函数， 用这种方式实现多值返回。 

`func f(arg1, arg2 int) (ret1, ret2 int) `  其实可以简单的理解为 类似的函数 `func f(arg1,arg2 int,ret1,ret2 *int) ` 直接该百年后面两个指针指向内存的值，即可实现多值返回的功能

 定义的一个 go 函数 ： `func f(arg1, arg2 int) (ret1, ret2 int) ` GO的做法是在传入的参数之上留了两个空位，被调者直接将返回值放在这两空位， 函数f调用前其内存布局是这样的： 

```
为ret2保留空位
为ret1保留空位
参数3
参数2
参数1 <-SP
```

调用之后变为：

```
为ret2保留空位
为ret1保留空位
参数2
参数1  <-FP
保存PC <-SP
f的栈
...
```

![img](https://tiancaiamao.gitbooks.io/go-internals/content/zh/images/3.2.funcall.jpg?raw=true)

**这就是Go和C函数调用协议中很重要的一个区别：为了实现多值返回，Go是使用栈空间来返回值的。而常见的C语言是通过寄存器来返回值的。** 

## GO 关键字

1. 对于一个 语句： `go f(x,y,z)` 函数f, 变量x,y,z的值都是在原goroutine 中计算的，只有函数f的执行是在新的goroutine中的。显然**新的gorouine 不能和当前的go线程公用一个栈** 

2. 在进行使用时，go会将函数f 和参数大小，作为函数`runtime.newproc` 的参数传递进去，并开启一个线程。下面时`newproc` 的注释

   ```Create a new g running fn with siz bytes of arguments. Put it on the queue of g&#39;s waiting to run. The compiler turns a go statement into a call to this. Cannot split the stack because it assumes that the arguments are available sequentially after &amp;fn; they would not be copied if a stack split occurred. go:nosplit
   Create a new g running fn with siz bytes of arguments. Put it on the queue of g's waiting to run. The compiler turns a go statement into a call to this. Cannot split the stack because it assumes that the arguments are available sequentially after &fn; they would not be copied if a stack split occurred. go:nosplit
   ```

3. `runtime.newproc` 这个函数会创建一个栈空间，将栈参数的12个字节拷贝到新栈空间中并让栈指针指向参数。

4. 可以这么说 `go f(args)` 也就是相当于 ： `runtime.newproc(size,f,args)` 的一个语法糖而已

5. 在`runtime.newproc`中，会新建一个栈空间，将栈参数的n个字节拷贝到新栈空间中并让栈指针指向参数。这时的线程状态有点像当被调度器剥夺CPU后一样，寄存器PC、SP会被保存到类似于进程控制块的一个结构体struct G内。f被存放在了struct G的entry域，后面进行调度器恢复goroutine的运行，新线程将从f开始执行。

## Defer关键字

1. 一个函数中如果有多个defer则其执行顺序会像栈一样，越后面的defer越最先被调用
2. 在函数中 return 和 defer一块使用时尤其需要注意 ： 
   1. **defer 确实是在 函数return 语句之前进行调用的, 但是要明确一点，return 语句是不是原子化的！**
   2. 函数返回的过程时这样的，先给返回值赋值，然后调用defer表达式，最后才是返回到调用函数中
   3. 返回值 = xxx , 调用 defer 函数，空的return。

**看下面的例子：**

```go
func TestDefer1(t *testing.T) {
	fmt.Println(test1()) // 1
	fmt.Println(test2()) // 5 
	fmt.Println(test3()) // 1

}

func test1() (result int) {
	defer func() {
		result++
	}()
	return 0
}

func test2() (r int) {
	t := 5
	defer func() {
		t = t + 5
	}()
	return t
}

func test3() (r int) {
	defer func(r int) {
		r = r + 5
	}(r)
	return 1
}
```

**以上例子同等func：**  

```go
func TestDefer1(t *testing.T) {
	fmt.Println(test1Equal())
	fmt.Println(test2Equal())
	fmt.Println(test3Equal())
}

func test1Equal()(result int){
	result = 0  //return语句不是一条原子调用，return xxx其实是赋值＋ret指令
	func() { //defer被插入到return之前执行，也就是赋返回值和ret指令之间
		result++
	}()
	return
}

func test2Equal()(result int){
	t := 5
	result = t //赋值指令
	func(){
		t = t + 5
	}()
	return
}

func test3Equal()(r int){
	r = 1  //给返回值赋值
	func(r int) {        //这里改的r是传值传进去的r，不会改变要返回的那个r值
		r = r + 5
	}(r)
	return        //空的return
}
```

其实仔细考虑下也是，func的调用其实就是以一个指针传参的形式来进行的。这样的进行赋值的操作完全说的过去的！

defer确实是在return之前调用的。但表现形式上却可能不像。本质原因是return xxx语句并不是一条原子指令，defer被插入到了赋值 与 ret之间，因此可能有机会改变最终的返回值。

 

defer 其实 是`runtime.deferproc` 的语法糖而已

**goroutine的控制结构中，有一张表记录defer，调用runtime.deferproc时会将需要defer的表达式记录在表中，而在调用runtime.deferreturn的时候，则会依次从defer表中出栈并执行。**

## 连续栈

goroutine可以初始时只给栈分配很小的空间，然后随着使用过程中的需要自动地增长。这就是为什么Go可以开千千万万个goroutine而不会耗尽内存。

而连续栈就是Go为goroutine 分配栈空间的技术

1. 当goroutine遇到执行栈空间不足时，会触发中断。进入到运行时库，运行时库会保存goroutine执行的上下文。为当前的执行栈重新分配一个新的更大执行栈空间，并将原有栈空间中的数据拷贝到新的执行栈上面
2. 在Go的运行时库中，每个goroutine对应一个结构体G，大致相当于进程控制块的概念。这个结构体中保存了stackbase和stackguard，用于确定这个goroutine使用的栈空间信息。

----

1. 使用分段栈的函数头几个指令检测SP和stackguard，调用runtime.morestack
2. runtime.morestack函数的主要功能是保存当前的栈的一些信息，然后转换成调度器的栈调用runtime.newstack
3. runtime.newstack函数的主要功能是分配空间，装饰此空间，将旧的frame和arg弄到新空间
4. 使用gogocall的方式切换到新分配的栈，gogocall使用的JMP返回到被中断的函数
5. 继续执行遇到RET指令时会返回到runtime.lessstack，lessstack做的事情跟morestack相反，它要准备好从new stack到old stack

整个过程有点像一次中断，中断处理时保存当时的现场，弄个新的栈，中断恢复时恢复到新栈中运行。栈的收缩是垃圾回收的过程中实现的．当检测到栈只使用了不到1/4时，栈缩小为原来的1/2.

### Goroutine  和 JVM thread 的区别

Goroutine的数量可以比JVM thread 开启数量高一个数量级，为什么会是这样？

**线程的概念**：

1. A single cpu core can only run one thread by turly concurrent。

2. 基于原因1，所以如果线程数量大于core 数量的话，就需要CPU来管理线程实现其”并发“，也就是 pause - rerume操作
3. 实现pause - resume操作的话，就需要**一个PC指针保存被暂停线程的执行到哪里了，一个stack 来保存目前thread 的执行上下文状态**

JVM使用的是 system thread，而开启每个 thread 是需要分配一个固定大小的栈内存给这个thread，**在64位机器上面这个固定大小为1MB** ，你可以使得这个固定大小小一些，但是带来的问题就是会遇到 stack overflow的问题！

根据上述Goroutine的stack 大小是动态分配的，所以更加灵活，对于并发的小thread，可以做到比JVM thread 高若干个数量级的优势。



另外: 由于JVM使用的是系统级别的thread , 它就涉及到上下文切换的问题 ： [context switch](https://en.wikipedia.org/wiki/Context_switch) ,而这个上线文切换过程中是需要多个操作的，消耗的时间大概为 ：1-100µ seconds  

而Goroutine 其实就是将多个协程都安排在同一个系统级别下的Thread下进行运行的。这样就规避了OS Thread的context switch 问题

### Kernal thread vs User space thread

Kernal thread advantage 

1. 因为内核完全了解所有线程，所以Scheduler可能决定给拥有大量操作的线程留出的时间多于具有少量操作的线程。
2. 对于有大量堵塞的线程，使用 kernal thread是更好的选择

User Space thread 的缺点

1. 用户级线程不像其他任何东西那样是一个完美的解决方案，它们是一种权衡。由于用户级线程对操作系统不可见，因此它们与操作系统无法很好地集成。因此，Os可以做出糟糕的决定，例如使用空闲线程调度进程，阻止其线程启动I / O的进程，即使进程有其他线程可以运行和取消调度具有锁定线程的进程。解决这个问题需要在内核和用户级线程管理器之间进行通信。线程和操作系统内核之间缺乏协调。因此，整个过程得到一个时间片，无论过程是否有一个线程或1000个线程。由每个线程决定放弃对其他线程的控制。用户级线程需要非阻塞系统调用，即多线程内核。否则，整个进程将在内核中被阻塞，即使进程中仍有可运行的线程。例如，如果一个线程导致页面错误，则进程阻塞。

## GO 中的闭包

go 闭包的一个例子：

```go

func f(i int) func() int {
    return func() int {
        i++
        return i
    }
}
```

**闭包的环境中引用的变量是不可以在函数栈中进行分配** 



1. Go语言支持闭包
2. Go语言能通过escape analyze识别出变量的作用域，自动将变量在堆上分配。将闭包环境变量在堆上分配是Go实现闭包的基础。
3. 返回闭包时并不是单纯返回一个函数，而是返回了一个结构体，记录下函数返回地址和引用的环境中的变量地址。