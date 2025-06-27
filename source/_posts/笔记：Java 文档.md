---
title: 笔记：文档框架
date: 2025-04-13
categories:
  - Java
  - 文档框架
tags: 
author: 霸天
layout: post
---
# 一、理论

### 1. Swagger

#### 1.1. Swagger 概述

Swagger 可以概括为 Swagger 规范（如何定义 API）+ Swagger 工具链（辅助开发和文档生成）。它涵盖了 API 开发的整个生命周期：设计、实现、测试、文档生成，是现代 RESTful API 开发的主流解决方案

---


#### 1.2. Swagger 规范

Swagger 规范是一种描述 RESTful API 的规范，它允许开发者以机器可读的格式（<font color="#ff0000">通常是 JSON 或 YAML</font>）定义 API 的接口、操作、请求、响应、认证方式以及其他相关信息，<font color="#ff0000">这种文件一般叫做 Swagger 文件（OpenAPI 文件、API 文档）。</font>

在 2016 年，Swagger 规范被捐赠给了 **OpenAPI Initiative**，成为一个开放的标准，并更名为 **OpenAPI Specification (OAS)**。OpenAPI 现在已经成为 RESTful API 领域的一个广泛接受的标准。

可以简单地理解为，Swagger 2.0 对应的是 Swagger 规范，而 Swagger 3.0 则对应 OpenAPI 规范。

----


#### 1.3. Swagger 工具链

##### 1.3.1. Swagger UI

Swagger UI 能根据 API 文档提供一个交互式的 API 文档界面，用户可以直接通过浏览器进行 API 的测试和查看，其使用方法是：

==1.引入 springfox-swagger-ui 依赖==
引入 [springfox-swagger-ui 依赖](https://mvnrepository.com/artifact/io.springfox/springfox-swagger-ui)
```
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>2.9.2</version>
</dependency>
```


==2.输入访问 Swagger UI 的 URL==
```
http://localhost:8080/swagger-ui.html
```

---


#### 1.4. Swagger Editor

Swagger Editor 是一个[在线编译器](https://editor.swagger.io/)，用于编写和查看 Swagger 规范文件（Swagger 2.0 或 OpenAPI 3.0 格式）。

---


### 2. API 文档的书写方式

#### 2.1. 手写 API 文档

手写 OpenAPI 文档是指开发者自己手动编写符合 OpenAPI 规范的 YAML 或 JSON 文件，以描述 API 的结构、路径、请求、响应等，例如：
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

---


#### 2.2. 自动生成 API 文档

自动生成 API 文档是指通过**文档框架**（如 Springfox 或 Spring Doc）通过注解自动生成 API 文档，一般我们使用这种方式。

---


### 3. API 文档的下载方式

==1.JSON 类型文档==
在 IDEA 中下载插件 `OpenAPI Specifications`，可以右键 `Export OpenAPI` 导出 JSON 类型的 API 文档
![](source/_posts/笔记：Java%20文档/image-20250413223859767.png)

![](source/_posts/笔记：Java%20文档/image-20250413223920691.png)


==2.YAML 类型文档==

访问 [http://localhost:8080/v3/api-docs.yaml](http://localhost:8080/v3/api-docs.yaml)

---


# 二、实操

### 1. 使用 Springfox 生成 Swagger 2.0 文档

#### 1.1. 引入相关依赖

引入 [springfox-swagger2 依赖](https://mvnrepository.com/artifact/io.springfox/springfox-swagger2)、[springfox-swagger-ui 依赖](https://mvnrepository.com/artifact/io.springfox/springfox-swagger-ui)
```
<dependencies>
    <!-- Swagger 2 -->
    <dependency>
        <groupId>io.springfox</groupId>
        <artifactId>springfox-swagger2</artifactId>
        <version>2.9.2</version>
    </dependency>

    <!-- Swagger UI -->
    <dependency>
        <groupId>io.springfox</groupId>
        <artifactId>springfox-swagger-ui</artifactId>
        <version>2.9.2</version>
    </dependency>
</dependencies>
```

---


#### 1.2. 标注相关注解

由于当前开发主要使用 springdoc，因此关于 Springfox 的讲解暂时暂停，之后不再提供相关信息。

---


### 2. 使用 Springdoc 生成 OpenAPI 文档

#### 2.1. 代码结构目录
```
Spring_Data_MySQL
├── src
│   └── main
│       ├── java
│       │   └── com.example.spring_data_mybatis
│       │       ├── Application.java                          // 项目启动类
│       │       ├── config
|       |       |   └──OpenApiConfig                          // OpenAPI 配置文件
│       │       └── controller
│       │           └── TestController.java                   // TestController 类
│       └── resources
```

---


#### 2.2. 创建 Spring Web 项目

这里采用 IDEA 提供的脚手架创建 Spring Web 项目，分别勾选：
1. ==Web==
	1. Spring Web

---


#### 2.3. 引入相关依赖

引入 [spring-openapi-stater-webmvc-ui 依赖](https://mvnrepository.com/artifact/org.springdoc/springdoc-openapi-starter-webmvc-ui)
```
<dependencies>
    <!-- SpringDoc OpenAPI 依赖，会自动引入 Swagger UI 依赖 -->
    <dependency>
        <groupId>org.springdoc</groupId>
        <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
        <version>2.6.0</version> <!-- 根据需要选择版本 -->
    </dependency>
</dependencies>
```

---


#### 2.4. 准备测试代码

==1.AdminController.java==
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


==2.UserController.java==
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


==3.OrderController.java==
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


==4.ProductController.java==
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


#### 2.5. 进行相关配置

```
# 1. 精简版
springdoc:
  api-docs:
    path: /v3/api-docs
    enabled: true

  swagger-ui:
    path: /swagger-ui.html
    enabled: true
    url: /v3/api-docs
    display-request-duration: true
    tags-sorter: alpha
    operations-sorter: alpha
    max-displayed-tags: 20
    try-it-out-enabled: true

  group-configs:
    - group: "group1"
      packages-to-scan:
        - com.example.demo4.controller
      packages-to-exclude:
        - com.example.demo4.controller.BodyController


# 2. 注解版
springdoc:
  api-docs:                         # API 文档配置
    path: /v3/api-docs              # JSON 类型 API 文档的路径，默认是 /v3/api-docs（加 .yaml 则为 yaml 类型文档）
    enabled: true                   # 启用或禁用 API 文档的生成，默认是 true，即启用文档生成

  swagger-ui:                       # Swagger UI 配置
    path: /swagger-ui.html          # Swagger UI 的访问路径
    enabled: true                   # 启用或禁用 Swagger UI，默认是 true
    url: /v3/api-docs               # Swagger UI 加载 API 文档的 URL，默认是 /v3/api-docs
    display-request-duration: true  # 是否显示请求的持续时间，默认是 false
    tags-sorter: alpha              # 控制 API 标签（tags）列表的排序方式，默认是 alpha
    operations-sorter: alpha        # 控制 API 操作的排序方式，默认是 alpha
    max-displayed-tags: 20          # 限制显示的最大标签数量（-1 是无限制）
    try-it-out-enabled: true        # 启用 Swagger UI 中的试用功能，默认是 true（禁用只能查看 API 文档，不能进行请求测试）

  group-configs:                    # 文档的分组配置
    - group: "group1"               # 定义分组名称
      packages-to-scan:             # 扫描指定包下的所有 API
        - com.example.demo4.controller
      packages-to-exclude:          # 排除指定包中的 API
        - com.example.demo4.controller.BodyController
```

> [!NOTE] 注意事项：对此处 `group-configs` 的理解

![](source/_posts/笔记：Java%20文档/image-20250413212434485.png)

---


#### 2.6. 创建 Springdoc 配置类

```
@Configuration                                        // 配置类
@OpenAPIDefinition(                                   // 文档基本信息
    info = @Info(
        title = "My API",
        version = "v1",
        description = "This is the API documentation for my application",
        contact = @Contact(name = "John Doe", email = "john.doe@example.com")
    )
)
public class OpenApiConfig {
    .......
}
```

---


#### 2.7. 标注相关注解

##### 2.7.1. Controller 注解

###### 2.7.1.1. @Tag

`@Tag` 用于为 Controller 类或方法添加标签，可以用来组织 API 文档中的接口分组。
```
@Tag(name = "Admin API",description = "Operation related to admins")
@Tag(name = "User API", description = "Operations related to users")
@RestController
public class UserController {
    .......
}
"""
1. name：
	1. 用于指定标签的名称，用于分组 API。需要注意的是，name 属性不能设置多个值，如果你需要给接口分配多个标签，你可以使用多个 @Tag 注解
2. description：
	1. 用于为标签提供详细描述，说明该标签对应的 API 的功能或业务模块
"""
```

> [!NOTE] 注意事项：对此处 `@Tag` 的理解
> 1. 可以将 `@Tag` 看作是在 `group-configs` 定义的大分组基础上，通过 `@Tag` 进行更细粒度的子分组
> 2. 每个子分组必须有一个名称。默认情况下，Controller 类在 `Swagger UI` 中会自动生成一个默认标签（例如，`UserController` 默认的标签名为 `user-controller`）。而通过 `@Tag` 注解，你可以手动为该类指定标签
> 3. 注意：Swagger 不仅会扫描 `@Tag` 注解，还会自动识别 Spring Boot 中的组件类（如 `@RestController`）。这也是为什么我前面提到：“Controller 类在 `Swagger UI` 中会自动生成一个默认标签（例如，`UserController` 默认的标签名为 `user-controller`）。

![](source/_posts/笔记：Java%20文档/image-20250413214149924.png)

---


###### 2.7.1.2. @Operation

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
1. ==summary==：
	1. 对 API 方法的简短描述
2. ==description==：
	1. 对 API 方法的详细描述
3. ==tags==：
	1. 用于指定 API 方法的标签
4. ==perameters==：
	1. 包含由 `@Parameter` 注解组成的数组，用于**描述接口参数**
	2. `@Parameter` 注解的具体属性，可以通过 `Ctrl + 鼠标点击` 查看其来源，或参考下文的 `@Parameter` 部分，其中列出了常用属性
5. ==requestBody==：
	1. `@RequestBody` 注解的实例，用于描述请求体的内容，通常用于 `POST`、`PUT`、`PATCH` 等请求方法
	2. `@RequestBody` 注解的具体属性，可以通过 `Ctrl + 鼠标点击` 查看其来源，或参考下文的 `@RequestBody` 部分，其中列出了常用属性
6. ==responses==：
	1. 包含由 `@ApiResponse` 注解构成的数组，用于描述接口可能返回的 HTTP 响应
	2. `@ApiResponse` 注解的具体属性，可以通过 `Ctrl + 鼠标点击` 查看其来源，或参考下文的 `@ApiResponse` 部分，其中列出了常用属性
7. ==deprecated==：
	1. 用于标记接口是否已弃用。如果设置为 `true`，表示该接口已不推荐使用，通常会在文档中注明。默认为 `false`

> [!NOTE] 注意事项：对此处 `tags` 的理解
> 1. 默认情况下，一个 Controller 类中的所有方法，都会在类标注的 `@Tag` 中，即类级别的标签。而在此处使用 `tags`，单独指定方法自己的标签
> 2. 注意：未真正脱离类 Tag

![](source/_posts/笔记：Java%20文档/image-20250413215556974.png)

---


###### 2.7.1.3. @Parameter

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
1. ==name==：
	1. 描述的参数的名称，通常与方法参数名称一致。
2. ==description==：
	1. 用于描述参数的含义或功能。
3. ==example==：
	1. 用于为参数提供示例值，帮助开发者理解该参数的典型值。
4. ==schema==：
	1. 指向参数对应的模型类（Model 类）
5.  ==in==：
	1. 用于指定参数的来源，通常是 path、query、header 或 cookie。
	2. <font color="#00b0f0">path</font>：表示参数来自 URL 路径，通常用于路径变量（例如 /users/{id}）
	3. <font color="#00b0f0">query</font>：表示参数来自 URL 查询字符串，通常用于 GET 请求中的查询参数
	4. <font color="#00b0f0">header</font>：表示参数来自 HTTP 请求头部（例如授权令牌 Authorization）
	5. <font color="#00b0f0">cookie</font>：表示参数来自 HTTP 请求的 Cookie（例如 JSESSIONID）
6. ==required==：
	1. 用于指定参数是否为必需的，默认值为 false。
7. ==allowEmptyValue==：
	1. 用于指示该参数是否允许为空值，通常应用于查询参数，默认值为 false。

---


###### 2.7.1.4. @RequestBody

``` 
@RequestBody(  
        description = "User data",  
        required = true,  
        content = @Content(xxxxx)  
)
```
1. ==description==：
	1. 描述请求体的内容
2. ==required==：
	1. 指定请求体是否是必需的，默认是 true
3. ==content==: 
	1. `@Content` 注解的实例，用于描述请求体或响应体的内容类型和结构
	2. `@Content` 注解的具体属性，可以通过 `Ctrl + 鼠标点击` 查看其来源，或参考下文的 `@Content` 部分，其中列出了常用属性

---


###### 2.7.1.5. @ApiResponse
```
@ApiResponse(
	responseCode = "200",
	description = "Successfully retrieved the user",
	content = @Content(xxxxxx)
	)
```
1. ==responseCode==：
	1. 用于指定响应的 HTTP 状态码，例如 200、400、404、500 等
2. ==description==：
	1. 用于为该响应状态码提供详细的描述
3. ==content==: 
	1. `@Content` 注解的实例，用于描述请求体或响应体的内容类型和结构
	2. `@Content` 注解的具体属性，可以通过 `Ctrl + 鼠标点击` 查看其来源，或参考下文的 `@Content` 部分，其中列出了常用属性

---


###### 2.7.1.6. @Content

`@Content` 注解用于描述请求体或响应体的内容类型和结构
```
content = @Content(
	mediaType = "application/json",
	schema = @Schema(implementation = User.class),
	examples = @ExampleObject(value = "{\"name\":\"John Doe\",\"age\":30}"),
)
"""
1. mediaType：
	1. 用于指定响应体的 MIME 类型，例如 application/json、text/plain 等
2. schema：
	1. 指向对应的模型类（Model 类）
3. examples：
	1. 用于提供内容的示例数据
"""
```


---


##### 2.7.2. Model 注解

###### 2.7.2.1. @Schema

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
"""
1. name：
	1. 指定模型的名称。
2. description：
	1. 为该模型提供简短描述，说明它的用途或功能  
3. required：
	1. 表示该属性是否是必需的，默认为 false  
4. example：
	1. 属性实例数据
5. defaultValue：
	1. 字段的默认值  
6. format：
	1. 指定字段的格式，通常用于日期、时间等特殊格式字段，有助于文档生成工具（如 Swagger/OpenAPI）正确地展示字段的数据格式
	2. date-time：表示日期和时间，符合 ISO 8601 标准，格式为 yyyy-MM-dd'T'HH:mm:ssZ，例如 2024-01-01T12:00:00Z ，适用类型：LocalDateTime, Date, ZonedDateTime 等
	3. date：仅表示日期，格式为 yyyy-MM-dd，例如 2024-01-01，适用类型：LocalDate
	4. time：仅表示时间，格式为 HH:mm:ss，例如 12:00:00，适用类型：LocalTime
"""
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
![](image-20250517160013458.png)

---


#### 2.8. 访问 UI 文档

UI 文档默认为： http://localhost:8080/swagger-ui/index.html

---


### 3. 使用 Knife4j 生成 OpenAPI 文档

**Knife4j** 基于 Swagger-UI，提供了更多功能、更友好的用户界面和强大的定制能力，从而显著提升了 API 文档的开发与使用体验。

与 **Springfox** 或 **Springdoc** 生成的 API 文档兼容，**Knife4j** 并不会引入新的 API 注解，而是通过优化和增强现有的文档功能来提升用户体验。

其使用方式与 **Springdoc** 生成 OpenAPI 文档的方法几乎相同，唯一的区别是引入的依赖包需要替换为 [knife4j-openapi3-jakarta-spring-boot-starter](https://mvnrepository.com/artifact/com.github.xiaoymin/knife4j-openapi3-jakarta-spring-boot-starter)，而非 `springdoc-openapi-starter-webmvc-ui`。
```
<dependencies>
	<!-- 此依赖，会自动引入 Springdoc 和 Swagger UI 的依赖 -->
	<dependency>  
	    <groupId>com.github.xiaoymin</groupId>  
	    <artifactId>knife4j-openapi3-jakarta-spring-boot-starter</artifactId>  
	    <version>4.5.0</version>  
	</dependency>
</dependencies>
```

在配置时，除了基本配置外，您还可以进行更多自定义配置，参考：[Knife4j 增强模式](https://doc.xiaominfo.com/docs/features/enhance)。

然后我们可以访问 Knife4j UI 文档： http://localhost:8080/doc.html

---


### 4. 使用 Apifox

#### 4.1. 创建 Java 项目

![](image-20250517161956335.png)

---


#### 4.2. 安装 Apifox 插件

![](image-20250517162033549.png)

---


#### 4.3. 创建团队和项目

![](image-20250517162135984.png)

---


#### 4.4. 配置 Java 项目与 Apifox 项目映射关系

![](image-20250517162243052.png)

---


#### 4.5. 上传 Java 项目

![](image-20250517162530905.png)

---

