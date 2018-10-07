---
title:lambda 表达式笔记
---
## 遍历

+ 以前的循环方式

```Java  

for (String player : players) {  
     System.out.print(player + "; ");  
}  

```
  
+ 使用 lambda 表达式以及函数操作(functional operation)

``` Java 
players.forEach((player) -> System.out.print(player + "; "));

```  
   
+ 在 Java 8 中使用双冒号操作符(double colon operator)

```  
players.forEach(System.out::println);
```   

## 匿名内部类

+ 原来的表达式

```  
btn.setOnAction(new EventHandler<ActionEvent>() {  
          @Override  
          public void handle(ActionEvent event) {  
              System.out.println("Hello World!");   
          }  
    });  
```

+ lambda表达式

```  
btn.setOnAction(event -> System.out.println("Hello World!"));
```  

## 集合操作
  
+ 排序

``` Java  
Arrays.sort(players, (String s1, String s2) -> (s1.compareTo(s2)));
```  
+ 筛选

## 集合操作（Stream）

创建一个简单的集合作为例子

```Java
	List<TestBean> beans = new ArrayList<TestBean>();
	int count = 0;
	while (count++ < 50)
	    beans.add(new TestBean(count,"我的是ID为"+ count +"的成员"));
```

### filter

filter比较简单，例如下面的例子。遍历所有id号大于20的元素并输出
 
``` Java
beans.stream()
        .filter(testBean ->(testBean.getId()>20))
        .forEach(testBean -> {
            System.out.println(testBean);
        });
```

###  map

将testBean 映射为它的content.

```Java
beans.stream()
        .filter(testBean ->(testBean.getId()>20))
        .map(TestBean::getContent)
        .forEach(testBean -> {
            System.out.println(testBean); //输出的的内容为testbean的content内容
        });
```

