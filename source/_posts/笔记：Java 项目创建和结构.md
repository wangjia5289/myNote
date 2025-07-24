---
title: 笔记：Java 项目创建和结构
date: 2025-03-12
categories:
  - Java
  - Java 基础
  - Java 项目创建和结构
tags: 
author: 霸天
layout: post
---
## Spring Web 项目

### 创建方式

#### 4.2. 使用 Spring / IDEA 提供的脚手架

使用 Spring 提供的脚手架 [Spring Initializr](https://start.spring.io/) 创建项目，勾选
1. Web
	1. Spring Web
下载并解压 ZIP 包，然后在 IntelliJ IDEA 中选择 **Open** 打开该项目。

> [!NOTE] 注意事项
> 1. 如果使用 IDEA 提供的脚手架，只需要在 IntelliJ IDEA 中创建项目即可。
> 2. 脚手架默认使用最新版本进行创建。如果你想使用旧版本，可以在项目生成后手动修改 `spring-boot-starter-parent` 的版本，并重新加载项目。需要注意，Spring Boot 3 默认要求 JDK 17，因此你也需要根据实际情况调整 JDK 版本
> 3. Spring Boot 的历史版本可以在 Maven Central 仓库中查看：[Spring Boot 版本](https://central.sonatype.com/artifact/org.springframework.boot/spring-boot/versions)
> 4. 补充：`Maven Central` 是依赖的官方仓库源，而 `mvnrepository.com` 则是一个提供依赖搜索与索引的第三方黄页工具站点

---


##### 4.2.2. 手动添加 Spring Boot 依赖 (继承 Parent)

==1.继承 spring-boot-starter-parent==
```
# 继承 spring-boot-starter-parent
<parent>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-starter-parent</artifactId>  
    <version>3.4.1</version>  
</parent>
```

> [!NOTE] 补充：spring-boot-starter-parent
> [spring-boot-starter-parent](https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-parent) 是 Spring Boot 的官方父级项目，提供以下核心功能：
> 1. <font color="#00b0f0">依赖版本统一管理</font>：预定义常用依赖的兼容版本，避免版本冲突。
> 2. <font color="#00b0f0">默认构建配置</font>：内置 Maven 插件（如 `spring-boot-maven-plugin`）的优化配置。
> 3. <font color="#00b0f0">资源过滤</font>：自动处理 `application.properties` 和 `application.yml` 的占位符替换。


==2.引入所需的起步依赖==
```
# 引入所需的起步依赖
<dependency>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-starter-web</artifactId>  
    <version>XXXXXX</version>  
</dependency>  
```

> [!NOTE] 注意事项
> 1. 继承父项目后，起步依赖无需指定版本号，而是由父级统一管理

---


##### 4.2.3. 手动添加 Spring Boot 依赖 (不继承 Parent)

==1.引入 spring-boot-depencies==
```
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-dependencies</artifactId>
  <type>pom</type>
  <scope>import</scope>
</dependency>
```

> [!NOTE] 补充：spring-boot-depencies
> [spring-boot-depencies](https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-dependencies) 是一个 Spring Boot 提供的父级 BOM（Bill of Materials）文件，主要用于统一管理 Spring Boot 项目的依赖版本。


==2.引入所需的起步依赖==
```
# 引入所需的起步依赖
<dependency>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-starter-web</artifactId>  
</dependency>  
```

> [!NOTE] 注意事项
> 1. 起步依赖无需指定版本号，而是由 `spring-boot-depencies` 一管理
> 2. 常用起步依赖有见 `Categories/Spring/Boot 常用起步依赖`

---


#### 4.3. 创建主类并添加注解

在 `src/main/java/com/example` 下创建一个主类 `Application`，并添加 `@SpringBootApplication` 注解，它是 Spring Boot 应用的入口点
```
/**
* @SpringBootApplication 是组合注解，包含
*    1. @SpringBootConfigutation：标识为配置类
*    2. @EnableAutoConfiguration：启用自动配置
*    3. @ComponentScan：扫描当前包及子包的组件
*/

@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

> [!NOTE] 注意事项
> 1. 主类的写法非常重要，需要牢牢记住

---


#### 4.4. 对项目进行必要配置

```
spring:
  application:
    name: xxxxxx                    # 项目名称，一般也作为 Consul 注册的名称
server:  
  port: 8080                        # 项目端口号
  servlet:  
    encoding:                       # 项目编码
      enabled: true  
      charset: UTF-8  
      force: true
```

---


#### 4.5. 检查打包插件
如果我们使用 Dockerfile 的多阶段构建（multi-stage build）来打包 Spring Boot 项目，那么需要注意：

默认情况下，当执行 `mvn package` 命令时，Maven 只会将项目自身的 `.class` 文件和 `resources` 文件打包进生成的 JAR 文件，不会自动将外部依赖（如 Spring Boot 框架或第三方库）一同打包进去。

因此，打包出来的 JAR 文件并不能独立运行，除非目标环境已经具备所有所需依赖。这在容器环境中通常是不满足的。

为了生成可独立运行的可执行 JAR（fat jar 或 Uber JAR），我们需要在 `pom.xml` 中配置相关的打包插件：
- `spring-boot-maven-plugin`（适用于 Spring Boot 项目，官方推荐）
- `maven-shade-plugin`（适用于通用 Java 项目）

通常情况下，如果我们使用 Spring Initializr 等 Spring Boot 脚手架生成项目，它会自动为我们添加 `spring-boot-maven-plugin` 插件。但如果是手动创建项目，或者部分脚手架没有包含它，就需要我们手动添加。因此，建议检查一下项目中是否配置了打包插件，以确保能够正确构建可运行的 JAR 包：
```
<build>  
    <plugins>  
        <plugin>  
            <groupId>org.springframework.boot</groupId>  
            <artifactId>spring-boot-maven-plugin</artifactId>  
        </plugin>  
    </plugins>  
</build>
```



## Web 项目

### 普通 Web 项目

1. 打开 IDEA 工具，安装 JavaToWeb 插件
2. 创建空项目
3. 配置 Maven、JDK
4. 创建一个 Maven 模块
5. 右键，点击 JBLJavaToWeb
6. Ctrl + F5 刷新一下
7. 添加打包插件
	1. 如果我们使用 Dockerfile 的多阶段构建（multi-stage build）来打包 Java Web 项目，那么需要注意：
	2. 默认情况下，当执行 `mvn package` 命令时，Maven 只会将项目自身的 `.class` 文件和 `resources` 文件打包进生成的 JAR 文件，不会自动将外部依赖（如 Spring Boot 框架或第三方库）一同打包进去。
	3. 因此，打包出来的 JAR 文件并不能独立运行，除非目标环境已经具备所有所需依赖。这在容器环境中通常是不满足的。
	4. 为了生成可独立运行的可执行 JAR（fat jar 或 Uber JAR），我们需要在 `pom.xml` 中配置相关的打包插件：
		- `spring-boot-maven-plugin`（适用于 Spring Boot 项目，官方推荐）
		- `maven-shade-plugin`（适用于通用 Java 项目）
	5. 所以我们需要手动添加[打包插件](https://mvnrepository.com/artifact/org.apache.maven.plugins/maven-shade-plugin)：
```
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-shade-plugin</artifactId>
  <version>3.2.4</version>
  <executions>
    <execution>
      <phase>package</phase>
      <goals><goal>shade</goal></goals>
      <configuration>
        <transformers>
          <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
            <mainClass>com.example.Main</mainClass>
          </transformer>
        </transformers>
      </configuration>
    </execution>
  </executions>
</plugin>
```









### 2. Web 应用开发

![](image-20250516194807264.png)


以追加 Spring Web 依赖的方式，勾选：
1. ==Web==
	1. Spring Reactive Web

### 3. 创建 Web 项目

---


### 4. 创建 Spring Web 项目

#### 4.1. 创建 Spring Web 项目

---


### 5. 创建 Spring Cloud 项目

这里采用 IDEA 提供的脚手架创建 Spring Boot 项目，分别勾选：
1. ==Web==
	1. Spring Web
2. ==Spring Cloud Config==：
	1. Consul Configuration
3. ==Spring Cloud Discovery==
	1. Consul Discovery
4. ==Spring Cloud Routing==：
	1. Openfeign
	2. Gateway

---


### 6. 补充：如何追加依赖

==1.安装 editstarters 插件==
![](image-20250503170333048.png)


==2.进行追加依赖==
1. 找到 Pom.xml
2. 右键，选择 `Generate`
3. 选择 `Edit Starters`
4. URL 保持默认的 `https://start.spring.io/`，选择 OK
5. 刷新 Maven

---


### 7. 补充：手动导入项目

问题核心是：Maven 依赖没导进来，我们手动导一下即可
![](image-20250413192204786.png)

---





# 完善 Spring Cloud 项目结构以实现成熟架构

**引言**

现代应用程序日益增长的复杂性要求采用模块化和组织良好的架构。Spring Cloud 提供了一套全面的工具，用于构建分布式系统，而一个成熟的项目结构对于确保应用程序的可维护性、可伸缩性和长期成功至关重要。组织良好的多模块架构具有诸多优势，包括改进的可维护性，允许团队独立处理不同的功能模块；增强的可伸缩性，可以根据需要单独扩展特定模块；提高代码可重用性，通用组件可以在不同的模块之间共享；以及促进更好的团队协作，每个团队可以专注于特定的模块 1。本报告旨在提供一个全面的指南，用于将所提供的基本 Spring Cloud 项目结构增强为一个更成熟和健壮的设计，其中融入了行业最佳实践。

**分析现有结构**

所提供的项目结构以根包 `com.example` 为基础，组织成三个功能模块 `module1`、`module2` 和 `module3`，以及一个用于通用配置和工具的 `application` 模块。每个功能模块都包含一个标准的 Spring Boot 应用程序结构，包括启动类 (`Application.java`)、控制器层 (`controller`)、业务层 (`service`，进一步划分为接口 `inter` 和实现 `impl`)、持久层 (`repository`) 和模型层 (`model`，细分为实体 `entity`、数据传输对象 `dto` 和视图对象 `vo`)。`application` 模块则包含用于配置的 `config` 包和用于通用工具类的 `util` 包。

现有结构展现了初步的模块化组织和基本的层次分离，这有助于代码的组织和理解 5。每个模块的独立性允许在一定程度上实现关注点分离。通用配置和工具类的集中化管理也是一个积极的方面。然而，根据成熟的 Spring Cloud 架构原则和研究资料，该结构仍有改进的空间。首先，模块的划分似乎是基于技术上的分离（例如，不同的功能模块），而不是基于明确的业务领域或特定的功能。其次，该结构缺乏关于模块之间依赖关系管理的具体说明。第三，没有明确提及集中式配置或服务发现机制，这对于构建可伸缩和弹性的分布式系统至关重要。此外，`application` 模块的范围可能需要扩展，以包含诸如日志记录、异常处理和安全性配置等关键方面。最后，该结构没有提供关于构建配置、测试策略或文档实践的信息。

**多模块组织的最佳实践**

在构建一个成熟的 Spring Cloud 项目时，选择合适的多模块组织策略至关重要。主要有两种常见的方法：基于领域驱动设计 (DDD) 和基于技术分层 3。

领域驱动设计强调根据业务能力或领域来组织模块。例如，在一个电子商务应用中，可能会有 `customer`、`order` 和 `product` 等模块 4。这种方法的优势在于它将软件结构与业务紧密结合，改善了技术和业务人员之间的沟通，并促进了业务特性的独立开发和部署 4。相反，基于技术分层的组织方式则根据技术职责来划分模块，例如 `api`、`service` 和 `data-access` 9。这种方法的好处是关注点分离清晰，易于维护各个技术层。然而，纯粹的技术分离可能导致开发人员需要跨多个模块处理单个功能，从而降低效率 4。对于复杂的业务应用程序，领域驱动设计通常能带来更好的可维护性和可伸缩性。对于较简单的应用程序或围绕技术专长组建的团队，技术分层可能就足够了。项目应该根据其具体需求和团队结构来选择最合适的策略。

识别逻辑模块需要仔细考虑应用程序的业务功能和技术需求。模块可以基于业务能力（如用户管理、订单处理、报告）或技术关注点（如核心库、基础设施服务）来划分。关键在于确保每个模块都有清晰的边界和明确的职责 4。此外，还应考虑模块的独立部署和可伸缩性 8。对于较小的项目，可以先从单一项目开始，并按功能进行包组织，随着应用程序的增长，再逐步重构为多模块结构 14。

在多模块项目中，依赖管理至关重要。父 POM（在 Maven 中）或根 `build.gradle`（在 Gradle 中）用于管理子模块的依赖关系 1。Maven 的 `pom.xml` 文件中的 `<modules>` 标签和 Gradle 的 `settings.gradle` 文件中的 `include` 语句用于定义子模块 2。为了提高模块的独立性并降低耦合，应该尽量减少模块之间的依赖关系 4。Gradle 提供了 `api` 和 `implementation` 依赖项的概念，用于控制模块之间依赖项的可见性 4。推荐使用物料清单 (BOM) 来一致地管理所有模块中第三方依赖项的版本 18。父模块应该指定子模块，并可能在 Maven 的 `dependencyManagement` 部分管理公共依赖项 2。每个模块都应该包含其自身功能所需的所有构件，包括其自身的依赖项，以最大限度地减少对其他模块的依赖 4。

**增强模块结构**

在每个模块内部，采用标准的包结构有助于代码的组织和可维护性。常见的做法是将关注点分离到不同的包中，例如 `controller`、`service`（进一步细分为 `inter` 和 `impl` 子包）、`repository`、`model`（细分为 `entity`、`dto` 和 `vo`）、`config`、`util`、`exception`，以及可选的 `security` 和 `validation` 包 3。`controller` 包负责处理传入的 HTTP 请求并返回响应；`service` 包包含业务逻辑，通常接口定义在 `inter` 子包中，而具体实现在 `impl` 子包中；`repository` 包管理数据访问和持久化；`model` 包定义数据结构，包括 JPA 实体、用于数据交换的数据传输对象 (DTO) 和用于表示数据的视图对象 (VO)；`config` 包存放应用程序特定的配置类；`util` 包包含可重用的工具类；`exception` 包定义自定义异常类以实现更好的错误处理；`security` 包管理身份验证和授权逻辑；`validation` 包实现数据验证规则。按照功能组织包（例如，`controller`、`service`、`repository`、`model`）是一种常见且推荐的做法，可以更好地导航和组织模块内的代码 5。然而，有些团队更喜欢按特性组织，将特定特性的所有相关类（控制器、服务、存储库）分组到专用包中 3。

坚持关注点分离原则至关重要，确保每个组件（类、包、模块）都有特定的职责，避免混合不同的关注点 19。这有助于提高可维护性、可测试性并降低引入错误的风险。控制器应主要处理 HTTP 请求，服务应包含业务逻辑，而存储库应管理数据持久性 6。

模型层需要进一步细化 `entity`、`dto` 和 `vo` 类的用途和用法 5。`entity` 通常表示持久化数据模型，通常映射到数据库表。`dto` 是用于在层或服务之间交换数据的数据传输对象，通常针对特定用例定制，并且可能与实体不同。`vo` 是表示不可变数据结构或用于向 UI 展示数据的视图对象。使用 DTO 有助于将 API 与内部数据模型（实体）解耦，从而允许两者独立更改而不会直接影响彼此 9。`entity` 和 `model` 作为包名称的选择可能取决于模型对象是否严格绑定到 ORM 或代表更广泛的领域模型 21。

**使用 Spring Cloud Consul 实现集中式配置**

Spring Cloud Consul Config 提供了一种强大的机制来管理分布式应用程序的集中式配置 22。它利用 Consul 的 Key/Value 存储作为配置数据的存储中心 22。使用 Consul 进行集中式配置管理具有诸多优势，例如动态配置更新、版本控制以及跨多个服务的一致配置源。

要配置 Spring Cloud Consul Config，通常需要在 `bootstrap.yml` 文件（在 `application.yml` 之前加载）中进行配置，尤其是在使用较旧的 Spring Cloud 版本时。对于 Spring Boot 2.4+，推荐使用 `spring.config.import` 属性在 `application.yml` 中启用 Consul 配置 22。以下是一些基本的配置示例：

YAML

```
spring:
  cloud:
    consul:
      host: localhost # Consul 代理的主机名或 IP 地址 [22, 23, 28, 29, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42]
      port: 8500    # Consul 代理的端口号（默认为 8500）[22, 23, 28, 29, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42]
      config:
        enabled: true # 启用 Consul 配置 [22, 23, 30, 33, 34, 35, 36, 40, 41]
        format: YAML    # Consul 中配置数据的格式（可以是 YAML 或 PROPERTIES）[23, 28, 29, 32, 34, 35, 36]
        data-key: data  # 保存 YAML/Properties 数据的 Key [23, 28, 29, 32, 35, 36]
        fail-fast: false # 在开发环境中，如果 Consul 不可用，则不停止应用程序启动 [23, 32, 40, 43]
```

或者，对于 Spring Boot 2.4+：

YAML

```
spring:
  config:
    import: consul:
```

Consul 服务器的详细信息和应用程序名称通常在 `bootstrap.yml` 中配置，因为该文件在应用程序生命周期的早期加载 22。`spring.cloud.bootstrap.enabled` 属性会影响 Consul 配置是否应放置在 `bootstrap.yml` 或 `application.yml` 中，尤其是在较旧的 Spring Cloud 版本中 23。

在 Consul 中，建议将配置键组织在一个公共前缀下（例如 `/config`），后跟应用程序名称和可选的配置文件（例如 `/config/my-app`、`/config/my-app,dev`）22。Consul Config 从多个上下文中加载属性，更具体的上下文（例如，带有配置文件的应用程序）优先于不太具体的上下文（例如，默认的 `application` 上下文）23。为了更好地组织，建议将配置属性以 YAML 或 Properties 格式存储在单个键（例如 `data`）中 23。

要启用来自 Consul 的配置更改的动态更新，可以使用 `@RefreshScope` 注解标记使用配置属性的 Spring Bean 22。此外，还可以使用 `/refresh` actuator 端点手动触发配置重新加载 23。Spring Cloud Consul 还提供了 Config Watch 功能，用于自动检测和重新加载配置更改 23。

**使用 Spring Cloud Consul 集成服务发现**

服务发现是微服务或分布式环境中的关键概念，它允许服务在没有硬编码网络位置的情况下相互查找和通信 22。Spring Cloud Consul Discovery 利用 Consul 作为服务注册表和发现提供者 22。Spring Cloud Consul 利用 Consul HTTP API 进行服务注册和发现 37。

要集成服务发现，首先需要将 `spring-cloud-starter-consul-discovery` 依赖项添加到项目中 37。然后，在 `bootstrap.yml` 或 `application.yml` 中进行以下配置：

YAML

```
spring:
  application:
    name: your-service-name # 您的服务名称 [22, 29, 31, 32, 33, 35, 37, 38, 41, 42]
  cloud:
    consul:
      host: localhost
      port: 8500
      discovery:
        enabled: true # 启用服务发现 [29, 37, 38, 44]
        prefer-ip-address: true # 注册服务的 IP 地址 [32, 33, 38]
        instance-id: ${spring.application.name}:${random.value} # 可选：设置唯一的实例 ID [22, 29, 31, 32, 35, 37, 38, 41]
        healthCheckPath: ${management.server.servlet.context-path}/actuator/health # 健康检查路径 [22, 28, 29, 35, 37, 38]
        healthCheckInterval: 15s # 健康检查间隔 [22, 28, 29, 35, 37, 38]
        metadata:
          version: v1 # 可选：添加元数据 [29, 35, 37]
```

Consul 默认会自动创建一个 HTTP 健康检查，每 10 秒访问已注册服务的 `/health` 端点 35。如果应用程序使用非默认的上下文路径，可以使用 `spring.cloud.consul.discovery.healthCheckPath` 自定义健康检查路径 22。可以使用 `spring.cloud.consul.discovery.healthCheckInterval` 配置健康检查间隔 22。可以使用 `spring.cloud.consul.discovery.metadata` 或 `spring.cloud.consul.discovery.tags` 向服务注册添加元数据 29。

**添加通用配置和实用程序**

为了进一步完善项目结构，需要考虑添加通用配置和实用程序。可以将跨多个模块共享的配置放在 Consul Config 的默认上下文中 (`/config/application`) 23。另一种方法是创建一个专用的 "config" 模块，其中包含通用的配置类，并将其作为依赖项包含在其他模块中。

对于可重用的实用程序类和辅助函数，建议创建一个单独的模块（例如 `common-utils`）2。这有助于提高代码重用率并减少模块之间的重复代码。创建一个单独的库模块来存放公共服务和实用程序是多模块项目中的良好实践 2。

应该实现一致且集中的异常处理方法，可以在 `application` 模块中使用全局异常处理程序，也可以在每个模块的 `exception` 包中实现 6。确保应用程序返回一致的错误响应格式。此外，应该标准化所有模块的日志记录，可以在父 POM 或专用配置模块中配置通用的日志记录框架（例如 SLF4j with Logback）20。

**构建和测试的最佳实践**

在多模块项目中，优化构建过程至关重要。Maven 或 Gradle 的构建配置文件应该定义模块的构建顺序（如果需要）。可以使用 Maven profiles 或 Gradle build variants 来针对不同的环境进行构建。Spring Boot Maven 插件和 Gradle 插件用于打包和运行 Spring Boot 应用程序 2。Maven 和 Gradle 都提供了管理多模块项目的构建配置的方法 1。

自动化测试对于确保每个模块的质量至关重要。每个模块都应该包含单元测试、集成测试和端到端测试 4。测试源代码应该放在每个模块的单独测试源目录 (`src/test/java`) 中。如果依赖模块在测试期间需要测试实用程序或数据，可以为模块创建单独的测试 JAR 4。

**文档**

清晰全面的文档对于每个模块和整个项目都是必不可少的 3。应该记录每个模块的用途、依赖关系和主要功能。对于 REST API，建议使用 Swagger (OpenAPI) 等工具进行文档化。

**结论**

通过采纳这些最佳实践，可以将提供的基本 Spring Cloud 项目结构增强为一个成熟且健壮的架构。选择与项目特定需求相符的模块组织策略（DDD 或技术分层）至关重要。实施集中式配置和服务发现（使用 Spring Cloud Consul）可以提高应用程序的可维护性、可伸缩性和弹性。添加通用配置和实用程序模块可以促进代码重用并确保跨应用程序的一致性。最后，采用全面的构建和测试策略以及提供清晰的文档对于项目的长期成功至关重要。随着应用程序的发展，持续改进和调整项目结构也是至关重要的 4。

**附录：推荐的包结构**

|   |   |
|---|---|
|**包名**|**描述/职责**|
|`controller`|处理传入的 HTTP 请求并返回响应。|
|`service`|包含业务逻辑。通常包含 `inter` 子包（接口定义）和 `impl` 子包（接口实现）。|
|`repository`|管理数据访问和持久化操作。|
|`model`|定义应用程序的数据结构。通常包含 `entity` 子包（JPA 实体）、`dto` 子包（数据传输对象）和 `vo` 子包（视图对象）。|
|`config`|包含应用程序特定的配置类，例如数据库配置、安全配置等。|
|`util`|包含可重用的实用程序类和辅助函数。|
|`exception`|定义应用程序特定的自定义异常类，用于更好的错误处理。|
|`security`|管理应用程序的身份验证和授权逻辑。|
|`validation`|实现数据验证规则，例如使用 Bean Validation API。|

**附录：常用 Spring Cloud Consul 配置属性**

|   |   |   |   |
|---|---|---|---|
|**属性名**|**描述**|**示例值**|**配置文件**|
|`spring.cloud.consul.host`|Consul 代理的主机名或 IP 地址。|`localhost`, `192.168.1.100`|`bootstrap.yml`|
|`spring.cloud.consul.port`|Consul 代理的端口号（默认为 8500）。|`8500`|`bootstrap.yml`|
|`spring.config.import`|启用 Consul 配置导入（Spring Boot 2.4+）。|`consul:`|`application.yml`|
|`spring.cloud.consul.config.enabled`|启用 Consul 配置（较旧版本）。|`true`, `false`|`bootstrap.yml`|
|`spring.cloud.consul.config.format`|Consul 中配置数据的格式。|`YAML`, `PROPERTIES`|`bootstrap.yml`|
|`spring.cloud.consul.config.data-key`|保存 YAML/Properties 数据的 Key。|`data`|`bootstrap.yml`|
|`spring.cloud.consul.config.fail-fast`|如果 Consul 不可用，是否停止应用程序启动。|`true`, `false`|`bootstrap.yml`|

**附录：常用 Spring Cloud Consul 服务发现配置属性**

|   |   |   |   |
|---|---|---|---|
|**属性名**|**描述**|**示例值**|**配置文件**|
|`spring.cloud.consul.discovery.enabled`|启用服务发现。|`true`, `false`|`bootstrap.yml`|
|`spring.application.name`|应用程序的名称，用于在 Consul 中注册服务。|`user-service`, `order-service`|`bootstrap.yml`|
|`spring.cloud.consul.discovery.instanceId`|服务的唯一实例 ID。|`${spring.application.name}:${random.value}`|`bootstrap.yml`|
|`spring.cloud.consul.discovery.prefer-ip-address`|是否注册服务的 IP 地址而不是主机名。|`true`, `false`|`bootstrap.yml`|
|`spring.cloud.consul.discovery.healthCheckPath`|Consul 执行健康检查的端点路径。|`${management.server.servlet.context-path}/actuator/health`|`bootstrap.yml`|
|`spring.cloud.consul.discovery.healthCheckInterval`|Consul 执行健康检查的时间间隔。|`15s`, `1m`|`bootstrap.yml`|
|`spring.cloud.consul.discovery.metadata.version`|附加到服务注册的元数据。|`v1`, `v2`|`bootstrap.yml`|



# -------




```
com.example.myproject  
├── Application.java                             // 启动类，包含 main 方法  
├── config                                       // 配置类  
│   ├── SecurityConfig.java  
│   ├── DataSourceConfig.java  
│   └── AppConfig.java  
├── controller                                   // 控制器层
│   ├── StudentController.java  
│   └── ScoreController.java  
├── service                                      // 业务层
│   ├── inter                                    // 服务接口  
│   │   ├── StudentService.java  
│   │   └── ScoreService.java  
│   └── impl                                     // 服务实现  
│       ├── StudentServiceImpl.java  
│       └── ScoreServiceImpl.java  
├── repository                                   // 仓库层，处理数据库交互  
│   ├── StudentRepository.java  
│   └── ScoreRepository.java  
├── model                                        // 模型层，包含实体、DTO 和 VO  
│   ├── entity                                   // 实体类，映射数据库表  
│   │   ├── Student.java  
│   │   └── Score.java  
│   ├── dto                                      // 数据传输对象，服务间数据传递  
│   │   ├── StudentDTO.java  
│   │   └── ScoreDTO.java  
│   └── vo                                       // 视图对象，前端展示数据  
│       ├── StudentVO.java  
│       └── ScoreVO.java  
├── util                                         // 工具类，包含常用工具  
│   ├── DateUtils.java  
│   ├── StringUtils.java  
│   └── EncryptionUtils.java  
├── exception                                    // 自定义异常类  
│   └── CustomException.java  
└── common                                       // 通用工具和常量  
    ├── Constants.java  
    └── CommonUtils.java  
```





```
Spring_Data_MySQL
├── src
│   └── main
│       ├── java
│       │   └── com.example.spring_data_mybatis
│       │       ├── Application.java                          // 项目启动类
│       │       ├── config
│       │       │   └── mapper
│       │       │       ├── DataSource1Config.java            // 数据源1配置类
│       │       │       └── DataSource2Config.java            // 数据源2配置类
│       │       ├── mapper                                    // 或者是 DAO
│       │       │   ├── datasource1
│       │       │   │   └── UserMapper.java                   // UserMapper 接口（数据源1下）
│       │       │   └── datasource2
│       │       │       └── CarMapper.java                    // CarMapper 接口（数据源2下）
│       │       ├── model
│       │       │   └── entity
│       │       │       ├── datasource1
│       │       │       │   └── User.java                     // User 实体类（数据源1下）
│       │       │       └── datasource2
│       │       │           └── Car.java                      // Car 实体类（数据源2下）
│       │       ├── service
│       │       │   ├── datasource1
│       │       │   │   ├── UserService.java                  // UserService 接口（数据源1下）
│       │       │   │   └── UserServiceImpl.java              // UserService 实现类
│       │       │   └── datasource2
│       │       │       ├── CarService.java                   // CarService 接口（数据源2下）
│       │       │       └── CarServiceImpl.java               // CarService 实现类
│       │       └── controller
│       │           └── TestController.java                   // TestController 类
│       └── resources
│           └── mapper
│               ├── datasource1
│               │   └── UserMapper.xml                        // MyBatis 映射文件（数据源1下）
│               └── datasource2
│                   └── CarMapper.xml                         // MyBatis 映射文件（数据源2下）
```

### 2、多个模块

在 Spring Boot 项目中，可以创建多个模块，并将它们作为子包组织在根包下。每个子包中可以按照“单一模块”结构进行定义，保持模块的独立性和可维护性。项目结构示例如下：
```
com.example.根包
|-- module1               // 第一个模块
|   |-- Application.java  // 启动类
|   |-- controller        // 控制器层
|   |   |-- StudentController.java
|   |   |-- ScoreController.java
|   |-- service           // 业务层
|   |   |-- inter         // 业务层接口
|   |   |-- impl          // 接口实现包
|   |-- repository        // 持久层
|   |-- model             // 模型层
|   |   |-- entity        // 实体类
|   |   |-- dto           // 数据传输对象
|   |   |-- vo            // 视图对象
|-- module2               // 第二个模块
|   |-- Application.java  // 启动类
|   |-- controller        // 控制器层
|   |-- service           // 业务层
|   |-- repository        // 持久层
|   |-- model             // 模型层
|-- module3               // 第三个模块
|   |-- Application.java  // 启动类
|   |-- controller        // 控制器层
|   |-- service           // 业务层
|   |-- repository        // 持久层
|   |-- model             // 模型层
|-- application           // 通用配置或工具包
|   |-- config            // 配置文件
|   |-- util              // 工具类

```

---




















	