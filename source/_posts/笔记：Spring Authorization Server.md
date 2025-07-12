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

### 客户端应用能注册哪些信息

客户端应用必须在我们的授权服务器完成注册，才能获得 Access Token 的颁发资格。那么，客户端在注册时需要提交哪些信息呢？在 Spring Authorization Server 中，类 `RegisteredClient` 定义了客户端应用可注册的具体属性，源码中的属性包括：
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

这里只列举我们常用的一些属性，其他次要或无关的属性暂不展开：

<font color="#92d050">1. id</font>
这个字段不是客户端应用提交的注册信息，而是我们入库时生成的自增主键（或者是 UUID 等格式），Spring 会使用该 ID 作为数据结构中的键，例如 `Map<String, RegisteredClient>`。

你可能会疑问，为什么不直接用 `clientId` 作为数据库主键？原因在于客户端应用可能会重置 `clientId`，而数据库主键要求必须保持唯一且不可变。

<font color="#92d050">2. clientId</font>
clientId 是 OAuth2 协议中用于标识客户端的重要凭证之一，当客户端应用进行注册时，由**我们生成**一个 `clientId` 并返回给他们。


<font color="#92d050">3. clientIdIssuedAt</font>
这是 `clientId` 的发放时间，**同样由我们生成**，通常在生成 `clientId` 时一并创建，该时间用于便于后续跟踪和判断 `clientId` 的有效性。

要求以时间戳的形式，建议在数据库中使用 `TIMESTAMP` 类型存储，格式示例为：`2025-07-11 12:34:56.789 +00:00`。

需要注意的是，Spring Authorization Server 并不会对该时间戳进行任何处理，这主要是为了便于我们开发人员对 `clientId` 的生命周期进行跟踪和管理。


<font color="#92d050">4. clientSecret</font>
clientSecret 同样是 OAuth2 协议中用于标识客户端的重要凭证之一，当客户端应用进行注册时，由我们生成一个 `clientSecret` 并返回给他们。

可以将 `clientId` 理解为账号，而 `clientSecret` 则相当于密码。


<font color="#92d050">5. clientSecretExpiresAt</font>
这是 `clientSecret` 的过期时间，主要用于支持密钥轮换场景，可设置为 `NULL` 表示永不过期（实际应用中通常为不过期）。

要求以时间戳的形式，建议在数据库中使用 `TIMESTAMP` 类型存储，格式示例为：`2025-07-11 12:34:56.789 +00:00`。


需要注意的是，Spring Authorization Server 同样并不会对该时间戳进行任何处理，这主要是为了便于我们开发人员对 `clientSecret` 进行轮换处理等。


<font color="#92d050">6. clientName</font>
客户端应用的名称，由客户端应用在注册时指定，用于展示给用户，例如在授权确认页面中，会显示为 “XXX 正在请求访问你的信息，是否同意”。


<font color="#92d050">7. clientAuthenticationMethods</font>
客户端如何进行身份认证，简单来说，就是在哪个位置、以何种方式传递 `client_id` 和 `client_secret`，需要客户端应用在注册时进行指定，常见的方式包括：
1. client_secret_basic（默认）：
	1. 通过 HTTP Basic 的方式传递 `client_id` 和 `client_secret`
	2. 简单来说，就是将其放入请求头的 Authorization 中，格式为：`Authorization: Basic <Base64 编码后的 clientId:clientSecret>`，例如：`Authorization: Basic Y2xpZW50MTIzOnNlY3JldFhZWg==`
2. client_secret_post：
	1. 通过表单参数的方式传递 `client_id` 和 `client_secret`
3. none：
	1. 仅传 `client_id`，不传 `client_secret`，常用于 SPA 单页应用或移动端应用（这也是引入 PKCE 的原因）

需要注意的是，客户端可以在注册时，可以注册多个身份认证方式，Spring Authorization Server 会按顺序依次尝试这些方式，直到认证成功为止。


<font color="#92d050">8. authorizationGrantTypes</font>
客户端支持的授权方式，需要客户端应用在注册时进行指定（指定未来可能会用到哪些授权方式）常见包括：
1. authorization_code：
	1. 授权码模式
2. client_credentials：
	1. 客户端凭证式
3. refresh_token：
	1. 刷新令牌式
4. password：
	1. 密码式，已被弃用，不推荐使用

需要注意的是：
1. Spring Authorization Server 从未支持过隐藏式（implicit）。
2. 客户端在注册时可以注册多个授权方式，但 Spring Authorization Server 并不会按顺序依次尝试这些方式。而是在授权请求时，客户端必须明确指定使用其中的一种授权方式。
	1. 简单来说，注册时支持注册多个授权方式，表示未来可能会用到这些授权方式
	2. 但授权时，只能选择其中一种进行使用
	3. 如果授权请求中指定的授权模式未在注册时指定，服务器将拒绝处理该请求。


<font color="#92d050">9. redirectUris</font>
用户授权成功后的回调地址（redirect URI），由客户端应用在注册时进行指定，而且在发起授权请求时必须携带**完全一致**的值，否则服务器将拒绝处理请求，例如：`https://example.com/oauth2/callback`


<font color="#92d050">10. scopes</font>
客户端请求的权限范围，其是我们高度自定义的，客户端应用在注册时需勾选其可能会用到的权限范围（scope），表示未来可能申请这些权限。

实际授权请求时，客户端携带所需的 scope，服务器根据请求的 scope 颁发相应的 Access Token，确保客户端应用使用该 Access Token 只能访问授权范围内的信息。

需要注意的是，注册时勾选的 scope 并非必须全部使用，而是以授权请求中携带的 scope 为准。如果授权请求中指定的 scope 未在注册时配置，服务器将拒绝该请求。

我们在自定义权限范围时，一般遵循这样的命名规则：
1. 模块:资源:操作
	1. 细粒度权限范围
	2. 例如读取用户信息的权限，可以命名为：`user:user:read`
2. 模块:操作
	1. 粗粒度权限范围
	2. 例如读取用户信息的权限，可以命名为：`user:read`

OAuth 的 Scope 本质上是高度自定义的，但 OpenID Connect（OIDC）规范定义了一些标准的 scope，包括：
1. openid
	1. 表示请求 OpenID Connect 身份令牌（ID Token），如果客户端应用需要我们返回 ID Token，则需要在授权请求中携带 openid
	2. 返回的 ID Token 为 JWT 格式，默认仅包含 `sub` 字段（即 `userInfoRequest.getPrincipal().getName()` 的值），使用 RS256 算法（私钥签名、公钥验证）。
	3. 需要注意的是，默认情况下公私钥对是在应用启动时内存中动态生成的。可通过 `/.well-known/jwks.json` 端点获取当前启动的公钥用于验证 ID Token。
	4. 但应用重启或多实例部署时，密钥对会发生变化，导致之前颁发的 ID Token 无法验证。实际环境中，应使用固定的密钥对，确保 ID Token 在服务重启后依然可验证。
2. profile
	1. 请求访问用户的基本信息，如姓名、头像、性别等
3. email
	1. 请求访问用户的电子邮件地址
4. address
	1. 请求访问用户的地址信息
5. phone
	1. 请求访问用户的电话号码
6. offline_access
	1. 请求返回 Refresh Token，以实现用户离线访问。
	2. 如果未集成 OIDC，在 `authorizationGrantTypes` 中配置了 `refresh_token`，Spring Authorization Server 会返回 Refresh Token
	3. 但是在集成 OIDC 后，必须携带 `offline_access`，授权服务器才会返回 Refresh Token，因为其要求更为严格。


<font color="#92d050">11. clientSettings</font>
客户端级别的配置，需要客户端应用在注册时指定，格式为 JSON，通常在数据库中以 `TEXT` 类型进行存储。可配置的项包括：
1. requireAuthorizationConsent(boolean)
	1. 是否强制用户在授权时显示“授权确认页”（consent screen），即使用户已授权
	2. 默认值为 `false`
2. requireProofKey(boolean)
	1. 是否启用 PKCE
	2. 默认值为 `false`
```
{
  "requireAuthorizationConsent": true,
  "requireProofKey": false
}
```


<font color="#92d050">12. tokenSettings</font>
设置各种令牌的配置，可配置的项包括：
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















