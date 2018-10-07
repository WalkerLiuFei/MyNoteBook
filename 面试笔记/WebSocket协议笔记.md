---
title:WebSocket协议笔记
---

# 握手

## 客户端发出的握手信息类似：


```
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Origin: http://example.com                //源标示
Sec-WebSocket-Protocol: chat, superchat   //子协议选项
Sec-WebSocket-Version: 13               
```

+ 第5行：sec-WebSocket-Key: 
+ 第6行：Origin头，表明源标示
+ 第7行：Sec-WebSocket-protocol。子协议选项，用于标示客户端和服务端使用的是哪一种子协议 

## 服务端回应的握手信息类式：

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocol: chat
```

+ 第4行：Sec-WebSocket-Accept标示server是否接受了client的请求。这个值必须要不为空，并且符合client Request请求的规则才算是满足了请求
+ 第5行：返回的子协议，必须是client请求时携带的子协议之一
+ 服务端的返回也可以携带cookie信息

## 其他 

+ WebSocket类似于Http，同样是一个基于TCP的应用层协议，它和HTTP的关系就是请求可以作为一个升级请求（Upgrade request）经由 HTTP 服务器解释（也就是可以使用 Nginx 反向代理一个 WebSocket）。

+ 握手信息的头字段是没有顺序要求的
+ 关闭握手的操作可以是任何一端主动发出，因为WebSocket的通信本就是全双工的

