Golang 陷阱

1. http://devs.cloudimmunity.com/gotchas-and-common-mistakes-in-go-golang/index.html



```go

/**
defer 的栈的特性： defer 是在线程栈上面一个专门为defer设计的栈,
	1. 执行顺序是先进后出
		https://www.vividcortex.com/blog/2014/01/15/two-go-memory-leaks/
	2. 这个栈深度是有限的，所以不建议在循环里面使用

 */
func TestDefer(t *testing.T) {
	for count := 0; count < 100; count++ {
		defer fmt.Println(count)
	}
}

type student struct {
	Name string
	Age  int
}

/**
	在Go的for…range循环中，Go始终使用值拷贝的方式代替被遍历的元素本身，
	简单来说，就是for…range中那个value，是一个值拷贝，而不是元素本身。
	这样一来，当我们期望用&获取元素的地址时，实际上只是取到了value这个临时变量的地址

	通过对临时变量的输出即可看出端倪
 */
func TestStruct(t *testing.T) {
	m := make(map[string]*student)
	stus := []student{
		{Name: "zhou", Age: 24},
		{Name: "li", Age: 23},
		{Name: "wang", Age: 22},
	}
	fmt.Println("-----------wrong -------------")
	//错误写法
	for _, stu := range stus {
		fmt.Println(unsafe.Pointer(&stu)) //会发现3个输出都是一样的
		m[stu.Name] = &stu
	}
	for key, value := range m {
		fmt.Printf("key : %s,value : %v \n", key, value)
	}
	fmt.Println("----------- Right -------------")
	//正确写法
	for index := 0; index < len(stus); index++ {
		m[stus[index].Name] = &stus[index]
	}
	for key, value := range m {
		fmt.Printf("key : %s,value : %v \n", key, value)
	}
	fmt.Println(m)
}

/**
	为什么下面的代码会随机输出？  一个典型的data race 案例 ： https://golang.org/doc/articles/race_detector.html

实际上第一行是否设置CPU为1都不会影响后续代码。两个for循环内部go func 调用参数i的方式是不同的，导致结果完全不同。这也是新手容易遇到的坑。
第一个go func中i是外部for的一个变量，地址不变化。遍历完成后，最终i=10。故go func执行时，i的值始终是10（10次遍历很快完成）。
第二个go func中i是函数参数，与外部for中的i完全是两个变量。尾部(i)将发生值拷贝，go func内部指向值拷贝地址。

 */
func TestWaiGroup(t *testing.T) {
	//runtime.GOMAXPROCS(1)
	var wg = sync.WaitGroup{}
	wg.Add(20)
	for i := 0; i < 10; i++ {
		go func() {
			fmt.Println("i: ", i)
			wg.Done()
		}()
	}
	for i := 0; i < 10; i++ {
		go func(i int) {
			fmt.Println("i: ", i)
			wg.Done()
		}(i)
	}
	wg.Wait()
}

/**

为什么下面的代码会随机抛出异常 ?

单个chan如果无缓冲时，将会阻塞。但结合 select可以在多个chan间等待执行。有三点原则：
	1. select 中只要有一个case能return，则立刻执行。
	2. 当如果同一时间有多个case均能return则伪随机方式抽取任意一个执行。
	3. 如果没有一个case能return则可以执行”default”块。
	此考题中的两个case中的两个chan均能return，则会随机执行某个case块。故在执行程序时，有可能执行第二个case，触发异常。具体参见官方文档
 */
func TestError(t *testing.T) {
	runtime.GOMAXPROCS(1)
	int_chan := make(chan int, 1)
	string_chan := make(chan string, 1)
	int_chan <- 1
	string_chan <- "hello"
	select {
	case value := <-int_chan:
		fmt.Println(value)
	case value := <-string_chan:
		panic(value)
	}
}


/**
在解题前需要明确两个概念：


 **defer 关键字修饰的 函数调用时，  参数首先发生值拷贝！**
不管代码顺序如何，defer calc func中参数b必须先计算，故会在运行到第三行时，执行calc("10",a,b)输出：10 1 2 3得到值3，将cal("1",1,3)存放到延后执执行函数队列中。

执行到第五行时，现行计算calc("20", a, b)即calc("20", 0, 2)输出：20 0 2 2得到值2,将cal("2",0,2)存放到延后执行函数队列中。

执行到末尾行，按队列先进后出原则依次执行：cal("2",0,2)、cal("1",1,3) ，依次输出：2 0 2 2、1 1 3 4 。
 */
func calc(index string, a, b int) int {
	ret := a + b
	fmt.Println(index, a, b, ret)
	return ret
}


func TestCal(t *testing.T) {
	a := 1                               //line 1
	b := 2                               //2
	defer calc("1", a, calc("10", a, b)) //3
	a = 0                                //4
	defer calc("2", a, calc("20", a, b)) //5
	b = 1                                //6
}


/**
	为什么下面的代码会抛出异常？ 怎么解决？
虽然有使用sync.Mutex做写锁，但是map是并发读写不安全的。map属于引用类型，并发读写时多个协程见是通过指针访问同一个地址，
即访问共享变量，此时同时读写资源存在竞争关系。会报错误信息:“fatal error: concurrent map read and map write”。

可以在在线运行中执行，复现该问题。那么如何改善呢? 当然Go1.9新版本中将提供并发安全的map。首先需要了解两种锁的不同：

sync.Mutex互斥锁
sync.RWMutex读写锁，基于互斥锁的实现，可以加多个读锁或者一个写锁。

 */

type UserAges struct {
	ages map[string]int
	sync.RWMutex
}

func (ua *UserAges) Add(name string, age int) {
	ua.Lock()
	defer ua.Unlock()
	ua.ages[name] = age
}

func (ua *UserAges) Get(name string) int {
	if age, ok := ua.ages[name]; ok {
		return age
	}
	return -1
}

func (ua *UserAges) RightGet(name string) int {
	ua.RLock()
	defer ua.RUnlock()
	if age, ok := ua.ages[name]; ok {
		return age
	}
	return -1
}

func TestUserGetSet(t *testing.T) {
	count := 1000
	gw := sync.WaitGroup{}
	gw.Add(count * 3)
	u := UserAges{ages: map[string]int{}}
	add := func(i int) {
		u.Add(fmt.Sprintf("user_%d", i), i)
		gw.Done()
	}
	for i := 0; i < count; i++ {
		go add(i)
		go add(i)
	}
	for i := 0; i < count; i++ {
		go func(i int) {
			defer gw.Done()
			//u.Get(fmt.Sprintf("user_%d", i))
			u.RightGet(fmt.Sprintf("user_%d", i))
		}(i)
	}
	gw.Wait()
	fmt.Println("Done")
}

```









1. request : currency_name ,address  , response : currency_name ,address
   1. request 和 response  中的currency_name是要入的币种
2. 映射检查 
   1. currency_name 没有对应的映射币种，返回错误
   2. currency_name 映射币种 在库中没有request 的address 返回错误
   3. currency_name 和对应的映射币种地址都已经存在，直接返回成功 currency_name， address 
3. 调wo端的 importKey接口，待响应成功，front端 拷贝 入库 address
   1. 出现异常就响应失败
4. 流程结束