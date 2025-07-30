---
title: 笔记：Java 注解
date: 2025-07-30
categories:
  - Java 注解
tags: 
author: 霸天
layout: post
---
---

是的，**所有标注在方法上的注解，如果它们的保留策略（`@Retention`）设置为 `CLASS` 或 `RUNTIME`，都会被保存在方法区的元数据中。**

---

让我们详细解释一下：

当 Java 编译器处理源代码时，它会将注解的信息嵌入到生成的 `.class` 文件中。这些信息最终会在 JVM 加载类时，被存储到**方法区（或元空间）**的相应位置。

关键在于注解的 **保留策略（`@Retention`）**：

1. **`@Retention(RetentionPolicy.SOURCE)`**：
    
    - **保留到源代码阶段**：这类注解只在源代码中存在。编译器在编译 `.java` 文件时会读取它们，但**不会将它们写入 `.class` 文件**。
        
    - **示例**：`@Override`、`@SuppressWarnings`。
        
    - **是否保存在元数据中？** **否**。因为它们在编译后就被丢弃了，所以方法区的元数据中不会有它们的信息。
        
2. **`@Retention(RetentionPolicy.CLASS)`**：
    
    - **保留到字节码阶段**：这类注解的信息会**被编译器写入 `.class` 文件**。JVM 在加载 `.class` 文件时，会将这些信息存储到方法区的元元数据中。
        
    - **示例**：通常是框架在编译时进行检查或代码生成时使用的注解，例如一些 Lint 工具或字节码增强库使用的注解。
        
    - **是否保存在元数据中？** **是**。但这些信息在运行时通常**不能通过反射机制直接访问**到。
        
3. **`@Retention(RetentionPolicy.RUNTIME)`**：
    
    - **保留到运行时阶段**：这类注解的信息不仅会被编译器写入 `.class` 文件，而且在 JVM 加载类到方法区后，这些信息在程序**运行时依然可以通过 Java 反射 API**（如 `Method.getAnnotations()`）进行访问。
        
    - **示例**：Spring 框架中的 `@Autowired`、JPA 中的 `@Entity`、JUnit 中的 `@Test` 等。
        
    - **是否保存在元数据中？** **是**。
        

### 总结

所以，你的问题可以更精确地回答为：

- **如果一个注解的 `@Retention` 是 `CLASS` 或 `RUNTIME`，那么它肯定会以某种形式（作为属性，Attribute）保存在方法区的元数据中。** 这样 JVM 就能知道这个注解的存在。
    
- 如果 `@Retention` 是 `SOURCE`，那么它不会进入 `.class` 文件，自然也不会在方法区的元数据中。
    

大多数你平时在框架中使用的、需要在运行时被读取和处理的注解（如 Spring 的配置注解、JPA 的实体映射注解），它们的 `@Retention` 都是 `RUNTIME`，因此它们的信息确实都保存在方法区中。

---