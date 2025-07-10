---
title: 笔记：OAuth 协议
date: 2025-07-06
categories:
  - Java
  - 安全框架
  - OAuth 规范
tags: 
author: 霸天
layout: post
password: wq666666
---
## Spring Security OAuth 授权服务器配置

### Spring Security OAuth 授权服务器配置模板


#### 配置表单登录










# 二、实操

## 使用 Spring Security 开发授权码模式 + PKCE + OIDC + Refresh Token 一条龙系统

### 开发授权服务器

#### 创建 Spring Web 项目，添加相关依赖

创建时：
1. Web
2. Spring Web
3. Template Engines
	1. Thymeleaf
4. Security
	1. Spring Security
	2. OAuth2 Authorization Server
5. SQL
	1. JDBC API
	2. MyBatis Framework
	3. MySQL Driver

创建后：添加 [spring-security-oauth2-jose 依赖](https://mvnrepository.com/artifact/org.springframework.security/spring-security-oauth2-jose)
```
<dependency>
	<groupId>org.springframework.security</groupId>
	<artifactId>spring-security-oauth2-jose</artifactId>
</dependency>
```

> [!NOTE] 注意事项
> 1. `spring-security-oauth2-jose` 是 Spring 官方为 OAuth2 提供的 JWT 支持模块，而 `jjwt-*` 是 Okta 社区维护的第三方库 JJWT。
> 2. 虽然 JJWT 也能生成 JWT，但它与 Spring Security 的集成度较低，许多功能（如 token 签发、校验、JWK 支持等）都需要我们手动实现。它与我们原先采用的 JJWT 库存在不少差异，不能直接沿用过去的写法，需要根据新库的方式进行修改。
> 3. Thymeleaf 是个模板引擎，负责把模板（html文件）渲染成最终的HTML页面返回给浏览器，只有在 `@Controller` 返回字符串为视图名时，Thymeleaf 才会介入渲染页面。




































