---
title: 笔记：JDBC
date: 2025-04-10
categories:
  - 数据库
  - 关系型数据库（SQL）
  - 同步阻塞规范
  - JDBC
tags: 
author: 霸天
layout: post
---
# 一、理论

### 1. 导图：[Map：JDBC](../../maps/Map：JDBC.xmind)

---


### 2. JDBC 概述

JDBC（Java Database Connectivity）是一组统一的接口，允许 Java 应用程序通过调用这些接口与不同的数据库进行交互。简而言之，Java 调用 JDBC 接口，接口再执行 SQL 语句，从而实现与多种数据库的通信。

---


# 二、实操

### 1. 引入相关依赖
```
<!-- 1.MySQL JDBC 驱动依赖（数据库驱动依赖），使用那个数据库就添加那个数据库的驱动依赖 -->
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.33</version> 
</dependency>

<!-- 2.Druid 连接池 -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid</artifactId>
    <version>1.2.15</version>
</dependency>
```

---




















