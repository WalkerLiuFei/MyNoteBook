---
title: 你应该了解的Java那些面试点
---

# Java基础

+ **你对ClassLoader的理解**
 + ClassLoader默认提供的三个ClassLoader，Bootstrap, Extension ClassLoader，APPClassLoader，后面这俩其实是第一个加载出来的，第一个是属于JVM的。不是一个层次的东西
 + Bootstrp 负责加载jdk的核心库。
 + Extension ClassLoader：扩展类加载器，负责加载Java的扩展类库
 + APP ClassLoader：系统类加载器，负责加载ClassPath下所有的jar包和Class文件
 + 可以继承ClassLoader实现自己的类加载器 
 + 如果你在去找通过Class类 找ClassLoader 它返回null的时候，说明他的类加载器是BootStrapClassLoader，所以说不要通过java核心库去找ClassLoader 加载配置文件什么的
    `System.out.println(String.class.getClassLoader());`就是输出的null
 + 双亲委派模型,其实就是父类的Class先去加载，父类找不到，再交给子类的ClassLoader加载....这样就不会造成类加载时的冲突
 + 对于不同ClassLoader加载出来的类，虽然它们有可能会是同一个字节码文件(同一个类文件)JVM也会认为他们属于不同的类
 + 利用 java -Xbootstrap/a; 添加bootstrap 需要加载的类的jar包的路径
+ 64位JDK 不变不如32位JDK

+ **谈谈你对Java线程池的理解**
>  + **实现原理**：关键词 `生产者消费者模型`、`HashSet`、`ThreadPoolExcutors`、`额外的任务管理线程`。核心类是 ThreadPoolExcutors，其维护一个corePoolSize，maxPoolSize,缓冲队列和一个HashSet 作为Task集合。内部通过一个BlockQueue作为 work queue维护一个生产消费者模型。线程池其实就是一个HashSet来存放所有的工作集。内部维护了一个名字叫Worker的类，用来管理工作集的执行，换句话说，ThreadPoolExcutors也额外需要一个线程来执行它维护的线程。

+ Syntronized 属于悲观锁，lock属于乐观锁。lock更方便用于负责逻辑的。 使用lock可以方便的控制 thread wait等
+ java中的公平锁和非公平锁
 + 公平锁：先来先得
 + 费公平锁：不管先后顺序，来了就给（看释放时间）。
 + Java中的RetreenLock 默认是非公平锁
+ ConcurrentHashMap 和Collections.synchronized(map)之间的区别：
	**ConcurrentHashMap允许多个线程同时修改Map中的元素，而不是阻塞其中的某个线程.Collections.synchronized(map)是利用阻塞线程来实现的线程安全访问。如果你想及时看到Map中实时更新的元素，使用Collections.Syntronized。如果你的往Map里面插入的操作很多，而读的操作很少。并且想拥有较好的性能。那么使用ConcurrentHashMap**，另外ConcurrentHashMap不允许插入null值。。HashTable和Collections.syntronized(HashMap)的表现是一致的。都是在整个集合上建立一个读写锁。concurrentMap主要关注的是get动作
+ ConcurrentMap中的**分段锁技术**
	+ ConcurrentHashMap是由Segment数组结构和HashEntry数组结构组成。Segment是一种可重入锁ReentrantLock，在ConcurrentHashMap里扮演锁的角色，HashEntry则用于存储键值对数据。
	+ ConcurrentHashMap内部使用段(Segment)来表示这些不同的部分，每个段其实就是一个小的hash table，它们有自己的锁。只要多个修改操作发生在不同的段上，它们就可以并发进行。同样当一个线程占用锁访问其中一个段数据的时候，其他段的数据也能被其他线程访问。
	+ 最后就是读写分离，也就是说对于Map中保存的对象进行读操作是不需要获取锁，只有在对某个元素进行写操作，才需要获取当前的这个元素所在的Segment的锁
+ java dump 文件
 + heap dump:用来分析java对象的内存占用情况 ： jmap -dump:live ,format=b,file=heap.hprof xxx(java 进程号)
 + thread dump：用来分析java应用的执行情况。一般用来解决java执行时的block问题：jstack XXX(java 进程号)
 + 使用工具MAT来分析dump文件是一个比较好的选择
+ java -x m命令

+ java内存模型
 + 线程共享
  + 程序计数器：java执行字节码的指示器
  + Java虚拟机堆：用来为java方法执行时建立的内存模型，包含局部变量，操作数栈，动态链接，方法出口。
 + 线程共享
  + Java堆，用来存放对象，是GC主要区域，分为新生代和老生代
  + 方法区（永久代）：用于存储虚拟机加载的类信息，常量，静态变量等。这个区域的内存回收主要针对的是常量池的回收和对类型的卸载！

<br>

+ JavaGC 算法，可达性分析算法。判断一个对象的引用是否能达到GC Root Set。不能则说明是可以回收的


+ 垃圾回收算法
>
 + 标记清除算法：先对所有的需要回收的对象进行标记，然后统一回收。缺点是会产生大量的内存碎片
 + 复制算法：将可用内存按照容量分成大小相同的两块，每次使用的时候只用其中的一块。使用完毕后，进行一次内存回收。将还存活的对象复制到另一块内存，并对先前的那块内存进行清理。缺点显而易见
 + 分代回收算法：老年代单线程标记整理算法，新生代复制收集算法
 + 可以利用--XX:PretenureSizeThrshold 参数 设定一个阈值，令大于这个阈值的对象直接在老年代分配。这样做的目的是避免Eden区以及Survivor区之间发生大量的内存复制（新生代采用内存复制算法来进行GC）
 + 通过指定MinorGC次数(默认15次)的对象将进入老年代区域
 + 注意区别 full gc ，minor gc  的区别已经出发条件

<br/>

+ **java 中计算hashcode 用 31作为因子?**
 >  首先因为其是一个素数


+ **利用 foreach 循环删除元素时为什么会抛出一个 ConncurrentException ？** (测试环境 jdk 8)
 >
 + 首先，foreach 循环只能遍历 可以Iterable的对象，比如Collections包里面各类集合类和 数组。Iterable 对象的遍历在编译期被编程成 `for(Iterator i = list.iterator;i.next;){...}`,而数组的的编译成（之前会首先有个引用！）， `for(int index = 0;index < arr.length();index ++){...}`
 + 在单线程的情况下并不是真正意义上的并发异常。原因是，可 Iterable 的集合类，内部维护了一个子类Iterator,这个子类有一个 属性 expectCount 这个count是当前list集合元素的数量。如果你利用List 直接去Remove，会造成这个属性值和 list值不一致！如果你在modify list元素数量然后再用Iterator 去 进行next 操作的时候会抛出一个ConcurrentException
+ **InputStream 和Reader的区别**
 + Reader 读取Unicode编码（16位）的字符，InputStream用于读ASCII和二进制字符。
 + 一般用Reader来读取文本文档。明显的，用Reader用来读取少了编解码环节。使得效率更高
 + transient 关键字用来控制属性的序列化，当transient修饰一个属性时，这个属性是不会被JVM序列化的。

