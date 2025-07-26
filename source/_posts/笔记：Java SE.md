---
title: 笔记：Java SE
date: 2025-05-07
categories:
  - Java
  - Java 基础
  - Java SE
tags: 
author: 霸天
layout: post
---
### 0、导图：[Map：Java SE](../../maps/Map：JavaSE.xmind)

![](image-20250703093849042.png)
![](image-20250703223607629.png)
还可以这样写


无线循环，while(true)
\


### 函数式接口，lambda 表达式：
### 4. 函数狮子接口

这种一般都是让我们自己写逻辑，而不是那种给我们实现类的
![](image-20250510221406130.png)



lambde 表达式的口诀，复制小括号，

![](image-20250510221434222.png)

返回值![](image-20250510221448648.png)





是的，除了运行 `.jar` 文件，Java 启动命令还有很多形式，**`-XX:PermSize` 和 `-XX:MaxPermSize` 等 JVM 参数都是通用的**，无论你运行的是 `.jar`、类文件 `.class`、还是带主类的模块，都可以用。

---

## ✅ 常见几种启动方式 + JVM 参数用法

---

### 1. **运行 `.jar` 文件**

```bash
java -XX:PermSize=64m -XX:MaxPermSize=128m -jar yourApp.jar
```

> `-jar` 会自动识别 `MANIFEST.MF` 中的 `Main-Class`。

---

### 2. **直接运行 `.class` 文件**

```bash
java -XX:PermSize=64m -XX:MaxPermSize=128m com.example.Main
```

前提：你当前路径或 `CLASSPATH` 里能找到 `com/example/Main.class`。

---

### 3. **运行模块化程序（Java 9+）**

```bash
java --module-path mods -m my.module/com.example.Main
```

可以加上：

```bash
-XX:MetaspaceSize=64m -XX:MaxMetaspaceSize=128m
```

> Java 9 之后模块化程序开始流行，用 `--module-path` 和 `-m`。

---

### 4. **运行 JShell（交互式）**

```bash
jshell -J-XX:MaxPermSize=128m
```

> `-J` 表示将参数传给 JVM，而不是传给 jshell 命令本身。

---

### 5. **使用 `java` 执行临时脚本（Java 11+ 支持单文件运行）**

```bash
java -XX:MaxPermSize=128m Hello.java
```

但要注意：JDK 11 开始就没 PermGen 了，所以这个参数会被忽略。你可以测试下：

```bash
java -XX:+PrintCommandLineFlags -version
```

---

### 6. **运行测试类 / Spring Boot 启动类 / IDEA 调试时也可加这些参数**

你可以在：

- `IDEA` 的 **Run Configuration** 里添加 VM 参数
    
- `Maven`/`Gradle` 的 `java` 命令中加上
    
- `Tomcat` 的 `setenv.sh / setenv.bat` 添加
    

---

## ❗ 注意事项

- **JDK 8+ 不再支持 PermSize / MaxPermSize**，写了也没用；
    
- 如果你写了会提示警告：
    
    ```bash
    Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=64m; support was removed in 8.0
    ```
    

---

需要我帮你写个常用 JVM 启动模板（兼容 JDK 7 / 8 / 11）也可以。你可以告诉我你的 JDK 版本和项目类型（Spring Boot？JavaFX？普通 jar？）我给你定制。





