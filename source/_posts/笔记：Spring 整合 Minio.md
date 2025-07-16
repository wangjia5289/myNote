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
>1. MinioClient 用于与 MinIO 服务建立连接，发送请求。而所谓的 Minio 配置，就算将 MinioClient 声明为一个 Bean，使用时直接注入即可
>2. MinioClient 底层基于 OkHttpClient，使用的是 HTTP 连接池机制，这与我们日常理解的 MySQL 等数据库连接池有所不同，所以 MinioClient 是线程安全的，但是高并发时也会存在瓶颈

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






















