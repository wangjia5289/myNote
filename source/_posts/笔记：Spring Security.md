---
title: 笔记：Spring Security
date: 2025-05-18
categories:
  - Java
  - Spring 生态
  - Spring Security
tags: 
author: 霸天
layout: post
---
# 一、理论

## 1. 导图

![](source/_posts/笔记：Spring%20Security/Map：SpringSecurity.xmind)

---


## 2. Spring Security 执行流程

<font color="#92d050">1. 用户请求（客户端请求）</font>
每次用户访问受 `Spring Security` 保护的资源，都会经过以下流程


<font color="#92d050">2. SecurityContextPersistenceFilter 介入</font>
自动为本线程初始化 `SecurityContextHolder` 并根据 `JSESSIONID` 向 `HttpSession` 查找 `SecurityContext`（其内保存最重要的 `Authentication`）
1. 若存在 `SecurityContext`，便将其加载到本线程的 `SecurityContextHolder` 中（基于 HttpSession 实现 “记住我” 功能，我们也可基于 JWT 实现 “记住我” 功能）
2. 若不存在 `SecurityContext`，则在本线程中自动初始化一个新的 `SecurityContext`

即使我们不打算通过 `HttpSession` 实现 “记住我” 功能（如使用 JWT），甚至完全不使用 `HttpSession`，我们仍然建议保留这个过滤器，因为它自动为本线程初始化 `SecurityContextHolder`、并自动创建 `SecurityContext`，这个能力实在太香了。
![](image-20250628224744251.png)


<font color="#92d050">3. CsrfFilter 介入</font>
该过滤器会尝试从我们配置的 CSRF Token 存储位置（配置的 `HttpSessionCsrfTokenRepository`）中加载 CSRF Token，且所有请求都会经过这一尝试。若成功加载，Token 会被放入 `HttpServletRequest` 中；如果未加载到，则会创建一个新的 CSRF Token，并同样放入请求中。

对于启用 CSRF 防护的路径，当执行修改服务器状态的敏感操作（如 POST、PUT、DELETE、PATCH 等）时，过滤器会检查前端是否在指定的位置（配置的 `.setHeaderName("X-CSRF-TOKEN")` 等位置）携带了 Token。若未携带，则抛出 `MissingCsrfTokenException`；若携带，则会将前端 Token 与 `HttpServletRequest` 中的 Token 进行比对，匹配则继续执行，不匹配则抛出 `InvalidCsrfTokenException`。

> [!NOTE] 注意事项
> 1. CSRF 相关的异常都是 `AccessDeniedException` 的子类，所以我们应该在处理 `AccessDeniedException` 时处理这两个异常


<font color="#92d050">4. UsernamePasswordAuthenticationFilter 介入</font>
该过滤器主要用于前后端未分离的场景，用于处理默认 `/login` 路径下的登录请求。  

在前后端分离的架构中无需深入关注其具体逻辑，只需了解其在过滤器链中的位置，以便在插入自定义过滤器时能准确定位。


<font color="#92d050">5. AnonymousAuthenticationFilter 介入</font>
如果当前没有任何 `Authentication`，系统会自动创建一个匿名身份，以避免后续流程中出现空指针异常。


<font color="#92d050">6. FilterSecurityInterceptor 介入</font>
首先检查当前线程中是否存在 `Authentication`（无论是否为匿名身份），如果不存在，则抛出 `AuthenticationException`，表示用户尚未进行认证。

接着判断是否为匿名用户访问受保护资源：即若用户尚未认证（即为匿名身份 `Authentication`），且访问的资源未被标注为 `permitAll`，则抛出 `AuthenticationException` 异常。

最后检查当前用户是否具备访问目标资源或方法的权限（包括资源级别和方法级别的访问控制），若权限不足，则抛出 `AccessDeniedException`。
> [!NOTE] 注意事项
> 1. 整个流程中的异常由 `ExceptionTranslation` 过滤器统一处理，负责捕获**整个过滤器链中**抛出的 `AuthenticationException` 和 `AccessDeniedException` 异常，并执行相应的处理逻辑。


<font color="#92d050">7. 执行 API</font>
在这一步，才真正开始执行我们的 API 逻辑；如果是登录 API，并且通过 AuthenticationManager 进行认证，流程如下：
![](image-20250628210023140.png)


<font color="#92d050">8. SecurityContextPersistenceFilter 再次介入</font>
它会自动将本线程的 `SecurityContext` 存入服务器的 `HttpSession`，以便在后续请求中维持用户身份（需要手动开启）

 随后，过滤器会清空本线程 `SecurityContextHolder`，防止 `SecurityContext` 在后续请求中被无意复用，从而确保每个请求都能独立执行认证和授权流程。

----


## 3. Spring Security 配置

### 3.1. Spring Security 配置模板

Spring Security 的配置主要在 Java 配置类中进行，而不是在 `application.yml` 文件中，因为安全配置通常涉及到逻辑和条件判断，这些无法简单地通过属性文件表达，我们可以完成以下配置：

SecurityConfiguration 类在 `com.example.securitywithhttpsession.configuration` 包下，直接粘贴这份配置模板，再根据下方的详细说明按需进行调整。
```
@Configuration
@EnableMethodSecurity // 1. 启用方法级别的访问控制
@EnableWebSecurity // 2. 启用 Spring Security 安全机制
public class SecurityConfiguration {

    // 详见下文：配置 CSRF 攻击防护（配置 CsrfFilter 过滤器）
    @Bean
    public CsrfTokenRepository csrfTokenRepository() {

        HttpSessionCsrfTokenRepository repository = new HttpSessionCsrfTokenRepository(); // CSRF Token 生成与加载的位置，这里是生成并保存在 HttpSession 中，并从 HttpSession 中加载

        repository.setHeaderName("X-CSRF-TOKEN"); // 前端可在 X-CSRF-TOKEN 请求头中携带 CSRF Token
        repository.setParameterName("_csrfToken"); // 前端可在 _csrfToken 请求体中携带 CSRF Token
        return repository;
    }

    // 3. 配置 CORS 跨域资源共享
    @Bean
    public CorsConfigurationSource corsConfigurationSource() {

        CorsConfiguration configuration = new CorsConfiguration();
        configuration.setAllowedOrigins(List.of("http://frontend.example.com", "http://localhost:3000"));
        configuration.setAllowedOriginPatterns(List.of("http://*.example.com"));
        configuration.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        configuration.setAllowedHeaders(List.of("*"));
        configuration.setAllowCredentials(true);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        Map<String, CorsConfiguration> configMap = new HashMap<>();
        configMap.put("/api/**", configuration);
        configMap.put("/admin/**", configuration);
        source.setCorsConfigurations(configMap);

        return source;
    }

    // 4. 配置 AuthenticationManager
    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration authenticationConfiguration) throws Exception {
        return authenticationConfiguration.getAuthenticationManager(); // 从 Spring Security 配置中获取其默认的 AuthenticationManager 实例
    }

    // 5. 配置 密码加密器
    @Bean
    public PasswordEncoder passwordEncoder() {
        return NoOpPasswordEncoder.getInstance(); // 返回合适的实现类
    }

    // 配置 SecurityFilterChain，即我们熟知的那些过滤器链
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity httpSecurity) throws Exception {
        httpSecurity
                // 6. 禁用默认表单登录，同时禁用 UsernamePasswordAuthenticationFilter 过滤器
                .formLogin(form -> {
                    form.disable();
                })

                // 7. 禁用默认注销功能
                .logout(logout -> {
                    logout.disable();
                })

                // 8. 配置资源级别的访问控制
                .authorizeHttpRequests(auth -> {
                    auth
                        .requestMatchers("/public/**").permitAll()
                        .anyRequest().authenticated(); // 其他所有路径均需通过认证
                })

                // 9. 配置用户 未认证、权限不足 的处理
                .exceptionHandling(handler -> {
                    handler
                        // 未认证时的响应（处理 AuthenticationException 异常）
                        .authenticationEntryPoint((request, response, authException) -> {
                            response.setContentType("application/json;charset=UTF-8");
                            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                            response.getWriter().write("{\"error\":\"未认证，请先登录\"}");
                        })
                        // 权限不足时的响应（处理 AccessDeniedException 异常）
                        .accessDeniedHandler((request, response, accessDeniedException) -> {
                            response.setContentType("application/json;charset=UTF-8");
                            // 先判断是否是 CSRF 相关异常
                            if (accessDeniedException instanceof MissingCsrfTokenException
                                    || accessDeniedException instanceof InvalidCsrfTokenException) {
                                response.setStatus(HttpServletResponse.SC_FORBIDDEN);
                                response.getWriter().write("{\"error\":\"CSRF Token 校验失败，无法访问此资源\"}");
                            } else {
                                // 普通权限不足
                                response.setStatus(HttpServletResponse.SC_FORBIDDEN);
                                response.getWriter().write("{\"error\":\"权限不足，无法访问此资源\"}");
                            }
                        });
                })

                // 10. 配置 HttpSession
                .sessionManagement(session -> {
                    session
                        .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
                        .maximumSessions(1)
                        .maxSessionsPreventsLogin(true);
                })

                // 11. 配置 SecurityContextPersistenceFilter 过滤器
                .securityContext(security -> {
                    security.requireExplicitSave(false);
                })

                // 12. 配置 CSRF 攻击防护（配置 CsrfFilter 过滤器）
                .csrf(csrf -> {
                    csrf
                        .ignoringRequestMatchers("/login") // 忽略对这些路径的 CSRF 保护（默认全部保护）
                        .csrfTokenRepository(csrfTokenRepository()); // 使用我们自定义的 Token 存储库
                })
                // 13. 添加自定义过滤器
                .addFilterAt(xxxx);
        return httpSecurity.build(); // 构建 SecurityFilterChain 对象
    }
}
```

---


### 3.2. 启用方法级别的访问控制

我们先来了解一下资源级别的访问控制，资源级别的访问控制是指：只有具备指定权限或身份的用户，才能访问特定路径下的资源，例如 `/api/admin/**` ，详见下文：资源级别的访问控制。

而方法级别的访问控制则是指：用户必须具备指定的权限或身份，才能调用某方法，详细步骤如下：

<font color="#92d050">1. 配置类上添加 @EnableMethodSecurity 注解</font>
```
@Configuration  
@EnableMethodSecurity  // 启用方法级别的访问控制  
@EnableWebSecurity  
public class SecurityConfig {  
  ......
}
```


<font color="#92d050">2. 进行方法级别的访问控制</font>
需要注意的是，方法级别的访问控制，一般进行在服务层（Service 层）、控制器层（Controller 层）
```
@Service
public class UserService {
    @PreAuthorize("hasRole('ADMIN')")
    public void adminMethod() {
        // 只有 ROLE_ADMIN 角色可以执行该方法
    }

    @PreAuthorize("hasAuthority('user:user:select')") 
    public void readUser() {
        // 只有具有 user:user:select 权限的用户可以执行 用户模块的用户表的查询操作
    }
}
```

> [!NOTE] 注意事项
> 1. 如果你在 `Service` 层进行了方法级别的访问控制，那么就有可能出现：能够执行前半部分逻辑，但调用 `Service` 方法时会被拒绝，并抛出 AccessDeniedException
```
@GetMapping("/test")  
public String test() {  
    System.out.println("正在执行只有 test:test:test 权限才能执行的 Service 方法");  
    System.out.println("现在的 Authentication 信息如下：");  
    System.out.println(AuthenticationUtils.getAuthentication());  
    String testString = testService.test();  // 对 testService 进行方法级别的访问控制
    return testString;  
}
```

---


### 3.3. 启用 Spring Security 安全机制

在配置类中加上 `@EnableWebSecurity` 后，Spring 会自动注册一个叫做 `FilterChainProxy` 的安全过滤器链，包含十几个内置的安全过滤器，这些过滤器，就是我们熟知的那些过滤器。

> [!NOTE] 注意事项
> 1. 即使你不显式添加这个注解，只要引入了 `spring-boot-starter-security`，Spring Boot 就会自动注册默认的安全过滤器链。但如果你需要自定义安全配置类（例如我们自己的 `SecurityConfiguration`），就必须显式启用它，以确保你的配置生效

---


### 3.4. 配置 CORS 跨域资源共享

CORS 是指：出于安全考虑，浏览器默认会阻止网页从一个源（如 `http://a.com`）向另一个源（如 `http://b.com/api/data`）发起请求，这种“同源策略”能够有效防范诸如 CSRF、XSS 等跨站攻击。但在一些场景下我们确实需要跨源访问，例如当前端和后端分离部署在不同的源时，仍然需要让前端能够访问后端接口。

这时，如果后端的 `http://b.com/api/data` 启用了 CORS 跨域资源共享机制，那么浏览器就允许网页从 `http://a.com` 向这个接口发起请求并成功获取数据。

需要注意的是，CORS 跨域资源共享与域名解析得到的 IP 地址无关，而是基于“源”（Origin）来判断。那么什么是“源”呢？“源”由协议（scheme）+ 域名（host）+ 端口号（port）三部分组成，不包括 URL 后面的路径和参数。例如，`https://example.com/page` 与 `https://example.com/another` 属于同源，但 `http://example.com/page` 与 `https://example.com/page` 就不同源。即便它们甚至解析到相同的 IP，只要协议、域名或端口任一部分不完全相同，就被视为不同源。

需要注意的是，**Spring Security 默认是关闭 CORS 的**，如果需要使用跨域资源共享，必须手动开启并配置相应的跨域策略。

除此之外，CORS 只能解决跨域访问的问题，用户权限不足时依然会被拒绝访问。如果用户没有登录，也就是证明没有开启 “记住我” 功能，那么每个请求都会以匿名身份（匿名 Authentication）访问，对于需要认证的资源，尤其是权限要求较高的资源，这些请求会被 Spring Security 拒绝。换句话说，即使跨域访问成功，没有足够权限也无法访问受保护资源。

通常情况下，我们希望用户先登录，这样就证明启用了 “记住我” 功能，当访问后端的时候，就会携带凭证，自动完成认证逻辑，Spring Security 就能自动识别用户权限。但需要注意的是，如果使用基于 HttpSession 的认证，由于后端服务器可能是集群架构，用户需要在多台服务器上重复登录，带来管理上的复杂性。

因此，结合 CORS 跨域和权限控制，通常推荐使用 JWT（JSON Web Token）认证机制代替 HttpSession，因为 JWT 是无状态的，适合分布式环境，具体实践包括：
1. 对公共资源，允许跨域访问，且在 Spring Security 中使用 `permitAll` 放行，无需登录即可访问；
2. 对私密资源，要求用户先登录，获取 JWT 凭证，前端在跨域请求时携带该 JWT，后端通过 Spring Security 进行权限校验。对于更敏感的操作，还要启用 CSRF 防护，要求前端携带 CSRF Token；
3. 登录认证基于 JWT，避免使用 HttpSession，因为 HttpSession 依赖单台服务器，难以应对集群环境，JWT 则是通用且无状态的认证方案
4. 需要注意的是，如果只是我们自己的项目，仅供内部使用，或者是一些功能较简单的小型项目，后端生成令牌给前端，前端携带令牌访问后端接口，这种情况下我们完全可以自定义过滤器、自行处理令牌校验等逻辑，前后端达成一致即可，问题不大。但如果目标是打造一个对外公开、安全、合规的系统，那就必须遵循 OAuth 的标准化流程和规范。详见笔记：OAuth 协议
```
// 启用 CORS 跨域资源共享，并配置其跨域规则
@Bean
public CorsConfigurationSource corsConfigurationSource() {

	CorsConfiguration configuration = new CorsConfiguration();
	configuration.setAllowedOrigins(List.of("http://frontend.example.com", "http://localhost:3000"));
	configuration.setAllowedOriginPatterns(List.of("http://*.example.com"));
	configuration.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
	configuration.setAllowedHeaders(List.of("*"));
	configuration.setAllowCredentials(true);

	UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
	Map<String, CorsConfiguration> configMap = new HashMap<>();
	configMap.put("/api/**", configuration);
	configMap.put("/admin/**", configuration);
	source.setCorsConfigurations(configMap);

	return source;
}
```

> [!NOTE] 注意事项
> 1. 之前开启 CORS 跨域资源共享，需要写 `http.cors` 开启 CORS，并且配置 `CorsConfigurationSource` 跨域规则，现在只需要配置 `CorsConfigurationSource` 既开启了 CORS，又配置了跨域规则

<font color="#92d050">1. configuration.setAllowedOrigins</font>
用于配置**哪些源可以跨域访问我们**，不支持通配符，需精确指定 “源”（Origin），即需精确指定协议 + 域名 + 端口，注意不能包含后续的 URL 路径部分（因为不符合 Origin 的写法）
```
configuration.setAllowedOrigins(List.of("http://frontend.example.com:8888", "http://frontend.example.com", "*"));
```

> [!NOTE] 注意事项
> 1. 未指定端口，将使用协议的默认端口（http 默认是 80 端口，https 默认是 443 端口）
> 2. 不能与 `configuration.setAllowedOriginPatterns` 同时使用


<font color="#92d050">2. configuration.setAllowedOriginPatterns</font>
与 `configuration.setAllowedOrigins` 类似，但能支持通配符（`?`、`*`、`**`）
```
configuration.setAllowedOriginPatterns(List.of("http://*.example.com"));
```

> [!NOTE] 注意事项
> 1. 由于是源匹配，不能包含后续的 URL 路径部分，所以 `**` 是不适用于这种情况的
> 2. 补充一下三种通配符常见用法：
```
// 1. ?，匹配单个字符
configuration.setAllowedOriginPatterns(List.of("http://a?.example.com"));


// 2. *，匹配单个路径段的多个字符
configuration.setAllowedOriginPatterns(List.of("http://*.example.com"));


// 3. **，匹配任意层级的路径段（不适用于 Origin 匹配，适用于 URL 匹配）
source.registerCorsConfiguration("/api/**", config);
```


<font color="#92d050">3. configuration.setAllowedMethods</font>
指定允许哪些 HTTP 方法能够执行跨域请求，例如：
```
configuration.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS", "*"));
```

> [!NOTE] 注意事项
> 1. 若要允许所有 HTTP 方法都能够执行跨域请求，可以写成：
```
configuration.setAllowedMethods(List.of("*"));
```


<font color="#92d050">4. configuration.setAllowedHeaders</font>
配置前端在跨域请求中**允许携带的请求头**，如果前端发送的请求头未被我们明确允许，浏览器会直接拦截请求，根本不会发送到服务器，例如：
```
configuration.setAllowedHeaders(List.of("Content-Type", "Authorization", "*"));
```

> [!NOTE] 注意事项
> 1. 若要允许前端携带任意请求头，可以写成：
```
configuration.setAllowedHeaders(List.of("*"));
```


<font color="#92d050">5. configuration.setAllowCredentials  </font>
在讲解这个之前，我们先补充一个前置知识：当我们是同源访问时，前端访问后端，一般是使用 AJAX 进行请求的，然后浏览器会自动携带 Cookie 发送到后端，这点大家都很熟悉。但是在跨域访问时，出于安全考虑，浏览器默认不会携带 Cookie。如果想让浏览器携带 Cookie，前端必须在 AJAX 请求中添加 `credentials: 'include'`，这样 Cookie 才会被发送。
```
// 前端请求示例：
fetch('http://api.example.com/user/info', {
  method: 'GET',
  headers: {
    'Authorization': 'Bearer xxxxxx'
  },
  credentials: 'include'
});
```

但并不是说你前端愿意携带 Cookie，我后端就理所当然地愿意接收。因为后端并不知道你发送过来的 Cookie 是出于正常用途，还是存在安全隐患。也就是说，我们的 `configuration.setAllowCredentials` 默认值是 `false`，即使前端执意携带 Cookie 发起请求，接口本身执行并正常返回了响应，浏览器也会因为安全策略自动屏蔽返回结果，导致前端无法正常获取数据。

如果将其设置为 `true`，意味着前后端达成共识，允许使用 Cookie，这样跨域请求才能正常携带 Cookie 并获得返回结果：
```
configuration.setAllowCredentials(true);
```

> [!NOTE] 注意事项
> 1. 当设置 `configuration.setAllowCredentials(true);` 时，就不能再允许所有来源访问，即不能使用 `configuration.setAllowedOrigins(List.of("*"));`，需要明确指定允许的域名（Spring Security 为了我们的安全真是煞费苦心了）
> 2. 如果不涉及到 Cookie，其实不用去做这样一系列操作
> 3. 跨域请求不携带 Cookie，指的是通过 AJAX 发起的请求；但如果是通过 HTML 元素导航发起的请求，比如 `<img src="https://bank.com/transfer?to=attacker&amount=10000" />`，无论是否跨域，浏览器都会自动携带对应 Cookie。这也是为什么我们需要 CSRF 防护的原因。


<font color="#92d050">6. source.setCorsConfigurations</font>
指定我们配置的跨域规则作用于那些路径
```
configMap.put("/api/**", configuration);
```

> [!NOTE] 注意事项
> 1. 全部路径我们可以写成：
```
configMap.put("/**", configuration);
```

----


### 3.5. 配置 AuthenticationManager

在 `WebSecurityConfigurerAdapter` 时代，`AuthenticationManager` 是由 Spring 官方默认注册为 Bean 的，因此我们可以直接注入使用。

但从 Spring Boot 3 开始，Spring 的设计理念发生了变化：“最少暴露、最少干预”，也就是说框架不再自动为你暴露未显式声明的组件，目的是减少默认暴露导致的 Bean 冲突或安全隐患。

也就是说，Spring 将配置的主动权交还给开发者，所有需要使用的组件开发者都必须通过 `@Bean` 显式声明，而不是官方为你偷偷注入。即使 Spring Security 已经内部实现了默认的 `AuthenticationManager`，但也不会自动将其注册为 Bean。例如它在源码中是这样定义的：
```
public class AuthenticationConfiguration {  
  
    private AuthenticationManager getAuthenticationManagerBean() {  
        return (AuthenticationManager)this.lazyBean(AuthenticationManager.class);  
    }  
    
}
```

如果开发者想使用 `AuthenticationManager`，就需要自己显式地从官方配置中获取其默认实现，或者直接自定义一个。通常我们使用的 `AuthenticationManager` 实际上就是官方默认实现的那个，除非你有特殊需求需要进行自定义配置，因此我们可以这样手动声明：
```
@Bean  
public AuthenticationManager authenticationManager(AuthenticationConfiguration authenticationConfiguration) throws Exception {  
    return authenticationConfiguration.getAuthenticationManager(); // 从 Spring Security 配置中获取其默认的 AuthenticationManager 实例
}
```

> [!NOTE] 注意事项
> 1. 如果我们是使用 `AuthenticationManager` 进行认证，它会自动将用户发送来的用户名和密码，与我们的 `CustomerUserDetailsImpl` 中返回的用户名和密码进行比对，这是我们已知的逻辑。那你可能会有疑问：它在比对前，肯定需要先用密码加密器对用户发送来的明文密码进行加密，然后再比对吧？可我并没有做任何相关配置，`AuthenticationManager` 怎么知道该使用哪个加密器？
> 2. 其实，只要你注册了一个类型为 `PasswordEncoder` 的 接口 Bean，这个 接口 Bean 有一个具体实现，`AuthenticationManager` 就会知道使用这个 `PasswordEncoder` Bean 与其具体实现，对密码进行加密，**无需我们手动配置**。
> 3. 同样的，只要你注册了一个类型为 `UserDetailsService` 的 Bean（接口 Bean），这个 接口 Bean 有一个具体的实现，`AuthenticationManager` 就会知道使用这个 `UserDetailsService` Bean 与其具体实现，去获取 `CustomerUserDetailsImpl` **无需我们手动配置**。
> 4. 上述，只限于接口 Bean 只有一个具体实现，如果有多个具体实现，那就要我们进行配置了，因为 Spring Security 虽然知道用这个 Bean，但是并不知道使用哪一个具体实现
```
/**
 * ============================================
 * Spring IoC 声明 Bean 的常用方式 1
 * --------------------------------------------
 * 概念：
 * - 同时注册了 UserDetailsService 接口类型的 Bean 和 CustomerUserDetailsImplService 实现类类型的 Bean
 * - CustomerUserDetailsImplService 是该接口的一个具体实现类，IoC 容器中可能存在多个这样的实现类 Bean
 * - 我们既可以注入 CustomerUserDetailsImplService 类 Bean，也可以注入 UserDetailsService 接口 Bean
 * - 如果注入的是 UserDetailsService，且只有一个实现类，那么调用接口方法时，实际就是调用该实现类的方法
 * - 如果存在多个实现类，则需要通过配置明确指定使用哪个实现类
 * - 简而言之，此方式支持一个接口 Bean 有多个实现类 Bean，切换实现时只需调整配置，指定使用哪一个实现即可
 * ============================================
 */
@Service
public class CustomerUserDetailsImplService implements UserDetailsService {
	......
}


/**
 * ============================================
 * Spring IoC 声明 Bean 的常用方式 2
 * --------------------------------------------
 * 概念：
 * - 仅注册了 UserDetailsService 类型的 Bean，返回的 CustomerUserDetailsImplService 实例是其具体实现类
 * - 此方式下，一个接口 Bean 只能绑定一个实现类，若要更换实现，需在此方法中直接修改返回的实例。
 * ============================================
 */
@Bean
public UserDetailsService userDetailsService() {
    return new CustomerUserDetailsImplService();
}
```

----


### 3.6. 配置密码加密器

在实际应用中，用户密码绝不能以明文方式存储，因为这会带来极大的安全风险。为确保安全性，我们必须在存储前对密码进行加密处理。基于此，我们面临两大问题：
1. 如何实现密码加密，即如何将明文密码转换为**不可逆**的密文，从而保护用户密码安全
2. 如何校验密码，即用户提交的明文密码如何与存储的密文密码进行对比，从而验证密码的正确性

`PasswordEncoder` 是 Spring Security 提供的一个接口，同时 Spring Security 也提供了该接口的很多实现类（未声明未 Bean），能够解决上述两大问题

配置密码加密器，就是将 `PasswordEncoder` 声明为一个 Bean，并指定返回一个合适的的实现类。这样当我们注入这个 Bean，并调用其接口方法时，实际执行的就是这个实现类的逻辑，这正体现了 Spring IoC 的核心理念：**面向接口编程，运行时注入实现**。不理解这一点，那确实建议回去复习一下 IoC，我们密码加密器的配置，以及如何使用密码加密器解决上述两大问题，常按照这种流程：
<font color="#92d050">1. 配置密码加密器</font>
``` 
/**
 * ============================================
 * 配置密码加密器
 * --------------------------------------------
 * Spring Security 提供的常见实现类：
 * - BCryptPasswordEncoder
 *      - 基于 BCrypt 算法的密码加密工具，不能解密，只能对比，是一种提供较高安全的密码加密方式，能够有效防止暴力破解，适用于大多数应用
 *      - return new BCryptPasswordEncoder();
 * - NoOpPasswordEncoder
 *      - 不对密码进行加密，直接返回原始密码，通常用于开发、测试环境，不建议在生产环境中使用。
 *      - return NoOpPasswordEncoder.getInstance();
 * - Pbkdf2PasswordEncoder
 *      - 基于 PBKDF2 算法的密码加密工具
 * - SCryptPasswordEncoder
 *      - 基于 scrypt 算法的密码加密工具
 * ============================================
 */
@Bean
public PasswordEncoder passwordEncoder() {
	return NoOpPasswordEncoder.getInstance(); // 返回合适的实现类
}
```


<font color="#92d050">2. 使用 PasswordEncoder 实现密码加密</font>
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


<font color="#92d050">3. 校验密码匹配</font>
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

        // 使用 matches 方法验证密码是否匹配
        boolean matches = passwordEncoder.matches(rawPassword, encodedPassword);
        
        return matches ? "Password is valid" : "Invalid password";
    }
}
```

> [!NOTE] 注意事项
> 1. 使用 `matches` 方法时，实际上是**先对 rawPassword 进行加密**，然后再与数据库中的加密密码进行匹配）
> 2. 如果我们是使用 `AuthenticationManager` 进行认证，它会自动将用户发送来的用户名和密码，与我们的 `CustomerUserDetailsImpl` 中返回的用户名和密码进行比对，这是我们已知的逻辑。那你可能会有疑问：它在比对前，肯定需要先用密码加密器对用户发送来的明文密码进行加密，然后再比对吧？可我并没有做任何相关配置，`AuthenticationManager` 怎么知道该使用哪个加密器？
> 3. 其实，只要你注册了一个类型为 `PasswordEncoder` 的 接口 Bean，这个 接口 Bean 有一个具体实现，`AuthenticationManager` 就会知道使用这个 `PasswordEncoder` Bean 与其具体实现，对密码进行加密，**无需我们手动配置**。
> 4. 同样的，只要你注册了一个类型为 `UserDetailsService` 的 Bean（接口 Bean），这个 接口 Bean 有一个具体的实现，`AuthenticationManager` 就会知道使用这个 `UserDetailsService` Bean 与其具体实现，去获取 `CustomerUserDetailsImpl` **无需我们手动配置**。
> 5. 上述，只限于接口 Bean 只有一个具体实现，如果有多个具体实现，那就要我们进行配置了，因为 Spring Security 虽然知道用这个 Bean，但是并不知道使用哪一个具体实现
```
/**
 * ============================================
 * Spring IoC 声明 Bean 的常用方式 1
 * --------------------------------------------
 * 概念：
 * - 同时注册了 UserDetailsService 接口类型的 Bean 和 CustomerUserDetailsImplService 实现类类型的 Bean
 * - CustomerUserDetailsImplService 是该接口的一个具体实现类，IoC 容器中可能存在多个这样的实现类 Bean
 * - 我们既可以注入 CustomerUserDetailsImplService 类 Bean，也可以注入 UserDetailsService 接口 Bean
 * - 如果注入的是 UserDetailsService，且只有一个实现类，那么调用接口方法时，实际就是调用该实现类的方法
 * - 如果存在多个实现类，则需要通过配置明确指定使用哪个实现类
 * - 简而言之，此方式支持一个接口 Bean 有多个实现类 Bean，切换实现时只需调整配置，指定使用哪一个实现即可
 * ============================================
 */
@Service
public class CustomerUserDetailsImplService implements UserDetailsService {
	......
}


/**
 * ============================================
 * Spring IoC 声明 Bean 的常用方式 2
 * --------------------------------------------
 * 概念：
 * - 仅注册了 UserDetailsService 类型的 Bean，返回的 CustomerUserDetailsImplService 实例是其具体实现类
 * - 此方式下，一个接口 Bean 只能绑定一个实现类，若要更换实现，需在此方法中直接修改返回的实例。
 * ============================================
 */
@Bean
public UserDetailsService userDetailsService() {
    return new CustomerUserDetailsImplService();
}
```

---


### 3.7. 配置资源级别的访问控制

```
.authorizeHttpRequests(auth -> {
	auth
		.requestMatchers("/public/**").permitAll()
		.anyRequest().authenticated(); // 其他所有路径均需通过认证
})
```

> [!NOTE] 注意事项
> 1. 同样能支持通配符（`?`、`*`、`**`）
> 2. 如需放行所有请求，可配置为：
```
// 允许匿名 Authentication 身份即可访问所有请求（非 Authentication 用户仍然不能访问）
http.authorizeHttpRequests(auth -> auth.anyRequest().permitAll());
```

<font color="#92d050">1. auth.requestMatchers()</font>
设置**指定资源路径**的访问控制规则，例如：
```
/**
 * ============================================
 * 指定资源路径的访问控制规则
 * --------------------------------------------
 * 常用规则：
 * - permitAll()
 *      - 允许所有用户访问该资源（说是所有用户，其实你至少要有匿名的 Authentication）
 * - denyAll()
 *      - 禁止所有用户访问该资源。
 * - hasRole()
 *      - 要求用户必须具备指定角色才能访问
 *      - 需要注意的是，该方法会自动在角色名前添加 "ROLE_" 前缀，即：hasRole("ADMIN") 代表 ROLE_ADMIN
 * - hasAuthority()
 *      - 要求用户必须具备指定权限才能访问，我们用这个就好
 *      - 需要注意的是，不会自动添加前缀，提供完整权限名称即可
 * - authenticated()
 *      - 要求用户已通过身份验证（即非匿名 Authenticated 而是认证 Authenticated）后才能访问。
 * - access()
 *      - 支持使用 SpEL 表达式，实现更复杂的访问控制逻辑。
 *      - 例如：access("hasRole('ADMIN') and hasIpAddress('192.168.1.0/24')") ，是说此路径仅允许拥有 ADMIN 角色且 IP 地址位于 192.168.1.0/24 网段的用户访问
 * ============================================
 */
.authorizeHttpRequests(auth -> {
	auth
		.requestMatchers("/public/**").permitAll()
		.anyRequest().authenticated(); // 其他所有路径均需通过认证
})
```


<font color="#92d050">2. auth.anyRequest()</font>
除已配置的资源路径外，其余所有资源路径的访问控制规则
```
.authorizeHttpRequests(auth -> {
	auth
		.requestMatchers("/public/**").permitAll()
		.anyRequest().authenticated(); // 其他所有路径均需通过认证
})
```

----


### 3.8. 配置 HttpSession

```
http.sessionManagement(session -> session  
    .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)  
    .maximumSessions(1)
    .maxSessionsPreventsLogin(true)
);
```

> [!NOTE] 注意事项
> 1. Spring Security 本身不负责设置 `HttpSession` 的会话超时时间。会话超时时间由 Servlet 容器或 Spring Boot 配置决定。
> 2. `HttpSession` 存储在服务器端，别误以为存储在用户端了
> 3. 如果不希望使用会话，可以配置为：
```
http.sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS);
```

<font color="#92d050">1. session.sessionCreationPolicy()</font>
用于配置会话的创建策略，常见策略有：
1. SessionCreationPolicy.ALWAYS：
	1. 始终创建会话。每个请求都会新建一个 HttpSession，覆盖之前的会话，并返回新的 JSESSIONID Cookie。
2. SessionCreationPolicy.IF_REQUIRED（默认）：
	1. 按需创建会话。当需要使用 HttpSession 时，Spring Security 会自动创建会话，并返回对应的 JSESSIONID Cookie。
3. SessionCreationPolicy.STATELESS：
	1. 不使用会话（每次请求都不会创建或使用现有的 Session）
	2. 适用于无状态应用场景（如 JWT、OAuth2）
```
.sessionManagement(session -> {  
    session  
        .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED);
})
```


<font color="#92d050">2. session.maximumSessions()</font>
并发会话控制，即限制每个用户在同一时间内的会话数量，也即用户可在多少台设备上同时登录（默认情况下，不限制）
```
session.maximumSessions(1)
.sessionManagement(session -> {  
    session  
        .maximumSessions(1);
})
```


<font color="#92d050">3. maxSessionsPreventsLogin()</font>
是否阻止新会话登录
```
/**
 * ============================================
 * 是否阻止新会话登录
 * --------------------------------------------
 * 选项：
 * - true
 *      - 达到最大会话数后阻止新会话登录，不允许替换旧会话。
 * - false
 *      - 允许新会话登录并替换旧会话
 *      - 此时旧会话将被注销，旧的 JSESSIONID 失效。
 *      - 我们通知用户被挤下线，并重定向到登录页面要求重新登录。
 * ============================================
 */
.sessionManagement(session -> {  
    session  
        .maxSessionsPreventsLogin(true);  
})
```

-----


### 3.9. 配置 SecurityContextPersistenceFilter 过滤器

```
// 禁用 SecurityContextPersistenceFilter 过滤器（强烈不建议禁用）
http.securityContext(securityContext -> securityContext.disable())  
```

> [!NOTE] 注意事项
> 1. `SecurityContextPersistenceFilter` 过滤器会在当前线程中的 `SecurityContext` 发生变化时，将其自动保存到 `HttpSession` 中。但需要注意的是，这一功能默认并未开启，需要我们**显式手动开启**：
```
http.securityContext(securityContext -> securityContext.requireExplicitSave(false));
```

----


### 3.10. 配置 CSRF 攻击防护（配置 CsrfFilter 过滤器）

Spring Security 默认启用跨站请求伪造（CSRF）防护机制，基于 **CSRF Token** 实现安全校验。其工作原理如下：
1. 用户发出请求时，CsrfFilter 过滤器会尝试从我们配置的 CSRF Token 存储位置（配置的 `HttpSessionCsrfTokenRepository`）中加载 CSRF Token，且所有请求都会经过这一尝试。若成功加载，Token 会被放入 `HttpServletRequest` 中；如果未加载到，则会创建一个新的 CSRF Token，并同样放入请求中。
2. 当用户进行登录时，我们可以从 `HttpServletRequest` 中拿到 CSRF Token，并返回给前端
3. 前端需妥善保存该 Token（不得存入 Cookie），在执行修改服务器状态的敏感操作（如 POST、PUT、DELETE、PATCH 等）时，前端必须将该 Token 携带，通常以请求头（默认名称：`X-CSRF-TOKEN`）或请求参数（名称：`_csrf`）的方式携带。
4. 对于启用 CSRF 防护的路径（默认全部启用），当执行修改服务器状态的敏感操作（如 POST、PUT、DELETE、PATCH 等）时，过滤器会检查前端是否在指定的位置（配置的 `.setHeaderName("X-CSRF-TOKEN")` 等位置）携带了 Token。若未携带，则抛出 `MissingCsrfTokenException`；若携带，则会将前端 Token 与 `HttpServletRequest` 中的 Token 进行比对，匹配则继续执行，不匹配则抛出 `InvalidCsrfTokenException`。
5. 即使攻击者能利用用户的 Cookie 或者 JWT 发起请求，由于敏感操作必须携带合法的 CSRF Token，而攻击者无法获取或伪造该 Token，因而能有效防止 CSRF 攻击。

```
@Bean
public CsrfTokenRepository csrfTokenRepository() {

	HttpSessionCsrfTokenRepository repository = new HttpSessionCsrfTokenRepository(); // CSRF Token 生成与加载的位置，这里是生成并保存在 HttpSession 中，并从 HttpSession 中加载

	repository.setHeaderName("X-CSRF-TOKEN"); // 前端可在 X-CSRF-TOKEN 请求头中携带 CSRF Token
	repository.setParameterName("_csrfToken"); // 前端可在 _csrfToken 请求体中携带 CSRF Token
	return repository;
}

http.csrf(csrf -> {  
    csrf  
        .ignoringRequestMatchers("/login") // 忽略对这些路径的 CSRF 保护（默认全部保护）  
        .csrfTokenRepository(csrfTokenRepository()); // 使用我们自定义的 Token 存储库  
});
```

> [!NOTE] 注意事项
> 1. 同样能支持通配符（`?`、`*`、`**`）
> 2. CSRF 攻击的本质是：利用浏览器自动携带 Cookie 的特性，攻击者诱导用户在已登录的网站执行非本意的操作
> 3. 如果我们基于 JWT，通常可以直接禁用 CSRF 防护，因为浏览器是不会自动发送 JWT，必须前端主动将 Token 放到请求头中，因此攻击者无法通过简单的跨站请求强制浏览器带上有效的 JWT
> 4. 开发过程中，可以先禁用 CSRF 防护，到最后再根据 API 隐私进行 CSRF 防护
> 5. 如果想要禁用 CSRF 防护，可以配置为：
```
http.csrf(csrf -> csrf.disable());
```

----


### 3.11. 添加自定义过滤器

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

-----


## 4. Spring Security 核心 API

### 4.1. Authentication

`Authentication` 是 Spring Security 中的核心接口，负责封装用户的身份信息和认证数据，其源码定义如下：
```
public interface Authentication extends Principal, Serializable {

	// 用户拥有的权限集合，比如 ["ROLE_USER", "ROLE_ADMIN"]
    Collection<? extends GrantedAuthority> getAuthorities();

	// 用户的密码，一般设置为 null
    Object getCredentials();

	// 请求附加信息，比如客户端 IP、会话 ID，以及其他自定义信息
    Object getDetails();

	// 用户对象，即 UserDetails 对象，比如 CustomerUserDetailsImpl
    Object getPrincipal();

	// 用户是否已经过认证（true 表示认证的 Authentication，false 表示匿名的 Authentication）
    boolean isAuthenticated();

	// 用于手动设置用户是否认证（只能将 true 设置为 false，不能将 false 设置为 true）
    void setAuthenticated(boolean isAuthenticated) throws IllegalArgumentException;
}
```

该接口有多种实现类，其中最常用的是 `UsernamePasswordAuthenticationToken`，用于表示已认证用户的身份认证信息（`AuthenticationManager` 默认使用的就是这种）；而 `AnonymousAuthenticationToken` 则常用于表示匿名用户的身份（`AnonymousAuthenticationFilter` 默认使用的就是这种）。

我们常说将 `Authentication` 放入线程中，如果不通过 `AuthenticationManager` 来完成认证流程，而是手动构造并设置一个 `Authentication` 对象，通常做法如下：
```
// 1. 加载用户信息  
UserDetails userDetails = userDetailsService.loadUserByUsername(username);


// 2. 创建 Authentication 对象（选择一个合适的实现类，创建实现类的对象）  
UsernamePasswordAuthenticationToken auth =  new UsernamePasswordAuthenticationToken( userDetails, null, userDetails.getAuthorities() );  


// 3. 设置请求附加信息
auth.setDetails(自定义类的对象);  


// 4. 将认证信息存入本线程的安全上下文  
SecurityContextHolder.getContext().setAuthentication(auth);
```

> [!NOTE] 注意事项
> 1. 使用这种构造方式创建的 `Authentication` 对象，其 `isAuthenticated()` 默认为 `true`，因为构造函数中传入了权限（`authorities`），Spring Security 会自动将其标记为已认证。如果我们未传入权限，则默认为 `false`。
> 2. Spring Security **不允许**我们手动将 `isAuthenticated` 从 `false` 设置为 `true`。若希望构造一个 `isAuthenticated` 为 `true` 的 `Authentication` 对象，必须使用带权限参数的构造方法重新创建对象
```
// isAuthenticated 为 false 的构造方法
UsernamePasswordAuthenticationToken auth =  new UsernamePasswordAuthenticationToken( userDetails, null );  


// isAuthenticated 为 true 的构造方法
UsernamePasswordAuthenticationToken auth =  new UsernamePasswordAuthenticationToken( userDetails, null, userDetails.getAuthorities() );  
```

----


# 二、实操

## 1. 基本使用

### 1.1. 基于 HttpSession 的 Spring Security

#### 1.1.1. 创建 Spring Web 项目，添加相关依赖

创建时：
1. Web：
	1. Spring Web
2. Security：
	1. Spring Security
3. SQL
	1. JDBC API
	2. MyBatis Framework
	3. MySQL Driver

---


#### 1.1.2. 创建用户-角色-权限表

我们一般会创建五个表：`users` 表（用户表）存储所有注册用户的信息，`roles` 表（角色表）定义了系统中存在的各种角色，`user_role` 表（用户-角色关联表）用于建立用户和角色之间的多对多关系，`authorities` 表（权限表）定义了系统中的各种操作权限，`role_authoritie` 表（角色-权限表）用于建立角色和权限之间的多对多关系

<font color="#92d050">1. users 表（用户表）</font>

| 列名                           | 数据类型        | 约束        | 索引   | 默认值 | 示例值                               | 说明                                    |
| ---------------------------- | ----------- | --------- | ---- | --- | --------------------------------- | ------------------------------------- |
| **user_id**                  | int         | 主键约束、自增属性 | 主键索引 | 自增  | 1                                 | 用户唯一标识符（将 username 直接作为主键也是一种常见做法）    |
| **username**                 | varchar(20) | 唯一约束      | 唯一索引 |     | john                              | 用户名                                   |
| **password**                 | varchar(80) |           |      |     | $2a$10$abcdefghijklmnopqrstuvwxyz | 加密后密码，推荐长度设的长一些，以兼容现代加密算法（禁止明文密码直接入库） |
| **is_accountNonExpired**     | tinyint(1)  |           |      | 1   | 1                                 | 账户是否没过期（1 代表没过期）                      |
| **is_accountNonLocked**      | tinyint(1)  |           |      | 1   | 1                                 | 账户是否没锁定                               |
| **is_credentialsNonExpired** | tinyint(1)  |           |      | 1   | 1                                 | 密码是否没过期                               |
| **is_enabled**               | tinyint(1)  |           |      | 1   | 1                                 | 用户是否启用                                |
| **email**                    | VARCHAR(20) | 唯一约束      | 唯一索引 |     | john@example.com                  | 邮箱                                    |
| **phone_number**             | VARCHAR(20) | 唯一约束      | 唯一索引 |     | 13800138000                       | 电话号码                                  |


```
# 1. 创建表
CREATE TABLE IF NOT EXISTS users (
    user_id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(20),
    password VARCHAR(80),
    is_accountNonExpired TINYINT(1) DEFAULT 1,
    is_accountNonLocked TINYINT(1) DEFAULT 1,
    is_credentialsNonExpired TINYINT(1) DEFAULT 1,
    is_enabled TINYINT(1) DEFAULT 1,
    email VARCHAR(20),
    phone_number VARCHAR(20)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

ALTER TABLE users ADD CONSTRAINT unique_username UNIQUE (username);

ALTER TABLE users ADD CONSTRAINT unique_email UNIQUE (email);

ALTER TABLE users ADD CONSTRAINT unique_phone UNIQUE (phone_number);
```

> [!NOTE] 注意事项
> 1. username、password、isAccountNonExpired、isAccountNonLocked、isCredentialsNonExpired、isEnabled、authorities 这七个字段是 `userDetails` 接口的默认属性，一般在数据库表中要全部包含
> 2. email、phone_number 等字段，是我们自己扩展的字段。
> 3. 虽然 camelCase 在 Java 中使用广泛，例如 phoneNumber，但在 SQL 表列名中更建议统一为 snake_case，例如 phone_number


<font color="#92d050">2. roles 表（角色表）</font>

| 列名            | 数据类型        | 约束        | 默认值 | 索引   | 示例值        | 说明                                             |
| ------------- | ----------- | --------- | --- | ---- | ---------- | ---------------------------------------------- |
| **role_id**   | int         | 主键约束、自增属性 | 自增  | 主键索引 | 1          | 角色唯一标识符                                        |
| **role_name** | varchar(20) | 唯一约束      |     | 唯一索引 | ROLE_ADMIN | 角色名称，推荐全大写的格式 (Spring Security 约定以 `ROLE_` 开头) |
```
# 1. 创建表
CREATE TABLE roles (
    role_id INT AUTO_INCREMENT PRIMARY KEY,
    role_name VARCHAR(50) UNIQUE
) COMMENT='角色表';

ALTER TABLE roles ADD CONSTRAINT unique_role_name UNIQUE (role_name);


# 2. 插入数据
INSERT INTO roles (role_name) VALUES 
	('ROLE_ADMIN'),
	('ROLE_USER');
```


<font color="#92d050">3. user_role 表（用户-角色关联表）</font>

| 列名          | 数据类型 | 约束                                                 | 索引     | 默认值 | 示例值 | 说明           |
| ----------- | ---- | -------------------------------------------------- | ------ | --- | --- | ------------ |
| **user_id** | int  | 主键约束（与 role_id 联合主键）<br>外键约束（指向 users 表中的 user_id） | 联合主键索引 |     | 1   | users 表中的 id |
| **role_id** | int  | 主键约束（与 user_id 联合主键）<br>外键约束（指向 roles 表中的 role_id） | 联合主键索引 |     | 1   | roles 表中的 id |

```
# 1. 创建表
CREATE TABLE user_role (
    user_id    INT         NOT NULL,
    role_id    INT         NOT NULL,
    PRIMARY KEY (user_id, role_id),
    FOREIGN KEY (user_id) REFERENCES users (user_id),
    FOREIGN KEY (role_id) REFERENCES roles (role_id)
) ;
```

> [!NOTE] 注意事项
> 1. 两表的关联表，一般是取其两表名的单数形式，以 `_` 进行衔接


<font color="#92d050">4. authorities 表（权限表）</font>

| 列名                 | 数据类型        | 约束        | 索引   | 默认值 | 示例值                     | 说明                                |
| ------------------ | ----------- | --------- | ---- | --- | ----------------------- | --------------------------------- |
| **authority_id**   | int         | 主键约束、自增属性 | 主键索引 | 自增  | 1                       | 权限唯一标识                            |
| **authority_name** | varchar(50) | 唯一约束      | 唯一索引 |     | finance:invoice:approve | 权限名称，推荐全小写的格式（常采用 模块：资源：操作 的命名方式） |

```
# 1. 创建表
CREATE TABLE IF NOT EXISTS authorities (
    authority_id INT AUTO_INCREMENT PRIMARY KEY,
    authority_name VARCHAR(50)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

ALTER TABLE authorities ADD CONSTRAINT unique_authority_name UNIQUE (authority_name);


# 2. 插入数据
INSERT INTO authorities (authority_name) VALUES
    ('test:test:test');
```

> [!NOTE] 注意事项
> 1. 表名叫做 `authorities`，列名叫做 `authority_xxx`


<font color="#92d050">5. role_authority 表（角色-权限关联表）</font>

| 列名               | 数据类型 | 约束                                                             | 索引     | 默认值 | 示例值 | 说明                 |
| ---------------- | ---- | -------------------------------------------------------------- | ------ | --- | --- | ------------------ |
| **role_id**      | int  | 主键约束（与 authoritie_id 联合主键）<br>外键约束（指向 roles 表中的 role_id）       | 联合主键索引 |     | 1   | roles 表中的 id       |
| **authority_id** | int  | 主键约束（与 role_id 联合主键）<br>外键约束（指向 authorities 表中的 authoritie_id） | 联合主键索引 |     | 1   | authorities 表中的 id |

```
# 1. 创建表
CREATE TABLE role_authority (
    role_id INT NOT NULL,
    authority_id INT NOT NULL,
    PRIMARY KEY (role_id, authority_id),
    FOREIGN KEY (role_id) REFERENCES roles(role_id),
    FOREIGN KEY (authority_id) REFERENCES authorities(authority_id)
) ;


# 2. 插入数据
INSERT INTO role_authority (role_id, authority_id) VALUES
    (1, 1);
```

> [!NOTE] 注意事项
> 1. 五张表符合 RBAC 规范，详见下文：RBAC 规范

---


#### 1.1.3. 使用 Spring Data MyBatis 实现查询用户的基本信息和权限

##### 1.1.3.1. 前置步骤

详见笔记：Spring Data MyBatis

----


##### 1.1.3.2. 编写 User Entity 类

User Entity 类位于 `com.example.securitywithhttpsession.entity` 包下
```
public class User {

	// users 表中的数据（用户基本信息）
    private Integer userId;
    private String username;
    private String password;
    private Integer isAccountNonExpired;
    private Integer isAccountNonLocked;
    private Integer isCredentialsNonExpired;
    private Integer isEnabled;
    private String email;
    private String phoneNumber;
    
	// authorities 表中的数据（用户的权限，不要忘记添加这个）
    private List<SimpleGrantedAuthority> authorities;

	// 构造方法
    // getter 方法
	// setter 方法
	// equals 方法
	// hashCode 方法
	// toString 方法
}
```

> [!NOTE] 注意事项
> 1. 与数据库表映射的类，通常称为 Entity 类，也可称为 DO 类或 PO 类，统属 POJO 类，通常只包含 getter、setter 、equals、hashCode、toString 方法及构造方法，不应包含业务逻辑方法
> 2. 这里写 Integer 其实不太好，最好的是写 Boolean，但是也不影响，详见下文：实现 UserDetails 接口
> 3. 使用 MyBatisX 插件生成的 POJO 类默认包含 getter、setter、equals、hashCode、toString 方法，但不包含构造方法。
> 	1. 我们可以手动补全有参和无参构造方法；
> 	2. 同时我么也可以删除自动生成的 equals、hashCode、toString 方法，改为使用 IDEA 生成
> 4. 本 User 类不仅与 Users 表映射，还包含了 authorities 表中的 `authoritie_name` 字段。因此，别忘了添加 `private List<String> authorities;` 及其对应的方法（包括 getter、setter、equals、hashCode、toString 方法以及构造方法）
> 5. 数据库中的表名一般使用复数形式，如 users，而在 Java 中则采用单数形式命名，如 User

![](image-20250630183241212.png)

-----


##### 1.1.3.3. 编写 UserMapper 接口，并定义查询方法

User Entity 类位于 `com.example.securitywithhttpsession.mapper` 包下
```
@Mapper  
public interface UserMapper {  

    User getUserByUserName(@Param("username") String username) ;  
  
}
```

----


##### 1.1.3.4. 编写查询方法对应的 SQL 语句（编写 UserMapper.xml）

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.securitywithhttpsession.mapper.UserMapper">

    <resultMap id="BaseResultMap" type="com.example.securitywithhttpsession.entity.User">
            <id property="userId" column="user_id" jdbcType="INTEGER"/>
            <result property="username" column="username" jdbcType="VARCHAR"/>
            <result property="password" column="password" jdbcType="VARCHAR"/>
            <result property="isAccountnonexpired" column="is_accountNonExpired" jdbcType="TINYINT"/>
            <result property="isAccountnonlocked" column="is_accountNonLocked" jdbcType="TINYINT"/>
            <result property="isCredentialsnonexpired" column="is_credentialsNonExpired" jdbcType="TINYINT"/>
            <result property="isEnabled" column="is_enabled" jdbcType="TINYINT"/>
            <result property="email" column="email" jdbcType="VARCHAR"/>
            <result property="phoneNumber" column="phone_number" jdbcType="VARCHAR"/>
            <!-- 不要忘记添加 authorities 的映射关系 -->
            <collection property="authorities" ofType="org.springframework.security.core.authority.SimpleGrantedAuthority">
                <constructor>
                    <arg column="authority_name" javaType="java.lang.String" />
                </constructor>
            </collection>
    </resultMap>

    <select id="getUserByUserName" resultMap="BaseResultMap" parameterType="String">
        SELECT
            users.user_id,
            users.username,
            users.password,
            users.is_accountNonExpired,
            users.is_accountNonLocked,
            users.is_credentialsNonExpired,
            users.is_enabled,
            users.email,
            users.phone_number,
            authorities.authority_name
        FROM users
                 LEFT JOIN user_role ON users.user_id = user_role.user_id
                 LEFT JOIN role_authority ON user_role.role_id = role_authority.role_id
                 LEFT JOIN authorities ON role_authority.authority_id = authorities.authority_id
        WHERE users.username = #{username}
    </select>

</mapper>
```

> [!NOTE] 注意事项
> 1. `username`、`password`、`is_accountNonExpired`、`is_accountNonLocked`、`is_credentialsNonExpired`、`is_enabled` 以及 `authority_name` 是必须要查询的字段，因为这些字段会被封装到 `CustomerUserDetailsImpl` 中，供 Spring Security 进行认证与授权使用。
> 2. 其他字段是否查询，可以根据实际需求灵活决定。即使你没有将 `email`、`phoneNumber` 等扩展信息封装进 `CustomerUserDetailsImpl`，这个方法依然可以作为一个通用的用户信息查询入口来使用。比如你后续业务中需要查询 `email` ，完全也可以直接调用这个方法。换句话说，这个方法不仅仅是在封装 `CustomerUserDetailsImpl` 时用得上，未来在其他场景中也很可能会复用，所以稍微 “富余” 一些是没问题的。

---


#### 1.1.4. 实现 UserDetails 接口

我们通常会定义一个类 `CustomerUserDetailsImpl` 来实现 Spring Security 提供的 `UserDetails` 接口。那么，这个类到底是做什么用的呢？我们先来看一下 `UserDetails` 接口的源码：
```
public interface UserDetails extends Serializable {  

	// 获取用户权限集合
    Collection<? extends GrantedAuthority> getAuthorities();  
    
    // 获取密码（加密后的密码）
    String getPassword();  
    
	// 获取用户名（如 "admin"）
    String getUsername();  
    
	// 账户是否没过期（true 代表没过期，false 代表过期）
    default boolean isAccountNonExpired() {  
        return true; // 默认返回 true
    }  
    // 账户是否没锁定
    default boolean isAccountNonLocked() {  
        return true;  
    }  
    // 凭证（密码）是否没过期
    default boolean isCredentialsNonExpired() {  
        return true;  
    }  
    // 账户是否启用
    default boolean isEnabled() {  
        return true;  
    }  
}
```

Spring Security 是通过调用 `UserDetails` 接口中定义的方法来获取用户信息的（毕竟 Spring Security 并不了解你项目中具体使用了什么类、字段名是什么，对吧，所以不会直接从我们的 User Entity 中拿数据）。

但我们也注意到，`UserDetails` 毕竟只是一个接口，它的方法并没有默认实现和返回值。也就是说，当 Spring Security 调用这些接口方法时，由于接口本身没有具体实现，自然无法拿到到任何值。这也正是我们为什么需要实现一个 `CustomUserDetailsImpl` 类来实现这个接口的原因。我们必须为接口中的每个方法提供具体实现，确保它们能返回对应的值，这样当 Spring Security 调用这些方法时，才能拿到用户信息。

其实我们也发现了，像 `isAccountNonExpired()`、`isAccountNonLocked()`、`isCredentialsNonExpired()`、`isEnabled()` 这几个方法，Spring Security 是提供了默认值的；但另外三个关键方法（如 `getUsername()`、`getPassword()`、`getAuthorities()`）是没有默认实现的，也就是说我们至少必须实现这三个方法，否则编译会直接报错。
```
public class CustomerUserDetailsImpl implements UserDetails {  

    @Override  
    public Collection<? extends GrantedAuthority> getAuthorities() {  
        return null;  
    }  
  
    @Override  
    public String getPassword() {  
        return null;  
    }  
  
    @Override  
    public String getUsername() {  
        return null;  
    }  
  
    @Override  
    public boolean isAccountNonExpired() {  
        return UserDetails.super.isAccountNonExpired();  
    }  
  
    @Override  
    public boolean isAccountNonLocked() {  
        return UserDetails.super.isAccountNonLocked();  
    }  
  
    @Override  
    public boolean isCredentialsNonExpired() {  
        return UserDetails.super.isCredentialsNonExpired();  
    }  
  
    @Override  
    public boolean isEnabled() {  
        return UserDetails.super.isEnabled();  
    }  
}
```

那我们为什么非得实现 `UserDetails` 呢？你说 Spring Security 知不知道这些值，关我什么事？我自己知道就行了呗。但其实你仔细一想，我们就能发现 Spring Security 这么设计是有它的道理的，核心原因就在于：Spring Security 的整个认证和授权流程，底层都是围绕一个 `Authentication` 对象来构建的，而这个对象的核心信息，其实都来自我们实现的 `CustomerUserDetailsImpl`

我们必须要搞清楚以下几点：
1. Spring Security 是通过 `CustomerUserDetailsImpl` 来构建 `Authentication` 对象的
2. 在登录流程中，通常有两种典型的方式：
	1. 通过 `AuthenticationManager` 进行认证（自定义登录 API，使用 AuthenticationManager 进行认证逻辑）：
		1. Spring Security 会自动去校验用户提交的用户名和密码，是否与 `CustomerUserDetailsImpl` 中提供的用户名和密码相匹配；
		2. 如果匹配成功，Spring Security 就会自动将 `CustomerUserDetailsImpl` 封装为一个 `Authentication` 对象，并存入当前线程；
		3. 校验的时候，用户信息是从 `CustomerUserDetailsImpl` 中读取的，封装为 `Authentication` 对象的时候，其值也是从此读取的。
		4. 需要注意的是：这个过程中 Spring Security 还会自动帮你判断用户是否启用、是否锁定、密码是否过期等等。如果你选择下面的手动认证逻辑，那这些判断就必须你自己来实现。
	2. 自定义认证逻辑（自定义登录 API，不用 `AuthenticationManager`，自己写认证逻辑）：
		1. 你仍然需要自己去校验前端提交的用户名和密码，和 `CustomerUserDetailsImpl` 中提供的数据是否一致；
		2. 验证通过后，你需要手动将 `CustomerUserDetailsImpl` 封装为一个 `Authentication` 对象，并保存到当前线程中；
		3. 其实在校验阶段，用户名和密码放不放到 `CustomerUserDetailsImpl` 中都无所谓，你甚至可以直接从`User Entity` 中拿值来对比。更高级一点，你也可以使用扫码登录、验证码、OAuth2 等方式，不再局限于用户名密码；
		4. 但是，一旦你需要构建 `Authentication`，你最终还是得使用 `CustomUserDetailsImpl`，因为 `Authentication` 中的数据必须要从它那里来。
		5. 所以我们可以得出一个结论：在认证过程中不一定非得依赖 `CustomerUserDetailsImpl`，但在封装 `Authentication` 的过程中，最终数据必须来自这个实现类。
3. 最后就是权限判断的问题。权限判断是根据当前线程中保存的 `Authentication` 对象来进行的，而这个对象里的权限信息，不就是我们 `CustomerUserDetailsImpl` 的 `getAuthorities()` 方法中返回的吗？所以说本质上这些权限数据，也都是从 `CustomerUserDetailsImpl` 里来的

所以，如果我们使用了 Spring Security，那么实现 `CustomerUserDetailsImpl` 就显得尤为关键。那问题来了：我们到底是如何将 `UserMapper` 查询到的 `User` 对象中的信息，封装进 `CustomerUserDetailsImpl` 的？

首先思考下：如果是要把 `User` 对象中的属性同步到 `CustomerUserDetailsImpl` 中，我们一般会想到以下两种方式：
1. 显示赋值方式：
	1. 我们可以先使用 `UserMapper` 查询出一个 `User` 对象，然后 `new` 一个 `CustomerUserDetailsImpl` 对象，接着再逐个地将 `User` 的属性赋值到这个对象中。这种方式直观易懂，但写起来繁琐冗长，尤其当字段较多时，容易出错。
2. 构造方法方式（推荐）：
	1. 在 `CustomerUserDetailsImpl` 中定义一个以 `User` 为参数的构造方法，那么当我们 `new` 它的时候，直接传入一个 `User` 对象，构造函数内部再将 `User` 的各个属性赋值到当前对象中。这样封装更简洁、更优雅，也更利于后续维护

但这里我们需要特别注意一点：`CustomerUserDetailsImpl` 本质上并不是一个普通的属性容器（如果它只是一些简单的属性，那显式赋值和构造方法两种方式都可以使用）。它作为 `UserDetails` 的实现类，其核心职责是通过方法的返回值来向 Spring Security 暴露用户信息的（也就是说，它的用户数据并不是通过公共属性暴露的，而是通过接口方法提供的）

因此，我们在封装时更适合采用构造方法的方式。具体来说，就是在 `CustomerUserDetailsImpl` 中定义一个接收 `User` 对象的构造函数，在构造函数中将该 `User` 对象保存为内部私有字段。然后在 `getUsername()`、`getPassword()`、`getAuthorities()` 等方法中，直接返回这个 `User` 对象中的相应属性值，大致如下：
```
public class CustomerUserDetailsImpl implements UserDetails {

    private final User user;

    public CustomerUserDetailsImpl(User user) {
        this.user = user;
    }

    @Override
    public String getUsername() {
        return user.getUsername();
    }

    @Override
    public String getPassword() {
        return user.getPassword();
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return user.getAuthorities(); 
    }

    @Override
    public boolean isAccountNonExpired() {
        return user.getIsAccountnonexpired();
    }

    @Override
    public boolean isAccountNonLocked() {
        return user.getIsAccountnonlocked();
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return user.getIsCredentialsnonexpired();
    }

    @Override
    public boolean isEnabled() {
        return user.getIsEnabled();
    }
}
```

那我们写好这个类之后，只需要在某个方法中，先通过 `UserMapper` 查询出一个 `User` 对象，然后直接执行 `new CustomerUserDetailsImpl(user)`，就能创建出一个封装了用户信息的 `CustomerUserDetailsImpl` 实例。

其实这个步骤，Spring Security 早已帮我们考虑好了，它已经提供了一个专门的接口：`UserDetailsService`。我们只需要实现这个接口，并重写它的 `loadUserByUsername` 方法，在这个方法中使用 `UserMapper` 查询出对应的 `User`，然后直接 `return new CustomerUserDetailsImpl(user)` 即可。

这样一来，当我们调用这个方法的时候，它就会执行查询用户信息，并返回一个包含完整用户数据的 `CustomerUserDetailsImpl` 对象，供后续构建 `Authentication` 使用。

需要注意的是，`CustomerUserDetailsImpl` 并不属于传统意义上的三层架构（Controller-Service-Repository），严格来说，它应当放置在 `com.example.securitywithhttpsession.entity` 包下，作为用户实体信息的一个安全扩展模型。

此外，Spring Security 提供的 `UserDetails` 接口默认只包含七个核心字段：`username`、`password`、`isAccountNonExpired`、`isAccountNonLocked`、`isCredentialsNonExpired`、`isEnabled` 和 `authorities`。

如果你希望在 `Authentication` 中携带更多的信息（例如 `email` 和 `phoneNumber`），是可以通过扩展 `CustomerUserDetailsImpl` 来实现的。这样做的好处是：你可以直接从 `SecurityContext` 中获取 `Authentication`，再从中获取这些扩展信息，无需再根据 `username` 查询数据库。这种方式在某些业务中能显著简化逻辑，提高效率。

不过需要明确的是，Spring Security 的设计初衷并不是鼓励我们将过多的用户字段直接放进 `Authentication` 对象中。是否将这些字段一并封装，应该根据你的具体业务场景权衡决定，避免过度冗余或信息泄露风险。

CustomerUserDetailsImpl 类在`com.example.securitywithhttpsession.entity` 包下，最终，我们实现的代码就如下所示：
```
public class CustomerUserDetailsImpl implements UserDetails {

    private final User user;

    public CustomerUserDetailsImpl(User user) {
        this.user = user;
    }

    // 必须实现的 3 个方法
    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return user.getAuthorities();
    }

    @Override
    public String getPassword() {
        return user.getPassword();
    }

    @Override
    public String getUsername() {
        return user.getUsername();
    }

    // 可选择实现的 4 个方法
    @Override
    public boolean isAccountNonExpired() {
        return user.getIsAccountnonexpired() != null && user.getIsAccountnonexpired() != 0;
    }

    @Override
    public boolean isAccountNonLocked() {
        return user.getIsAccountnonlocked() != null && user.getIsAccountnonlocked() !=0;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return user.getIsCredentialsnonexpired() != null && user.getIsCredentialsnonexpired() != 0;
    }

    @Override
    public boolean isEnabled() {
        return user.getIsEnabled() != null && user.getIsEnabled() != 0;
    }
    
    // 选择性扩展的字段
    public String getEmail() {
        return user.getEmail();
    }
    
    public String getPhoneNumber() {
        return user.getPhoneNumber();
    }
}
```

> [!NOTE] 注意事项
> 1. 由于我们的 `User` 中字段是 `private Integer isAccountNonExpired;`，而接口方法 `public boolean isAccountNonExpired()` 需要返回 boolean 类型，所以在实现时必须手动判断，如：`return user.getIsAccountNonExpired() != null && user.getIsAccountNonExpired() != 0;`
> 2. 其实没必要这么复杂，主要原因是我们在 `User` 中使用了 Integer 类型。对于 MyBatis，如果你把字段声明成 `private boolean isAccountNonExpired;` 在相同的 SQL 语句下，它会自动把数据库中的 TINYINT(1) 映射成 boolean 类型，这样我们只需直接返回：`return user.isAccountNonExpired();`
> 3. 这样写就不需要额外判断了，而且更简洁，也比较推荐。但既然代码已经写成这样，我们就保持现有实现，下面给你展示这种推荐的写法：
```
// User Entity
public class User {

	// users 表中的数据（用户基本信息）
    private Integer userId;
    private String username;
    private String password;
    private Boolean isAccountNonExpired;
    private Boolean isAccountNonLocked;
    private Boolean isCredentialsNonExpired;
    private Boolean isEnabled;
    private String email;
    private String phoneNumber;
    
	// authorities 表中的数据（用户的权限，不要忘记添加这个）
    private List<SimpleGrantedAuthority> authorities;

    // getter 方法
	// setter 方法
	// equals 方法
	// hashCode 方法
	// toString 方法
}

// CustomerUserDetailsImpl
public class CustomerUserDetailsImpl implements UserDetails {

    private final User user;

    public CustomerUserDetailsImpl(User user) {
        this.user = user;
    }

    // 必须实现的 3 个方法
    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return user.getAuthorities();
    }

    @Override
    public String getPassword() {
        return user.getPassword();
    }

    @Override
    public String getUsername() {
        return user.getUsername();
    }

    // 可选择实现的 4 个方法
    @Override
    public boolean isAccountNonExpired() {
        return user.getIsAccountnonexpired();
    }

    @Override
    public boolean isAccountNonLocked() {
        return user.getIsAccountnonlocked();
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return user.getIsCredentialsnonexpired();
    }

    @Override
    public boolean isEnabled() {
        return user.getIsEnabled();
    }
    
    // 选择性扩展的字段
    public String getEmail() {
        return user.getEmail();
    }
    
    public String getPhoneNumber() {
        return user.getPhoneNumber();
    }
}
```

---


#### 1.1.5. 实现 UserDetailsService 接口

我们一般创建 `CustomerUserDetailsImplService` 类，用于实现这个接口，并重写它的 `loadUserByUsername` 方法，在这个方法中使用 `UserMapper` 查询出对应的 `User`，然后直接 `return new CustomerUserDetailsImpl(user)` 即可。

CustomerUserDetailsImplService 类位于 `com.example.securitywithhttpsession.service` 包下
```
@Service
public class CustomerUserDetailsImplService implements UserDetailsService {

    @Autowired
    private UserMapper userMapper;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userMapper.getUserByUserName(username);
        if (user == null) {
            throw new UsernameNotFoundException("user not found" + username);
        }
        return new CustomerUserDetailsImpl(user);
    }
}
```

> [!NOTE] 注意事项
> 1. 写成 `private final UserMapper userMapper` 的话，就不能使用 `@Autowired` 这种方式进行注入，必须要使用构造注入的方式进行注入

----


#### 1.1.6. 进行 Spring Security 配置

详见上文：Spring Security 配置

-----


#### 1.1.7. 编写 秘钥生成、加密、解密 工具类

这个密钥生成工具类用于创建 AES 对称加密所需的密钥、加密、解密，我们可以在保存数据时（如注册用户时），可以对如手机号、邮箱等敏感信息进行加密，并在需要时解密还原。  

我们常用的 `BCryptPasswordEncoder` 属于单向加密工具，无法还原原文，只能用于对加密结果进行比对验证，而这个工具，主要是加密那些不是特别敏感，但也比较敏感的信息，我们可以进行还原。

EncryptionUtils 类位于 `com.example.securitywithhttpsession.util` 包下，关于如何使用 Java 进行密钥生成、加密、解密，详见笔记：Java 生成密钥与加密解密
```
public class EncryptionUtils {
    
    /**
     * ============================================
     * 1. 生成 Base64 编码后的 AES 密钥方法
     * --------------------------------------------
     * 
     * ============================================
     */
    public static String generateSecretKey() throws NoSuchAlgorithmException {
        // 创建 AES 密钥生成器
        KeyGenerator keyGenerator = KeyGenerator.getInstance("AES");
        // 指定密钥的位数，AES 支持 128、192、256 位
        keyGenerator.init(256);
        // 生成密钥
        SecretKey secretKey = keyGenerator.generateKey();
        // 将密钥编码为 Base64 字符串
        return Base64.getEncoder().encodeToString(secretKey.getEncoded());
    }

    /**
     * ============================================
     * 2. 加密方法
     * --------------------------------------------
     * 传入参数：
     * - String plainText
     *      - 要加密的明文字符串
     * - String base64Key
     *      - Base64 编码后的 AES 密钥字符串
     * ============================================
     */
    public static String encrypt(String plainText, String base64Key) throws Exception {
    
        // 把传入的 Base64 编码密钥转换为字节数组，供 AES 使用
        byte[] keyBytes = Base64.getDecoder().decode(base64Key);
        
        // 用字节数组创建一个 AES 对称密钥对象，供后续加密初始化使用
        SecretKeySpec secretKeySpec = new SecretKeySpec(keyBytes, "AES");
        
        // 获取 Cipher 实例，使用 AES 算法
        Cipher cipher = Cipher.getInstance("AES");
        
        // 初始化加密器，设置为加密模式，并指定使用的密钥
        cipher.init(Cipher.ENCRYPT_MODE, secretKeySpec);
        
        // 执行加密，得到密文字节数组
        byte[] encrypted = cipher.doFinal(plainText.getBytes());
        
        // 把密文字节用 Base64 编码，便于作为字符串返回、存储或传输
        return Base64.getEncoder().encodeToString(encrypted);
    }

    /**
     * ============================================
     * 3. 解密密方法
     * --------------------------------------------
     * 传入参数：
     * - String encryptedText
     *      - 加密后的密文字符串
     * - String base64Key
     *      - Base64 编码后的 AES 密钥字符串
     * ============================================
     */
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


#### 1.1.8. 编写 从线程获取 Authentication、CustomerUserDetailsImpl 工具类

AuthenticationUtils 类在 `com.example.securitywithhttpsession.util` 包下 
```
public class AuthenticationUtils {
    
    // 从本线程中获取 Authentication
    public static Authentication getAuthentication() {
        return SecurityContextHolder.getContext().getAuthentication();
    }
    
    // 从本线程中获取 CustomerUserDetailsImpl
    public static CustomerUserDetailsImpl getCustomerUserDetailsImpl() {
        return (CustomerUserDetailsImpl) SecurityContextHolder.getContext().getAuthentication().getPrincipal();
    }
}
```

---


#### 1.1.9. 实现 注册 API、登录 API、授权 API、测试 API、注销 API

在 `com.example.securitywithhttpsession.controller` 下创建 `AuthController`
```
@RestController
public class AuthController {

    @Autowired
    private AuthenticationManager authenticationManager;

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Autowired
    private UserMapper userMapper;

    @Autowired
    private UserRoleMapper userRoleMapper;

    @Autowired
    private TestService testService;

    // 注册方法
    @PostMapping("/public/signup")
    public String signUp(@RequestBody User user) {
    
        String encodePassword = passwordEncoder.encode(user.getPassword());

        int i = userMapper.insertUser(user.getUsername(), encodePassword, user.getEmail(), user.getPhoneNumber());
        if (i != 1) {
            return "服务器繁忙，请稍后再试";
        }
        return "用户注册成功";
    }

    // 登录方法
    @PostMapping("/public/login")
    public String logIn(@RequestParam("username") String username,
                        @RequestParam("password") String password,
                        HttpServletRequest request,
                        HttpServletResponse response) {
        try {
            // 将 username 和 password 封装成 UsernamePasswordAuthenticationToken
            UsernamePasswordAuthenticationToken token =
                    new UsernamePasswordAuthenticationToken(username, password);

            // 传递给 AuthenticationManager 进行认证
            Authentication auth = authenticationManager.authenticate(token);

            // 将 Authentication 保存到当前线程的 SecurityContext
            SecurityContextHolder.getContext().setAuthentication(auth);

            // 获取当前请求中的 CSRF Token（此 token 是 Spring Security 自动生成并放入 request 中的）
            CsrfToken csrfToken = (CsrfToken) request.getAttribute(CsrfToken.class.getName());

            // 将 token 返回给前端，常见做法是放入响应头，也可以放入响应体
            response.setHeader("X-CSRF-TOKEN", csrfToken.getToken());

            return "登录成功，欢迎用户：" + auth.getName();
        } catch (AuthenticationException e) {
            return "登录失败，用户名或密码错误";
        }
    }

    // 授权方法
    @PostMapping("/grantaccess")
    public String grantAccess(@RequestParam("userid") int userid,
                              @RequestParam("roleid") int roleid) {
        int i = userRoleMapper.insertUserRole(userid, roleid);
        if (i != 1) {
            return "服务器繁忙，请稍后再试";
        }
        return "成功将 " + roleid + " 授权给 " + userid;
    }

    // 测试方法
    @GetMapping("/test")
    public String test() {
        System.out.println("正在执行只有 test:test:test 权限才能执行的 Service 方法");
        System.out.println("现在的 Authentication 信息如下：");
        System.out.println(AuthenticationUtils.getAuthentication());
        String testString = testService.test();
        return testString;
    }

    // 注销方法
    @PostMapping("/public/logOut")
    public ResponseEntity<String> logout(HttpServletRequest request) {
        // 尝试获取 HttpSession，如果没有对应的 Session，就返回 null，不要新建 Session
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
> 2. 为了实现这些 API，我还另外书写了 UserRole.java、UserRoleMapper.java、UserRoleMapper.xml、TestService.java，并补充了 UserMapper.java、UserMapper.xml 详细请下载源码查看：summer/SecurityWithHttpSession

----


### 1.2. 基于 JWT 的 Spring Security

#### 1.2.1. JWT 概述

若是基于 HttpSession 的登录方式，如果后端服务器只有一台，那还算可行，但我们仍需关注诸如 CSRF 等安全风险。然而在现实应用中，后端服务器显然不可能只有一台，通常是多台服务器协同处理请求。在这种场景下，假如用户在登录时被负载均衡到服务器 1，并在该服务器上创建了 HttpSession，该服务器确实保存了用户的 Authentication 信息；但后续请求若被分配到服务器 2，由于该服务器并无用户的 Session 信息，就会导致用户“看似从未登录”，需要重新认证。同理，如果接下来被路由到服务器 3、服务器 4……难道用户每次都要重新登录？显然，在多服务器环境下，单纯依赖 HttpSession 并不可靠。此时，我们是否应当引入一种能够**支持单点登录**（SSO）的机制，来统一管理用户身份认证信息？

而 JWT（JSON Web Token）就能够实现单点登录，其核心价值在于：无需依赖服务器端的会话存储（如 `HttpSession`），便可安全地完成用户身份验证，并携带必要的用户信息进行传递。

JWT 由三部分组成：**Header**、**Payload** 和 **Signature**，中间用英文句点 `.` 分隔，形成 `xxxxx.yyyyy.zzzzz` 的格式，例如：
```
eyJhbGciOiJIUzUxMiJ9.eyJzdWIiOiJhZG1pbiIsImNyZWF0ZWQiOjE1NTY3NzkxMjUzMDksImV4cCI6MTU1NzM4MzkyNX0.d-iki0193X0bBOETf2UN3r3PotNIEAV7mzIxxeI5IxFyzzkOZxS0PGfF_SK6wxCv2K8S0cZjMkv6b5bCqc0VBw
```

<font color="#92d050">1. Header（头部）</font>
Header 用于声明该令牌的类型，这里是 JWT，以及 JWT 中用于生成和验证签名（即 Signature）的加密算法的种类（如 HMAC SHA256 或 RSA 等，作用于签名的算法）

其是一个 JSON 对象，通常长这样：
```
{
  "alg": "HS256",     // 表示签名使用的算法，如 HMAC SHA256
  "typ": "JWT"        // 表示类型是 JWT
}
```

在生成 JWT 时会对其进行 Base64URL 编码，形成：
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
```


<font color="#92d050">2. Payload（负载）</font>
Payload 中一般包含两部分内容：第一部分是 Registered Claims，这是 JWT 推荐的标准字段，具有约定俗成的意义。这些字段虽然是可选的，但非常推荐使用，因为它们有助于后端进行有效的校验，比如判断 Token 是否已过期、是否有效、是否来自合法的客户端等。

| 字段    | 示例                 | 含义                                                          |
| ----- | ------------------ | ----------------------------------------------------------- |
| `iss` | `"auth.myapp.com"` | 表示签发者是谁                                                     |
| `sub` | `"user123"`        | 表示主题，通常用于标识用户身份（如用户 ID、用户名），实际中，可以选择不使用，而是在第二部分自定义字段来表示用户身份 |
| `aud` | `"myapp-client"`   | 表示接收方是谁                                                     |
| `exp` | `1710000000`       | 表示过期时间，要以时间戳的形式                                             |
| `nbf` | `1709900000`       | 表示生效时间，要以时间戳的形式                                             |
| `iat` | `1709890000`       | 表示签发时间，要以时间戳的形式                                             |
| `jti` | `"a-b-c-d-e"`      | 表示 JWT 的唯一标识 ID，用于防止重复                                      |

第二部分是 Private Claims，这是根据业务场景自定义的数据字段，可以根据实际需求灵活添加。前端与后端需要对这些字段的格式和含义达成一致约定。例如：
```
{
  "userId": "123456",
  "username": "alice",
  "role": "admin",
}
```

最终，我们 JWT 的 Payload 写法如下：
```
{
  "userId": "123456",
  "username": "alice",
  "role": "admin",
  "iat": 1710000000,
  "exp": 1710003600
}
```

在生成 JWT 时会对其进行 Base64URL 编码，形成：
```
eyJ1c2VySWQiOiIxMjM0NTYiLCJ1c2VybmFtZSI6ImFsaWNlIiwicm9sZSI6ImFkbWluIiwiaWF0IjoxNzEwMDAwMDAwLCJleHAiOjE3MTAwMDM2MDB9
```


> [!NOTE] 注意事项
> 1. 实际上，JWT 的 Payload 部分包含三类字段。除了我们前面提到的 Registered Claims 和 Private Claims 之外，还有一种是 Public Claims。这类字段也是自定义的，但需要遵循一定的命名规范（如使用 URI 命名空间），主要用于跨组织共享场景，在实际开发中，Public Claims 使用较少。


<font color="#92d050">3. Signature（签名）</font>
Signature 用于防止数据被篡改、伪造，验证数据完整性和来源，它的生成方式大致如下。
```
Signature = HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```

由前两部分的 Base64URL 编码拼接而成的字符串，会结合我们的密钥 `secret` 以及 Header 中指定的签名算法（`alg`）生成签名。

这个密钥是保存在我们的服务器中，当 JWT 被客户端发送回来时，服务器会使用这个密钥和签名算法对前两部分重新计算签名，并将其与 JWT 中附带的签名部分进行比对。如果比对一致，就说明该令牌未被篡改，可信；否则，说明令牌可能已被伪造或篡改，应视为无效。

> [!NOTE] 注意事项
> 1. 对于 JWT 的安全性，我们必须对其进行了解：
> 	1. 由于密钥是保存在服务器中的，只有你掌握，即使攻击者截获了 JWT，也无法伪造一个合法的 Token。因为即便他可以从 JWT 中解析出 Header 和 Payload，但没有你的密钥，就无法生成一个正确的签名。使用其他密钥伪造出来的签名，无法通过我们服务器使用原密钥重新计算的签名比对，因此验证会失败。
> 	2. 虽然 JWT 的 Header 和 Payload 看起来像一串很高端的字符，但实际上它们只是经过了 Base64URL 编码（注意不是 Base64 编码），目的是便于传输，并没有加密。因此任何人都可以解码查看其中的内容。例如在网站 [https://jwt.io/](https://jwt.io/) 上，就可以轻松将一个 JWT 解码并查看 Header 和 Payload 中的信息。
> 	3. 这也提醒我们：JWT 虽然具有防伪能力，能够有效防止前端伪造 Token，但由于其前两部分很容易被解码，因此不能将敏感信息（如密码、邮箱、手机号等）放入 Header 或 Payload 中。一般来说，像过期时间、生效时间、签发时间、用户 ID、用户名等信息已经足够。同时，为防止 Token 在传输过程中被截获，应通过 HTTPS 进行传输，而非明文 HTTP
> 	4. 如果攻击者真的截获了 JWT，在其未过期的有效时间内，是可以冒充用户发起请求的。如果系统只靠 Token 的签名来识别身份，因此无法判断是真用户还是伪装者。因此我们应尽量缩短 JWT 的有效期，例如设置为 15 分钟，以减小安全风险。
> 	5. 除此之外，我们还应配合一系列安全机制，辅助判断用户行为是否异常或存在风险。例如，在执行敏感操作时，可以要求用户进行额外验证，如输入身份证号码、手机验证码或进行人脸识别等，以增强身份校验的可靠性，这类多因子验证手段有助于防止 Token 被盗用后的非法操作，进一步提升系统的整体安全性。
> 2. 前端发送 JWT 的格式一般为：Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6...
> 3. 我们一定要抛弃对 Cookie + HttpSession 的固有偏见，以及对 JWT 的祛魅：
> 	1. 并非使用 Cookie 就不安全，关键在于合理配置 Cookie 属性，如 HttpOnly、Secure、SameSite 等。GitHub 实际上就是依赖 Cookie 来管理登录状态的。如果 Cookie 那么容易被截获，GitHub 官方又怎么会采用这种方案呢
> 	2. 有人认为 Cookie 容易被截获，但实际上，只要信息暴露在前端，无论是放在 LocalStorage 还是 Cookie，都存在被截获的风险，难道你使用 JWT 就没有被截获的风险了吗，要知道 JWT 也是需要保存在前端的，JWT 同样可能被截获，因为它暴露在前端。
> 	3. 那你说：JWT 能保证数据未被篡改。其实 Cookie 也可以做到，比如 Spring Security 的 RememberMe Cookie 就与 JWT 很类似。
> 	4. 实际上，Cookie 在某些方面甚至比 JWT 更灵活，例如在符合 OAuth 标准的授权服务器中，Spring Security 官方就推荐我们使用 RememberMe Cookie 实现登录，如果你非要使用 JWT 实现登录，那 Spring Security 也推荐你把 JWT 放在 Cookie 里，因为 Cookie 的自动携带特性能简化很多流程。
> 	5. 所以我们之所以选择拥抱 JWT，而不再采用 Cookie + HttpSession 的登录方式，其实我们真正抛弃的是 HttpSession。因为 HttpSession 天然依赖服务端状态，不适合后端多服务或分布式系统的场景，仅此而已。而 Cookie 本身依然是非常强大的机制。

---


#### 1.2.2. 创建 Spring Web 项目，添加相关依赖

创建时：
1. Web：
	1. Spring Web
2. Security：
	1. Spring Security
3. SQL
	1. JDBC API
	2. MyBatis Framework
	3. MySQL Driver

创建后：添加 [jjwt-api 依赖](https://mvnrepository.com/artifact/io.jsonwebtoken/jjwt-api)、[jjwt-impl 依赖](https://mvnrepository.com/artifact/io.jsonwebtoken/jjwt-impl)、[jjwt-jackson 依赖](https://mvnrepository.com/artifact/io.jsonwebtoken/jjwt-jackson)
``` 
<dependency>  
    <groupId>io.jsonwebtoken</groupId>  
    <artifactId>jjwt-api</artifactId>  
    <version>0.11.2</version>  
</dependency>  
<dependency>  
    <groupId>io.jsonwebtoken</groupId>  
    <artifactId>jjwt-impl</artifactId>  
    <version>0.11.2</version>  
    <scope>runtime</scope>  
</dependency>  
<dependency>  
    <groupId>io.jsonwebtoken</groupId>  
    <artifactId>jjwt-jackson</artifactId>  
    <version>0.11.2</version>  
    <scope>runtime</scope>  
</dependency>
```

----


#### 1.2.3. 前置步骤

和基于 HttpSession 的 Spring Security 的步骤 1.1.2 ~ 1.1.8 近似一致

----



#### 1.2.4. 编写 JwtResponse DTO 类

JwtResponse 类位于 `com.example.securitywithjwt.dto` 包下
```
public class JwtResponse {

    private String token;

	// 构造方法
    // getter 方法
	// setter 方法
	// equals 方法
	// hashCode 方法
	// toString 方法
}
```

> [!NOTE] 注意事项
> 1. 用于前后端或服务间传输数据的类，通常称为 DTO 类，属于 POJO 类

----


#### 1.2.5. 编写 JWT 生成、提取工具类

JwtUtil 类位于 `com.example.securitywithjwt.util` 包下
```
public class JwtUtil {

    /**
     * ============================================
     * JWT 的加密密钥
     * --------------------------------------------
     * 要求：
     * - 不同加密算法，密钥长度要求不同
     *      - HMAC-SHA256：32 字节（256 bit）
     *      - HMAC-SHA384：48 字节（384 bit）
     *      - HMAC-SHA512：64 字节（512 bit）
     * - 我们使用的时 HMAC-SHA512 加密算法，即密钥长度至少 64 字节。在 UTF-8 编码下，一个英文占一个字节，也就是说字符串长度至少为 64
     * - JJWT 关注的是密钥中包含的真实位数，而不是你包装后的长度。简单来说，你可以将一个 46 字符的字符串进行 Base64 编码，编码后的字节数恰好是 64 字节
     * - 在内存中占用 512 bit，看似长度变大了，但 JJWT 仍会报错，因为这只是包装变大了，实际只有 376 位的“真实信息量”
     *
     * 注意事项：
     * - JWT 签名实际上需要的是一个 二进制的密钥对象，而不是简单的字符串
     * - 本方法是根据一个字节数组创建一个 符合 HMAC SHA 算法要求的密钥对象（字节数组必须满足 512 bit，即 64 字节，也就是说字符串长度至少为 64，这样获取到的字符串字节数组就满足 512 bit）
     * ============================================
     */
    private static final SecretKey SECRET_KEY = Keys.hmacShaKeyFor("abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789++".getBytes(StandardCharsets.UTF_8));

    private static final long EXPIRATION_TIME = 1000 * 60 * 60 * 24; // Token过期时间，以毫秒为单位，这里是 1 天

    private static final String TOKEN_PREFIX = "Bearer "; // JWT在HTTP请求中的标准前缀，前端发送 JWT 的格式一般为：Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6...

    private static final String HEADER_STRING = "Authorization"; // HTTP请求头中存放 JWT 的字段名

    // 生成 JWT
    public static String generateToken(CustomerUserDetailsImpl customerUserDetails) {
        return Jwts.builder()
                .signWith(SignatureAlgorithm.HS512, SECRET_KEY)// Header 部分，使用 HS512 算法
                .setIssuedAt(new Date()) // Payload 部分，设置签发时间（Registered Claims 部分），JWT 库会自动把它转换成秒级时间戳
                .setExpiration(new Date(System.currentTimeMillis() + EXPIRATION_TIME)) // Payload 部分，设置过期时间（Registered Claims 部分），JWT 库会自动把它转换成秒级时间戳
                .claim("username", customerUserDetails.getUsername()) // Payload 部分，设置用户的用户名（Private Claims 部分）
                .compact(); // 组合 JWT
    }

    // 从 JWT 中提取用户名
    public static String getUsernameFromToken(String token) {
        try {
            Claims claims = Jwts.parserBuilder()
                    .setSigningKey(SECRET_KEY)
                    .build()
                    .parseClaimsJws(token)  // 自动检查签名 & 过期时间
                    .getBody();

            return claims.get("username", String.class);

        } catch (io.jsonwebtoken.ExpiredJwtException e) {
            System.out.println("Token 已过期");
        } catch (io.jsonwebtoken.SignatureException e) {
            System.out.println("Token 签名不合法，可能被伪造");
        } catch (Exception e) {
            System.out.println("Token 解析失败：" + e.getMessage());
        }
        return null;
    }

    // 从 Authorization头中提取 JWT
    public static String extractBearerToken(String authorizationHeader) {
        if (authorizationHeader != null && authorizationHeader.startsWith(TOKEN_PREFIX)) {
            return authorizationHeader.substring(7);  // 去除"Bearer "前缀
        }
        return null;
    }
}
```

----

#### 1.2.6. 编写 JwtRequestFilter 过滤器类

该自定义过滤器负责：每次请求到达时，首先从请求头中的 `Authorization` 字段提取出 JWT，并校验其签名是否合法以及是否过期，校验通过后，从 JWT 中解析出 `username`。

随后，通过 `username` 调用 `CustomerUserDetailsImplService` 加载完整的 `CustomerUserDetailsImpl` 用户信息，并将其封装为 `Authentication` 对象，最终将该对象设置到当前线程的安全上下文中。

如果不能从请求头中提取出 JWT，则直接跳过该过滤器。

JwtRequestFilter 类位于 `com.example.securitywithjwt.filter` 包下
```
@Component
public class JwtRequestFilter extends OncePerRequestFilter {

    @Autowired
    private UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        // 从请求头中提取 JWT
        String token = JwtUtil.extractBearerToken(request.getHeader("Authorization"));

        // 如果 token 存在，则进行认证
        if (token != null) {
            // 从 token 中提取 username，验证已在 getUsernameFromToken 中完成
            String username = JwtUtil.getUsernameFromToken(token);
            // 如果 username 有效且当前上下文未认证
            if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
                try {
                    // 加载用户信息
                    UserDetails userDetails = userDetailsService.loadUserByUsername(username);
                    // 创建认证对象
                    UsernamePasswordAuthenticationToken auth =
                            new UsernamePasswordAuthenticationToken(
                                    userDetails, null, userDetails.getAuthorities()
                            );
                    // 设置请求详情
                    auth.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                    // 将认证信息存入安全上下文
                    SecurityContextHolder.getContext().setAuthentication(auth);
                } catch (Exception e) {
                    // 加载用户信息失败时记录日志，不影响后续流程
                    logger.error("Failed to load user details for username: " + username, e);
                }
            }
        }

        // 继续执行过滤器链
        filterChain.doFilter(request, response);
    }
}
```

----


#### 1.2.7. 添加 JwtRequestFilter 过滤器到 Security 过滤器链

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

----


#### 1.2.8. 实现 注册 API、登录 API、授权 API、测试 API、注销 API

AuthController 类位于 `com.example.securitywithjwt.controller` 包下
```
@RestController
public class AuthController {

    @Autowired
    private AuthenticationManager authenticationManager;

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Autowired
    private UserMapper userMapper;

    @Autowired
    private UserRoleMapper userRoleMapper;

    @Autowired
    private TestService testService;

    // 注册方法
    @PostMapping("/public/signup")
    public String signUp(@RequestBody User user) {
        String encodePassword = passwordEncoder.encode(user.getPassword());

        int i = userMapper.insertUser(user.getUsername(), encodePassword, user.getEmail(), user.getPhoneNumber());
        if (i != 1) {
            return "服务器繁忙，请稍后再试";
        }
        return "用户注册成功";
    }

    // 登录方法
    @PostMapping("/public/login")
    public String logIn(@RequestParam("username") String username,
                        @RequestParam("password") String password,
                        HttpServletRequest request,
                        HttpServletResponse response) {
        try {
            // 将 username 和 password 封装成 UsernamePasswordAuthenticationToken
            UsernamePasswordAuthenticationToken token =
                    new UsernamePasswordAuthenticationToken(username, password);

            // 传递给 AuthenticationManager 进行认证
            Authentication auth = authenticationManager.authenticate(token);

            // 将 Authentication 保存到当前线程的 SecurityContext
            SecurityContextHolder.getContext().setAuthentication(auth);

            // 获取当前请求中的 CSRF Token（此 token 是 Spring Security 自动生成并放入 request 中的）
            CsrfToken csrfToken = (CsrfToken) request.getAttribute(CsrfToken.class.getName());

            String jwtToken = JwtUtil.generateToken(AuthenticationUtils.getCustomerUserDetailsImpl());

            // 将 token 返回给前端，常见做法是放入响应头，也可以放入响应体
            response.setHeader("X-CSRF-TOKEN", csrfToken.getToken());
            response.setHeader("JWT-TOKEN", jwtToken);

            return "登录成功，欢迎用户：" + auth.getName();
        } catch (AuthenticationException e) {
            return "登录失败，用户名或密码错误";
        }
    }

    // 授权方法
    @PostMapping("/grantaccess")
    public String grantAccess(@RequestParam("userid") int userid,
                              @RequestParam("roleid") int roleid) {
        int i = userRoleMapper.insertUserRole(userid, roleid);
        if (i != 1) {
            return "服务器繁忙，请稍后再试";
        }
        return "成功将 " + roleid + " 授权给 " + userid;
    }

    // 测试方法
    @GetMapping("/test")
    public String test() {
        System.out.println("正在执行只有 test:test:test 权限才能执行的 Service 方法");
        System.out.println("现在的 Authentication 信息如下：");
        System.out.println(AuthenticationUtils.getAuthentication());
        String testString = testService.test();
        return testString;
    }

    // 注销方法
    @PostMapping("/public/logOut")
    public void logout(HttpServletRequest request) {
        // 注销方法我们可以使用黑名单的方式，但是最简单的方式是，前端直接把 JWT 丢弃。
    }
}
```

> [!NOTE] 注意事项
> 1. 配置了密码加密器后，AuthenticationManager 会将用户提交的密码加密并与数据库中查询出的密码进行匹配。如果数据库中仍是明文密码，将无法通过校验，返回“登录失败，用户名或密码错误”。
> 2. 为了实现这些 API，我还另外书写了 UserRole.java、UserRoleMapper.java、UserRoleMapper.xml、TestService.java，并补充了 UserMapper.java、UserMapper.xml 详细请下载源码查看：summer/SecurityWithHttpSession

----


## 2. 业务处理

### 2.1. Spring Security 集成 OAuth2

详见笔记：OAuth 协议

-----








