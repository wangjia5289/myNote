---
title: 笔记：Java 实现加密、解密、编码、解码
date: 2025-07-03
categories: 
tags: 
author: 霸天
layout: post
---
## 加密










## 编码

### Base64 编码

#### Base64 编码概述

Base64 使用一个**包含 64 个字符的字符表**，将任意二进制数据映射成可打印字符。简单来说，当我们对字符串 `abc` 进行 Base64 编码时：
```
String original = "abc";
```

在内存中，这个字符串对应的字节数组（UTF‑8）是：
```
[01100001, 01100010, 01100011]
   a                 b              c
```

Base64 的编码过程是将每 3 个字节（24 位） 划分为 4 组×6 位
```
[01100001][01100010][01100011] // 原始字节
		↓↓↓ 拆成 ↓↓↓
[011000][010110][001001][100011] // 每组 6 位
```

然后分别在下面这个 64 字符表中，查表并输出 4 个字符
```
A-Z（26个）  
a-z （26个）  
0-9 （10个）  
+ 和 /
```

这里 abc 在内存中正好就是 3 个字节（24位），然后进行 Base64 编码：
```
String original = "abc";


String encoded = Base64.getEncoder().encodeToString(original.getBytes("UTF-8")); // original.getBytes(StandardCharsets.UTF_8) 返回的是 byte[] 字节数组，也就是在内存中的二进制数据
```

编码后的结果为：
```
YWJj
```

编码后的结果占用的内存是 **4 字节（32位）**

> [!NOTE] 注意事项
> 1. 编码后字符数 = ((原始字节数 + 2) / 3) × 4   // 注意是整数除法（向下取整、截断小数）
> 2. 在 UTF-8 编码下，一个英文字符占用 1 个字节，因此如果希望 Base64 编码后的字符数为 64 个，那么原始数据的字节数应为 48 字节，也就是 48 个英文字符
> 3. 需要注意的是，Base64 编码结果中可能会出现 `=` 号，它不属于 Base64 的 64 个字符表，这是由于 Base64 是将每 3 字节（24 位）转换成 4 个 Base64 字符（每个字符 6 位），当原始数据长度不是 3 的倍数时（比如剩余 1 或 2 个字节），为了凑满 24 位的长度，使用 `=` 作为填充符号进行补齐。
```
any → YW55（无需补位）

an → YW4=（补 1 个 =）

a → YQ==（补 2 个 =）
```

----


#### Base64 编码、解码工具类

```
public class Base64Utils {

    /**
     * ============================================
     * 将字符串进行 Base64 编码
     * --------------------------------------------
     * 传入参数：
     * - String input
     *      - 需要进行编码的字符串
     *
     * 注意事项：
     * - 看似是对字符串直接进行 Base64 编码，其实是对其在内存中经 UTF-8 编码后的的字节数组进行 Base64 编码
     * - getBytes(StandardCharsets.UTF_8) 是获取 UTF-8 形式的字节数组
     * ============================================
     */
    public static String encodeString (String input) {
        if (input == null) return null;
        return Base64.getEncoder().encodeToString(input.getBytes(StandardCharsets.UTF_8));
    }

    /**
     * ============================================
     * 将 Base64 字符串进行解码成字符串
     * --------------------------------------------
     * 传入参数：
     * - String base64Str
     *      - 需要进行解码的 Base64 字符串
     * ============================================
     */
    public static String decodeToString (String base64Str) {
        if (base64Str == null) return null;
        byte[] decodedBytes = Base64.getDecoder().decode(base64Str);
        return new String(decodedBytes, StandardCharsets.UTF_8);
    }

    /**
     * ============================================
     * 将字节数组进行 Base64 编码
     * --------------------------------------------
     * 传入参数：
     * - byte[] date
     *      - 需要进行编码的字节数组
     * ============================================
     */
    public static String encodeBytes(byte[] data) {
        if (data == null) return null;
        return Base64.getEncoder().encodeToString(data);
    }

    /**
     * ============================================
     * 将 Base64 字符串解码为字节数组
     * --------------------------------------------
     * 传入参数：
     * - String base64Str
     *      - 需要进行解码的 Base64 字符串
     * ============================================
     */
    public static byte[] decodeToBytes(String base64Str) {
        if (base64Str == null) return null;
        return Base64.getDecoder().decode(base64Str);
    }

    /**
     * ============================================
     * 将 InputStream 的内容进行 Base64 编码
     * --------------------------------------------
     * 传入参数：
     * - InputStream in
     *      - 需要进行编码的 InputStream
     *
     * 注意事项：
     * - 这串代码适用于小型数据流
     * - 若是大型数据流，需要优化代码，如使用缓存流、分块处理等
     * ============================================
     */
    public static String encodeStream(InputStream in) throws IOException {
        if (in == null) return null;
        byte[] bytes = in.readAllBytes();
        return Base64.getEncoder().encodeToString(bytes);
    }

    /**
     * ============================================
     * 将 Base64 字符串写入 OutputStream
     * --------------------------------------------
     * 传入参数：
     * - String base64Str
     *      - 要进行解码的 Base64 字符串
     * - OutputStream out
     *      - 解码内容要写入的 OotputStream
     * ============================================
     */
    public static void decodeToStream(String base64Str, OutputStream out) throws IOException {
        if (base64Str == null || out == null) return;
        byte[] data = Base64.getDecoder().decode(base64Str);
        out.write(data);
        out.flush();
    }
}
```

---




### Base64URL 编码

#### Base64URL 编码概述

Base64 编码后的字符串中可能会包含字符如 `+`、`/` 和 `=`，而这些字符在 URL、HTTP 头、文件名等场景中可能引起语义歧义或被错误转义。

为了解决 Base64 在 Web 场景中的兼容性问题，诞生了 **Base64URL 编码规范**。Base64URL 是专为 URL 和 Web 应用设计的安全变种，避免了标准 Base64 中可能造成冲突的字符，如：

| 编码字符 | Base64 编码 | Base64URL 编码 |
| ---- | --------- | ------------ |
| `+`  | +         | `-`          |
| `/`  | /         | `_`          |
| `=`  | =         | 通常省略         |

> [!NOTE] 注意事项
> 1. JWT 明确规定必须使用 Base64URL 编码。我们常见的使用方式是将 JWT 放入 HTTP 请求头中，例如：`Authorization: Bearer <token>`。从协议上讲，请求头是支持标准 Base64 的，因此你可能会觉得没必要使用 Base64URL
> 2. 然而，JWT 并不仅仅出现在请求头中，它还常用于 URL 参数（如 OAuth 重定向中的 token）、Cookie、Web Storage 等 Web 场景，在这些环境中，标准 Base64 中的 `+`、`/` 和 `=` 字符会引发各种问题：
> 	1. 例如在 URL 场景下：
> 		1. `+` 在 URL 中可能被解释为空格
> 		2. `/` 是路径分隔符
> 		3. `=` 通常用于参数赋值或被截断、转义
> 3. 因此，为了确保在各种 Web 环境中的兼容性，JWT 规范强制要求使用 Base64URL 编码

-----


#### Base64URL 编码、解码工具类

```
public class Base64UrlUtils {

    /**
     * ============================================
     * 将字符串进行 Base64URL 编码
     * --------------------------------------------
     * 传入参数：
     * - String input
     *      - 需要进行编码的字符串
     * ============================================
     */
    public static String encodeString (String input) {
        if (input == null) return null;
        byte[] data = input.getBytes(StandardCharsets.UTF_8);
        return Base64.getUrlEncoder().encodeToString(data);
    }

    /**
     * ============================================
     * 将 Base64URL 字符串进行解码成字符串
     * --------------------------------------------
     * 传入参数：
     * - String base64UrlStr
     *      - 需要进行解码的 Base64URL 字符串
     * ============================================
     */
    public static String decodeToString (String base64UrlStr) {
        if (base64UrlStr == null) return null;
        byte[] decoded = Base64.getUrlDecoder().decode(base64UrlStr);
        return new String(decoded, StandardCharsets.UTF_8);
    }

    /**
     * ============================================
     * 将字节数组进行 Base64URL 编码
     * --------------------------------------------
     * 传入参数：
     * - byte[] data
     *      - 需要进行编码的字节数组
     * ============================================
     */
    public static String encodeBytes(byte[] data) {
        if (data == null) return null;
        return Base64.getUrlEncoder().encodeToString(data);
    }

    /**
     * ============================================
     * 将 Base64URL 字符串解码为字节数组
     * --------------------------------------------
     * 传入参数：
     * - String base64UrlStr
     *      - 需要进行解码的 Base64URL 字符串
     * ============================================
     */
    public static byte[] decodeToBytes(String base64UrlStr) {
        if (base64UrlStr == null) return null;
        return Base64.getUrlDecoder().decode(base64UrlStr);
    }

    /**
     * ============================================
     * 将 InputStream 的内容进行 Base64URL 编码
     * --------------------------------------------
     * 传入参数：
     * - InputStream in
     *      - 需要进行编码的 InputStream
     * ============================================
     */
    public static String encodeStream(InputStream in) throws IOException {
        if (in == null) return null;
        byte[] bytes = in.readAllBytes();
        return Base64.getUrlEncoder().encodeToString(bytes);
    }

    /**
     * ============================================
     * 将 Base64URL 字符串写入 OutputStream
     * --------------------------------------------
     * 传入参数：
     * - String base64UrlStr
     *      - 要进行解码的 Base64URL 字符串
     * - OutputStream out
     *      - 解码内容要写入的 OutputStream
     * ============================================
     */
    public static void decodeToStream(String base64UrlStr, OutputStream out) throws IOException {
        if (base64UrlStr == null || out == null) return;
        byte[] data = Base64.getUrlDecoder().decode(base64UrlStr);
        out.write(data);
        out.flush();
    }
}
```

-----


### Java 程序 代码 → 保存 → 编译 → 运行 流程

假如你在 Java 程序中，写了这样一段代码：
```
public class Demo {
    public static void main(String[] args) {
        String s = "中";
        byte[] utf8 = s.getBytes(StandardCharsets.UTF_8);
        String base64 = Base64.getEncoder().encodeToString(utf8);
        System.out.println(base64);
    }
}
```

当你保存这段代码时，编辑器会将 `.java` 文件以 UTF-8 的方式将所有内容编码为字节并写入硬盘，例如 `"中"` 会被编码为字节序列 `E4 B8 AD`。

在编译阶段，`javac` 编译器从硬盘读取 `.java` 文件，并按照 UTF-8 解码字节，将 `E4 B8 AD` 还原为字符 `"中"`

当程序运行，执行到 `String s = "中";` 这一行时，由于 JVM 内部的字符串编码是 UTF-16，所以JVM 会将 `"中"` 存储为 UTF-16 编码的字符串，在内存中对应的编码值是 `0x4E2D`（不同的数据类型在 Java 中对应着不同的编码形式）

执行到 `byte[] utf8 = s.getBytes(StandardCharsets.UTF_8);` 时，会将 UTF-16 编码的字符串转换为 UTF-8 编码的字节数组，即`0x4E2D` → `[0xE4, 0xB8, 0xAD]`

当执行 `String base64 = Base64.getEncoder().encodeToString(utf8);` 时，就是将上述 UTF-8 字节数组进行 Base64 编码，最终得到字符串 `"5Lit"`，并在控制台输出。

> [!NOTE] 注意事项
> 1. 我们常说 Java 使用 UTF-8 编码，实际上指的是源代码文件在保存到硬盘时采用 UTF-8 作为编码格式

---
















