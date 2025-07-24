<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark@2x.png" class="logo" width="120"/>

# 授权码模式（Authorization Code）+ PKCE（授权码交换的密钥证明）+ OIDC（身份认证协议）详细流程

以 **Gitee 使用第三方 GitHub 登录**为例，为您详细解析完整的 OAuth2.0 + PKCE + OIDC 授权流程。

## 前置准备：Gitee 在 GitHub 注册 OAuth 应用

在开始授权流程之前，Gitee 需要在 GitHub 开发者平台注册 OAuth 应用[^1]：

**注册步骤**：

1. 登录 GitHub，进入个人设置页面
2. 左侧边栏选择"开发人员设置" → "OAuth Apps"
3. 点击"新建 OAuth 应用"
4. 填写应用信息：
    - **Application name**: Gitee
    - **Homepage URL**: https://gitee.com
    - **Application description**: Gitee第三方登录
    - **Authorization callback URL**: https://gitee.com/auth/github/callback

**注册成功后获得**：

- **Client ID**: `5a179b878a9f6ac42acd` (示例值)
- **Client Secret**: `b4ef1e52bb143e30e149bb426916f911` (示例值，需保密)


## 完整授权流程详解

### <span style="background:#9254de">1. Gitee 前端页面发起请求，并跳转到 GitHub 授权页</span>

前端的行为可能如下所示：

1. xxxxxxxxx
```javascript
// 前端生成 PKCE 参数（如果使用 PKCE）
async function generatePKCEParams() {
    const codeVerifier = generateRandomString(128);
    const encoder = new TextEncoder();
    const data = encoder.encode(codeVerifier);
    const digest = await crypto.subtle.digest('SHA-256', data);
    const codeChallenge = base64URLEncode(digest);
    
    // 存储 code_verifier 用于后续验证
    sessionStorage.setItem('code_verifier', codeVerifier);
    
    return {
        codeVerifier,
        codeChallenge,
        codeChallengeMethod: 'S256'
    };
}

// 发起授权请求
const { codeChallenge } = await generatePKCEParams();
const state = generateRandomString(32); // 防止 CSRF 攻击
sessionStorage.setItem('oauth_state', state);

// 构造授权 URL
const authUrl = `https://github.com/login/oauth/authorize?` +
    `client_id=5a179b878a9f6ac42acd&` +
    `redirect_uri=${encodeURIComponent('https://gitee.com/auth/github/callback')}&` +
    `response_type=code&` +
    `scope=openid user email&` +  // OIDC 需要 openid scope
    `state=${state}&` +
    `code_challenge=${codeChallenge}&` +
    `code_challenge_method=S256`;

window.location.href = authUrl;
```

**参数解释**：

1. **client_id**: Gitee 向 GitHub 开放平台注册应用时获得的 Client ID
2. **redirect_uri**: GitHub 授权后将跳转回 Gitee 的地址（回调地址）
3. **response_type**: 告诉 GitHub 要使用"授权码模式"
4. **scope**: 请求的权限范围，`openid` 是 OIDC 必需的，`user email` 是访问用户基本信息和邮箱
5. **state**: Gitee 随机生成的 CSRF Token，用于防止 CSRF 攻击[^2][^3]
6. **code_challenge**: PKCE 机制的挑战码，防止授权码劫持[^4][^5]
7. **code_challenge_method**: 固定为 "S256"，表示使用 SHA256 哈希

Gitee 的前端页面通过 `window.location.href` 跳转到 GitHub 提供的登录页面。

### <span style="background:#9254de">2. 用户在 GitHub 上进行身份认证和授权</span>

**用户认证流程**：

1. GitHub 检查用户是否已登录
2. 如果未登录，显示 GitHub 登录页面
3. 用户输入 GitHub 用户名和密码完成认证

**用户授权流程**：

1. GitHub 验证 client_id 和 redirect_uri 的合法性
2. 显示授权同意页面，明确列出 Gitee 请求的权限：
    - "是否允许 Gitee 访问你的基本信息"
    - "是否允许 Gitee 访问你的邮箱地址"
    - "是否允许 Gitee 验证你的身份"（OIDC）
3. 用户点击"授权"按钮同意授权

**GitHub 服务器处理**：

- 存储 `code_challenge` 和 `code_challenge_method` 与即将生成的授权码关联
- 生成一次性授权码（Authorization Code）
- 准备重定向响应


### <span style="background:#9254de">3. GitHub 授权后，携带授权码重定向回 Gitee 后端地址</span>

GitHub 在用户授权后，会将页面重定向到 Gitee 预先注册并指定回调的**后端地址**：

```http
HTTP/1.1 302 Found
Location: https://gitee.com/auth/github/callback?
    code=gho_16C7e42F292c6912E7710c838347Ae178B4a&
    state=81fda4a8066fb1ea3310d3bf577ece61a8e0286c03f82c91
```

**Gitee 后端接收处理**：

```javascript
// 后端回调处理
app.get('/auth/github/callback', async (req, res) => {
    const { code, state } = req.query;
    
    // 1. 验证 state 参数防止 CSRF 攻击
    const storedState = req.session.oauth_state;
    if (state !== storedState) {
        return res.status(400).json({ error: 'Invalid state parameter' });
    }
    
    // 2. 获取存储的 code_verifier
    const codeVerifier = req.session.code_verifier;
    
    // 3. 使用授权码和 PKCE 参数交换令牌
    const tokenResponse = await exchangeCodeForTokens(code, codeVerifier);
    
    // 4. 处理返回的令牌...
});
```


### <span style="background:#9254de">4. Gitee 后端使用授权码交换访问令牌</span>

Gitee 后端收到授权码后，向 GitHub 的令牌端点发起 POST 请求：

```javascript
async function exchangeCodeForTokens(authorizationCode, codeVerifier) {
    const tokenResponse = await fetch('https://github.com/login/oauth/access_token', {
        method: 'POST',
        headers: {
            'Accept': 'application/json',
            'Content-Type': 'application/x-www-form-urlencoded',
        },
        body: new URLSearchParams({
            grant_type: 'authorization_code',
            client_id: '5a179b878a9f6ac42acd',
            client_secret: 'b4ef1e52bb143e30e149bb426916f911', // 机密客户端需要
            code: authorizationCode,
            redirect_uri: 'https://gitee.com/auth/github/callback',
            code_verifier: codeVerifier  // PKCE 验证密钥
        })
    });
    
    return await tokenResponse.json();
}
```

**GitHub 服务器验证过程**[^6][^7]：

1. **验证授权码**：检查 code 是否有效且未过期
2. **验证 PKCE**：
    - 对收到的 `code_verifier` 进行 SHA256 哈希并 Base64URL 编码
    - 与之前存储的 `code_challenge` 进行比较
    - 如果匹配，说明请求来自同一客户端
3. **验证客户端凭据**：检查 client_id 和 client_secret
4. **验证回调地址**：确保 redirect_uri 与注册时一致

### <span style="background:#9254de">5. GitHub 返回令牌响应</span>

验证成功后，GitHub 返回包含多种令牌的响应[^8][^9]：

```json
{
    "access_token": "gho_16C7e42F292c6912E7710c838347Ae178B4a",
    "refresh_token": "ghr_1B4a2e77838347a7E420ce178f2E7c6912E7710c83",
    "id_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJodHRwczovL2dpdGh1Yi5jb20iLCJzdWIiOiIxMjM0NTY3OCIsImF1ZCI6IjVhMTc5Yjg3OGE5ZjZhYzQyYWNkIiwiZXhwIjoxNjQ2MDg3MDAwLCJpYXQiOjE2NDYwODMwMDAsIm5hbWUiOiJKb2huIERvZSIsImVtYWlsIjoiam9obi5kb2VAZXhhbXBsZS5jb20ifQ.signature",
    "token_type": "Bearer",
    "expires_in": 3600,
    "scope": "openid user email"
}
```

**令牌类型解释**：

**Access Token**[^10]:

- 用于访问 GitHub API 资源
- 生命周期较短（通常1-2小时）
- 不包含用户身份信息

**ID Token**（OIDC 特有）[^9][^11][^12]：

- JWT 格式的身份令牌
- 包含用户身份验证信息
- 结构化的用户声明（claims）

**Refresh Token**：

- 用于获取新的 access_token
- 生命周期较长（通常数天到数月）
- 可以实现长期会话管理


### <span style="background:#9254de">6. Gitee 验证和解析 ID Token（OIDC 特有）</span>

ID Token 是 OIDC 协议的核心特性，Gitee 需要验证其完整性和真实性[^13]：

```javascript
// 解码 ID Token（JWT 格式）
function decodeIDToken(idToken) {
    const [header, payload, signature] = idToken.split('.');
    
    return {
        header: JSON.parse(base64URLDecode(header)),
        payload: JSON.parse(base64URLDecode(payload)),
        signature: signature
    };
}

// 验证 ID Token
async function validateIDToken(idToken) {
    const decoded = decodeIDToken(idToken);
    
    // 1. 验证签名（使用 GitHub 的公钥）
    const isSignatureValid = await verifyJWTSignature(idToken, githubPublicKey);
    
    // 2. 验证标准声明
    const now = Math.floor(Date.now() / 1000);
    const claims = decoded.payload;
    
    if (claims.exp < now) {
        throw new Error('ID Token 已过期');
    }
    
    if (claims.aud !== '5a179b878a9f6ac42acd') {
        throw new Error('ID Token 受众不匹配');
    }
    
    if (claims.iss !== 'https://github.com') {
        throw new Error('ID Token 颁发者不正确');
    }
    
    // 3. 提取用户信息
    return {
        userId: claims.sub,
        name: claims.name,
        email: claims.email,
        authenticatedAt: claims.auth_time
    };
}
```

**ID Token 载荷示例**[^12]：

```json
{
    "iss": "https://github.com",           // 颁发者
    "sub": "12345678",                     // 用户唯一标识符
    "aud": "5a179b878a9f6ac42acd",        // 受众（Gitee 的 client_id）
    "exp": 1646087000,                     // 过期时间
    "iat": 1646083000,                     // 颁发时间
    "auth_time": 1646082000,               // 认证时间
    "name": "John Doe",                    // 用户姓名
    "email": "john.doe@example.com",       // 用户邮箱
    "email_verified": true                 // 邮箱是否已验证
}
```


### <span style="background:#9254de">7. Gitee 访问用户信息端点（可选）</span>

如果需要更多用户信息，Gitee 可以使用 Access Token 调用 GitHub 的 UserInfo 端点：

```javascript
// 获取用户详细信息
async function getUserInfo(accessToken) {
    const response = await fetch('https://api.github.com/user', {
        headers: {
            'Authorization': `Bearer ${accessToken}`,
            'User-Agent': 'Gitee-OAuth-Client',
            'Accept': 'application/vnd.github.v3+json'
        }
    });
    
    if (!response.ok) {
        throw new Error('Failed to fetch user info');
    }
    
    return await response.json();
}
```

**GitHub UserInfo 响应示例**：

```json
{
    "login": "johndoe",
    "id": 12345678,
    "node_id": "MDQ6VXNlcjEyMzQ1Njc4",
    "avatar_url": "https://avatars.githubusercontent.com/u/12345678?v=4",
    "name": "John Doe",
    "email": "john.doe@example.com",
    "bio": "Software Developer",
    "location": "San Francisco, CA",
    "blog": "https://johndoe.dev",
    "public_repos": 42,
    "followers": 100,
    "following": 50,
    "created_at": "2020-01-01T00:00:00Z",
    "updated_at": "2023-01-01T00:00:00Z"
}
```


### <span style="background:#9254de">8. Gitee 完成用户登录和会话建立</span>

Gitee 完成最终的用户认证和系统集成：

```javascript
// 处理登录完成
async function completeLogin(idTokenClaims, userInfo, accessToken) {
    // 1. 查找或创建本地用户账户
    let localUser = await findUserByGitHubId(idTokenClaims.userId);
    
    if (!localUser) {
        // 创建新用户账户
        localUser = await createUser({
            githubId: idTokenClaims.userId,
            username: userInfo.login,
            email: idTokenClaims.email,
            name: idTokenClaims.name,
            avatar: userInfo.avatar_url
        });
    } else {
        // 更新用户信息
        await updateUser(localUser.id, {
            lastLogin: new Date(),
            avatar: userInfo.avatar_url
        });
    }
    
    // 2. 建立用户会话
    const sessionToken = generateSessionToken();
    await createUserSession({
        userId: localUser.id,
        sessionToken: sessionToken,
        accessToken: accessToken,  // 存储以供后续 API 调用
        refreshToken: refreshToken,
        expiresAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000) // 30天
    });
    
    // 3. 设置认证 Cookie
    res.cookie('gitee_session', sessionToken, {
        httpOnly: true,
        secure: true,
        sameSite: 'Strict',
        maxAge: 30 * 24 * 60 * 60 * 1000
    });
    
    // 4. 重定向到应用首页
    res.redirect('https://gitee.com/dashboard');
}
```


## 各组件的作用和区别详解

### **PKCE 的作用和与普通 OAuth2 的区别**

**PKCE 的安全作用**[^14][^15][^7]：

1. **防止授权码劫持**：
    - **攻击场景**：恶意应用在同一设备上注册相同的 redirect URI，截获授权码
    - **PKCE 防护**：即使授权码被截获，没有 code_verifier 也无法换取令牌
    - **验证机制**：授权服务器验证 SHA256(code_verifier) === code_challenge
2. **动态密钥交换**：
    - 每次授权请求生成唯一的 code_verifier（43-128位随机字符串）
    - code_challenge = Base64URL(SHA256(code_verifier))
    - 避免了静态 client_secret 在公共客户端的泄露风险

**与普通 OAuth2 的对比**：


| 特性 | 普通 OAuth2 | OAuth2 + PKCE |
| :-- | :-- | :-- |
| 客户端类型 | 主要适用于机密客户端 | 专为公共客户端设计，机密客户端也推荐使用 |
| 安全密钥 | 静态 client_secret | 动态 code_verifier + code_challenge |
| 授权码保护 | 依赖 client_secret 验证 | 双重保护：client_secret + PKCE |
| 移动应用安全 | 存在授权码劫持风险 | 有效防止授权码劫持 |
| 实现复杂度 | 相对简单 | 需要额外的 PKCE 参数生成和验证 |

### **OIDC 的作用和与普通 OAuth2 的区别**

**OIDC 的身份认证作用**[^8][^16][^9]：

1. **标准化身份验证**：
    - **ID Token**：JWT 格式的身份令牌，包含用户身份信息
    - **UserInfo 端点**：标准化的用户信息获取接口
    - **标准化声明**：sub, name, email, picture 等标准化用户属性
2. **增强安全特性**：
    - **nonce 参数**：防止重放攻击
    - **auth_time 声明**：记录认证时间，支持会话管理
    - **数字签名验证**：ID Token 可验证完整性和真实性
3. **单点登录（SSO）支持**：
    - ID Token 可在多个应用间传递身份信息
    - 用户只需认证一次即可访问多个集成应用

**与普通 OAuth2 的对比**：


| 特性 | 普通 OAuth2 | OAuth2 + OIDC |
| :-- | :-- | :-- |
| 主要目的 | 授权（Authorization） | 认证（Authentication）+ 授权 |
| 令牌类型 | Access Token, Refresh Token | + ID Token（JWT格式） |
| 用户信息 | 需要单独 API 调用获取 | ID Token 直接包含基本身份信息 |
| 标准化程度 | API 端点和数据格式各厂商不同 | 统一的 UserInfo 端点和标准声明 |
| 身份验证证明 | 无直接身份验证证明 | ID Token 提供身份验证证明 |
| 安全增强 | 基础的 state 参数 | + nonce, auth_time 等安全参数 |

## 总结

通过 **授权码模式 + PKCE + OIDC** 的组合，我们实现了：

1. **OAuth2 授权码模式**：提供了基础的安全授权框架
2. **PKCE 增强**：防止了公共客户端的授权码劫持攻击[^4][^5]
3. **OIDC 身份层**：标准化了身份验证和用户信息获取[^9][^17]

这种组合方案特别适合现代 Web 应用、单页应用（SPA）和移动应用，提供了业界最佳的安全实践。相比传统的自定义登录实现，这套标准化方案具有更高的安全性、更好的互操作性和更完善的生态支持。

<div style="text-align: center">⁂</div>

[^1]: https://docs.github.com/zh/apps/oauth-apps/building-oauth-apps/creating-an-oauth-app

[^2]: https://www.cnblogs.com/blowing00/p/14872312.html

[^3]: https://hypc.github.io/2021/12/08/oauth2-csrf/

[^4]: https://blog.csdn.net/weixin_42218884/article/details/142645702

[^5]: https://blog.csdn.net/weixin_45946850/article/details/130540948

[^6]: https://experienceleague.adobe.com/zh-hans/docs/workfront/using/adobe-workfront-api/api-notes/oauth-app-pkce-flow

[^7]: https://blog.csdn.net/catoop/article/details/114080598

[^8]: https://www.fortinet.com/cn/resources/cyberglossary/oidc

[^9]: https://blog.csdn.net/onePlus5T/article/details/141431879

[^10]: https://www.apiseven.com/blog/openid-vs-oauth

[^11]: https://auth0.com/docs/secure/tokens/id-tokens/id-token-structure

[^12]: https://jfrog.com/help/r/jfrog-platform-administration-documentation/understanding-the-oidc-token?contentId=0ZYaa0URMhmt35UvU4fmsg

[^13]: https://curity.io/resources/learn/validating-an-id-token/

[^14]: https://quant67.com/post/wauth/pkce.html

[^15]: https://juejin.cn/post/7307855762000707622

[^16]: https://www.lumin.tech/articles/openid-connect/

[^17]: https://authing.co/blog/558

[^18]: https://www.authing.cn/blog/576

[^19]: https://www.cnblogs.com/atwood-pan/p/17787904.html

[^20]: http://www.javaboy.org/2020/0424/github-oauth2.html

[^21]: https://blog.csdn.net/onePlus5T/article/details/141432732

[^22]: https://learn.microsoft.com/zh-cn/entra/identity-platform/v2-protocols-oidc

[^23]: https://apifox.com/apiskills/how-to-use-github-oauth2/

[^24]: https://www.cnblogs.com/myshowtime/p/15555538.html

[^25]: https://www.microsoft.com/zh-cn/security/business/security-101/what-is-openid-connect-oidc

[^26]: https://www.ruanyifeng.com/blog/2019/04/github-oauth.html

[^27]: https://tonyxu.io/blog/oauth2-pkce-flow-zh/

[^28]: https://www.authing.cn/blog/413

[^29]: https://blog.csdn.net/qq_15015743/article/details/123973898

[^30]: https://authing.csdn.net/644b609c3399b617f0c020dc.html

[^31]: https://tonybai.com/2023/12/22/understand-oidc-by-example/

[^32]: https://juejin.cn/post/7128318034166415397

[^33]: https://learn.microsoft.com/zh-tw/entra/identity-platform/v2-oauth2-auth-code-flow

[^34]: https://www.clougence.com/cc-doc/operation/system_manage/sso/sso_oidc

[^35]: https://www.nowcoder.com/discuss/353150087394172928

[^36]: https://blog.logto.io/zh-TW/oauth-2-1

[^37]: https://stackoverflow.com/questions/77081384/what-is-the-different-between-normal-authorization-code-grant-flow-vs-pkce

[^38]: https://docs.authing.cn/v2/concepts/oidc/oidc-overview.html

[^39]: https://cloud.tencent.com/developer/ask/sof/116202660

[^40]: https://developer.genesys.cloud/authorization/platform-auth/use-pkce

[^41]: https://www.cnblogs.com/Zhang-Xiang/p/14733907.html

[^42]: https://blog.csdn.net/kidkid1208/article/details/123335719

[^43]: https://christianlydemann.com/implicit-flow-vs-code-flow-with-pkce/

[^44]: https://juejin.cn/post/7318436351609733130

[^45]: https://juejin.cn/post/7310411848263073818

[^46]: https://blog.csdn.net/m0_46198325/article/details/125385632

[^47]: https://stackoverflow.com/questions/39339942/passing-client-secret-to-users-in-github-api

[^48]: https://www.cnblogs.com/holyading/p/18420448

[^49]: https://docs.redhat.com/zh-cn/documentation/red_hat_quay/3.8/html/use_red_hat_quay/github-app

[^50]: https://github.com/hwi/HWIOAuthBundle/issues/1584

[^51]: https://juejin.cn/post/7337481770408558601

[^52]: https://blog.csdn.net/hong10086/article/details/88532936

[^53]: https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/authenticating-to-the-rest-api-with-an-oauth-app

[^54]: https://www.skjava.com/article/3381231971

[^55]: https://docs.redhat.com/zh-cn/documentation/red_hat_quay/3.11/html/use_red_hat_quay/github-app

[^56]: https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/authorizing-oauth-apps

[^57]: https://www.cnblogs.com/lishuaiqi/p/15606687.html

[^58]: https://docs.github.com/zh/apps/oauth-apps/building-oauth-apps/authorizing-oauth-apps

[^59]: https://github.com/simplabs/ember-simple-auth/issues/226

[^60]: https://cloud.tencent.cn/developer/article/2185002

[^61]: https://docs.github.com/zh/apps/oauth-apps/building-oauth-apps/best-practices-for-creating-an-oauth-app

[^62]: https://support.heateor.com/get-github-client-id-client-secret/

[^63]: https://blog.csdn.net/qq_33240556/article/details/140529234

[^64]: https://docs.github.com/zh/apps/creating-github-apps/registering-a-github-app/registering-a-github-app

[^65]: https://blog.csdn.net/jiangyang100/article/details/128818827

[^66]: https://cloud.tencent.com.cn/developer/information/使用pkce的安全asp.net核心3.1授权码流

[^67]: https://docs.gitlab.com/ci/secrets/id_token_authentication/

[^68]: https://juejin.cn/post/7308782430591369266

[^69]: https://www.cnblogs.com/saaspeter/p/17171161.html

[^70]: https://maodanp.github.io/2022/03/09/PKCE/

[^71]: https://oa.dnc.global/-Json-Web-Token-JWT-40-.html

[^72]: https://www.cdxy.me/?p=695

[^73]: https://time.geekbang.org/comment/nice/260670

[^74]: https://connect2id.com/learn/openid-connect

[^75]: https://blog.csdn.net/littlelittlebai/article/details/110948676

[^76]: https://time.geekbang.org/column/article/260670

