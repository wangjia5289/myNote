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

| 列名                           | 数据类型        | 约束        | 默认值 | 示例值                               | 说明                                 |
| ---------------------------- | ----------- | --------- | --- | --------------------------------- | ---------------------------------- |
| **user_id**                  | int         | 主键约束、自增属性 | 自增  | 1                                 | 用户唯一标识符（将 username 直接作为主键也是一种常见做法） |
| **username**                 | varchar(20) | 唯一约束      |     | john                              | 用户名                                |
| **password**                 | varchar(20) |           |     | $2a$10$abcdefghijklmnopqrstuvwxyz | 加密后密码（禁止明文密码直接入库）                  |
| **is_accountNonExpired**     | tinyint(1)  |           | 1   | 1                                 | 账户是否没过期（1 代表没过期）                   |
| **is_accountNonLocked**      | tinyint(1)  |           | 1   | 1                                 | 账户是否没锁定                            |
| **is_credentialsNonExpired** | tinyint(1)  |           | 1   | 1                                 | 密码是否没过期                            |
| **is_enabled**               | tinyint(1)  |           | 1   | 1                                 | 用户是否启用                             |
| **email**                    | VARCHAR(20) | 唯一约束      |     | john@example.com                  | 邮箱                                 |
| **phoneNumber**              | VARCHAR(20) | 唯一约束      |     | 13800138000                       | 电话号码                               |

```
# 1. 创建表
CREATE TABLE IF NOT EXISTS users (
    user_id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(20),
    password VARCHAR(20),
    is_accountNonExpired TINYINT(1) DEFAULT 1,
    is_accountNonLocked TINYINT(1) DEFAULT 1,
    is_credentialsNonExpired TINYINT(1) DEFAULT 1,
    is_enabled TINYINT(1) DEFAULT 1,
    email VARCHAR(20),
    phoneNumber VARCHAR(20)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

ALTER TABLE users ADD CONSTRAINT unique_username UNIQUE (username);

ALTER TABLE users ADD CONSTRAINT unique_email UNIQUE (email);

ALTER TABLE users ADD CONSTRAINT unique_phone UNIQUE (phoneNumber);


# 2. 插入数据
INSERT INTO users (username, password, email, phoneNumber)
VALUES
    ('alice', 'pass123', 'alice@example.com', '13800000000'),
    ('bob', 'pass456', 'bob@example.com', '13900000001'),
    ('BaTian', 'pass789', 'batian@example.com', '13700000002');
```

> [!NOTE] 注意事项
> 1. username、password、isAccountNonExpired、isAccountNonLocked、isCredentialsNonExpired、isEnabled、authorities 这七个字段是 `userDetails` 接口的默认属性，一般在数据库表中要全部包含
> 2. email、phoneNumber 等字段，是我们自己扩展的字段


<span style="background:#fff88f">2. roles 表（角色表）</span>

| 列名                           | 数据类型        | 约束        | 默认值 | 示例值                               | 说明                                    |
| ---------------------------- | ----------- | --------- | --- | --------------------------------- | ------------------------------------- |
| **role_id**                  | int         | 主键约束、自增属性 | 自增  | 1                                 | 角色唯一标识符                               |
| **role_name**                | varchar(20) | 唯一约束      |     | ROLE_ADMIN                        | 角色名称 (Spring Security 约定以 `ROLE_` 开头) |
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

| 列名          | 数据类型        | 约束                                                 | 默认值 | 示例值 | 说明           |
| ----------- | ----------- | -------------------------------------------------- | --- | --- | ------------ |
| **user_id** | int         | 主键约束（与 role_id 联合主键）<br>外键约束（指向 users 表中的 user_id） |     | 1   | users 表中的 id |
| **role_id** | varchar(20) | 主键约束（与 user_id 联合主键）<br>外键约束（指向 roles 表中的 role_id） |     | 1   | roles 表中的 id |
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


<span style="background:#fff88f">4. authorities 表（权限表）</span>

| 列名                  | 数据类型        | 约束        | 默认值 | 示例值                     | 说明                       |
| ------------------- | ----------- | --------- | --- | ----------------------- | ------------------------ |
| **authoritie_id**   | int         | 主键约束、自增属性 | 自增  | 1                       | 权限唯一标识                   |
| **authoritie_name** | varchar(50) | 唯一约束      |     | finance:invoice:approve | 权限名称（常采用 模块：资源：操作 的命名方式） |

```
# 1. 创建表
CREATE TABLE IF NOT EXISTS authorities (
    authoritie_id INT AUTO_INCREMENT PRIMARY KEY,
    authoritie_name VARCHAR(50)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

ALTER TABLE authorities ADD CONSTRAINT unique_authoritie_name UNIQUE (authoritie_name);


# 2. 插入数据
INSERT INTO authorities (authoritie_name) VALUES
    ('test:test:test');
```


<span style="background:#fff88f">5. role_authoritie 表（角色-权限关联表）</span>

| 列名                | 数据类型 | 约束                                                             | 默认值 | 示例值 | 说明                 |
| ----------------- | ---- | -------------------------------------------------------------- | --- | --- | ------------------ |
| **role_id**       | int  | 主键约束（与 authoritie_id 联合主键）<br>外键约束（指向 roles 表中的 role_id）       |     | 1   | roles 表中的 id       |
| **authoritie_id** | int  | 主键约束（与 role_id 联合主键）<br>外键约束（指向 authorities 表中的 authoritie_id） |     | 1   | authorities 表中的 id |

```
# 1. 创建表
CREATE TABLE role_authority (
    role_id INT NOT NULL,
    authoritie_id INT NOT NULL,
    PRIMARY KEY (role_id, authoritie_id),
    FOREIGN KEY (role_id) REFERENCES roles(role_id),
    FOREIGN KEY (authoritie_id) REFERENCES authorities(authoritie_id)
) ;


# 2. 插入数据
INSERT INTO role_authority (role_id, authoritie_id) VALUES
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
    
	// authorities 表中的数据（用户的权限）
    private List<String> authorities;

    // getter 方法
	// setter 方法
	// equals 方法
	// hashCode 方法
	// toString 方法
}
```

> [!NOTE] 注意事项
> 1. 与数据库表映射的类通常称为 Entity 类，也可称为 DO 类或 PO 类，统属 POJO 类，通常只包含 getter、setter 、equals、hashCode、toString 方法及构造方法，不应包含业务逻辑方法
> 2. 数据库中的表名一般使用复数形式，如 users；而在 Java 中则采用单数形式命名，如 User
> 3. 使用 MyBatisX 插件生成的 POJO 类默认包含 getter、setter、equals、hashCode、toString 方法，但不包含构造方法。
> 	1. 我们可以手动补全有参和无参构造方法；
> 	2. 同时我么也可以删除自动生成的 equals、hashCode、toString 方法，改为使用 IDEA 生成
> 4. 别忘了添加 `private List<String> authorities;` 及其对应方法（getter、setter、equals、hashCode、toString 方法及构造方法）

-----





























