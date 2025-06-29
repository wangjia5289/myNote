# 一、理论



### 5. 常见 “记住我” 方式

“‘记住我’功能是指在用户身份验证成功后，如何保持其登录状态。由于每次请求都要求重新登录会严重影响用户体验，因此在用户首次登录成功后，系统会返回一个凭证。用户在随后的请求中携带该凭证，系统通过验证凭证来识别用户身份，从而避免频繁的重新登录，确保用户能够持续保持登录状态。

常见的 “记住我” 方式有：
1. ==JSESSIONID Cookie + HttpSession（默认）==：
	1. Spring Security 默认的就是这个
	2. 将 `SecurityContext` （包含最重要的 `Authentication`）保存在 `HttpSession` 中，下次请求通过 `JSESSIONID` 找到 `HttpSession` 再找到 `Authentication`
	3. 依赖于：`SecurityContextPersistenceFilter` 过滤器
2. ==JWT Cookie（主流）==：
	1. 当前主流的方式

---






![](image-20250618183214684.png)
![](image-20250618183157822.png)

![](image-20250618183219357.png)



# 二、实操

### 1. 基本使用




#### 1.3. 使用 MyBatis 实现查询逻辑

##### 1.3.1. 编写 Model Entity Pojo 类

```
public class UserWithAuthoritie {

	// users 表中的数据
    private Integer userId;
    private String username;
    private String password;
    private Boolean isAccountNonExpired;
    private Boolean isAccountNonLocked;
    private Boolean isCredentialsNonExpired;
    private Boolean isEnabled;
    private String email;
    private String phoneNumber;
    
	// authorities 表中的数据
    private List<String> authorities;

    // getter/setter ...
}
```

---


##### 1.3.2. 编写 Mapper 接口

```
@Mapper
public interface UserWithAuthoritieMapper {

	// 查询用户
    UserWithAuthoritie findUserWithAuthoritieByUserName(String username);
    
}
```

---


##### 1.3.3. 编写 SQL 语句

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="com.example.security.mapper.UserWithAuthoritieMapper">

    <resultMap id="UserWithAuthoritieResultMap" type="com.example.security.model.entity.UserWithAuthoritie">
        <id     property="userId"                   column="user_id"                   jdbcType="INTEGER"/>
        <result property="username"                 column="username"                  jdbcType="VARCHAR"/>
        <result property="password"                 column="password"                  jdbcType="VARCHAR"/>
        <result property="isAccountNonExpired"      column="isAccountNonExpired"       jdbcType="TINYINT"/>
        <result property="isAccountNonLocked"       column="isAccountNonLocked"        jdbcType="TINYINT"/>
        <result property="isCredentialsNonExpired"  column="isCredentialsNonExpired"   jdbcType="TINYINT"/>
        <result property="isEnabled"                column="isEnabled"                 jdbcType="TINYINT"/>
        <result property="email"                    column="email"                     jdbcType="VARCHAR"/>
        <result property="phoneNumber"              column="phoneNumber"               jdbcType="VARCHAR"/>
        <collection property="authorities" ofType="java.lang.String">
	        <result column="authoritie_name" jdbcType="VARCHAR"/>
	    </collection>
    </resultMap>

    <!-- 查询 -->
    <select id="findUserWithAuthoritieByUserName"
            parameterType="string"
            resultMap="UserWithAuthoritieResultMap">
        SELECT
            u.user_id,
            u.username,
            u.password,
            u.isAccountNonExpired,
            u.isAccountNonLocked,
            u.isCredentialsNonExpired,
            u.isEnabled,
            u.email,
            u.phoneNumber,
            ua.authoritie_name
        FROM users u
                 LEFT JOIN user_authorities ua ON u.username = ua.username
        WHERE u.username = #{username}
    </select>
</mapper>
```

---


#### 1.4. 实现 UserDetails 接口

`CustomerDetailsImpl` 不属于三层架构，严格来说，他应该放到 `model.entity`
```
public class CustomerDetailsImpl implements UserDetails {

    private final String username;
    private final String password;
    private final Boolean isAccountNonExpired;
    private final Boolean isAccountNonLocked;
    private final Boolean isCredentialsNonExpired;
    private final Boolean isEnabled;
    private final List<GrantedAuthority> authorities;

    // 扩展的字段
    private final String email;
    private final String phoneNumber;
    private final Integer userId; // 如果用户名不是 ID，我们一般需要它这个ID，就加上

    public CustomerDetailsImpl(UserWithAuthoritie user) {
        this.username = user.getUsername();
        this.password = user.getPassword();
        this.isAccountNonExpired = user.getAccountNonExpired();
        this.isAccountNonLocked = user.getAccountNonLocked();
        this.isCredentialsNonExpired = user.getCredentialsNonExpired();
        this.isEnabled = user.getEnabled();
        this.email = user.getEmail();
        this.phoneNumber = user.getPhoneNumber();
        this.userId = user.getUserId();
        List<GrantedAuthority> grantedAuthorities = new ArrayList<>();
        if (user.getAuthorities() != null) {
            for (String authority : user.getAuthorities()) {
                grantedAuthorities.add(new SimpleGrantedAuthority(authority));
            }
        }
        this.authorities = grantedAuthorities;

    }

    // 扩展的方法
    public String getEmail() {
        return email;
    }

    public String getPhoneNumber() {
        return phoneNumber;
    }
    
	public Integer getUserId() {  
	    return userId;  
	}

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return this.authorities;
    }

    @Override
    public String getPassword() {
        return this.password;
    }

    @Override
    public String getUsername() {
        return this.username;
    }

    @Override
    public boolean isAccountNonExpired() {
        return this.isAccountNonExpired;
    }

    @Override
    public boolean isAccountNonLocked() {
        return this.isAccountNonLocked;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return this.isCredentialsNonExpired;
    }

    @Override
    public boolean isEnabled() {
        return this.isEnabled;
    }
}
```

---


#### 1.5. 实现 UserDetailsService 接口

```
@Service // 不要忘记
public class CustomerDetailsServiceImpl implements UserDetailsService {

    @Autowired
    private UserWithAuthoritieMapper userWithAuthoritieMapper;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        UserWithAuthoritie user = userWithAuthoritieMapper.findUserWithAuthoritieByUserName(username);
        if (user == null) {
            throw new UsernameNotFoundException("用户未找到: " + username);
        }
        return new CustomerDetailsImpl(user);
    }
}
```

---


#### 1.6. 进行 Security 相关配置

```
@Configuration
@EnableMethodSecurity // 1. 启用方法级别的访问控制
@EnableWebSecurity // 2. 启用 Spring Security 安全功能
public class SecurityConfig {
    private final CustomerDetailsServiceImpl customerdetails;

    public SecurityConfig(CustomerDetailsServiceImpl customerdetails) {
        this.customerdetails = customerdetails;
    }

    // 3. 配置 AuthenticationManager
    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration configuration) throws Exception {
        return configuration.getAuthenticationManager(); // 返回默认的 AuthenticationManager
    }

    // 4. 配置 密码加密器，用于 AuthenticationManager 加密 password 后与 CustomerDetailsImpl 进行对比
    @Bean
    public PasswordEncoder passwordEncoder() {
        return NoOpPasswordEncoder.getInstance();
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            // 5. 禁用默认表单登录（也就禁用了 UsernamePasswordAuthenticationFilter 过滤器）
            .formLogin(form -> form.disable())
            // 6. 禁用默认注销功能
            .logout(logout -> logout.disable())
            // 7. 资源级别的访问控制
            .authorizeHttpRequests(auth -> auth
                    .requestMatchers("/auth/login", "auth/logout").permitAll()
                    .requestMatchers("/auth/test").hasAnyAuthority("ROLE_CEO")
                    .anyRequest().authenticated()
            )
            // 8. 用户 未认证、权限不足 的处理
            .exceptionHandling(handler -> handler
                    // 未认证时的响应
                    .authenticationEntryPoint((request, response, authException) -> {
                                response.setContentType("application/json;charset=UTF-8");
                                response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                                response.getWriter().write("{\"error\":\"未认证，请先登录\"}");
                            }
                    )
                    // 权限不足时的响应
                    .accessDeniedHandler((request, response, accessDeniedException) -> {
                                response.setContentType("application/json;charset=UTF-8");
                                response.setStatus(HttpServletResponse.SC_FORBIDDEN);
                                response.getWriter().write("{\"error\":\"权限不足，无法访问此资源\"}");
                            }
                    )
            )
            // 9. HttpSession 相关配置
            .sessionManagement(session -> session
                    .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
            )
            // 10. 默认 SecurityContext 查找和保存策略
            .securityContext(context -> context
                    .requireExplicitSave(false)
            )
            // 11. 配置 UserDetailsServiceImpl，用于 AuthenticationManager 从数据库中取 UserDetailsImpl
            .userDetailsService(customerdetails) // 显式告诉用你的UserDetailsService
            // 12. CSRF 攻击防护
            .csrf(csrf -> csrf.disable());

        return http.build();
    }
}
```

---


#### 1.7. 创建 秘钥生成、加密、解密 工具类

这个密钥生成工具类用于创建 AES 对称加密所需的密钥、加密、解密，我们可以在保存数据时，可以对如手机号、邮箱等敏感信息进行加密，并在需要时解密还原。  

我们常用的 `BCryptPasswordEncoder` 属于单向加密工具，无法还原原文，只能用于对加密结果进行比对验证。
```
@Component
public class EncryptionUtils {

    // 生成 Base64 编码的 AES 密钥（需保存）
    public static String generateSecretKey() throws NoSuchAlgorithmException {
        KeyGenerator keyGenerator = KeyGenerator.getInstance("AES");
        keyGenerator.init(256); // 支持 128、192、256 位
        SecretKey secretKey = keyGenerator.generateKey();
        return Base64.getEncoder().encodeToString(secretKey.getEncoded());
    }

    // 加密（是用密钥加密）
    public static String encrypt(String plainText, String base64Key) throws Exception {
        byte[] keyBytes = Base64.getDecoder().decode(base64Key);
        SecretKeySpec secretKeySpec = new SecretKeySpec(keyBytes, "AES");

        Cipher cipher = Cipher.getInstance("AES"); 
        cipher.init(Cipher.ENCRYPT_MODE, secretKeySpec);
        byte[] encrypted = cipher.doFinal(plainText.getBytes());

        return Base64.getEncoder().encodeToString(encrypted);
    }

    // 解密（使用密钥解密）
    public static String decrypt(String encryptedText, String base64Key) throws Exception {
        byte[] keyBytes = Base64.getDecoder().decode(base64Key);
        SecretKeySpec secretKeySpec = new SecretKeySpec(keyBytes, "AES");

        Cipher cipher = Cipher.getInstance("AES");
        cipher.init(Cipher.DECRYPT_MODE, secretKeySpec);
        byte[] decrypted = cipher.doFinal(Base64.getDecoder().decode(encryptedText));

        return new String(decrypted);
    }
}
```

---


#### 1.8. 实现 登录 API、测试 API、注销 API

```
@RestController
@RequestMapping("/auth")
public class AuthController {

    private final AuthenticationManager authenticationManager;

    public AuthController(AuthenticationManager authenticationManager) {
        this.authenticationManager = authenticationManager;
    }

    @PostMapping("/login")
    public String login(String username, String password) {
        try {
            // 将 username 和 password 封装成 UsernamePasswordAuthenticationToken
            UsernamePasswordAuthenticationToken token =
                    new UsernamePasswordAuthenticationToken(username, password);

            // 传递给 AuthenticationManager 进行认证
            Authentication auth = authenticationManager.authenticate(token);

            // 将 Authentication 保存到当前线程的 SecurityContext
            SecurityContextHolder.getContext().setAuthentication(auth);

            return "登录成功，欢迎用户：" + auth.getName();
        } catch (AuthenticationException e) {
            return "登录失败，用户名或密码错误";
        }
    }

    @GetMapping("/test")
    public String test() {
        return "您已实现从 HttpSession 中获取 Authentication";
    }

    @PostMapping("/logout")
    public ResponseEntity<String> logout(HttpServletRequest request) {
        // 获取当前 session，不创建新 session
        HttpSession session = request.getSession(false);
        if (session != null) {
            // 使 session 失效，完成注销
            session.invalidate();
        }
        return ResponseEntity.ok("用户已成功注销");
    }
}
```

> [!NOTE] 注意事项
> 1. 配置了密码加密器后，AuthenticationManager 会将用户提交的密码加密并与数据库中查询出的密码进行匹配。如果数据库中仍是明文密码，将无法通过校验，返回“登录失败，用户名或密码错误”。
> 2. 因此，我们需要确保数据库中保存的也是经过加密处理的密码。

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

引入 [jjwt-api 依赖](https://mvnrepository.com/artifact/io.jsonwebtoken/jjwt-api/0.12.6)
``` 
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.6</version>
</dependency>
```

---


##### 1.2. 创建用户信息存储表，存储用户信息

我们一般会创建三个表：`users` 表存储用户信息，`authorities` 表存储用户权限信息，`user_authoritie` 表存储用户和权限关联，实现一个用户多个权限，一个权限多个用户。

==1.users 表==

| 列名                          | 数据类型        | 说明                  | 约束        | 默认值 |
| --------------------------- | ----------- | ------------------- | --------- | --- |
| **user_id**                 | int         | ID（也可直接拿来做账号）       | 主键约束、自增属性 |     |
| **username**                | varchar(20) | 用户名（账号，必写）          | 唯一约束      |     |
| **password**                | varchar(20) | 加密密码（必写）            |           |     |
| **isAccountNonExpired**     | tinyint(1)  | 账户是否没过期（必写，1 代表没过期） |           | 1   |
| **isAccountNonLocked**      | tinyint(1)  | 账户是否没锁定（必写）         |           | 1   |
| **isCredentialsNonExpired** | tinyint(1)  | 密码是否没过期（必写）         |           | 1   |
| **isEnabled**               | tinyint(1)  | 用户是否启用（必写）          |           | 1   |
| **email**                   | VARCHAR(20) | 邮箱（示例）              | 唯一约束      |     |
| **phoneNumber**             | VARCHAR(20) | 电话号码（示例）            | 唯一约束      |     |

```
# 1. 创建表
CREATE TABLE IF NOT EXISTS users (
    user_id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(20),
    password VARCHAR(20),
    isAccountNonExpired TINYINT(1) DEFAULT 1,
    isAccountNonLocked TINYINT(1) DEFAULT 1,
    isCredentialsNonExpired TINYINT(1) DEFAULT 1,
    isEnabled TINYINT(1) DEFAULT 1,
    email VARCHAR(20),
    phoneNumber VARCHAR(20)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

ALTER TABLE users ADD CONSTRAINT unique_user UNIQUE (username, email, phoneNumber);


# 2. 插入数据
INSERT INTO users (username, password, email, phoneNumber)
VALUES
    ('alice', 'pass123', 'alice@example.com', '13800000000'),
    ('bob', 'pass456', 'bob@example.com', '13900000001'),
    ('BaTian', 'pass789', 'batian@example.com', '13700000002');
```
![](image-20250519165406858.png)


==2.authorities==

| 列名                  | 数据类型        | 说明    | 约束        |
| ------------------- | ----------- | ----- | --------- |
| **authoritie_id**   | int         | ID    | 主键约束、自增属性 |
| **authoritie_name** | varchar(20) | 权限的名称 | 唯一约束      |

```
# 1. 创建表
CREATE TABLE IF NOT EXISTS authorities (
    authoritie_id INT AUTO_INCREMENT PRIMARY KEY,
    authoritie_name VARCHAR(20)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

ALTER TABLE authorities ADD CONSTRAINT authoiritie UNIQUE (authoritie_name);


# 2. 插入数据
INSERT INTO authorities (authoritie_name)
VALUES 
    ('ROLE_USER'),
    ('ROLE_ADMIN'),
    ('ROLE_CEO');
```
![](image-20250519165427042.png)


==3.user_authoritie 表==

| 列名              | 数据类型        | 约束                         |
| --------------- | ----------- | -------------------------- |
| username        | varchar(20) | 与 `authoritie_name` 设置联合主键 |
| authoritie_name | varchar(20) | 与 `username` 设置联合主键        |

```
# 1. 创建表
CREATE TABLE user_authorities (
    username VARCHAR(20) NOT NULL,
    authoritie_name VARCHAR(20) NOT NULL,
    PRIMARY KEY (username, authoritie_name),
    FOREIGN KEY (username) REFERENCES users (username),
    FOREIGN KEY (authoritie_name) REFERENCES authorities (authoritie_name)
);


# 2. 插入数据
INSERT INTO user_authorities (username, authoritie_name)
VALUES
    ('alice', 'ROLE_USER'),
    ('alice', 'ROLE_ADMIN'),
    ('bob', 'ROLE_USER'),
    ('BaTian', 'ROLE_USER'),
    ('BaTian', 'ROLE_ADMIN'),
    ('BaTian', 'ROLE_CEO');
```
![](image-20250519165451890.png)

> [!NOTE] 注意事项
> 1. 我们是使用 `hasAuthority()` 进行资源级别的访问控制和方法级别的访问控制
> 2. 如果是使用 `hasRole()` ，我们会去掉 `ROLE_` 前缀

---


##### 1.3. 使用 MyBatis 实现查询逻辑

###### 1.3.1. 编写 Model Entity Pojo 类

```
public class UserWithAuthoritie {

	// users 表中的数据
    private Integer userId;
    private String username;
    private String password;
    private Boolean isAccountNonExpired;
    private Boolean isAccountNonLocked;
    private Boolean isCredentialsNonExpired;
    private Boolean isEnabled;
    private String email;
    private String phoneNumber;
    
	// authorities 表中的数据
    private List<String> authorities;

    // getter/setter ...
}
```

---


###### 1.3.2. 编写 Mapper 接口

```
@Mapper
public interface UserWithAuthoritieMapper {

	// 查询用户
    UserWithAuthoritie findUserWithAuthoritieByUserName(String username);
    
}
```

---


###### 1.3.3. 编写 SQL 语句

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="com.example.security.mapper.UserWithAuthoritieMapper">

    <resultMap id="UserWithAuthoritieResultMap" type="com.example.security.model.entity.UserWithAuthoritie">
        <id     property="userId"                   column="user_id"                   jdbcType="INTEGER"/>
        <result property="username"                 column="username"                  jdbcType="VARCHAR"/>
        <result property="password"                 column="password"                  jdbcType="VARCHAR"/>
        <result property="isAccountNonExpired"      column="isAccountNonExpired"       jdbcType="TINYINT"/>
        <result property="isAccountNonLocked"       column="isAccountNonLocked"        jdbcType="TINYINT"/>
        <result property="isCredentialsNonExpired"  column="isCredentialsNonExpired"   jdbcType="TINYINT"/>
        <result property="isEnabled"                column="isEnabled"                 jdbcType="TINYINT"/>
        <result property="email"                    column="email"                     jdbcType="VARCHAR"/>
        <result property="phoneNumber"              column="phoneNumber"               jdbcType="VARCHAR"/>
        <collection property="authorities" ofType="java.lang.String" column="authoritie_name"/>
    </resultMap>

    <!-- 查询 -->
    <select id="findUserWithAuthoritieByUserName"
            parameterType="string"
            resultMap="UserWithAuthoritieResultMap">
        SELECT
            u.user_id,
            u.username,
            u.password,
            u.isAccountNonExpired,
            u.isAccountNonLocked,
            u.isCredentialsNonExpired,
            u.isEnabled,
            u.email,
            u.phoneNumber,
            ua.authoritie_name
        FROM users u
                 LEFT JOIN user_authorities ua ON u.username = ua.username
        WHERE u.username = #{username}
    </select>
</mapper>
```

---


###### 1.4. 实现 UserDetails 接口

`CustomerDetailsImpl` 不属于三层架构，严格来说，他应该放到 `model.entity`
```
public class CustomerDetailsImpl implements UserDetails {

    private final String username;
    private final String password;
    private final Boolean isAccountNonExpired;
    private final Boolean isAccountNonLocked;
    private final Boolean isCredentialsNonExpired;
    private final Boolean isEnabled;
    private final List<GrantedAuthority> authorities;

    // 扩展的字段
    private final String email;
    private final String phoneNumber;

    public CustomerDetailsImpl(UserWithAuthoritie user) {
        this.username = user.getUsername();
        this.password = user.getPassword();
        this.isAccountNonExpired = user.getAccountNonExpired();
        this.isAccountNonLocked = user.getAccountNonLocked();
        this.isCredentialsNonExpired = user.getCredentialsNonExpired();
        this.isEnabled = user.getEnabled();
        this.email = user.getEmail();
        this.phoneNumber = user.getPhoneNumber();
        List<GrantedAuthority> grantedAuthorities = new ArrayList<>();
        if (user.getAuthorities() != null) {
            for (String authority : user.getAuthorities()) {
                grantedAuthorities.add(new SimpleGrantedAuthority(authority));
            }
        }
        this.authorities = grantedAuthorities;

    }

    // 扩展的方法
    public String getEmail() {
        return email;
    }

    public String getPhoneNumber() {
        return phoneNumber;
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return this.authorities;
    }

    @Override
    public String getPassword() {
        return this.password;
    }

    @Override
    public String getUsername() {
        return this.username;
    }

    @Override
    public boolean isAccountNonExpired() {
        return this.isAccountNonExpired;
    }

    @Override
    public boolean isAccountNonLocked() {
        return this.isAccountNonLocked;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return this.isCredentialsNonExpired;
    }

    @Override
    public boolean isEnabled() {
        return this.isEnabled;
    }
}
```

---


###### 1.5. 实现 UserDetailsService 接口

```
@Service // 不要忘记
public class CustomerDetailsServiceImpl implements UserDetailsService {

    @Autowired
    private UserWithAuthoritieMapper userWithAuthoritieMapper;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        UserWithAuthoritie user = userWithAuthoritieMapper.findUserWithAuthoritieByUserName(username);
        if (user == null) {
            throw new UsernameNotFoundException("用户未找到: " + username);
        }
        return new CustomerDetailsImpl(user);
    }
}
```

---


##### 1.6. 进行 Security 相关配置

```
@Configuration
@EnableMethodSecurity // 1. 启用方法级别的访问控制
@EnableWebSecurity // 2. 启用 Spring Security 安全功能
public class SecurityConfig {
    private final CustomerDetailsServiceImpl customerdetails;

    public SecurityConfig(CustomerDetailsServiceImpl customerdetails) {
        this.customerdetails = customerdetails;
    }

    // 3. 配置 AuthenticationManager
    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration configuration) throws Exception {
        return configuration.getAuthenticationManager(); // 返回默认的 AuthenticationManager
    }

    // 4. 配置 密码加密器，用于 AuthenticationManager 加密 password 后与 CustomerDetailsImpl 进行对比
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            // 5. 禁用默认表单登录（也就禁用了 UsernamePasswordAuthenticationFilter 过滤器）
            .formLogin(form -> form.disable())
            // 6. 禁用默认注销功能
            .logout(logout -> logout.disable())
            // 7. 资源级别的访问控制
            .authorizeHttpRequests(auth -> auth
                    .requestMatchers("/auth/login", "auth/logout").permitAll()
                    .requestMatchers("/auth/test").hasAnyAuthority("ROLE_CEO")
                    .anyRequest().authenticated()
            )
            // 8. 用户 未认证、权限不足 的处理
            .exceptionHandling(handler -> handler
                    // 未认证时的响应
                    .authenticationEntryPoint((request, response, authException) -> {
                                response.setContentType("application/json;charset=UTF-8");
                                response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                                response.getWriter().write("{\"error\":\"未认证，请先登录\"}");
                            }
                    )
                    // 权限不足时的响应
                    .accessDeniedHandler((request, response, accessDeniedException) -> {
                                response.setContentType("application/json;charset=UTF-8");
                                response.setStatus(HttpServletResponse.SC_FORBIDDEN);
                                response.getWriter().write("{\"error\":\"权限不足，无法访问此资源\"}");
                            }
                    )
            )
            // 9. HttpSession 相关配置
            .sessionManagement(session -> session
                    .sessionCreationPolicy(SessionCreationPolicy.STATELESS) // 禁用 HttpSession
            )
            // 10. 配置 UserDetailsServiceImpl，用于 AuthenticationManager 从数据库中取 UserDetailsImpl
            .userDetailsService(customerdetails) // 显式告诉用你的UserDetailsService
			// 11. CSRF 攻击防护
			.csrf(csrf -> csrf.disable());
        return http.build();
    }
}
```

> [!NOTE] 注意事项
> 1. 虽然我们禁用了 `HttpSession`，导致负责从 `HttpSession` 中获取 `SecurityContext` 的 `SecurityContextPersistenceFilter` 似乎失去了作用，但我仍然需要它在每次请求开始时创建一个新的 `SecurityContext`。因此，我保留了该过滤器，但注意不要开启其保存 `SecurityContext` 的功能。

----

##### 创建 JWT 响应实体类

理应放到 `model.dto` 下
```
public class JwtResponse {

    private String token;

    // 构造器
    public JwtResponse(String token) {
        this.token = token;
    }

    // getter 和 setter
    public String getToken() {
        return token;
    }

    public void setToken(String token) {
        this.token = token;
    }
}
```

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


### Spring Boot 集成 HTTPS

---


# 补充

### 1. Spring Security 相关配置

#### 1.1. Security 相关配置模版

Spring Security 的配置主要在 Java 配置类中进行，而不是在 `application.yml` 文件中，因为安全配置通常涉及到逻辑和条件判断，这些无法简单地通过属性文件表达，我们可以完成以下配置：
```
@Configuration
@EnableMethodSecurity // 1. 启用方法级别的访问控制
@EnableWebSecurity // 2. 启用 Spring Security 安全功能
public class SecurityConfig {
    private final CustomerDetailsServiceImpl customerdetails;

    public SecurityConfig(CustomerDetailsServiceImpl customerdetails) {
        this.customerdetails = customerdetails;
    }

    // 3. 配置 AuthenticationManager
    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration configuration) throws Exception {
        return configuration.getAuthenticationManager(); // 返回默认的 AuthenticationManager
    }

    // 4. 配置 密码加密器，用于 AuthenticationManager 加密 password 后与 CustomerDetailsImpl 进行对比
    @Bean
    public PasswordEncoder passwordEncoder() {
        return NoOpPasswordEncoder.getInstance();
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            // 5. 禁用默认表单登录（也就禁用了 UsernamePasswordAuthenticationFilter 过滤器）
            .formLogin(form -> form.disable())
            
            // 6. 禁用默认注销功能
            .logout(logout -> logout.disable())
            
            // 7. 资源级别的访问控制
            .authorizeHttpRequests("见下文")
            
            // 8. 用户 未认证、权限不足 的处理
            .exceptionHandling(handler -> handler
                    // 未认证时的响应
                    .authenticationEntryPoint((request, response, authException) -> {
                                response.setContentType("application/json;charset=UTF-8");
                                response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                                response.getWriter().write("{\"error\":\"未认证，请先登录\"}");
                            }
                    )
                    // 权限不足时的响应
                    .accessDeniedHandler((request, response, accessDeniedException) -> {
                                response.setContentType("application/json;charset=UTF-8");
                                response.setStatus(HttpServletResponse.SC_FORBIDDEN);
                                response.getWriter().write("{\"error\":\"权限不足，无法访问此资源\"}");
                            }
                    )
            )
            
            // 9. HttpSession 相关配置
            .sessionManagement(session -> session
                    .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
            )
            
            // 10. 默认 SecurityContext 查找和保存策略
            .securityContext("见下文"）
            
            // 11. 配置 UserDetailsServiceImpl，用于 AuthenticationManager 从数据库中取 UserDetailsImpl
            .userDetailsService(customerdetails) // 显式告诉用你的UserDetailsService
            
            // 12. CSRF 攻击防护
            .csrf("见下文")
            
            // 13. CORS 支持
            .cors("见下文")
            
            // 14. 添加自定义过滤器
            .addFilterAt("见下文");
            
        return http.build(); // 构建 SecurityFilterChain 对象  
    }
}
```

---


#### 1.2. 方法级别的访问控制  

==1.配置类上添加注解==
```
@Configuration  
@EnableMethodSecurity  // 启用方法级别的访问控制  
@EnableWebSecurity  
public class SecurityConfig {  
  ......
}
```


==2.方法级别的访问控制==
需要注意，一般在服务层（Service 层） 进行控制。
```
@Service
public class UserService {
    @PreAuthorize("hasRole('ADMIN')")
    public void adminMethod() {
        // 只有 ADMIN 角色可以执行该方法
    }

    @PreAuthorize("hasAuthority('USER_READ')") 
    public void readUser() {
        // 只有具有 USER_READ 权限的用户可以执行
    }
}
```

---


#### 1.3. 密码加密器

##### 1.3.1. 密码加密器概述

在实际应用中，用户密码绝不能以明文方式存储，因为这会带来极大的安全风险。为确保安全性，我们必须在存储前对密码进行加密处理。

在 API 中验证密码时，我们面临的问题是如何将用户提交的明文密码与数据库中存储的加密密码进行比对。为了解决这一问题，需要使用密码加密器，它既能对明文密码进行加密，也能校验用户输入的密码是否与加密密码匹配。

而 `PasswordEncoder` 是 Spring Security 提供的一个接口，同时它也提供了很多实现，能**实现密码加密**和**校验密码匹配**：
1. ==实现密码加密==：
	1. 明文密码转换为不可逆的密文，从而保护用户密码安全。
2. ==校验密码匹配==：
	1. 在用户登录时，将输入的明文密码**加密后**与存储的密文密码进行比对，验证密码的正确性。
	2. 如果一个明文、一个密文进行对比，将验证错误

---


##### 1.3.2. PaswordEncoder 常用实现

Spring Security 提供了几种常见的 `PasswordEncoder` 实现类，常用的有：
1. ==BCryptPasswordEncoder==：
	1. 基于 BCrypt 算法的密码加密工具，提供较高安全的密码加密方式，能够有效防止暴力破解，适用于大多数应用。
	2. 需要注意的是：该方法不能解密，只能对比，无需秘钥。
	3. 通过 `return new BCryptPasswordEncoder();` 获得
2. ==NoOpPasswordEncoder==：
	2. 不对密码进行加密，直接返回原始密码，通常用于开发或测试环境，不建议在生产环境中使用
	3. 通过 `return NoOpPasswordEncoder.getInstance();` 获得
3. ==Pbkdf2PasswordEncoder==：
	1. 基于 PBKDF2 算法的密码加密工具。
4. ==SCryptPasswordEncoder==：
	1. 基于 scrypt 算法的密码加密工具。

---


##### 1.3.3. PasswordEncoder 使用步骤

==1.配置 PasswordEncoder==
根据 Spring Security 的要求，无论是否对密码进行加密，都必须配置一个密码加密器。具体来说，需要在 Spring Security 配置类中选择适当的实现类，并将其声明为一个 Bean。
``` 
@Configuration
@EnableWebSecurity 
public class SecurityConfig {
    @Bean 
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(); // 选择实现类
    }
}
```


==2.实现密码加密==
```
@Controller  
@RequestMapping("/security")  
public class PasswordController {  
    private final PasswordEncoder passwordEncoder;  
  
    @Autowired  
    public PasswordController(PasswordEncoder passwordEncoder) {  
        this.passwordEncoder = passwordEncoder;  
    }  
      
    // 通过方法处理密码加密  
    @RequestMapping("/encode-password")  
    @ResponseBody  
    public String encodePassword() {  
        String password = "myPasswordxxxxxxx";  
        // 在方法内调用 passwordEncoder 进行密码加密  
        String encodedPassword = passwordEncoder.encode(password);  
        return "Encoded Password: " + encodedPassword;  
    }  
}
```


==3.校验密码匹配==
```
@Controller
@RequestMapping("/security")  
public class PasswordController {

    private final PasswordEncoder passwordEncoder;

    @Autowired
    public PasswordController(PasswordEncoder passwordEncoder) {
        this.passwordEncoder = passwordEncoder;
    }

    // 通过方法验证密码
    @RequestMapping("/verify-password")
    @ResponseBody
    public String verifyPassword() {
        String rawPassword = "myPasswordxxxxxxx";
        String encodedPassword = "xxxxxxxxxxxxxxxxxx";  // 模拟数据库中的加密密码

        // 使用 matches 方法验证密码是否匹配（自动对 rawPassword 加密，然后与数据库中的加密密码进行匹配）
        boolean matches = passwordEncoder.matches(rawPassword, encodedPassword);
        
        return matches ? "Password is valid" : "Invalid password";
    }
}
```

---









#### 1.4. 资源级别的访问控制

```
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/public/**", "/error").permitAll()
    .requestMatchers("/admin/**").hasRole("ADMIN")
    .requestMatchers("/manage/**").hasAuthority("MANAGE_PRIVILEGE")
    .requestMatchers("/special/**").access("hasRole('ADMIN') and hasIpAddress('192.168.1.0/24')") 
    .anyRequest().authenticated() // 其他所有路径均需经过认证
"""
1. requestMatchers()：
	1. 设置 特定资源请求路径 访问控制规则
2. anyRequest()：
	1. 用于设置除上述规则外，其余 所有资源请求路径 的访问控制规则
3. permitAll()：
	1. 允许所有用户（包括未认证用户）访问该资源。
4. denyAll()：
	1. 禁止所有用户访问该资源。
5. hasRole()：
	1. 要求用户必须具备指定角色才能访问
	2. 注意：该方法会自动在角色名前添加 "ROLE_" 前缀，即：hasRole("ADMIN") 代表 ROLE_ADMIN
6. hasAuthority()：
	1. 要求用户必须具备指定权限才能访问，我们用这个就好
	2. 注意：不会自动添加前缀，提供完整权限名称即可
7. authenticated()：
	1. 要求用户已通过身份验证后才能访问。
	2. 仅限制陌生人，即未认证用户
8. access()：
	1. 支持使用 SpEL 表达式，实现更复杂的访问控制逻辑。
	2. 例如：access("hasRole('ADMIN') and hasIpAddress('192.168.1.0/24')") ，是说此路径仅允许拥有 ADMIN 角色且 IP 地址位于 192.168.1.0/24 网段的用户访问
"""
```

> [!NOTE] 注意事项
> 1. 如需放行所有请求，可配置为：
```
http.authorizeHttpRequests(auth -> auth.anyRequest().permitAll());
```

---


#### 1.5. HttpSession 相关配置

```
http.sessionManagement(session -> session  
    .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)  
    .maximumSessions(1)
    .maxSessionsPreventsLogin(true)
);
"""
1. sessionCreationPolicy()：会话创建策略
	1. SessionCreationPolicy.ALWAYS：
		1. 始终创建会话。每个请求都会新建一个 HttpSession，覆盖之前的会话，并返回新的 JSESSIONID Cookie。
	2. SessionCreationPolicy.IF_REQUIRED（默认）：
		1. 按需创建会话。当需要使用 HttpSession 时，Spring Security 会自动创建会话，并返回对应的 JSESSIONID Cookie。
	3. SessionCreationPolicy.STATELESS：
		1. 不使用会话，适用于无状态应用场景（禁用 HttpSession，如 JWT）
2. maximumSessions()：
	1. 并发会话控制。限制每个用户在同一时间内的会话数量，也即用户可在多少台设备上同时登录（默认情况下，不限制）。
3. maxSessionsPreventsLogin()：
	1. 是否阻止新会话登录。
	2. true：
		1. 达到最大会话数后阻止新会话登录，不允许替换旧会话。
	3. false(默认)：
		1. 允许新会话登录并替换旧会话，此时旧会话将被注销，旧的 JSESSIONID 失效。通常系统会通知用户被挤下线，并重定向到登录页面要求重新登录。
"""
```

> [!NOTE] 注意事项
> 1. Spring Security 本身不负责设置 `HttpSession` 的会话超时时间。会话超时时间由 Servlet 容器或 Spring Boot 配置决定。
> 2. `HttpSession` 存储在服务器端，别误以为存储在用户端了
> 3. 如果不希望使用会话，可以配置为：
```
http.sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS);
```

---


#### 1.6. 默认 SecurityContext 查找和保存策略

在执行 API 方法前：
1. 自动根据 `JSESSIONID` 向 `HttpSession` 查找 `SecurityContext`（内保存最重要的 `Authentication`）
2. 若存在 `SecurityContext`，便将其加载到本线程的 `SecurityContextHolder` 中，实现了 “记住我” 功能
3. 若不存在 `SecurityContext`，则在本线程中自动初始化一个新的 `SecurityContext` 

在执行 API 方法后：
1. 自动将本线程的 `SecurityContext` 存入 `HttpSession`，以便在后续请求中维持用户身份（**需要手动开启**）
2. 随后，过滤器会清空本线程 `SecurityContextHolder`，防止 `SecurityContext` 在后续请求中被无意复用，从而确保每个请求都能独立执行认证和授权流程。
```
// 1. 禁用默认 SecurityContext 查找策略（也就禁用了 SecurityContextPersistenceFilter 过滤器）
http.securityContext(securityContext -> securityContext.disable())  


// 2. 启用默认 SecurityContext **保存** 策略
http.securityContext(securityContext -> securityContext.requireExplicitSave(false));
```

---

	
#### 1.7. CSRF 攻击防护

Spring Security 默认启用跨站请求伪造（CSRF）防护机制，基于 **CSRF Token** 实现安全校验。其工作原理如下：
1. 用户首次登录页面时，需要我们手动生成一个随机的 CSRF Token，将其保存在服务器端的 **HttpSession** 中，并同步返回给前端。
2. 前端需妥善保存该 Token，在执行修改服务器状态的敏感操作（如 POST、PUT、DELETE、PATCH 等）时，前端必须将该 Token 携带，通常以请求头（默认名称：`X-CSRF-TOKEN`）或请求参数（名称：`_csrf`）的方式携带。
3. 服务器在接收到请求后，会比对请求中携带的 CSRF Token 与存储在 HTTP Session 中的 Token 是否一致。如果两者不匹配或者无 CSRF Token，则会抛出 `accessDeniedException`（无权限异常），从而阻止非法请求继续执行。
4. 即使攻击者能利用用户的 Cookie 发起请求，由于敏感操作必须携带合法的 CSRF Token，而攻击者无法获取或伪造该 Token，因而能有效防止 CSRF 攻击。
```
@Bean 
public CsrfTokenRepository csrfTokenRepository() { 

		HttpSessionCsrfTokenRepository repository = new HttpSessionCsrfTokenRepository(); // 在HttpSession 中存储
	
		repository.setHeaderName("X-CSRF-TOKEN"); // 可自定义请求头名称 
		repository.setParameterName("_csrfToken"); // 可自定义请求参数名称 
		return repository; 
}

http.csrf(csrf -> csrf
	.ignoringRequestMatchers("/login","/websocket/**", "/api/public/**"); // 忽略对这些路径的 CSRF 保护
	.csrfTokenRepository(csrfTokenRepository()) // 使用我们自定义的 Token 存储库
)；
"""
1. ignoringRequestMatchers()：
	1. 忽略对这些路径的 CSRF 保护
2. csrfTokenRepository()：
	1. 指定 Token 的存储库，一般使用我们自定义的 Token 存储库
"""
```

> [!NOTE] 注意事项
> 1. 如果基于 JWT，我们直接禁用 CSRF 防护
> 2. 如果你莫名其妙返回 `"error": "权限不足，无法访问此资源"` ，大概率是 CSRF 的问题
> 3. 如果想要禁用 CSRF 防护，可以配置为：
```
http.csrf(csrf -> csrf.disable());
```

----


#### 1.8. CORS 支持

Spring Security 默认关闭 CORS（跨域资源共享）。
```
// 1. 启用 CORS 支持
http.cors();

// 2. 配置跨域规则
@Bean
public CorsConfigurationSource corsConfigurationSource() {

	CorsConfiguration configuration = new CorsConfiguration();
	configuration.addAllowedOrigin("http://frontend.example.com"); // 只允许特定前端跨域
	configuration.addAllowedMethod("*"); // 支持所有方法
	configuration.addAllowedHeader("*"); // 支持所有头部
	configuration.setAllowCredentials(true); // 允许携带 Cookie、Session 等凭证

	UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
	source.registerCorsConfiguration("/**", configuration); // 应用于本程序哪些路径
	return source;
}
```

> [!NOTE] 注意事项
> 1. CORS 只解决跨域访问问题，对于受保护资源的访问，仍需要完成认证，甚至需要校验 CSRF Token。建议在跨域请求时携带一个 JWT，这样可以自动完成身份认证，无需手动处理。

---


#### 1.9. 添加自定义过滤器

```
// 1. 直接添加过滤器，添加的过滤器必须是 Spring Security 提供的过滤器或其子类的实例
http.addFilter(new CustomFilter());

// 2. 在指定的过滤器位置添加过滤器，新添加的过滤器会替换指定位置的原有过滤器
http.addFilterAt(new CustomFilter(),UsernamePasswordAuthenticationFilter.class);

// 3. 在指定过滤器之前添加过滤器，自定义过滤器会在指定过滤器之前执行。
http.addFilterBefore(new CustomFilter(),UsernamePasswordAuthenticationFilter.class);

// 4. 在指定过滤器之后添加过滤器，自定义过滤器会在指定过滤器之后执行。
http.addFilterAfter(new CustomFilter(),UsernamePasswordAuthenticationFilter.class);
```

---


### 2. JWT

#### 2.1. JWT 概述

JSON Web Token (JWT) 是一种基于JSON格式的开放标准（RFC 7519）身份验证和信息交换机制，其核心价值在于能够在不依赖服务器端会话存储（`HttpSession`）的情况下，安全地验证客户端身份并传递必要的用户信息，这使其成为现代Web应用、微服务架构和单点登录系统中的关键技术。

---


#### 2.2. JWT 组成部分

JWT（JSON Web Token）采用简洁明了的三段式结构，各部分之间以点号（`.`）分隔，形成类似 `"xxxxx.yyyyy.zzzzz"` 的字符串。这样的设计不仅保证了信息传输的高效性，也具备良好的结构可读性和完整性校验能力，例如：
```
eyJhbGciOiJIUzUxMiJ9.eyJzdWIiOiJhZG1pbiIsImNyZWF0ZWQiOjE1NTY3NzkxMjUzMDksImV4cCI6MTU1NzM4MzkyNX0.d-iki0193X0bBOETf2UN3r3PotNIEAV7mzIxxeI5IxFyzzkOZxS0PGfF_SK6wxCv2K8S0cZjMkv6b5bCqc0VBw
```

我们可以在该网站轻松解码： https://jwt.io/


==第一部分：Header（头部）==
Header 通常是一个 JSON 对象，用于说明令牌的类型（即 "JWT"）以及所使用的**签名算法**（如 HMAC SHA256 或 RSA 等，作用于签名的算法）。例如：
```
{ "alg": "HS256", "typ": "JWT" }
```
这个部分在编码后会使用 Base64Url 编码形成字符串，作为 JWT 的第一个部分。


==第二部分：Payload（载荷）==
Payload 是 JWT 的核心部分，包含了实际需要传输的声明（claims），如用户 ID、用户名、角色权限、令牌的签发时间（`iat`）、过期时间（`exp`）等。例如：
```
{ "sub": "1234567890", "iat": 1516239022, "exp": 1516239022 + 1000*60*60*24*15 }
```
Payload 同样会被进行 Base64Url 编码，成为 JWT 的第二部分。

注意：sub 是专门存用户名、用户 ID 的，iat 是专门存签发时间的，exp 是专门存过期时间的，其他的乱七八糟的字段都是我们的自定义字段


==第三部分：Signature（签名）==
Signature 是 **密钥 + 算法** 进行签名，用于验证 JWT 在传输过程中是否被篡改。生成方式通常如下：
```
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```
签名部分确保了前两部分内容的完整性，如果任何一部分被更改，签名验证将失败，从而防止伪造或篡改。

> [!NOTE] 注意事项
> 1. 由于密钥仅掌握在你手中，即使攻击者截获了 JWT，也无法伪造一个合法的 Token。这意味着 JWT 能够保证令牌的真实性，但无法阻止被盗用（你有这个密钥，能用它验证 JWT 是不是用你的密钥生成的。攻击者没有你的密钥，哪怕截获了用户名，用别的密钥伪造 JWT，也会被你的密钥一眼识破，无法通过验证。）
> 2. 由于 Header 和 Payload 是明文（Base64Url 编码）形式，任何人都可以解码查看内容，因此必须使用 HTTPS 进行传输，并且不要存储敏感隐私数据。
> 3. 如果攻击者获取了完整的 JWT（例如通过本地存储劫持），就可以在有效期内冒充用户发起请求。因此应尽可能**缩短 JWT 的有效期**（如 15 分钟），不然只能通过 “黑名单” 机制或 Refresh Token 来进行补救。

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


### RBAC

RBAC 的核心思想就是：“不直接给用户分配权限，而是把权限分配给角色，再把用户加入到相应角色”
1. ==用户（User）==：
	1. 系统中的操作主体，比如某个登录系统的人。
	2. 命名方式：
		1. 直接使用用户账号 / 用户名，例如：
		2. alice、bob、zhangsan
		3. user_001、admin_001
		4. alice@example、1386524789等等
	3. 注意事项：
		1. 唯一性是关键，用户名不能重复
2. ==角色（Role）==：
	1. 一组权限的集合，代表一种职责，比如“管理员”“编辑”“普通用户”。
	2. 命名方式：
		1. 按照 职责 / 岗位 的方式进行命名，例如：
		2. ceo（首席执行官）
		3. admin（管理员）
		4. editor（编辑）
3. ==权限（Permission）==：
	1. 对系统中资源的访问权，比如“读数据”“改配置”“删用户”等。
	2. 命名方式：
		1. 按照 模块_操作 的方式进行命名，例如：
		2. user_read（读取用户信息）
		3. order_delete（删除订单）

---






