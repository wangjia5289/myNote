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

![](source/_posts/笔记：Java%20项目创建和结构/image-20250516194807264.png)


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
![](source/_posts/笔记：Java%20项目创建和结构/image-20250503170333048.png)


==2.进行追加依赖==
1. 找到 Pom.xml
2. 右键，选择 `Generate`
3. 选择 `Edit Starters`
4. URL 保持默认的 `https://start.spring.io/`，选择 OK
5. 刷新 Maven

---


### 7. 补充：手动导入项目

问题核心是：Maven 依赖没导进来，我们手动导一下即可
![](source/_posts/笔记：Java%20项目创建和结构/image-20250413192204786.png)

---

