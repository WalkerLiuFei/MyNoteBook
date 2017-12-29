# Freemarker学习笔记

# 模板基础

+ 插值 `${....}` :解析变量输出真实的值
+ FTL标签 以`#`开头，用户自定义标签以 `@`开头
+ 注释`<#!-- 注释内容-->`
+ directives 指令：就是所谓的FTL标签，这些指令在和HTML中的元素中关系是一致的
 + ` <#if xxx> xxxx </#if>` 和 `<#if 条件> xxx  <else> </#if>`
 + include 指令： 例如 `<# include "/xxxx.html">` 这样会将这个文件解析添加到这个ftl文件中
+ 各种数据类型的build-in
 + **List 集合** 指令 
    <pre> 
     <#list animals as being>
     <tr><td>${being.name}<td>${being.price} Euros
     </#list> 
     //会被解析成下面的
    <tr><td>mouse<td>50 Euros
    <tr><td>elephant<td>5000 Euros
    <tr><td>python<td>4999 Euros
       </pre>
 + **Map类型：**遍历Map:Map mushat
    <pre> 
    	<#list myMapper keys as k >
    		${k} : ${myMapper[k]}
    	</ #list>
     </pre>


 + **自定义Bean**  通过调用属性名字的来访问相应属性



	<#if person??>
	   name : ${person.name} 
	   age  : ${person.age}
	</#if>

+ 处理不存在的变量 
 + 如果user 变量存在的话，会将括号括中的标签内容解析显示出来

```
<#if object??> 
	object exist
<#else>
       object not exist
<#/if>		
```


 + 如果 user 变量不存在 "anonymous" 会替代填充 `<h1>Welcome ${user!"Anonymous"}!</h1>`
+ 子程序 P:
 + 方法和函数
+ 用户自定义指令
 + **宏指令：**


# Programmer Guild

+ 在Freemarker模板中，可用的变量都是实现了freemarker.template.TemplateModel接口的Java对象。你自己的数据模型（例如你自定义的 一个JavaBean）会被替换成TemplateModel类型。这种特性叫做**Object Wraaper 对象包装**
+ 各个Freemarker支持的标量哎Java层的实现 
 + String ： SimpleScalar
 + Hash : SimpleHash,在FTL 中，容器是一成变的。就是说，你不能替换和移除容器中的子变量
 + List : SimpleSequence 
 + Collection : SimpleCollection
 + Date : SimpleDate
 + Method : 

## 定义实现自定义宏



## 定义实现自定义方法

1. 首先通过继承`TemplateMethodModelEx`实现自己要定义的方法类
  <pre>
  public class IndexMethod implements TemplateMethodModelEx {
      @Override
      public Object exec(List arguments) throws TemplateModelException {
          if (arguments.size() != 2) {
              throw new TemplateModelException("Wrong arguments");
          }
          int  result = -1;
          if (!(arguments.get(0) instanceof SimpleScalar &&arguments.get(1) instanceof SimpleScalar))
              return new SimpleNumber(result);
          String first = ((SimpleScalar)arguments.get(0)).getAsString();
          String second  = ((SimpleScalar)arguments.get(1)).getAsString();
      
          //arguments.stream().forEach(element -> System.out.println(element));
          return new SimpleNumber(second.indexOf(first));
      }
  }
  </pre>
2. 在向template 传值时，例如将上面的 类
  <pre>
  Map root = new HashMap();
  root.add("indexOf",new IndexMethod());
  </pre>
3. 在flt 中这样用
  <pre>
  <#assign x = "something">
  ${indexOf("met", x)}
  ${indexOf("foo", x)}
  </pre>

# FAQ

+ **SimpleHash VS DefaultMapAdapter**
 + SimpleHash相对于 DefalutMapAdapter 来说，效率更高
 + 如果你从Template Entry只取出2次以下时，DefaultMapAdapter的效率高于SimpleHash.相反的就应该使用DefaultMapAdapter 

## Freemarker Cache