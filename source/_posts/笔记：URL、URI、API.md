---
title: 笔记：URL、URI、API
date: 2025-07-08
categories:
  - 网络
  - URL、URI、API
tags: 
author: 霸天
layout: post
---
## 1. URL、URI、API 的区别

例如一个前端地址：`https://gitee.com/wangza`，它是一个 URL，不是一个 API（API 是接口功能，是需要程序调用的），但它能唯一标识一个前端资源，因此它是 URI。  

再看一个后端地址：`https://gitee.com/api/v5/users/wangza`，它是一个 URL，也是一个 API，同时它能唯一标识一个后端资源，所以它是 URI。  

再看一个后端相对路径：`/api/v5/users/wangza`，它不是一个 URL，但它是一个 API，而且它同样能唯一标识一个资源，因此它是 URI。

能唯一标识一个资源的路径，就叫 URI，我们只需要记住这张表：

| 地址                                      | 是 URL？ | 是 URI？ | 是 API？ | 说明     |
| --------------------------------------- | ------ | ------ | ------ | ------ |
| `https://gitee.com/wangza`              | ✅      | ✅      | ❌      | 前端网页地址 |
| `https://gitee.com/api/v5/users/wangza` | ✅      | ✅      | ✅      | 后端接口地址 |
| `/api/v5/users/123`                     | ❌      | ✅      | ✅      | 后端相对路径 |

----


## 2. URL

URL（统一资源定位符），俗称网址，一个完整的 URL 示例可能是：
```
https://www.example.com:443/folder/file.html?key1=value1&key2=value2#section1
```

包括以下几部分：
1. 协议（Scheme）：
	1. 指定访问资源所使用的协议，例如 http、https、ftp 等。
2. 主机名（Host）：
	1. 指定资源所在的服务器，通常是域名（如 `www.example.com`）或 IP 地址。
3. 端口号（Port）：
	1. 可选部分，指定访问服务器的端口，默认情况下，HTTP 使用 80，HTTPS 使用 443。
4. 资源路径（Path）：
	1. 指定服务器上资源的具体位置，例如 /folder/file.html。
5. 查询字符串（Query）：
	1. 可选部分，通常用于传递参训参数，格式为 `key1=value1&key2=value2`。  

* * *

## 3. RESTful URL

RESTful URL 是一种符合 REST（Representational State Transfer）架构风格的 URL 设计方式，旨在通过简单、统一的 HTTP 请求来操作资源。

其设计原则为：用 HTTP 请求的操作类型表示不同的操作，没有多余的动词，不搞 `/getUser`、`/updateUser` 这种 RPC 风格，例如：

| 动作     | HTTP 方法 | 路径（URL）      | 说明              |
| ------ | ------- | ------------ | --------------- |
| 获取所有用户 | GET     | `/users`     | 查询用户列表          |
| 获取一个用户 | GET     | `/users/123` | 查询 ID 为 123 的用户 |
| 创建新用户  | POST    | `/users`     | 添加新用户           |
| 更新用户信息 | PUT     | `/users/123` | 修改 ID 为 123 的用户 |
| 删除用户   | DELETE  | `/users/123` | 删除用户            |
