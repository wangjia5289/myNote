---
title: 笔记：跨站请求伪造攻击（CSRF）
date: 2025-04-05
categories:
  - 网络攻击
  - 跨站请求伪造攻击（CSRF）
tags: 
author: 霸天
layout: post
---
只要你使用 Cookie存储认证，就会存csrf攻击，例如remember-me cookie 和 jwt，remember-me cookie 还会对remember-me cookie 做一下检验，可你的jwt......
![](image-20250405170229690.png)

> [!NOTE] 注意事项
> 1. 通常会将 Token 存储在 `localStorage` 、`sessionStorage` 或 `IndexDB` 中，而非 `Cookie`，这样可以降低遭受 CSRF 攻击的风险，但仍需采取措施防范 XSS 攻击。
> 2. `remember-me Cookie` 不建议单独使用，单独使用时其安全性和灵活性通常不如基于 JWT（JSON Web Token） 的方案。







> [!NOTE] 注意事项
> 1. 通常会将 Token 存储在 `localStorage` 、`sessionStorage` 或 `IndexDB` 中，而非 `Cookie`，这样可以降低遭受 CSRF 攻击的风险，但仍需采取措施防范 XSS 攻击。
> 2. `remember-me Cookie` 不建议单独使用，单独使用时其安全性和灵活性通常不如基于 JWT（JSON Web Token） 的方案。

![](image-20250405170219569.png)














### 1、起因

在传统的 Web 应用中，许多请求（如提交表单、点击链接等）是由浏览器自动发送的，且浏览器会 自动携带与目标站点相关的 Cookie，**无论是跨站请求还是本站请求**。

---



### 2、攻击

假设你已经登录了某银行网站 `bank.com` ，并且浏览器保存了你的登录 Cookie。然后，你不小心访问了一个恶意网站 `malicious.com`。恶意网站可能会通过以下方式发起攻击：
```js
<img src="https://bank.com/transfer?to= 攻击者账号&amount=10000">
```

当你的浏览器加载这个恶意网站时，会尝试加载图片。浏览器会自动向 `bank.com` 发送一个请求：
```
GET /transfer?to=攻击者账号&amount=10000 HTTP/1.1
Host: bank.com 
Cookie: sessionId=123456...
```

由于你已经登录了 `bank.com` ，浏览器会自动携带你的 Cookie（包含你的身份信息）到 `transfer` API，如果银行网站的转账接口没有额外的验证（比如 CSRF Token 或二次确认）而只是例如依赖 `JSESSIONID` 验证 `HttpSession` 中的身份信息，该 API 可能会误认为请求是由登录用户发起的，它会认为这是一个合法的转账请求，并执行转账操作。

---



### 3、防范
防范 CSRF 攻击的常见方式主要有以下几种：

1.==使用 `SameSite` Cookie 属性==
`SameSite` 属性用于限制跨站请求时浏览器是否会携带 Cookie。通过设置为 `Strict` 或 `Lax`，可以有效阻止跨站请求携带用户的身份认证 Cookie
- <font color="#00b0f0">Strict</font>：仅允许同站请求携带 Cookie
- <font color="#00b0f0">Lax</font>：允许部分跨站 **GET** 请求携带 Cookie（默认）
- <font color="#00b0f0">None</font>：允许跨站携带 Cookie（必须配合 `Secure` 使用）

==2.使用 随机 CSRF Token==
Spring Security 默认的 CSRF 防护就是使用的这种方式，为每个敏感操作（如表单提交、删除操作等）生成一个 **随机的 CSRF Token**。每次请求时，客户端都需要把这个 Token 作为请求的一部分（通常是表单字段或 HTTP 头部），后端验证这个 Token 是否有效。只有当 Token 匹配时，操作才会被执行。这种方式有效防止恶意站点伪造请求，因为它无法获得有效的 Token。
![](image-20250405170201196.png)

==3.使用 `Referer` 或 `Origin` Header 验证==
后端可以检查请求的 **`Referer`** 或 **`Origin`** 请求头，确保请求来源于合法的站点，如果请求来源不符合要求，可以拒绝该请求。
- **`Referer`**：标识请求的来源页面。
- **`Origin`**：标识请求的源站点。

==4.确保敏感操作使用 `POST` 请求==
尽量使用 **`POST`** 请求而非 **`GET`** 请求来执行敏感操作，因为 GET 请求可以通过简单的 URL 链接发起，容易受到 CSRF 攻击。

==5.验证 Cookie 与请求头一致==
利用浏览器的 SameSite Cookie 属性，结合 `Access-Control-Allow-Origin` 等 HTTP 头部，确保跨站请求时不能泄露 Cookie。

==6.双重身份验证（2FA)==
对于敏感操作（如资金转账、删除账户等），可以结合双重身份验证（例如短信或邮件验证码、Google Authenticator）来增加安全性，即使攻击者伪造请求，仍然需要有效的第二步认证。

==7.限制请求来源==
对特定敏感请求，后端可以进行 IP 白名单或请求频率限制，增加防护层级，避免恶意请求的爆发。

==8.最小权限原则==
限制系统用户的权限，确保即使账户被冒用，攻击者也无法进行高权限操作。例如，用户默认只能访问与其身份相关的资源，除非特别授权。

---



### 4、总结

CSRF 攻击就像“冒充你的签名”去办理业务。防范的关键是：
1. <font color="#00b0f0">不让攻击者冒充你</font>（使用 CSRF Token、SameSite Cookie）；
2. <font color="#00b0f0">验证请求是否真的由你发起</font>（检查 Referer、Origin 头）；
3. <font color="#00b0f0">增加额外的安全层</font>（如 2FA、频率限制）通过。

---



### 5、反攻






在前后端分离架构中，JWT（JSON Web Token）一般有三种常见的存储方式，各有优缺点，具体选用取决于安全性、使用场景和需求。我们逐一分析：

---

### **1. `localStorage`（本地存储）**

**使用场景**：单页应用（SPA）、前端频繁操作 JWT。

**实现方式**：

```javascript
// 存储 JWT
localStorage.setItem('token', jwtToken);

// 获取 JWT
const token = localStorage.getItem('token');

// 删除 JWT（登出时）
localStorage.removeItem('token');
```

**优点**：

- 简单易用，存取方便。
- 生命周期与浏览器窗口一致，不随页面刷新丢失。
- 容量较大（一般 5MB）。

**缺点**：

- **存在 XSS（跨站脚本攻击）风险**：如果站点存在 XSS 漏洞，攻击者可以通过恶意脚本轻松窃取 JWT。
- 所有同源页面都能访问，扩大了攻击面。

**安全建议**：

- 确保前端严格防范 XSS（如使用 CSP、转义输出、禁用内联脚本等）。
- 在需要更高安全性的应用中谨慎使用。

---

### **2. `sessionStorage`（会话存储）**

**使用场景**：只需在当前会话中保持登录状态，页面关闭即登出。

**实现方式**：

```javascript
// 存储 JWT
sessionStorage.setItem('token', jwtToken);

// 获取 JWT
const token = sessionStorage.getItem('token');

// 删除 JWT（登出时）
sessionStorage.removeItem('token');
```

**优点**：

- 简单易用，与 `localStorage` 类似。
- 生命周期短，仅限当前页面会话，关闭页面后自动清除。
- 容量与 `localStorage` 相同。

**缺点**：

- **仍存在 XSS 风险**。
- 页面刷新不会丢失，但页面关闭即丢失 JWT，不适合需要长时间保持登录的场景。

**安全建议**：

- 与 `localStorage` 相同，加强 XSS 防范措施。

---

### **3. HTTP-only Cookie（推荐，用于增强安全性）**

**使用场景**：需要更强的安全性，防范 XSS、自动携带凭证。

**实现方式（后端设置 Cookie）**：

```java
// Spring Security 设置 Cookie
ResponseCookie jwtCookie = ResponseCookie.from("token", jwtToken)
    .httpOnly(true) // 防止前端 JS 访问，防范 XSS
    .secure(true)   // 仅 HTTPS 传输
    .sameSite("Strict") // 防止 CSRF 攻击
    .path("/")      // Cookie 全站有效
    .maxAge(Duration.ofDays(7)) // Cookie 生命周期
    .build();

response.setHeader(HttpHeaders.SET_COOKIE, jwtCookie.toString());
```

**前端请求（自动携带 Cookie）**：

```javascript
fetch('/api/protected', {
  method: 'GET',
  credentials: 'include' // 跨域请求也会自动携带 Cookie
});
```

**优点**：

- **防范 XSS 攻击**：因为 HTTP-only Cookie 无法被 JavaScript 访问，减少被窃取的风险。
- **自动随请求携带**：每次请求都会自动带上 Cookie，无需手动设置请求头。
- **生命周期可控**：可通过 Cookie 属性控制过期时间。

**缺点**：

- **存在 CSRF 风险**：因为 Cookie 会自动携带，攻击者可在用户不知情的情况下发起跨站请求。
- 配置稍复杂，需要前后端跨域时支持 `CORS`（`credentials: include`）。
- 需要 HTTPS 确保 Cookie 传输安全。

**安全建议**：

- 开启 `SameSite=Strict` 或 `SameSite=Lax`，限制跨站请求。
- 搭配 CSRF Token 使用，双重防范。
- 确保使用 HTTPS。

---

### **4. 内存中（少见）**

**使用场景**：只在当前页面生命周期中短暂存储。

**实现方式**：

```javascript
let jwtToken = null;

// 登录时
jwtToken = 'your-jwt-token';

// 请求时
fetch('/api/protected', {
  headers: { 'Authorization': `Bearer ${jwtToken}` }
});

// 登出时
jwtToken = null;
```

**优点**：

- **最安全**：只存在于当前页面内存中，不会被 XSS 或 CSRF 直接窃取。
- 生命周期极短，页面刷新或关闭即消失。

**缺点**：

- 页面刷新或跳转会导致 JWT 丢失。
- 不适合需要保持登录状态的场景。

---

### **推荐选择总结**

|存储方式|防 XSS|防 CSRF|生命周期|自动携带|使用难度|
|---|---|---|---|---|---|
|`localStorage`|❌|✅|持久（页面刷新不丢失）|❌|简单|
|`sessionStorage`|❌|✅|短暂（页面关闭丢失）|❌|简单|
|HTTP-only Cookie|✅|❌|可控（根据设置）|✅|中等|
|内存中|✅|✅|短暂（页面刷新丢失）|❌|简单|

---

### **最佳实践建议**

1. **安全性优先（推荐）**：HTTP-only Cookie + CSRF 防范（`SameSite=Strict`、CSRF Token、Origin 校验）。
2. **灵活性优先**：`localStorage` 或 `sessionStorage`，同时加强 XSS 防护（CSP、转义输出、禁用内联脚本）。
3. **短期敏感操作**：将 JWT 保存在内存中，只用于当前页面生命周期。

如果你的 JWT 主要用于 API 调用，并且希望自动携带认证凭证，**HTTP-only Cookie 是最安全的选择**。  
如果你的应用是 SPA，且希望前端灵活控制 token，`localStorage` 或 `sessionStorage` 会更方便，但请务必加强 XSS 防范。