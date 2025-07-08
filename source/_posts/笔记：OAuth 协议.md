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
# 一、理论

## 1. 导图

![](source/_posts/笔记：OAuth%20协议/Map：OAuth%20协议.xmind)

---
## 2. OAuth2 协议

Auth 表示授权（Authorization），O 表示开放（Open），OAuth2 是一种开放式的授权协议，用于在不暴露用户凭据的前提下，让第三方应用获得访问权限。

![](image-20250706092222409.png)

> [!NOTE] 注意事项
> 1. 客户端应用可能是：前端、后端、移动 APP、微信小程序等

---


## 3. OAuth2.1 协议

OAuth 2.1 目前仍处于草案阶段，但正逐渐被广泛接受，有望成为主流。其较于 OAuht2 的主要变化包括：
1. 对授权码模式（Authorization Code）进行了增强，引入了 PKCE（授权码交换的密钥证明）。
	1. 需要注意的是，OAuth 2.1 明确指出：**只要使用授权码模式，必须同时使用 PKCE**，以提升安全性
2. 移除了不再安全的隐藏式（Implicit）和密码式（Password）授权模式，新增了刷新令牌模式（Refresh Token），OAuth 2.1 最终保留三种主流授权模式：
	1. 授权码模式（Authorization Code）+ PKCE（授权码交换的密钥证明）
	2. 客户端凭证模式（Client Credentials）
	3. 刷新令牌模式（Refresh Token）

> [!NOTE] 注意事项
> 1. Spring Security 目前既能支持 OAuth 2.1 的三种主流模式，也仍保留对旧有隐式和密码模式的兼容支持
> 2. 这是因为很多老系统还在用这两种不安全的授权模式，但官方安全建议是新项目应避免使用隐式和密码模式，并且尽量使用授权码 + PKCE 来保障安全

----


## 2.2. OAuth2 中的角色

![](PixPin_2025-07-06_16-38-42.png)

----


### 授权码模式（Authorization Code）+ PKCE（授权码交换的密钥证明）+OIDC（身份认证协议）

OAuth2 的授权码模式（Authorization Code Grant）是目前最常见、最安全、同时也是实现最复杂的一种授权方式，尤其适用于前后端分离的系统架构。

以 Gitee 使用第三方 GitHub 登录为例，在整个授权流程中，Gitee 作为 OAuth 客户端，而 GitHub 同时提供授权服务器与资源服务器的角色。整体流程如下所示：

<span style="background:#9254de">1. Gitee 前端页面发起请求，并跳转到 GitHub 授权页</span>
前端的行为可能如下所示：
```
GET https://github.com/login?
    client_id=5a179b878a9f6ac42acd
    &return_to=/login/oauth/authorize?
        client_id=5a179b878a9f6ac42acd
        &redirect_uri=https%3A%2F%2Fgitee.com%2Fauth%2Fgithub%2Fcallback
        &response_type=code
        &scope=user
        &state=81fda4a8066fb1ea3310d3bf577ece61a8e0286c03f82c91
“”“
1. client_id
	1. Gitee 向 GitHub 开放平台注册应用时获得的 Client ID。告诉 GitHub：“我是哪个客户端”
2. return_to
	1. 登录成功后 GitHub 要跳转的页面（即 OAuth 授权页）
3. redirect_uri
	1. GitHub 授权后将跳转回 Gitee 的地址（回调地址）
4. response_type
	1. 告诉 GitHub 要使用哪种授权模式，这里是 “授权码模式”
5. scope
	1. 请求的权限范围，这里是访问你的 GitHub 基本 User 信息
6. state
	1. Gitee 随机生成的 CSRF Token，用于 Gitee 防止 CSRF 攻击，回调时 GitHub 会原样返回（不是 GitHub 防止 CSRF 攻击）
”“”
```

Gitee 的前端页面通过 `window.location.href` 跳转到 GitHub 提供的登录页面。用户在 GitHub 成功登录后，才会被引导进入授权页面（先登录才能授权，不登录你怎么授权）
![](image-20250707084023923.png)

![](image-20250707085858048.png)

![](image-20250707085938292.png)

> [!NOTE] 注意事项
> 1. 在授权页面中，一些应用的授权服务器会明确列出授权项，例如：“是否允许 Gitee 访问你的基本信息”、“是否允许 Gitee 访问你的电话号码”、“是否允许 Gitee 访问你的邮箱地址”等。这些授权项通常由客户端申请的 `scope` 决定，并由授权服务器展示给用户确认


<span style="background:#9254de">2. GitHub 授权后，携带授权码重定向回 Gitee 后端地址</span>
GitHub 在用户授权后，会将页面重定向到 Gitee 预先注册并指定回调的**后端地址**。
![](image-20250707090310873.png)

虽然我们在这里看到了一个页面，但它本质上仍然是由后端控制器返回的。其可能是采用了 **Spring Boot + Thymeleaf** 的服务端渲染方式，这意味着：**Spring Boot 既负责处理后端逻辑，也负责返回 HTML 页面**。对应的代码可能如下所示：
```
@Controller
@RequestMapping("/auth/github")
public class GitHubOAuthController {

    @GetMapping("/callback")
    public String githubCallback(
            @RequestParam("code") String code,
            @RequestParam("state") String state,
            HttpSession session,
            Model model) {

        // 1. 验证 CSRF Token（state），防止跨站请求伪造

        // 2. 使用 code 调用 GitHub 授权服务器 API，获取 Access Token

        // 3. 使用 Access Token 调用 GitHub 资源服务器 API，获取用户信息

        // 4. 根据 GitHub 返回的用户信息判断是否已绑定 Gitee 手机号

        if (isBound) {
            // 用户已绑定，查询并同步本地用户信息
            // 然后重定向到 Gitee 首页
            return "redirect:/home";
        } else {
            // 用户未绑定，渲染绑定手机号页面
            model.addAttribute("githubUser", user);
            return "bind-phone";  // 绑定手机号的 Thymeleaf 模板
        }
    }
}
```

> [!NOTE] 注意事项
> 1. 实际开发中，不建议在 Controller 中编写过多业务逻辑。你可以简单地理解：Controller 的职责是接收请求、协调流程并返回结果，而真正的业务处理应交由 Service 层完成


<span style="background:#9254de">3. Gitee 后端携带 code，调用 GitHub 授权服务器 API，获取 Access Token</span>
其实我们可以注意到一个问题：在发起授权请求时，`Client Id` 是直接暴露在 URL 中的；授权成功后，GitHub 返回的 `code` 也同样出现在回调的 URL 中。那有没有可能，如果这些 URL 被第三方窃取，攻击者就能利用这些信息伪造客户端、换取 Access Token，甚至获取用户的私密信息？

如果只看表面，的确存在这样的风险。但是，别忘了我们在注册第三方应用时，一般会获得两个核心凭证：`Client Id` 和 `Client Secret`。

其中，`Client Secret` 正是在这个阶段起作用的：后端在用 `code` 向授权服务器（如 GitHub）换取 `access_token` 时，**必须携带 `Client Secret` 一并提交**，GitHub 会对其进行校验，以此确认调用方的身份是否合法

> [!NOTE] 注意事项
> 1. 之所以让后端发起换 token 请求，最根本的原因就是：**`client_secret` 绝不能暴露给前端**，一旦被暴露到前端，它就可能被窃取，而攻击者就能冒充你的客户端发起授权，窃取用户数据，这将是非常严重的安全漏洞
> 2. 在后端发起的请求中，一切通信都发生在服务器内部，不会暴露在用户浏览器或网络面前，从而**有效避免了密钥泄露的风险**
> 3. 这是 OAuth2 中授权码模式强调换 token 必须由后端发起的根本原因，也是为什么授权码模式适合前后端分离时使用的根本原因


<span style="background:#9254de">4. GitHub 授权服务器返回 Access Token</span>


<span style="background:#9254de">5. Gitee 后端携带 Access Token，调用 GitHub 资源服务器 API，获取 schope 指定的信息</span>


<span style="background:#9254de">6. GitHub 资源服务器返回对应信息</span>
























# ------------------




#### 3.2. OAuth2.1 三种授权模式

##### 3.2.1. 

---


##### 3.2.2. 客户端凭证模式（Client Credentials）

----


##### 3.2.3. 刷新令牌模式（Refresh Token）

---


#### 3.3. 客户端应用配置

##### 3.3.1. 配置模板

在 OAuth2 Client 中有一个类叫做 `ClientRegistration`，它规定了客户端应用应该配置哪些信息。而对于一些常见的大型网站的 Provider，OAuth2 Client 提供了一个名为 `CommonOAuth2Provider` 的枚举类（enum 类），其中包含了 GOOGLE、GITHUB、FACEBOOK、OKTA 的常用 Provider 的预设配置。你可以通过 `Ctrl + N` 快捷键搜索他们，查看其源码内容。

我们一般在 `application.yaml` 中进行配置，这里是配置模板：
```
spring:
  security:
    oauth2:
      client:
        registration:
          gitee:                                                                            # registration 的自定义名称
            client-id: xxxxxx                                                      # 在 gitee 注册后，返回的 Client ID
            client-secret: xxxxxx                                               # 在 gitee 注册后，返回的 Client Secret
            authorization-grant-type: authorization_code      # 指定授权模式，这里是授权码模式
            redirect-uri: xxxxxx                                                # 回调地址，必须与在 gitee 注册中的一致
            scope:                                                                    # 授权范围，我们在 gitee 注册时，勾选的权限
              - user_info
            provider: gitee                                                       # 对应的 provider 名字，与下面的对应
        provider:
          gitee:                                                                            # provider 的自定义名字
            authorization-uri: xxxxxx                                        # gitee 文档中给我们的，进行认证的地址（授权服务器的 API）
            token-uri: xxxxxx                                                    # gitee 文档中给我们的，用于获取 access_token 的地址（授权服务器的 API）
            user-info-uri: xxxxxx                                              # gitee 给我们的，用户信息的接口地址，即拿到 access_token 后，会请求这个 URI 去获取用户信息（资源服务器的 API）
            user-name-attribute: xxxxxx                                  # 从获取到的用户信息中，取那个字段当作用户名（最终会被封装到 OAuth2User.getName() 中）
```

> [!NOTE] 注意事项
> 1. 现在能有哪些授权模式？
> 2. 详细其他的配置，还是那句话，等你摸明白了原理，去看源码，然后还有一些配置等待你的挖掘
> 3. 这里是精简版：
```
spring:
  security:
    oauth2:
      client:
        registration:
          gitee:
            client-id: xxxxxx
            client-secret: xxxxxx
            authorization-grant-type: authorization_code
            redirect-uri: xxxxxx
            scope:
              - user_info
            provider: gitee
        provider:
          gitee:
            authorization-uri: xxxxxx
            token-uri: xxxxxx
            user-info-uri: xxxxxx
            user-name-attribute: xxxxxx
```

--------


































### 2.4. OAuth2 的四种授权模式

#### 2.4.1. 授权码模式（Authorization Code）


---


#### 2.4.2. 隐藏式（Implicit）

---


#### 2.4.3. 密码式（Password）

---


#### 2.4.4. 客户端凭证模式（Client Credentials）

----


# 二、实操

## 1. 使用 Spring Security 实现授权码模式 + PKCE

### 1.1. 需求定位

如果你只是接入第三方登录平台，比如在登录极客时间时，选择第三方登录方式“使用微信登录”，那么你只需要开发**客户端应用**，例如：
![](image-20250706220734441.png)

如果你希望像微信一样，为其他系统提供登录能力（即让别人用你平台做第三方登录），那你就需要开发**授权服务器**（负责发 token）和**资源服务器**（负责提供受保护资源）

如果你要开发一整套系统（例如企业内部统一认证平台、SaaS 平台等），那么你就需要同时开发：**客户端应用、授权服务器和资源服务器**，以实现完整的 OAuth2 流程。

----


### 1.2. 接入登录平台

#### 1.2.1. 接入 Gitee

##### 1.2.1.1. 注册 Gitee 账号

![](image-20250707104557256.png)

----------


##### 1.2.1.2. 注册客户端应用

一般是需要我们在对应平台的开放平台上，注册自己的客户端应用。 比如，如果我们希望接入微博登录功能，就需要先在微博开放平台上注册我们的应用，Gitee 注册客户端应用的位置如下：
![](image-20250707120251369.png)

![](image-20250707153632809.png)

![](image-20250707122604264.png)

> [!NOTE] 注意事项
> 1. 对于 Gitee 来说，我们是第三方应用，但是对于我们来说，Gitee 才是第三方应用
> 2. Spring Security 规定了回调函数标准前缀为 /login/oauth2/code/xxx，详见：OAuth2LoginAuthenticationFilter 的源码
> 3. 我们需要保存三个东西：
```
// 1. 回调地址
http://localhost:8080/login/oauth2/code/gitee


// 2. Client ID
8f19e249debb105f28e7140ef036273e2a935720b729ff733fcffc73d6f15890


// 3. Client Secret
6c3c5e11b906c5bde6bedf7e667d68a8e453cb3e7aae820c525831990f8e57d1
```

----


##### 1.2.1.3. 查看 Gitee 的 OpenAPI 文档

查看： https://gitee.com/api/v5/swagger#/getV5ReposOwnerRepoSubscribers?ex=no

----


##### 1.2.1.4. 创建 Spring Web 项目，添加相关依赖

创建时：
1. Web：
	1. Spring Web
2. Security：
	1. Spring Security
	2. OAuth2 Client
3. SQL
	1. JDBC API
	2. MyBatis Framework
	3. MySQL Driver

----


##### 1.2.1.5. 前置步骤

和基于 JWT 的 Spring Security 的步骤 1.2.2 ~ 1.2.7 近似一致，详见笔记：Spring Security

-----


##### 1.2.1.6. 进行客户端应用配置

----




















#### 1.2.2. 接入微信







-----


### 1.3. 开发登录平台，供其他应用接入

## Spring Authorization Server 概述

Spring Authorization Server 是由 Spring 团队官方推出的一个授权服务器框架，用于实现 OAuth2 授权协议以及 OpenID Connect（OIDC）身份层协议。

----


# 二、实操

## 使用 Spring Security 实现授权码模式 + PKCE +OIDC

### 开发 授权服务器 + 资源服务器（登录平台）

#### 开发 授权服务器

##### 1.2.1.4. 创建 Spring Web 项目，添加相关依赖











































-----


### 1.4. 开发一条龙服务系统

#### 1.4.1. 开发客户端应用

---


#### 1.4.2. 开发授权服务器

----


#### 1.4.3. 开发资源服务器

-----









