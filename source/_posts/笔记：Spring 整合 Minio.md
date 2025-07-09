---
title: 笔记：Spring 整合 Minio
date: 2025-05-13
categories:
  - 数据管理
  - 数据的组织方式
  - 对象存储
  - Minio
  - Spring 整合 Minio
tags: 
author: 霸天
layout: post
---
# 一、理论



## Bucket

### Bucket 概述

Bucket 是存储 Object 的逻辑空间，每个 Bucket 之间的数据是相互隔离的，对用户而言，相当于存放文件的顶层**文件夹**

----


### Bucket 策略

我们可以使用 Minio SDK 进行指定 Bucket 策略：
```
@PostMapping("/setBucketPolicy")
public String setBucketPolicy(@RequestParam String bucketName, @RequestBody String policyJson) {
	try {
		minioClient.setBucketPolicy(
				SetBucketPolicyArgs.builder()
						.bucket(bucketName)
						.config(policyJson)
						.build()
		);
		return "桶 [" + bucketName + "] 策略设置成功";
	} catch (Exception e) {
		return "设置桶策略失败：" + e.getMessage();
	}
}
```

在这里需要我们传入 JSON，我们请求体需要这样写：
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"AWS": ["*"]},
      "Action": ["s3:GetObject"],
      "Resource": ["arn:aws:s3:::your-bucket-name/*"]
    }
  ]
}
```
<font color="#92d050">1. Version</font>
表示 Bucket 策略语法的版本，目前写 `"2012-10-17"` 是标准写法


<font color="#92d050">2. Effect</font>
表示我们写的这个策略是允许还是禁止，在多个策略冲突时，Deny 权限优先生效，可选值有：
1. Allow
2. Deny

> [!NOTE] 注意事项：策略冲突
> 1. 比如给用户绑定了某个权限策略，用户可能属于一个或多个组，这些组也会有自己的策略，桶可能设置了自己的访问策略，限制谁能访问，系统可能还有一些默认策略或者管理员设置的全局策略
> 2. 两个策略同时作用于同一个用户时，策略内容“冲突”了：一个允许，一个拒绝，Deny（拒绝）优先生效


<font color="#92d050">3. Principal</font>
指定那个用户可以执行或者拒绝谁执行哪些操作，需要看我们的 Effect，如果是 Allow，就说明是可以执行，如果是 Deny 就是指定谁不可以执行，常见有：
1. * 
	1. 所有用户
2. user-arn
	1. 某个用户的用户名

> [!NOTE] 注意事项
> 1. AWS 是指用户


<font color="#92d050">4. Action</font>
指定动作，常见有：
1. Bucket 级操作：
	1. s3:ListBucket
		1. 列出桶内的对象列表
	2. s3:GetBucketLocation
		1. 获取桶所在区域（Region）信息
	3. s3:ListBucketMultipartUploads
		1. 查看这个桶中进行中的多部分上传任务（分片上传）
	4. s3:GetBucketPolicy
		1. 查看桶策略
	5. s3:PutBucketPolicy
		1. 设置桶策略
	6. s3:DeleteBucketPolicy
		1. 删除桶策略
2. Object 级操作：
	1. s3:GetObject
		1. 下载 Object
	2. s3:PutObject
		1. 上传对象
	3. s3:DeleteObject
		1. 删除对象
	4. s3:AbortMultipartUpload
		1. 取消一个正在进行的分片上传任务
	5. s3:ListMultipartUploadParts
		1. 查看某个分片上传任务的已上传部分
	6. s3:GetObjectVersion
		1. 获取某个版本对象（MinIO默认不开启版本控制）
3. s3:ListBucket
	1. 列出桶中的对象列表
4. s3:GetBucketLocation
	1. 获取桶的区域信息
5. s3:*
	1. 所有操作

> [!NOTE] 注意事项
> 1. 获取桶的区域信息，也就是这个桶属于哪个区域（Region），在 MinIO 社区版中区域的概念有时是“虚拟”的（比如默认都叫 `us-east-1`），**但权限上仍然要考虑是否开放 `s3:GetBucketLocation`**，否则 SDK/CLI 工具可能报 403。


<font color="#92d050">5. Resource</font>
操作哪些资源，常见有：
1. arn:aws:s3:::my-bucket
	1. 桶本身
2. arn:aws:s3:::my-bucket/*
	1. 桶内所有 Object
3. arn:aws:s3:::my-bucket/public/*
	1. 某些 Object

> [!NOTE] 注意事项
> 1. 除 Minio SDK 外，还可以通过 Web UI、Minio CLI 进行指定
> 2. 需要注意的是，社区版不能通过 Web UI 进行指定

----


### Bucket 标签

一组键值对标签，用来给桶做标识或分类，前端需要使用 JSON 格式发过来：
```
{
  "env": "production",
  "team": "devops",
  "project": "minio-demo"
}
```






### Bucket 生命周期

其实这不能叫做 Bucket 生命周期，而应该叫做 Object 生命周期，但是在操作维度上还是属于 Bucket 生命周期，我们可以使用 Minio SDK 进行指定 Bucket 生命周期：
```
@PostMapping("/setBucketLifecycle")
public String setBucketLifecycle(@RequestParam String bucketName,
								 @RequestBody LifecycleConfiguration lifecycleConfigurationXml) {
	try {
		minioClient.setBucketLifecycle(
				SetBucketLifecycleArgs.builder()
						.bucket(bucketName)
						.config(lifecycleConfigurationXml) // 传入 LifecycleConfigutation 类型的 Bucket 生命周期
						.build()
		);
		return "桶 [" + bucketName + "] 生命周期设置成功";
	} catch (Exception e) {
		return "设置桶生命周期失败：" + e.getMessage();
	}
}
```

在这里需要我们传入 XML，我们请求体需要这样写：
```
<LifecycleConfiguration xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
  <Rule>
    <ID>expire-temp-files</ID>
    <Status>Enabled</Status>
    <Filter>
      <Prefix>temp/</Prefix>
    </Filter>
    <Expiration>
      <Days>7</Days>
    </Expiration>
  </Rule>
  <Rule>
    <ID>expire-logs</ID>
    <Status>Enabled</Status>
    <Filter>
      <Prefix>logs/</Prefix>
    </Filter>
    <Expiration>
      <Days>90</Days>
    </Expiration>
      <AbortIncompleteMultipartUpload>
  <DaysAfterInitiation>3</DaysAfterInitiation>
</AbortIncompleteMultipartUpload>
  </Rule>
  <Rule>
  <ID>delete-old-versions</ID>
  <Status>Enabled</Status>
  <Filter>
    <Prefix>archive/</Prefix>
  </Filter>
  <NoncurrentVersionExpiration>
    <NoncurrentDays>30</NoncurrentDays>
  </NoncurrentVersionExpiration>
</Rule>
</LifecycleConfiguration>
```

<font color="#92d050">1. LifecycleConfiguration</font>
根标签，表示这是一个生命周期配置文件，其 xmlns 是 XML 命名空间，指定该 XML 遵循 S3 生命周期配置的标准格式


<font color="#92d050">2. ID</font>
该规则（Role）的唯一标识，方便管理，最多 255 个字符


<font color="#92d050">3. Statue</font>
启用或禁用该条规则，可选值：
1. Enabled
2. Disabled


<font color="#92d050">4. Filter</font>
对象过滤器，常用有：
1. Prefix
	1. 匹配对象键名前缀
	2. 如 temp/ 指的匹配桶中所有以 `temp/` 开头的对象
2. And
	1. 可组合多个条件，一般是组合前缀和标签
```
<Filter>
    <And>
        <Prefix>log/</Prefix>
        <Tag>
            <Key>env</Key>
            <Value>dev</Value>
        </Tag>
        <Tag>
            <Key>type</Key>
            <Value>debug</Value>
        </Tag>
    </And>
</Filter>
```

> [!NOTE] 注意事项
> 1. Tag 是作用在 Object 上的


<font color="#92d050">5. Expiration</font>
过期策略，Days 与 Date 两选一，过期后将被删除
```
// 1. 30 天后过期
<Expiration>
  <Days>30</Days>
</Expiration>


// 2. 在固定日期后过期，必须填写标准 ISO 格式，这里是在2025年12月31日0时0分0秒0毫秒过期
<Expiration>
  <Date>2025-12-31T00:00:00.000Z</Date>
</Expiration>
```

> [!NOTE] 注意事项
> 1. 生命周期配置是**整体配置**，不是单挑规则独立操作，也就是说：每次调用 `setBucketLifecycle` 都是覆盖桶的**全部生命周期配置**


<font color="#92d050">6. NoncurrentVersionExpiration</font>
仅适用于 **版本控制开启的桶**，当对象成为旧版本后几天删除，只能使用 NoncurrentDays
```
<NoncurrentVersionExpiration>
  <NoncurrentDays>30</NoncurrentDays>
</NoncurrentVersionExpiration>
```


<font color="#92d050">7. AbortIncompleteMultipartUpload</font>
从分片上传开始算起，几天后自动终止“未完成的分片上传”任务，节省空间，只支持 DaysAfterInitiation
```
<AbortIncompleteMultipartUpload>
  <DaysAfterInitiation>3</DaysAfterInitiation>
</AbortIncompleteMultipartUpload>
```










## Minio 配置

MinioConfiguration 配置类在 `com.example.minio.configuration` 包下
```
@Configuration
public class MinioConfiguration {

    @Bean(name = "minioClientA")
    public MinioClient minioClientA() {
        return MinioClient.builder()
                .endpoint("http://192.168.136.8:9000")
                .credentials("acount", "password")
                .build();
    }

    @Bean(name = "minioClientB")
    public MinioClient minioClientB() {
        return MinioClient.builder()
                .endpoint("http://192.168.136.9:9000")
                .credentials("adminB", "adminB123")
                .build();
    }
}
```

> [!NOTE] 注意事项
>1. 配置方面，我们只需将 MinioClient 声明为一个 Bean，使用时直接注入即可
>2. MinioClient 用于与 MinIO 服务建立连接，发送请求
>3. MinioClient 底层基于 OkHttpClient，使用的是 HTTP 连接池机制，这与我们日常理解的 MySQL 等数据库连接池有所不同，所以 MinioClient 是线程安全的

-----


# 二、实操

## 基本使用

### 创建 Spring Web 项目，添加相关依赖

创建时：
1. Web：
	1. Spring Web

创建后：添加 [minio 依赖](https://mvnrepository.com/artifact/io.minio/minio) 
``` 
<dependency>
    <groupId>io.minio</groupId>
    <artifactId>minio</artifactId>
    <version>8.5.17</version>
</dependency>
```

> [!NOTE] 注意事项
> 1. 由于目前未提供 `**-minio-starter` 的起步依赖，官方推荐的方式是直接添加 Minio 官方提供的 Minio Java SDK

----


### 进行 Minio 配置

----


### 使用 MinioClient 向 Minio 发送请求

```
@RestController
public class ObjectOperation {

    @Resource(name = "minioClientA")
    private MinioClient minioClient;

    @GetMapping("/bucketExists")
    public Boolean bucketExists(@RequestParam String bucketName) {
        try {
            return minioClient.bucketExists(
                    BucketExistsArgs.builder().bucket(bucketName).build()
            );
        } catch (ErrorResponseException e) {
            throw new RuntimeException("MinIO 响应错误：可能是桶名非法或服务端拒绝请求", e);
        } catch (InsufficientDataException e) {
            throw new RuntimeException("MinIO 响应数据不足，可能是连接中断或内容丢失", e);
        } catch (InternalException e) {
            throw new RuntimeException("MinIO 客户端内部异常，通常是 SDK bug 或序列化问题", e);
        } catch (InvalidKeyException e) {
            throw new RuntimeException("访问 MinIO 时的密钥无效，检查 AccessKey 和 SecretKey", e);
        } catch (InvalidResponseException e) {
            throw new RuntimeException("MinIO 返回了无法解析的响应，可能是服务异常或版本不兼容", e);
        } catch (IOException e) {
            throw new RuntimeException("I/O 异常，可能是网络问题或底层流读写失败", e);
        } catch (NoSuchAlgorithmException e) {
            throw new RuntimeException("本地不支持 MinIO 所需的加密算法（如 SHA-256）", e);
        } catch (ServerException e) {
            throw new RuntimeException("MinIO 服务端异常，请检查服务是否正常", e);
        } catch (XmlParserException e) {
            throw new RuntimeException("MinIO 响应中的 XML 解析失败，可能格式不符或数据损坏", e);
        }
    }
}
```

---


## 业务处理

### Bucket 常用操作

```
@RestController
@RequestMapping("/minio/bucket")
public class BucketMethod {

    @Resource(name = "minioClientA")
    private MinioClient minioClient;

    /**
     * ============================================
     * 1. 创建 Bucket（minioClient.makeBucket）
     * --------------------------------------------
     * 返回值：
     * - 无返回值
     *
     * 注意事项：
     * - MakeBucketArgs.builder().bucket(bucketName).build() 方法是先创建一个构建器（builder），设置好参数后，构造出最终的 MakeBucketArgs 实例，Minio SDK 中大量使用了这种用法
     * - 创建 Bucket 时，若该 Bucket 已存在则会抛异常，通常先判断该 Bucket 是否已存在
     * ============================================
     */
    @PostMapping("/createBucket")
    public String createBucket(@RequestParam String bucketName) {
        try {
            boolean found = minioClient.bucketExists(
                    BucketExistsArgs.builder().bucket(bucketName).build()
            );

            if (!found) {
                minioClient.makeBucket(
                        MakeBucketArgs.builder().bucket(bucketName).build()
                );
                return "桶 [" + bucketName + "] 创建成功";
            } else {
                return "桶 [" + bucketName + "] 已存在，无需创建";
            }
        } catch (Exception e) {
            return "创建桶失败：" + e.getMessage();
        }
    }


    /**
     * ============================================
     * 2. 删除 Bucket（minioClient.removeBucket）
     * --------------------------------------------
     * 返回值
     * - 无返回值
     * ============================================
     */
    @DeleteMapping("/deleteBucket")
    public String deleteBucket(@RequestParam String bucketName) {
        try {
            minioClient.removeBucket(
                    RemoveBucketArgs.builder().bucket(bucketName).build()
            );
            return "桶 [" + bucketName + "] 删除成功";
        } catch (Exception e) {
            return "删除桶失败：" + e.getMessage();
        }
    }


    /**
     * ============================================
     * 3. 获取所有当前用户可访问的 Bucket（minioClient.listBuckets）
     * --------------------------------------------
     * 注意事项：
     * - 该方法用于列出当前用户有权限访问的所有存储桶
     * - 每个 bucket 对象都包含存储桶的名称、创建时间等信息
     * - 这里我只把每个 bucket 的 存储桶名称 拿出来了
     * ============================================
     */
    @GetMapping("/listBuckets")
    public List<String> listBuckets() {
        try {
            return minioClient.listBuckets() // 返回的是 List <Bucket>，Bucket 是 MinIO 提供的类，表示一个桶对象。
                    .stream() // 把这个桶列表转换成 Java Stream 流，方便做批量处理，返回的是 Stream <Bucket>
                    .map(bucket -> bucket.name()) // 对每一个 Bucket 对象，取出它的名字，返回 Stream <String>
                    .collect(Collectors.toList()); // 把每一个 Bucket 的名字收集成一个列表，返回 List <String>
        } catch (Exception e) {
            throw new RuntimeException("获取桶列表失败：" + e.getMessage(), e);
        }
    }


    /**
     * ============================================
     * 4. 设置 Bucket 的策略（minioClient.setBucketPolicy）
     * --------------------------------------------
     * 返回值：
     * - 无返回值
     *
     * 注意事项：
     * - 前端应该传递 JSON 类型的 Bucket 策略
     * - 用 String 类型接收 Bucket 策略
     * ============================================
     */
    @PostMapping("/setBucketPolicy")
    public String setBucketPolicy(@RequestParam String bucketName, @RequestBody String policyJson) {
        try {
            minioClient.setBucketPolicy(
                    SetBucketPolicyArgs.builder()
                            .bucket(bucketName)
                            .config(policyJson) // 传入 JSON 格式的字符串类型的 Bucket 策略
                            .build()
            );
            return "桶 [" + bucketName + "] 策略设置成功";
        } catch (Exception e) {
            return "设置桶策略失败：" + e.getMessage();
        }
    }


    /**
     * ============================================
     * 5. 获取 Bucket 的策略（minioClient.getBucketPolicy）
     * --------------------------------------------
     * 返回值：
     * - 返回 JSON 格式的字符串（还是 String，里面是 JSON 格式的，如："{\"username\":\"wangza\",\"role\":\"admin\"}"）
     *
     * 注意事项：
     * - 若 Bucket 未设置过策略，会抛出异常或返回空
     * - 刚创建 Bucket，你看到的 private，不算设置过策略
     * ============================================
     */
    @GetMapping("/getBucketPolicy")
    public String getBucketPolicy(@RequestParam String bucketName) {
        try {
            return minioClient.getBucketPolicy(
                    GetBucketPolicyArgs.builder()
                            .bucket(bucketName)
                            .build()
            );
        } catch (Exception e) {
            return "获取桶策略失败：" + e.getMessage();
        }
    }


    /**
     * ============================================
     * 6. 设置 Bucket 的生命周期（minioClient.setBucketLifecycle）
     * --------------------------------------------
     * 返回值：
     * - 无返回值
     *
     * 注意事项：
     * - 前端应该传递 XML 类型的 Bucket 生命周期
     * ============================================
     */
    @PostMapping("/setBucketLifecycle")
    public String setBucketLifecycle(@RequestParam String bucketName,
                                     @RequestBody LifecycleConfiguration lifecycleConfigurationXml) {
        try {
            minioClient.setBucketLifecycle(
                    SetBucketLifecycleArgs.builder()
                            .bucket(bucketName)
                            .config(lifecycleConfigurationXml) // 传入 LifecycleConfigutation 类型的 Bucket 生命周期
                            .build()
            );
            return "桶 [" + bucketName + "] 生命周期设置成功";
        } catch (Exception e) {
            return "设置桶生命周期失败：" + e.getMessage();
        }
    }


    /**
     * ============================================
     * 7. 获取 Bucket 的生命周期
     * --------------------------------------------
     * 返回值：
     * - 返回 XML 格式字符串
     *
     * 注意事项：
     * - 若 Bucket 未配置过生命周期，会抛异常或返回空
     * ============================================
     */
    @GetMapping("/getBucketLifecycle")
    public String getBucketLifecycle(@RequestParam String bucketName) {
        try {
            return minioClient.getBucketLifecycle(
                    GetBucketLifecycleArgs.builder()
                            .bucket(bucketName)
                            .build()
            ).toString();
        } catch (Exception e) {
            return "获取桶生命周期失败：" + e.getMessage();
        }
    }


    /**
     * ============================================
     * 8. 设置 Bucket 的标签
     * --------------------------------------------
     * 返回值：
     * - 无返回值
     *
     * 注意事项：
     * - 前端应该传递 JSON 类型的 Bucket 标签
     * - 用 String 类型接收 Bucket 标签
     * ============================================
     */
    @PostMapping("/setBucketTags")
    public String setBucketTags(@RequestParam String bucketName,
                                @RequestBody String tagsJson) {
        try {
            // 将 JSON 转为 Map<String, String>
            Map<String, String> tags = new ObjectMapper()
                    .readValue(tagsJson, new TypeReference<Map<String,String>>() {});
            minioClient.setBucketTags(
                    SetBucketTagsArgs.builder()
                            .bucket(bucketName)
                            .tags(tags) // 传入 JSON 格式的字符串类型的 Bucket 标签
                            .build()
            );
            return "桶 [" + bucketName + "] 标签设置成功";
        } catch (Exception e) {
            return "设置桶标签失败：" + e.getMessage();
        }
    }


    /**
     * ============================================
     * 9. 获取 Bucket 的标签
     * --------------------------------------------
     * 返回值：
     * - 返回标签的键值对 Map
     *
     * 注意事项：
     * - 若 Bucket 未设置过标签，会抛异常或返回空 Map
     * ============================================
     */
    @GetMapping("/getBucketTags")
    public Map<String, String> getBucketTags(@RequestParam String bucketName) {
        try {
            return minioClient.getBucketTags(
                    GetBucketTagsArgs.builder().bucket(bucketName).build()
            ).get();
        } catch (Exception e) {
            throw new RuntimeException("获取桶标签失败：" + e.getMessage(), e);
        }
    }
}
```

---


### Object 常用操作






















