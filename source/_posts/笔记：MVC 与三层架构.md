---
title: 笔记：MVC 与三层架构
date: 2025-04-11
categories:
  - 设计模式
  - MVC 与三层架构
tags: 
author: 霸天
layout: post
---
MVC（Model-View-Controller）是一种常用的软件设计模式，通过将应用程序分为三个主要部分——模型（Model）、视图（View）和控制器（Controller），实现了代码的分离，从而提高了应用的可维护性、可扩展性和可测试性。
1. ==模型（M）==：负责应用程序的业务逻辑和数据处理，通常包括以下两个层次：
    1. <font color="#00b0f0">Service 层</font>：处理具体的业务逻辑和数据管理。
    2. <font color="#00b0f0">DAO 层</font>：负责与数据库进行交互，通常使用 JDBC 或 ORM 框架（如 Hibernate、MyBatis）进行数据持久化操作。
2. ==视图（V）==：负责数据的呈现和用户界面的展示，通常无需关注业务逻辑，仅关注如何展现数据。
    1. 在传统的 Java Web 应用中，JSP 用于动态生成 HTML 页面。
    2. 在现代前端应用中，框架如 React、Vue 或 Angular 通常负责构建用户界面，开发者更多关注前端交互逻辑。
3. ==控制器（C）==：MVC 的核心部分，负责协调模型层和视图层之间的交互。通常包括以下组件：
    1. <font color="#00b0f0">控制器（Servlet）</font>：处理 HTTP 请求，调用模型层的业务逻辑，并返回 HTTP 响应。
    2. <font color="#00b0f0">过滤器（Filter）</font>：在请求到达控制器之前进行预处理，如身份验证或日志记录。
    3. <font color="#00b0f0">监听器（Listener）</font>：监控应用程序或请求生命周期中的事件，如上下文初始化、销毁等。

![](source/_posts/笔记：MVC%20与三层架构/image-20250411213131995.png)

> [!NOTE] 注意事项
> 1. Service 接口本身不需要添加 `@Service`，只需要在 ServiceImpl 上添加
> 2. 同样的，Dao 接口本身不需要添加 `@Repository`，只需要在 DaoImpl 上添加，可由于代理的使用，我们可以选择在 Dao 接口本身添加，或者直接不添加。
> 3. 不要忘了，实现类要去实现接口，要 `implements`


















