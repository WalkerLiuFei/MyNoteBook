# 深入GO（Goroutine 调度）

## 参考 

[What exactly does runtime.Gosched do?](https://stackoverflow.com/questions/13107958/what-exactly-does-runtime-gosched-do)

[concurrency](https://golang.org/doc/effective_go.html#concurrency)

[Go并发机制](https://github.com/k2huang/blogpost/blob/master/golang/%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/%E5%B9%B6%E5%8F%91%E6%9C%BA%E5%88%B6/Go%E5%B9%B6%E5%8F%91%E6%9C%BA%E5%88%B6.md)

[an-intro-about-goroutine-scheduler/](https://tonybai.com/2017/06/23/an-intro-about-goroutine-scheduler/) 



## G,P,M 调度模型

和goroutine 调度关的数据结构都定义在了 ： [runtime2](https://golang.org/src/runtime/runtime2.go) 中



![img](https://github.com/k2huang/blogpost/raw/master/golang/%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/%E5%B9%B6%E5%8F%91%E6%9C%BA%E5%88%B6/imgs/2.png)



### 对于 结构体 g ：



1. 包含了栈信息的 stackbase 和 stackguard和包含有运行函数信息的fnstart
2. **上下文信息保存在结构体的sched域中。goroutine是轻量级的`线程`或者称为`协程`，切换时并不必陷入到操作系统内核中，所以保存过程很轻量。看一下结构体G中的Gobuf，其实只保存了当前栈指针，程序计数器，以及goroutine自身。** 

### 结构体M

M是machine的缩写，是对机器的抽象，每个m都是对应到一条操作系统的物理线程。M必须关联了P才可以执行Go代码，但是当它处理阻塞或者系统调用中时，可以不需要关联P。

结构体M其实就是一个OS Thread

**Go运行时系统中的调度器的主要职责就是将G公平合理的安排到多个M上去执行**

### 结构体P

Go1.1中新加入的一个数据结构，它是Processor的缩写。结构体P的加入是为了提高Go程序的并发度，实现更好的调度。M代表OS线程。P代表Go代码执行时需要的资源。当M执行Go代码时，它需要关联一个P，当M为idle或者在系统调用中时，它也需要P。有刚好GOMAXPROCS个P。所有的P被组织为一个数组，在P上实现了工作流窃取的调度器。**每个P都维护有一个 Goroutine的队列** 

结构体P中也有相应的状态：

```
Pidle,
Prunning,
Psyscall,
Pgcstop,
Pdead,
```

## Sched

Sched是调度实现中使用的数据结构，该结构体的定义在文件proc.c中。

大多数需要的信息都已放在了结构体M、G和P中，Sched结构体只是一个壳。可以看到，其中有M的idle队列，P的idle队列，以及一个全局的就绪的G队列。Sched结构体中的Lock是非常必须的，如果M或P等做一些非局部的操作，它们一般需要先锁住调度器。

## Goroutine的声明周期

### Gorourtine的创建

1. 利用`runtime.newproc` 创建一个goroutine，创建后的Goroutine 处于 waiting状态等待执行
2. `runtime.newproc(size, f, args)`功能就是创建一个新的g，这个函数不能用分段栈，因为它假设参数的放置顺序是紧接着函数f的（见前面函数调用协议一章，有关go关键字调用时的内存布局）。分段栈会破坏这个布局。

总结 : **newproc -> newproc1 -> (如果P数目没到上限)wakep -> startm -> (可能引发)newm -> newosproc -> (线程入口)mstart -> schedule -> execute -> goroutine运行** 

### 进出系统调用

1. 假设goroutine"生病"了，它要进入系统调用了，暂时无法继续执行。进入系统调用时，如果系统调用是阻塞的，goroutine会被剥夺CPU，将状态设置成Gsyscall后放到就绪队列。 
2. Go的syscall库中提供了对系统调用的封装，它会在真正执行系统调用之前先调用函数.entersyscall，并在系统调用函数返回后调用.exitsyscall函数。这两个函数就是通知Go的运行时库这个goroutine进入了系统调用或者完成了系统调用，调度器会做相应的调度。



## goroutine的消亡以及状态变化

goroutine的消亡比较简单，注意在函数newproc1，设置了fnstart为goroutine执行的函数，而将新建的goroutine的sched域的pc设置为了函数runtime.exit。当fnstart函数执行完返回时，它会返回到runtime.exit中。这时Go就知道这个goroutine要结束了，runtime.exit中会做一些回收工作，会将g的状态设置为Gdead等，并将g挂到P的free队列中。





**从以上的分析中，其实已经基本上经历了goroutine的各种状态变化。在newproc1中新建的goroutine被设置为Grunnable状态，投入运行时设置成Grunning。在entersyscall的时候goroutine的状态被设置为Gsyscall，到出系统调用时根据它是从阻塞系统调用中出来还是非阻塞系统调用中出来，又会被设置成Grunning或者Grunnable的状态。在goroutine最终退出的runtime.exit函数中，goroutine被设置为Gdead状态。**





![img](https://tiancaiamao.gitbooks.io/go-internals/content/zh/images/5.2.goroutine_state.jpg?raw=true)

## Goroutine 的调度



1. 其实Go语言中的goroutine就是协程。每个结构体G中有一个sched域就是用于保存自己上下文的。这样，这种goroutine就可以被换出去，再换进来。这种上下文保存在用户态完成，不必陷入到内核，非常的轻量，速度很快。保存的信息很少，只有当前的PC,SP等少量信息。只是由于要优化，所以代码看上去更复杂一些，比如要重用内存空间所以会有gfree和mhead之类的东西。

#### 抢占式调度

当GC时，GC线程会通知其他所有线程先停下来，如果有一个线程没有被停下来，则GC会一直等待那个线程。

抢占式调度会解决这个问题。

### sysmon 

在一个main函数里面调用 `numGo := runtime.NumGoroutine()` 会发现结果为2.

Go程序的初始化过程中有提到过，runtime开了一条后台线程，运行一个sysmon函数。这个函数会周期性地做epoll操作，同时它还会检测每个P



1. 如果检测到某个P状态处于Psyscall超过了一个sysmon的时间周期(20us)，并且还有其它可运行的任务，则切换P。



## 调度算法

按照是否有优先级，调度算法主要可分为

- 1、基于优先级的可抢占式调度

  如果出现具有更高优先级的任务处于就绪状态，将当前任务停止运行，把cpu的控制权交给高优先级任务。

- 2、时间片轮转调度

  各个任务按照先入先出原则排成一个队列，新任务入队尾，调度时选队首。

- 3、轮询，直到  ①任务执行完毕     ②任务主动放弃cpu    ③ 任务IO操作阻塞

### 协程特点

- 一个线程中模拟多个协程并发执行
- 没有优先级、非抢占式调度：任务不能主动抢占时间片
- 每个协程都有自己的堆栈和局部变量
- 采用轮询的调度方式

### 其他

1. `GOMAXPROCS `： 

#### 3.2 调度过程

Go运行时完整的调度系统是很复杂，很难用一篇文章描述的清楚，这里只能从宏观上介绍一下，让大家有个整体的认识。

```
// Goroutine1
func task1() {
    go task2()
    go task3()
}
```

假如我们有一个G(Goroutine1)已经通过P被安排到了一个M上正在执行，在Goroutine1执行的过程中我们又创建两个G，这两个G会被马上放入与Goroutine1相同的P的本地G任务队列中，排队等待与该P绑定的M的执行，这是最基本的结构，很好理解。 关键问题是:
**a. 如何在一个多核心系统上尽量合理分配G到多个M上运行，充分利用多核，提高并发能力呢？**
如果我们在一个Goroutine中通过**go**关键字创建了大量G，这些G虽然暂时会被放在同一个队列, 但如果这时还有空闲P（系统内P的数量默认等于系统cpu核心数），Go运行时系统始终能保证至少有一个（通常也只有一个）活跃的M与空闲P绑定去各种G队列去寻找可运行的G任务，该种M称为**自旋的M**。一般寻找顺序为：自己绑定的P的队列，全局队列，然后其他P队列。如果自己P队列找到就拿出来开始运行，否则去全局队列看看，由于全局队列需要锁保护，如果里面有很多任务，会转移一批到本地P队列中，避免每次都去竞争锁。如果全局队列还是没有，就要开始玩狠的了，直接从其他P队列偷任务了（偷一半任务回来）。这样就保证了在还有可运行的G任务的情况下，总有与CPU核心数相等的M+P组合 在执行G任务或在执行G的路上(寻找G任务)。
**b. 如果某个M在执行G的过程中被G中的系统调用阻塞了，怎么办？**
在这种情况下，这个M将会被内核调度器调度出CPU并处于阻塞状态，与该M关联的其他G就没有办法继续执行了，但Go运行时系统的一个监控线程(sysmon线程)能探测到这样的M，并把与该M绑定的P剥离，寻找其他空闲或新建M接管该P，然后继续运行其中的G，大致过程如下图所示。然后等到该M从阻塞状态恢复，需要重新找一个空闲P来继续执行原来的G，如果这时系统正好没有空闲的P，就把原来的G放到全局队列当中，等待其他M+P组合发掘并执行。
[![img](https://github.com/k2huang/blogpost/raw/master/golang/%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/%E5%B9%B6%E5%8F%91%E6%9C%BA%E5%88%B6/imgs/3.jpg)](https://github.com/k2huang/blogpost/blob/master/golang/%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/%E5%B9%B6%E5%8F%91%E6%9C%BA%E5%88%B6/imgs/3.jpg)
**c. 如果某一个G在M运行时间过长，有没有办法做抢占式调度，让该M上的其他G获得一定的运行时间，以保证调度系统的公平性?**
我们知道linux的内核调度器主要是基于时间片和优先级做调度的。对于相同优先级的线程，内核调度器会尽量保证每个线程都能获得一定的执行时间。为了防止有些线程"饿死"的情况，内核调度器会发起抢占式调度将长期运行的线程中断并让出CPU资源，让其他线程获得执行机会。当然在Go的运行时调度器中也有类似的抢占机制，但并不能保证抢占能成功，因为Go运行时系统并没有内核调度器的中断能力，它只能通过向运行时间过长的G中设置抢占flag的方法温柔的让运行的G自己主动让出M的执行权。 
说到这里就不得不提一下Goroutine在运行过程中可以动态扩展自己线程栈的能力，可以从初始的2KB大小扩展到最大1G（64bit系统上），因此在每次调用函数之前需要先计算该函数调用需要的栈空间大小，然后按需扩展（超过最大值将导致运行时异常）。Go抢占式调度的机制就是利用在判断要不要扩栈的时候顺便查看以下自己的抢占flag，决定是否继续执行，还是让出自己。
运行时系统的监控线程会计时并设置抢占flag到运行时间过长的G，然后G在有函数调用的时候会检查该抢占flag，如果已设置就将自己放入全局队列，这样该M上关联的其他G就有机会执行了。但如果正在执行的G是个很耗时的操作且没有任何函数调用(如只是for循环中的计算操作)，即使抢占flag已经被设置，该G还是将一直霸占着当前M直到执行完自己的任务。