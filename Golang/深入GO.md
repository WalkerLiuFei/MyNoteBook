# 深入GO（基本数据类型）

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

1. new(T)返回一个*T，是一个已经指向已清零内存的指针，而make返回一个复杂的结构

#### slice 和 unsafe.Pointer的相互转换

1. slice 转 Pointer比较容易 ： 

   ```go
   s := make([]byte, 200)
   ptr := unsafe.Pointer(&s[0]）
                         
                        
   ```

2. Pointer 转 slice比较麻烦，下面是几种方式 ：

   ```go
    
   func TestPointerToSlice(t *testing.T){
   
   	//s := make([]byte, 200)
   	//ptr := unsafe.Pointer(&s[0])
   
   	backingArray := [10]byte{1,2,3,4,5,6,7,8,9,10}
   	ptr := unsafe.Pointer(&backingArray[0])
   
   
   	s := ((*[10]byte)(ptr))[:10]
   	fmt.Println(s)
   
   	var s1 = struct {
   		addr uintptr
   		len int
   		cap int
   	}{uintptr(ptr), 10, 10}
   	s2 := *(*[]byte)(unsafe.Pointer(&s1))
   	fmt.Println(s2)
   	//比较推荐的是这种	
   	var o []byte
   	sliceHeader := (*reflect.SliceHeader)((unsafe.Pointer(&o)))
   	sliceHeader.Cap = 10
   	sliceHeader.Len = 10
   	sliceHeader.Data = uintptr(ptr)
   	fmt.Println(o)
   
   }
   ```

#### map的实现



![Screen Shot](https://www.ardanlabs.com/images/goinggo/Screen+Shot+2013-12-31+at+7.01.15+PM.png)

 [参考](https://www.ardanlabs.com/blog/2013/12/macro-view-of-map-internals-in-go.html)  [参考2](https://dave.cheney.net/2018/05/29/how-the-go-runtime-implements-maps-efficiently-without-generics) 

1. map的底层实现是利用hash表实现的，其数据结构

   ```c
   struct Hmap
   {
       uint8   B;    // 可以容纳2^B个项
       uint16  bucketsize;   // 每个桶的大小
   
       byte    *buckets;     // 2^B个Buckets的数组
       byte    *oldbuckets;  // 前一个buckets，只有当正在扩容时才不为空
   };
   ```

2. **java 中解决Hash 冲突的方式是通过拉链法来解决的，即:在遇到hash冲突时在键值所在的槽开一个链表，冲突会以链表的形式接在之前的node上面，而GO使用的是`open addressing ` 中的hash 桶的的方式来解决的 **  [参考](https://www.jianshu.com/p/dbe7a1ea5928) ,[线性探测](https://zh.wikipedia.org/wiki/%E7%BA%BF%E6%80%A7%E6%8E%A2%E6%B5%8B) ，[线性探测2](http://www.cs.rmit.edu.au/online/blackboard/chapter/05/documents/contribute/chapter/05/linear-probing.html) 

3. 这个hash结构使用的是一个可扩展哈希的算法， 由hash值mod当前hash表大小决定某一个值属于哪个桶， 而hash表大小是2 的指数， 即上面结构体中的2^B。 只有当map在扩容时，oldbuckets才不为空。bucket = 2 * oldbuckets

4. Bucket的结构如下所示 ： 

   ```c
   struct Bucket
   {
   uint8 tophash[BUCKETSIZE]; // hash值的高8位....低位从bucket的array定位到bucket
   Bucket *overflow; // 溢出桶链表， 如果有
   byte data[1]; // BUCKETSIZE keys followed by BUCKETSIZE values
   };
   ```

5. 注意到一个细节 ： Bucket中所有的key放置到一起，所有的value放置到一起，比如 map[int64]int8这样的map结构，可以解决掉内存对齐的问题

6. map使用的是**增量扩容**，即在map扩容以后 ,原本的旧元素要移动到新的位置上，这个移动的操作不是一下子完成的，而是在每次 insert ，remove的时候完成的！正是由于这个工作是逐渐完成的， 这样就会导致一部分数据在old table中， 一部分在new table中， 所以对于hash table的
   insert, remove, lookup操作的处理逻辑产生影响。 只有当所有的bucket都从旧表移到新表之后， 才会将oldbucket释放掉。 

7. A map value is a pointer to a `runtime.hmap` structure.

#### nil的语义 

1. 任何类型在未初始化时都对应一个零值： 布尔类型是false， 整型是0， 字符串是""， 而指针， 函数，
   interface， slice， channel和map的零值都是nil 

2. 一个interface 在没有进行初始化时，对应的值是nil,也就是说 `var v interface` ，此时v就是一个nil ，

3. 此时v就是一个nil。 在底层存储上， 它是一个空指针。 与之不同的情况是， interface值为空。 比如：

   ```
   var v *T
   var i interface{}
   i = v
   ```

   此时i是一个interface， 它的值是nil， 但它自身不为nil。 通过这个例子更能体现出 

   ```go
   type Error struct {
   	errCode uint8
   }
   
   func (e *Error) Error() string {
   	switch e.errCode {
   	case 1:
   		return "file not found"
   	case 2:
   		return "time out"
   	case 3:
   		return "permission denied"
   	default:
   		return "unknown error"
   	}
   }
   
   func checkError(err error) {
   	if err != nil {
   		panic(err)
   	}
   }
   
   func TestError(t *testing.T) {
   	var e *Error
   	//checkError(e)
   	if e != nil{
   		panic(e)
   	}
   	fmt.Println("test passed")
   }
   
   ```
