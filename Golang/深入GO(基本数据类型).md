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

 [参考](https://www.ardanlabs.com/blog/2013/12/macro-view-of-map-internals-in-go.html)  [参考2](https://dave.cheney.net/2018/05/29/how-the-go-runtime-implements-maps-efficiently-without-generics)  [closed hashing](https://www.cs.wcupa.edu/rkline/ds/closed-hashing.html) 

map 的源码实现位置 ： `go/src/runtime/hashmap.go`

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

3. go 里面的map，每个bucket 的entry 数量为8

4. 你需要知道value的size 大小，因为这会影响bucket的数据结构，

5. 这个hash结构使用的是一个可扩展哈希的算法， 由hash值mod当前hash表大小决定某一个值属于哪个桶， 而hash表大小是2 的指数， 即上面结构体中的2^B。 只有当map在扩容时，oldbuckets才不为空。bucket = 2 * oldbuckets

6. Bucket的结构如下所示 ： 

   ```go
   // A header for a Go map.
   type hmap struct {
   	// Note: the format of the Hmap is encoded in ../../cmd/internal/gc/reflect.go and
   	// ../reflect/type.go. Don't change this structure without also changing that code!
   	count     int // # live cells == size of map.  Must be first (used by len() builtin)
   	flags     uint8
   	B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
   	noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
   	hash0     uint32 // hash seed
   
   	buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
   	oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
   	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)
   
   	extra *mapextra // optional fields
   }
   
   ```

7. 注意到一个细节 ： Bucket中所有的key放置到一起，所有的value放置到一起，比如 map[int64]int8这样的map结构，可以解决掉内存对齐的问题

8. map使用的是**增量扩容**，即在map扩容以后 ,原本的旧元素要移动到新的位置上，这个移动的操作不是一下子完成的，而是在每次 insert ，remove的时候完成的！正是由于这个工作是逐渐完成的， 这样就会导致一部分数据在old table中， 一部分在new table中， 所以对于hash table的
   insert, remove, lookup操作的处理逻辑产生影响。 只有当所有的bucket都从旧表移到新表之后， 才会将oldbucket释放掉。 

9. A map value is a pointer to a `runtime.hmap` structure.

10. go map 不是一个引用变量，并且不是通过引用来进行传递的。**`a map value is a pointer to a runtime.hmap structure `** map在编译期间会被生成runtime.hashmap.

11.  java Hash Map 和 Go map的比较的优缺点

    1. java hashmap的优点 ： HashMap是通用的，并且支持泛型

    2. key必须是个对象，这样会加重GC的压力，并且也会加重内存的负担，因为Object实际上在JVM中是以指针的方式在线程栈中进行引用的。
       1. Hash Map 是以 open hashing的方式来解决键冲突的，这导致会在Hash表意外的开辟新的内存空间
       2. hash 和equip 函数需要同时实现

12. 在编译期间 : map的操作会进行重写，类似的事情同样发生在channels 上面，因为map和channels 是相对复杂的数据结构，所以像类似select,send receive这样的操作最后是交给runtime完成的， 而像slice这样的，因为相对简单，所以通过本地编译是可行的

    ```
    v := m["key"]     → runtime.mapaccess1(m, ”key", &v)
    v, ok := m["key"] → runtime.mapaccess2(m, ”key”, &v, &ok)
    m["key"] = 9001   → runtime.mapinsert(m, ”key", 9001)
    delete(m, "key")  → runtime.mapdelete(m, “key”)
    ```

13. 编译后，生成一个maptype 对象，这个maptype对象是runtime.mapaccess1的第一个参数，每个maptype 对象含有针对这个键值对的具体信息，包括类型描述，键值指向的地址

    ```go
    
    type maptype struct {
    	typ           _type
    	key           *_type
    	elem          *_type
    	bucket        *_type // internal type representing a hash bucket
    	hmap          *_type // internal type representing a hmap
    	keysize       uint8  // size of key slot
    	indirectkey   bool   // store ptr to key instead of key itself
    	valuesize     uint8  // size of value slot
    	indirectvalue bool   // store ptr to value instead of value itself
    	bucketsize    uint16 // size of bucket
    	reflexivekey  bool   // true if k==k for all keys
    	needkeyupdate bool   // true if we need to update key on an overwrite
    }
    ```

14. 对于_type，我们称之为类型描述器 `\_type.alg` field`   就是用来定义 hash和equal函数的

    ```go
    type _type struct {
    	size       uintptr
    	ptrdata    uintptr // size of memory prefix holding all pointers
    	hash       uint32
    	tflag      tflag
    	align      uint8
    	fieldalign uint8
    	kind       uint8
    	alg        *typeAlg
    	// gcdata stores the GC type data for the garbage collector.
    	// If the KindGCProg bit is set in kind, gcdata is a GC program.
    	// Otherwise it is a ptrmask bitmap. See mbitmap.go for details.
    	gcdata    *byte
    	str       nameOff
    	ptrToThis typeOff
    }
    ```

    ```go
    // typeAlg is also copied/used in reflect/type.go.
    // keep them in sync.
    type typeAlg struct {
    	// function for hashing objects of this type
    	// (ptr to object, seed) -> hash
    	hash func(unsafe.Pointer, uintptr) uintptr
    	// function for comparing objects of this type
    	// (ptr to object A, ptr to object B) -> ==?
    	equal func(unsafe.Pointer, unsafe.Pointer) bool
    }
    
    ```






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

#### others

1. go 不支持参考变量，也就是说，Go不支持多个变量值指向同一个地址的操作。

