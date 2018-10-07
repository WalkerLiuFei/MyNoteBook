# 深入GO

## 基本数据类型	

### string

1. GO中的字符串在内存中实际上是由两个2字长(byte)的数据结构表示的，其中前一个是指向字符串的指针，另外一个是长度值。
2. 切分长度s[i:j] 会得到一个新的字符串数据结构

### slice

```
struct Slice{
	//must not move anying 
    byte* array; //actual data
    uintgo len;  // number of elements
    uintgo cap;  //allocated number of elements
}
```

slice 的扩容规则是 ： 

- 如果新的大小是当前大小2倍以上，则大小增长为新大小
- 否则循环以下操作：如果当前大小小于1024，按每次2倍增长，否则每次按当前大小1/4增长。直到增长的大小超过或等于新大小。

1. 一个slice是一个数组某个部分的引用，在内存中它由三个包含三个域的结构体，指向slice中的第一个元素的指针，slice长度，以及slice的容量。
2. **数组的slice并不会复制一份数据，它只是创建了如上所述的一个包含三个域的数据结构**
3. 如果一个slice run out of capacity a new ,larger backing array is created and the old backing array is copied into the new created backing array , and the old array remains existing. 如果有任何指向原未扩容内存地址的指针，依旧是有效的！
4.  看下面的这个例子。通过这个例子应该明白 ：slice capcity的作用和 slice长度的作用。 capcity 是backing array 的长度，而 slice 的长度是限制索引的。 所以第二次的超过 slice长度的切割是合法的！

```
slice := []int{1,2,3,4,5,6}
y := slice[1:3]
	
fmt.Println(len(y))
fmt.Println(y)
//fmt.Println(y[3]) //error
y = y[0:5]
fmt.Println(len(y))
fmt.Println(y)
fmt.Println(y[3]) // 输出 为5
最终的结果是： 【2,3,4,5,6】	
	
```

​	

#### make 和 new

1. new(T)返回一个*T，返回的这个指针可以被隐式的消除引用，