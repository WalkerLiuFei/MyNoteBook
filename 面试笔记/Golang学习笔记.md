---
title: Golang 语法学习笔记(一)
---


+ 在Golang中以大写字母开头的 结构体，函数，变量是外部可见的！
+ defer 声明用来在 函数范围内所有的函数执行完之后才会被执行，可以类比于 Java 中finally的使用。（defer statement defers the execution of a function until the surrounding function returns.）

+ 在import package 中出现的 "import for side-effect" error：**golang 遵循一个原则，“用多少，拿多少”，如果你拿的东西比你用的多，它就会报出error。。在这里你导入的包，包里面的东西你从来没用过他就报出这个error。同样的变量声明也是一样的！** 

+ golang 中 range遍历

```Golang
	
	strings := []string{"1256","dasdal"};
	for _,str := range strings{} //遍历的是值
	for str := range strings{} //遍历的是index
	//在用range遍历一个字符串的时候，其可以返回两个值
	//第一个值是 index,第二个值是 value，类型为run == int32 
	for i, c := range "go" {
	      fmt.Println(i, c)
	}

```
  
+ Golang 中，如果你想对某个对象进行判`nil`。是做不到的，Golang类似C语言，nil 是对空指针准备的。。。也就是是说
**Golang中的nil和Java 中的Null并不是一码事，nil是为指针准备的！！**

+ Golang 中并没有什么**char** ,你可以利用 int16 两个字节作为 Java中char的替代者
