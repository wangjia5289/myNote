---
title: 笔记：Spring Data MyBatis
date: 2025-04-10
categories:
  - 数据管理
  - 关系型数据库
  - MySQL
  - Spring Data MyBatis
tags: 
author: 霸天
layout: post
---
# 一、理论

### 1. 导图：[Map：Spring Data MyBatis](Map：SpringDataMyBatis.xmind)

---


### 2. MyBatis 概述

在传统的 JDBC 编程中，SQL 语句与 Java 代码紧密结合，导致代码结构复杂且难以维护。而 MyBatis 是一个用于简化数据库操作的持久层框架，它将 SQL 语句与 Java 代码分离，**通过 XML 或注解的方式将 SQL 语句与 Java 方法映射在一起**。

与 Hibernate 等 ORM 框架相比，MyBatis 更加灵活，适合对 SQL 有更高要求的场景。

---


### 3. MyBatis 核心 API

#### 3.1. 核心 API 种类

MyBatis 中的核心 API 有：SqlSessionFactoryBuilder  ->  SqlSessionFactory  ->  SqlSession -> Mapper 接口代理对象
```
// 1. SqlSessionFactoryBuilder
SqlSessionFactoryBuilder sqlSessionFactoryBuilder = new SqlSessionFactoryBuilder();


// 2. SqlSessionFactory
// 2.1. 读取 MyBatis 全局配置文件
InputStream is = Resources.getResourceAsStream("mybatis-config.xml"); 

// 2.2. 以默认环境创建 sqlSessionFactory
SqlSessionFactory sqlSessionFactory = sqlSessionFactoryBuilder.build(is);

// 2.3. 以指定环境创建 sqlSessionFactory
SqlSessionFactory sqlSessionFactory = sqlSessionFactoryBuilder.build(is,"环境id");

																	 
// 3. SqlSession
// 3.1. 以关闭自动提交模式，开启事务管理的方式
SqlSession sqlSession = sqlSessionFactory.openSession();              

// 3.2. 以自动提交的方式
SqlSession sqlSession = sqlSessionFactory.openSession(true);


// 4. Mapper 接口代理对象
UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
```

---


#### 3.2. SqlSessionFactoryBuilder

`SqlSessionFactoryBuilder` 的主要职责是通过调用 `build` 方法读取 MyBatis 的全局配置文件，基于这些配置信息创建并返回一个 `SqlSessionFactory` 实例。

简单来说，`SqlSessionFactoryBuilder` 会加载整个配置文件，并根据文件中指定的某个环境信息，创建一个相应的 `SqlSessionFactory` 对象。

> [!NOTE] 注意事项
> 1. `SqlSessionFactory` 实例创建完成后，`SqlSessionFactoryBuilder` 实例就没有什么用处了。
> 2. `SqlSessionFactoryBuilder` 是不可重用的，每次需要创建一个新的 `SqlSessionFactory` 时，都必须实例化一个新的 `SqlSessionFactoryBuilder`。
> 3. 通常情况下，应用程序只需要在启动时创建一个 `SqlSessionFactory` 实例，然后在整个应用程序的生命周期中重用该实例。`SqlSessionFactory` 是线程安全的，并且可以在多线程环境下重用，因此不需要为每个数据库操作创建新的 `SqlSessionFactory` 实例。

---


#### 3.3. SqlSessionFactory

一个环境对应一个 `SqlSessionFactory` 对象。 `SqlSessionFactory` 的主要职责是通过调用 `openSession`方法生成的 `SqlSession` 实例。

> [!NOTE] 注意事项
> 1. `SqlSessionFactory` 是线程安全的，通常在应用程序启动时创建一个实例，并在整个应用程序的生命周期中共享。多个线程可以共享同一个 `SqlSessionFactory` 实例，无需担心并发问题。

---


#### 3.4. SqlSession

`SqlSession` 代表与数据库的一次会话，负责在这次会话期间执行数据库操作，其主要职责是：
1. ==执行 SQL 语句==：
	1. 调用 `SqlSession` 的实例方法可以执行各种 SQL 语句
	2. 例如：`inset()、delete()、update()、selectOne()、selectList()、selectMap()`
	3. 需要注意的是，通常我们并不直接使用 `SqlSession` 来执行 SQL 语句，而是通过获取代理对象来间接执行 SQL。实际上，代理对象在内部仍然调用了这些方法来执行 SQL 语句。
2. ==获取代理对象==：
	1. 获取某个 Mapper 接口的代理对象
3. ==控制事务管理==：
	1. 在创建 `SqlSession` 时，如果传入参数 `true`，表示该 `SqlSession` 将采用手动事务管理模式。在这种模式下，事务的提交和回滚需要由开发者显式控制，可以使用 `commit()` 和 `rollback()` 方法进行事务管理。如果未传入参数 `true` ，代表是自动提交事务，需要我们自己控制提交和回滚。
	2. 需要注意的是，上面的描述仅限于 `<transactionManager type="JDBC">`，而对于 `Spring Data MyBatis`，我们一般是使用 `@Transactional` 进行声明式事务管理，或者使用 `PlatformTransactionManager` 进行编程式事务管理
4. ==资源管理==

> [!NOTE] 注意事项
> 1. `SqlSession` 不是线程安全的，不能在多个线程中共享同一个 `SqlSession` 实例。每个线程都应该独立获取 `SqlSession`，确保并发操作的安全性。

---


#### 3.5. Mapper 接口的代理对象

Mapper 接口的代理对象是 MyBatis 为 Mapper 接口自动生成的实现类。以前，我们需要手动实现 `Dao` 接口并生成 `DaoImpl` 类，然后通过 `DaoImpl` 对象调用方法执行 SQL。现在，我们不再需要手动实现 `Dao`，MyBatis 会自动生成并提供代理对象，直接用于执行 SQL。

---


### 4. MyBatis 中的动态代理

在传统的 JavaWeb 开发中，Dao 层通常需要先声明一个 `Dao` 接口，然后创建 `DaoImpl` 来实现该接口。

而 MyBatis 通过动态代理机制简化了这一过程。你只需要定义一个 `Mapper` 接口和对应的 SQL 语句，无需手动实现该接口。MyBatis 会自动生成 `Mapper` 接口的实现类，并通过动态代理在运行时生成代理对象。

当你通过 `Mapper` 接口的代理对象调用方法时，实际上是由 MyBatis 实现类中的相应方法处理这些调用。看似直接执行 SQL 查询，实际上是通过实现类的方法来间接执行的。

---


### 5. MyBatis 事务管理（原生）

在 MyBatis 中，即使你使用了 Mapper 接口的代理对象来调用 SQL 语句，但是关于事务管理这一方面，还是要由 `SqlSession` 进行管理

注意，在原生 MyBatis 开发中，我们确实要这样进行事务管理，但是在 `Spring Data MyBatis` 中，我们要使用 Spring 提供的事务管理
```
# 1. 声明 SqlSession 时，不传递 true，代表手动管理事务
SqlSession sqlSession = sqlSessionFactory.openSession();


# 2. 提交事务
sqlSession.commit();


# 3. 回滚事务
sqlSession.rollback();
```

---


### 6. Spring 提供的事务管理

#### 6.1. Spring 事务管理机制概述

Spring 事务管理提供了两种主要方式：
1. ==声明式事务管理（推荐）==：
	1. 基于 AOP 代理机制，在方法执行前后自动进行事务管理，是比较推荐的一种方式
2. ==编程式事务管理==：
	1. 编程式事务管理允许开发者手动控制事务的创建、提交和回滚。在一些特殊情况下，声明式事务管理不适用时，可以选择使用编程式事务管理。

---


#### 6.2. 声明式事务管理（推荐）

声明式事务通常通过在 `ServiceImpl` 类或方法上添加 `@Transactional` 注解来配置。当 `Controller` 或其他 `ServiceImpl` 调用该方法时，它就成为了一个事务性操作（即 B 方法）。
```
# 1. 在 ServiceImpl 类上使用事务
@Service
public class UserServiceImpl implements UserService {

    @Override
    @Transactional(
            propagation = Propagation.MANDATORY,
            isolation = Isolation.READ_COMMITTED,
            timeout = 30,
            rollbackFor = {SQLException.class, IllegalArgumentException.class},
            noRollbackFor = {IOException.class, TimeoutException.class}
    )
    public void createUser(String username, String password) {
        System.out.println("Creating user with username: " + username);
    }
}


# 2. 其他地方调用事务方法
@RestController
@RequestMapping("/users")
public class UserController {

    @Autowired
    private UserService userService;

    @PostMapping("/create")
    public String createUser(@RequestParam String username, @RequestParam String password) {
        try {
            userService.createUser(username, password);       # 调用事务方法
            return "User created successfully!";
        } catch (Exception e) {
            return "Failed to create user: " + e.getMessage();
        }
    }
}
```

> [!NOTE] 注意事项
> 1. <font color="#00b0f0">@Transactional</font>：
> 	- 声明式事务管理，用于定制事务的行为
> 	- <font color="#7030a0">标注在类上</font>：
> 		- 类中的所有 **public 方法**都会 **继承** 该事务的行为
> 		- 注意：私有方法和非 public 方法不会触发事务
> 	- <font color="#7030a0">标注在方法上</font>：
> 		- 事务只会应用于标注该注解的 **单一方法**，不会影响其他方法。
> 2. <font color="#00b0f0">propagation</font>：
> 	- 事务传播行为，定义了多个事务之间的交互方式
> 	- <font color="#7030a0">Propagation.REQUIRED（默认）</font>：
> 		- 以 A 方法（如 `Controller` 或其他 `ServiceImpl` 中的方法）调用 **B 方法（已标注 `@Transactional`的本方法）**为例。
> 		- 如果 A 已有事务（也被 `@Transactional` 标注），B 会加入 A 的事务，B 失败时 **A 的事务回滚**
> 		- 如果 A 没有事务，B 会创建新的事务，B 失败时 **只回滚 B 的事务**，A 继续执行
> 	- <font color="#7030a0">REQUIRES_NEW</font>：
> 		- 不管 A 是不是已经有事务，B 总会创建新的事务，A 的事务会被挂起，直到 B 完成才恢复
> 		- 如果 B 失败，B 的事务回滚，A 不受影响，A 继续执行并提交事务
> 	- <font color="#7030a0">SUPPORTS</font>：
> 		- 如果 A 已有事务，B 会加入 A 的事务，B 失败时 **不会影响 A 的事务**（与 `REQUIRED` 的区别）
> 		- 如果 A 没有事务，B 会以非事务方式执行，B 失败时只回滚自己的事务，A 不受影响
> 	- <font color="#7030a0">NOT_SUPPORTED</font>：
> 		- 不管 A 是不是已经有事务，B 都会以非事务方式执行，A 的事务会被挂起，直到 B 完成才恢复
> 		- 如果 B 失败，B 的事务回滚，A 不受影响，A 继续执行并提交事务
> 3. <font color="#00b0f0">isolation</font>：
> 	- 指定事务隔离级别，主要是针对相同资源（如数据库中的行、表等）的并发操作时，事务之间该如何隔离
> 	- <font color="#7030a0">Isolation.READ_UNCOMMITTED</font>：读未提交
> 	- <font color="#7030a0">Isolation.READ_COMMITTED</font>：读已提交，默认
> 	- <font color="#7030a0">Isolation.REPEATABLE_READ</font>：可重复读
> 	- <font color="#7030a0">Isolation.SERIALIZABLE</font>：可串行化
> 4. <font color="#7030a0">timeout</font>：
> 	- 事务的最大执行时间，单位为秒（`S`）。默认值为 `-1`，表示没有时间限制。
> 	- 如果事务在指定时间内未完成，Spring 会自动回滚事务并抛出 `TransactionTimedOutException` 异常。
> 	- 设置超时的主要原因是防止多个事务竞争同一资源时可能出现死锁。通过设置超时，可以在死锁发生时及时回滚事务，避免死锁长时间影响系统
> 	- 此外，超时设置还可以帮助避免由于网络延迟导致的事务执行超时问题
> 5. <font color="#7030a0">rollbackFor</font>：
> 	- 默认情况下，Spring 只会回滚运行时异常（`RuntimeException`）和 `Error` 类型的异常。其他异常类型（如 `checked exceptions`）不会导致事务回滚。
> 	- 通过 `rollbackFor` 可以扩展这一行为，指定哪些异常会导致事务回滚
> 6. <font color="#7030a0">noRollbackFor</font>：
> 	- 与 `rollbackFor` 相对，指定哪些异常不会导致事务回滚

---


#### 6.3. 编程式事务管理

编程式事务管理允许开发者手动控制事务的创建、提交和回滚。在一些特殊情况下，声明式事务管理不适用时，可以选择使用编程式事务管理。

Spring 提供了 `PlatformTransactionManager` 接口及其实现类来手动管理事务。

```
@Service  
public class UserService {  
      
    @Autowired  
    @Qualifier("transactionManager1")        // 我们为每个数据源都配置的 PlatformTransactionManager
    private PlatformTransactionManager transactionManager;  
    
    public void createUser(User user) {  
    
        // 创建事务定义  
        DefaultTransactionDefinition def = new DefaultTransactionDefinition();  
        def.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);  
  
        // 开始事务  
        TransactionStatus status = transactionManager.getTransaction(def);  
        
        try {  
            // 业务逻辑  
            userRepository.save(user);  
  
            // 提交事务  
            transactionManager.commit(status);  
        } catch (Exception e) {  
            // 回滚事务  
            transactionManager.rollback(status);  
            throw e;  // 重新抛出异常  
        }  
    }  
}
```

---


### 7. Spring 提供的异步操作


---


# 二、实操

### 1. 基本使用

#### 1.1. 基本使用方法概述

简单来说，就是我们先配**置数据源**，一个 IP 下的一个数据库就对应一个数据源。接着，在这个数据库中，我们需要对哪些表进行增删改查，就为每张表分别编写一个对应的 **Pojo 类**。

然后，通过 **Mapper 接口**来定义对表的操作。一个 Pojo 类对应一个 Mapper 接口，在接口中声明操作该表的所有方法。

写完方法之后，就需要**编写对应的 SQL** 语句，方式有两个：
1. 通过注解在 `Mapper` 接口中直接编写 SQL 语句，尽管这种方式简便，但缺乏可维护性并且不支持复杂的映射规则和 SQL 语句
2. 通过 `Mapper XML` 文件编写 SQL 语句，并且可以设置详细的映射规则，提供了更强的灵活性，支持复杂的映射规则和 SQL 语句（一个 Mapper 接口对应一个 Mapper XML 映射文件）

接下来，我们需要**注册 SQL**，即建立 SQL 与 Mapper 方法之间的映射关系。

最后，通过 MyBatis **创建 Mapper 接口的代理对象**，我们**调用这个代理对象**的方法，其实就是在执行与方法绑定的 SQL 语句。

---


#### 1.2. 创建 Spring Web 项目，引入 MyBatis 相关依赖

这里采用 IDEA 提供的脚手架创建 Spring Boot 项目，分别勾选：
1. ==Web==
	1. Spring Web
2. ==SQL==
	1. JDBC API
	2. MyBatis Framework
	3. MySQL Driver

---


#### 1.3. 进行 MyBatis 相关配置

##### 1.3.1. 通用配置

```
# 1. 精简版
mybatis:  
  configuration:  
    map-underscore-to-camel-case: true
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl

spring:  
  datasource:  
    datasource1:  
      jdbc-url: jdbc:mysql://192.168.136.7:3306/repository1?useUnicode=true&characterEncoding=utf8&serverTimezone=UTC  
      username: root  
      password: wq666666
      driver-class-name: com.mysql.cj.jdbc.Driver

    datasource2:  
      jdbc-url: jdbc:mysql://192.168.136.7:3306/repository2?useUnicode=true&characterEncoding=utf8&serverTimezone=UTC  
      username: root  
      password: wq666666
      driver-class-name: com.mysql.cj.jdbc.Driver


# 2. 注释版
mybatis:  
  configuration:  
    map-underscore-to-camel-case: true                         // 开启自动驼峰命名映射
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl      // 控制台输出 SQL 日志
spring:  
  datasource:  
    datasource1:                                               // 数据源1（名称自定义）
      jdbc-url: jdbc:mysql://192.168.136.7:3306/repository1?   // 数据源位置，精确到库           useUnicode=true&characterEncoding=utf8&serverTimezone=UTC  
      username: root                                           // MySQL 用户
      password: wq666666                                       // MySQL 用户密码
      driver-class-name: com.mysql.cj.jdbc.Driver              // MySQL 8.0 以上的 JDBC 驱动
    datasource2:                                               // 数据源2（名称自定义）
      jdbc-url: jdbc:mysql://192.168.136.7:3306/?repository2?useUnicode=true&characterEncoding=utf8&serverTimezone=UTC  
      username: root  
      password: wq666666
      driver-class-name: com.mysql.cj.jdbc.Driver  
"""
1. jdbc:mysql://192.168.136.7:3306/repository1?useUnicode=true&characterEncoding=utf8&serverTimezone=UTC  
	1. jdbc:mysql
		1. 使用 JDBC 连接到 MySQL
	2. repository1
		1. 使用的库的名称，让我们无需 use <repository-name>
	3. useUnicode=true
		1. 表示启用 Unicode 字符集支持。
	4. characterEncoding=utf8
		1. 指定数据库连接的字符编码方式为 UTF-8
	5. serverTimezone=UTC  
		1. 指定 MySQL 服务器的时区为 UTC（协调世界时）
"""
```

---


##### 1.3.2. 配置类配置

==1.数据源 1 配置类==
注意三处：Mapper 接口位置、Mapper XML 位置、Model Entity Pojo 位置
```
@Configuration                                // 表明是一个配置类
@MapperScan(basePackages = "com.example.spring_data_mybatis.mapper.datasource1", sqlSessionFactoryRef = "sqlSessionFactory1")  // 扫描 Mapper 接口（生成代理对象，注册 SQL 与 Mapper 之间的映射（注册方式 2）），指定 sqlSessionFactory
@EnableTransactionManagement                  // 启用 Spring 的声明式事务管理
public class DataSource1Config {              // DataSource1 的配置类

    @Primary
    @Bean(name = "dataSource1")
    @ConfigurationProperties(prefix = "spring.datasource.datasource1") 
    public DataSource dataSource1() {         // 定义了一个名为 “dataSource1” 的数据源 Bean
        return DataSourceBuilder.create().build();
    }

    @Primary
    @Bean(name = "sqlSessionFactory1")
    public SqlSessionFactory sqlSessionFactory1(@Qualifier("dataSource1") DataSource dataSource) throws Exception {                            // 定义了本数据源的 SqlSessionFactory
    
        SqlSessionFactoryBean sessionFactory = new SqlSessionFactoryBean();
        
        sessionFactory.setDataSource(dataSource);
        
        sessionFactory.setMapperLocations(new PathMatchingResourcePatternResolver()  
        .getResources("classpath*:mapper/dataSource1/*.xml"));      // DataSource1 Mapper XML 的位置，注册 SQL 与 Mapper 之间的映射（注册方式2）
        
		sessionFactory.setTypeAliasesPackage(
		"com.example.spring_data_mysql.model.entity.datasource1");  // DataSource1 Entity Pojo 的位置，为全类名起类型别名
		
        return sessionFactory.getObject();
    }

    @Primary
    @Bean(name = "transactionManager1")
    public PlatformTransactionManager transactionManager1(@Qualifier("dataSource1") DataSource dataSource) {                                 // 定义了本数据源的 事务管理器
        return new DataSourceTransactionManager(dataSource);
    }
}
```

> [!NOTE] 注意事项
> 1. 每个数据源（`dataSource1`、`dataSource2`）都需要创建**独立的**配置类，并进行相应的配置
> 2. 如果 SQL 写在 Mapper XML 中，则必须使用 `setMapperLocations` 来注册 SQL 与 Mapper 的映射。
> 3. 如果 SQL 是通过注解写在 Mapper 接口中的，则需要使用 `@Mapper` 注解来扫描接口，这样不仅可以生成接口的 代理对象，还能够自动注册 SQL 与 Mapper 之间的映射，**实现一举两得**。
> 4. 无论 SQL 写在哪里，都**必须使用** `@MapperScan` 来扫描 `Mapper` 接口，确保生成对应的代理对象


==2.数据源 2 创建配置类==
```
@Configuration  
@MapperScan(basePackages = "com.example.spring_data_mybatis.model.entity.datasource2",  
        sqlSessionFactoryRef = "sqlSessionFactory2")  
@EnableTransactionManagement  
public class DataSource2Config {  
  
    @Bean(name = "dataSource2")  
    @ConfigurationProperties(prefix = "spring.datasource.datasource2")  
    public DataSource dataSource2() {  
        return DataSourceBuilder.create().build();  
    }  
  
    @Bean(name = "sqlSessionFactory2")  
    public SqlSessionFactory sqlSessionFactory2(@Qualifier("dataSource2") DataSource dataSource) throws Exception {  
        SqlSessionFactoryBean sessionFactory = new SqlSessionFactoryBean();  
        sessionFactory.setDataSource(dataSource);  
        sessionFactory.setMapperLocations(new PathMatchingResourcePatternResolver()  
                .getResources("classpath*:mapper/dataSource2/*.xml"));  
        sessionFactory.setTypeAliasesPackage("com.example.spring_data_mysql.model.entity.datasource2");  
        return sessionFactory.getObject();  
    }  
  
    @Bean(name = "transactionManager2")  
    public PlatformTransactionManager transactionManager2(@Qualifier("dataSource2") DataSource dataSource) {  
        return new DataSourceTransactionManager(dataSource);  
    }  
}
```

---


##### 1.3.3. 连接池配置

Spring Boot 默认使用 **HikariCP** 作为连接池，其默认配置已经能够满足大多数场景下的性能和高可用性要求。如果在特定情况下有特殊需求，也可以根据实际情况自定义连接池配置。

---


#### 1.4. 编写 Model Entity Pojo 类

在 ORM（对象关系映射）中，Entity Pojo 类用于承载和传递与业务相关的数据。通过 Entity Pojo类，我们可以将查询结果映射到类的属性，或者将类的属性保存到数据库中。

每个 Entity Pojo 类对应一个数据库表，充当该表的数据载体。一个 Entity Pojo 对象对应数据库中的一行数据，类中的属性对应表中的列字段。

与普通的 Pojo 类不同，在 ORM 中，Entity Pojo 类通常仅包含基本的私有属性、最基本的构造方法、setter、getter、toString 等方法。它不得包含复杂的业务逻辑或操作，避免引入额外的逻辑。
```
public class User {

    // 1. 私有属性
    private Integer id;
    private String firstName;
    private String lastName;
    private String email;

  	// 2. 构造方法（无参构造、有参构造）
    
    // 3. Getter 方法
    
    // 4. Setter 方法
    
    // 5. toString 方法
}
```

> [!NOTE] 注意事项
> 1. Entity Pojo 类名应与数据库表名对应，以便直观且方便地进行维护。例如：`t_car` 对应 `Car`，`car` 对应 `Car`。
> 2. 对于私有属性，建议使用包装类进行声明，因为包装类能够更好地处理 `null` 值
> 3. 私有属性的名称应与数据库表中的列名对应，通常采用驼峰命名法。例如：`first_name` 对应 `firstName`，`email` 对应 `email`
> 4. 对于 Entity Pojo 类，一般不需要也不应该加 `@Component` 注解，因为他们仅仅只是数据载体

---


#### 1.5. 编写 Mapper 接口

##### 1.5.1. Mapper 接口概述

我们将在 Mapper 接口中定义与 Pojo 所对应的表相关的**操作方法**（如查询、插入、更新、删除等）。

每个表对应一个 Pojo，每个 Pojo 又对应一个 Mapper 接口。

Mapper 接口的作用类似于传统的 Dao 接口，通常位于 `mapper` 或 `dao` 包下。
```
@Mapper                                           // 表示 MyBatis 的 Mapper 接口
@Repository
public interface UserMapper {  
    List<User> getUserById(@Param("id") int id);  
  
    List<User> getAllUsers();  
}
```

> [!NOTE] 注意事项
> 1. Mapper 接口的命名通常采用 `Entity Pojo + Mapper` 的形式，例如：`t_user` 对应 `User`，则 Mapper 接口命名为 `UserMapper`
> 2. `@Mapper` 别忘记加入，这样当我们使用 `@MapperScan` 扫描该类的时候，会自动创建代理对象注入到 IOC 容器中（无需再额外使用 `@Repository` 标注了）
> 	1. `@Mapper` + `@Repository`
> 	2. `@Mapper` + `@MapperScan`（推荐，省事）

---


##### 1.5.2. Mapper 接口定义方法

###### 1.5.2.1. 方法名

方法名称的命名应确保清晰且易于维护，通常以动词开头，以明确表示所执行的操作。例如：
1. `insertUser`
2. `updateUser`
3. `deleteUser`
4. `selectUserById`

> [!NOTE] 注意事项：务必注意
> 1. 如果是使用 Mapper XML 书写 SQL 语句，务必保证**方法的名称**和 **SQL 的 ID** **一致**

---


###### 1.5.2.2. 返回类型

1. ==DML 语句==：
	1. 使用 `int` 作为返回类型
	2. 返回的是本次操作所影响的行数
2. ==DQL 语句==：
	1. <font color="#00b0f0">统计查询（聚合函数</font>）：
		1. 使用 `int` 作为返回值
	2. <font color="#00b0f0">查询结果有 Pojo 载体</font>：
		1. 使用 `List<Pojo>` 作为返回值
		2. `List<Pojo>` 中的每个 Pojo 对象都代表一行数据
	3. <font color="#00b0f0">查询结果无 Pojo 载体</font>：
		1. 使用 `List<Map<String,Object>>` 作为返回值

---


###### 1.5.2.3. 参数类型

参数类型可以根据实际需求灵活选择，Java 中的所有数据类型均可作为参数传入。

无论方法中需要传递多少个参数，我们都**推荐为每个参数显式使用 `@Param` 注解**。这是 MyBatis 提供的注解，用于为传入 SQL 的参数命名，以确保 SQL 语句中能够准确引用对应参数，提升可读性和可维护性。
```
@Mapper
public interface UserMapper {

    User findUserByIdAndName(@Param("id") int id, @Param("name") String name);

}
```

需要注意的是：**在 MyBatis 中，方法传入的参数会被封装为类似 Map 的结构，然后传递给对应的 SQL 语句**。这个 Map 中的每个键名，默认是由 MyBatis 自动生成的（例如 `param1`、`param2` 等），不仅不直观，而且不利于后期维护。

这就是我们**推荐为每个参数显式使用 `@Param` 注解**的原因。通过 `@Param`，我们可以清晰地为每个参数指定名称，确保 SQL 语句中能够准确引用，提升代码可读性。

例如上面这个例子，传递到 SQL 层时，参数结构大致为：
```
{
  "id": 1,
  "name": "张三"
}
```

如果方法中包含多个集合参数，比如两个 `List` 类型的集合，参数结构大致为：
```
{
  "users": [                                                 // List<User> 
    {"id": 1, "name": "张三"},
    {"id": 2, "name": "李四"}
  ],
  "products": [                                              // List<Product>
    {"id": 101, "carName": "宝马"},
    {"id": 102, "carName": "奔驰"}
  ]
}
```

> [!NOTE] 注意事项
> 1. 只在 Mapper 接口中需要标注 `@Param`，其他地方不用标注

---


#### 1.6. 编写 SQL 语句

##### 1.6.1. 编写 SQL 语句概述

虽然我们可以在两个地方编写 SQL 语句，但是我们推荐使用 XML 方式编写 SQL语句：
```
<!DOCTYPE mapper  
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"  
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">  
<mapper namespace="com.example.spring_data_mybatis.mapper.datasource1.UserMapper">
    <select id="getUserById" resultType="User" >      // 标红不用管
        select * from user where id = #{id}    
	</select>  
	
    <select id="getAllUsers" resultType="User">  
        select * from user    
	</select>  
</mapper>
```

---


##### 1.6.2. SQL 语句编写规则

###### 1.6.2.1. 书写位置

在 MyBatis 中，SQL 语句与 Java 代码是分离的，SQL 语句可以通过以下两种方式进行定义：

1. 通过注解在 `Mapper` 接口中直接编写 SQL 语句，尽管这种方式简便，但缺乏可维护性并且不支持复杂的映射规则和 SQL 语句
2. 通过 `Mapper XML` 文件编写 SQL 语句，并且可以设置详细的映射规则，提供了更强的灵活性，支持复杂的映射规则和 SQL 语句（一个 Mapper 接口对应一个 Mapper XML 映射文件）

无论选择哪种方式，我们都需要注册 SQL 与 Mapper 之间的映射（`mapper-locations` 或 `@MapperScan`），并且扫描 Mapper 接口，让 MyBatis 为其生成代理对象（`@MapperScan`）

> [!NOTE] 注意事项
> 1. 还是推荐在 `Mapper XML` 中编写 SQL 语句


---


###### 1.6.2.2. 书写要求

1. 虽然你可以在 `@Mapper` 接口或 XML 映射文件中书写**任意类型**的 SQL（包括 DDL、DCL 等），但 MyBatis 的核心定位是数据操作层，主要聚焦在 **DQL（查询）和 DML（增删改）** 上。其他类型的语句虽然技术上可行，但 不推荐使用，建议**直接规避**此类用法。
2. 编写 SQL 时，**请勿在语句末尾添加分号**（`;`）

---


###### 1.6.2.3. 占位符传值

1. ==#{parameter-name}==：
	1. MyBatis 会对参数值进行预处理，确保参数被正确转义（例如：字符串 `"asc"` 被转义为 `'asc'`，字符 `'男'` 保持为为 `'男'`，数字 `1` 保持为 `1`），从而有效防止 SQL 注入攻击。通常用于查询条件、字段赋值等场景
2. ==${parameter-name}==：
	1. 用于动态拼接 SQL 语句，将参数值直接嵌入 SQL 中，不做任何预处理（例如：字符串 `"asc"` 为 `asc`，字符 `'男'` 为 `男`，数字 `1` 为 `1`）。这种方式存在 SQL 注入的风险，因此严禁使用用户输入的数据作为 `${}` 的参数
```
// 1. 定义 SQL 语句
<select id="findUserByIdAndName" resultType="User">
    SELECT * FROM user WHERE id = #{id} AND name = #{name} 
</select>



// 2. 传递参数值
public interface UserMapper {
    User findUserByIdAndName(@Param("id") int id, @Param("name") String name);
}
```

> [!NOTE] 注意事项：
> 1. 推荐使用 `@Param` 为每个参数指定 `param-name`，以提高代码可读性和可维护性。
> 2. `@Param("id")` 中的名称应与 SQL 语句中的 `#{id}` 保持一致，确保正确映射
> 3. 支持传递 `List`、`Map`、`Pojo` 等参数类型。如果是单个对象，直接使用其键或属性；如果是多个对象，则需要使用 `#{param-name.xxx}` 来指定具体那个对象的键或属性
```
// 1. 定义 SQL 语句
<select id="findUserByCriteria" resultType="User">
    SELECT * FROM user WHERE id = #{user.id} AND address = #{address.city}
</select>


// 2. 传递参数值
public interface UserMapper {
    User findUserByCriteria(@Param("user") User user, @Param("address") Address address);  
}
```

---


###### 1.6.2.4. 动态 SQL

==1.动态 SQL 概述==

我们在定义 SQL 语句时，通常会使用占位符（如 `#{name}`）来传值。但这时就需要考虑几种常见的情况：
1. 在批量处理的时候，可能需要 N 组占位符：例如 `INSERT INTO user (name, age, email) VALUES (#{name1}, #{age1}, #{email1}),(#{name2}, #{age2}, #{email2}),(#{name3}, #{age3}, #{email3});` 真能手动一个一个传 `name1、name2、...` 吗？不太现实。
2. 再比如，我们定义了两个参数 `#{name}` 和 `#{age}`，但调用时只传了 `name`，如果直接执行这个 SQL 就会报错。那怎么办？

这就引出了我们的动态 SQL，原生 MySQL 并不支持动态 SQL，但 MyBatis 提供了非常强大的动态 SQL 标签机制，帮我们优雅地解决这些问题。


==2.if：逐个判断==
逐个检查每个 `<if>` 语句，只要条件成立，就会进行拼接。
```
<insert id="insertUser">
    INSERT INTO user (name, age)
    VALUES
    <if test="name != null">#{name},</if>
    <if test="age != null">#{age}</if>
</insert>
```

> [!NOTE] 注意事项
> 1. 动态 SQL 系列的功能远不止判断某个字段是否为 null。我们不应该把它简单理解为仅用于“!= null”的判断，而应将其看作一种强大的条件拼接工具 —— 只要条件成立，就能灵活地动态生成 SQL 语句。
```
<select id="selectByUsernameAndEmailLike" resultType="com.macro.mall.tiny.model.UmsAdmin">
    select username,
           password,
           icon,
           email,
           nick_name   as nickName,
           note,
           create_time as createTime,
           login_time  as loginTime,
           status
    from ums_admin
    where 1=1
    <if test="username!=null and username!=''">
        and username like concat('%',#{username},'%')
    </if>
    <if test="email!=null and email!=''">
        and email like concat('%',#{email},'%')
    </if>
</select>
```


==3.choose：多个选一==
类似于 `if-else` 结构，可以根据多个条件判断，从中**选择第一个符合条件的**拼接。
```
<select id="findUser" resultType="User">
    SELECT * FROM user where 
        <choose>
            <when test="name != null">WHERE name = #{name}</when>
            <when test="age != null">WHERE age = #{age}</when>
            <otherwise>WHERE status = 'active'</otherwise>
        </choose>
    </where>
</select>
```


==4.trim：添加前后缀==
```
<insert id="insertUser">
    INSERT INTO user
    <trim prefix="(" suffix=")" suffixOverrides=",">
        <if test="name != null">name,</if>
        <if test="age != null">age,</if>
    </trim>
    VALUES
    <trim prefix="(" suffix=")" suffixOverrides=",">
        <if test="name != null">#{name},</if>
        <if test="age != null">#{age},</if>
    </trim>
</insert>
"""
1. prefix="("
	1. 前缀是 (
2. suffix=")"
	1. 后缀是 )
3. suffixOverrides=","
	1. 去掉最后一个多余的 ，
"""
```


==5.foreach：遍历集合==
遍历 数组（`Array`）和 集合（`List、Set、Map`），用于批量操作
```
<insert id="insertUsers">
    INSERT INTO user (name, age)
    VALUES
    <foreach collection="users" item="user" separator=",">
        (#{user.name}, #{user.age})
    </foreach>
</insert>
"""
1. collection="users"：
	1. 需要遍历的集合是 users，这通常是一个 List、Array、Set 的集合
2. item="user"：
	2. 用于指定当前遍历元素的名称。
	3. 每次遍历时，集合中的当前元素（例如 users 集合中的每个 User 对象，User1、User2 等）会被绑定到名为 user 的变量上。在这次遍历中，我们可以通过 user 来访问对象的属性（user.id、user.name 等）。
	4. 需要注意的是，user 只是一个示例名称，你可以根据需要自由命名，如 temp 等都可以。
3. separator:","：
	1. 指定集合元素之间的分隔符
"""
```

---


#### 1.7. 后续操作

接下来，我们需要编写 Service 接口及其实现类，在 ServiceImpl 中调用 Mapper 接口的方法，最后由 Controller 层调用 ServiceImpl 进行业务处理。

---


### 2. 业务处理

#### 2.1. 插入操作

##### 2.1.1. 常规插入

```
<insert id="insertUser" >
    INSERT INTO user (name, age) VALUES (#{name}, #{age})
</insert>
```
> [!NOTE] 注意事项
> 1. 虽然我们可以直接书写任意的 SQL 插入语句，但在处理**批量插入**时，传入的参数往往非常多，不太可能一个个手动传入。而且，涉及**主键回显**的场景时，如果还要单独写一个方法来获取主键，就显得繁琐。因此，我们通常会采用更简洁、统一的方式来同时实现批量插入和主键回显，具体见下文。

---


##### 2.1.2. 批量插入

```
<insert id="insertUsers">
    INSERT INTO user (name, age) VALUES
    <foreach collection="users" item="user" separator=",">
        (#{user.name}, #{user.age})
    </foreach>
</insert>
```

---


##### 2.1.3. 主键回显

```
# 1. 定义主键回显
<insert id="insertUser" useGeneratedKeys="true" keyProperty="id" keyColumn="user_id">
	INSERT INTO user (name, email) VALUES (#{name}, #{email})
</insert>
"""
1. useGeneratedKeys="true"：
	1. 告诉 MyBatis 去启用 自动获取数据库生成的主键。
2. keyProperty="id"：
	1. 告诉 MyBatis 将生成的主键值赋值给 Pojo 对象的 id 属性
3. keyColumn="user_id"：
	1. 显式指定数据库中的主键列名，适用于 Java 对象的主键属性与数据库表中的主键列名不一致的情况。
	2. 如果 keyColumn 与 keyProperty 相同，则可以省略 keyColumn 配置。
	3. 如果前面配置了结果映射 <id column="user_id" property="id" /> ，这里也可以省略
"""


# 2. 使用示例
int rowsAffected = userMapper.insertUser(user);        // 返回受影响的行数
System.out.println(user.getId());                      // 输出主键值
```




#### 2.2. 删除操作


#### 2.3. 修改操作


#### 2.4. 查询操作











##### 2.4.1. 分页查询

虽然可以使用原生 SQL 实现分页查询，但需要手动计算偏移量和总数，使用起来较为繁琐。通过 MyBatis 的分页插件 PageHelper，只需传入页码和每页条数，插件会自动计算并返回分页结果。

==1.引入 MyBatis 分页插件依赖==
引入 [MyBatis 分页插件依赖](https://mvnrepository.com/artifact/com.github.pagehelper/pagehelper-spring-boot-starter)
```
<dependency>
    <groupId>com.github.pagehelper</groupId>
    <artifactId>pagehelper-spring-boot-starter</artifactId>
    <version>2.1.0</version>
</dependency>
```


==2.编写分页方法==
需要注意的是，分页逻辑应写在 Service 层，因为插件是通过拦截 Mapper 方法（如 `umsResourceDao.selectListByCategoryId`）来修改底层 SQL，自动添加 `LIMIT` 和 `COUNT(*)` 语句，我们得再包一层才行。
```
@Override
public PageInfo<UmsResource> page(Integer pageNum, Integer pageSize, Long categoryId) {
    // 1. 告诉 PageHelper 你想要第几页，每页几条
    PageHelper.startPage(pageNum, pageSize);

    // 2. 写你的正常查询逻辑（PageHelper 会拦截这个查询）
    List<UmsResource> resourceList = umsResourceDao.selectListByCategoryId(categoryId);

    // 3. 把结果用 PageInfo 包装，里面有分页的各种信息
    return new PageInfo<>(resourceList);
}
```

---


# 三、工具

### 1. MyBatisX 插件

IDEA 中的 MyBatisX 插件可以帮助我们：一键生成 Entity Pojo、Mapper 接口和 Mapper XML 映射文件

==1.安装 MyBatisX 插件==
![](image-20250413092353572.png)


==2.点击MyBatisX-Generator==
![](image-20250517142511693.png)


==3.配置实体类生成位置==
![](image-20250517142611270.png)

> [!NOTE] 注意事项
> 1.  `base path` 多级要用 `/` 感觉挺奇怪的，其他地方没有问题


==4.配置 Mapper 接口 和 XML 文件生成位置==
![](image-20250413094940754.png)

---


# 四、补充

### 1. 连接池预热

> [!NOTE] 注意事项
> 1. 不同的数据源在第一次使用时，通常都会初始化各自的连接池，因此你会感觉第一次进行操作会很慢，后续操作会很快。这一步也叫做：**连接池预热**

---


### 2. 项目目录结构

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

---


### 3. 原生的 全局配置文件 书写方法

#### 3.1. 全局配置文件概述

MyBatis 全局配置文件，一般命名为：`mybatis-config.xml`，其规定 MyBatis 和那个数据库连接，如何连接，以及注册和处理 SQL 映射

> [!NOTE] 注意事项
> 1. 在原生 MyBatis 开发中，我们确实要使用 `mybatis-config.xml`
> 2.但是如果是 Spring Data MyBatis，我们用不到这个配置文件

---


#### 3.2. 文件基本结构

```
# 头文件
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
PUBLIC "-//mybatis.org//DTD Config 3.0//EN"	
"http://mybatis.org/dtd/mybatis-3-config.dtd">

<configuration>
  // 1. 全局配置
  
  // 2. 属性
  
  // 3. 类型别名
  
  // 4. 插件
  
  // 5. 环境配置
  
  // 6. 映射器

</configuration>
```

---


#### 3.3. 全局配置

全局配置用于配置 MyBatis 的行为和特性，具体有哪些配置，请看：[MyBatis 3 官方文档](https://mybatis.org/mybatis-3/zh_CN/configuration.html#settings)
```  
<settings>

  // 1. 开启驼峰命名规则
  <setting name="mapUnderscoreToCamelCase" value="true"/> 


  // 2. 其他配置项 
  <setting name="配置名" value="对应值">

</settings>
```

---


#### 3.4. 属性

属性允许在**本文件内**进行复用。如果同时定义了内部属性和外部属性，内部属性的优先级更高。
```
// 1. 声明属性
<properties>

  // 1.1. 声明内部属性
  <property name="property-name" value="property-value"/>
  
  // 1.2. 引用外部属性文件（properties 文件）
  <properties resource="db.properties"/> 	             // 从 resource 中加载
  <properties url="file:/C:/config/db.properties"/>		 // 从 本地文件系统 中加载
  
</properties>


// 2. 文件内复用属性（${property-name}）
<environments default="development">
  <environment id="development">
	  <transactionManager type="JDBC"/>
	  <dataSource type="POOLED">
		  <property name="driver" value="${db.driver}"/>
		  <property name="url" value="${db.url}"/>
		  <property name="username" value="${db.username}"/>
		  <property name="password" value="${db.password}"/>
	  </dataSource>
  </environment>
</environments>
```

---


#### 3.5. 类型别名

在之前的 Mapper XML 映射文件中，我们需要引用 **Entit**y 类的**全类名**，这样显得繁琐。而类型别名的引入，可以为长的全类名起一个简短的别名，使得在 Mapper XML 中引用时更加简便。

需要注意的是，在 Mapper XML 映射文件中，`type` 和 `resultType` **可以使用类的别名**，但 `namespace` **不支持使用别名。**
```
// 1. 逐个定义类别名
// 1.1. 显示声明类别名
<configuration>
  <typeAliases>
	  <typeAlias alias="User" type="com.example.model.User"/>
	  <typeAlias alias="Order" type="com.example.model.Order"/>
  </typeAliases>
</configuration>

// 1.2. 直接使用类名作为类别名
<configuration>
  <typeAliases>
	  <typeAlias type="com.example.model.User"/>
	  <typeAlias type="com.example.model.Order"/>
  </typeAliases>
</configuration>


// 2. 批量定义类别名
<configuration>
  <typeAliases>
	  <package name="com.example.model"/>      // 包下所有类的类别名都是类名
  </typeAliases>
</configuration>
```

---


#### 3.6. 插件

---


#### 3.7. 环境配置

环境配置使我们能够为不同的环境（如开发、测试、生产）设置不同的数据源和事务管理策略。这样，我们就可以通过 `SqlSessionFactoryBuilder` 轻松创建连接到各个数据源的 `SqlSessionFactory`

通常情况下，我们会将环境配置**具体到某 IP 下面的库**，关于数据源的相关配置，请看：[MyBatis 3 官方文档](https://mybatis.org/mybatis-3/zh_CN/configuration.html#environments)
```
<environments default="default-environment-id">
  <environment id="environment-id">
	  // 1. 配置事务管理
	  <transactionManager type="事务管理器类型">
		  // 事务管理器的具体配置项
	  </transactionManager>
	  
	  // 2. 配置数据源
	  <dataSource type="数据源类型">
	  
		  // 2.1. 数据源配置
		  <property name="driver" value="com.mysql.cj.jdbc.Driver"/>
		  <property name="url" value="jdbc:mysql://localhost:3306/mydatabase"/>
          <property name="username" value="root"/>
          <property name="password" value="password"/>
          
          // 2.2. 连接池配置
          <property name="poolMaximumActiveConnections" value="10"/>
	      <property name="poolMaximumIdleConnections" value="5"/>
		  <property name="poolTimeToWait" value="30000"/>
		  
	  </dataSource>
  </environment>
</environments>
```

> [!NOTE] 注意事项
> 1. <font color="#00b0f0">default-environment-id</font>：
> 	- 如果在使用 `SqlSessionFactoryBuilder` 创建 `SqlSessionFactory` 时没有指定环境 id，系统会默认使用该 id 所对应的环境来创建对象。
> 2. <font color="#00b0f0">事务管理器类型</font>：
> 	- <font color="#7030a0">JDBC</font>：
> 		- 直接使用 JDBC 的事务管理方式，适用于简单的应用或对事务控制要求较低的场景
> 		- 在这种模式下，我们可以通过 `SqlSession` 手动控制事务，进行显式提交和回滚。
> 	- <font color="#7030a0">MANAGED</font>：
> 		- 由外部容器管理事务，适用于 J2EE 环境。事务通常由应用服务器或其他外部管理器控制
> 	- <font color="#7030a0">SPRING</font>：
> 		- 集成 Spring 框架的事务管理，适用于使用 Spring 的应用程序，能够充分利用 Spring 的事务管理优势。
> 		- 如果需要 Spring 与 MyBatis 集成，通常不需要额外配置这些，因为 Spring 已经为我们完成了配置。
> 3. <font color="#00b0f0">数据源类型</font>：
> 	- <font color="#7030a0">POOLED</font>：
> 		- 使用 MyBatis 内置的 `PooledDataSource` 连接池管理数据库连接
> 		- MyBatis 在启动时读取配置文件并初始化数据库连接。调用 `SqlSessionFactory.openSession()` 时，`SqlSessionFactory` 会从连接池中获取已有连接，而不是每次都重新建立连接。空闲连接会被保留在池中，以备后续使用。
> 	- <font color="#7030a0">UNPOOLED</font>：
> 		- 不使用连接池，适用于简单应用或测试环境
> 		- 每次创建 `SqlSession` 时，MyBatis 都会建立新的数据库连接，并在会话结束时关闭该连接。每次请求都会创建并销毁连接
> 	- <font color="#7030a0">JNDI</font>：
> 		- 通过 JNDI 查找在容器中查找连接池，通常用于 Web 容器（如 Tomcat 或 JBoss）中，适合生产环境配置。

---


#### 3.8. 映射器

映射器用于**注册 SQL 与 Mapper 之间的映射**，简单理解为将 SQL 语句与 Mappper 接口的方法进行绑定，如果调用了方法，则会执行 SQL 语句。

注册的原则是：SQL 写在哪里，就注册在哪里。如果 SQL 写在 Mapper XML 映射文件中，就注册映射文件；如果 SQL 通过注解写在 Mapper 接口中，则注册 Mapper 接口。
```
<configuration>
  <mappers>
	  <!-- 1. 通过资源文件加载 Mapper XML 文件 -->
	  <mapper resource="com/example/mapper/UserMapper.xml"/>
	  
	  <!-- 2. 通过 URL 动态加载 Mapper XML 文件 -->
	  <mapper url="http://example.com/mapper/UserMapper.xml"/>
	  
	  <!-- 3. 通过 class 加载 Mapper 接口全类名 -->
	  <mapper class="com.example.mapper.UserMapper"/>
	  
	  <!-- 4. 批量加载某包内的所有 Mapper 接口（全类名） -->
	  <package name="com.example.mapper"/>
  </mappers>
</configuration>
```

---


### 4. Mapper XML 映射文件 书写方法

#### 4.1. 映射文件概述

通常情况下，一个数据库表对应一个 Pojo 实体类；每个 Pojo 类对应一个 `Mapper` 接口；而每个 `Mapper` 接口则对应一个 Mapper XML 映射文件，用于定义具体的 SQL 映射关系。

----


#### 4.2. 文件基本结构

```
// 头文件
<!DOCTYPE mapper
PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
"http://mybatis.org/dtd/mybatis-3-mapper.dtd">

// 对应 Mapper 接口的全类名（必须是全类名，不是 Pojo，是 Mapper 接口）
<mapper namespace="com.example.MyMapper">
  // 1. 结果映射
  
  // 2. 插入操作
  
  // 3. 删除操作
  
  // 4. 修改操作
  
  // 5. 查询操作

  // 6. 可复用 SQL 片段
</mapper>
```

---


#### 4.3. 结果映射模版

在**执行查询**时（其他 SQL 没有这个烦恼），如果数据库的列名与 POJO 对象的属性名不一致（例如数据库中的 `user_name` 列与 POJO 中的 `name` 属性），MyBatis 默认情况下无法自动进行映射，这会导致查询结果无法正确赋值给 Java 对象的属性，或者 Java 对象的属性无法正确映射到数据库。

为了解决这个问题，我们通常使用 **结果映射** 来手动指定列名与属性名之间的对应关系。
```
<resultMap id="carResultMap" type="User">                // 映射 User Pojo 对象
  <id column="car_id" property="id" />                   // id 列映射 id 属性
  <result column="car_num" property="carNum" />          // car_num 列映射 carNum 属性
  <result column="guide_price" property="guidePrice" />  // guide_price 列映射 guidePrice 属性
</resultMap>
```

> [!NOTE] 注意事项
> 1. 在定义完结果映射后，可以在后续操作中放心大胆的直接**使用数据库中的列名来编写 SQL 语句**，而不是使用 POJO 的属性名。（千万注意是要以数据库中的列名而不是 POJO 的属性名，这一点非常重要。）
> 2. MyBatis 也提供了**自动驼峰命名映射**，即将带有下划线的数据库列名映射为驼峰命名的 Java 属性名，例如`user_name` 列会被自动映射为 `userName` 属性，无需我们在结果映射中进行复杂映射
```
// 1. 原生 MyBatis 开启自动驼峰命名映射
<setting name="mapUnderscoreToCamelCase" value="true"/>     // 在 mybatis-config.xml 中配置


// 2. Spring Data MyBatis 开启自动驼峰命名映射
mybatis:                                                     // 在 application.yaml 中配置
  configuration:  
    map-underscore-to-camel-case: true
```



太好了，你问的这个是 MyBatis 映射中的一个关键点，直接关乎“一对多关系”如何在实体里表现出来。

---

### 1. `<collection>` 标签是什么？

`<collection>` 是 MyBatis 的 resultMap 配置里的一个标签，用来映射**一对多关联关系**，把数据库查询的多条“多”的数据，自动装配到实体对象的集合属性里。

---

### 2. 你的例子：

```xml
<collection property="authorities" ofType="string" column="authoritie_name" />
```

- **property="authorities"**  
    表示在你的 Java 实体类里，有一个叫 `authorities` 的属性，通常是个 `List` 或 `Set`，用来存放权限集合。
    
- **ofType="string"**  
    表示 `authorities` 集合里存放的元素类型是 `String`，也就是权限名字（比如 `"ROLE_USER"`，`"ROLE_ADMIN"`）。
    
- **column="authoritie_name"**  
    这是 SQL 结果集中对应的列名，也就是说，MyBatis 会从查询结果的 `authoritie_name` 这一列拿数据，一条记录对应一个权限字符串，自动放进 `authorities` 集合里。
    

---

### 3. 为什么要用 `<collection>`？

假设你查一个用户和他所有权限（多条权限）是多条数据库记录，比如：

|user_id|username|...|authoritie_name|
|---|---|---|---|
|1|alice|...|ROLE_USER|
|1|alice|...|ROLE_ADMIN|

如果没有 `<collection>`，MyBatis 默认每条记录会映射成一个 User 对象，数据重复。

有了 `<collection>`，MyBatis 会自动把多条权限合并到一个 User 实例的 `authorities` 集合里，**避免重复，方便后续使用**。

---

### 4. 你对应的实体里 `authorities` 应该是这样：

```java
private List<String> authorities;
```

---

### 总结一句话：

`<collection>` 就是告诉 MyBatis，“这里有个一对多的集合字段，帮我从多行数据的某列提取值，装进实体的集合属性里。”

---

你还想要我帮你写个简单示例代码或流程图，方便理解吗？

































---


#### 4.4. 插入操作模版

```
<insert
  id="uniqueId"                      <!-- 必须：SQL 唯一标识，对应 Mapper 接口的方法（两者一致） -->
  parameterType="parameterType"      <!-- 可选：输入参数的类型 -->
  keyProperty="keyProperty"          <!-- 可选：主键属性（主键回显时使用） -->
  keyColumn="keyColumn"              <!-- 可选：主键列（主键回显时使用）  -->
  useGeneratedKeys="true|false"      <!-- 可选：是否使用自动生成的键（主键回显时使用）  -->
  timeout="timeoutValue">            <!-- 可选：SQL 语句的超时时间 -->
  <!-- 任意的 SQL 插入语句-->
</insert>
```

> [!NOTE] 注意事项：输入参数的类型
> 1. 如果参数是简单类型、包装类，无论传几个参数，我们不用指定 `parameterType`
> 2. 如果是 `entity` 或者其他对象，我们一定要具体到**全限定类名**或**类别名**，例如：`User`
> 3. 其他的还有 array、list、set、map

---


#### 4.5. 删除操作模版

```
<delete
  id="uniqueId"                      <!-- 必须：SQL 唯一标识，对应 Mapper 接口的方法（两者一致） -->
  parameterType="parameterType"      <!-- 可选：输入参数的类型 -->
  timeout="timeoutValue">            <!-- 可选：SQL 语句的超时时间 -->
  
  <!-- 任意的 SQL 删除语句 -->
  
</delete>
```

---


#### 4.6. 修改操作模版

```
<update
  id="uniqueId"                      <!-- 必须：SQL 唯一标识，对应 Mapper 接口的方法（两者一致） -->
  parameterType="parameterType"      <!-- 可选：输入参数的类型 -->
  flushCache="true|false"            <!-- 可选：是否刷新缓存 -->
  timeout="timeoutValue">            <!-- 可选：SQL 语句的超时时间 -->

  <!-- 任意的 SQL 修改语句 -->
  
</update>
```

---


#### 4.7. 查询操作模版

```
<select 
  id="uniqueId"                         <!-- 必须：SQL 唯一标识，对应 Mapper 接口的方法（两者一致） -->
  parameterType="parameterType"         <!-- 可选：输入参数的类型 -->
  resultType="resultType"               <!-- 可选：返回结果的类型（int、Pojo、java.util.Map） -->
  resultMap="resultMapId"               <!-- 可选：自定义结果映射（前面定义的结果映射的 ID） -->
  timeout="timeoutValue">               <!-- 可选：SQL 语句的超时时间 -->
  
  <!-- 任意的 SQL 查询语句 -->
  
</select>
```

---


#### 4.8. 可复用 SQL 片段模版

妈的，这里面也能写 sql，进行简化重复编写
```
<?xml version="1.0" encoding="UTF-8"?>  
<!DOCTYPE mapper  
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"  
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">  
<mapper namespace="com.example.malltinymybatis.mapper.UmsAdminMapper">  
  
    <resultMap id="BaseResultMap" type="com.example.malltinymybatis.model.entity.UmsAdmin">  
            <id property="id" column="id" jdbcType="BIGINT"/>  
            <result property="username" column="username" jdbcType="VARCHAR"/>  
            <result property="password" column="password" jdbcType="VARCHAR"/>  
            <result property="icon" column="icon" jdbcType="VARCHAR"/>  
            <result property="email" column="email" jdbcType="VARCHAR"/>  
            <result property="nickName" column="nick_name" jdbcType="VARCHAR"/>  
            <result property="note" column="note" jdbcType="VARCHAR"/>  
            <result property="createTime" column="create_time" jdbcType="TIMESTAMP"/>  
            <result property="loginTime" column="login_time" jdbcType="TIMESTAMP"/>  
            <result property="status" column="status" jdbcType="INTEGER"/>  
    </resultMap>  
  
    <sql id="Base_Column_List">  
        id,username,password,        
        icon,email,nick_name,        
        note,create_time,login_time,        
        status    
	</sql>  
</mapper>
```

---


### 5. 注解式 SQL 书写方法

#### 5.1. 插入 SQL

##### 5.1.1. 常规插入
```
public interface UserMapper {
    
    @Insert("INSERT INTO user (name, age) VALUES (#{name}, #{age})")
    void insertUser(User user);
}
```

---


##### 5.1.2. 主键回显
```
@Mapper
public interface UserMapper {

    @Insert("INSERT INTO user (name, email) VALUES (#{name}, #{email})")
    @Options(useGeneratedKeys = true, keyProperty = "id", keyColumn = "user_id")
    void insertUser(User user)
    
}
```
1. ==@Insert==：
	1. 执行插入操作的注解
2. ==@Options==：
	2. <font color="#00b0f0">useGeneratedKeys=true</font>:
		1. 告诉 MyBatis 去启用 **自动获取数据库生成的主键**。
	3. <font color="#00b0f0">keyProperty="id"</font>:
		2. 告诉 MyBatis **将生成的主键值赋值给 Pojo 对象的 `id` 属性**
	4. <font color="#00b0f0">keyColumn="user_id"</font>:
		3. 显式指定数据库中的主键列名，适用于 Java 对象的主键属性与数据库表中的主键列名不一致的情况。
		4. 如果 `keyColumn` 与 `keyProperty` 相同，则可以省略 `keyColumn` 配置。

---


#### 5.2. 删除 SQL

```
@Mapper
public interface UserMapper {

    @Delete("DELETE FROM user WHERE id = #{id}")
    int deleteById(int id);
    
}
```

---


#### 5.3. 修改 SQL
```
@Mapper
public interface UserMapper {

    @Update("update t_car set car_num=#{carNum},brand=#{brand},guide_price=#{guidePrice} where id = #{id}")
    int update(Car car);
    
}
```

---


#### 5.4. 查询 SQL
```
@Mapper
public interface CarMapper {

    @Select("select * from t_car where id = #{id}")
	@Results({
		@Result(column = "id", property = "id", id = true),
		@Result(column = "car_num", property = "carNum"),
		@Result(column = "brand", property = "brand"),
})
	User selectUserWithCustomMapping(int id);
}
```


---


### 6. 常用连接池

Spring Boot MyBatis 默认使用 HikariCP 作为连接池，因为其性能最高，关于连接池的配置我们都能在其 README 或文档中找到：
1. Druid：[Druid 官方文档](https://github.com/alibaba/druid/wiki/DruidDataSource配置属性列表) 
2. HikariCP：[HikariCP 官方文档](https://github.com/brettwooldridge/HikariCP?tab=readme-ov-file)
3. C3P0：[C3P0 官方文档](https://github.com/swaldman/c3p0)
4. Spring Boot 配置连接池：[Spring Boot 中配置连接池的属性](https://docs.spring.io/spring-boot/appendix/application-properties/index.html#appendix.application-properties.actuator)

---


### 7. List Pojo 如何处理

```
@Service  
public class UserServiceImpl implements UserService {  
  
    @Autowired  
    private UserMapper userMapper;  
  
  
    @Override  
    public List<User> getUserById(int id) {  
        List<User> users = userMapper.getUserById(id);  
  
        // 终端遍历输出查询到的数据  
        for (User user : users) {  
            System.out.println("User ID: " + user.getId()  
                    + ", FirstNmae: " + user.getFirstName()  
                    + ", LastName: " + user.getLastName());  
        }  
  
        return users;  
    }  
  
    @Override  
    public List<User> getAllUsers() {  
        List<User> users = userMapper.getAllUsers();  
  
        // 终端遍历输出查询到的数据  
        for (User user : users) {  
            System.out.println("User ID: " + user.getId()  
                    + ", FirstNmae: " + user.getFirstName()  
                    + ", LastName: " + user.getLastName());  
        }  
  
        return users;  
    }  
}
```

---


### 8. List <Map<String,Object>> 如何处理

```
@Service
public class UserServiceImpl implements UserService {

    @Autowired
    private UserMapper userMapper;

    @Override
    public List<Map<String, Object>> getUserById(int id) {
        List<Map<String, Object>> users = userMapper.getUserById(id);

        // 终端遍历输出查询到的数据
        for (Map<String, Object> user : users) {
            // 根据列名获取值
            System.out.println("User ID: " + user.get("id") 
                    + ", FirstName: " + user.get("first_name") 
                    + ", LastName: " + user.get("last_name"));
        }

        return users;
    }

    @Override
    public List<Map<String, Object>> getAllUsers() {
        List<Map<String, Object>> users = userMapper.getAllUsers();

        // 终端遍历输出查询到的数据
        for (Map<String, Object> user : users) {
            // 根据列名获取值
            System.out.println("User ID: " + user.get("id")
                    + ", FirstName: " + user.get("first_name") 
                    + ", LastName: " + user.get("last_name"));
        }

        return users;
    }
}
```

# 有意思
之所以 `UsersMapper` 能“开箱即用”而你的 `AuditsCyfhMapper` 会拿不到 Bean，关键就在于接口方法签名和 XML 配置完全对上了：

1. **方法参数与 `parameterType` 一致**
    
    ```java
    // 接口
    Users findUserWithAuthoritieByUserName(String username);
    ```
    
    ```xml
    <!-- XML -->
    <select id="findUserWithAuthoritieByUserName"
            parameterType="string"
            resultMap="BaseResultMap">
      …
    </select>
    ```
    
    这里接口只要一个 `String` 参数，XML 里也声明 `parameterType="string"`，MyBatis 就能顺利为该方法生成代理，Spring 也就能把它当 Bean 注入。
    
2. **返回值和 `resultMap` 匹配**  
    接口返回的是单一的 `Users` 对象，XML 用 `<resultMap>` 把查询列一一映射到 `Users` 的属性上，流程一气呵成。
    

---

### 对比你的 `AuditsCyfhMapper`

- **接口却写成了**
    
    ```java
    Integer insertAuditsCyfh(int userId, String title, String content);
    ```
    
- **XML 却声明**
    
    ```xml
    <insert id="insertAuditsCyfh"
            parameterType="com.example.db.entity.AuditsCyfh">
       …
    </insert>
    ```
    
    这样 MyBatis 就找不到签名匹配的映射——它在 XML 里看不到三个独立参数应该怎么装配到 `AuditsCyfh` 对象上，所以根本不会为 `insertAuditsCyfh(int,String,String)` 生成代理，Spring 在启动时也就没法注入 `auditsCyfhMapper`，导致它在 Controller 里一直是 `null`。
    

---

#### 如何让两者“同样没事”？

你要么

1. **统一成「传实体」的方式**
    
    ```java
    // 接口
    Integer insertAuditsCyfh(AuditsCyfh record);
    ```
    
    ```xml
    <!-- XML -->
    <insert id="insertAuditsCyfh"
            parameterType="com.example.db.entity.AuditsCyfh"
            useGeneratedKeys="true" keyProperty="articleId">
      INSERT INTO Audits_cyfh (user_id,title,content)
      VALUES (#{userId},#{title},#{content})
    </insert>
    ```
    

要么  
2. **改成「独立参数 + map」的方式**

```java
// 接口
Integer insertAuditsCyfh(@Param("userId") int userId,
                         @Param("title")  String title,
                         @Param("content")String content);
```

```xml
<!-- XML -->
<insert id="insertAuditsCyfh" parameterType="map" useGeneratedKeys="true" keyProperty="articleId">
  INSERT INTO Audits_cyfh (user_id,title,content)
  VALUES (#{userId},#{title},#{content})
</insert>
```

把签名和 `parameterType` 对齐之后，MyBatis 就能给 `AuditsCyfhMapper` 生成代理，Spring 也能正常注入，你就不会再拿到 `null` 了。
![](image-20250617163656683.png)



因为其实Controller 的返回值是返回给客户端的，我门随意返回都无所谓，但是在 Mapper 接口中，我们的返回值必须好好的整一整，因为我们执行 SQL 语句他返回的值肯定是固定的

然后 Mapper 传入的参数也需要好好搞一搞，因为是需要传入我们 SQL 与的

# 很有意思
我们平常都是select all 对吧，就是不管我们知道是根据 ID 只查一条也是selec all 因为你懒得取搞那么多烦人的事情，但是我们一般也会用到一条的时候，例如这种情况对吧，然后我们就可以使用实体类，然后get（0）就算是获取他的第一条了，但是这个也很有问题，就是你查全区get0，如果这个id有还好，让你查到了，没查到也罢，会给你提示，但是如果你是直接查全文，那么如果已经没有3了，就会拿到4，就和你想的不一样
![](image-20250617175958685.png)


![](image-20250617175835700.png)