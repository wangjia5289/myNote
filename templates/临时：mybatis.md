### TypeHandler

在 MyBatis 中，TypeHandler 用于 **Java 类型与数据库类型之间的转换**。它负责把 Java 数据类型转换成 JDBC 类型（数据库数据类型）写入数据库，也负责把数据库查询结果转换成 Java 对象。

MyBatis 中内置了一些 TypeHandler，这也是我们为什么能实现例如从 instant 转 timestamp、tiny 转boolean 等等操作，自定义TypeHandler





























、、、、、、、、、、


![](image-20250714161254838.png)






![](image-20250714103337217.png)

![](image-20250714103343673.png)






























![](image-20250701101838609.png)

![](image-20250701101830916.png)

![](image-20250701101544473.png)


> [!NOTE] 注意事项
> 1. 与数据库表映射的类通常称为 Entity 类，也可称为 DO 类或 PO 类，统属 POJO 类，通常只包含 getter、setter 、equals、hashCode、toString 方法及构造方法，不应包含业务逻辑方法
> 2. 数据库中的表名一般使用复数形式，如 users；而在 Java 中则采用单数形式命名，如 User
> 3. 别忘了添加 `private List<String> authorities;` 及其对应方法
> 4. 使用 MyBatisX 插件生成的 POJO 类默认包含 getter、setter、equals、hashCode、toString 方法，但不包含构造方法。
> 	1. 我们可以手动补全有参和无参构造方法；
> 	2. 同时我么也也可以删除自动生成的 equals、hashCode、toString 方法，改为使用 IDEA 生成



> [!NOTE] 注意事项
> 1. username、password、isAccountNonExpired、isAccountNonLocked、isCredentialsNonExpired、isEnabled、authorities 这七个字段是 `userDetails` 接口的默认属性，一般在数据库表中要全部包含
> 2. email、phone_number 等字段，是我们自己扩展的字段。
> 3. 虽然 camelCase 在 Java 中使用广泛，例如 phoneNumber，但在 SQL 表列名中更建议统一为 snake_case，例如 phone_number









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

##### 单数据源

```
# application.yml
mybatis:
  configuration:
    map-underscore-to-camel-case: true
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
  mapper-locations: classpath*:mapper/*.xml
  type-aliases-package: com.example.spring_data_mybatis.model.entity

spring:
  datasource:
    url: jdbc:mysql://192.168.136.7:3306/repository1?useUnicode=true&characterEncoding=utf8&serverTimezone=UTC
    username: root
    password: wq666666
    driver-class-name: com.mysql.cj.jdbc.Driver









# application.yml
mybatis:
  configuration:
    map-underscore-to-camel-case: true      # 驼峰映射：user_name ↔ userName
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl   # 在控制台打印 SQL

  # 指定所有 Mapper XML 的位置（支持通配符）
  mapper-locations: classpath*:mapper/*.xml  

  # 实体类的包，给全类名起别名，简化 Mapper XML 中的 type 写法
  type-aliases-package: com.example.spring_data_mybatis.model.entity

spring:
  datasource:
    # JDBC URL（数据库地址 + 库名 + 编码 + 时区）
    url: jdbc:mysql://192.168.136.7:3306/repository1?useUnicode=true&characterEncoding=utf8&serverTimezone=UTC
    username: root                           # 数据库用户名
    password: wq666666                       # 数据库密码
    driver-class-name: com.mysql.cj.jdbc.Driver  # MySQL 8 驱动
```

别忘了创建一个 MybatisConfigutation，使用 Mapper scan 去扫描 mapper 接口




##### 多数据源

==1.通用配置==
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


==2.数据源 1 配置类==
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


==3.数据源 2 创建配置类==
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


#### 1.4. 配置连接池

Spring Boot 默认使用 **HikariCP** 作为连接池，其默认配置已经能够满足大多数场景下的性能和高可用性要求。如果在特定情况下有特殊需求，也可以根据实际情况自定义连接池配置。

---


#### 1.5. 编写 Model Entity Pojo 类

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


#### 1.6. 编写 Mapper 接口

##### 1.6.1. Mapper 接口概述

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


##### 1.6.2. Mapper 接口定义方法

###### 1.6.2.1. 方法名

方法名称的命名应确保清晰且易于维护，通常以动词开头，以明确表示所执行的操作。例如：
1. `insertUser`
2. `updateUser`
3. `deleteUser`
4. `selectUserById`

> [!NOTE] 注意事项：务必注意
> 1. 如果是使用 Mapper XML 书写 SQL 语句，务必保证**方法的名称**和 **SQL 的 ID** **一致**

---


###### 1.6.2.2. 返回类型

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


###### 1.6.2.3. 参数类型

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


#### 1.7. 编写 SQL 语句

##### 1.7.1. 编写 SQL 语句概述

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


##### 1.7.2. SQL 语句编写规则

###### 1.7.2.1. 书写位置

在 MyBatis 中，SQL 语句与 Java 代码是分离的，SQL 语句可以通过以下两种方式进行定义：

1. 通过注解在 `Mapper` 接口中直接编写 SQL 语句，尽管这种方式简便，但缺乏可维护性并且不支持复杂的映射规则和 SQL 语句
2. 通过 `Mapper XML` 文件编写 SQL 语句，并且可以设置详细的映射规则，提供了更强的灵活性，支持复杂的映射规则和 SQL 语句（一个 Mapper 接口对应一个 Mapper XML 映射文件）

无论选择哪种方式，我们都需要注册 SQL 与 Mapper 之间的映射（`mapper-locations` 或 `@MapperScan`），并且扫描 Mapper 接口，让 MyBatis 为其生成代理对象（`@MapperScan`）

> [!NOTE] 注意事项
> 1. 还是推荐在 `Mapper XML` 中编写 SQL 语句


---


###### 1.7.2.2. 书写要求

1. 虽然你可以在 `@Mapper` 接口或 XML 映射文件中书写**任意类型**的 SQL（包括 DDL、DCL 等），但 MyBatis 的核心定位是数据操作层，主要聚焦在 **DQL（查询）和 DML（增删改）** 上。其他类型的语句虽然技术上可行，但 不推荐使用，建议**直接规避**此类用法。
2. 编写 SQL 时，**请勿在语句末尾添加分号**（`;`）

---


###### 1.7.2.3. 占位符传值

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


###### 1.7.2.4. 动态 SQL

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


#### 1.8. 后续操作

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



> [!NOTE] 注意事项
> 1. 使用 MyBatisX 插件生成的Entity 类，默认是不包含其有参无参构造的，只是包含了，getter、setter、equals、hashCode、toString 方法，我们可以手动添加有参构造和无参构造


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



```
<!-- 一次调用，批量插入主表 + 三个关联表 -->
    <insert id="insertClientFull" parameterType="Client">
        <!-- 1. 主表 -->
        INSERT INTO oauth_clients (
        id,
        client_id,
        client_id_issued_at,
        client_secret,
        client_secret_expires_at,
        client_name,
        redirect_uris,
        client_settings,
        token_settings
        ) VALUES (
        #{id},
        #{clientId},
        #{clientIdIssuedAt},
        #{clientSecret},
        #{clientSecretExpiresAt},
        #{clientName},
        #{redirectUris, typeHandler=com.example.oauthserverwithmyproject.handler.RedirectUrisTypeHandler},
        #{clientSettings, typeHandler=com.example.oauthserverwithmyproject.handler.ClientSettingsTypeHandler},
        #{tokenSettings, typeHandler=com.example.oauthserverwithmyproject.handler.TokenSettingsTypeHandler}
        );

        <!-- 2. 认证方式 -->
        INSERT INTO oauth_client_authentication_methods ( client_id, method )
        VALUES
        <foreach collection="authenticationMethods" item="m" separator=",">
            (#{clientId}, #{m.value})
        </foreach>;

        <!-- 3. 授权类型 -->
        INSERT INTO oauth_client_authorization_grant_types ( client_id, grant_type )
        VALUES
        <foreach collection="authorizationGrantTypes" item="g" separator=",">
            (#{clientId}, #{g.value})
        </foreach>;

        <!-- 4. 作用域 -->
        INSERT INTO oauth_client_scopes ( client_id, scope )
        VALUES
        <foreach collection="scopes" item="s" separator=",">
            (#{clientId}, #{s})
        </foreach>
    </insert>
```


一个 SQL 多个语句，需要开启jdbc:mysql://192.168.136.8:3306/security?useUnicode=true&characterEncoding=utf8&serverTimezone=UTC&**allowMultiQueries=true**
![](image-20250714142338238.png)

![](image-20250714142356095.png)

![](image-20250714142401540.png)



## ---------------
干脆不支持扁平化的就子查询，能由构造函数的就直接join好了

一般是：

![](image-20250714111238419.png)


这两种写法各有场景和利弊，具体怎么选，取决于你对性能、代码复杂度、维护性的侧重点。下面帮你分析一下：

---

## 1. `<collection>` 直接映射子对象（无子查询）

```xml
<collection property="authorities"
            ofType="org.springframework.security.core.authority.SimpleGrantedAuthority">
    <constructor>
        <arg column="role_name" javaType="java.lang.String"/>
    </constructor>
</collection>
```

### 优点

- 结构简单，写在同一个 resultMap 里。
    
- 适合**SQL 语句已包含关联数据**，比如通过 JOIN 一次查出所有信息。
    

### 缺点

- SQL 会变复杂，JOIN 导致结果集行数增多（行爆炸），性能可能下降。
    
- 可能导致数据重复（比如主表字段重复出现多次）。
    

### 适用场景

- 你的查询 SQL 已经联表查询了所有相关数据。
    
- 关联数据量不大，且性能要求不高。
    

---

## 2. `<collection>` 使用子查询（`select` 属性）

```xml
<collection property="authorities"
            ofType="org.springframework.security.core.authority.SimpleGrantedAuthority"
            select="selectAuthoritiesByUserId"/>
```

### 优点

- SQL 简单，主查询只查主表。
    
- 关联数据通过单独查询加载，结构清晰。
    
- 避免 JOIN 导致行爆炸和数据重复。
    

### 缺点

- 额外的数据库查询，可能导致 N+1 问题（查询次数增多）。
    
- 需要写额外的子查询 SQL。
    

### 适用场景

- 关联数据量较大。
    
- 不想写复杂的多表 JOIN。
    
- 可以接受多次查询换取主查询简洁和性能。
    

---

## 3. 综合建议

|场景|选择|
|---|---|
|数据量小，关系简单|直接写 `<collection>` 嵌套映射|
|关联表数据多、复杂|使用子查询 `<collection select=...>`|
|需要分页、过滤等|子查询方式更灵活|

---

## 4. 小技巧

- 如果用子查询方式，尽量优化子查询（加索引、批量加载）。
    
- 复杂情况下，考虑用 MyBatis 的二级缓存减少数据库访问。
    
- 你也可以结合缓存或分布式缓存方案降低多次查询压力。
    

---

如果你想，我可以帮你写一套基于子查询和基于 JOIN 的示例 SQL 和 Mapper XML，你看着选用也方便。










![](image-20250630164405309.png)



`jdbcType` 是 MyBatis 中 `<resultMap>` 和 `<parameterMap>` 标签下的一个属性，用于**指定 Java 属性与数据库字段之间的 JDBC 类型映射**，主要起两个作用：

---

### ✅ 1. **明确类型映射，避免类型不匹配**

- 数据库字段的类型有很多种，比如 `VARCHAR`、`INTEGER`、`TINYINT`、`TIMESTAMP` 等，而 MyBatis 需要将这些类型正确转换为 Java 中的类型。
    
- 一般情况下，MyBatis 会自动根据 JDBC 元数据推断类型。但如果你指定了 `jdbcType`，它就会按照你指定的类型来进行处理，更加稳妥。
    

例如：

```xml
<result property="isEnabled" column="is_enabled" jdbcType="TINYINT"/>
```

这里你显式告诉 MyBatis：`is_enabled` 是数据库中的 `TINYINT` 类型，通常用于表示布尔值（0/1），MyBatis 就会按这个标准去读取并转换为 Java 中的 `Boolean` 或 `Integer`。

---

### ✅ 2. **处理 NULL 值时更准确**

当数据库字段为 `NULL` 时，如果没有指定 `jdbcType`，可能会出现：

- 数据映射异常
    
- 插入时报错（尤其是调用存储过程时）
    

比如在 `insert` 或 `update` 中使用：

```xml
<if test="email != null">
  email = #{email, jdbcType=VARCHAR},
</if>
```

这可以确保即使 `email == null`，JDBC 也知道它的目标类型是 `VARCHAR`，避免报错。

---

### 🔍 支持的 `jdbcType` 值来自 `java.sql.Types`，常用的包括：

|jdbcType|对应 SQL 类型|映射的 Java 类型|
|---|---|---|
|`VARCHAR`|字符串类型|`String`|
|`INTEGER`|整数|`Integer`|
|`TINYINT`|小整型（如布尔值）|`Integer` / `Boolean`|
|`BIGINT`|大整型（如主键）|`Long`|
|`TIMESTAMP`|时间戳|`java.sql.Timestamp` / `Date`|
|`BOOLEAN`|布尔型|`Boolean`|

---

### 🧠 总结一句话：

> `jdbcType` 用于明确数据库字段的类型，有助于 MyBatis 正确进行类型映射、处理空值和避免运行时类型错误。写不写都行，但在某些边界情况下（如 `null`、布尔值、存储过程）**建议明确指定**。


## ----------------2


![](image-20250630164501699.png)



你这个写法其实非常典型，MyBatis 中的 `<collection>` 标签是 **用来映射一对多（或集合类型）属性的**，你写的内容表示：

```xml
<collection property="authorities"
            ofType="org.springframework.security.core.authority.SimpleGrantedAuthority">
    <constructor>
        <arg column="role_name" javaType="java.lang.String"/>
    </constructor>
</collection>
```

意思是：

> `User` 类中有一个属性叫 `authorities`，类型是 `List<SimpleGrantedAuthority>`，它的每一项是通过调用 `SimpleGrantedAuthority(String roleName)` 构造函数创建的，其中 `role_name` 是 SQL 查询结果中的字段。

---

### ✅ 回答你的问题一：这段 `<collection>` 的作用是什么？

这是 MyBatis 中处理**嵌套集合属性**的方式。你定义的是：

```java
private List<SimpleGrantedAuthority> authorities;
```

而数据库里用户和权限是 1:N 的关系，比如：

|user_id|username|role_name|
|---|---|---|
|1|alice|ROLE_USER|
|1|alice|ROLE_ADMIN|
|2|bob|ROLE_USER|

你希望查一个用户，同时把他所有的权限装进 `List<SimpleGrantedAuthority>`，就需要用到 `<collection>`。

---

![](image-20250714094045430.png)

### ✅ 回答你的问题二：能不能在 `<resultMap>` 里写 `<collection>`？

**完全可以！**  
并且就是应该这么写。你要在 `BaseResultMap` 中嵌套 `<collection>` 来表示“一个用户拥有多个权限”。

示例结构如下：

```xml
<resultMap id="BaseResultMap" type="com.example.securitywithhttpsession.entity.User">
    <id property="userId" column="user_id" jdbcType="INTEGER"/>
    <result property="username" column="username" jdbcType="VARCHAR"/>
    <result property="password" column="password" jdbcType="VARCHAR"/>
    <!-- 其他字段略 -->

    <collection property="authorities"
                ofType="org.springframework.security.core.authority.SimpleGrantedAuthority">
        <constructor>
            <arg column="role_name" javaType="java.lang.String"/>
        </constructor>
    </collection>
</resultMap>
```

---

### 🧠 注意事项

1. **你的 SQL 要包含多行重复的用户记录**，每行只差在 `role_name` 上（即做了 Join，比如 user + role 表），才能装满这个集合。
    
2. `authorities` 是通过构造函数创建的，所以你用 `<constructor>` 和 `<arg>` 来描述；如果你换成自定义的类，有 setter，也可以用 `<result>` 方式。
    
3. 建议在 XML 语句中配合 `DISTINCT` 或 `GROUP BY` 防止结果重复加载。
    

---

### ✅ 示例 SQL（配合上面的 `<collection>`）

```xml
<select id="selectUserWithAuthorities" resultMap="BaseResultMap">
    SELECT 
        u.user_id, u.username, u.password, ..., 
        r.role_name
    FROM user u
    LEFT JOIN user_role ur ON u.user_id = ur.user_id
    LEFT JOIN role r ON ur.role_id = r.role_id
    WHERE u.username = #{username}
</select>
```

---

如果你愿意，我可以帮你完整拼出带 `<resultMap>` 和 `<select>` 的 XML 映射文件。是否需要？

![](image-20250630164833697.png)


![](image-20250630165046151.png)


![](image-20250630165129961.png)

![](image-20250630165135983.png)

非常好，这个问题抓得很关键！我们来**一步一步拆开讲清楚**这个 `<constructor><arg ... /></constructor>` 的意思，尤其是它为什么要这样写，适用于什么样的 Java 类。

---

## 🎯 问题核心：为什么要用 `<constructor>`？

### ✅ 背景：你要把 SQL 查询结果映射成 Java 对象

在你这个例子中，数据库返回了一列：

```sql
role_name
---------
ROLE_USER
ROLE_ADMIN
```

你希望这些行变成 Java 中的对象，比如：

```java
new SimpleGrantedAuthority("ROLE_USER");
new SimpleGrantedAuthority("ROLE_ADMIN");
```

而 `SimpleGrantedAuthority` 是 Spring Security 提供的权限类，它的定义是这样的：

```java
public class SimpleGrantedAuthority implements GrantedAuthority {
    private final String role;

    public SimpleGrantedAuthority(String role) {
        this.role = role;
    }

    @Override
    public String getAuthority() {
        return this.role;
    }
}
```

> ❗ 你看到了吗？它：
> 
> - 没有无参构造方法
>     
> - 也没有 `setAuthority()` 或类似方法
>     
> - 只能通过构造函数来创建！
>     

---

## 🔧 于是你就得告诉 MyBatis：

> “嘿，我要创建一个 `SimpleGrantedAuthority` 对象，它只能通过构造函数，并且这个构造函数的参数，是 SQL 中那一列 `role_name` 的值。”

---

## ✅ 解释代码

```xml
<constructor>
    <arg column="role_name" javaType="java.lang.String"/>
</constructor>
```

这句话的含义是：

|元素|含义|
|---|---|
|`<constructor>`|指定你要用构造函数来创建对象|
|`<arg>`|指定构造函数的一个参数|
|`column="role_name"`|这个参数来自 SQL 查询结果中的 `role_name` 列|
|`javaType="java.lang.String"`|这个参数是 Java 中的字符串类型，对应构造函数的 `String` 参数|

> 💡 换句话说，这行代码就是：`new SimpleGrantedAuthority(结果集中 role_name 这一列的值)`

---

## 🧠 什么时候需要 `<constructor>`？

- 当你要映射的类 **没有无参构造器**，也 **没有 setter 方法**（即“只读”类）；
    
- 只能通过 `new 构造函数(...)` 的方式创建；
    
- 典型例子就是 `SimpleGrantedAuthority`、`java.time.LocalDateTime`、一些值对象（Value Object）。
    

---

## ✅ 小总结：你现在就可以这样理解：

```xml
<constructor>
    <arg column="role_name" javaType="java.lang.String"/>
</constructor>
```

==就是告诉 MyBatis：这个类没法用无参构造器+setter 来组装，只能用构造函数来装，而且这个构造函数需要一个从 SQL 中 `role_name` 列拿到的字符串。==

---

如果你对 Java 类和构造函数的映射机制还不太熟悉，我也可以通过一个小案例来图解这个流程，要不要我帮你做一个例子？



![](image-20250630165421792.png)


![](image-20250630165448012.png)

![](image-20250630165521024.png)


![](image-20250630165530896.png)

![](image-20250630165535784.png)




![](image-20250630165613284.png)

![](image-20250630165627907.png)

![](image-20250630165633897.png)
![](image-20250630165638775.png)


简单来说，就是

你的resultMap 其实之影响查询，那么如果插入，需不需要要写paratype呢？








![](image-20250704115942238.png)

usermapper 返回的，是要经过controller 进行处理的










## --------


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
简单来说啊，就是如果你使用resultType，他会直接将结果 -> pojo，那如果你没开启什么驼峰映射啊什么的，如果列名和属性名不一致，那就xxxx了

如果你使用resultMap，它会将结果 -> resultMap -> pojo(resultMap 中的 type)
![](image-20250630184136049.png)


![](image-20250630184156172.png)

![](image-20250630184233952.png)

![](image-20250630184249382.png)
![](image-20250630184256176.png)

![](image-20250630184315442.png)

![](image-20250630184425409.png)

其实 Mapper 接口中的参数类型啊，应为是要传递给 XML文件的，也就是我们#{} 的参数，所以限制比较多，什么@Param 这种，但是返回类型啥的那就无须多言了，根本无需在意


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


