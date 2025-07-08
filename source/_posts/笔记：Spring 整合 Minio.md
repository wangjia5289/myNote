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

Bucket 策略时用于指定什么人可以对 Bucket 进行扫描操作，我们可以在 Minio Web UI、Java、Minio CLi 进行指定，但是在社区版的 Minio 中，不能使用 Web UI 进行指定，我们可以使用 Java 进行指定：
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

在这里需要我们传入 String policyJson，我们请求体需要这样写：
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

----








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






















