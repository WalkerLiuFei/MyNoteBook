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