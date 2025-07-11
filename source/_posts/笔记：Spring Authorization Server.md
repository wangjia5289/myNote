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

## 导图

![](Map：SpringAuthorizationServer%201.xmind)

---


## Spring Authorization Server 执行流程

### OAuth2AuthorizationEndpointFilter 介入













## Spring Authorization Server 配置

### Spring Authorization Server 配置模板

AuthServerConfiguration 类在 `com.example.oauthserverwithmyproject.configuration` 包下，直接粘贴这份配置模板，再根据下方的详细说明按需进行调整。













# 二、实操

## 注册客户端应用

### 了解客户端应用能注册哪些信息

客户端应用需要在我们的授权服务器进行注册他的应用，这样我们才能为其颁发 Access Token，那客户端应用在注册时需要注册哪些信息呢？在 Spring Authorization Server 中有一个类叫做 RegisteredClient，其的属性指定了客户端应用可以注册哪些信息，其源码的属性为：
```
public class RegisteredClient implements Serializable { 
    private String id;  
    private String clientId;  
    private Instant clientIdIssuedAt;  
    private String clientSecret;  
    private Instant clientSecretExpiresAt;  
    private String clientName;  
    private Set<ClientAuthenticationMethod> clientAuthenticationMethods;  
    private Set<AuthorizationGrantType> authorizationGrantTypes;  
    private Set<String> redirectUris;  
    private Set<String> postLogoutRedirectUris;  
    private Set<String> scopes;  
    private ClientSettings clientSettings;  
    private TokenSettings tokenSettings;
}
```

这里只列举我们常用的一些属性，一些无关紧要的属性就不管他了：
<font color="#92d050">1. id</font>
这个与我们的 OAuth 关系不大，主要是 Spring 会将这个 ID 用作数据结构中的 key，例如 `Map<String, RegisteredClient>`。如果我们将注册的客户端应用保存在数据库中，那么这个 ID 就应为数据库中的主键；即便我们直接通过 Spring 的 Bean 注册客户端，也仍然需要传入一个 UUID。


<font color="#92d050">2. clientId</font>
客户端 ID 是 OAuth2 协议中用于标识客户端的重要凭证之一。当客户端应用进行注册时，我们应当生成一个 `clientId` 并返回给他们。


<font color="#92d050">3. clientIdIssuedAt</font>
`clientId` 的发放时间，便于跟踪或判断其是否有效，需要保存为时间戳，推荐使用 TIMESTAMP 保存，格式为：2025-07-11 12:34:56.789 +00:00


<font color="#92d050">4. clientSecret</font>
客户端 Secret 同样是 OAuth2 协议中用于标识客户端的重要凭证之一。当客户端应用进行注册时，我们应当生成一个 `clientSecret` 并返回给他们。

我们可以理解为 clientId 是账号，而 clientSecret 为密码


<font color="#92d050">5. clientSecretExpiresAt</font>
`clientSecret` 什么时候过期，用于密钥轮换场景，可为 NULL，表示不过期（一般都是不过期的场景）
需要注意的是，Spring Authorization Server 不会自动对过期的 clientSecret 进行操作，需要我们自己在业务层或者过滤器进行处理


<font color="#92d050">6. clientName</font>
客户端名称，用于展示给用户，例如授权确认页面中看到的 “XXX访问你的信息，是否同意”


<font color="#92d050">7. clientAuthenticationMethods</font>
客户端如何进行**身份认证**，简单来说就算在哪里以及如何传clientId 和 clientSecret，常用的有：
1. client_secret_basic（默认）：
	1. 通过 HTTP Basic 的方式传 client_id 和 client_secret
	2. 简单来说就是放到 Authorization 请求头中，格式为：Authorization: Basic <Base64 编码后的clientId:clientSecret>，例如 `Authorization: Basic Y2xpZW50MTIzOnNlY3JldFhZWg==`
2. client_secret_post：
	1. 通过表单参数传 client_id 和 client_secret
3. none：
	1. 不传 `client_secret`，只传 `client_id`，常用于 SPA 单页应用和移动端应用（这也是为什么引入 PKCE 的原因）


<font color="#92d050">8. authorizationGrantTypes</font>
客户端支持的授权方式，常见有：
1. authorization_code：
	1. 授权码模式
2. client_credentials：
	1. 客户端凭证式
3. refresh_token：
	1. 刷新令牌式
4. password：
	1. 密码式，已弃用，不推荐使用

需要注意的是，Spring Authorization Server 从来没有对隐藏式的支持


<font color="#92d050">9. redirectUris</font>
授权成功后跳转的回调地址，是由客户端应用在注册时指定的，并且客户端应用在发起授权请求的时候，必须携带相同的值，否则服务器拒绝。


<font color="#92d050">10. scopes</font>
客户端请求的权限范围，其实这些我们都可以自定义，例如：读取用户信息我们可以写user:user:read，想细粒度就模块：资源：操作，粗糙力度就模块：操作，但是 OIDC 规范了一些标准的 scope，例如：
1. openid
	1. 表示请求 OpenID Connect 身份令牌，如果要使用 OIDC 功能，必须要包含这个。必须包含以启用 OIDC 功能，这个返回的 ID Token 是一个 JWT 格式的，其规定和默认只返回 sub，也就是userInfoRequest.getPrincipal().getName()，使用 RS256 加密方式，即用私钥对 JWT 进行签名，公钥用于验证
	2. 需要注意的是，默认的公钥私钥都是应用在启动时在内存中生成的，我们可以查看 /.well-known/jwks.json 端点获取应用这次启动的公钥，用于验证对不对，但是一旦应用重启，或者你有很多的授权服务器，这个就不好管理了，并且会造成之前颁发的 ID Token 由于密钥和私钥不同无法验证，真实环境下，我们需要使用固定的密钥对来保证 ID Token 在服务重启后仍然可验证
2. profile
	1. 请求访问用户的基本信息，如姓名、头像、性别等
3. email
	1. 请求访问用户的电子邮件地址
4. address
	1. 请求访问用户的地址信息
5. phone
	1. 请求访问用户的电话号码
6. offline_access
	1. 请求返回 Refresh Token，实现用户离线访问。
	2. 虽然你在 authorizationGrantTypes 中有refresh_token，Spring AuhorizationServer 也会返回 Refresh Token，但是如果你集成了 OIDC，那它会更加严格，就算说你必须携带这个才会返回 Refresh Token


<font color="#92d050">11. clientSettings</font>
客户端级别的配置，在数据库中一般用 TEXT 进行存储，可配置的项有：
1. requireAuthorizationConsent(boolean)
	1. 强制用户在授权时显示“授权确认页”（consent screen），即便用户已授权，例如 requireAuthorizationCConsent(true)，可选值有：
		1. true：
		2. false（默认值）
2. requireProofKey(boolean)
	1. 是否启用 PKCE，例如 requireProofKey(true)，可选值有：
		1. true
		2. false（默认值）
3. jwkSetUrl(String)
	1. 指定客户端的 JWK Set 地址，用于验证 JWT 客户端认证
	2. 仅限客户端凭证，默认值为 null
4. settings(Map<String, Object>)
	1. 允许你设置自定义的扩展字段（可在服务端通过 `ClientSettings.getSetting(key)` 读取）
```
ClientSettings clientSettings = ClientSettings.builder()
    .requireAuthorizationConsent(true)
    .requireProofKey(true)
    .jwkSetUrl("https://client.example.com/jwks.json")
    .setting("custom.setting", "my-value")
    .build();
```

在数据库中，我们以 JSON 进行存储，例如：
```
{
  "requireAuthorizationConsent": true,
  "requireProofKey": false,
  "jwkSetUrl": null,
  "customSettings": {
    "custom.setting": "my-value"
  }
}
```


<font color="#92d050">12. tokenSettings</font>
关于 access_token、refresh_token 等的配置，常用配置项有：
1. accessTokenTimeToLive(Duration)

| 配置方法                                    | 类型         | 默认值              | 说明                                              |
| --------------------------------------- | ---------- | ---------------- | ----------------------------------------------- |
| `accessTokenTimeToLive(Duration)`       | `Duration` | 5 分钟             | 访问令牌的有效期                                        |
| `refreshTokenTimeToLive(Duration)`      | `Duration` | 30 天             | 刷新令牌的有效期                                        |
| `reuseRefreshTokens(boolean)`           | `boolean`  | `true`           | 刷新令牌是否可重复使用（`true`：可反复用；`false`：每次刷新都会生成新的刷新令牌） |
| `idTokenTimeToLive(Duration)`           | `Duration` | 5 分钟             | ID Token 的有效期（OIDC 场景）                          |
| `authorizationCodeTimeToLive(Duration)` | `Duration` | 5 分钟             | 授权码的有效期                                         |
| `accessTokenFormat(AccessTokenFormat)`  | 枚举         | `SELF_CONTAINED` | 访问令牌格式（自包含 JWT 或引用型 Token）                      |
- **accessTokenTimeToLive**：控制客户端拿到的访问令牌在多长时间后失效，影响资源服务器的访问权限有效期。
    
- **refreshTokenTimeToLive**：刷新令牌的有效期，决定客户端在多长时间内可以用刷新令牌换取新的访问令牌。
    
- **reuseRefreshTokens**：是否允许刷新令牌重复使用。设置为 `false` 能提高安全性，防止刷新令牌被重放利用。
    
- **authorizationCodeTimeToLive**：授权码有效期，过期后用户必须重新授权获取新的授权码。
    
- **accessTokenFormat**：通常选择 `SELF_CONTAINED`（JWT），也可以选引用型（opaque token），但需要额外支持 Token 存储和校验。

```
TokenSettings tokenSettings = TokenSettings.builder()
    .accessTokenTimeToLive(Duration.ofMinutes(60))         // 访问令牌有效期 60 分钟
    .refreshTokenTimeToLive(Duration.ofDays(14))           // 刷新令牌有效期 14 天
    .reuseRefreshTokens(false)                              // 刷新令牌使用一次后失效，强制发新令牌
    .authorizationCodeTimeToLive(Duration.ofMinutes(10))   // 授权码有效期 10 分钟
    .build();
```


### 创建 客户端应用 数据库表















