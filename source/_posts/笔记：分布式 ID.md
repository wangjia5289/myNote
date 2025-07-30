---
title: 笔记：分布式 ID
date: 2025-07-12
categories:
  - Java
  - 分布式与微服务
  - 分布式 ID
tags: 
author: 霸天
layout: post
---
## UUID

### UUID 概述

UUID 是一个固定长度为 128 位（16 字节）的通用唯一识别码，通常以 32 个十六进制字符表示，并使用 4 个连字符分隔为 5 组，例如：
```
// 32 个十六进制数字 + 4 个 - 连字符（共 36 个字符）
xxxxxxxx-xxxx-Mxxx-Nxxx-xxxxxxxxxxxx
"""
1. x
	1. 十六进制数字（0 ~ f）
2. M
	1. 版本位，UUID 的版本号（1 ~ 5）
3. N
	1. 变体位，UUID 的布局规则
"""
```

UUID 的设计目标是：在全球范围内生成的 UUID 都不会重复。其一共有 5 个标准版本（还有部分扩展版本不常用）：
1. v1
	1. 基于当前时间戳 + 当前设备的 MAC 地址生成
2. v2
	1. 很少使用，基于 POSIX UID/GID
3. v3
	1. 用名字 + 命名空间，用 MD5 哈希生成
4. v4
	1. 最常用，通过随机数生成
5. v5
	1. 类似 v3，但用 SHA-1 替代 MD5

其优点是：
1. 跨平台、跨语言兼容性好，广泛支持
2. 不易被预测，安全性高，几乎不可能重复
3. 可以在本地独立生成，不依赖数据库或网络

其缺点是：
1. 占用空间大，需要 128 位（bit）
2. 可读性差，不便于人工识别或排序

> [!NOTE] 注意事项
> 1. `xxxxxxxx-xxxx-Mxxx-Nxxx-xxxxxxxxxxxx` 是 UUID 的标准字符串表示，共有 36 个字符组成，但在底层，UUID 实际只保存了 128 位的数据，连字符仅用于显示时的格式化

---


### UUID v1

UUID v1 是一种 **基于时间戳 + 设备信息（MAC 地址）** 的生成方式，其组成结构为：
1. 时间戳（60 位，精确到 100 ns）
	1. 时间戳是从 1582-10-15 00:00:00 UTC 起开始计时（这是 UUID 规范规定的纪元时间）
	2. 单位是 100 纳秒（即每秒有 10^7 个单位），因此精度很高，可以在极短时间内生成大量唯一 ID
2. 设备信息（48 位，MAC 地址）
	1. 通常是本机的 MAC 地址
	2. 例如：`00:0a:95:9d:68:16` → 48 位 进行填入
	3. 注意事项：如果获取不到 MAC 地址，有些库会用随机数代替
3. 时钟序列（14 位）
	1. 防止系统时间回拨或时间没有前进，从而造成重复 UUID，例如：
		1. 手动调整时间
			1. 用户手动更改系统时间
		2. NTP 同步
			1. NTP 同步可能会导致时间小幅度回拨
		3. 时间没有前进
	2. 如果发现当前时间戳 ≤ 上一次生成 UUID 的时间戳，说明系统时间回拨或没有前进
		1. 为了确保生成的 ID 仍然唯一，此时时钟序列会被增加 1
		2. 为了能够读取上一次生成 UUID 的时间戳，通常会将其保存在文件、内存或数据库中
4. 版本信息（4 位）
	1. 对于 UUID v1，固定为 0001
5. 变体信息（2 位）
	1. UUID 规范定义了几种变体，其中最常用的是 DCE/RFC 4122 变体

其优点是：
1. 唯一性好
	1. 不同设备 + 高精度时间
2. 有序性好
	1. 按时间递增，适合数据库索引

其缺点是：
1. 泄漏隐私
	1. 能看出生成的时间，甚至能暴露设备 MAC 地址
2. 时钟回拨问题
	1. 需要额外时钟序列来补偿

Java UUID 标准库 `java.util.UUID` 默认不支持 v1，只支持 v4，我们可以使用第三方库 `uuid-creator`：添加 [uuid-creator 依赖](https://mvnrepository.com/artifact/com.github.f4b6a3/uuid-creator)
```
<dependency>
    <groupId>com.github.f4b6a3</groupId>
    <artifactId>uuid-creator</artifactId>
    <version>6.0.0</version>
</dependency>
```

代码如下：
```
public class UUIDV1 {  
  
    public static void main(String[] args) {  
  
        UUID uuidV1 = UuidCreator.getTimeBased();  
        System.out.println(uuidV1);  
    }  
}
```

---


### UUID v2

UUID v2 是 UUID 中**最少见、最不推荐使用**的一种版本，大部分 Java 类库都**不支持 UUID v2**，包括 `uuid-creator` 库。这里干脆不使用 UUID v2，改用其他版本

---


### UUID v3

UUID v3 是一种 **基于命名空间（namespace）+ 名称（name）+ MD5 哈希算法** 生成的 UUID，具有**稳定性强、可复现**的特点。也就是说，它不是随机生成的，而是**可预测、可重现**的：**只要命名空间和名称的组合相同，生成的 UUIDv3 始终一致**！

UUID v3 的本质是：UUID = MD5(namespace + name) → 得到 128 位结果，然后将特定位替换为版本号和变体信息

UUID 规范中预定义了一些命名空间 UUID：
1. NAMESPACE_DNS
	1. 用于 DNS 域名
2. NAMESPACE_URL
	1. 用于 URL
3. NAMESPACE_OID
	1. 用于 OID (对象标识符)
4. NAMESPACE_X500_DN
	1. 用于 X.500 DN (目录名)

在 `uuid-creator` 工具中也对这些命名空间进行了封装，例如可以通过 `UuidNamespace.NAMESPACE_X500` 访问这些枚举类的属性。

当然，我们也可以自定义一个 UUID（通常是 UUID v4）作为命名空间使用，例如：
```
UUID namespace = UUID.randomUUID();  
```

其优点是：
1. 可预测、可重现
	1. 适用于需要重复输入相同 UUID 的场景
2. 基于标准 MD5 哈希算法，计算性能高

其缺点是：
1. 随机性较差
	1. 与 `UUID v4` 这种完全基于随机数生成的 UUID 相比，`UUID v3` 确实没有随机性
2. MD5 已被认为不够安全，容易发生碰撞
	1. 这里的 “碰撞” 是指输入不同的数据，经过哈希计算后，却会产生相同的哈希输出。
	2. 如果两个不同的 “命名空间 + 名称” 组合经过 MD5 哈希后，产生了相同的 128 位哈希值，那么它们就会生成相同的`UUID v3`
	3. 虽然实际应用中遇到这种 “意外” 碰撞的概率极低，但MD5 的弱点使得它不再适合高安全要求的场景，例如授权码等
	4. 如果对哈希安全性有要求，推荐使用 UUID v5（使用 SHA-1 哈希算法）

Java UUID 标准库 `java.util.UUID` 默认不支持 v3，只支持 v4，我们可以使用第三方库 `uuid-creator`：添加 [uuid-creator 依赖](https://mvnrepository.com/artifact/com.github.f4b6a3/uuid-creator)
```
<dependency>
    <groupId>com.github.f4b6a3</groupId>
    <artifactId>uuid-creator</artifactId>
    <version>6.0.0</version>
</dependency>
```

代码如下：
```
public class UUIDV3 {  
      
    public static void main(String[] args) {  
  
        // 自定义 namespace（使用 UUID V4 生成随机 UUID）  
        UUID namespace = UUID.randomUUID();  
          
        UUID uuidV3 = UuidCreator.getNameBasedMd5(namespace,"example.com");  
        System.out.println(uuidV3);  
    }  
}
```

---


### UUID v4

UUID v4 是完全基于高质量的随机数（pseudo-random），没有任何时间戳、MAC 地址、命名空间等信息，可以理解为：UUID v4 = 随机生成 122 位 + 插入版本号和变体位

Java UUID 标准库 `java.util.UUID` 默认支持 v4，代码如下：
```
public class UUIDV4 {  
  
    public static void main(String[] args) {  
          
        UUID uuidV4 = UUID.randomUUID();  
        System.out.println(uuidV4);  
    }  
}
```

----


### UUID v5

与 UUID v3 类似，但是不是使用 MD5，而是使用 SHA-1

Java UUID 标准库 `java.util.UUID` 默认不支持 v5，只支持 v4，我们可以使用第三方库 `uuid-creator`：添加 [uuid-creator 依赖](https://mvnrepository.com/artifact/com.github.f4b6a3/uuid-creator)
```
<dependency>
    <groupId>com.github.f4b6a3</groupId>
    <artifactId>uuid-creator</artifactId>
    <version>6.0.0</version>
</dependency>
```

代码如下：
```
public class UUIDV5 {  
  
    public static void main(String[] args) {  
  
        // 自定义 namespace（使用 UUID V4 生成随机 UUID）  
        UUID namespace = UUID.randomUUID();  
  
        UUID uuidV5 = UuidCreator.getNameBasedSha1(namespace,"example.com");  
        System.out.println(uuidV5);  
    }  
}
```

---














