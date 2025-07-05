


---


### 2. 业务处理

#### 2.1. 集成 JWT

##### 前言

我们这里只把用户名封进 Token，每次操作再去查一次数据库——反正有 Redis 托底，不虚。

----


##### 1.1. 创建 Spring Web 项目，添加 Security 相关依赖

1. ==Web==：
	1. Spring Web
2. ==Security==：
	1. Spring Security
3. ==SQL==
	1. JDBC API
	2. MyBatis Framework
	3. MySQL Driver

---


##### 引入 JWT 相关依赖



---



---


##### 创建 JWT 生成和解析工具类

```
@Component
public class JwtUtil {
    private static final String SECRET_KEY = "aVeryLongAndSecureSecretKeyThatIsAtLeast64BytesLong";  // 密钥，长度至少为64字节，建议 base64 编码
    private static final long EXPIRATION_TIME = 1000 * 60 * 60 * 24; // Token过期时间：1天
    private static final String TOKEN_PREFIX = "Bearer "; // JWT在HTTP请求中的标准前缀
    private static final String HEADER_STRING = "Authorization"; // HTTP请求头中存放JWT的字段名

    // 仅使用用户名生成Token
    public static String generateToken(UserDetails userDetails) {
        return Jwts.builder()
                .setSubject(userDetails.getUsername())  // 将用户名作为Token的主题
				.claim("name", "张三") // 自定义字段1，随便写的
	            .claim("age", "18") // 自定义字段2，随便写的
                .setIssuedAt(new Date())  // 设置签发时间
                .setExpiration(new Date(System.currentTimeMillis() + EXPIRATION_TIME))  // 设置过期时间
                .signWith(SignatureAlgorithm.HS512, SECRET_KEY)  // 使用HS512算法和密钥签名
                .compact();
    }

    // 从Token中提取用户名
    public static String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }

    // 通用方法：从Token中提取指定的声明
    private static <T> T extractClaim(String token, ClaimsResolver<T> claimsResolver) {
        final Claims claims = extractAllClaims(token);
        return claimsResolver.resolve(claims);
    }

    // 解析Token并提取所有声明（Claims）
    private static Claims extractAllClaims(String token) {
        try {
            return Jwts.parser()
                    .setSigningKey(SECRET_KEY)
                    .build()
                    .parseClaimsJws(token)
                    .getBody();
        } catch (JwtException e) {
            throw new RuntimeException("无效或被篡改的Token", e);
        }
    }

    // 判断Token是否已过期
    private static boolean isTokenExpired(String token) {
        return extractExpiration(token).before(new Date());
    }

    // 提取Token的过期时间
    private static Date extractExpiration(String token) {
        return extractClaim(token, Claims::getExpiration);
    }

    // 校验Token是否有效：用户名匹配且未过期
    public static boolean validateToken(String token, UserDetails userDetails) {
        final String username = extractUsername(token);
        return (username.equals(userDetails.getUsername()) && !isTokenExpired(token));
    }

    // 检查Token是否被篡改
    public static boolean isTokenTampered(String token) {
        try {
            extractAllClaims(token); // 尝试解析，失败则表示被篡改
            return false;
        } catch (JwtException e) {
            return true;  // 解析失败，说明Token被篡改
        }
    }

    // 从Authorization头中提取Bearer Token
    public static String extractBearerToken(String authorizationHeader) {
        if (authorizationHeader != null && authorizationHeader.startsWith(TOKEN_PREFIX)) {
            return authorizationHeader.substring(7);  // 去除"Bearer "前缀
        }
        return null;
    }

    // 如果Token已过期则刷新，否则返回原Token
    public static String refreshToken(String token, UserDetails userDetails) {
        if (isTokenExpired(token)) {
            return generateToken(userDetails);
        }
        return token;
    }

    // 函数式接口：用于提取声明
    @FunctionalInterface
    public interface ClaimsResolver<T> {
        T resolve(Claims claims);
    }
}
```

> [!NOTE] 注意事项
> 1. 工具类如果全是 static 方法，没必要声明为组件；但如果方法大多是私有的，声明为组件再注入使用，好像也没什么额外麻烦。

---


##### 创建 JwtRequestFilter 过滤器

每次请求进来时，先从请求头的 Authorization 字段提取 JWT Token，检查这个 Token 是否存在且有效（包括签名和过期时间），然后从 Token 中解析出用户名。

接着通过用户名调用 `CustomerServiceImpl` 加载完整的 `CustomerServiceImpl` ，再次验证 Token 是否与用户信息匹配且未过期。如果验证通过，就用这些用户信息创建一个认证对象（Authentication），并把它放到当前线程的安全上下文里。这样，后续的业务逻辑和安全机制就能通过这个上下文知道当前请求的用户是谁，实现基于 JWT 的无状态认证。最后再继续执行剩下的过滤器链，保证请求正常流转。
```
@Component  
public class JwtRequestFilter extends OncePerRequestFilter {  
  
    private final JwtUtil jwtUtil;  
    private final UserDetailsService userDetailsService;  
  
    // 构造器注入  
    public JwtRequestFilter(JwtUtil jwtUtil, UserDetailsService userDetailsService) {  
        this.jwtUtil = jwtUtil;  
        this.userDetailsService = userDetailsService;  
    }  
  
    @Override  
    protected void doFilterInternal(HttpServletRequest request,  
                                    HttpServletResponse response,  
                                    FilterChain filterChain)  
            throws ServletException, IOException {  
  
        String token = resolveToken(request);  
  
        if (token != null) {  
            try {  
                String username = jwtUtil.extractUsername(token);  
                if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {  
                    UserDetails userDetails = userDetailsService.loadUserByUsername(username);  
  
                    if (jwtUtil.validateToken(token, userDetails)) {  
                        UsernamePasswordAuthenticationToken auth =  
                                new UsernamePasswordAuthenticationToken(  
                                        userDetails, null, userDetails.getAuthorities()  
                                );  
                        auth.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));  
                        SecurityContextHolder.getContext().setAuthentication(auth);  
                    }  
                }  
            } catch (Exception e) {  
                // Token 无效或过期时清除身份信息  
                SecurityContextHolder.clearContext();  
            }  
        }  
  
        filterChain.doFilter(request, response);  
    }  
  
    private String resolveToken(HttpServletRequest request) {  
        String header = request.getHeader("Authorization");  
        if (StringUtils.hasText(header) && header.startsWith("Bearer ")) {  
            return header.substring(7);  
        }  
        return null;  
    }  
}
```

---


#### 添加 JwtRequestFilter 过滤器

```
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    @Autowired
	private JwtRequestFilter jwtRequestFilter;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http.addFilterBefore(jwtRequestFilter, UsernamePasswordAuthenticationFilter.class); 
        
    }
}
```

---


#### 实现 登录 API、测试 API、注销 API

```
@RestController
@RequestMapping("/auth")
public class AuthController {

    private final AuthenticationManager authenticationManager;
    private final JwtUtil jwtUtil;

    // 构造器注入
    public AuthController(AuthenticationManager authenticationManager, JwtUtil jwtUtil) {
        this.authenticationManager = authenticationManager;
        this.jwtUtil = jwtUtil;
    }

    @PostMapping("/login")
    public ResponseEntity<?> login(
            @RequestParam String username,
            @RequestParam String password) {

        try {
            // 1. 使用 AuthenticationManager 实现认证
            UsernamePasswordAuthenticationToken authToken =
                    new UsernamePasswordAuthenticationToken(username, password);
            Authentication authentication = authenticationManager.authenticate(authToken);

            // 2. 从 Authentication 中获取 UserDetails
            UserDetails userDetails = (UserDetails) authentication.getPrincipal();

            // 3. 生成 JWT
            String jwt = jwtUtil.generateToken(userDetails);

            // 4. 构建成功响应
            Map<String, Object> result = new HashMap<>();
            result.put("status", "登录成功");
            result.put("JWT", jwt);
            return ResponseEntity.ok(result);

        } catch (AuthenticationException ex) {
            // 5. 抛出异常，交由全局异常处理器处理
            throw ex;
        }
    }

    @GetMapping("/test")
    public ResponseEntity<?> test() {
        // 测试 API
        return ResponseEntity.ok("测试成功");
    }

    @PostMapping("/logout")
    public ResponseEntity<?> logout() {
        // 注销 API
        // 由于 JWT 是无状态的，注销通常在客户端完成，也就是上前端的人来完成这个。
        return ResponseEntity.ok("注销成功");
    }
}
```
![](image-20250520105139572.png)

> [!NOTE] 注意事项
> 1. 如果登录不上去，看看是不是因为密码加密器
> 2. 你会发现，JWT 真是有点“吓人”——不管你重启服务器还是干啥，这个 JWT 都有效。只要密钥不换，它就能一直用下去，简直“一辈子有效”。所以，我们得给 JWT 设置个短一点的有效期，再通过 HTTPS 安全传输，才能放心用。

---


#### JWT 执行流程

==1.用户登录==
用户把用户名和密码发给服务器。


==2.生成 Token==
服务器验证通过后，创建一个 JWT，里面通常会写入用户的 ID、权限角色、过期时间等信息，然后把这个 Token 发回给客户端。


==3.客户端保存 Token==
客户端收到 Token 后，自己保存下来（比如放在浏览器的 localStorage 或 Cookie 里）。


==4.后续请求带上 Token==
每次客户端再发请求时，把这个 Token 加在 HTTP 的请求头里（一般是 `Authorization: Bearer <token>` 格式）


==5.服务器接收到 Token==
解码 Header 和 Payload，提取其中的用户信息；验证签名：使用服务器的密钥检查 Token 是否被篡改；接着确认 Token 是否在有效期内。  

如果所有验证都通过，服务器便可基于 Payload 中的数据处理请求；若验证失败，则返回错误信息，通常会要求用户重新登录。

我们这里只把用户名封进 Token，每次操作再去查一次数据库——反正有 Redis 托底，不虚。

---












