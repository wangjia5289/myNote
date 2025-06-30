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

## 导图

![](source/_posts/笔记：Spring%20Security/Map：SpringSecurity.xmind)

---


## Spring Security 执行流程

<span style="background:#fff88f">1. 用户请求（客户端请求）</span>
每次用户访问受 `Spring Security` 保护的资源，都会经过以下流程


<span style="background:#fff88f">2. SecurityContextPersistenceFilter 介入</span>
自动为本线程初始化 `SecurityContextHolder` 并根据 `JSESSIONID` 向 `HttpSession` 查找 `SecurityContext`（其内保存最重要的 `Authentication`）
1. 若存在 `SecurityContext`，便将其加载到本线程的 `SecurityContextHolder` 中（基于 HttpSession 实现 “记住我” 功能，我们也可基于 JWT 实现 “记住我” 功能）
2. 若不存在 `SecurityContext`，则在本线程中自动初始化一个新的 `SecurityContext`

即使我们不打算通过 `HttpSession` 实现 “记住我” 功能（如使用 JWT），甚至完全不使用 `HttpSession`，我们仍然建议保留这个过滤器，因为它自动为本线程初始化 `SecurityContextHolder`、并自动创建 `SecurityContext`，这个能力实在太香了。
![](image-20250628224744251.png)


<span style="background:#fff88f">3. UsernamePasswordAuthenticationFilter 介入</span>
该过滤器主要用于前后端未分离的场景，用于处理默认 `/login` 路径下的登录请求。  

在前后端分离的架构中无需深入关注其具体逻辑，只需了解其在过滤器链中的位置，以便在插入自定义过滤器时能准确定位。


<span style="background:#fff88f">4. AnonymousAuthenticationFilter 介入</span>
如果当前没有任何 `Authentication`，系统会自动创建一个匿名身份，以避免后续流程中出现空指针异常。


<span style="background:#fff88f">5. FilterSecurityInterceptor 介入</span>
首先检查当前线程中是否存在 `Authentication`（无论是否匿名） ，如果不存在，则抛出 `AuthenticationException`（表示未认证）

接着检查当前权限是否有权访问对应的资源或方法（即资源级别的访问控制、方法级别的访问控制），若无权限，则抛出 `AccessDeniedException`
> [!NOTE] 注意事项
> 1. 整个流程中的异常由 `ExceptionTranslation` 过滤器统一处理，负责捕获**整个过滤器链中**抛出的 `AuthenticationException` 和 `AccessDeniedException` 异常，并执行相应的处理逻辑。


<span style="background:#fff88f">6. 执行 API</span>
在这一步，才真正开始执行我们的 API 逻辑；如果是登录 API，并且通过 AuthenticationManager 进行认证，流程如下：
![](image-20250628210023140.png)


<span style="background:#fff88f">7. SecurityContextPersistenceFilter 再次介入</span>
它会自动将本线程的 `SecurityContext` 存入服务器的 `HttpSession`，以便在后续请求中维持用户身份（需要手动开启）

 随后，过滤器会清空本线程 `SecurityContextHolder`，防止 `SecurityContext` 在后续请求中被无意复用，从而确保每个请求都能独立执行认证和授权流程。

----


# 二、实操

## 基本使用

### 基于 HttpSession 的 Spring Security

#### 1.1. 创建 Spring Web 项目，添加 Security 相关依赖

1. Web：
	1. Spring Web
2. Security：
	1. Spring Security
3. SQL
	1. JDBC API
	2. MyBatis Framework
	3. MySQL Driver

---


#### 1.2. 创建用户-角色-权限表

我们一般会创建五个表：`users` 表（用户表）存储所有注册用户的信息，`roles` 表（角色表）定义了系统中存在的各种角色，`user_role` 表（用户-角色关联表）用于建立用户和角色之间的多对多关系，`authorities` 表（权限表）定义了系统中的各种操作权限，`role_authoritie` 表（角色-权限表）用于建立角色和权限之间的多对多关系

<span style="background:#fff88f">1. users 表（用户表）</span>

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


# 2. 插入数据
INSERT INTO users (username, password, email, phone_number)
VALUES
    ('alice', 'pass123', 'alice@example.com', '13800000000'),
    ('bob', 'pass456', 'bob@example.com', '13900000001'),
    ('BaTian', 'pass789', 'batian@example.com', '13700000002');
```

> [!NOTE] 注意事项
> 1. username、password、isAccountNonExpired、isAccountNonLocked、isCredentialsNonExpired、isEnabled、authorities 这七个字段是 `userDetails` 接口的默认属性，一般在数据库表中要全部包含
> 2. email、phone_number 等字段，是我们自己扩展的字段。
> 3. 虽然 camelCase 在 Java 中使用广泛，例如 phoneNumber，但在 SQL 表列名中更建议统一为 snake_case，例如 phone_number


<span style="background:#fff88f">2. roles 表（角色表）</span>

| 列名            | 数据类型        | 约束        | 默认值 | 索引   | 示例值        | 说明                                    |
| ------------- | ----------- | --------- | --- | ---- | ---------- | ------------------------------------- |
| **role_id**   | int         | 主键约束、自增属性 | 自增  | 主键索引 | 1          | 角色唯一标识符                               |
| **role_name** | varchar(20) | 唯一约束      |     | 唯一索引 | ROLE_ADMIN | 角色名称 (Spring Security 约定以 `ROLE_` 开头) |
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
	('ROLE_USER'),
	('ROLE_MANAGER'),
	('ROLE_GUEST');
```


<span style="background:#fff88f">3. user_role 表（用户-角色关联表）</span>

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


# 2. 插入数据
INSERT INTO user_role (user_id, role_id) VALUES
	(1, 1);
```

> [!NOTE] 注意事项
> 1. 两表的关联表，一般是取其两表名的单数形式，以 `_` 进行衔接


<span style="background:#fff88f">4. authorities 表（权限表）</span>

| 列名                 | 数据类型        | 约束        | 索引   | 默认值 | 示例值                     | 说明                       |
| ------------------ | ----------- | --------- | ---- | --- | ----------------------- | ------------------------ |
| **authority_id**   | int         | 主键约束、自增属性 | 主键索引 | 自增  | 1                       | 权限唯一标识                   |
| **authority_name** | varchar(50) | 唯一约束      | 唯一索引 |     | finance:invoice:approve | 权限名称（常采用 模块：资源：操作 的命名方式） |

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


<span style="background:#fff88f">5. role_authority 表（角色-权限关联表）</span>

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

---


#### 使用 Spring Data MyBatis 实现查询用户的基本信息和权限

##### 前置步骤

详见笔记：Spring Data MyBatis

----


##### 编写 User Entity 类

User Entity 类位于 `com.example.securitywithhttpsession.entity` 包下
```
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
```

> [!NOTE] 注意事项
> 1. 与数据库表映射的类通常称为 Entity 类，也可称为 DO 类或 PO 类，统属 POJO 类，通常只包含 getter、setter 、equals、hashCode、toString 方法及构造方法，不应包含业务逻辑方法
> 2. 使用 MyBatisX 插件生成的 POJO 类默认包含 getter、setter、equals、hashCode、toString 方法，但不包含构造方法。
> 	1. 我们可以手动补全有参和无参构造方法；
> 	2. 同时我么也可以删除自动生成的 equals、hashCode、toString 方法，改为使用 IDEA 生成
> 3. 本 User 类不仅与 Users 表映射，还包含了 authorities 表中的 `authoritie_name` 字段。因此，别忘了添加 `private List<String> authorities;` 及其对应的方法（包括 getter、setter、equals、hashCode、toString 方法以及构造方法）
> 4. 数据库中的表名一般使用复数形式，如 users，而在 Java 中则采用单数形式命名，如 User

![](image-20250630183241212.png)

-----


##### 编写 UserMapper 接口，并定义查询方法

User Entity 类位于 `com.example.securitywithhttpsession.mapper` 包下
```
@Mapper  
public interface UserMapper {  

    User getUserByUserName(@Param("username") String username) ;  
  
}
```

----


##### 编写查询方法对应的 SQL 语句（编写 UserMapper.xml）

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
            <collection property="authorities" ofType="org.springframework.security.core.authority.SimpleGrantedAuthority">
                <constructor>
                    <arg column="authority_name" javaType="java.lang.String" />
                </constructor>
            </collection>
    </resultMap>

    <sql id="Base_Column_List">
        user_id,
        username,
        password,
        is_accountNonExpired,
        is_accountNonLocked,
        is_credentialsNonExpired,
        is_enabled,
        email,
        phone_number
    </sql>

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


#### 实现 UserDetails 接口

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
        return true;                                                                     // 默认返回 true
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

Spring Security 是通过调用 `UserDetails` 提供的方法，来获取用户信息的（不是使用我们的 User Entity，因为Spring Security 咋知道你使用什么东西对吧，而是调用 `UserDetails` 的方法来获取的这些信息的）

可是我们也发现了 `UserDetails` 毕竟只是接口，它的方法没有返回值，也就是 Spring Security 调用这些接口方法，由于这些接口方法没有具体的实现，所以 Spring Security 根本拿不到值，这也就是为什么我们必须要去用 `CustomerUserDetailsImpl` 去实现这个接口，其目的就是为每个方法都做一个具体的实现，有一个返回值，这样我们的 Spring Seucurity 调用接口方法的时候能有一个值（其实我们发现了，像isAccountNonExpired()、isAccountNonLocked()、isCredentialsNonExpired()、isEnabled()这几个方法，也是默认有返回值的对吧，但是其他三个方法没有默认实现，也就是说我们至少也要去实现其他三个方法，否则编译报错），然后我们去实现这个接口：
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

那我们为什么非要实现 `UserDetails` 呢，你 Spring Security 知不知道这些值，管我啥事，我能知道就行了呗，其实我们知道 Spring Security 最核心的就是有一个 `Authentication` 对象保存在本线程对吧，其实 Spring Security 为什么要知道这些值，其实很重要，我们必须要熟悉以下事情：
1. Spring Security 是要用 `CustomerUserDetailsImpl` 来构建 `Authentication` 对象的
2. 我们的登录流程中，一般会是下面两种流程：
	1. 如果你是通过 `AuthenticationManager` 进行认证（自定义登录 API，选择使用 AuthenticationManager 进行认证逻辑）：
		1. Spring Security 会自动校验用户提交过来的用户名和密码与 `CustomerUserDetailsImpl` 中的用户名和密码是否匹配
		2. 如果匹配，则自动将 `CustomerUserDetailsImpl` 封装为 `Authentication`，最后保存到当前线程。
		3. 在认证的过程中，所有数据都是要从 `CustomerUserDetailsImpl` 中获取的，在封装为 `Authentication` 的过程中，Authentication 中的数据也都是从这个实现类中获取的
	2. 如果你是自定义登录逻辑（自定义登录 API，不使用 AuthenticationManager 进行认证逻辑，而是自己设计一套认证逻辑）：
		1. 首先你仍然要比对 `CustomerUserDetailsImpl` 中的用户名和密码与用户提交的用户名密码是否一致
		2. 验证通过后，我们就手动将这个 `CustomerUserDetailsImpl` 封装成一个 `Authentication` 对象，并保存到当前线程中。
		3. 这个过程中，其实在验证的时候，用户名和密码在不在 `CustomerUserDetailsImpl` 中都无所谓，哪怕你的用户名和密码在 `User Entity` 中也能拿来与前端用户提交过来的密码进行比较，一样能比较，而且，你不但可以使用用户名和密码了，对吧，这个你就随意了，对吧，因为你没有 `CustomerUserDetailsImpl` 的限制了，对吧，你不必须使用用户名、密码了，你也可以整一些高端的，什么扫码啊，验证码啊，都行。但是你进行手动封装 Authentication 的时候，还是必须要有 `CustomerUserDetailsImpl` 的，
		4. 也就是说在认证过程中，不必须 `CustomerUserDetailsImpl` ,但是在封装为 `Authentication` 的过程中，Authentication 中的数据必须是从这个实现类中获取的






简单来说，就是将我们通过 `UserMapper` 查询到的 `User` 对象（也就是数据库中的用户信息），再次封装到 `CustomerUserDetailsImpl` 中。

为什么要这样做？因为后续我们要使用 `CustomerUserDetailsImpl` 来构建 `Authentication` 对象，例如：
1. 如果你使用的是自定义登录 API，一般会经过如下流程：
	1. 首先比对 `CustomerUserDetailsImpl` 中的用户名和密码与用户提交的用户名密码是否一致。验
	2. 验证通过后，我们就手动将这个 `CustomerUserDetailsImpl` 封装成一个 `Authentication` 对象，并保存到当前线程中。
2. 如果你是通过 `AuthenticationManager` 进行认证，那流程会更自动化：
	1. 

那问题来了：我们到底是怎么把 `User` 中的信息封装进 `CustomerUserDetailsImpl` 的？这通常有两种方式：
1. 显式赋值方式：
	1. 先使用 `UserMapper` 获取到 `User` 对象，然后再`new` 一个 `CustomerUserDetailsImpl` 对象，再逐个将 `User` 的属性赋值给它
2. 构造方法方式（推荐）：
	1. 在 `CustomerUserDetailsImpl` 的构造方法中传入一个 `User` 对象，那么当我们 `new` 这个对象的时候，构造函数内部直接把 `User` 的属性赋值给自身，不需要我们再逐个将 `User` 的属性赋值给它

而 Spring Security 本身采用的就是第二种方式，那我们是不是就要自己写一个方法，先用 `UserMapper` 查询出 `User`，然后再 `new CustomerUserDetailsImpl(user)` 呢？

其实我们不需要自己额外写方法去查询 `User` 并再将其封装成 `CustomerUserDetailsImpl` 并返回，因为 Spring Security 已经为我们提供了一个接口：`UserDetailsService`。我们只需要实现这个接口，并重写它的 `loadUserByUsername` 方法即可。

在这个方法中，我们可以使用 `UserMapper` 去查询出对应的 `User` 对象，然后直接 `return new CustomerUserDetailsImpl(user)`。这样一来，每当我们调用这个方法时，Spring Security 就会自动完成用户信息的查询与封装，返回一个包含了用户数据的 `CustomerUserDetailsImpl` 实例。

需要注意的是，`CustomerUserDetailsImpl` 并不属于传统意义上的三层架构（Controller-Service-Repository），严格来说，它应当放置在 `com.example.securitywithhttpsession.entity` 包下，作为用户实体信息的一个安全扩展模型。

此外，Spring Security 提供的 `UserDetails` 接口默认只包含七个核心字段：`username`、`password`、`isAccountNonExpired`、`isAccountNonLocked`、`isCredentialsNonExpired`、`isEnabled` 和 `authorities`。

如果你希望在 `Authentication` 中携带更多的信息（例如 `email` 和 `phoneNumber`），是可以通过扩展 `CustomerUserDetailsImpl` 来实现的。这样做的好处是：你可以直接从 `SecurityContext` 中获取 `Authentication`，再从中获取这些扩展信息，无需再根据 `username` 查询数据库。这种方式在某些业务中能显著简化逻辑，提高效率。

不过需要明确的是，Spring Security 的设计初衷并不是鼓励我们将过多的用户字段直接放进 `Authentication` 对象中。是否将这些字段一并封装，应该根据你的具体业务场景权衡决定，避免过度冗余或信息泄露风险。



用户名、密码、权限状态等信息，进行认证和授权判断；
```
public class CustomerUserDetailsImpl implements UserDetails {

    private final String username;
    private final String password;
    private final Boolean isAccountNonExpired;
    private final Boolean isAccountNonLocked;
    private final Boolean isCredentialsNonExpired;
    private final Boolean isEnabled;
    private final List<GrantedAuthority> authorities;

    // 扩展的字段
    private final String email;
    private final String phoneNumber;
    private final Integer userId; // 如果用户名不是 ID，我们一般需要它这个ID，就加上

    public CustomerDetailsImpl(UserWithAuthoritie user) {
        this.username = user.getUsername();
        this.password = user.getPassword();
        this.isAccountNonExpired = user.getAccountNonExpired();
        this.isAccountNonLocked = user.getAccountNonLocked();
        this.isCredentialsNonExpired = user.getCredentialsNonExpired();
        this.isEnabled = user.getEnabled();
        this.email = user.getEmail();
        this.phoneNumber = user.getPhoneNumber();
        this.userId = user.getUserId();
        List<GrantedAuthority> grantedAuthorities = new ArrayList<>();
        if (user.getAuthorities() != null) {
            for (String authority : user.getAuthorities()) {
                grantedAuthorities.add(new SimpleGrantedAuthority(authority));
            }
        }
        this.authorities = grantedAuthorities;

    }

    // 扩展的方法
    public String getEmail() {
        return email;
    }

    public String getPhoneNumber() {
        return phoneNumber;
    }
    
	public Integer getUserId() {  
	    return userId;  
	}

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return this.authorities;
    }

    @Override
    public String getPassword() {
        return this.password;
    }

    @Override
    public String getUsername() {
        return this.username;
    }

    @Override
    public boolean isAccountNonExpired() {
        return this.isAccountNonExpired;
    }

    @Override
    public boolean isAccountNonLocked() {
        return this.isAccountNonLocked;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return this.isCredentialsNonExpired;
    }

    @Override
    public boolean isEnabled() {
        return this.isEnabled;
    }
}
```




















