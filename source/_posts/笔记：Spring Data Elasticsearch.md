---
title: 笔记：Spring Data Elasticsearch
date: 2025-04-22
categories:
  - Java
  - Spring 生态
  - Spring Data
  - Spring Data Elasticsearch
tags: 
author: 霸天
layout: post
---
# 实操

### 创建 Spring Web 项目并添加 ES 相关依赖

1. ==Web==
	1. Spring Web
2. ==NoSQL==
	1. Spring Data Elasticsearch

---


### 配置 Elasticsearch 连接

```
spring:  
  elasticsearch:  
    uris: https://192.168.136.8:9200  
    username: elastic  
    password: wq666666  
    connection-timeout: 5s  
    socket-timeout: 30s
```













