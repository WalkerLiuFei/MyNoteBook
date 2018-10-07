---
title: Web 相关
---

# 怎样管理Session


1. 服务器端的产生Session ID ： 
2. 服务器端和客户端存储Session ID ： 客户端（Brower 一般保存在 Cookie 中，Android 这边一般就是保存到SP文件中），Server端一般就是保存到Redis中，不需要做持久换处理
3. 从HTTP Header中提取Session ID ： 提取Cookie 中的键值对
4. 根据Session ID从服务器端的Hash中获取请求者身份信息
5. 如果客户端Cookie被禁用，那么利用重写 URL的方式来标记客户端
利用缓存数据库Redis来管理Session
+ 键-值存储模型
+ 读写速度
+ 支持数据过期和清除
+ 部署简单，语法简单

## 分布式Session的实现机制
## Session 安全性问题

