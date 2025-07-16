---
title: 笔记：Minio 基础
date: 2025-05-13
categories:
  - 数据管理
  - 对象存储
  - Minio
  - Minio 基础
tags: 
author: 霸天
layout: post
---
## Minio 概述

----


## Minio 语法

### 系统命令

<font color="#92d050">1. 连接 Minio 服务（配置 Minio 服务别名）</font>
```
mc alias set <Minio 服务别名> <Minio 地址> <用户名> <密码>
“”“
1. 示例值：
	1. mc alias set myminio http://127.0.0.1:9000 root wq666666
”“”
```

> [!NOTE] 注意事项
> 1. 给 MinIO 服务起别名 `myminio`，以后就可以通过 `myminio` 来访问，而不用每次输入 URL 和账号密码：
```
// 1. 使用 Minio 服务别名
mc cp ./file.txt myminio/mybucket/


// 2. 不使用 Minio 服务别名
mc --insecure --config-dir /root/.mc cp ./file.txt http://127.0.0.1:9000/mybucket/file.txt --access-key root --secret-key wq666666
```

----


### 集群命令

<font color="#92d050">1. 查看服务信息（基本健康状态）</font>
```
mc admin info <alias>
```
![](image-20250715224906230.png)

---


### 用户命令

<font color="#92d050">1. 添加用户</font>
```
mc admin user add <alias> <username> <password>
“”“
1. 示例值：
	1. mc admin user add myminio alice StrongPass123
2. 注意事项：
	1. 添加的用户，默认是启用的
	2. 被禁用的用户，才需要启用
”“”
```


<font color="#92d050">2. 禁用用户</font>
```
mc admin user disable <alias> <username>
```


<font color="#92d050">3. 启用用户</font>
```
mc admin user enable <alias> <username>
```


<font color="#92d050">4. 删除用户</font>
```
mc admin user remove <alias> <username>
```


<font color="#92d050">5. 列出所有用户</font>
```
mc admin user list <alias>
```


<font color="#92d050">6. 为用户设置权限</font>


<font color="#92d050">7. 查看用户的权限</font>


<font color="#92d050">8. 修改 Root 密码</font>
```
mc admin user password <alias> <username> <newpassword>
“”“
1. 注意事项：
	1. Root 密码长度至少 8 个字符
”“”
```

----


### 用户组命令

---


### Bucket 操作命令

<font color="#92d050">1. 创建 Bucket</font>
```
mc mb <alias>/<bucket>
“”“
1. alias：
	1. 配置好的 Minio 服务别名
2. 示例：
	1. mc mb myminio/mybucket
3. 注意事项：
	1. Bucket 名字长度至少 3 个字符
”“”
```


<font color="#92d050">2. 删除 Bucket</font>
```
// 1. 删除空 Bucket（不能删除有内容的 Bucket）
mc rb <alias>/<bucket>


// 2. 强制删除 Bucket
mc rb <alias>/<bucket> --force
```


<font color="#92d050">3. 列出当前用户可访问的所有 Bucket</font>
```
mc ls <alias>
“”“
1. 示例：
	1. mc ls myminio
”“”
```


<font color="#92d050">4. 列出某 Bucket 中当前用户可访问的所有 Object</font>
```
mc ls <alias>/<bucket>
“”“
1. 示例：
	1. mc ls myminio/mybucket
2. 注意事项：
	1. 加上 --versions 表示查看对象的所有版本
”“”
```


<font color="#92d050">5. 设置 Bucket 策略</font>
```
// 1. 内置策略
mc anonymous set [download|upload|public|private] <alias>/<bucket>
“”“
1. download：
	1. 允许匿名用户下载指定 bucket 中的对象，但不允许删除、上传、列出 bucket 中的对象
	2. 下载对象时，必须知道对象的完整路径，例如 http://minio.example.com/mybucket/myfile.txt
2. upload：
	1.允许匿名用户向指定 bucket 上传对象，但不允许删除、下载、列出 bucket 中的对象
3. public：
	1. 允许匿名用户对指定 bucket 进行完整操作，包括上传、下载、列出和删除对象
4. private：
	1. 完全禁止匿名用户访问指定 bucket，所有操作（上传、下载、列出、删除）均不允许
”“”


// 2. 自定义访问策略
推荐使用 SDK 设置，详见下文：Bucket 自定义访问策略
```


<font color="#92d050">6. 获取 Bucket 策略</font>
```
mc anonymous get <alias>/<bucket>
```


<font color="#92d050">7. 设置 Bucket 标签</font>
`mc` 没有直接设置 Bucket 标签的命令，但可以通过 SDK 操作


<font color="#92d050">8. 获取 Bucket 标签</font>
`mc` 没有直接显示 Bucket 标签的命令，但可以通过 SDK 操作


<font color="#92d050">9. 启用 Bucket 版本控制</font>
```
mc version enable <alias>/<bucket-name>
```


<font color="#92d050">10. 禁用 Bucket 版本控制</font>
```
mc version suspend <alias>/<bucket-name>
```


<font color="#92d050">11. 设置 Bucket 生命周期</font>
推荐使用 SDK 设置，详见下文：Bucket 生命周期


<font color="#92d050">12. 获取 Bucket 生命周期</font>
`mc` 没有直接显示 Bucket 标签的命令，但可以通过 SDK 操作

---


### Object 操作命令

<font color="#92d050">1. 上传对象</font>
```
// 1. 上传文件
mc cp <本地文件> <alias>/<bucket>/
“”“
1. 示例：
	1. mc cp ./report.pdf myminio/mybucket/
”“”


// 2. 上传整个目录
mc cp -r <本地目录> <alias>/<bucket>/
“”“
1. 示例：
	1. mc cp -r ./images/ myminio/mybucket/photos/
”“”
```


<font color="#92d050">2. 下载对象</font>
```
mc cp <alias>/<bucket>/<object> <本地路径>
“”“
1. 示例：
	1. mc cp myminio/mybucket/report.pdf ./downloads/
”“”
```


<font color="#92d050">3. 列出对象</font>
```
// 1. 列出根目录下的对象和子目录
mc ls <alias>/<bucket>/


// 2. 递归列出桶内所有对象，包括所有子目录和子目录中的文件
mc ls --recursive <alias>/<bucket>/
```


<font color="#92d050">4. 查看对象元数据</font>
```
mc stat <alias>/<bucket>/<object>
“”“
1. 示例：
	1. mc stat myminio/mybucket/report.pdf
”“”
```


<font color="#92d050">5. 删除对象</font>
```
// 1. 删除单个对象
mc rm --force <alias>/<bucket>/<object>
“”“
1. 示例：
	1. mc rm --force myminio/mybucket/report.pdf
2. 注意事项：
	1. 默认会提示确认，你可以按提示确认或者直接加 --force 跳过
”“”


// 2. 递归强制删除目录及所有内容
mc rm --recursive --force <alias>/<bucket>/<prefix/>
“”“
1. 示例：
	1. mc rm --recursive --force myminio/mybucket/logs/2024/
”“”
```


<font color="#92d050">6. 移动 / 重命名对象</font>
```
mc mv <alias>/<bucket>/<source> <alias>/<bucket>/<destination>
“”“
1. 示例：
	1. mc mv myminio/mybucket/file1.txt myminio/mybucket/file2.txt
”“”
```


<font color="#92d050">7. 同步目录</font>
实现本地目录与 Minio 同步，达到备份的效果
```
// 1. 将本地目录的内容完整同步到 MinIO
mc mirror <本地目录> <alias>/<bucket>/
“”“
1. 示例：
	1. mc mirror ./localdir myminio/bucket/
2. 注意事项：
	1. 会删除桶中多余但本地不存在的文件
”“”


// 2. 将桶内内容完整同步到本地目录
mc mirror <alias>/<bucket>/ <本地目录>
“”“
1. 示例：
	1. mc mirror myminio/bucket/ ./localdir
2. 注意事项：
	1. 删除本地多余但桶中不存在的文件
”“”
```


<font color="#92d050">8. 生成临时访问 URL（预签名 URL）</font>
```
mc presign <alias>/<bucket>/<object> --expiry <过期秒数>
“”“
1. 示例：
	1. mc presign myminio/mybucket/file.zip --expiry 3600
”“”
```


<font color="#92d050">9. 设置对象标签</font>
```
mc tag set <alias>/<bucket>/<object> "<key>=<value>&<key>=<value>"
“”“
1. 注意事项：
	1. 会覆盖之前的 对象 标签
”“”
```


<font color="#92d050">10. 查看对象标签</font>
```
mc tag get <alias>/<bucket>/<object>
```


<font color="#92d050">11. 删除对象标签</font>
```
mc tag remove <alias>/<bucket>/<object>
```


<font color="#92d050">12. 拷贝对象</font>
```
mc cp <alias1>/<bucket1>/<object> <alias2>/<bucket2>/
```

---


## Bucket

### Bucket 概述

Bucket 是存储 Object 的逻辑空间，每个 Bucket 之间的数据是相互隔离的，对用户而言，相当于存放文件的顶层**文件夹**

----


### Bucket 自定义访问策略

Bucket 内置策略是 MinIO 提供的简化权限控制方式之一，但我们也可以通过自定义 JSON 实现更细粒度、更复杂的访问控制逻辑。需要注意的是，Bucket 策略采用覆盖式配置，即每次设置都会替换上一次的策略内容，因此同一时间只能存在一个生效策略。

当请求为不带凭证的匿名访问时，系统只会依据 Bucket 策略来判断是否允许操作。而对于带有凭证的认证请求，则会**先检查用户策略**，再检查**对应的 Bucket 策略**，**两者必须同时允许**，请求才会被执行。正因为这种双重判断机制，就可能出现这样一种情况：匿名用户可以访问某对象，但认证用户因用户策略未授权而无法访问相同资源。

我们一般通过 MinIO SDK 为指定的 Bucket 设置自定义策略：
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

这里需要传入完整的 JSON 策略，示例请求体格式如下：
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
表示我们写的这个策略是允许还是禁止，可选值有：
1. Allow
2. Deny


<font color="#92d050">3. Principal</font>
指定哪些用户可以执行或拒绝执行某些操作，常见的 `Principal` 取值包括：
1. * 
	1. 表示所有用户，包括匿名用户和认证用户
2. user-arn
	1. 表示具体某个用户的用户名，用于精确控制该用户的权限

> [!NOTE] 注意事项
> 1. AWS 是指用户
> 2. 是可以执行还是拒绝执行，取决于策略中的 `Effect` 字段：若为 `Allow`，表示允许执行相应操作；若为 `Deny`，则表示明确禁止执行。


<font color="#92d050">4. Action</font>
用于指定允许或拒绝执行的具体操作，常见操作包括：
1. Bucket 级操作：
	1. s3:ListBucket
		1. 列出桶中的对象列表
	2. s3:GetBucketLocation
		1. 获取桶的所属区域（Region）信息
	3. s3:ListBucketMultipartUploads
		1. 查看该桶中正在进行的多部分（分片）上传任务
	4. s3:GetBucketPolicy
		1. 查看桶的访问策略
	5. s3:PutBucketPolicy
		1. 设置桶的访问策略
	6. s3:DeleteBucketPolicy
		1. 删除桶的访问策略
2. Object 级操作：
	1. s3:GetObject
		1. 下载对象
	2. s3:PutObject
		1. 上传对象
	3. s3:DeleteObject
		1. 删除对象
	4. s3:AbortMultipartUpload
		1. 取消一个正在进行的分片上传任务
	5. s3:ListMultipartUploadParts
		1. 查看指定分片上传任务中已上传的部分
	6. s3:GetObjectVersion
		1. 获取对象的指定版本（MinIO 默认未启用版本控制，需要我们开启）
3. s3:*
	1. 表示允许所有操作

> [!NOTE] 注意事项
> 1. 获取桶的区域信息，即判断桶属于哪个 Region。
> 2. 在 MinIO 社区版中，Region 往往是“虚拟”的（默认通常为 `us-east-1`），但权限上仍然要考虑是否开放，但若未开放 `s3:GetBucketLocation` 权限，某些 SDK 或 CLI 工具可能因无法获取区域而返回 403 错误。


<font color="#92d050">5. Resource</font>
用于指定允许或拒绝操作的具体资源，常见资源包括：
1. arn:aws:s3:::my-bucket
	1. 表示桶本身，适用于桶级操作，如 `s3:ListBucket`、`s3:GetBucketPolicy` 等
2. arn:aws:s3:::my-bucket/*
	1. 示该桶内的所有对象，适用于对象级操作，如 `s3:GetObject`、`s3:PutObject` 等
3. arn:aws:s3:::my-bucket/public/*
	1. 表示桶中某个前缀路径下的对象，例如 `public` 文件夹下的所有对象，可用于限制访问特定目录内容

----


### Bucket 生命周期

其实，更准确地说，这个操作是针对 Object 生命周期的设置，但从操作维度上来看，它仍归类为 Bucket 生命周期。

需要特别注意的是，Bucket 生命周期配置和 Bucket 策略一样，**都是整体性的配置**，每次设置都会**覆盖之前的所有规则**。我们可以通过 SDK 为指定的 Bucket 配置生命周期策略：
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

这里我们需要传入 XML 格式的配置，客户端请求体示例如下：
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
根标签，表示这是生命周期配置文件，xmlns 指定了该 XML 遵循 S3 生命周期配置的标准格式。


<font color="#92d050">2. ID</font>
规则（Rule）的唯一标识，用于管理区分，长度最多支持 255 个字符。


<font color="#92d050">3. Statue</font>
用于启用或禁用该条规则，可选值包括：
1. Enabled
2. Disabled


<font color="#92d050">4. Filter</font>
用于设置对象的过滤条件，从而精确控制生命周期规则的应用对象。常见选项包括：
1. Prefix
	1. 指定对象键名前缀，匹配所有以该前缀开头的对象。
	2. 例如 `<Prefix>temp/</Prefix>` 表示匹配所有以 `temp/` 开头的对象。
2. And
	1. 用于组合多个过滤条件，通常结合前缀（Prefix）和标签（Tag）共同生效。
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
> 1. 标签（Tag）是对象上的，而不是 Bucket 上的


<font color="#92d050">5. Expiration</font>
用于设置对象的过期策略，当对象达到指定时间后将被自动删除
```
// 1. 对象创建后，指定天数过期
<Expiration>
  <Days>30</Days>
</Expiration>


// 2. 在固定日期后过期，必须使用符合 ISO 8601 标准的时间格式（这里表示对象将在 2025 年 12 月 31 日 00:00:00.000（UTC 时间）过期并被删除）
<Expiration>
  <Date>2025-12-31T00:00:00.000Z</Date>
</Expiration>
```


<font color="#92d050">6. NoncurrentVersionExpiration</font>
适用于**开启版本控制的 Bucket**，当对象成为旧版本后，在指定天数内自动删除该旧版本。只支持 `<NoncurrentDays>` 设置天数：
```
<NoncurrentVersionExpiration>
  <NoncurrentDays>30</NoncurrentDays>
</NoncurrentVersionExpiration>
```


<font color="#92d050">7. AbortIncompleteMultipartUpload</font>
用于自动清理**未完成的分片上传任务**，从分片上传发起时间算起，超过指定天数后自动终止，避免浪费存储空间。只支持 `<DaysAfterInitiation>` 设置天数：
```
<AbortIncompleteMultipartUpload>
  <DaysAfterInitiation>3</DaysAfterInitiation>
</AbortIncompleteMultipartUpload>
```

---


## Object

### Object 概述

Object 是存储到 Minio 中的基本对象，对用户而言，相当于**文件**

----

