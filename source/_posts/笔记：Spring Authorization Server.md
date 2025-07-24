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

## 1. 导图

![](Map：SpringAuthorizationServer 1.xmind)

---


## 2. Spring Authorization Server 执行流程

### 2.1. OAuth2AuthorizationEndpointFilter 介入













## 3. Spring Authorization Server 配置

### 3.1. Spring Authorization Server 配置模板

AuthServerConfiguration 类在 `com.example.oauthserverwithmyproject.configuration` 包下，直接粘贴这份配置模板，再根据下方的详细说明按需进行调整。













# 二、实操

## 1. 创建 Spring Web 项目，添加相关依赖

1. Web
	1. Spring Web
2. Template Engines
	1. Thymeleaf
3. Security
	1. Spring Security
	2. OAuth2 Authorization Server
4. SQL
	1. JDBC API
	2. MyBatis Framework
	3. MySQL Driver


> [!NOTE] 注意事项
> 1. `spring-security-oauth2-jose` 是 Spring 官方为 OAuth2 提供的 JWT 支持模块，而 `jjwt-*` 是 Okta 社区维护的第三方库 JJWT。
> 2. 虽然 JJWT 也能生成 JWT，但它与 Spring Security 的集成度较低，许多功能（如 token 签发、校验、JWK 支持等）都需要我们手动实现。它与我们原先采用的 JJWT 库存在不少差异，不能直接沿用过去的写法，需要根据新库的方式进行修改。
> 3. Thymeleaf 是个模板引擎，负责把模板（html文件）渲染成最终的HTML页面返回给浏览器，只有在 `@Controller` 返回字符串为视图名时，Thymeleaf 才会介入渲染页面。

-----


## 2. 实现 注册客户端应用 功能

### 2.1. 客户端应用能注册哪些信息

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
这个字段不是客户端应用提交的注册信息，而是我们入库时生成的主键（常用 UUID），Spring 会使用该 ID 作为数据结构中的键，例如 `Map<String, RegisteredClient>`。

Spring Authorization Server 所需的 id 类型为 String，我们可以在数据库中使用 int 类型进行存储，使用时进行数据类型转换

> [!NOTE] 注意事项
> 1. 为什么不用自增主键而是 UUID
> 	1. Spring Authorization Server 要求 id 类型为 String，但自增主键为 Int，两者类型不匹配，需额外编写 TypeHandler 进行转换，太麻烦
> 	2. 直接采用 UUID 作为主键，避免类型转换，而且更简洁、符合分布式设计理念
> 2. 为什么不直接用 `clientId` 作为数据库主键？
> 	1. 原因在于客户端应用可能会重置 `clientId`，而数据库主键要求必须保持唯一且不可变。


<font color="#92d050">2. clientId</font>
clientId 是 OAuth2 协议中用于标识客户端的重要凭证之一，当客户端应用进行注册时，由**我们生成**一个 `clientId` 并返回给他们。

Spring Authorization Server 所需的 clientId 类型为 String，我们可以在数据库中使用 int 类型进行存储，使用时进行数据类型转换


<font color="#92d050">3. clientIdIssuedAt</font>
这是 `clientId` 的发放时间，**同样由我们生成**，通常在生成 `clientId` 时一并创建，该时间用于便于后续跟踪和判断 `clientId` 的有效性。

Spring Authorization Server 所需的 clientIdIssuedAt 类型为 Instant，Instant 的格式为：`YYYY-MM-DDTHH:mm:ssZ`，如`2025-07-13T01:23:45Z`，表示 UTC 上的一个时间点。

建议在数据库中使用带时区的 `timestamp` 类型存储，格式为：`YYYY-MM-DD HH:mm:ss`，如 `2025-07-13 01:23:45`

> [!NOTE] 注意事项
> 1. Spring Authorization Server 并不会对该时间戳进行任何处理，这主要是为了便于我们开发人员对 `clientId` 的生命周期进行跟踪和管理。
> 2. 推荐使用 `timestamp` 类型进行存储，而不是 `VARCHAR`，因为前者可以与 `Instant` 类型无缝映射，避免手动解析字符串
> 3. 建议使用带时区的 `timestamp` 类型，是因为它表示的是 UTC 上的一个绝对时间点。而不带时区的时间类型仅代表本地时间，可能导致语义不明确或出现误差
> 4. MySQL 虽然不支持真正意义上的带时区 `timestamp`，但其 `timestamp` 类型在内部始终以 UTC 存储。在插入数据时，如果未显式指定时区，MySQL 会按当前系统时区进行转换，为了确保行为一致，建议在插入前显式设置为 UTC，例如：
```
SET time_zone = '+00:00';
INSERT INTO registered_client (client_id, client_id_issued_at)
VALUES ('my-client', '2025-07-13 01:23:45');
```


<font color="#92d050">4. clientSecret</font>
clientSecret 同样是 OAuth2 协议中用于标识客户端的重要凭证之一，当客户端应用进行注册时，由我们生成一个 `clientSecret` 并返回给他们，可以将 `clientId` 理解为账号，而 `clientSecret` 则相当于密码。

Spring Authorization Server 所需的 clientSecret 类型为 String，格式为：`{bcrypt}$2a$10$...`，格式通常为：`{bcrypt}$2a$10$...`。其中 `{bcrypt}` 是加密前缀，用于指示 Spring Authorization Server 采用哪种方式验证客户端提交的 `clientSecret` 是否与数据库中存储的加密值一致。常见的前缀包括：
1. {noop}
	1. 明文密码（不加密）
2. {bcrypt}
	1. 使用 BCrypt 加密
3. {pbkdf2}
	1. 使用 PBKDF2 加密
4. {argon2}
	1. 使用 Argon2 加密

其相关代码逻辑大致如下：
```
// 1. 生成未加密的 clientSecret
String rawPassword = UUID.randomUUID().toString();


// 2. 将未加密的 clientSecret 进行加密，并加上对应加密算法的前缀
String encodedPassword = "{bcrypt}" + passwordEncoder.encode(rawPassword);


// 3. 将处理好的 clientSecret 保存到数据库
registeredClient.setClientSecret(encodedPassword);


// 4. 返回给客户端应用的是 未加密的 clientSecret（rawPassword）
```


<font color="#92d050">5. clientSecretExpiresAt</font>
这是 `clientSecret` 的过期时间，主要用于支持密钥轮换场景，可设置为 `NULL` 表示永不过期（实际应用中通常为不过期）。

Spring Authorization Server 所需的 clientSecretExpiresAt 类型为与 clientIdIssuedAt 一致


<font color="#92d050">6. clientName</font>
客户端应用的名称，由客户端应用在注册时指定，用于展示给用户，例如在授权确认页面中，会显示为 “XXX 正在请求访问你的信息，是否同意”。

Spring Authorization Server 所需的 clientName 类型为 String


<font color="#92d050">7. clientAuthenticationMethods</font>
客户端如何进行身份认证，简单来说，就是在哪个位置、以何种方式传递 `client_id` 和 `client_secret`，需要客户端应用在注册时进行指定，常见的方式包括：
1. client_secret_basic（默认）：
	1. 通过 HTTP Basic 的方式传递 `client_id` 和 `client_secret`
	2. 简单来说，就是将其放入请求头的 Authorization 中，格式为：`Authorization: Basic <Base64 编码后的 clientId:clientSecret>`，例如：`Authorization: Basic Y2xpZW50MTIzOnNlY3JldFhZWg==`
2. client_secret_post：
	1. 通过表单参数的方式传递 `client_id` 和 `client_secret`
3. none：
	1. 仅传 `client_id`，不传 `client_secret`，常用于 SPA 单页应用或移动端应用（这也是引入 PKCE 的原因）

> [!NOTE] 注意事项
> 1. 客户端在注册时，可以注册多个身份认证方式，Spring Authorization Server 会按顺序依次尝试这些方式，直到认证成功为止。
> 2. 在数据库中存储，就用小写格式存储即可，Spring Authorization Server 会自动进行处理，不要用大写格式存储


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

> [!NOTE] 注意事项
> 1. Spring Authorization Server 从未支持过隐藏式（implicit）。
> 2. 客户端在注册时可以注册多个授权方式，但 Spring Authorization Server 并不会按顺序依次尝试这些方式。而是在授权请求时，客户端必须明确指定使用其中的一种授权方式。
> 	1. 简单来说，注册时支持注册多个授权方式，表示未来可能会用到这些授权方式
> 	2. 但授权时，只能选择其中一种进行使用
> 	3. 如果授权请求中指定的授权模式未在注册时指定，服务器将拒绝处理该请求。
> 3. 这些在数据库中也以小写格式存储


<font color="#92d050">9. redirectUris</font>
用户授权成功后的回调地址（redirect URI），由客户端应用在注册时进行指定，而且在发起授权请求时必须携带**完全一致**的值，否则服务器将拒绝处理请求，例如：`https://example.com/oauth2/callback`

Spring Authorization Server 所需的 clientName 类型为 String

> [!NOTE] 注意事项
> 1. Spring Authorization Server 允许一个客户端配置多个 redirectUris，并会依次处理这些地址。
> 2. 但出于安全考虑，通常我们只允许一个客户端应用注册一个重定向地址。


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

> [!NOTE] 注意事项
> 1. 我们在颁发 Access Token 的时候，第三方客户端应用肯定不能和我们自己家的客户端应用拥有相同的权限。我们颁发给自家客户端的是具备完整访问权限的 Token，而第三方客户端只能访问部分数据。
> 2. 因此，我们可以设置一个只有我们自己知道、第三方客户端无法获取的 scope，通过这个 scope，我们就能获取到所有权限。


<font color="#92d050">11. clientSettings</font>
客户端级别的配置，需要客户端应用在注册时指定，Spring Authorization Server 所需的 clientSettings 类型为 JSON，通常在数据库中以 `TEXT` 类型进行存储。可配置的项包括：
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
各种令牌的配置，需要客户端应用在注册时指定，Spring Authorization Server 所需的 tokenSettings 类型为 JSON，通常在数据库中以 `TEXT` 类型进行存储。可配置的项包括：
1. accessTokenTimeToLive(Duration)
	1. 访问令牌（Access Token）的有效期，以分钟为单位
	2. 例如：`accessTokenTimeToLive(5)` 是 5 分钟
2. accessTokenFormat(AccessTokenFormat)
	1. 访问令牌（Access Token）的格式
	2. AccessTokenFormat.REFERENCE
		1. 访问令牌格式为引用类型
		2. 即 Access Token 是一个随机 ID，不包含任何业务信息，例如：`bdf89d81-8c41-47ad-a3cf-3cb257c13b7c`
		3. 授权服务器需自行维护 Access Token 与用户及权限信息的映射关系，能根据 Access Token 查出对应的用户
	3. AccessTokenFormat.SELF_CONTAINED
		1. 访问令牌格式为自包含类型
		2. 即 Access Token 中包含用户的基本信息，不用授权服务器自己维护，常以 JWT 格式
3. refreshTokenTimeToLive(Duration)
	1. 刷新令牌（Refresh Token）的有效期，以天为单位
	2. 例如：`refreshTokenTimeToLive(30)` 是 30 天
4. reuseRefreshTokens(boolean)
	1. 刷新令牌是否可重复使用
	2. true：
		1. 可重复使用
	3. false：
		1. 每次刷新都会生成新的刷新令牌
5. idTokenTimeToLive(Duration)
	1. ID 令牌（ID Token）的有效期，以分钟为单位
	2. 例如：`idTokenTimeToLive(10)` 是 10 分钟
6. authorizationCodeTimeToLive(Duration)
	1. 授权码（Authorization Code）的有效期，以分钟为单位
	2. 例如：`authorizationCodeTimeToLive(5)` 是 5 分钟
```
{
  "access_token_time_to_live": 300,
  "access_token_format": "REFERENCE",
  "reuse_refresh_tokens": false,
  "refresh_token_time_to_live": 2592000,
  "id_token_time_to_live": 600,
  "authorization_code_time_to_live": 300
}
```

> [!NOTE] 注意事项
> 1. 在数据库中一般采用秒为单位存储，这是一种行业规范，Spring Authorization Server 不会自动进行单位转换，需由开发者自行处理。

----


### 2.2. 创建 客户端应用 相关 MySQL 表

#### 2.2.1. ER 图

![](image-20250713165050243.png)

----


#### 2.2.2. oauth_clients 表（客户端应用表）

| 列名                           | 数据类型         | 约束   | 索引   | 默认值          | 示例值                                                                                                                                                                                                                                                   | 说明        |
| ---------------------------- | ------------ | ---- | ---- | ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------- |
| **id**                       | varchar(100) | 唯一约束 | 唯一约束 |              | 550e8400-e29b-41d4-a716-446655440000                                                                                                                                                                                                                  | ......... |
| **client_id**                | varchar(100) | 唯一约束 | 唯一索引 |              | web-client                                                                                                                                                                                                                                            | ......... |
| **client_id_issued_at**      | timestamp    |      |      | 当前 timestamp | 2025-07-13 12:00:00                                                                                                                                                                                                                                   | ......... |
| **client_secret**            | varchar(100) |      |      |              | {bcrypt}$2a$10$...                                                                                                                                                                                                                                    | ......... |
| **client_secret_expires_at** | timestamp    |      |      | NULL         | 2025-07-13 12:00:00                                                                                                                                                                                                                                   | ......... |
| **client_name**              | varchar(100) |      |      |              | 吧唧                                                                                                                                                                                                                                                    | ......... |
| **redirect_uris**            | varchar(100) |      |      |              | `https://example.com/oauth2/callback`                                                                                                                                                                                                                 | ......... |
| **client_settings**          | text         |      |      | {}           | {<br>  "requireAuthorizationConsent": true,<br>  "requireProofKey": false<br>}                                                                                                                                                                        | ......... |
| **token_settings**           | text         |      |      | {}           | {<br>  "access_token_time_to_live": 300,<br>  "access_token_format": "REFERENCE",<br>  "reuse_refresh_tokens": false,<br>  "refresh_token_time_to_live": 2592000,<br>  "id_token_time_to_live": 600,<br>  "authorization_code_time_to_live": 300<br>} | ......... |
```
CREATE TABLE IF NOT EXISTS oauth_clients (
    id VARCHAR(100),
    client_id VARCHAR(100),
    client_id_issued_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    client_secret VARCHAR(100),
    client_secret_expires_at TIMESTAMP DEFAULT NULL,
    client_name VARCHAR(100),
    redirect_uris VARCHAR(100),
    client_settings TEXT,
    token_settings TEXT
);


ALTER TABLE oauth_clients ADD CONSTRAINT uk_oauth_clients_id UNIQUE (id);


ALTER TABLE oauth_clients ADD CONSTRAINT uk_oauth_clients_client_id UNIQUE (client_id);
```

----


#### 2.2.3. oauth_client_authentication_methods 表（身份认证方式表）

| 列名            | 数据类型         | 约束                 | 索引   | 默认值 | 示例值                                  | 说明                                                                    |
| ------------- | ------------ | ------------------ | ---- | --- | ------------------------------------ | --------------------------------------------------------------------- |
| **id**        | int          | 主键约束、自增属性          | 主键索引 | 自增  | 1                                    | .........                                                             |
| **client_id** | varchar(100) | 外键约束（→ clients.id） |      |     | 550e8400-e29b-41d4-a716-446655440000 | 是 `oauth_clients` 表中的 `id` 字段，而不是 `oauth_clients` 表中的 `client_id` 字段。 |
| **method**    | varchar(100) |                    |      |     | none                                 | .........                                                             |
```
CREATE TABLE IF NOT EXISTS oauth_client_authentication_methods (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id VARCHAR(100),
    method VARCHAR(100)
);


ALTER TABLE oauth_client_authentication_methods ADD CONSTRAINT fk_ocam_client_id_to_oc_id FOREIGN KEY (client_id) REFERENCES oauth_clients(id);
```

----


#### 2.2.4. oauth_client_authorization_grant_types 表（授权方式表）

| 列名             | 数据类型         | 约束                 | 索引   | 默认值 | 示例值                                  | 说明                                                        |
| -------------- | ------------ | ------------------ | ---- | --- | ------------------------------------ | --------------------------------------------------------- |
| **id**         | int          | 主键约束、自增属性          | 主键索引 | 自增  | 1                                    | .........                                                 |
| **client_id**  | varchar(100) | 外键约束（→ clients.id） |      |     | 550e8400-e29b-41d4-a716-446655440000 | 是 `clients` 表中的 `id` 字段，而不是 `clients` 表中的 `client_id` 字段。 |
| **grant_type** | varchar(100) |                    |      |     | authorization_code                   | .........                                                 |
```
CREATE TABLE oauth_client_authorization_grant_types (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id VARCHAR(100),
    grant_type VARCHAR(100)
);


ALTER TABLE oauth_client_authorization_grant_types ADD CONSTRAINT fk_ocagt_client_id_to_oc_id FOREIGN KEY (client_id) REFERENCES oauth_clients(id);
```

----


#### 2.2.5. oauth_client_scopes 表（权限表）

| 列名            | 数据类型         | 约束                 | 索引   | 默认值 | 示例值                                  | 说明                                                        |
| ------------- | ------------ | ------------------ | ---- | --- | ------------------------------------ | --------------------------------------------------------- |
| **id**        | int          | 主键约束、自增属性          | 主键索引 | 自增  | 1                                    | .........                                                 |
| **client_id** | varchar(100) | 外键约束（→ clients.id） |      |     | 550e8400-e29b-41d4-a716-446655440000 | 是 `clients` 表中的 `id` 字段，而不是 `clients` 表中的 `client_id` 字段。 |
| **scope**     | varchar(100) |                    |      |     | openid                               | .........                                                 |

```
CREATE TABLE IF NOT EXISTS oauth_client_scopes (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id VARCHAR(100),
    scope VARCHAR(100)
);


ALTER TABLE oauth_client_scopes ADD CONSTRAINT fk_ocs_client_id_to_os_id FOREIGN KEY (client_id) REFERENCES oauth_clients(id);
```

----


#### 2.2.6. oauth_system_scopes 表（系统权限表）

| 列名                    | 数据类型         | 约束        | 索引   | 默认值 | 示例值         | 说明        |
| --------------------- | ------------ | --------- | ---- | --- | ----------- | --------- |
| **id**                | int          | 主键约束、自增属性 | 主键索引 | 自增  | 1           | ......... |
| **scope_name**        | varchar(100) |           |      |     | openid      | ......... |
| **scope_description** | varchar(100) |           |      |     | 申请 ID Token | ......... |

```
CREATE TABLE IF NOT EXISTS oauth_system_scopes (
    id INT AUTO_INCREMENT PRIMARY KEY,
    scope_name VARCHAR(100),
    scope_description VARCHAR(100)
);


INSERT INTO oauth_system_scopes (scope_name, scope_description) VALUES
  ('openid', '请求返回 OPENID Connect 身份令牌，其包含用户基本信息'),
  ('offline_access', '请求返回 Refresh Token'),
  ('user:info', '请求返回用户基本信息');
```

------


### 2.3. 进行 MyBatis 配置

<font color="#92d050">1. application.yaml 处</font>
```
spring:
  datasource:
    url: jdbc:mysql://192.168.136.8:3306/security?useUnicode=true&characterEncoding=utf8&serverTimezone=UTC
    username: root
    password: wq666666
    driver-class-name: com.mysql.cj.jdbc.Driver

mybatis:
  configuration:
    map-underscore-to-camel-case: true
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
  mapper-locations: classpath*:mapper/*.xml
  type-aliases-package: com.example.oauthserverwithmyproject.entity
```


<font color="#92d050">2. MybatisConfiguration 处</font>
```
@Configuration  
@MapperScan("com.example.oauthserverwithmyproject.mapper")  
public class MybatisConfiguration {  

}
```

----


### 2.4. 编写 Client Entity 类

Client 类位于 `com.example.oauthserverwithmyproject.entity` 包下
```

```

----


### 2.5. 编写 ClientMapper 接口

ClientMapper 类位于 `com.example.oauthserverwithmyproject.mapper` 包下
```

```

----


### 2.6. 编写 RedirectUrisTypeHandler 类

RedirectUrisTypeHandler 类位于 `com.example.oauthserverwithmyproject.handler` 包下
```
@MappedTypes(Set.class)
public class RedirectUrisTypeHandler extends BaseTypeHandler<Set<String>> {

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, Set<String> parameter, JdbcType jdbcType) throws SQLException {
        if (parameter == null || parameter.isEmpty()) {
            ps.setString(i, null);
        } else {
            ps.setString(i, parameter.iterator().next());
        }
    }

    @Override
    public Set<String> getNullableResult(ResultSet rs, String columnName) throws SQLException {
        String value = rs.getString(columnName);
        return value == null ? Collections.emptySet() : Collections.singleton(value);
    }

    @Override
    public Set<String> getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        String value = rs.getString(columnIndex);
        return value == null ? Collections.emptySet() : Collections.singleton(value);
    }

    @Override
    public Set<String> getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        String value = cs.getString(columnIndex);
        return value == null ? Collections.emptySet() : Collections.singleton(value);
    }
}
```

---


### 2.7. 编写 ClientSettingsTypeHandler 类

ClientSettingsTypeHandler 类位于 `com.example.oauthserverwithmyproject.handler` 包下
```
@MappedTypes(ClientSettings.class)  
public class ClientSettingsTypeHandler extends BaseTypeHandler<ClientSettings> {  
  
    private static final ObjectMapper objectMapper = new ObjectMapper();  
  
    @Override  
    public void setNonNullParameter(PreparedStatement ps, int i, ClientSettings parameter, JdbcType jdbcType) throws SQLException {  
        String json = null;  
        try {  
            json = objectMapper.writeValueAsString(parameter.getSettings());  
        } catch (JsonProcessingException e) {  
            throw new RuntimeException(e);  
        }  
        ps.setString(i, json);  
    }  
  
    @Override  
    public ClientSettings getNullableResult(ResultSet rs, String columnName) throws SQLException {  
        return fromJson(rs.getString(columnName));  
    }  
  
    @Override  
    public ClientSettings getNullableResult(ResultSet rs, int columnIndex) throws SQLException {  
        return fromJson(rs.getString(columnIndex));  
    }  
  
    @Override  
    public ClientSettings getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {  
        return fromJson(cs.getString(columnIndex));  
    }  
  
    private ClientSettings fromJson(String json) {  
        if (json == null || json.isBlank()) {  
            return ClientSettings.builder().build();  
        }  
        try {  
            Map<String, Object> map = objectMapper.readValue(json, new TypeReference<>() {});  
            return ClientSettings.withSettings(map).build();  
        } catch (Exception e) {  
            throw new RuntimeException("Failed to parse ClientSettings JSON", e);  
        }  
    }  
}
```

---


### 2.8. 编写 TokenSettingsTypeHandler 类

TokenSettingsTypeHandler 类位于 `com.example.oauthserverwithmyproject.handler` 包下
```
@MappedTypes(TokenSettings.class)
public class TokenSettingsTypeHandler extends BaseTypeHandler<TokenSettings> {

    private static final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, TokenSettings parameter, JdbcType jdbcType) throws SQLException {
        String json = null;
        try {
            json = objectMapper.writeValueAsString(parameter.getSettings());
        } catch (JsonProcessingException e) {
            throw new RuntimeException(e);
        }
        ps.setString(i, json);
    }

    @Override
    public TokenSettings getNullableResult(ResultSet rs, String columnName) throws SQLException {
        return fromJson(rs.getString(columnName));
    }

    @Override
    public TokenSettings getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        return fromJson(rs.getString(columnIndex));
    }

    @Override
    public TokenSettings getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        return fromJson(cs.getString(columnIndex));
    }

    private TokenSettings fromJson(String json) {
        if (json == null || json.isBlank()) {
            return TokenSettings.builder().build();
        }
        try {
            Map<String, Object> map = objectMapper.readValue(json, new TypeReference<>() {});
            return TokenSettings.withSettings(map).build();
        } catch (Exception e) {
            throw new RuntimeException("Failed to parse TokenSettings JSON", e);
        }
    }
}
```

----


































