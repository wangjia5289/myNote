---
title: 笔记：Spring Authorization Server
date: 2025-07-09
categories:
  - Java
  - Spring 生态
  - Spring Authorization Server
tags: 
author: 霸天
layout: post
---
# 一、理论


## Spring Authorization Server 执行流程











## Spring Authorization Server 配置

### Spring Authorization Server 配置模板

AuthServerConfiguration 类在 `com.example.oauthserverwithmyproject.configuration` 包下，直接粘贴这份配置模板，再根据下方的详细说明按需进行调整。









### 配置表单登录

<font color="#92d050">1. form.loginPage</font>
指定自定义登录页面的 URI，默认是 /login，但是如果我们写了这个 form.loginPage，他就用我们自己的页面了，即便还是 /login，我们也需要书写自定义的 login 页面
```
form.loginPage("/login")
```

在 OAuth 中，一般是配置成，当抛出 AuthenticationException 异常时，证明用户还未进行登录（不能使用permitall，要使用authenticated，因为用户必须进行登录）的时候，我们在配置 ExceptionTranslationFilter 的时候，需要将页面重定向到我们这里指定的登录页面，例如这样做：
```
.exceptionHandling(handler -> {  
    handler  
		.authenticationEntryPoint(new LoginUrlAuthenticationEntryPoint("/login"));  
})
```

> [!NOTE] 注意事项
> 1. 如果我们没有配置 `ExceptionTranslationFilter` 的行为，即没有配置 `.exceptionHandling` ，它默认就是跳转到我们配置的这 `form.loginPage("/login")`，但是如果你配置了，他就不能默认了，你需要手动进行配置


<font color="#92d050">2. form.loginProcessingUrl</font>
指定处理登录提交的请求 URI，默认也是 `/login`，如果你自定义登录页面，那你发送表单的时候，就应该指向这个地址，Spring Security 会自动拦截这个地址并进行登录，例如：
```
.loginProcessingUrl(“"/process-login")”)


<form method="post" action="/login">  
    <input type="text" name="username" placeholder="用户名" required autofocus />  
    <input type="password" name="password" placeholder="密码" required />  
    <input type="submit" value="登录" />  
</form>
```

> [!NOTE] 注意事项
> 1. 你可能会问，默认的登陆页面也是 /login，默认的处理登录请求的，也是 /login，那他们没有冲突吗？其实他们是根据 HTTP 请求的方法类型判断的，GET 请求就是返回页面，POST 请求就是用来处理登录请求的


<font color="#92d050">3. form.defaultSuccessUrl</font>
设置登录成功后重定向的默认 URI，传入 true 标识总是重定向到该 URI，传入 false 会重定向到用户之前访问的那个页面，但是被我们重定向到登录页面的那个页面
```
.defaultSuccessUrl("/dashboard", true)
```


<font color="#92d050">4. form.successHandler</font>
自定义登录成功的处理逻辑。
```
.successHandler((request, response, authentication) -> {
	// response.sendRedirect("/home");
	response.getWriter().write("登录成功！");
})
```


<font color="#92d050">4. form.failureUrl</font>
登录失败后跳转的页面，这里是 跳转到 /login 页面，error 是我们携带的参数，只不过这个参数没有赋值（有的参数不需要赋值，只要存在就够了）
```
.failureUrl("/login?error")
```


<font color="#92d050">5. form.failureHandler</font>
自定义登录失败的处理逻辑
```
.failureHandler((request, response, exception) -> {
	// response.sendRedirect("/login?error=" + exception.getMessage());
	response.getWriter().write("登录失败：" + exception.getMessage());
})
```


<font color="#92d050">6. form.usernameParameter</font>
指定用户名表单参数名，默认是 `username`，我们配置的 `form.loginProcessingUrl` 显示要根据我们配置的表单参数名拿到参数才行呢
```
.usernameParameter("myUsername")
```


<font color="#92d050">7. form.passwordParameter</font>
指定密码表单参数名，默认是 `password`。
```
.passwordParameter("myPassword")
```


<font color="#92d050">8. form.permitAll</font>
允许所有用户访问登录页和登录处理接口，通常必加，否则未登录用户无法访问登录页，你的确可以配合 `.authorizeHttpRequests()` 明确放行，但 `.formLogin().permitAll()` 是专门针对表单登录机制设计的快捷方式，写上更稳妥。

`.formLogin().permitAll()` 既保证了 URL 放行，也保证了登录流程相关过滤器链的正常执行，这个放行权限不是单纯 URL 授权能替代的。

> [!NOTE] 注意事项
> 1. 禁用表单登录：
```
.formLogin(form -> {form.disable();})
```





# 二、实操





















`OAuth2AuthorizationServerConfiguration.applyDefaultSecurity(http)` 是 Spring Security OAuth2 授权服务器配置中的一个关键方法，它为授权服务器设置了一套默认的安全过滤器链。这些过滤器用于保护授权服务器的端点（如授权端点、令牌端点等），并确保 OAuth2 和 OIDC 协议的正确执行。下面我将详细梳理它启用了哪些过滤器，以及每个过滤器的作用，用通俗易懂的语言讲解。

---

## 一、整体作用
`OAuth2AuthorizationServerConfiguration.applyDefaultSecurity(http)` 方法的作用是为授权服务器配置一个默认的 `SecurityFilterChain`，专门处理与 OAuth2 授权服务器相关的请求（比如 `/oauth2/authorize`、`/oauth2/token` 等端点）。它通过 Spring Security 的过滤器机制，确保这些端点安全、可靠，并符合 OAuth2/OIDC 协议的标准。

这个方法会启用一组过滤器，共同完成以下任务：
1. 保护授权服务器的端点（比如要求用户登录）。
2. 处理 OAuth2 授权流程（比如授权码、令牌发放）。
3. 支持 OIDC 协议（比如生成 ID Token）。
4. 提供安全防护（防止 CSRF、验证请求等）。

---

## 二、启用的过滤器
`OAuth2AuthorizationServerConfiguration.applyDefaultSecurity(http)` 并不是直接列出所有过滤器，而是通过 Spring Security 的默认配置和 OAuth2 模块，间接应用了一组过滤器。这些过滤器主要来自 Spring Security 的核心过滤器链和 OAuth2 授权服务器的专用过滤器。以下是可能启用的主要过滤器，以及它们的作用：

### 1. **SecurityContextPersistenceFilter**
- **作用**：管理 `SecurityContext`（安全上下文）的存储和加载。
- **通俗讲解**：这个过滤器就像一个“门卫记事本”，记录当前请求是谁发来的（比如用户身份）。它会把用户的登录信息（比如用户名、权限）从 session 加载到请求中，处理完后再存回去。这样，后续过滤器和端点就能知道“这个请求是哪个用户发起的”。
- **在 OAuth2 中的作用**：确保授权端点（`/oauth2/authorize`）知道用户是否已登录，因为 OAuth2 授权码流通常需要用户登录同意。

### 2. **CsrfFilter**
- **作用**：防止跨站请求伪造（CSRF）攻击。
- **通俗讲解**：CSRF 攻击就像有人假冒你提交表单（比如偷偷让你转账）。这个过滤器检查请求是否带了正确的 CSRF 令牌（一个随机码），确保请求是用户自己发的，不是坏人伪造的。
- **在 OAuth2 中的作用**：保护 `/oauth2/authorize` 端点的表单提交（比如用户点击“同意授权”时）。不过，对于 `/oauth2/token` 等 API 端点，CSRF 防护可能会被禁用（因为它们通常是机器调用，不是浏览器表单）。
	 
### 3. **UsernamePasswordAuthenticationFilter**（或类似表单登录过滤器）
- **作用**：处理表单登录请求（用户名和密码）。
- **通俗讲解**：这是“登录窗口”，当用户访问 `/oauth2/authorize` 但没登录时，会跳转到登录页面。用户输入用户名密码后，这个过滤器验证身份，成功就放行。
- **在 OAuth2 中的作用**：确保用户在授权之前必须登录（因为 OAuth2 授权码流需要用户同意）。你的代码里用了 `http.formLogin(Customizer.withDefaults())`，所以这里会启用默认的表单登录机制。

### 4. **OAuth2AuthorizationServerConfigurerFilter**（自定义过滤器集合）
- **作用**：这是 Spring Security OAuth2 的核心过滤器集合，包含多个子过滤器，专门处理 OAuth2 协议的逻辑。
- **通俗讲解**：这就像一个“OAuth2 专属门卫团队”，负责处理授权服务器的各种请求，比如生成授权码、颁发令牌、验证客户端等。它不是单个过滤器，而是一个包含多个子过滤器的配置。
- **子过滤器包括**：
  - **OAuth2AuthorizationEndpointFilter**：
    - **作用**：处理 `/oauth2/authorize` 端点，生成授权码。
    - **通俗讲解**：这是“授权窗口”，用户在这里同意或拒绝给客户端授权。比如，用户登录后，页面会显示“某某应用想访问你的信息，同意吗？”。这个过滤器检查请求参数（`client_id`、`scope` 等），验证客户端合法性，然后生成授权码。
  - **OAuth2TokenEndpointFilter**：
    - **作用**：处理 `/oauth2/token` 端点，颁发访问令牌（access token）和刷新令牌（refresh token）。
    - **通俗讲解**：这是“令牌发放处”，客户端拿着授权码（或刷新令牌）来这里换真正的通行证（access token）。它会验证授权码、客户端身份、PKCE 参数（如果启用了），然后生成 JWT 或其他令牌。
  - **OAuth2TokenIntrospectionEndpointFilter**：
    - **作用**：处理 `/oauth2/introspect` 端点，验证令牌是否有效。
    - **通俗讲解**：这是“令牌检查站”，客户端或资源服务器可以拿令牌来问“这个令牌还好使吗？”。过滤器会检查令牌是否过期、是否被撤销。
  - **OAuth2TokenRevocationEndpointFilter**：
    - **作用**：处理 `/oauth2/revoke` 端点，撤销令牌。
    - **通俗讲解**：这是“令牌销毁处”，如果令牌泄露或不需要了，可以来这里作废它。
  - **OidcProviderConfigurationEndpointFilter**：
    - **作用**：处理 `/.well-known/openid-configuration` 端点，提供 OIDC 配置信息。
    - **通俗讲解**：这是“服务说明书”，告诉客户端授权服务器支持哪些功能（比如支持的 scope、端点地址等）。客户端访问这个端点就能知道怎么跟服务器互动。
  - **OidcUserInfoEndpointFilter**：
    - **作用**：处理 `/userinfo` 端点，返回用户信息。
    - **通俗讲解**：这是“用户信息窗口”，客户端用 access token 来这里查用户的身份信息（比如姓名、邮箱）。你的代码里自己实现了 `/userinfo` 端点，但这个过滤器是默认支持的。

### 5. **JwkSetEndpointFilter**
- **作用**：处理 `/oauth2/jwks` 端点，公开 JWK（JSON Web Key）信息。
- **通俗讲解**：这是“公钥发布处”，服务器把公钥放这里，客户端或资源服务器可以用它验证 JWT 签名。你的代码里配置了 `JWKSource`，这个过滤器会用它来提供公钥。
- **在 OAuth2 中的作用**：支持 JWT 令牌的验证，确保客户端能信任服务器发的令牌。

### 6. **ExceptionTranslationFilter**
- **作用**：处理认证和授权失败的异常，转换为合适的 HTTP 响应。
- **通俗讲解**：这是“错误处理员”，如果用户没登录、权限不够，或请求有问题，它会返回标准的错误信息（比如 401 未授权、403 禁止访问）。
- **在 OAuth2 中的作用**：确保 OAuth2 端点返回符合协议的错误响应（比如 `invalid_grant`、`invalid_client`）。

### 7. **FilterSecurityInterceptor**
- **作用**：执行访问控制，检查请求是否满足权限要求。
- **通俗讲解**：这是“权限检查员”，确保只有合法用户和客户端能访问特定端点。比如，`/oauth2/authorize` 要求用户登录，`/oauth2/token` 要求客户端提供正确参数。
- **在 OAuth2 中的作用**：保护授权服务器的端点，防止未授权访问。

---

## 三、过滤器执行顺序
Spring Security 的过滤器链按固定顺序执行，`applyDefaultSecurity` 确保这些过滤器以正确的优先级排列。你的代码里设置了 `@Order(Ordered.HIGHEST_PRECEDENCE)`，意味着这个过滤器链优先级最高，会先处理 OAuth2 相关的请求（比如 `/oauth2/*`）。大致顺序如下：
1. `SecurityContextPersistenceFilter`：加载安全上下文。
2. `CsrfFilter`：检查 CSRF 令牌。
3. `UsernamePasswordAuthenticationFilter`：处理登录（如果需要）。
4. OAuth2 专属过滤器（`OAuth2AuthorizationEndpointFilter`、`OAuth2TokenEndpointFilter` 等）：处理 OAuth2 逻辑。
5. `JwkSetEndpointFilter`、`OidcProviderConfigurationEndpointFilter` 等：处理辅助端点。
6. `ExceptionTranslationFilter`：处理错误。
7. `FilterSecurityInterceptor`：最后检查权限。

---

## 四、为什么需要这些过滤器？
这些过滤器共同实现了 OAuth2 和 OIDC 的核心功能，同时保证了安全性：
- **安全性**：`CsrfFilter`、`FilterSecurityInterceptor` 防止攻击和未授权访问。
- **协议支持**：`OAuth2AuthorizationEndpointFilter`、`OAuth2TokenEndpointFilter` 实现授权码流和令牌发放。
- **OIDC 功能**：`OidcProviderConfigurationEndpointFilter`、`OidcUserInfoEndpointFilter` 支持身份认证。
- **JWT 支持**：`JwkSetEndpointFilter` 提供公钥验证。

---

## 五、总结
`OAuth2AuthorizationServerConfiguration.applyDefaultSecurity(http)` 启用了一组过滤器，覆盖了 OAuth2 授权服务器的核心功能，包括用户登录、授权码生成、令牌发放、用户信息查询、令牌验证和撤销等。每个过滤器就像门卫系统的一个岗位，各自负责一块任务，共同确保服务器安全、高效地运行。

如果你想深入了解某个过滤器的实现细节，或者想知道如何自定义它们，随时告诉我！























