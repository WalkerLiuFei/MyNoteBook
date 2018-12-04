# Golang 笔记

1. log的框架 [Uber的zap](https://github.com/uber-go/zap)

2. 对一个 io.writer的初始化：通过一个 new方法 ： ` var writer io.ReadWriter = new(bytes.Buffer)`

3. 接上面第二个 [new vs make](https://www.godesignpatterns.com/2014/04/new-vs-make.html)

4. 单元测试的文件必须以 xxx_test结尾，单元测试函数必须以 Testxxxx_Test(t *testing) 的形式

5. 如果一个单元测试引用了 同一个包下面的 Type或者方法之类，不能go test xx.go。 参数为单元测试文件的命令来跑单元测试，必须要按照 go tes packe_name  的命令跑。如果用当前的包已经被添加到 gopath下面，则可以只包含包名。不然必须要包名的绝对路径

6. [how to write Go code](https://golang.org/doc/code.html) 

   1. **Go 支持唯一workspace 路径，这个workspace就是你的GOPATH** 这意味着，你所有的开发等都需要在这个workspace下面完成！

   2. 这个workspace 下面有三个目录分别为 

      1. src : source code
      2. pkg : contain package objects 
      3. bin : constains exectable commands

      ```
      bin/
          hello                          # command executable
          outyet                         # command executable
      pkg/
          linux_amd64/
              github.com/golang/example/
                  stringutil.a           # package object
      src/
          github.com/golang/example/
              .git/                      # Git repository metadata
      	hello/
      	    hello.go               # command source
      	outyet/
      	    main.go                # command source
      	    main_test.go           # test source
      	stringutil/
      	    reverse.go             # package source
      	    reverse_test.go        # test source
          golang.org/x/image/
              .git/                      # Git repository metadata
      	bmp/
      	    reader.go              # package source
      	    writer.go              # package source
          ... (many more repositories and packages omitted) ...
      ```

      3. 

7. ```go
   switch numPointer.(type) {
   	case *uint8,*int:
   		byteLen = 1
   	case *uint16,*int16:
   		byteLen = 2
   	case *uint32,*int32:
   		byteLen = 3
   	case *uint64,*int64:
   		byteLen = 4
   	}
   ```

   以上代码，`变量名.(type)` 的形式 加上 switch语句，可以进行类型匹配，然后进行相应处理。。**另外注意switch语句中case的使用，相同的是写到同一行的！！！并且没有break语句**

   1. Golang中的匿名函数 (函数字面量)： 通过一个变量引用一个函数的字面量，然后通过这个变量来引用和这个函数，这个功能也可以被称之为`闭包`

      ```go
      func squares() func() int {
          var x int
          return func() int {
              x++
              return x * x
          }
      }
      func main() {
          f := squares()
          fmt.Println(f()) // "1"
          fmt.Println(f()) // "4"
          fmt.Println(f()) // "9"
          fmt.Println(f()) // "16"
      }
      ```

      ​

8. 可以通过[delve ](https://github.com/derekparker/delve) 来debug 。

9. String 拼接的几种方式 : 

   - strings.Join 最慢
   - fmt.Sprintf 和 string + 差不多
   - bytes.Buffer又比上者快约500倍

10. 通过匿名集合访问方法集

  1. 类型T方法集包含所有receiver T方法。
  2. 类型*T方法集包含所有receiver T+*T方法。
  3. 匿名嵌入S，T方法集包含所有receiver S方法。
  4. 匿名嵌入*S，T方法集包含所有receiver S+*S方法。
  5. 匿名嵌入S或*S，*T方法集包含所有receiver S+*S方法

  ```go
  type S struct{} 
    
  type T struct{ 
     S                  // 匿名嵌入字段 
  } 
    
  func(S)sVal()  {} 
  func(*S)sPtr() {} 
  func(T)tVal()  {} 
  func(*T)tPtr() {} 
    
  func methodSet(a interface{}) {           // 显示方法集里所有方法名字 
     t:=reflect.TypeOf(a) 
    
     for i,n:=0,t.NumMethod();i<n;i++ { 
         m:=t.Method(i) 
         fmt.Println(m.Name,m.Type) 
      } 
  } 
    
  func main() { 
     var t T
    
     methodSet(t)                 // 显示T方法集 
     println("----------") 
     methodSet(&t)                // 显示 *T方法集 
  }
  
  ```

11. 接口还有一个重要特征：将对象赋值给接口变量时，会复制该对象。并且你不可以修改它，因为其实unaddressable的

    ```go
    func main() { 
       d:=data{100} 
       var t interface{} =d
      
       p:= &t.(data)           // 错误:cannot take the address of t.(data) 
       t.(data).x=200         // 错误:cannot assign to t.(data).x
    }
    
    
    func main() { 
       d:=data{100} 
       var t interface{} = &d
      
       t.(*data).x=200
       println(t.(*data).x) 
    }
    
    ```

12. 只有当接口变量内部的两个指针（itab，data）都为nil时，接口才等于nil。

    ```go
    func main() { 
       var a interface{} =nil
       var b interface{} = (*int)(nil) 
      
       println(a==nil,b==nil) 
    }
    ```

13. 等待多个任务结束，推荐使用sync.WaitGroup。通过设定计数器，让每个goroutine在退出前递减，直至归零时解除阻塞

    ```go
    func main() { 
       var wg sync.WaitGroup
      
       for i:=0;i<10;i++ { 
           wg.Add(1)             // 累加计数 
      
           go func(id int) { 
               defer wg.Done()          // 递减计数 
      
               time.Sleep(time.Second) 
               println("goroutine",id, "done.") 
            }(i) 
        } 
      
       println("main...") 
       wg.Wait()                   // 阻塞，直到计数归零 
       println("main exit.") 
    }
    ```

14. `runtime.Gosched` 让出当前线程等待下一次调用再执行，`runtime.Goexit()` 终止当前任务，不会引发panic，不会影响其他任务的执行

15. 在通过传递指针来共享数据复制时，要额外注意并发安全！比如传递slice！

16. `chan` 内置函数cap和len返回缓冲区大小和当前已缓冲数量；而对于同步通道则都返回0，据此可判断通道是同步还是异步

17. 对于关闭通道 : **一次性事件用close效率更好，没有多余开销。** 连续或多样性事件，可传递不同数据标志实现。还可使用sync.Cond实现单播或广播事件。

18. 对于已经closed的通道或者nuil通道 ：发送和接收都有相应的规则

    1. 向已经关闭的通道发送数据会引发panic
    2. 从已关闭通道接收数据，返回缓冲或者零值
    3. 无论首发，nil通道都会堵塞
    4. 重复关闭，或关闭nil通道都会引发panic错误
    5. **在chan 的接收端close 会引发panic**

19. 通过类型执行来创建单向通道 ，对于单向通道，不能做逆向操作，不然会引发panic！并且无法将单向通道转换回去

    ```go
    c := make(chan int,2)
    var send chan <-int = c
    var receive  <- chan int = c
    ```

20. 可以使用使用signal包里面的方法对进程的运行状态进行监听

21. 将发往通道的数据打包，减少传输次数，可有效提升性能。从实现上来说，通道队列依旧使用锁同步机制，单次获取更多数据（批处理），可改善因频繁加锁造成的性能问题。

22. goroutine处于发送或接收阻塞状态，但一直未被唤醒。垃圾回收器并不收集此类资源，导致它们会在等待队列里长久休眠，形成资源泄漏

23. 将Mutex作为匿名字段时，相关方法必须实现为pointer-receiver，否则会因复制导致锁机制失效。

    ```go
    type data struct{ 
       sync.Mutex
    } 
      
    func(d data)test(s string) { 
       d.Lock() 
       defer d.Unlock() 
        for i:=0;i<5;i++ { 
           println(s,i) 
           time.Sleep(time.Second) 
        } 
    } 
      
    func main() { 
       var wg sync.WaitGroup
       wg.Add(2) 
      
       var d data //这样调用是不行的，匿名字段的锁会进行复制，导致同步失败
      
       go func() { 
           defer wg.Done() 
           d.test("read") 
        }() 
      
       go func() { 
           defer wg.Done() 
           d.test("write") 
        }() 
      
       wg.Wait() 
    }
    ```

24. 对使用Mutex的相关建议 ： 

    1. 对性能要求较高时，应避免使用defer Unlock。
    2. 读写并发时，用RWMutex性能会更好一些。
    3. 对单个数据读写保护，可尝试用原子操作。
    4. 执行严格测试，尽可能打开数据竞争检查。

25. 包名与目录名并无关系，不要求保持一致，但是同一目录下所有的源文件必须保持同一个包名

26. `go list net/...`       # 显示包路径列表（"..." 表示其下所有包）

27. 几个被特殊保留的包的名称

    1. `main` : 可执行入口（入口函数 main.main）
    2. `all` ： 标准库已经以及`GOPATH`中能找到的库
    3. `std`,`cmd`  : 标准库及工具链
    4. `documention` 存储文档信息，无法导入（和目录名无关）。

28. 所有保存在internal目录下的包（包括自身）仅能被其父目录下的包（含所有层次的子目录）访问。

29. vendor 就是针对本工程的依赖

30. 使用reflect 反射类型时，需要区分Type和Kind。前者表示真实类型（静态类型），后者表示其基础结构（底层类型）类别

31. 利用反射构造一些复杂的对象

    1. 构造一个数组 ： `   a:=reflect.ArrayOf(10,reflect.TypeOf(byte(0))) ` 
    2. 构造一个map ： `reflect.MapOf(reflect.TypeOf(""),reflect.TypeOf(0)) ` 

32. 指针变量的类型就是指针 ptr，必要和原类型混淆

33. benchmark 测试的原理 ： **它通过逐步调整B.N值，反复执行测试函数，直到能获得准确的测量结果** 

34. 如果在测试函数中要执行一些额外操作，那么应该临时阻止计时器工作。

35. **benchmark 测试是调用的函数的性能**

    ```go
    func add(x, y int) byte {
    	return byte(x + y)
    }
    
    func heap()[]byte{
    	return make([]byte,1024 * 1024 * 10)
    }
    
    func BenchmarkAdd(b *testing.B) {
    	fmt.Println("b.N=", b.N)
    	b.ReportAllocs()
    	for i := 0; i < b.N; i++ {
    		_ = add(1, 2)
    		_ = heap()
    	}
    }
    
    ```


