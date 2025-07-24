---
title: 笔记：Spring Aop
date: 2025-03-11
categories:
  - Java
  - Spring 生态
  - Spring Aop
tags: 
author: 霸天
layout: post
---
# 一、理论 

### 0、导图：[Map：Spring AOP](Map：SpringAOP.xmind)

---


### 1、横切关注点

#### 1.1、模型图

![](image-20250311102729843.png)

---


#### 1.2、常见关注点

1. **日志记录**：自动记录方法的调用、输入输出参数、执行时间等，避免在每个方法中手动编写日志代码。

2. **事务管理**：在数据操作前后自动管理事务，确保数据一致性。

3. **权限验证**：在方法调用前检查用户权限。

4. **性能监控**：记录方法执行时间，帮助优化系统性能。

5. **异常处理**：集中管理异常处理逻辑，避免在多个地方重复代码。

6. **其他功能增强

---


### 2、AOP

#### 2.1、AOP 概述

==问题1：横切关注点的管理==
在复杂的应用程序中，诸如日志记录、事务管理等功能常常在多个模块中反复出现。这些与核心业务逻辑无关的功能统称为横切关注点。由于它们与核心业务逻辑无关却紧密交织，导致了以下问题：
- **代码冗余**：相同的横切关注点在不同模块中重复实现。
- **维护成本高**：当横切关注点发生变化时，开发者需要在多个地方修改，容易出错，增加了维护的复杂度。
- **代码可读性下降**：核心业务逻辑与横切关注点混杂，使代码难以理解和维护。
为了解决这些问题，我们可以考虑将横切关注点抽离出来，进行统一管理。在核心业务逻辑执行时，再将这些横切关注点插入到相应的执行流程中。这种方法能够提高代码的可维护性和可读性，降低冗余和复杂性。


==问题2：功能增强的复杂性==
在开发过程中，我们经常需要对现有方法进行增强，例如在方法执行流程中插入其他功能。代理机制使我们能够在不改变目标对象的情况下添加新功能，因此代理模式常被用于功能增强。然而，手动创建代理类并管理代理行为在大规模应用中往往显得繁琐且容易出错。
为了解决这一问题，我们可以考虑对代理模式进行封装，以简化操作，使开发者能够更轻松地应用增强功能，从而降低复杂性和出错率。


==问题的核心==
这两个问题的核心在于：如何统一管理横切关注点，以及如何在方法（核心业务）的执行流程中轻松地插入其他功能（横切关注点）。我们需要一种简单而有效的方式，使这些功能的插入既灵活又不增加开发和维护的复杂性。


==AOP 解决方案==
AOP（Aspect-Oriented Programming，面向切面编程）是一种通过“切面”实现横切关注点管理的编程方式。它使开发者能够在不修改原始业务逻辑的前提下，为系统添加或修改功能增强，从而提升系统的灵活性和可维护性。AOP 通过将横切关注点与业务逻辑分离，实现集中管理和自动化增强，解决了以下两个关键问题：
1. **集中管理横切关注点**：
    - AOP 使用“切面”（Aspect）将横切关注点的逻辑从业务代码中抽离，减少代码重复，提升代码的清晰度。常见的横切关注点包括日志记录、安全性检查和事务管理等。
2. **简化功能增强实现**：
    - AOP 封装了代理模式，使开发者无需手动编写代理类。通过简单的配置或注解，开发者可以动态为目标方法添加增强逻辑，例如在方法调用前后执行额外操作，如日志记录、权限验证或性能监控。

---


#### 2.2、AOP 的实现方式

AOP（面向切面编程）的实现方式一般是基于**代理模式**或**字节码增强技术**来实现，具体取决于框架的设计和实现。

---


#### 2.3、常见 AOP 框架和库
| 框架             | 织入时机             | 特点                   | 适用场景                |
| -------------- | ---------------- | -------------------- | ------------------- |
| **AspectJ**    | 编译期 / 类加载期 / 运行期 | 功能强大，支持多种织入方式，适合大型项目 | 需要强大的功能并独立于框架       |
| **Spring AOP** | 运行期              | 基于代理实现，简单易用，性能稍弱     | 如果项目使用的是 Spring     |
| **JBoss AOP**  | 编译期 / 运行期        | 与 JBoss 深度集成，配置稍复杂   | 如果是 Java EE 项目      |
| **Guice AOP**  | 运行期              | 简单直观，但功能有限           | 如果项目基于 Guice 依赖注入框架 |

---


#### 2.4、Spring AOP 是什么

Spring AOP 是 Spring 框架提供的 AOP 实现，专门用于与 Spring 应用程序集成。它允许开发者在 Spring 管理的 Bean 对象中定义切面和切点，从而在方法执行前后插入横切逻辑。

---


#### 2.5、Spring AOP 七大术语

##### 2.5.1、目标对象（Target）

目标对象是我们希望在其上应用切面的对象。通过切面增强目标对象的功能，就是在目标对象的目标方法上进行增强，通常是通过在方法执行前、后或抛出异常时插入特定的逻辑。
``` 
// 目标类
public class UserService {
    // 目标方法
    public void createUser(String username) {
        System.out.println("User created: " + username);
    }
}
```

---


##### 2.5.2、连接点（Joinpoint）

连接点指的是在目标对象的方法执行过程中，切面可以织入的具体位置，包括方法执行前、方法执行后、以及方法抛出异常后等常见的连接点。

---


##### 2.5.3、切面（Aspect）

切面定义了在哪些切点应用特定的逻辑（切面 = 切面类 + 切点 + 通知）。一个程序可以包含多个切面，而每个切面可以包含多个通知，并可应用于多个切点。通过切面，能够将横切关注点从核心业务逻辑中分离出来，从而提高代码的可维护性和可复用性。
``` 
@Aspect // 在 Java 中，一个切面就是一个 @Aspect 标注的切面类
@Order(1) // 规定在多个切面中，本切面的执行顺序，数字越小越先执行
public class LoggingAspect {
    ......
}
```

---


##### 2.5.4、切点（Pointcut）

###### 2.5.4.1、切点概述

切点是一个表达式，用于指定在目标方法的哪些连接点上应用通知，简单来说，就是定义了通知的执行位置。
```
@Aspect
@Order(1)
public class LoggingAspect {

    // 切点：匹配所有控制器方法
    @Pointcut("execution(* com.example.controller.*.*(..))")
    public void controllerMethods() {}
    
}
```

---


###### 2.5.4.2、切点表达式书写方法

![](image-20250311103200725.png)

----


###### 2.5.4.3、切点表达式应用方法

==1.分离切点定义与通知（推荐）==
``` 
/**
在这种方式中，切点定义和通知是分开的，切点通过`@Pointcut 注解定义，通知通过`@Before 和`@After 等注解绑定到具体的切点。这种方式结构清晰，适合在切面中有多个切点的场景。
**/

@Aspect
@Component
public class LoggingAspect {
    
    // 定义切点1：需要的切点
    @Pointcut("execution(* com.example.UserService.do1()) || execution(* com.example.UserService.do3())")
    public void selectedMethods() {}
    
    // 定义切点2：只是用来演示，演示切面类中可以有很多切点
    @Pointcut("execution(* com.example.controller..*(..))")
    public void controllerMethods() {}

    // 前置通知
    @Before("controllerMethods()")
    public void logBeforeController(JoinPoint joinPoint) {
        System.out.println("Controller method executing: " + joinPoint.getSignature().getName());
    }

    // 后置通知
    @After("controllerMethods()")
    public void logAfterController(JoinPoint joinPoint) {
        System.out.println("Controller method executed: " + joinPoint.getSignature().getName());
    }

}
```


==2.内联切点定义与通知==
``` 
/*
在这种方式中，切点直接内联在通知上，切点和通知定义在一起。这种方式代码更加简洁，适合简单的场景，尤其是只有少量切点和通知的情况。
*/

@Aspect
@Component
public class LoggingAspect {
    
    // 切点 + 前置通知
    @Before("execution(* com.example.UserService.do1()) || execution(* com.example.UserService.do3())")
    public void logBeforeController(JoinPoint joinPoint) {
        System.out.println("Controller method executing: " + joinPoint.getSignature().getName());
    }

    // 切点 + 后置通知
    @After("execution(* com.example.UserService.do1()) || execution(* com.example.UserService.do3())")
    public void logAfterController(JoinPoint joinPoint) {
        System.out.println("Controller method executed: " + joinPoint.getSignature().getName());
    }
}
```

---


##### 2.5.5、通知（Advice）

###### 2.5.5.1、通知概述

通知是切面中具体的操作逻辑，它定义了在特定的连接点（如方法执行前、后、抛出异常时等）时执行的行为。
```
@Before("execution(* com.example.service.*.*(..))")
public void logBefore(JoinPoint joinPoint) {
    System.out.println("Before method: " + joinPoint.getSignature());
}
```

---


###### 2.5.5.2、通知的种类

1. **环绕通知（@Around ）**：围绕目标方法执行，既可以在方法前，也可以在方法后执行，还可以决定是否执行目标方法（最强大，也最常用）。
2. **前置通知（@Before）**：在目标方法执行前执行。
3. **后置通知（@AfterRetruning）**：在目标方法执行后执行（无论是否抛出异常）。
4. **异常通知（@AfterThrowing）**：在目标方法抛出异常时执行。
5. **最终通知（@After）**：在目标方法执行后最终执行，无论方法是正常返回还是抛出异常。

---


###### 2.5.5.3、通知的顺序
![](image-20250311103243112.png)

---


###### 2.5.5.4、通知的应用方法

``` 
@Component
@Aspect(1) // 指定多个通知之间的执行顺序
@Component
public class MyAspect {

// 环绕通知：@Around
@Around("execution(* com.powernode.spring6.service.OrderService.*(..))")
public void aroundAdvice(ProceedingJoinPoint proceedingJoinPoint) throws Throwable {
     // 前方法
	System.out.println("环绕通知开始");
	// 固定写法
	proceedingJoinPoint.proceed();
	 // 后写法
	System.out.println("环绕通知结束");
}

// 前置通知：@Before
@Before("execution(* com.powernode.spring6.service.OrderService.*(..))")
public void beforeAdvice() {
	System.out.println("前置通知");
}

// 后置通知：@AfterRetruning
@AfterReturning("execution(* com.powernode.spring6.service.OrderService.*(..))")
public void afterReturningAdvice() {
	System.out.println("后置通知");
}

// 异常通知：@AfterThrowing
@AfterThrowing("execution(* com.powernode.spring6.service.OrderService.*(..))")
public void afterThrowingAdvice() {
	System.out.println("异常通知");
}

// 最终通知：@After
@After("execution(* com.powernode.spring6.service.OrderService.*(..))")
public void afterAdvice() {
	System.out.println("最终通知");
}

}
```

---


##### 2.5.6、织入（Weaving）

织入是将切面与目标程序类型结合的过程，决定了切面何时以及如何应用。不同框架的织入时机有所不同，Spring AOP 的织入发生在运行时，切面会动态地应用到目标对象的连接点上。

---


##### 2.5.7、代理对象（Proxy）

代理对象是由 AOP 框架创建的，它是在目标对象被织入切面后生成的新对象。作为目标对象的包装，代理对象将切面逻辑与目标对象的业务逻辑结合起来。它封装了目标对象的所有方法，并在方法调用时执行相应的切面增强逻辑（通知）。

---


# 二、实操

### 1、基本开发步骤

#### 1.1、前言：配置方式 概述

在 Spring AOP 中，通常有以下三种方法进行配置：

1. ==基于注解的 AOP==：结合 Spring 框架和 AspectJ 框架，通过注解来定义切面和切点。
2. ==基于 XML 的 AOP==：同样结合 Spring 和 AspectJ，通过 XML 配置来定义切面和切点。
3. ==Spring 自有的 AOP 实现==：基于 XML 配置，使用 Spring 自身的 AOP 功能。

在实际开发中，通常使用 Spring + AspectJ 的组合，我们学习第一种。

---

#### 1.2、创建 Spring AOP 项目

在创建 Spring IOC 项目（引入 [Spring Context 依赖](https://mvnrepository.com/artifact/org.springframework/spring-context)）的基础上，再引入 [Spring AOP 依赖](https://mvnrepository.com/artifact/org.springframework/spring-aop)、[Spring Aspects 依赖](https://mvnrepository.com/artifact/org.springframework/spring-aspects)、[Aspectj Weaver 依赖](https://mvnrepository.com/artifact/org.aspectj/aspectjweaver)
```
<dependencies>

    <!-- 1. Spring Context 依赖 -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <version>xxxxxx</version>
    </dependency>
    
    <!-- 2. Spring AOP 核心依赖 -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-aop</artifactId>
        <version>xxxxxx</version>
    </dependency>

    <!-- 3. Spring AOP 的注解支持 -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-aspects</artifactId>
        <version>xxxxxx</version>
    </dependency>

    <!-- 4. AspectJ 语言支持 -->
    <dependency>
        <groupId>org.aspectj</groupId>
        <artifactId>aspectjweaver</artifactId>
        <version>xxxxxx</version>
    </dependency>
    
</dependencies>
```

---


#### 1.3、找到目标对象
```
// 目标类
public class UserService {
    // 目标方法
    public void createUser(String username) {
        System.out.println("User created: " + username);
    }
}
```

---


#### 1.4、创建切面，定义切点、通知
``` 
@Aspect // 在 Java 中，一个切面就是一个 @Aspect 标注的切面类
@Order(1) // 规定在多个切面中，本切面的执行顺序，数字越小越先执行
public class LoggingAspect {
    ......
}
```

---


#### 1.5、将目标对象、切面声明为 Bean

==1.目标对象声明为 Bean==
```
// 目标类
@Component
public class UserService {
    public void createUser(String username) {
        System.out.println("User created: " + username);
    }
}
```

==2.切面对象声明为 Bean==
```
@Component
@Aspect // 在 Java 中，一个切面就是一个 @Aspect 标注的切面类
@Order(1) // 规定在多个切面中，本切面的执行顺序，数字越小越先执行
public class LoggingAspect {
    ......
}
```

> [!NOTE] 疑问：为什么要将目标对象、切面声明为 Bean
> 1. 使得 Spring AOP 能够在运行时自动为符合切点条件的 Bean 生成代理对象，从而实现 AOP 功能的自动织入。

---


#### 1.6、启用 AspectJ 自动代理功能

在 Spring IoC 的配置类中，通过使用 `@EnableAspectJAutoProxy` 注解来启用 AspectJ 的自动代理功能。
``` 
// 配置类
@Configuration
@EnableAspectJAutoProxy // 启用 AspectJ 自动代理
public class ApplicationContextConfig {
    // 配置类内容
}
```

---


### 2、Boot 简化开发

