# 深入理解 JVM第七章笔记

## 类的加载
**类的生命周期**
1. 加载
2. 连接
   1. 验证
   2. 准备
   3. 解析
3. 初始化
4. 使用
5. 卸载

##### 有且仅有下面的条件会导致累的初始化
+ 遇到对类进行`new`，`getstatic`,`putstatic`,`invokestatic` 时
+ 反射调用
+ 一个类初始化时，要求父类必须初始化，
+ main方法所在的类
+ 调用java 1.7以后的动态语言支持的功能时


```
class SuperClass{
    static {
        System.out.println("SuperClasss init");
    }
    public static String value = "123";
    //
    public static final String value2 = "1234";
}
class SubClass extends SuperClass{
    static {
        System.out.println("SubClass init");
    }
}
public class Classloader1 {
    public static void main(String[] args){
        //常量在编译阶段会存入调用类的常量池中，本质上并没有直接引用到定义常量的类，
        // 这样调用不会触发定义常量的类的初始化
        System.out.println(SubClass.value2);
        // 这样调用也不会初始化SubClass 或者SuperClass
        SubClass[] subClasses = new SubClass[]{};
    }
}

```