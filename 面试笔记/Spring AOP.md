---
title: Spring AOP编程 
---

## AOP
### AOP概念
+ 切面：切入系统的一个切面，比如事务管理是一个切面，权限管理也是一个切面
+ 连接点：进行横向切入的位置
+ 通知：切面在某个连接点执行的操作
 + Before advice
 + After returning advice
 + After throw exception
 + After advice
 + Around advice
+ 切点: 通知定义了 "什么"和 "何时"，切点定义了"何处"。切点会匹配通知要织入的一个或多个连接点
通常使用明确的的类或者方法来指定这些切点。  作用是通知被应用的位置
+ 引入：引入允许我们向现有的类中添加方法活属性
+ 织入：切面在指定的连接点被织入到目标对象中，在目标对象中的生命周期中有多个点可以织入
 + 编译器：需要特殊的编译器，例如AspectJ的织入编译器
 + 类加载期
 + 运行期：切面在应用运行期间的某个时刻织入。一般情况下，在织入切面的时候，AOP容器会为目标对象

### Spring AOP 笔记
+ Spring 中的AOP都是动态代理的AOP，即通过反射实现的。
+ <p style = "color : #f00">要导入aspectJrt 和aspectJrtweaver 两个依赖的jar包</p>
+ ![](e3cae7de-448b-4c69-aee2-94032a9b9d1b_128_files/d6640a43-6df6-47cf-a644-06a82e418ad6.png)
+ 运算符 还有 "||" "!" "&&" 三种操作运算符
+ 声明的切面类，要使其成为一个bean（一般是这样做的）
+ 利用@Pointcut("") ...声明一个切点
+ @Around 注解adviced 的方法一定要带一个ProceedingJoinPonit参数,这个参数在执行proceed()方法后会移动继续进行下面的advice...（如果这个proceed() 方法没有被执行，那么被Adviced的方法将不会被执行！！）其实ProceeingJIi


---
+ 注解arg("----"),注意，在应用时，要保证advice method 的参数个数和名字要个args() 里面的保持一致
+ within（""） ，可以接受包或者类的路径，用来限制advice方法在 只有在指定的这个包或者类下面的adviced方法被调用时才会被唤起
+ target 后面跟一个完整的类名，用来确定一个目标类
