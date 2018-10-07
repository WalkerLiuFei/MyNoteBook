---
title: Go Server(一)
---

# Go Web的工作方式

## 概念

+ Rquest : 包含有用户的请求信息，用来解析用户的请求信息，包括Method ，header，body等
+ Response: 服务器需要反馈给客户端的信息
+ Conn：用户的每次请求链接
+ Handler: 处理请求和生成返回信息的处理逻辑

+ 用户的每个连接都会交给一个新的goroutine 去处理。相互没有影响



![](https://astaxie.gitbooks.io/build-web-application-with-golang/content/zh/images/3.3.illustrator.png?raw=true)