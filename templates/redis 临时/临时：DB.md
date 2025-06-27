

功能框图
```
%% 配置主题为 neutral，移除黄色背景 %%{init: { 'theme': 'neutral' } }%% flowchart TB %% 核心系统节点，作为顶层 Core[文学网站核心系统] %% 用户模块相关节点，先定义外部节点，再用子图包裹内部逻辑 UserMod[用户模块] Core --> UserMod subgraph UM[用户模块内部逻辑] direction TB UserMod --> 未登录[用户未登录] 未登录 --> 游客角色 UserMod --> 注册登录[用户注册/登录] 注册登录 --> 用户身份 用户身份 --> 用户表 用户身份 --> 身份表 身份表 --> 会员角色 身份表 --> 版主角色 身份表 --> 管理员角色[总管理员角色] 版主角色 --> 风月版主[风花雪月版主角色] 版主角色 --> 校园版主[校园故事版主角色] UserMod --> 进入网站[用户进入网站] 进入网站 --> 登录判断{是否登录/注册?} 登录判断 -->|否| 游客角色 登录判断 -->|是| 执行登录注册[执行登录/注册] 执行登录注册 --> 验证身份[验证用户身份] 验证身份 --> 用户表 验证身份 --> 身份表 身份表 --> 会员角色 身份表 --> 管理员角色 身份表 --> 版主角色 版主角色 --> 茶余版主[茶余饭后版主角色] 版主角色 --> 风月版主 版主角色 --> 校园版主 end %% 文章模块相关节点，同样先定义外部节点，再用子图包裹 ArticleMod[文章模块] Core --> ArticleMod subgraph AM[文章模块内部逻辑] direction TB ArticleMod --> 茶余[茶余饭后] 茶余 --> 茶余文章表[文章表] 茶余 --> 茶余审核表[审核表] 茶余 --> 茶余留言表[留言表] ArticleMod --> 风月[风花雪月] 风月 --> 风月文章表[文章表] 风月 --> 风月审核表[审核表] 风月 --> 风月留言表[留言表] ArticleMod --> 校园[校园故事] 校园 --> 校园文章表[文章表] 校园 --> 校园审核表[审核表] 校园 --> 校园留言表[留言表] end
```










根据您的需求，我将为您设计一个文学网站的数据库，并提供详细的建表语句和数据字典示例。以下是基于问题描述的完整数据库设计：

### 数据库设计概述
该文学网站需要支持以下功能：
- 用户登录后才能发表文章。
- 文章按版块分类（如茶余饭后、风花雪月、校园故事等）。
- 游客和会员均可浏览文章并留言，游客留言需提供email。
- 版主负责文章的增删和录用，版主由总管理员授权。

为此，我设计了以下6个表：
1. **用户表（User）**：存储用户信息及角色（会员、版主、管理员）。
2. **版块表（Section）**：存储版块信息。
3. **文章表（Article）**：存储文章信息。
4. **留言表（Comment）**：存储文章的留言。
5. **审核表（Audit）**：记录版主对文章的审核。
6. **版主授权表（Moderator_Section）**：记录版主与版块的对应关系。

---

### 建表语句
以下是以MySQL为例的建表语句，包含主键、外键约束及必要字段：

```sql
-- 用户表
CREATE TABLE User (
    user_id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    password VARCHAR(50) NOT NULL,
    role ENUM('member', 'moderator', 'admin') NOT NULL,
    email VARCHAR(100)
);

-- 版块表
CREATE TABLE Section (
    section_id INT AUTO_INCREMENT PRIMARY KEY,
    section_name VARCHAR(50) NOT NULL
);

-- 文章表
CREATE TABLE Article (
    article_id INT AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(100) NOT NULL,
    content TEXT NOT NULL,
    author_id INT NOT NULL,
    section_id INT NOT NULL,
    publish_date DATETIME NOT NULL,
    FOREIGN KEY (author_id) REFERENCES User(user_id),
    FOREIGN KEY (section_id) REFERENCES Section(section_id)
);

-- 留言表
CREATE TABLE Comment (
    comment_id INT AUTO_INCREMENT PRIMARY KEY,
    article_id INT NOT NULL,
    user_id INT,
    guest_email VARCHAR(100),
    content TEXT NOT NULL,
    comment_date DATETIME NOT NULL,
    FOREIGN KEY (article_id) REFERENCES Article(article_id),
    FOREIGN KEY (user_id) REFERENCES User(user_id)
);

-- 审核表
CREATE TABLE Audit (
    audit_id INT AUTO_INCREMENT PRIMARY KEY,
    article_id INT NOT NULL,
    moderator_id INT NOT NULL,
    status ENUM('pending', 'approved', 'rejected') NOT NULL,
    audit_date DATETIME NOT NULL,
    FOREIGN KEY (article_id) REFERENCES Article(article_id),
    FOREIGN KEY (moderator_id) REFERENCES User(user_id)
);

-- 版主授权表
CREATE TABLE Moderator_Section (
    moderator_id INT NOT NULL,
    section_id INT NOT NULL,
    PRIMARY KEY (moderator_id, section_id),
    FOREIGN KEY (moderator_id) REFERENCES User(user_id),
    FOREIGN KEY (section_id) REFERENCES Section(section_id)
);
```

---

### 数据字典示例
以下是各表的数据字典，包含字段名称、数据类型、约束、键信息及含义。

#### 表2.1 用户信息数据字典（User）

| 序号  | 数据项名称    | 数据类型                                 | 范围约束 | 关键数据项 | 含义     |
| --- | -------- | ------------------------------------ | ---- | ----- | ------ |
| 1   | user_id  | INT                                  | 自增   | 主键    | 用户ID   |
| 2   | username | VARCHAR(50)                          |      |       | 用户名    |
| 3   | password | VARCHAR(50)                          |      |       | 密码     |
| 4   | role     | ENUM('member', 'moderator', 'admin') |      |       | 角色     |
| 5   | email    | VARCHAR(100)                         |      |       | 邮箱（可选） |

#### 表2.2 版块信息数据字典（Section）

| 序号 | 数据项名称   | 数据类型      | 范围约束 | 关键数据项 | 含义     |
|------|--------------|---------------|----------|------------|----------|
| 1    | section_id   | INT          | 自增     | 主键       | 版块ID   |
| 2    | section_name | VARCHAR(50)  |          |            | 版块名称 |

#### 表2.3 文章信息数据字典(Article)

| 序号  | 数据项名称        | 数据类型         | 范围约束 | 关键数据项                  | 含义     |
| --- | ------------ | ------------ | ---- | ---------------------- | ------ |
| 1   | article_id   | INT          | 自增   | 主键                     | 文章ID   |
| 2   | title        | VARCHAR(100) |      |                        | 文章标题   |
| 3   | content      | TEXT         |      |                        | 文章内容   |
| 4   | author_id    | INT          |      | 外键（User.user_id）       | 作者ID   |
| 5   | section_id   | INT          |      | 外键（Section.section_id） | 所属版块ID |
| 6   | publish_date | DATETIME     |      |                        | 发表日期   |

#### 表2.4 留言信息数据字典(Comment)

| 序号  | 数据项名称        | 数据类型         | 范围约束 | 关键数据项                  | 含义        |
| --- | ------------ | ------------ | ---- | ---------------------- | --------- |
| 1   | comment_id   | INT          | 自增   | 主键                     | 留言ID      |
| 2   | article_id   | INT          |      | 外键（Article.article_id） | 文章ID      |
| 3   | user_id      | INT          |      | 外键（User.user_id）       | 留言者ID（会员） |
| 4   | guest_email  | VARCHAR(100) |      |                        | 游客email   |
| 5   | content      | TEXT         |      |                        | 留言内容      |
| 6   | comment_date | DATETIME     |      |                        | 留言日期      |

#### 表2.5 审核信息数据字典(Audit)

| 序号 | 数据项名称  | 数据类型      | 范围约束                 | 关键数据项             | 含义       |
|------|-------------|---------------|--------------------------|-----------------------|------------|
| 1    | audit_id    | INT          | 自增                     | 主键                  | 审核ID     |
| 2    | article_id  | INT          |                          | 外键（Article.article_id） | 文章ID |
| 3    | moderator_id| INT          |                          | 外键（User.user_id）  | 版主ID     |
| 4    | status      | ENUM('pending', 'approved', 'rejected') |        |                       | 审核状态   |
| 5    | audit_date  | DATETIME     |                          |                       | 审核日期   |

#### 表2.6 版主授权信息数据字典(Moderator_Section)

| 序号 | 数据项名称  | 数据类型 | 范围约束 | 关键数据项             | 含义     |
|------|-------------|----------|----------|-----------------------|----------|
| 1    | moderator_id| INT     |          | 外键（User.user_id）  | 版主ID   |
| 2    | section_id  | INT     |          | 外键（Section.section_id） | 版块ID |

---

### 表间关系说明
- **用户表和文章表**：一对多（一个用户可发表多篇文章）。
- **文章表和版块表**：多对一（多篇文章属于一个版块）。
- **文章表和留言表**：一对多（一篇文章可有多个留言）。
- **用户表和留言表**：一对多（一个用户可发表多个留言）。
- **文章表和审核表**：一对多（一篇文章可有多次审核记录）。
- **用户表（版主）和版块表**：多对多（通过版主授权表关联）。

此设计满足了文学网站的用户管理、文章管理、留言功能及版主授权需求。如需进一步调整或补充，请随时告知！


### 建表语句

用户表（Users）
```
CREATE TABLE Users (
    user_id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    password VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE,
    registration_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    update_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB;
```

身份表（Roles）
```
CREATE TABLE Roles (
    role_id INT AUTO_INCREMENT PRIMARY KEY,
    role_name VARCHAR(50) UNIQUE NOT NULL
) ENGINE=InnoDB;
```

用户-身份中间表（User_Roles）
```
CREATE TABLE User_Roles (
    user_id INT,
    role_id INT,
    PRIMARY KEY (user_id, role_id),
    FOREIGN KEY (user_id) REFERENCES Users(user_id) ON DELETE CASCADE,
    FOREIGN KEY (role_id) REFERENCES Roles(role_id) ON DELETE CASCADE
) ENGINE=InnoDB;
```

版块表（Sections）
```
CREATE TABLE Sections (
    section_id INT AUTO_INCREMENT PRIMARY KEY,
    section_name VARCHAR(50) UNIQUE NOT NULL,
    description TEXT,
    creation_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    update_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB;
```

茶余饭后相关表
```
文章
CREATE TABLE Articles_cyfh ( 
    article_id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    title VARCHAR(100) NOT NULL,
    content TEXT NOT NULL,
    status ENUM('pending', 'approved', 'rejected') DEFAULT 'pending',
    creation_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    update_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES Users(user_id) ON DELETE CASCADE
) ENGINE=InnoDB;

审核
CREATE TABLE Audits_cyfh (
    audit_id INT AUTO_INCREMENT PRIMARY KEY,
    article_id INT NOT NULL,
    moderator_id INT NOT NULL,
    action ENUM('approve', 'reject') NOT NULL,
    action_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    comments TEXT,
    FOREIGN KEY (article_id) REFERENCES Articles_cyfh(article_id) ON DELETE CASCADE,
    FOREIGN KEY (moderator_id) REFERENCES Users(user_id) ON DELETE CASCADE
) ENGINE=InnoDB;

评论
CREATE TABLE Comments_cyfh (
    comment_id INT AUTO_INCREMENT PRIMARY KEY,
    article_id INT NOT NULL,
    user_id INT,
    email VARCHAR(100),
    content TEXT NOT NULL,
    creation_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (article_id) REFERENCES Articles_cyfh(article_id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES Users(user_id) ON DELETE SET NULL
) ENGINE=InnoDB;


文章留言中间表
CREATE TABLE Article_Comments_cyfh (
    article_id INT,
    comment_id INT,
    PRIMARY KEY (article_id, comment_id),
    FOREIGN KEY (article_id) REFERENCES Articles_cyfh(article_id) ON DELETE CASCADE,
    FOREIGN KEY (comment_id) REFERENCES Comments_cyfh(comment_id) ON DELETE CASCADE
) ENGINE=InnoDB;
```
风花雪月和校园故事表：结构类似，表名分别为Articles_fhxy、Audits_fhxy、Comments_fhxy、Article_Comments_fhxy和Articles_xygs、Audits_xygs、Comments_xygs，Article_Comments_xygs字段定义一致，仅表名不同。


风花雪月相关表
```
-- 文章表
CREATE TABLE Articles_fhxy (
    article_id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    title VARCHAR(100) NOT NULL,
    content TEXT NOT NULL,
    status ENUM('pending', 'approved', 'rejected') DEFAULT 'pending',
    creation_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    update_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES Users(user_id) ON DELETE CASCADE
) ENGINE=InnoDB;

-- 审核表
CREATE TABLE Audits_fhxy (
    audit_id INT AUTO_INCREMENT PRIMARY KEY,
    article_id INT NOT NULL,
    moderator_id INT NOT NULL,
    action ENUM('approve', 'reject') NOT NULL,
    action_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    comments TEXT,
    FOREIGN KEY (article_id) REFERENCES Articles_fhxy(article_id) ON DELETE CASCADE,
    FOREIGN KEY (moderator_id) REFERENCES Users(user_id) ON DELETE CASCADE
) ENGINE=InnoDB;

-- 评论表
CREATE TABLE Comments_fhxy (
    comment_id INT AUTO_INCREMENT PRIMARY KEY,
    article_id INT NOT NULL,
    user_id INT,
    email VARCHAR(100),
    content TEXT NOT NULL,
    creation_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (article_id) REFERENCES Articles_fhxy(article_id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES Users(user_id) ON DELETE SET NULL
) ENGINE=InnoDB;

-- 文章留言中间表
CREATE TABLE Article_Comments_fhxy (
    article_id INT,
    comment_id INT,
    PRIMARY KEY (article_id, comment_id),
    FOREIGN KEY (article_id) REFERENCES Articles_fhxy(article_id) ON DELETE CASCADE,
    FOREIGN KEY (comment_id) REFERENCES Comments_fhxy(comment_id) ON DELETE CASCADE
) ENGINE=InnoDB;

```



校园故事相关表
```
-- 文章表
CREATE TABLE Articles_xygs (
    article_id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    title VARCHAR(100) NOT NULL,
    content TEXT NOT NULL,
    status ENUM('pending', 'approved', 'rejected') DEFAULT 'pending',
    creation_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    update_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES Users(user_id) ON DELETE CASCADE
) ENGINE=InnoDB;

-- 审核表
CREATE TABLE Audits_xygs (
    audit_id INT AUTO_INCREMENT PRIMARY KEY,
    article_id INT NOT NULL,
    moderator_id INT NOT NULL,
    action ENUM('approve', 'reject') NOT NULL,
    action_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    comments TEXT,
    FOREIGN KEY (article_id) REFERENCES Articles_xygs(article_id) ON DELETE CASCADE,
    FOREIGN KEY (moderator_id) REFERENCES Users(user_id) ON DELETE CASCADE
) ENGINE=InnoDB;

-- 评论表
CREATE TABLE Comments_xygs (
    comment_id INT AUTO_INCREMENT PRIMARY KEY,
    article_id INT NOT NULL,
    user_id INT,
    email VARCHAR(100),
    content TEXT NOT NULL,
    creation_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (article_id) REFERENCES Articles_xygs(article_id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES Users(user_id) ON DELETE SET NULL
) ENGINE=InnoDB;

-- 文章留言中间表
CREATE TABLE Article_Comments_xygs (
    article_id INT,
    comment_id INT,
    PRIMARY KEY (article_id, comment_id),
    FOREIGN KEY (article_id) REFERENCES Articles_xygs(article_id) ON DELETE CASCADE,
    FOREIGN KEY (comment_id) REFERENCES Comments_xygs(comment_id) ON DELETE CASCADE
) ENGINE=InnoDB;

```