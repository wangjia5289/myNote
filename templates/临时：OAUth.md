

### 实现 RegisteredClientRepository 接口

`RegisteredClientRepository` 接口 —— 是 Spring 授权服务器获取客户端信息的**唯一入口**你实现它，就是告诉 Spring：

> **“你以后别用内存了，我来给你数据库查 client。”**

```

```
非常好，咱们逐个拆解这三个方法是干什么用的，结合你用 Spring Authorization Server 的实际情况来看：

---

## 🌟 `RegisteredClientRepository` 接口 —— 是 Spring 授权服务器获取客户端信息的**唯一入口**

你实现它，就是告诉 Spring：

> **“你以后别用内存了，我来给你数据库查 client。”**

---

### 🔧 方法详解：

---

### `public void save(RegisteredClient registeredClient)`

#### ✅ 作用：

> 把一个 `RegisteredClient` 保存（insert）到数据库中。

#### 🧠 用途：

- 如果你允许系统**动态注册客户端**（比如 OpenID Connect 的动态 client registration），这个方法就会被调用。
    
- 在一些初始化脚本中也可能用这个方法保存预设客户端。
    

#### ❗你项目中：

- **一般不需要实现**，因为你可能是用手动 SQL 或初始化脚本插入客户端数据。
    
- 可以直接抛异常：
    

```java
@Override
public void save(RegisteredClient registeredClient) {
    throw new UnsupportedOperationException("客户端信息只读，禁止新增");
}
```

---

### `public RegisteredClient findById(String id)`

#### ✅ 作用：

> 通过 **RegisteredClient 的主键 ID** 查找 client。

#### 🧠 用途：

- Spring 授权服务器在处理令牌请求时，**某些流程会用到 ID 来定位 RegisteredClient**（而不是 client_id）。
    

#### 📌 举例：

- 一些授权流程内部用的是 UUID 主键（你表里的 `id` 字段）查数据。
    

#### 💡 建议：

- 实现这个方法，调用你的 `ClientMapper.selectById(id)`。
    

---

### `public RegisteredClient findByClientId(String clientId)`

#### ✅ 作用：

> 根据客户端提交的 `client_id` 查找 RegisteredClient。

#### 🧠 用途：

- **这是使用频率最高的方法！**
    
- OAuth2 客户端认证时，会通过 `client_id` 来找对应的 client，然后比对 `client_secret`、判断授权类型、Token 有效期等。
    

#### ✅ 必须实现：

```java
@Override
public RegisteredClient findByClientId(String clientId) {
    Client client = clientMapper.selectByClientId(clientId);
    return convertToRegisteredClient(client);
}
```

---

## ✅ 总结一下三者职责对比：

|方法|用途|必须实现？|建议|
|---|---|---|---|
|`save(RegisteredClient)`|保存 client 到数据库|❌|一般项目中不使用，可以抛异常|
|`findById(String id)`|通过主键 ID 查 client（Spring 内部可能调用）|✅|实现|
|`findByClientId(String clientId)`|通过 client_id 查 client（最常用）|✅|必须实现|

---

### ✅ 下一步建议：

我可以帮你把这三个方法都实现好，配合你的 `ClientMapper` 和实体类，写出完整的 `RegisteredClientRepository` 实现类。你愿意贴一下你的 `Client` 表结构或实体类代码吗？我就能生成最贴合你的项目的实现。





### 前端发送来的格式

```
{
  "id": "abc123",
  "clientId": "my-client",
  "clientIdIssuedAt": "2025-07-14T06:00:00Z",
  "clientSecret": "secret",
  "clientSecretExpiresAt": "2025-12-31T23:59:59Z",
  "clientName": "Test Client",
  "redirectUris": ["https://example.com/callback"],
  "clientSettings": {
    "requireProofKey": true,
    "requireAuthorizationConsent": false
  },
  "tokenSettings": {
    "accessTokenTimeToLive": 300,
    "refreshTokenTimeToLive": 1209600
  },
  "authenticationMethods": [
    {"value": "client_secret_basic"}
  ],
  "authorizationGrantTypes": [
    {"value": "authorization_code"},
    {"value": "refresh_token"}
  ],
  "scopes": [
    "read", "write"
  ]
}
```


































好问题，咱们继续接着锤：

---

## ✅ 你在数据库中存的是秒（`token_settings` 字段里的 JSON 中是秒），**Spring 拿出来的时候，会自动还原成 `Duration` 类型**，不需要你自己换单位。

---

### 🔧 举个完整流程：

你代码里配置的是：

```java

TokenSettings.builder()

.accessTokenTimeToLive(Duration.ofMinutes(5)) // Duration 类型

.build();

```

Spring 存进数据库的是 JSON：

```json

{

"access_token_time_to_live": 300 // 秒

}

```

当 Spring Authorization Server 启动或从数据库加载这个 `RegisteredClient` 的时候，它会自动做：

```java

Duration duration = Duration.ofSeconds(300);

```

然后再还原成 `TokenSettings` 对象：

```java

TokenSettings tokenSettings = RegisteredClient.getTokenSettings();

Duration ttl = tokenSettings.getAccessTokenTimeToLive(); // Duration.ofSeconds(300)

System.out.println(ttl.toMinutes()); // 打印：5

```

---

### ✅ 换句话说：

| 阶段 | 数据类型 | 示例值 |

| ---------- | ----------- | ---------------------------------- |

| Java 写入 | `Duration` | `Duration.ofMinutes(5)` |

| 存入数据库 JSON | `Number`（秒） | `300` |

| Java 读出 | `Duration` | `Duration.ofSeconds(300)` ← 自动转换回来 |

---

### ✅ 你不需要自己转换单位

你在业务代码中拿到的永远是 `Duration` 类型的，Spring 帮你完成了：

> JSON 中的秒值 → Duration 实例（即使你写的是分钟、小时、天，都会自动还原）

---

### 🧠 顺便一嘴：

这个过程由 Spring 的 `OAuth2AuthorizationServerJackson2Module` + `RegisteredClientDeserializer` 处理，里面专门有：

```java

Duration.ofSeconds(jsonNode.get("access_token_time_to_live").asLong());

```

---

所以你只管安心用 `tokenSettings.getXXXTimeToLive()` 拿，**永远是标准 Java 的 `Duration` 类型，单位自动搞定，不用你手动除以 60 了。**





















# 一、理论


## Spring Authorization Server 执行流程


<font color="#92d050">4. UsernamePasswordAuthenticationFilter 介入</font>




### OAuth2AuthorizationEndpointFilter 介入






| 参数名                   | 是否必须 | Spring 默认处理情况                               | 后端需配置点                       | 备注                                             |
| --------------------- | ---- | ------------------------------------------- | ---------------------------- | ---------------------------------------------- |
| response_type         | 是    | 必须是 `code`，否则报错 `unsupported_response_type` | 默认只支持授权码模式，默认即可，无需额外配置       | 目前只支持授权码模式                                     |
| client_id             | 是    | 必须且要能在 RegisteredClientRepository 找到对应客户端   | 注册客户端时必须配置 `clientId`        | 否则报 `unauthorized_client`                      |
| redirect_uri          | 有条件  | 如果客户端注册多个 redirect_uri，必须传且完全匹配             | 注册客户端时配置 `redirectUri`（支持多个） | 只允许注册的 URI，防止重定向攻击                             |
| scope                 | 否    | 如果传，校验是否为注册客户端支持的 scope；没传用注册全部             | 注册客户端时配置支持的 `scope`          | 传了非法 scope 报错 `invalid_scope`                  |
| state                 | 否    | 直接原样返回，无需校验                                 | 无需配置                         | 防止 CSRF，建议前端传                                  |
| code_challenge        | 条件   | PKCE 开启时必须传，否则报错                            | 注册客户端时配置支持 PKCE              | PKCE 相关参数，增强安全                                 |
| code_challenge_method | 条件   | PKCE 开启时校验支持的算法，默认 `plain`、`S256`           | 注册客户端时配置支持 PKCE              | 与 code_challenge 配合使用                          |
| 自定义参数（如 `foo`）        | 否    | 默认忽略，不做处理                                   | 需自定义实现校验逻辑                   | 需扩展 `OAuth2AuthorizationRequestValidator` 或解析器 |





下面是一份对**OAuth 2.0 授权请求**可携带参数，以及在开启 OIDC 后新增参数的**全面汇总**。表格分为两部分：第一部分是标准 OAuth 2.0 参数，第二部分是 OpenID Connect 在授权请求层面引入的扩展参数。对每个参数都说明了其作用，以及 Spring Authorization Server 默认如何处理或可在哪些环节自定义。

---

## 一、标准 OAuth 2.0 授权请求参数

|参数|必选/可选|含义 & 作用|Spring 默认处理|
|---|---|---|---|
|`response_type`|必选|指定授权类型，如 `code`（授权码模式）或 `token`（隐式）|由 `OAuth2AuthorizationEndpointFilter` 校验，生成对应 `Authentication`|
|`client_id`|必选|客户端注册时分配的 ID|校验是否已在 `RegisteredClientRepository` 中注册|
|`redirect_uri`|如果注册时未预先声明则必选否则可选|授权完成后回调地址|校验与注册信息一致；可自定义 `RedirectUriValidator`|
|`scope`|可选（推荐）|请求的权限范围，如 `read write`|校验在 `RegisteredClient` 所允许的范围内，并注入到 `OAuth2AuthorizationConsent`|
|`state`|可选（强烈推荐）|防 CSRF、维护前后端状态一致|原样透传，之后在回调时由 Spring Security 自动校验|
|PKCE（授权码模式推荐）：||||
|– `code_challenge`|可选（强烈推荐）|PKCE Code Challenge|被 `OAuth2AuthorizationEndpointFilter` 收集并保存到授权请求上下文|
|– `code_challenge_method`|可选|摘要算法，如 `S256`|同上|
|扩展参数（RFC 8628、RFC 9068 等）||||
|`request`|可选|JWT 格式的整个授权请求|Spring 暂不默认支持，需要自定义 `OAuth2AuthorizationEndpointConfigurer`|
|`request_uri`|可选|指向已托管请求对象的 URI|同上|

> **自定义能力**
> 
> - 可以通过 `AuthorizationServerSettings` 定制端点 URL；
>     
> - 通过 `OAuth2AuthorizationEndpointConfigurer` 注册自定义参数校验器或解析器；
>     
> - 支持为客户端精细配置是否必须开启 PKCE、是否允许 `request` 等扩展。
>     

---

## 二、OpenID Connect 扩展参数

|参数|必选/可选|含义 & 作用|Spring 默认处理|
|---|---|---|---|
|`scope=openid`|必选（OIDC）|声明这是一次身份认证请求，会返回 ID Token|与标准 `scope` 一并校验；触发 `OidcAuthorizationEndpointFilter`|
|`nonce`|必选（OIDC）|防重放攻击：客户端自定义字符串，原样写入 ID Token 并在客户端校验|自动注入到授权请求上下文；最终写入 `IdToken` payload|
|`prompt`|可选|控制用户交互方式，如 `none`、`login`、`consent`、`select_account`|由 Spring 解析并映射到 `Prompt` 枚举，可用于自定义登录/同意逻辑|
|`max_age`|可选|指定用户认证的最大有效时长（秒），超过则强制重新登录|解析为 `max_age` 属性，和当前会话时间比对，触发再认证流程|
|`display`|可选|指定 UI 展示模式，如 `page`、`popup`、`touch`、`wap`|仅解析参数；前端可据此调整渲染方式|
|`login_hint`|可选|向认证服务器提示首选登录用户名或 ID|自动填充到登录页面输入框（需前端支持）|
|`id_token_hint`|可选|向授权服务器传入先前的 ID Token，用于单点登出或会话管理|Spring 可验证其有效性并据此实现 RP-Initiated Logout|
|`acr_values`|可选|指定认证上下文类参考值，如多因子认证等级|解析成列表，开发者可据此在自定义认证管理器中选择不同认证策略|
|`ui_locales`|可选|指定用户界面语言和区域，格式如 `en-US fr-CA`|解析并注入到 `OidcAuthorizationRequest`，前端据此渲染本地化内容|
|`claims`|可选|声明请求的用户信息字段（细粒度控制）|解析 JSON，对应 `RequestedClaim`，后续可在 `UserInfo` 或 ID Token 中返回|

> **OIDC 配置要点**
> 
> - 在 `AuthorizationServerSettings` 中可定制 `/userinfo`、`/jwks` 等路径；
>     
> - Spring Authorization Server 会自动注册 `OidcAuthorizationEndpointFilter`、`OidcUserInfoEndpointFilter` 等，完成参数校验与处理；
>     
> - 如需支持更多自定义声明、策略或 UI 格式，可继承相关 Filter/Configurer 并替换默认实现。
>     

---

### 小结

1. **OAuth 2.0 阶段**：只需关注 `response_type`、`client_id`、`redirect_uri`、`scope`、`state` 以及 PKCE 相关参数；
    
2. **开启 OIDC 后**：在上述基础上必须带上 `scope=openid` 和 `nonce`，并可根据需求添加 `prompt`、`max_age`、`claims` 等扩展；
    
3. **Spring 处理**：核心由 `OAuth2AuthorizationEndpointFilter` 和 `OidcAuthorizationEndpointFilter` 负责解析、校验并将参数注入到内部的授权请求对象里，后续由授权决策与令牌服务统一处理。
    

这份汇总既展示了协议层面的可选项，也标注了 Spring Authorization Server 的默认支持和可扩展点。希望能助你快速掌握授权请求的全貌！













## Spring Authorization Server 配置

### Spring Authorization Server 配置模板

AuthServerConfiguration 类在 `com.example.oauthserverwithmyproject.configuration` 包下，直接粘贴这份配置模板，再根据下方的详细说明按需进行调整。




### 配置端点路径

```
@Bean  
public AuthorizationServerSettings authorizationServerSettings() {  
    return AuthorizationServerSettings.builder()  
            .authorizationEndpoint("/custom/authorize")  
            .tokenEndpoint("/custom/token")  
            .clientRegistrationEndpoint("/custom/register")  
            .tokenRevocationEndpoint("/custom/revoke")  
            .tokenIntrospectionEndpoint("/custom/introspect")  
            .jwkSetEndpoint("/custom/jwks")  
            .build();  
}
```

<font color="#92d050">1. authorizationEndpoint</font>
指定授权端点，用于颁发授权令牌，默认是 /oauth2/authorize


<font color="#92d050">2. tokenEndponit</font>
指定令牌端点，用于颁发访问令牌，默认是/oauth2/token


<font color="#92d050">3. clientRegistrationEndpoint</font>
指定客户端注册端点，需要如果启用动态客户端注册，默认是/oauth2/register


<font color="#92d050">4. tokenRevocationEndpoint</font>
授权撤销端点允许客户端撤销访问令牌或刷新令牌，默认是/oauth2/revoke


<font color="#92d050">5. tokenIntrospectionEndpoint</font>
令牌校验缎带你，允许资源服务器校验访问令牌的有效性和状态，默认为：/oauth2/introspect


<font color="#92d050">6. jwkSetEndpoint</font>
JWK Set 端点，公开服务器的公钥，用于 JWT 验签，默认为/oauth2/jwks


<font color="#92d050">7. oidcUserInfoEndpoint</font>
默认为/userinfo

> [!NOTE] 注意事项
> 1. 这些路径建议保持默认，方便达成一致和统一




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

虽然我们没有将authenticationmanager 声明为Bean，但是他会在内部内部自动构造一个 `ProviderManager` 作为 `AuthenticationManager`


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















## 客户端注册放到数据库中

抱歉，刚才讲得太散了，让你觉得乱。我这次给你条理清晰、重点突出的版本，帮你一步步搞定用 MyBatis 把 RegisteredClient 存到数据库的事儿。

---

### 目标

把你现在在 `registeredClientRepository()` 里写死的客户端，改成存到数据库里，支持：

- 新增客户端（保存）
    
- 查询客户端（按 `clientId` 或 `id`）
    
- 更新客户端信息
    

---

### 核心步骤

1. **设计数据库表**，存 RegisteredClient 相关字段
    
2. **写实体类**映射表结构
    
3. **写 MyBatis Mapper 接口和 XML**，实现增删改查
    
4. **写 RegisteredClientRepository 实现类**，做数据库和 RegisteredClient 对象互转
    
5. **配置 Spring，把自定义 RegisteredClientRepository 注入**
    
6. **调用你的 repo 来操作数据库里的客户端**
    

---

### 1. 数据库表设计（MySQL）

```sql
CREATE TABLE oauth2_registered_client (
  id VARCHAR(100) PRIMARY KEY,
  client_id VARCHAR(100) UNIQUE NOT NULL,
  client_secret VARCHAR(200),
  client_authentication_methods TEXT,
  authorization_grant_types TEXT,
  redirect_uris TEXT,
  scopes TEXT,
  client_settings TEXT,
  token_settings TEXT
);
```

---

### 2. 实体类 RegisteredClientEntity

```java
public class RegisteredClientEntity {
    private String id;
    private String clientId;
    private String clientSecret;
    private String clientAuthenticationMethods;  // JSON字符串
    private String authorizationGrantTypes;      // JSON字符串
    private String redirectUris;                  // JSON字符串
    private String scopes;                        // JSON字符串
    private String clientSettings;                // JSON字符串
    private String tokenSettings;                 // JSON字符串

    // getters 和 setters
}
```

---

### 3. MyBatis Mapper 接口和 XML

接口 `RegisteredClientMapper.java`：

```java
@Mapper
public interface RegisteredClientMapper {
    void insert(RegisteredClientEntity client);
    RegisteredClientEntity selectById(String id);
    RegisteredClientEntity selectByClientId(String clientId);
    void update(RegisteredClientEntity client);
}
```

XML 映射 `RegisteredClientMapper.xml` 就是对表的增删改查语句，跟之前给你的类似。

---

### 4. RegisteredClientRepository 实现类

```java
@Component
public class MyBatisRegisteredClientRepository implements RegisteredClientRepository {

    private final RegisteredClientMapper mapper;
    private final ObjectMapper objectMapper = new ObjectMapper();

    public MyBatisRegisteredClientRepository(RegisteredClientMapper mapper) {
        this.mapper = mapper;
    }

    @Override
    public void save(RegisteredClient client) {
        RegisteredClientEntity entity = toEntity(client);
        if (mapper.selectById(client.getId()) == null) {
            mapper.insert(entity);
        } else {
            mapper.update(entity);
        }
    }

    @Override
    public RegisteredClient findById(String id) {
        RegisteredClientEntity entity = mapper.selectById(id);
        return entity == null ? null : toRegisteredClient(entity);
    }

    @Override
    public RegisteredClient findByClientId(String clientId) {
        RegisteredClientEntity entity = mapper.selectByClientId(clientId);
        return entity == null ? null : toRegisteredClient(entity);
    }

    private RegisteredClientEntity toEntity(RegisteredClient client) {
        RegisteredClientEntity entity = new RegisteredClientEntity();
        entity.setId(client.getId());
        entity.setClientId(client.getClientId());
        entity.setClientSecret(client.getClientSecret());
        try {
            entity.setClientAuthenticationMethods(objectMapper.writeValueAsString(
                    client.getClientAuthenticationMethods().stream()
                            .map(ClientAuthenticationMethod::getValue)
                            .collect(Collectors.toSet())));
            entity.setAuthorizationGrantTypes(objectMapper.writeValueAsString(
                    client.getAuthorizationGrantTypes().stream()
                            .map(AuthorizationGrantType::getValue)
                            .collect(Collectors.toSet())));
            entity.setRedirectUris(objectMapper.writeValueAsString(client.getRedirectUris()));
            entity.setScopes(objectMapper.writeValueAsString(client.getScopes()));
            entity.setClientSettings(objectMapper.writeValueAsString(client.getClientSettings().getSettings()));
            entity.setTokenSettings(objectMapper.writeValueAsString(client.getTokenSettings().getSettings()));
        } catch (Exception e) {
            throw new RuntimeException("序列化 RegisteredClient 失败", e);
        }
        return entity;
    }

    private RegisteredClient toRegisteredClient(RegisteredClientEntity entity) {
        try {
            RegisteredClient.Builder builder = RegisteredClient.withId(entity.getId())
                    .clientId(entity.getClientId())
                    .clientSecret(entity.getClientSecret());

            Set<String> authMethods = objectMapper.readValue(
                    entity.getClientAuthenticationMethods(), new TypeReference<Set<String>>() {});
            authMethods.forEach(m -> builder.clientAuthenticationMethod(new ClientAuthenticationMethod(m)));

            Set<String> grantTypes = objectMapper.readValue(
                    entity.getAuthorizationGrantTypes(), new TypeReference<Set<String>>() {});
            grantTypes.forEach(g -> builder.authorizationGrantType(new AuthorizationGrantType(g)));

            List<String> redirectUris = objectMapper.readValue(
                    entity.getRedirectUris(), new TypeReference<List<String>>() {});
            redirectUris.forEach(builder::redirectUri);

            Set<String> scopes = objectMapper.readValue(
                    entity.getScopes(), new TypeReference<Set<String>>() {});
            scopes.forEach(builder::scope);

            Map<String, Object> clientSettings = objectMapper.readValue(
                    entity.getClientSettings(), new TypeReference<Map<String, Object>>() {});
            builder.clientSettings(ClientSettings.withSettings(clientSettings).build());

            Map<String, Object> tokenSettings = objectMapper.readValue(
                    entity.getTokenSettings(), new TypeReference<Map<String, Object>>() {});
            builder.tokenSettings(TokenSettings.withSettings(tokenSettings).build());

            return builder.build();
        } catch (Exception e) {
            throw new RuntimeException("反序列化 RegisteredClientEntity 失败", e);
        }
    }
}
```

---

### 5. 在 Spring 配置类里注入这个实现

```java
@Configuration
@MapperScan("com.example.auth.mapper") // 你的 Mapper 包路径
public class AuthorizationServerConfig {

    @Bean
    public RegisteredClientRepository registeredClientRepository(MyBatisRegisteredClientRepository repo) {
        return repo;
    }

    // 其他授权服务器安全配置...
}
```

---

### 6. 用法示例

- **保存新客户端**
    

```java
RegisteredClient client = RegisteredClient.withId(UUID.randomUUID().toString())
    .clientId("my-client")
    .clientSecret("{noop}secret")
    .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
    .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
    .redirectUri("http://localhost:8080/callback")
    .scope("read")
    .clientSettings(ClientSettings.builder().requireAuthorizationConsent(true).build())
    .build();

registeredClientRepository.save(client);
```

- **查询客户端**
    

```java
RegisteredClient c = registeredClientRepository.findByClientId("my-client");
```

- **更新客户端**
    

```java
RegisteredClient oldClient = registeredClientRepository.findByClientId("my-client");
RegisteredClient updated = RegisteredClient.from(oldClient)
    .clientSecret("{noop}newSecret")
    .build();
registeredClientRepository.save(updated);
```

---

这就是核心流程，想明白了，实际代码写起来并不复杂。如果你想，我也可以帮你写一个完整可运行的项目 demo，你看怎么样？

----



## OIDC 到底规范了什么

OIDC（OpenID Connect）是在 OAuth 2.0 的基础上进行扩展和规范的一种协议，主要为了解决身份认证问题。它在 OAuth 2.0 的授权框架之上，增加了一系列标准化的功能和规范，以下是大总结：

### 1. **引入身份认证功能**
- OIDC 在 OAuth 2.0 的授权基础上增加了身份认证的能力，允许客户端在用户授权后获取用户的身份信息。
- **核心创新**：定义了 `id_token`，这是一个基于 JWT（JSON Web Token）的令牌，包含用户的身份信息（如唯一标识符 subject identifier）。

### 2. **标准化 Token 格式**
- OAuth 2.0 未定义 access token 的具体格式，而 OIDC 明确规定 `id_token` 必须采用 JWT 格式，确保其结构化和可验证性。
- access token 和 refresh token 的格式未强制要求，但与 `id_token` 配合使用。

### 3. **新增 UserInfo Endpoint**
- OIDC 引入了 UserInfo Endpoint，客户端可通过 access token 从该端点获取用户的详细信息（如姓名、邮箱等），增强了用户信息获取的标准化途径。

### 4. **定义标准 Scope**
- OIDC 规范了标准的 scope 参数，例如：
  - `openid`：表示这是一个 OIDC 请求，必须包含。
  - `profile`、`email` 等：用于请求用户的具体信息。
- 这些 scope 使权限请求更加清晰和一致。

### 5. **增强授权请求**
- 引入 `response_type` 参数，指定返回的令牌类型（如 `code`、`id_token`、`token`），支持不同的认证流程。
- 增加 `nonce` 参数，防止重放攻击，提升安全性。

### 6. **提供发现机制**
- OIDC 定义了发现机制，客户端可以通过标准化的方式动态获取 OIDC 提供者的配置信息和端点（如授权端点、令牌端点等）。

### 7. **支持动态客户端注册**
- 允许客户端在运行时向 OIDC 提供者注册，增加了灵活性。

### 8. **会话管理**
- 提供会话管理功能，例如检查用户登录状态、结束会话等，使认证过程更完整。

### 9. **安全性提升**
- 使用 JWT 和数字签名确保 `id_token` 的完整性和真实性。
- 引入 `state` 参数防止 CSRF（跨站请求伪造）攻击。

### 10. **标准化错误响应**
- OIDC 规范了错误响应的格式和内容，使错误处理更加统一和可预测。

### 11. **支持多种认证流程**
- 包括授权码流程、隐式流程和混合流程，适应不同应用场景的需求。

### 12. **用户同意机制**
- 在授权过程中，用户可以查看并同意客户端请求的权限，提升用户控制权。

### 13. **其他扩展功能**
- 支持多租户环境，适用于同一提供者服务多个租户的场景。
- 支持国际化，允许 token 和用户信息使用多语言。
- 提供扩展性，可通过自定义 claims 或额外令牌类型满足特定需求。

### 总结
OIDC 在 OAuth 2.0 的基础上，通过引入身份认证流程（`id_token`）、标准化令牌格式（JWT）、新增 UserInfo Endpoint、定义标准 scope、增强安全性、支持多种流程等方式，构建了一个更加规范、完整和安全的身份认证框架。它不仅保留了 OAuth 2.0 的授权能力，还为现代应用程序提供了一种标准化的用户身份验证解决方案。




















## Spring Security OAuth 授权服务器配置

### Spring Security OAuth 授权服务器配置模板


#### 配置表单登录










# 二、实操

## 使用 Spring Security 开发授权码模式 + PKCE + OIDC + Refresh Token 一条龙系统

### 开发授权服务器

#### 













































# 一、理论

## 1. 导图

![](source/_posts/笔记：OAuth%20协议/Map：OAuth%20协议.xmind)

---
## 2. OAuth2 协议

OAuth 表示授权（Authorization），O 表示开放（Open），OAuth2 是一种开放式的授权协议，用于在不暴露用户凭据的前提下，让第三方应用获得访问权限，OAuth2 有四种授权模式：
1. 授权码模式（Authorization Code）
2. 隐藏式（Implicit）
3. 密码式（Password）
4. 客户端凭证模式（Client Credentials）

![](image-20250706092222409.png)

> [!NOTE] 注意事项
> 1. 客户端应用可能是：前端、后端、移动 APP、微信小程序等

---


## 3. OAuth2.1 协议

OAuth 2.1 目前仍处于草案阶段，但正逐渐被广泛接受，有望成为主流。其较于 OAuht2 的主要变化包括：
1. 对授权码模式（Authorization Code）进行了增强，扩展了 PKCE（授权码交换的密钥证明）。
	1. 需要注意的是，OAuth 2.1 明确指出：**只要使用授权码模式，必须同时使用 PKCE**，以提升安全性
2. 移除了不再安全的隐藏式（Implicit）和密码式（Password）授权模式，新增了刷新令牌（Refresh Token）配套模式，OAuth 2.1 最终保留两种主流授权模式：
	1. 授权码模式（Authorization Code）+ PKCE（授权码交换的密钥证明）
	2. 客户端凭证模式（Client Credentials）
	3. 刷新令牌模式（Refresh Token）

> [!NOTE] 注意事项
> 1. Spring Security 目前既能支持 OAuth 2.1 的两种主流模式，也仍保留对旧有隐式和密码模式的兼容支持
> 2. 这是因为很多老系统还在用这两种不安全的授权模式，但官方安全建议是新项目应避免使用隐式和密码模式，并且尽量使用授权码 + PKCE 来保障安全
> 3. 刷新令牌配套模式，是指在使用授权码模式或密码模式时，授权服务器在发放访问令牌的同时也会一并发放刷新令牌。刷新令牌的有效期通常长于访问令牌，当访问令牌过期后，客户端可通过刷新令牌重新获取新的访问令牌，以延续会话而无需重新授权。

----


## 2.2. OAuth 协议中的角色

![](PixPin_2025-07-06_16-38-42.png)

----


### 理想状态下，授权码模式（Authorization Code）+ PKCE（授权码交换的密钥证明）+OIDC（身份认证协议）+ Refresh Token（刷新令牌）的流程

#### 基本流程

OAuth2 的授权码模式（Authorization Code Grant）是目前最常见、最安全、同时也是实现最复杂的一种授权方式，尤其适用于前后端分离的系统架构。

以 Gitee 使用第三方 GitHub 登录为例，在整个授权流程中，Gitee 作为 OAuth 客户端，而 GitHub 同时提供授权服务器与资源服务器的角色（需要注意的是，GitHub 并不是一个理想的授权服务器，例如其不支持 PKCE、OIDC，这里只是假设他为理想的情况），整体流程如下所示：

<font color="#92d050">1. Gitee 需要在 GitHub 中注册</font>
在这个过程中，Gitee 需要填写其客户端应用的信息，例如名称、首页地址、回调地址等。注册成功后，GitHub 会返回 Client ID 和 Client Secret，这两个参数是 GitHub 用于识别和验证 Gitee 客户端身份的依据。


<font color="#92d050">2. 用户在 Gitee 前端页面，点击 GitHub 登录</font>
![](image-20250707084023923.png)


<font color="#92d050">3. Gitee 前端向 Gitee 后端请求 GitHub 授权 URL</font>
其代码可能类似于这样：
```
	// 1. Gitee 前端调用 Gitee 后端，请求 GitHub 授权 URL
const res = await fetch('/api/oauth/github/authorize');


// 2. 从响应体 res 中解析出 JSON 并解构出 authUrl 变量
const { authUrl } = await res.json();
```


<font color="#92d050">4. Gitee 后端返回给前端 GitHub 授权 URL</font>
在此流程中，Gitee 需生成 PKCE 参数（codeChallengeMethod、codeVerifier、codeChallenge）和 CSRF Token，并保存 codeVerifier 与 CSRF Token，随后构造 GitHub 授权 URL。其代码可能类似于这样：
```
@GetMapping("/api/oauth/github/authorize")
public ResponseEntity<?> authorize() {

    // 1. 生成 PKCE 参数和 CSRF Token
	String codeChallengeMethod = "S256"; // 标识使用 SHA-256 对 codeVerfier 进行加密
    String codeVerifier = xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx; // 生成 codeVerfier
    String codeChallenge = sha256AndBase64Url(codeVerifier); // 先对 codeVerfier 进行 SHA-256 加密，再用 Base64 URL 进行安全编码，生成 codeChallenge
    String state = UUID.randomUUID().toString(); // 生成 CSRF Token


    // 2. 保存 codeVerfier、CSRF Token，常保存到 Redis 等


    // 3. 构造 GitHub 授权 URL
    String authUrl = UriComponentsBuilder.fromHttpUrl("https://github.com/login/oauth/authorize")
            .queryParam("client_id", "xxx")
            .queryParam("redirect_uri", "https://gitee.com/auth/github/callback")
            .queryParam("response_type", "code")
            .queryParam("scope", "openid user email")
            .queryParam("state", state)
            .queryParam("code_challenge", codeChallenge)
            .queryParam("code_challenge_method", codeChallengeMethod)  // 用变量替代硬编码
            .build().toUriString();


    // 4. 返回给前端跳转
    return ResponseEntity.ok(Map.of("authUrl", authUrl));
}
```

> [!NOTE] 注意事项
> 1. GitHub 授权 URL 不仅包括 GitHub 授权服务器提供的授权接口 API（`/login/oauth/authorize`），还需要携带一些参数，常携带的参数有：
> 	1. client_id
> 		1. 在 GitHub 注册时，GitHub 返回的 Client ID
> 	2. redirect_uri
> 		1. 在 GitHub 注册时，指定的 Gitee 后端回调地址
> 	3. response_type
> 		1. 告诉 GitHub 授权服务器你要进行什么类型的授权，常见类型有：
> 			1. code：
> 				1. 授权码模式，GitHub 将同时返回访问令牌和刷新令牌
> 	4. scop
> 	5. state
> 	6. code_challenge
> 	7. code_challenge_method
> 2. 之所以说 `/login/oauth/authorize` 是后端的接口 API，而不是前端的页面 URI，是因为授权服务器本身并不是前后端分离的架构，因此该路径由后端统一处理授权流程并返回相应页面，而非前端控制的路由页面


<font color="#92d050">5. 前端跳转到 GitHub 授权 URL</font>
其代码可能是这样：
```
window.location.href = authUrl;
```


<font color="#92d050">6. 用户在 GitHub 上进行身份认证和授权</font>
需要注意的是，在开发授权服务器时，并不推荐采用前后端分离的架构，而是 SpringBoot + Thymeleaf

以 GitHub 的授权 API 为例（`/login/oauth/authorize`），它既能判断用户是否已完成身份认证：若未认证，则先返回登录页面，完成登录后再跳转至授权确认页面；若已认证，则直接返回授权确认页面。其代码可能是这样：
```
@GetMapping("/login/oauth/authorize")
public void handleAuthorize(HttpServletRequest request, HttpServletResponse response) {
    OAuth2Request oauthRequest = parseRequest(request);
    
	// 1. 保存 code_challenge 和 code_challenge_method，通常保存在 Redis

    // 1. 判断用户是否已登录
    User user = getCurrentUser(request);
    
    // 2. 未登录：保存授权请求（通常放 HttpSession）并跳转到登录 API（逻辑 + 页面），登录成功后再由登录 API 跳转到授权确认 API（逻辑 + 页面）
    if (user == null) {
        request.getSession().setAttribute("OAUTH_REQUEST", oauthRequest);
        response.sendRedirect("/login");
        return; // 终止方法，不继续往下进行，由于方法的返回类型是 void，这里的 return; 什么都不返回
    }

    // 3. 已登录：是否已授权过？
    if (!hasUserAuthorized(user, oauthRequest)) {
        // 4. 跳转到授权确认页，授权确认后，发授权码，并跳转到 Gitee 的回调 URL
        renderConsentPage(user, oauthRequest, response);
    } else {
        // 5. 已授权：直接发授权码，并跳转到 Gitee 的回调 URL
        String code = generateAuthorizationCode(user, oauthRequest);
        response.sendRedirect(buildRedirectUri(oauthRequest.redirectUri, code, oauthRequest.state));
    }
}
```

> [!NOTE] 注意事项
> 1. 判断用户是否已登录的时候，我们传统的的登录方式，后端生成 JWT，前端手动把 JWT 放到请求头或者请求体发送到后端的方式就不行了，因为你是直接调用的 GitHub 授权服务器的后端 API，根本没有 GitHub 授权服务器的前端把用户的 JWT 发送过来，甚至说的过分一点，GitHUb 授权服务器是前后端不分离的，根本就没有 GitHub 前端
> 2. 那我们传统的 JWT 其实是解决不了这个问题，我们可以使用 RememberMe Cookie，或者如果你非用 JWT 的话，你可把 JWT 放到 Cookie 中，根据浏览器自动携带 Cookie 的特性进行完成


<font color="#92d050">7. GitHub 授权后，携带授权码、state 重定向回 Gitee 指定的回调地址（该地址为后端地址）</font>
其可能为：
```
HTTP/1.1 302 Found
Location: https://gitee.com/auth/github/callback?
    code=gho_16C7e42F292c6912E7710c838347Ae178B4a&
    state=81fda4a8066fb1ea3310d3bf577ece61a8e0286c03f82c91
```


8. Gitee 前端访问 GitHub 授权服务器提供的授权 API 获取 访问令牌

验证 state ，防止 CSRF 攻击，然后

```
    @GetMapping("/auth/github/callback")
    public ResponseEntity<?> githubCallback(@RequestParam("code") String code,
                                            @RequestParam("state") String state) {
        // 1. 验证 state（防止 CSRF）
        if (!isValidState(state)) {
            return ResponseEntity.status(HttpStatus.BAD_REQUEST).body("Invalid state");
        }

        // 2. 组装请求，向 GitHub 换取 access_token
        RestTemplate restTemplate = new RestTemplate();

        MultiValueMap<String, String> params = new LinkedMultiValueMap<>();
        params.add("client_id", CLIENT_ID);
        params.add("client_secret", CLIENT_SECRET);
        params.add("code", code);
        params.add("redirect_uri", REDIRECT_URI);
        params.add("code_verifier", getCodeVerifierFromSessionOrCache());

        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_FORM_URLENCODED);

        HttpEntity<MultiValueMap<String, String>> request = new HttpEntity<>(params, headers);

        ResponseEntity<String> response = restTemplate.postForEntity("https://github.com/login/oauth/access_token", request, String.class);

        // 3. 解析 GitHub 返回的 access_token 并做后续处理

        return ResponseEntity.ok("授权成功");
    }
```


9. GitHub 授权服务器验证 PKCE，返回访问令牌
他会拿 code_verfier 与我们刚才保存的 code_challenge_method 对 code_verdier 重新加密，与我们的 code_challenge 校验，然后，返回访问令牌：
```
{
    "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refresh_token": "def456...",
    "id_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
    "token_type": "Bearer",
    "expires_in": 1800,
    "scope": "openid profile read write"
}
```

<font color="#92d050">1. access_token</font>
用于访问受保护资源（API）的令牌,持有它的客户端可以携带它调用资源服务器接口，完成授权访问,一般是短期有效的，例如"expires_in": 1800 秒，，即 30 分钟


<font color="#92d050">2. refresh_token</font>
当 access_token 过期后，客户端可以用它向授权服务器请求新的 access_token，而不需要用户重新登录，有效期通常比 access_token 长很多


<font color="#92d050">3. id_token</font>
OIDC 专用的令牌，基于 JWT 格式，用来传递用户身份信息（认证信息），比如用户的唯一标识、登录时间、姓名、邮箱等，主要用于前端或客户端验证用户身份，实现单点登录（SSO）功能，不同于 access_token，id_token 不是用来调用 API 的

---


#### PKCE 扩展在本流程中的作用

当用户在 GitHub 授权页完成授权后，返回我们的回调地址时，会携带授权码（authorization code）。如果这个请求被劫持了，那么攻击者就可能拿着这段授权码，去 GitHub 的换令牌接口，换取我们的访问令牌（access token）。

为了防止这种情况，我们通常会在调用换取访问令牌的请求中额外携带 Client Secret，作为客户端的身份凭证。但对于移动应用来说，它们无法很好地保存这个 Client Secret。不是说是不能保存，而是**无法安全地保存**。因为这个值非常重要，一旦被放到前端，就有可能被劫持。而一旦泄露，攻击者就可以长期持有它，不断换取访问令牌。

Client Secret 的有效期非常长，除非 Gitee 官方手动从 GitHub 上注销，或者重置密钥，否则它始终有效，这是非常危险的。因此我们绝不能把 Client Secret 暴露在前端，所以在 Web 应用中，一般都会将换 token 的逻辑放在后端完成。

但移动端应用 / SPA 单页应用，没有后端，不得不放到前端，但是它们也无法安全保存 Client Secret，这时候我们该怎么办？我们既然不能用 Client Secret，那就得找到一种新的方式来保证安全，这正是 PKCE 在这里发挥关键作用的地方。

第一次访问授权页，发起授权请求时，客户端会携带一个随机字符串（Code Verifier），并用某种加密算法（Code Challenge Method）生成对应的 Code Challenge，然后将 Code Challenge 和 Code Challenge Method 一起发送给 GitHub 的授权服务端，GitHub 会将这些值与我们的授权码绑定保存。

等到后续客户端拿着授权码去换访问令牌的时候，需要把原始的 Code Verifier 一起提交上来。服务端根据我们的授权码，找到 Code Challenge Method 对 Code Verifier 进行加密处理，和保存的 Code Challenge 对比，如果一致，才会颁发访问令牌。

这个过程的意义，其实和 Client Secret 的作用很相似，那你可能会问，既然连 Client Secret 都不能好好保存了，难道 Code Verifier 就能安全保存吗？

其实 Code Verifier 和 Client Secret 最大的区别就在于：Code Verifier 是临时的、一次性的，用完即废，它只在换取 access_token 时使用一次，即便泄露也没有任何意义。而 Client Secret 是长期有效的，一旦泄露后果极其严重。

这就是 PKCE 最核心的意义：在没有 Client Secret 的情况下，只携带 Code Verifier，就能安全地换取 access_token。

> [!NOTE] 最理想的情况是
> 1. Web 应用，在换取访问令牌时，既携带 Client Secret，又携带 Code Verfier（即便携带 Code Verfier 并没有起到多少的额外作用）
> 2. 移动端应用 / SPA 单页应用，在换取访问令牌时，携带 Code Verfier

---


#### OIDC 规范在本流程中的作用

一般情况下，在原始的 OAuth 实现中，Access Token 的格式并不统一。有的厂商使用 JWT 格式，有的采用 UUID，还有的使用不可逆的加密字符串等等，形式多种多样，完全取决于厂商的实现方式。

如果厂商采用的是 JWT 格式，通常可以像我们传统使用 JWT 登录一样，直接对令牌进行解码，从中提取出用户名等信息，从而实现本地线程的登录和资源访问。

但如果厂商使用的是不可逆的加密字符串，那么他们需要保存这串令牌与用户之间的对应关系，一旦接收到这个字符串，就能识别出它对应的是哪个用户。

无论他们使用哪种格式，都需要


























<font color="#92d050">1. Gitee 前端页面发起请求，并跳转到 GitHub 授权页</font>
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


<font color="#92d050">2. GitHub 授权后，携带授权码重定向回 Gitee 后端地址</font>
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


<font color="#92d050">3. Gitee 后端携带 code，调用 GitHub 授权服务器 API，获取 Access Token</font>
其实我们可以注意到一个问题：在发起授权请求时，`Client Id` 是直接暴露在 URL 中的；授权成功后，GitHub 返回的 `code` 也同样出现在回调的 URL 中。那有没有可能，如果这些 URL 被第三方窃取，攻击者就能利用这些信息伪造客户端、换取 Access Token，甚至获取用户的私密信息？

如果只看表面，的确存在这样的风险。但是，别忘了我们在注册第三方应用时，一般会获得两个核心凭证：`Client Id` 和 `Client Secret`。

其中，`Client Secret` 正是在这个阶段起作用的：后端在用 `code` 向授权服务器（如 GitHub）换取 `access_token` 时，**必须携带 `Client Secret` 一并提交**，GitHub 会对其进行校验，以此确认调用方的身份是否合法

> [!NOTE] 注意事项
> 1. 之所以让后端发起换 token 请求，最根本的原因就是：**`client_secret` 绝不能暴露给前端**，一旦被暴露到前端，它就可能被窃取，而攻击者就能冒充你的客户端发起授权，窃取用户数据，这将是非常严重的安全漏洞
> 2. 在后端发起的请求中，一切通信都发生在服务器内部，不会暴露在用户浏览器或网络面前，从而**有效避免了密钥泄露的风险**
> 3. 这是 OAuth2 中授权码模式强调换 token 必须由后端发起的根本原因，也是为什么授权码模式适合前后端分离时使用的根本原因


<font color="#92d050">4. GitHub 授权服务器返回 Access Token</font>


<font color="#92d050">5. Gitee 后端携带 Access Token，调用 GitHub 资源服务器 API，获取 schope 指定的信息</font>


<font color="#92d050">6. GitHub 资源服务器返回对应信息</font>
























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

![|500](image-20250707104557256.png)

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

## 使用 Spring Security 实现授权码模式 + PKCE +OIDC +Refresh Token

### 开发 授权服务器 + 资源服务器（登录平台）

#### 开发 授权服务器

##### 1.2.1.4. 创建 Spring Web 项目，添加相关依赖

创建时：
1. Web
	1. Spring Web
2. Security
	1. Spring Security
	2. OAuth2 Authorization Server
3. SQL
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
> 2. 虽然 JJWT 也能生成 JWT，但它与 Spring Security 的集成度较低，许多功能（如 token 签发、校验、JWK 支持等）都需要我们手动实现。
> 3. 它与我们原先采用的 JJWT 库存在不少差异，不能直接沿用过去的写法，需要根据新库的方式进行修改。

----


##### 前置步骤

详见笔记：Spring Security 中，基于 JWT 的 Spring Security（1.2.3~1.2.4）

----


##### 编写 JWT 生成类

JwtUtil 类位于 `com.example.oauthserverwithmyproject.util` 包下
```
public class JwtUtil {

    private static final String SECRET = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789++";
    private static final SecretKeySpec SECRET_KEY = new SecretKeySpec(SECRET.getBytes(StandardCharsets.UTF_8), "HmacSHA512");

    private static final long EXPIRATION_TIME = 1000 * 60 * 60 * 24; // Token过期时间：1 天
    private static final String TOKEN_PREFIX = "Bearer ";
    private static final String HEADER_STRING = "Authorization";

    private static final JwtEncoder jwtEncoder = new NimbusJwtEncoder(new ImmutableSecret<>(SECRET_KEY));
    private static final JwtDecoder jwtDecoder = NimbusJwtDecoder.withSecretKey(SECRET_KEY).macAlgorithm(MacAlgorithm.HS512).build();

    // 生成 JWT
    public static String generateToken(CustomerUserDetailsImpl customerUserDetails) {
        Instant now = Instant.now();

        JwtClaimsSet claims = JwtClaimsSet.builder()
                .issuedAt(now)
                .expiresAt(now.plus(EXPIRATION_TIME, ChronoUnit.MILLIS))
                .claim("username", customerUserDetails.getUsername())
                .build();

        JwsHeader header = JwsHeader.with(MacAlgorithm.HS512).build();

        Jwt jwt = jwtEncoder.encode(JwtEncoderParameters.from(header, claims));
        return jwt.getTokenValue();
    }
}
```

> [!NOTE] 注意事项
> 1. 授权服务器中，只需要生成 JWT 的方法，这个 JWT 与我们的 OAuth 还无关，只是我们在完成授权之后，把我们自己生产的 JWT 发送给前端，实现登录，不能虽然在授权页中进行了登录和授权，当用户来访问我们的前端时，还是处于未登录的状态，这就不太好了，这个密钥一定要和我们的后端一致，不要我把 jwt 发到前端了， 前端也把 jwt 发到后端了，但是 jwt 不能被后端解析就好玩了

----









































-----


### 1.4. 开发一条龙服务系统

#### 1.4.1. 开发客户端应用

---


#### 1.4.2. 开发授权服务器

----


#### 1.4.3. 开发资源服务器

-----









