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









