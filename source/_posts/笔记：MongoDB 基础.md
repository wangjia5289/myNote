---
title: 笔记：MongoDB 基础
date: 2025-05-16
categories:
  - 数据管理
  - 非关系型数据库
  - 文档型
  - MongoDB
  - MongoDB 基础
tags: 
author: 霸天
layout: post
---



### MongoDB 语法

#### 库操作

==1.创建数据库==
```
# 1. 如果数据库不存在，则创建数据库，否则切换到指定数据库。
use <repository-name>
```

> [!NOTE] 注意事项
> 1. MongoDB 很“懒”，它不像 MySQL 那样执行 `CREATE DATABASE` 后立刻就把文件写好。
> 2. 如果你用 `use runoob` 切换了数据库，它并不会立即创建这个数据库，只有你真正**往这个数据库的集合里插入了数据**，MongoDB 才会真的在磁盘上“认认真真”把数据库搞出来；
> 3. 所以这就是我们使用 `show dbs` 查不出来的原因，我们可以先创建集合，插入一些数据


==2.删除数据库==
```
# 1. 切换到要删除数据库
use myDatabase


# 2. 删除数据库
db.dropDatabase()
```


==3.修改数据库==


==4.查询数据库==
```
# 1. 查询所有数据库
show dbs


# 2. 查看当前使用的数据库
db
```


==5.使用数据库==
```
# 1. 如果数据库不存在，则创建数据库，否则切换到指定数据库。
use <repository-name>
```

---


#### 集合操作

##### 前言

我们对集合进行任何操作之前，都要先用 `use 库名` 指定数据库，否则就会默认使用 `test` 数据库

----

##### 创建集合（☆）

###### 创建集合模版

```
# 1. 使用指定数据库（如果未使用 use 命令切换到其他数据库，则会默认使用 test 数据库）
use myDatabase


# 2. 创建集合（db 代表当前使用的数据库）
db.createCollection("<collection-name>", {<options>})
"""
1. options：
	1. capped：
	    1. 是否创建一个固定大小的集合。
	    2. 示例值：capped: true
	2. size：
	    1. 集合的最大大小（以字节为单位，仅在 capped 为 true 时有效）
	    2. 示例值：size: 10485760（10MB）
	3. max：
	    1. 集合中允许的最大文档数（仅在 capped 为 true 时有效）
	    2. 示例值：max: 5000 
	4. validator：
	    1. 用于文档验证的表达式，阻止不符合要求的数据插入或更新我们的文档。
	    2. 示例值：见下文
	5. validationLevel：
	    1. 指定文档验证的严格程度。
	    2. 示例值：validationLevel: "strict"
		    1. "off"：不进行验证；
		    2. "strict"（默认）：插入和更新操作都必须通过验证；
		    3. "moderate"：仅更新现有文档时验证，插入新文档不验证。
	6. validationAction：
	    1. 指定文档验证失败时的操作。
	    2. 示例值：validationAction: "error"
		    1. "error"（默认）：阻止插入或更新；
		    2. "warn"：允许插入或更新，但会发出警告。
	7. storageEngine：
	    1. 为集合指定存储引擎配置。
	    2. 示例值：见下文
	8. collation：
	    1. 指定集合的默认排序规则。
	    2. 示例值：collation: { locale: "en", strength: 2 }
2. 示例：
db.createCollection("myComplexCollection", {
  capped: true,
  size: 10485760,
  max: 5000,
  validator: { $jsonSchema: {
    bsonType: "object",
    required: ["name", "email"],
    properties: {
      name: {
        bsonType: "string",
        description: "必须为字符串且为必填项"
      },
      email: {
        bsonType: "string",
        pattern: "^.+@.+$",
        description: "必须为有效的电子邮件地址"
      }
    }
  }},
  validationLevel: "strict",
  validationAction: "error",
  storageEngine: {
    wiredTiger: { configString: "block_compressor=zstd" }
  },
  collation: { locale: "en", strength: 2 }
});
"""
```

> [!NOTE] 注意事项
> 1. 如果未使用 use 命令切换到其他数据库，则会默认使用 test 数据库
> 2. `db` 代表当前数据库

---


###### validator

---


###### storageEngine


---


###### collation

---


##### 删除集合

```
db.collection.drop()
```

---


##### 修改集合

---


##### 查询集合

```
# 1. 查看当前使用的数据库所有集合
show collections
```

---


#### 文档操作

==1.插入文档==


2.删除文档


3.修改文档


4.查询文档



### MongoDB 存储引擎

| **引擎**             | **特点**                                                                    | **适合场景**                     | **当前状况**                      |
| ------------------ | ------------------------------------------------------------------------- | ---------------------------- | ----------------------------- |
| **WiredTiger（默认）** | - 文档级并发控制<br>- 支持 zlib、snappy、zstd 等多种压缩，节省磁盘空间<br>- 支持多文档事务<br>- 可配置缓存大小 | 日常生产环境、对并发和事务有要求的应用，既要性能又要稳定 | MongoDB 默认引擎，功能强大，适合大多数场景，最推荐 |
| **MMAPv1（废弃）**     |                                                                           |                              | 自 MongoDB 4.0 起已废弃，不再维护，不推荐使用 |
| **In-Memory**      | - 数据完全保存在内存中，读写极快，但重启丢失所有数据                                               |                              | 直接上 Redis                     |
| **Encrypted**      | - 基于 WiredTiger，是其增强版本<br>- 支持数据文件加密                                      | 适用于对数据安全有要求的场景（如金融、医疗、政府等）   | 对安全要求高的场景强烈推荐使用               |

----















# 二、实操

### 环境搭建

#### 单机测试环境搭建

```
docker run --rm -d \
  --name secure-mongo \
  -p 27017:27017 \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=wq666666 \
  mongo:latest
```

---
































































