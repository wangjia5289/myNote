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
## 1. UUID

### 1.1. UUID 概述

UUID 是一个固定长度为 128 位（16 字节）的通用唯一识别码，通常以 32 个十六进制字符表示，并使用 4 个连字符分隔为 5 组，例如：
```
// 32 个十六进制数字 + 4 个 - 连字符（共 36 个字符）
xxxxxxxx-xxxx-Mxxx-Nxxx-xxxxxxxxxxxx
    |      |    |    |        |
	8      4    4    4        16
"""
1. x
	1. 十六进制数字（0 ~ f）
2. M
	1. 版本位，UUID 的版本号（1 ~ 5）
3. N
	1. 变体位，UUID 的布局规则
	2. 这里表面上是 4 位，但实际上变体信息通常只占其中的 2 位或 3 位
	3. 剩下的 2 位或 1 位是数据位，只是这里为了简化，直接把这 4 位都视作变体位来看了
	4. Variant 0（Apollo NCS 兼容）
		1. 二进制：
			1. 0xxx
	5. Variant 1（RFC 4122 标准）
		1. 二进制
			1. 10xx
	6. Variant 2（Microsoft GUID 历史兼容）
		1. 二进制
			1. 110x
"""
```

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


### 1.2. UUID v1

UUID v1 是一种 **基于 UUID 时间戳 + 设备信息（MAC 地址）** 的生成方式，其组成结构为：
1. 时间戳（60 位，精确到 100 ns）
	1. v1 使用的时间戳是 UUID 时间戳，从 1582-10-15 00:00:00 UTC 起开始
	2. 精确到的单位是 100 纳秒（即每秒有 10^7 个单位），因此精度很高，可以在极短时间内生成大量唯一 ID
2. 版本位（4 位）
	1. 固定为（0001）<sub>2</sub>
3. 变体位
	1. 使用的变体是 Variant 1（RFC 4122 标准）
4. 时钟序列（14 位）
	1. 防止系统时间回拨或时间没有前进，从而造成重复 UUID，例如：
		1. 手动调整时间
			1. 用户手动更改系统时间
		2. NTP 同步
			1. NTP 同步可能会导致时间小幅度回拨
		3. 时间没有前进
	2. 如果发现当前时间戳 ≤ 上一次生成 UUID 的时间戳，说明系统时间回拨或没有前进
		1. 为了确保生成的 ID 仍然唯一，此时时钟序列会被增加 1
		2. 为了能够读取上一次生成 UUID 的时间戳，通常会将其保存在文件、内存或数据库中
5. 设备信息（48 位，MAC 地址）
	1. 通常是本机的 MAC 地址
	2. 例如：`00:0a:95:9d:68:16` → 48 位 进行填入
	3. 注意事项：如果获取不到 MAC 地址，有些库会用随机数代替

![](image-20250730100600919.png)

其优点是：
1. 唯一性好
	1. 不同设备 + 高精度时间
2. 有序性好
	1. 按时间递增，适合数据库索引

其缺点是：
1. 隐私泄露
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


### 1.3. UUID v2

UUID v2 是 UUID 中**最少见、最不推荐使用**的一种版本，大部分 Java 类库都**不支持 UUID v2**，包括 `uuid-creator` 库。这里干脆不使用 UUID v2，改用其他版本

---


### 1.4. UUID v3

UUID v3 是一种 **基于命名空间（namespace）+ 名称（name）+ MD5 哈希算法** 生成的 UUID，具有**稳定性强、可复现**的特点。也就是说，它不是随机生成的，而是**可预测、可重现**的：**只要命名空间和名称的组合相同，生成的 UUIDv3 始终一致**！

UUID v3 的本质是：UUID = MD5(namespace + name) → 得到 128 位结果，然后将特定位替换为版本位（固定为（0011）<sub>2</sub>）和变位信息（使用的变体是 Variant 1，即RFC 4122 标准）

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
1. 无序性（不可排序），不适合做数据库主键
	1. 产生索引碎片
	2. 降低写入性能
	3. 降低查询性能
	4. 缓存效率低下
	5. 增加存储空间
2. 随机性较差
	1. 与 `UUID v4` 这种完全基于随机数生成的 UUID 相比，`UUID v3` 确实没有随机性
3. MD5 已被认为不够安全，容易发生碰撞
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


### 1.5. UUID v4（常用）

UUID v4 是完全基于高质量随机数（pseudo-random）生成的，可以理解为：UUID v4 = 随机生成 122 位，然后在特定位置上添加版本位（固定为（0100）<sub>2</sub>）和变体位（使用的变体是 Variant 1，即RFC 4122 标准）

其优点是：
1. 真正随机
2. 生成速度快
3. 无隐私泄露
4. 碰撞概率极低
	1. 想要碰撞一个 UUID v4，大约需要生成 2⁶⁴ 个 UUID 才有 50% 的概率碰撞上（生日驳论）

其缺点是：
1. 无序性（不可排序），不适合做数据库主键
	1. 产生索引碎片
	2. 降低写入性能
	3. 降低查询性能
	4. 缓存效率低下
	5. 增加存储空间

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


### 1.6. UUID v5

UUID v5 与 UUID v3 类似，区别在于它使用更安全的 SHA-1 哈希算法，而不是 MD5。

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


### 1.7. UUID v6

UUID v6 是 UUID v1 的升级版，它采用了更现代、更有序、更注重隐私的方式生成基于时间的 UUID，特别适合用于数据库主键、日志追踪等需要顺序性的场景。  

不过，现在我们有了功能更强、设计更先进的 UUID v7，因此这里就不再展开介绍 UUID v6。

---


### 1.8. UUID v7（推荐）

UUID v7 是目前中**最推荐**的一种 UUID 版本，是为了适应现代系统而设计的 “新一代时间型 UUID”，基于 **UNIX 时间戳（毫秒）+ 高随机性**，兼顾 **全局唯一、生成简单、支持排序、无隐私泄露** 四大优势，其组成结构为：
1. 时间戳（48 位，精确到 ms）
	1. v7 使用的时间戳是标准的 Unix 时间戳，从 1970-01-01 00:00:00 UTC 起开始
	2. 精确到的单位是 ms，即每秒有 1000 个单位
2. 版本位（4位）
	1. 固定为（0111）<sub>2</sub>
3. 毫秒内序列（12 位）
	1. 表示当前这一个毫秒内生成的是第几个 UUID
	2. 2<sup>12</sup> = 4096，可表示 0 → 4095，也就是说每毫秒支持 4096 个序列
4. 变体位（2 位）
	1. 使用的变体是 Variant 1（RFC 4122 标准）
5. 随机数（62 位）
	1. 加强唯一性
	2. 即使系统时间回拨，或者毫秒内序列冲突，也能保证 UUID 的唯一性

其缺点是：
1. 对系统时钟准确性的隐式信任
	1. 虽然我们有 62 位随机数来保障 UUID 的唯一性，但如果系统时钟频繁且大幅度地不准确，可能会导致 UUID 的**时间排序性不可靠**，从而影响其在数据库索引中的性能表现

Java UUID 标准库 `java.util.UUID` 默认不支持 v7，只支持 v4，我们可以使用第三方库 `uuid-creator`：添加 [uuid-creator 依赖](https://mvnrepository.com/artifact/com.github.f4b6a3/uuid-creator)
```
<dependency>
    <groupId>com.github.f4b6a3</groupId>
    <artifactId>uuid-creator</artifactId>
    <version>6.0.0</version>
</dependency>
```

代码如下：
```
public class UUIDV7 {  
  
    public static void main(String[] args) {  
  
        UUID uuidV7 = UuidCreator.getTimeOrderedEpoch();  
        System.out.println(uuidV7);  
    }  
}
```

---






