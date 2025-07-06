---
title: 笔记：Java 文档
date: 2025-04-13
categories:
  - Java
  - 程序文档
  - Java 文档
tags: 
author: 霸天
layout: post
---
# 一、理论

## 1. 导图

![](source/_posts/笔记：Java%20文档/Map：Java%20文档.xmind)

----


## 2. Swagger 和 OpenAPI 概述

Swagger 是一种 RESTful API 规范（用于定义 API 的格式）以及一套工具链（称为 Swagger 工具链，用于辅助开发和文档生成）

所谓 RESTful API 规范，是指开发者通过 JSON 或 YAML 文件，**描述** API 的接口路径、请求方式、参数格式、响应结构、认证机制等关键信息。

Swagger 规范正是这样一种定义 API 的规范标准。目前 Swagger 有两个主要版本：Swagger 2.0 和 Swagger 3.0。通常我们将 Swagger 2.0 称为 “Swagger 规范”，而 Swagger 3.0 则被正式命名为 “OpenAPI 规范”。

无论采用的是 Swagger 规范描述 API，还是采用 OpenAPI 规范描述 API，都可以借助 Swagger 工具链（如 Swagger UI、Swagger Codegen、Swagger Editor 等）来进行辅助开发和文档生成
```
openapi: 3.0.0
info:
  title: Sample API
  version: 1.0.0
paths:
  /users:
    get:
      summary: Get a list of users
      responses:
        '200':
          description: A list of users
          content:
            application/json:
              schema:
                type: array
                items:
                  type: object
                  properties:
                    id:
                      type: integer
                    name:
                      type: string
```

----


## 3. Swagger 工具链

### 3.1. Swagger UI

Swagger UI 能根据 API 文档提供一个交互式的 API 文档界面，用户可以直接通过浏览器进行 API 的测试和查看，其使用方法是：

<span style="background:#9254de">1. 引入 springfox-swagger-ui 依赖</span>
引入 [springfox-swagger-ui 依赖](https://mvnrepository.com/artifact/io.springfox/springfox-swagger-ui)
```
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>2.9.2</version>
</dependency>
```


<span style="background:#9254de">2. 输入访问 Swagger UI 的 URL</span>
```
http://localhost:8080/swagger-ui.html
```

> [!NOTE] 注意事项
> 1. Swagger UI 的 URL 要根据我们的配置的地址进行访问

---


### 3.2. Swagger Editor

Swagger Editor 是一个[在线编译器](https://editor.swagger.io/)，用于编写和查看 Swagger 规范文件（Swagger 2.0 或 OpenAPI 3.0 格式）。

---


### 3.3. SpringDoc 配置

#### 3.3.1. 配置模板

<span style="background:#9254de">1. 在 application.yaml 中</span>
```
springdoc:
  api-docs:                                                   # API 文档相关配置
    path: /v3/api-docs                              # 设置 OpenAPI 文档的访问路径
    enabled: true                                      # 是否开启文档接口，即是否能访问 /v3/api-docs
    version: openapi_3_0                          # 指定 OpenAPI 版本，可选有 openapi_3_0, openapi_3_1，默认是 openapi_3_0
    groups:
      enabled: true                                     # 是否启用分组功能，需结合下面的 group-configs 使用

  group-configs:                                           # 分组相关配置（需启用分组功能）
    - group: "controller1"                          # 分组的名称
      packages-to-scan:                             # 分组扫描（仅支持包名、不支持类名）
        - com.example.springdoctoopenapi.controller1
      packages-to-exclude:                        # 分组排除（仅支持包名、不支持类名）
        - com.example.springdoctoopenapi.controller1.xxxxx
    - group: "controller2"
      packages-to-scan:
        - com.example.springdoctoopenapi.controller2
      packages-to-exclude:
        - com.example.springdoctoopenapi.controller2.xxxxx

  swagger-ui:                                                 # Swagger UI 页面相关配置
    path: /swagger-ui.html                         # 设置 Swagger UI 的访问路径
    enabled: true                                        # 是否开启 Swagger UI 界面，即是否能访问 /swagger-ui.html
    url: /v3/api-docs                                   # 告诉 Swagger UI 去加载哪个 OpenAPI 文档
    display-request-duration: true             # 在 Swagger UI 页面中，每个接口请求下方显示耗时（ms）
    tags-sorter: alpha                                 # 控制标签（tags）排序方式，alpha 是按字母顺序排序
    operations-sorter: alpha                      # 控制接口方法（operations）的排序方式，alpha 是按字母顺序排序
    max-displayed-tags: 20                       # Swagger UI 中最多显示多少个 tags（控制器分组），-1 表示不限制
    try-it-out-enabled: true                       # 开启后可直接在页面上发起接口请求，若设为 false，则只能查看接口描述，不能发请求
```

> [!NOTE] 注意事项：
> 1. 精简版本如下：
```
springdoc:
  api-docs:
    path: /v3/api-docs
    enabled: true
    version: openapi_3_0
    groups:
      enabled: true

  group-configs:
    - group: "controller1"
      packages-to-scan:
        - com.example.springdoctoopenapi.controller1
      packages-to-exclude:
        - com.example.springdoctoopenapi.controller1.AdminController
    - group: "controller2"
      packages-to-scan:
        - com.example.springdoctoopenapi.controller2
      packages-to-exclude:
        - com.example.springdoctoopenapi.controller2.SupportController

  swagger-ui:
    path: /swagger-ui.html
    enabled: true
    url: /v3/api-docs
    display-request-duration: true
    tags-sorter: alpha
    operations-sorter: alpha
    max-displayed-tags: 20
    try-it-out-enabled: true
```


<span style="background:#9254de">2. 在配置类中</span>
在配置类中用于配置 API 文档的基本信息
```
@Configuration
@OpenAPIDefinition(                                   // 文档基本信息
    info = @Info(
        title = "My API",
        version = "v1",
        description = "This is the API documentation for my application",
        contact = @Contact(name = "John Doe", email = "john.doe@example.com")
    )
)
public class OpenApiConfiguration {

}
```

---



#### 3.3.2. API 文档相关配置

##### 3.3.2.1. 设置 OpenAPI 文档的访问路径

如果我们配置如下访问路径，就可以通过这个路径获取 JSON 或 YAML 格式的 API 文档：
```
springdoc:
  api-docs:
    path: /v3/api-docs 
```

<span style="background:#9254de">1. 获取 JSON 格式的文档</span>
访问： http://localhost:8080/v3/api-docs.json
![](image-20250706110513245.png)

> [!NOTE] 注意事项
> 1. 如果想下载 JSON 格式的文档，可以在 IDEA 中下载插件 `OpenAPI Specifications`，点击模块的右键 `Export OpenAPI` 导出 JSON 类型的 API 文档

![](source/_posts/笔记：Java%20文档/image-20250413223859767.png)

![](source/_posts/笔记：Java%20文档/image-20250413223920691.png)


<span style="background:#9254de">2. 获取 YAML 格式的文档</span>
访问： http://localhost:8080/v3/api-docs.yaml ，浏览器会自动下载 YAML 格式的文档
![](image-20250706110600089.png)

打开后的内容格式如下：
![](image-20250706110652313.png)

---


#### 3.3.3. 分组相关配置

![](image-20250706120148704.png)

----








> [!NOTE] 注意事项
> 1. 在使用 Spring Security 时，Swagger 文档页面可能会出现空白，这通常是由于其依赖的一些静态资源被 Spring Security 的资源访问控制机制拦截导致的。你可以按下 F12 打开浏览器开发者工具，查看哪些资源被拦截，然后将这些资源路径配置为 `permitAll`。常见被拦截的资源包括：
```
"/swagger-ui/**", "/swagger-resources/**", "webjars/**", "v3/**"
```

![](image-20250706105924291.png)

# 二、实操

## 1. 基本使用

### 1.1. 使用 SpringDoc 生成符合 OpenAPI 规范的文档

#### 1.1.1. 创建 Spring Web 项目

1. Web
	1. Spring Web

---


#### 1.1.2. 添加 SpringDoc 相关依赖

添加 [spring-openapi-stater-webmvc-ui 依赖](https://mvnrepository.com/artifact/org.springdoc/springdoc-openapi-starter-webmvc-ui)
```
<dependency>
	<groupId>org.springdoc</groupId>
	<artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
	<version>2.6.0</version>
</dependency>
```

> [!NOTE] 注意事项
> 1. spring-openapi-stater-webmvc-ui 依赖，会自动添加 Swagger UI 依赖

---


#### 1.1.3. 编写测试代码

##### 1.1.3.1. 在 Controller1 包下

<span style="background:#9254de">1. AdminController.java</span>
```
@RestController
public class AdminController {

    @GetMapping("/admin/dashboard")
    public String getAdminDashboard() {
        return "Welcome to the Admin Dashboard!";
    }

    @GetMapping("/admin/settings")
    public String getAdminSettings() {
        return "Admin settings page.";
    }
}
```


<span style="background:#9254de">2. UserController.java</span>
```
@RestController
public class UserController {

    @GetMapping("/user/profile")
    public String getUserProfile() {
        return "This is the user's profile page.";
    }

    @GetMapping("/user/settings")
    public String getUserSettings() {
        return "User settings page.";
    }
}
```


<span style="background:#9254de">3. OrderController.java</span>
```
@RestController
public class OrderController {

    @GetMapping("/order/history")
    public String getOrderHistory() {
        return "Here are your past orders.";
    }

    @GetMapping("/order/status")
    public String getOrderStatus() {
        return "Order is currently being processed.";
    }
}
```


<span style="background:#9254de">4. ProductController.java</span>
```
@RestController
public class ProductController {

    @GetMapping("/product/list")
    public String getProductList() {
        return "List of available products.";
    }

    @GetMapping("/product/details")
    public String getProductDetails() {
        return "Details of the selected product.";
    }
}
```

---


##### 1.1.3.2. 在 Controller2 包下

<span style="background:#9254de">1. ReportController.java</span>
```
@RestController
public class ReportController {

    @GetMapping("/report/daily")
    public String getDailyReport() {
        return "Daily report data.";
    }

    @GetMapping("/report/monthly")
    public String getMonthlyReport() {
        return "Monthly report data.";
    }
}
```

---


<span style="background:#9254de">2. NotificationController.java</span>
```
@RestController
public class NotificationController {

    @GetMapping("/notification/list")
    public String getNotificationList() {
        return "List of notifications.";
    }

    @GetMapping("/notification/settings")
    public String getNotificationSettings() {
        return "Notification settings page.";
    }
}
```

---


<span style="background:#9254de">3. FeedbackController.java</span>
```
@RestController
public class FeedbackController {

    @GetMapping("/feedback/submit")
    public String submitFeedback() {
        return "Submit your feedback here.";
    }

    @GetMapping("/feedback/status")
    public String getFeedbackStatus() {
        return "Feedback processing status.";
    }
}
```

---


<span style="background:#9254de">4. SupportController.java</span>
```
@RestController
public class SupportController {

    @GetMapping("/support/contact")
    public String getContactInfo() {
        return "Support contact information.";
    }

    @GetMapping("/support/faq")
    public String getFaq() {
        return "Frequently asked questions.";
    }
}
```

---


#### 1.1.4. 进行 SpringDoc 配置

---


#### 1.1.5. 标注 SpringDoc 相关注解

##### 1.1.5.1. Controller 注解

###### 1.1.5.1.1. @Tag

`@Tag` 用于为 Controller 类或方法添加标签，可以用来组织 API 文档中的接口分组。
```
@RestController
@Tag(name = "Admin API",description = "Operation related to admins")
@Tag(name = "User API", description = "Operations related to users")
public class UserController {
    .......
}
```

> [!NOTE] 注意事项：对此处 `@Tag` 的理解
> 1. 可以将 `@Tag` 理解为在 `group-configs` 所定义的大分组内部，用于进一步细分接口的子标签
> 2. 使用 `@Tag` 时可以手动指定标签名；如果未显式设置，系统会为每个类生成一个默认标签，例如 `UserController` 会对应默认标签 `user-controller`
> 3. 每个 `@Tag` 只能指定一个 `name`，但一个类可以标注多个 `@Tag`，用于归类到多个标签下

![](image-20250706121644671.png)

---


###### 1.1.5.1.2. @Operation

`@Operation` 用于标注一个**方法**，并对该方法进行详细说明。
```
@Operation(  
	summary = "Get user by ID",  
	description = "Fetch a user by their unique ID from the database.",  
	tags = {"User","Admin"},  
	parameters = {  
			@Parameter(xxxxxx),  
			@Parameter(xxxxxx)  
	},  
	requestBody = @io.swagger.v3.oas.annotations.parameters.RequestBody(xxxxxxx),  
	responses = {  
			@ApiResponse(xxxxxx),  
			@ApiResponse(xxxxxx)  
	},  
	deprecated = false  
  
)  
@GetMapping("/users/{id}")  
public User getUserById(  
        @PathVariable Long id,  
        @RequestParam String fields  
) {  
    // 查询用户并返回  
    return userService.getUserById(id, fields);  
}
```

<span style="background:#9254de">1. summary</span>
对 API 方法的简短描述

<span style="background:#9254de">2. description</span>
对 API 方法的详细描述


<span style="background:#9254de">3. tags</span>
用于指定 API 方法的标签


<span style="background:#9254de">4. perameters</span>
包含由 `@Parameter` 注解组成的数组，用于**描述接口参数**，而 `@Parameter` 注解的具体属性，可以通过 `Ctrl + 鼠标点击` 查看其来源，或详见下文：`@Parameter` ，其中列出了常用属性


<span style="background:#9254de">5. requestBody</span>
`@RequestBody` 注解的实例，用于描述请求体的内容，通常用于 `POST`、`PUT`、`PATCH` 等请求方法

`@RequestBody` 注解的具体属性，可以通过 `Ctrl + 鼠标点击` 查看其来源，或详见下文：`@RequestBody` ，其中列出了常用属性


<span style="background:#9254de">6. responses</span>
包含由 `@ApiResponse` 注解构成的数组，用于描述接口可能返回的 HTTP 响应，而 `@ApiResponse` 注解的具体属性，可以通过 `Ctrl + 鼠标点击` 查看其来源，或详见下文： `@ApiResponse` ，其中列出了常用属性


<span style="background:#9254de">7. deprecated</span>
用于标记接口是否已弃用。如果设置为 `true`，表示该接口已不推荐使用，通常会在文档中注明，默认为 `false`

> [!NOTE] 注意事项：对此处 `tags` 的理解
> 1. 默认情况下，一个 Controller 类中的所有方法，都会在类标注的 `@Tag` 中，即类级别的标签。而在此处使用 `tags`，单独指定方法自己的标签，那么该方法既会出现在 `@Tag` 标注的标签，又会出现在`tags` 标注的标签

![](image-20250706122621743.png)

---


###### 1.1.5.1.3. @Parameter

```
@Parameter(
    name = "id", 
    description = "User ID", 
    example = "123", 
    schema = @Schema(implementation = User.class),
    in = ParameterIn.PATH,
    required = true, 
    allowEmptyValue = true
)
```

<span style="background:#9254de">1. name  </span>
描述参数的名称，通常应与方法参数名保持一致。


<span style="background:#9254de">2. description  </span>
用于说明参数的含义或用途，便于使用者理解其业务意义。


<span style="background:#9254de">3. example  </span>
为参数提供一个示例值，帮助文档阅读者更直观地了解其典型输入。


<span style="background:#9254de">4. schema  </span>
指向参数所对应的数据模型类（Model 类），用于生成结构化文档。


<span style="background:#9254de">5. in  </span>
用于指定参数的来源位置。常见取值如下：  
1. path：
	1. 表示来自 URL 路径，常用于路径变量（如 `/users/{id}`）  
2. query：
	1. 表示来自 URL 查询字符串，用于 GET 请求中的查询参数  
3. header：
	1. 表示来自 HTTP 请求头部，例如 Authorization 令牌  
4. cookie：
	1. 表示来自 HTTP 请求的 Cookie，例如 JSESSIONID


<span style="background:#9254de">6. required  </span>
用于指定该参数是否为必填项，默认值为 `false`。若设置为 `true`，则参数在请求中必须存在。


<span style="background:#9254de">7. allowEmptyValue  </span>
指示该参数是否允许为空值，默认值为 `false`。

---


###### 1.1.5.1.4. @RequestBody

``` 
@RequestBody(  
        description = "User data",  
        required = true,  
        content = @Content(xxxxx)  
)
```

<span style="background:#9254de">1. description  </span>
描述请求体的内容。


<span style="background:#9254de">2. required  </span>
指定请求体是否为必填项，默认值为 `true`。


<span style="background:#9254de">3. content  </span>
`@Content` 注解的实例，用于描述请求体或响应体的内容类型与结构。而`@Content` 注解的具体属性可以通过 `Ctrl + 鼠标点击` 查看其来源，或详见下文：`@Content` ，其中列出了常用属性。

---


###### 1.1.5.1.5. @ApiResponse

```
@ApiResponse(
	responseCode = "200",
	description = "Successfully retrieved the user",
	content = @Content(xxxxxx)
	)
```

<span style="background:#9254de">1. responseCode  </span>
用于指定响应的 HTTP 状态码，例如 `200`、`400`、`404`、`500` 等。


<span style="background:#9254de">2. description  </span>
为对应的响应状态码提供详细描述，说明响应的含义或返回条件。


<span style="background:#9254de">3. content  </span>
`@Content` 注解的实例，用于描述请求体或响应体的内容类型与结构。而`@Content` 注解的具体属性可以通过 `Ctrl + 鼠标点击` 查看其来源，或详见下文：`@Content` 部分，其中列出了常用属性。

---


###### 1.1.5.1.6. @Content

`@Content` 注解用于描述请求体或响应体的内容类型和结构
```
content = @Content(
	mediaType = "application/json",
	schema = @Schema(implementation = User.class),
	examples = @ExampleObject(value = "{\"name\":\"John Doe\",\"age\":30}"),
)
```

<span style="background:#9254de">1. mediaType  </span>
用于指定响应体的 MIME 类型，例如 `application/json`、`text/plain` 等。


<span style="background:#9254de">2. schema  </span>
指向响应体对应的模型类（Model 类），用于定义返回数据的结构。


<span style="background:#9254de">3. examples  </span>
用于提供响应内容的示例数据，帮助使用者理解接口的典型返回结果

---


##### 1.1.5.2. Model 注解

###### 1.1.5.2.1. @Schema

```
@Schema(name = "User", description = "User object representing a system user")
public class User {  

    @Schema(description = "Unique identifier of the user", example = "1", defaultValue = "100")
    private Long id;  

    @Schema(description = "Username of the user", required = true, example = "john_doe")
    private String username;  

    @Schema(description = "Email address of the user", example = "john@example.com")
    private String email;  

    @Schema(description = "Age of the user", example = "25", defaultValue = "25")
    private Integer age;  

    @Schema(description = "Creation date of the user account", format = "date-time", example = "2024-01-01T00:00:00Z")
    private LocalDateTime createdAt;  

    // Getters and Setters
}
```

> [!NOTE] 注意事项
> 1. 如果你只是定义了一个 `User` 类，但没有在 Swagger 能扫描到的方法中使用它（作为参数或返回值），那么即便你写了 `@Schema` 注解，也不会在 Swagger UI 中显示
> 2. 只要在被 Swagger 扫描的方法中引用了 `User`（比如作为请求参数或响应结果），它就会自动展示出你标注的字段说明，例如：
```
@Operation(summary = "创建用户")
@PostMapping("/user")
public User createUser(@RequestBody User user) {
	// 业务逻辑
    return "hello";
}
```
![](source/_posts/笔记：Java%20文档/image-20250517160013458.png)

<span style="background:#9254de">1. name  </span>
指定模型属性的名称。

<span style="background:#9254de">2. description  </span>
为该属性提供简短描述，说明其用途或含义。

<span style="background:#9254de">3. required  </span>
表示该属性是否为必填项，默认值为 `false`。

<span style="background:#9254de">4. example  </span>
用于提供该字段的示例值，帮助理解其典型输入。

<span style="background:#9254de">5. defaultValue  </span>
指定该字段的默认值，当未显式赋值时将使用此值。

<span style="background:#9254de">6. format  </span>
用于指定字段的格式，特别适用于日期、时间等类型，便于文档工具正确渲染数据格式。  
1. date-time：
	1. 日期 + 时间，符合 ISO 8601 格式
	2. 如 `2024-01-01T12:00:00Z`，适用类型：`LocalDateTime`、`Date`、`ZonedDateTime` 等。  
2. date：
	1. 仅日期，格式为 `yyyy-MM-dd`
	2. 如 `2024-01-01`，适用类型：`LocalDate`。  
3. time：
	1. 仅时间，格式为 `HH:mm:ss`
	2. 如 `12:00:00`，适用类型：`LocalTime`。

---


#### 1.1.6. 访问 Swagger UI 界面

 Swagger UI 界面默认为： http://localhost:8080/swagger-ui/index.html

---

