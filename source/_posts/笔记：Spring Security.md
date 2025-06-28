---
title: 笔记：Spring Security
date: 2025-05-18
categories:
  - Java
  - Spring 生态
  - Spring Security
tags: 
author: 霸天
layout: post
---
# 一、理论

## 导图

![](source/_posts/笔记：Spring%20Security/Map：SpringSecurity.xmind)

---


## 3. Spring Security 登录图示

![](image-20250628210023140.png)

---


## Spring Security 执行流程

<span style="background:#fff88f">1. 用户请求（客户端请求）</span>
每次用户访问受 `Spring Security` 保护的资源，都会经过以下流程


<span style="background:#fff88f">2. SecurityContextPersistenceFilter 介入</span>
自动为本线程初始化 `SecurityContextHolder` 并根据 `JSESSIONID` 向 `HttpSession` 查找 `SecurityContext`（其内保存最重要的 `Authentication`）
1. 若存在 `SecurityContext`，便将其加载到本线程的 `SecurityContextHolder` 中（基于 HttpSession 实现 “记住我” 功能，我们也可基于 JWT 实现 “记住我” 功能）
2. 若不存在 `SecurityContext`，则在本线程中自动初始化一个新的 `SecurityContext`

即使我们不打算通过 `HttpSession` 实现 “记住我” 功能（如使用 JWT），甚至完全不使用 `HttpSession`，我们仍然建议保留这个过滤器，因为它自动为本线程初始化 `SecurityContextHolder`、并自动创建 `SecurityContext`，这个能力实在太香了。
![](image-20250628224744251.png)


<span style="background:#fff88f">3. UsernamePasswordAuthenticationFilter 介入</span>
该过滤器主要用于前后端未分离的场景，用于处理默认 `/login` 路径下的登录请求。  

在前后端分离的架构中无需深入关注其具体逻辑，只需了解其在过滤器链中的位置，以便在插入自定义过滤器时能准确定位。


<span style="background:#fff88f">4. AnonymousAuthenticationFilter 介入</span>
如果当前没有任何 `Authentication`，系统会自动创建一个匿名身份，以避免后续流程中出现空指针异常。

> [!NOTE] 注意事项
> 1. 



<span style="background:#fff88f">4.FilterSecurityInterceptor 介入</span>
首先检查当前线程中是否存在 `Authentication` ，如果不存在，则抛出 `AuthenticationException`（表示未认证）

接着检查当前权限是否有权访问对应的资源或方法（即资源级别的访问控制、方法级别的访问控制），若无权限，则抛出 `AccessDeniedException`
> [!NOTE] 注意事项
> 1. 整个流程中的异常由 `ExceptionTranslation` 过滤器统一处理，负责捕获**整个过滤器链中**抛出的 `AuthenticationException` 和 `AccessDeniedException` 异常，并执行相应的处理逻辑。
> 2. 我们的 /public 这些，时限制的必须要认证的用户，即认证的 authentication


5.执行 API


6.SecurityContextPersistenceFilter 再次介入
自动将本线程的 `SecurityContext` 存入 `HttpSession`，以便在后续请求中维持用户身份（需要手动开启）

 随后，过滤器会清空本线程 `SecurityContextHolder`，防止 `SecurityContext` 在后续请求中被无意复用，从而确保每个请求都能独立执行认证和授权流程。
















x


















