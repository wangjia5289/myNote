---
title: 笔记：Spring 整合 Minio
date: 2025-05-13
categories:
  - 数据管理
  - 对象存储
  - Minio
tags: 
author: 霸天
layout: post
---
# 一、理论

### 1. minio 核心 API

### 2. MinioClient 线程安全问题

关于 **MinIO Client（MinIO Java SDK）** 是否线程安全的问题，以下是详细分析：

---

### 3. **官方文档的说明**
根据 MinIO 官方文档和社区讨论，**MinIO Java SDK 的 `MinioClient` 实例是线程安全的**。这意味着：
- 你可以将同一个 `MinioClient` 实例在多个线程中共享使用，无需额外的同步措施。
- 客户端内部的连接池、重试逻辑和网络请求处理都设计为线程安全，符合现代多线程应用的需求。

**参考依据**：
- MinIO 的官方 GitHub 仓库（[https://github.com/minio/minio-java](https://github.com/minio/minio-java)）中，`MinioClient` 的文档和代码注释未提及线程安全性的限制，且其设计目标之一是支持高并发场景。
- 社区讨论（如 GitHub Issues 和技术博客）中普遍认为 `MinioClient` 是线程安全的，适合多线程环境。

---

### 4. **为什么 MinIO Client 是线程安全的？**
MinIO Client 的线程安全性主要依赖于以下设计：
- **无状态连接池**：`MinioClient` 使用底层 HTTP 连接池（如 Apache HttpClient 或 OkHttp），这些连接池本身是线程安全的，支持并发请求。
- **不可变配置**：`MinioClient` 实例一旦创建，其配置（如 endpoint、accessKey、secretKey）是固定的，不会被修改，避免了竞态条件。
- **同步/异步方法分离**：所有操作（如上传、下载、删除）都是同步或基于 Future/Callback 的异步实现，内部已处理并发问题。

---

### 5. **使用建议**
尽管 `MinioClient` 是线程安全的，但在实际使用中仍需注意以下几点：
#### 5.1. ✅ **推荐做法**
- **单例模式**：在应用中创建一个 `MinioClient` 实例并全局共享，避免重复创建开销。
  ```java
  MinioClient minioClient = new MinioClient("https://play.min.io", "YOUR_ACCESS_KEY", "YOUR_SECRET_KEY");
  // 在多线程中共享此实例
  ```
- **合理配置连接池**：通过自定义 HTTP 客户端调整连接池大小，以适应高并发场景。
  ```java
  OkHttpClient httpClient = new OkHttpClient.Builder()
      .connectionPool(new ConnectionPool(10, 1, TimeUnit.MINUTES))
      .build();
  MinioClient minioClient = new MinioClient("https://play.min.io", "YOUR_ACCESS_KEY", "YOUR_SECRET_KEY", httpClient);
  ```

#### 5.2. ⚠️ **需避免的问题**
- **共享可变状态**：如果业务逻辑中需要维护与 MinIO 操作相关的共享状态（如缓存、计数器），需自行加锁或使用线程安全的数据结构（如 `ConcurrentHashMap`）。
- **异常处理**：确保每个线程的 MinIO 操作独立处理异常，避免因单个线程失败影响整体流程。

---

### 6. **验证线程安全性的方法**
如果官方文档未明确说明，可以通过以下方式验证：
1. **源码分析**：
   - 检查 `MinioClient` 类的实现，确认其内部方法是否使用 `synchronized`、`ReentrantLock` 或无状态设计。
   - 查看是否依赖线程安全的第三方库（如 Apache HttpClient）。
2. **单元测试**：
   - 编写多线程测试用例，同时执行上传、下载等操作，观察结果是否一致。
   - 使用工具（如 JMeter）模拟高并发请求，检查是否存在资源竞争或死锁。

---

### 7. **其他语言 SDK 的线程安全性**
如果你使用的是其他语言的 MinIO SDK（如 Python、Node.js、Go），其线程安全性可能有所不同：
- **Go**：`minio-go` SDK 是线程安全的，因为 Go 的 Goroutine 模型和 SDK 内部设计天然支持并发。
- **Python**：`minio` 库在单线程中是安全的，但在多线程中需注意全局解释器锁（GIL）的影响，建议使用异步（`asyncio`）或进程池。
- **Node.js**：基于事件循环的非阻塞设计，天然适合多线程（通过 `worker_threads`）。

---

### 8. 总结
**MinIO Java SDK 的 `MinioClient` 是线程安全的**，适合在多线程环境中直接共享使用。但需注意：
- 合理配置连接池以适应并发需求。
- 避免在客户端外部维护共享可变状态。
- 参考官方文档和源码，确认其他语言 SDK 的线程安全性。

如果需要进一步验证，建议通过源码分析或编写多线程测试用例进行确认。


---

# 二、实操

### 1. 基本使用

##### 1.1.1. 创建 Spring Web 项目

1. ==Web==：
	1. Spring Web

---


##### 1.1.2. 引入 Minio 相关依赖

由于目前未提供 `**-minio-starter` 依赖，官方推荐的方式是直接添加  [minio 依赖](https://mvnrepository.com/artifact/io.minio/minio) 依赖项。
```
<dependency>
    <groupId>io.minio</groupId>
    <artifactId>minio</artifactId>
    <version>8.5.17</version>
</dependency>
```

---


##### 1.1.3. 创建 Minio 配置类

```
@Configuration  
public class MinioConfig {  
      
    @Bean  
    public MinioClient minioClient() {  
        return MinioClient.builder()  
                .endpoint("192.168.136.8:9000")  
                .credentials("admin", "admin123")  
                .build();  
    }  
}
```

---


##### 编写业务代码

```
@RestController  
public class MinioTest {  
  
    @Autowired  
    public MinioClient minioClient;  
  
    @GetMapping("/test")  
    public String testMinio() {  
        return minioClient.toString();  
    }  
}
```

---


### 2. 业务处理

#### 创建 Bucket

#### 2.1. Bucket 基础操作

1. `minioClient.bucketExists(BucketExistsArgs args)`：
    1. 用于检查指定名称的存储桶是否存在，返回一个布尔值
    2. 返回 `true` 表示该存储桶存在，返回 `false` 表示该存储桶不存在，常用于创建前的预检查
2. `minioClient.makeBucket(MakeBucketArgs args)`：
    1. 创建一个新的存储桶，若该存储桶已存在则会抛出异常；方法无返回值（`void`）
3. `minioClient.listBuckets()`：
    1. 用于列出当前用户有权限访问的所有存储桶，返回一个包含 `Bucket` 对象的列表；每个 `Bucket` 对象包含存储桶的名称和创建时间等信息
4. `minioClient.removeBucket(RemoveBucketArgs args)`：
    1. 删除指定名称的存储桶；如果存储桶不存在，或不为空（即内部还有对象），则删除操作会抛出异常
```
@RestController  
public class MinioTest {  
  
    @Autowired  
    public MinioClient minioClient;  
  
    // 1. minioClient.bucketExists(BucketExistsArgs args)  
    @GetMapping("/bucketExists")  
    public Boolean bucketExists(@RequestParam String bucketName) {  
        try {  
            return minioClient.bucketExists(  
                BucketExistsArgs.builder().bucket(bucketName).build()  
            );  
        } catch (Exception e) {  
            throw new ResponseStatusException(  
                HttpStatus.INTERNAL_SERVER_ERROR, "查询存储桶时存在错误：" + e.getMessage(),e  
            );  
        }  
    }  
  
    // 2. minioClient.makeBucket(MakeBucketArgs args)  
    @PostMapping("/makeBucket")  
    public String makeBucket(@RequestParam String bucketName) {  
        try {  
            minioClient.makeBucket(  
                MakeBucketArgs.builder().bucket(bucketName).build()  
            );  
            return "存储桶：" + bucketName + "创建成功";  
        } catch (Exception e) {  
            throw new ResponseStatusException(  
                HttpStatus.INTERNAL_SERVER_ERROR, "创建存储桶时出错: " + e.getMessage(), e  
            );  
        }  
    }  
  
    // 3. minioClient.listBuckets()  
    @GetMapping("/listBuckets")  
    public List<String> listBuckets() {  
        try {  
            return minioClient.listBuckets().stream()  
                .map(bucket -> bucket.name() + "---" + bucket.creationDate()).collect(Collectors.toList());  
        } catch (Exception e) {  
            throw new ResponseStatusException(  
                    HttpStatus.INTERNAL_SERVER_ERROR,  
                    "列出存储桶时出错: " + e.getMessage(), e  
            );  
        }  
    }  
  
    // 4. minioClient.removeBucket(RemoveBucketArgs args)  
    @DeleteMapping("/removeBucket")  
    public String removeBucket(@RequestParam String bucketName) {  
        try {  
            minioClient.removeBucket(  
                RemoveBucketArgs.builder().bucket(bucketName).build()  
            );  
            return "存储桶删除成功";  
        } catch (Exception e) {  
            throw new ResponseStatusException(  
                HttpStatus.INTERNAL_SERVER_ERROR, "删除存储桶时出错: " + e.getMessage(), e  
            );  
        }  
    }  
}
```

---


#### 2.2. Object 基础操作

1. `minioClient.putObject(PutObjectArgs args)`：
    1. 上传文件到指定存储桶
    2. 如果上传成功，返回类型为 `void`（不返回内容）；如果出错会抛异常
2. `minioClient.statObject(StatObjectArgs args)`：
    1. 用于检查指定的 Object 的状态
	2. 返回 `StatObjectResponse` 对象，包含：
		1. 对象大小（size）
		2. Content-Type
		3. 最后修改时间（lastModified）
		4. ETag（内容摘要）
		5. 自定义元数据等信息
3. `minioClient.getPresignedObjectUrl(GetPresignedObjectUrlArgs args)`：
    1. 用于生成一个 Object 的签名 URL，以便通过 HTTP 访问
    2. 返回一个 `String` 类型的 URL，带有授权签名，支持临时访问私有资源
4. `minioClient.getObject(GetObjectArgs args)`：
    1. 用于从指定的存储桶中下载文件
    2. 返回一个 `InputStream`，可以读取文件的内容流；需手动关闭流以释放资源
5. `minioClient.listObjects(ListObjectsArgs args)`：
    1. 用于列出指定存储桶中的所有对象
    2. 返回一个 `Iterable<Result<Item>>`，每个 `Item` 表示一个对象，可通过 `item.objectName()` 获取对象名；注意：是惰性加载，适合处理大批量数据
6. `minioClient.removeObject(RemoveObjectArgs args)`：
    1. 用于删除指定存储桶中的对象，需要指定存储桶名称和对象键
    2. 返回类型为 `void`，操作成功不返回内容；失败时抛出异常


![](image-20250514084845465.png)
![](image-20250514081937819.png)

![](image-20250514081945868.png)


```
@RestController  
public class MinioTest {  
  
    @Autowired  
    public MinioClient minioClient;  
  
    // 1. minioClient.putObject(PutObjectArgs args)  
    @PostMapping("/putObject")  
    public String putObject(  
            @RequestParam String bucketName,  
            @RequestParam String objectName,  
            @RequestParam MultipartFile file) {  
        try (InputStream in = file.getInputStream()) {  
            minioClient.putObject(  
                PutObjectArgs.builder()  
                    .bucket(bucketName)  
                    .object(objectName)  
                    .stream(in, file.getSize(), -1)  
                    .contentType(file.getContentType())  
                    .build()  
            );  
            return "上传成功: " + objectName;  
        } catch (Exception e) {  
            throw new ResponseStatusException(  
                    HttpStatus.INTERNAL_SERVER_ERROR,  
                    "上传文件时出错: " + e.getMessage(), e  
            );  
        }  
    }  
  
  
    // 2. minioClient.statObject(StatObjectArgs args)  
    @GetMapping("/statObject")  
    public StatObjectResponse statObject(  
            @RequestParam String bucketName,  
            @RequestParam String objectName) {  
        try {  
            return minioClient.statObject(  
                StatObjectArgs.builder()  
                    .bucket(bucketName)  
                    .object(objectName)  
                    .build()  
            );  
        } catch (Exception e) {  
            throw new ResponseStatusException(  
                    HttpStatus.INTERNAL_SERVER_ERROR,  
                    "获取对象状态时出错: " + e.getMessage(), e  
            );  
        }  
    }  
  
    // 3. minioClient.getPresignedObjectUrl(GetPresignedObjectUrlArgs args)  
    @GetMapping("/getPresignedObjectUrl")  
    public String getPresignedObjectUrl(  
            @RequestParam String bucketName,  
            @RequestParam String objectName,  
            @RequestParam(defaultValue = "3600") int expiry) {  
        try {  
            return minioClient.getPresignedObjectUrl(  
                GetPresignedObjectUrlArgs.builder()  
                    .method(Method.GET)  
                    .bucket(bucketName)  
                    .object(objectName)  
                    .expiry(expiry)  // 这个 URL 的过期时间，单位是 秒  
                    .build()  
            );  
        } catch (Exception e) {  
            throw new ResponseStatusException(  
                    HttpStatus.INTERNAL_SERVER_ERROR,  
                    "生成签名 URL 时出错: " + e.getMessage(), e  
            );  
        }  
    }  
  
  
    // 4. minioClient.getObject（GetObjectArgs args）  
    @GetMapping("/getObject")  
    public ResponseEntity<InputStreamResource> getObject(  
            @RequestParam String bucketName,  
            @RequestParam String objectName) {  
        try {  
            InputStream in = minioClient.getObject(  
                    GetObjectArgs.builder()  
                            .bucket(bucketName)  
                            .object(objectName)  
                            .build()  
            );  
            return ResponseEntity.ok()  
                    .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"" + objectName + "\"")  
                    .body(new InputStreamResource(in));  
        } catch (Exception e) {  
            throw new ResponseStatusException(  
                    HttpStatus.INTERNAL_SERVER_ERROR,  
                    "下载文件时出错: " + e.getMessage(), e  
            );  
        }  
    }  
  
    // 5. minioClient.listObjects(ListObjectsArgs args)  
    @GetMapping("/listObjects")  
    public List<String> listObjects(  
            @RequestParam String bucketName,  
            @RequestParam(defaultValue = "false") boolean recursive) {  
        try {  
            Iterable<Result<Item>> results = minioClient.listObjects(  
                ListObjectsArgs.builder()  
                    .bucket(bucketName)  
                    .recursive(recursive)  
                    .build()  
            );  
  
            return StreamSupport.stream(results.spliterator(), false)  
                    .map(r -> {  
                        try {  
                            return r.get().objectName(); // get() 可能抛异常  
                        } catch (Exception e) {  
                            throw new RuntimeException("读取对象名称失败", e);  
                        }  
                    })  
                    .collect(Collectors.toList());  
  
        } catch (Exception e) {  
            throw new ResponseStatusException(  
                    HttpStatus.INTERNAL_SERVER_ERROR,  
                    "列出对象时出错: " + e.getMessage(), e  
            );  
        }  
    }  
  
    // 6. minioClient.removeObject(RemoveObjectArgs args)  
    @DeleteMapping("/removeObject")  
    public String removeObject(  
            @RequestParam String bucketName,  
            @RequestParam String objectName) {  
        try {  
            minioClient.removeObject(  
                RemoveObjectArgs.builder()  
                    .bucket(bucketName)  
                    .object(objectName)  
                    .build()  
            );  
            return "删除成功: " + objectName;  
        } catch (Exception e) {  
            throw new ResponseStatusException(  
                    HttpStatus.INTERNAL_SERVER_ERROR,  
                    "删除对象时出错: " + e.getMessage(), e  
            );  
        }  
    }  
}
```





哦！你是在问：

> 「`minioClient.getPresignedObjectUrl(...)` 这个方法到底**干嘛用的**？」

好，我不卖关子，通俗直说👇

---

## 🎯 简洁答案：

> 这个方法用来生成一个**带授权签名的临时访问链接**，让**不登录 MinIO 的用户也能访问私有文件**。

---

## 🧠 举例说明（让你秒懂）：

比如你有一个保存在 MinIO 里的文件：

```
bucket: my-private-bucket  
object: /docs/secret.pdf
```

这个文件是私有的，默认别人看不到。但你想：

> “我给用户发个下载链接，他点一下就能下载，**不用登录、也不用发 token**。”

怎么办？用这个方法：

```java
String url = minioClient.getPresignedObjectUrl(
    GetPresignedObjectUrlArgs.builder()
        .method(Method.GET)
        .bucket("my-private-bucket")
        .object("docs/secret.pdf")
        .expiry(60 * 10)  // 10分钟有效
        .build()
);
```

🔗 它就会返回一个长长的链接，大概像这样：

```
https://minio.example.com/my-private-bucket/docs/secret.pdf?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=...
```

你把这个链接丢给前端、微信、邮件、钉钉，谁点都能直接访问这个私有文件。

---

## 📦 常见用途：

|场景|描述|
|---|---|
|✅ 下载文件|前端或第三方用户访问私有文件|
|✅ 预览图片/音频/视频|在不暴露 MinIO 密钥的情况下播放|
|✅ 前端直传（Method.PUT）|允许用户直接把文件上传到 MinIO，不经过后端|
|✅ 生成限时分享链接|链接有效期自动失效，不怕泄露|

---

## 💡一句话总结：

> `getPresignedObjectUrl(...)` 就是帮你生成一个“有时效、有权限”的外链链接，用来**安全地分享、下载或上传 MinIO 里的私有文件**。

---

想我顺便也讲讲 `Presigned PUT` 用于前端直传的逻辑吗？那玩意跟这个兄弟是亲戚 😄





### 3. 相关网站

1. minio 依赖地址：
	1. https://mvnrepository.com/artifact/io.minio/minio

















# 高级玩法

简单的stream 上传

![](image-20250513175631311.png)




![](image-20250514083308199.png)














