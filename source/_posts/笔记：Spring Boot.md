---
title: 笔记：Spring Boot
date: 2025-05-02
categories:
  - Java
  - Spring 生态
  - Spring Boot
tags: 
author: 霸天
layout: post
---
![](image-20250709172129814.png)


```
@Root(name = "LifecycleConfiguration")  
@Namespace(reference = "http://s3.amazonaws.com/doc/2006-03-01/")  
public class LifecycleConfiguration {  
  @ElementList(name = "Rule", inline = true)  
  private List<LifecycleRule> rules;  
  
  /** Constructs new lifecycle configuration. */  
  public LifecycleConfiguration(  
      @Nonnull @ElementList(name = "Rule", inline = true) List<LifecycleRule> rules) {  
    this.rules =  
        Collections.unmodifiableList(Objects.requireNonNull(rules, "Rules must not be null"));  
    if (rules.isEmpty()) {  
      throw new IllegalArgumentException("Rules must not be empty");  
    }  
  }  
  
  public List<LifecycleRule> rules() {  
    return rules;  
  }  
}
```

![](image-20250709074128434.png)





![](image-20250708185639252.png)


![](image-20250708182201629.png)



![](image-20250708180840473.png)



![](image-20250707155119821.png)

这个和reponse，没事就传进去玩呗








你在yaml中配置的，究竟去哪里了，spring ioc 也应该有这部分内容：
这是个非常核心又容易忽略的问题。Spring Boot 的“配置”其实不是直接配置到某个类上，而是**通过自动装配机制，把配置值绑定到相关的 Bean 或配置类中去**。你可以从这几个方面理解：

---

### 一、配置的来源

Spring Boot 支持多种配置来源，最常见的是：

- `application.properties` 或 `application.yml`
    
- 命令行参数、环境变量、`@Value` 注解
    
- 外部配置服务（如 Nacos、Consul）
    

这些配置本身只是**键值对**，但它们最终都会通过 **绑定** 或 **条件装配** 的方式作用于 Spring 容器中的某些 Bean。

---

### 二、配置去到了哪里？

1. **绑定到配置类（`@ConfigurationProperties`）**
    

Spring Boot 内置了大量的配置类，比如：

- `org.springframework.boot.autoconfigure.web.ServerProperties`  
    → 用来读取 `server.port`、`server.servlet.context-path` 等
    
- `org.springframework.boot.autoconfigure.jdbc.DataSourceProperties`  
    → 读取 `spring.datasource.url`、`spring.datasource.username`
    

Spring Boot 启动时，会自动扫描这些带有 `@ConfigurationProperties` 的类，把配置文件中的值注入进去。

2. **作用于自动配置类（`@EnableAutoConfiguration`）**
    

Spring Boot 的核心机制是“自动配置”，即：

```java
@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

这个注解背后其实是 `@EnableAutoConfiguration`，它会加载 `META-INF/spring.factories`（或 Boot 3.x 用 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`）中声明的自动配置类。

比如你配置了：

```yaml
spring.mvc.view.prefix: /WEB-INF/views/
```

这个值会绑定到 `WebMvcProperties` 中，再被 `WebMvcAutoConfiguration` 使用，来设置视图解析器。

3. **配置注入到 Bean 中（`@Value`）**
    

如果你写了：

```java
@Value("${my.custom.name}")
private String name;
```

这个是最直接的，Spring 会在容器初始化时，自动把配置文件里的值注入进去。

---

### 三、总结性一句话

> Spring Boot 配置文件中的值，**不是配置到某一个类上**，而是：
> 
> - **绑定到配置类（`@ConfigurationProperties`）**；
>     
> - **影响自动配置类的装配行为**；
>     
> - **可通过 `@Value` 注入到 Bean 中**；
>     
> - **最终决定 Bean 的属性值、是否加载某个 Bean 等行为**。
>     

---

### 如果你想“看见”这些配置是去哪了？

你可以：

#### ✅ 开启配置元信息元数据支持

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>
```

然后去看 IDE 中提示出来的 `@ConfigurationProperties` 类都有哪些字段支持配置。

#### ✅ 打开调试日志

```properties
logging.level.org.springframework.boot.autoconfigure=DEBUG
```

你会看到哪些自动配置类被加载了、哪些条件没满足没加载。

---

如你需要，我也可以帮你找几个典型配置，看它们是怎么绑定到 Spring Bean 的。要不要？










### @Scheduled(fixedRate = 5000)

定时任务，可以用来定时发邮件、定期清理数据库、晚上跑报表等

### 1. 常用注解

#### 1.1. Spring Web 相关注解

##### 1.1.1. @RequestMapping

`@RequestMapping` 用于将 HTTP 请求映射到控制器的方法。开发者可以通过这些注解定义特定的 URL 路径和请求方法，以处理不同类型的请求。这些注解可以应用于类或方法上。

其具有多种衍生注解，包括：
1. ==@GetMapping==：
	1. 用于处理 GET 请求，等同于 `@RequestMapping(method = RequestMethod.GET)`
2. ==@PostMapping==：
	1. 用于处理 POST 请求，等同于 `@RequestMapping(method = RequestMethod.POST)`
3. ==@PutMapping==：
	1. 用于处理 PUT 请求，等同于 `@RequestMapping(method = RequestMethod.PUT)`
4. ==@DeleteMapping==：
	1. 用于处理 DELETE 请求，等同于 `@RequestMapping(method = RequestMethod.DELETE)`
5. ==@PatchMapping==：
	1. 用于处理 PATCH 请求，等同于 `@RequestMapping(method = RequestMethod.PATCH)`
```
# 1. 语法结构
@RequestMapping(value = {"URI 路径1", "URI 路径2"},  
        method = RequestMethod.GET,  
        params = {"请求参数的条件1", "请求参数的条件2"},  
        headers = {"请求头的条件1", "请求头的条件2"},  
        consumes = {"请求体的 MIME 类型1", "请求体的 MIME 类型2"}  
)
"""
1. value：
	1. 固定模式：
		1. /testValue
	2. 通配符模式：
		1. ？：
			1. 匹配任意单个字符，例如：@RequestMapping("/x?z/testValueAnt")
		2. *：
			1. 匹配路径中的零个或多个字符，例如：@GetMapping("/files/*")
		3. **：
			1. 匹配多级路径，例如：@GetMapping("/api/**")
		4. {xx}：
			1. 用于动态路由，例如：@GetMapping("/users/{id}")
2. method：
	1. 指定 HTTP 方法，如 GET、POST、PUT、DELETE 、PATCH等，如果请求类型不匹配，会出现 405 错误
3. params:
	1. 指定请求参数的条件，只有满足条件的请求才会被该方法处理，如果请求参数条件不匹配，会出现 400 错误
	2. 例如：params = {"username", "password=100","!Host"} 表示需要包含 username 参数，不包含 Host 参数，且 password=100
4. headers：
	1. 指定请求头的条件，只有满足条件的请求才会被该方法处理，如果请求头条件不匹配，会出现 400 或 415 错误
	2. 例如：headers={"Referer", "!Host","X-Requested-With=XMLHttpRequest"} 表示需要包含 Referer 头，不包含 Host 头，且 X-Requested-With=XMLHttpRequest
5. consumes：
	1. 指定请求体的 MIME 类型，仅当请求的 Content-Type 符合时才会调用该方法，如果请全体的内容类型不匹配，会出现 415 错误
	2. 例如：consumes = {"application/json", "application/xml"}
"""
```

---


##### 1.1.2. @RestController

`@RestController` = `@Controller` + `@ResponseBody`

---


##### 1.1.3. @ResponseBody

这个注解可以作用在类或方法上，用来告诉 Spring：返回值应直接写入 HTTP 响应体，而不是跳转到视图层（如 JSP 或 HTML 模板）。通俗点说，它让方法的返回结果以 JSON 格式返回，并自动完成数据转换。

我们常用的 `@RestController` 实际上就是 `@Controller + @ResponseBody` 的组合，意味着类中所有方法默认都启用了 `@ResponseBody`，不需要每个方法单独添加。
 ```
@GetMapping("/hello")
@ResponseBody
public String hello() {
    return "Hello, World!";
}
```

---


##### 1.1.4. @RequestBody

`@RequestBody` 是用于将请求体中的 JSON 数据反序列化为 Java 对象的注解。（是默认集成的 Jackson 依赖）

注意：也能将请求体中的 XML 数据反序列化为 Java 对象，但需要添加 `Jackson XML` 依赖
```
# 1. 前端请求
POST /user
Content-Type: application/json

{
  "name": "Alice",
  "age": 28
}


# 2. 后端处理
@PostMapping("/user")
public String createUser(@RequestBody User user) {
    return "User created: " + user.getName();
}
"""
// User Pojo 类
public class User {
    private String name;
    private int age;
}
"""
```

> [!NOTE] 注意事项
> 1. 本注解专注以处理这种场景：
```
// 1. AJAX
fetch('/user', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    username: '张三',
    email: '123@example.com'
  })
})
.then(response => response.text())
.then(data => console.log(data));


// 2. 发送请求
POST /user HTTP/1.1
Content-Type: application/json

{
  "username": "张三",
  "email": "123@example.com"
}


// 3. 后端接收
@PostMapping("/user")
public String createUser(@RequestBody User user) {
    return "收到: 用户名=" + user.getUsername() + ", 邮箱=" + user.getEmail();
}
```

是把所有请求体都映射到一个参数，不是一个参数映射一个懂吗



---


##### 1.1.5. @RequestParam

`@RequestParam` 用于将 HTTP 请求参数（URL 查询参数、表单参数） 绑定到 Controller 方法的参数上：
1. ==URL 查询参数、表单参数==：
	1. 类似于：`GET /user?name=Tom&age=18`
```
# 1. 语法结构
@RequestParam(value = "参数名", required = 是否必须, defaultValue = "默认值")
"""
1. value：
	1. 请求参数的名称，如果方法参数名和请求参数名一致，可以省略
2. reuqired：
	1. 参数是否必须。
	2. 默认是 true，如果缺少这个参数，会抛出异常。
3. defaultValu：
	1. 设置默认值。
	2. 如果请求中没有这个参数，就使用默认值。
"""

# 2. 示例一
@GetMapping("/search")
public String search(@RequestParam String keyword) {
    return "Searching for: " + keyword;
}


# 3. 示例二
@GetMapping("/page")
public String page(@RequestParam("p") int pageNumber) {
    return "Page number: " + pageNumber;
}


# 4. 示例三
@GetMapping("/list")
public String list(@RequestParam(defaultValue = "1") int page,
                   @RequestParam(defaultValue = "10") int size) {
    return "Page: " + page + ", Size: " + size;
}
```

> [!NOTE] 注意事项
> 1. 本注解专注以处理这种场景：
```
// 1. HTML 表单
<form method="post" action="/submit">
  <input type="text" name="username" value="张三" />
  <input type="text" name="email" value="123@example.com" />
  <button type="submit">提交</button>
</form>


// 2. 发送请求
POST /submit HTTP/1.1
Content-Type: application/x-www-form-urlencoded

username=张三&email=123@example.com


// 3. 后端接收
@PostMapping("/submit")
public String submitForm(@RequestParam String username, @RequestParam String email) {
    return "收到: 用户名=" + username + ", 邮箱=" + email;
}
```

---


##### 1.1.6. @RequestPart

`@RequestPart` 专门用于处理 `multipart/form-data` 请求，用于将请求参数绑定到 Controller 方法的参数上，使用方法与 `@RequestParam` 类似。

> [!NOTE] 注意事项
> 1. 本注解专注以处理这种场景：
```
// 1. HTML 表单
<form method="post" action="/upload" enctype="multipart/form-data">
  <input type="text" name="description" value="我的照片" />
  <input type="file" name="file" />
  <button type="submit">上传</button>
</form>


// 2. JS
const formData = new FormData();
formData.append('description', '我的照片');
formData.append('file', document.querySelector('input[type="file"]').files[0]);
formData.append('metadata', JSON.stringify({ title: '我的照片' }));

fetch('/upload', {
  method: 'POST',
  body: formData
})
.then(response => response.text())
.then(data => console.log(data));


// 3. 发送请求
POST /upload HTTP/1.1
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary

------WebKitFormBoundary
Content-Disposition: form-data; name="description"
我的照片
------WebKitFormBoundary
Content-Disposition: form-data; name="file"; filename="photo.jpg"
Content-Type: image/jpeg
[二进制文件内容]
------WebKitFormBoundary
Content-Disposition: form-data; name="metadata"
Content-Type: application/json
{"title": "我的照片"}
------WebKitFormBoundary--


// 4. 后端接收
@PostMapping(value = "/upload", consumes = "multipart/form-data")
public String uploadData(@RequestPart("file") MultipartFile file,
                         @RequestPart("metadata") Metadata metadata,
                         @RequestParam("description") String description) {
    return "文件: " + file.getOriginalFilename() + 
           ", 标题: " + metadata.getTitle() + 
           ", 描述: " + description;
}
```

---

##### 1.1.7. @PathVariable

`@PathVariable` 用于将 URL 路径中的变量 绑定到方法参数上，是拿来处理RESTful 风格的路径参数的：
1. ==URL 路径中的变量==：
	1. 类似于：`@GetMapping("/user/{name}/{age}")` 和 `GET /user/laowang/18`
	2. 这就是 RESTful 风格的路径参数，不像我们传统的 `GET /user?name=Tom&age=18`
```
# 1. 语法结构
@PathVariable("路径变量名")
"""
注意事项：
1. 参数名一致时可以省略路径变量名
"""

# 2. 示例一
@GetMapping("/user/{id}")
public String getUser(@PathVariable("id") Long userId) {
    return "User ID: " + userId;
}


# 3. 示例二
@GetMapping("/book/{isbn}")
public String getBook(@PathVariable String isbn) {
    return "ISBN: " + isbn;
}
```

---


#### Spring IOC 相关注解

##### 组件相关注解

在类上标注 `@Component` 及其衍生注解（如 `@Service`, `@Repository`, `@Controller`），以指示该类是一个 Bean类并声明唯一个Bean 对象。
- `@Component`：标注通用组件
- `@Service`：标注业务逻辑层（service 层）
- `@Repository`：标注数据访问层（dao 层、mapper 层）
- `@Controller`：标注表现层（Web 层），用于**处理 HTTP 请求**
- `@RestController`：`@RestController` = `@Controller` + `@ResponseBody` ，使得每个方法的返回值都直接作为 HTTP 响应体返回。

> [!NOTE] 注意事项
> 1. `@Controller` 与其他 Bean 有所不同。通常我们通过注入 Bean 并手动调用其方法，而 `@Controller` 中的方法是由 HTTP 请求触发执行的，并不需要我们显式调用

---


#### Spring Data Redis 相关注解

##### @EnableCaching

在启动类或配置类上添加` @EnableCaching` 注解，用于启动缓存功能
```
@EnableCaching
@SpringBootApplication
public class MallTinyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MallTinyApplication.class, args);
    }
}
```

---


##### @Cacheable

`@Cacheable` 一般用在查询方法上。如果缓存里有数据，就直接取缓存，不会执行方法；如果缓存没有，就执行方法，并把返回结果放进缓存（有 `condition`、`unless` 不算）
```
# 1. 语法结构
@Cacheable(
    value     = "user",
    key       = "#id",
    condition = "#id > 100",
    unless    = "#result == null"
)
"""
1. value:
	1. 缓存的命名空间（必填）
	2. 和 key 一起组成 Redis 中的键，如：user::123，可以简单理解为是前缀
	3. 如果我们在配置类中配置了前缀，最后可能的结果为：mall:user::123
2. key：
	1. 设置在命名空间中的缓存 key 值，可以使用 SpEL 表达式定义；
3. condition：
	1. 条件符合则缓存。
4. unless：
	1. 条件符合则不缓存；


# 2. 示例一
@Cacheable(
    value     = "user",
    key       = "#id",
    condition = "#id > 100",
    unless    = "#result == null"
)
public User getUserById(Long id) {
    System.out.println("方法执行，id = " + id);
    return userRepository.findById(id).orElse(null);
}
```

注意：虽然正常情况下是按照上述流程执行的，但一旦使用了 `condition` 或 `unless`，这个顺序会被打乱。
==1.只有 condition 的情况==
```
flowchart LR
  A[开始调用] --> B{是否符合 condition}
  B -- 否 --> C[执行方法（跳过读写缓存）]
  C --> D[返回结果]
  B -- 是 --> E{是否缓存命中}
  E -- 是 --> F[返回缓存结果]
  E -- 否 --> G[执行方法]
  G --> H[写入缓存]
  H --> I[返回结果]
```
![](image-20250517211323498.png)


==2.只有 unless 的情况==
```
flowchart LR
  A[开始调用] --> B{是否缓存命中}
  B -- 是 --> C[返回缓存结果]
  B -- 否 --> D[执行方法]
  D --> E{是否符合 unless}
  E -- 是 --> F[返回结果（不写缓存）]
  E -- 否 --> G[写入缓存]
  G --> H[返回结果]
```
![](image-20250517211409902.png)


==3.conditon 和 unless 都有的情况==
```
flowchart LR
  A[开始调用] --> B{是否符合 condition}
  B -- 否 --> C[执行方法（跳过读写缓存）]
  C --> D[返回结果]
  B -- 是 --> E{是否缓存命中}
  E -- 是 --> F[返回缓存结果]
  E -- 否 --> G[执行方法]
  G --> H{是否符合 unless}
  H -- 是 --> I[返回结果]
  H -- 否 --> J[写入缓存]
  J --> I
```
![](image-20250517210734445.png)

---


##### @CachePut

`@CachePut` 一般用在新增方法上，每次执行时都会把返回结果存入缓存（有 `condition`、`unless` 不算），使用方法和 `@Cacheable` 一样

---


##### @CacheEvict

`@CacheEvict` 一般用在更新或删除方法上，每次执行时会清空缓存（有 `condition` 不算），使用方法和 `@Cacheable` 类似，不过，该注解没有 `unless` 属性，只有 `value`、`key` 和 `condition` 三个属性。并且 `condition` 的判断方式相对简单，只要条件满足就执行清空缓存操作，否则不清空。

---



![](image-20250617161457306.png)










# 乱七八糟

### 1. Spring Boot 如何加载配置文件

1. ==Spring Boot 启动后加载配置文件的顺序==
	1. <font color="#00b0f0">读取本地 bootstrap.yml</font>
		1. 仅用来告诉 Spring：去哪里（Consul 地址）、哪个前缀（prefix）、哪种上下文（context）拉取远程配置。
	2. <font color="#00b0f0">从 Consul 拉取远程配置</font>
	    1. 由 bootstrap 上下文发起，拉到的配置被放入父环境（PropertySource 名称类似 `bootstrap` 或 `configService`）。
	3. <font color="#00b0f0">读取本地 application.yml</font>
	    1. 把本地项目配置加载到子环境中。
	4. <font color="#00b0f0">读取命令行参数、系统属性、环境变量</font>
	    1. 最后注入，用于临时调试或生产环境动态调整。
	5. <font color="#00b0f0">注意</font>：
		- 加载顺序 ≠ 覆盖顺序，后加载的并不一定就能覆盖前面的。
2. ==配置文件覆盖优先级（前覆盖后）==：
	1. 命令行参数 / 系统属性 / 环境变量
	2. 远程配置中心（Consul）
	3. 本地 `application.yml`
	4. 本地 `bootstrap.yml`




@GetMapping("/flatmap") 路径推荐小写，可以flatMap，也可以推荐flat-map

@RequestParam

![](image-20250513162343394.png)
分别该怎么拿

```
@Configuration  
public class RabbitCallBackConfig implements RabbitTemplate.ConfirmCallback, RabbitTemplate.ReturnsCallback {  
    @Autowired  
    private RabbitTemplate rabbitTemplate;  
  
    @PostConstruct  
    public void initRabbitTemplate() {  
        rabbitTemplate.setConfirmCallback(this);  
        rabbitTemplate.setReturnsCallback(this);  
    }  
    
    @Override  
    public void confirm(CorrelationData correlationData, boolean ack, String cause) {  
        // 当消息发送到交换机 成功或失败 时，会调用这个方法  
        System.out.println("confirm（）回调函数打印CorrelationData："+correlationData);  
        System.out.println("confirm（）回调函数打印ack："+ack);  
        System.out.println("confirm（）回调函数打印cause："+cause);  
    }  
  
    @Override  
    public void returnedMessage(ReturnedMessage returnedMessage) {  
        // 当消息发送到队列 失败时，才会调用这个方法  
        System.out.println("消息主体: " + new String(returnedMessage.getMessage().getBody()));  
        System.out.println("应答码: " + returnedMessage.getReplyCode());  
        System.out.println("描述: " + returnedMessage.getReplyText());  
        System.out.println("消息使用的交换器 exchange : " + returnedMessage.getExchange());  
        System.out.println("消息使用的路由键 routing : " + returnedMessage.getRoutingKey());  
    }  
}
```


但是如何让这个方法从一开始就进行工作呢？加postConstruct 方法，死去的回忆又开始攻击我了
![](image-20250510204814351.png)
这样就能实现初始化rabbitconfig的时候调用这个方法，对tabbittemplate 进行增强

---









![](image-20250517125404803.png)












