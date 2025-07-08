---
title: 笔记：Java 异常处理
date: 2025-04-11
categories:
  - Java
  - 异常处理
tags: 
author: 霸天
layout: post
---
![](image-20250705191045840.png)

这种该怎么办



![](source/_posts/笔记：Java%20异常处理/image-20250513160838820.png)



在 `mono.onErrorResume(Function<Throwable, Mono<? extends T>> fallback)` 中，`Throwable` 是 Java 中所有错误和异常的父类。它表示一个异常或错误的对象，是所有 `Exception` 和 `Error` 类的基类。

### 详细解释：

1. **`Throwable`**：
    
    - `Throwable` 是 Java 中的所有异常和错误的根类。
        
    - 它分为两大类：
        
        - **`Error`**：通常用于表示 JVM 内部的严重错误，如 `OutOfMemoryError`，这些一般不被捕获和处理。
            
        - **`Exception`**：表示程序中可以捕获和处理的异常。包括受检异常（如 `IOException`）和运行时异常（如 `NullPointerException`）。
            
    - 你可以把 `Throwable` 看作一个包含所有错误和异常的容器。
        
2. **`Throwable` 在 `onErrorResume` 中的作用**：
    
    - `onErrorResume` 是一个错误处理操作符，它会在 `Mono` 或 `Flux` 流中遇到异常时执行。
        
    - `Throwable` 作为参数传递给你提供的 `Function<Throwable, Mono<? extends T>>`，表示发生的错误或异常。
        
    - 你可以在这个函数中处理该异常并返回一个替代的 `Mono`，从而避免流的失败（比如通过返回一个默认值的 `Mono`）。
        

### 代码示例：

```java
Mono<String> mono = Mono.fromCallable(() -> {
    throw new RuntimeException("Oops!");
});

mono.onErrorResume(throwable -> {
    System.out.println("Caught error: " + throwable.getMessage());
    return Mono.just("Recovered from error");
}).subscribe(System.out::println);
```

- 当 `Mono.fromCallable` 抛出 `RuntimeException` 时，`onErrorResume` 会捕获该异常（即 `throwable`），并执行提供的函数，返回一个新的 `Mono`，在该例中返回的是 `Mono.just("Recovered from error")`，从而避免了流的失败。
    

### 总结：

- **`Throwable`** 是一个非常广泛的基类，涵盖了所有异常和错误类型。
    
- 在 `onErrorResume` 中，**`Throwable`** 表示发生的错误或异常类型，你可以根据它来处理错误并返回一个替代的 `Mono`。
![](source/_posts/笔记：Java%20异常/image-20250411224927524.png)


![](source/_posts/笔记：Java%20异常/image-20250411224723236.png)








我来回答你的问题，分步骤解释清楚。

### 1. 抛出一个 RuntimeException，异常消息为 "Oops!"
在 Java 中，抛出一个 `RuntimeException` 非常简单。你可以使用 `throw` 关键字，并创建一个 `RuntimeException` 实例，指定异常消息为 "Oops!"。代码示例如下：

```java
throw new RuntimeException("Oops!");
```

这段代码会在运行时抛出一个 `RuntimeException`，并带有消息 "Oops!"。由于 `RuntimeException` 是非受检异常（unchecked exception），你无需在方法声明中显式声明它，也无需用 `try-catch` 强制捕获。

---

### 2. 常见的异常有哪些？
Java 中有许多常见的异常，有些是开发者在日常编程中经常遇到的。下面列举一些常见的异常及其含义：

#### 常见的非受检异常（RuntimeException 的子类）
- **NullPointerException**：尝试对 `null` 对象进行操作时抛出，例如调用 `null` 对象的某个方法。
- **ArrayIndexOutOfBoundsException**：访问数组时，索引超出数组范围（例如访问 `arr[5]` 而数组长度只有 3）。
- **IllegalArgumentException**：传递给方法的参数不合法，例如传入负数给需要正数的方法。
- **ClassCastException**：尝试将对象强制转换为不兼容的类型，例如将 `String` 转为 `Integer`。
- **ArithmeticException**：算术运算出错，例如除以零（`1 / 0`）。

#### 常见的受检异常（需要显式处理）
- **IOException**：输入输出操作出错，例如文件不存在、读取文件失败或权限不足。
- **SQLException**：数据库操作出错，例如连接失败或 SQL 语法错误。

这些异常只是冰山一角，但它们在开发中出现频率较高，值得特别关注。

---

### 3. 这些异常都是 Java 自带的吗？还是框架有一些异常？
- **Java 自带的异常**：上述提到的 `NullPointerException`、`ArrayIndexOutOfBoundsException`、`IllegalArgumentException`、`ClassCastException`、`ArithmeticException`、`IOException` 和 `SQLException` 都是 Java 标准库自带的，定义在 `java.lang`、`java.io` 或 `java.sql` 等包中。这些异常由 Java 核心 API 提供，适用于通用的编程场景。
- **框架特定的异常**：除了 Java 自带的异常，许多框架会定义自己的异常类，以适应特定需求。例如：
  - **Spring 框架**：提供了 `DataAccessException`，用于封装数据库操作中的异常。
  - **Hibernate**：有 `HibernateException`，用于处理 ORM（对象关系映射）相关的错误。
这些框架异常通常是对 Java 标准异常的扩展或封装，方便开发者在特定上下文中使用。

因此，异常来源可以分为两类：Java 自带的通用异常和框架定义的特定异常。

---

### 4. 异常那么多，我怎么记啊？
面对种类繁多的异常，记住所有异常确实不现实，也没必要。以下是一些实用的建议，帮助你掌握和记忆异常：

#### 理解异常的分类和层次
Java 的异常体系是一个树状结构：
- **Throwable**：所有异常和错误的根类。
  - **Error**：表示严重问题（如 `OutOfMemoryError`），通常不由程序处理。
  - **Exception**：表示可处理的异常。
    - **RuntimeException**：非受检异常，运行时抛出。
    - **其他 Exception**：受检异常，编译时要求处理。

理解这个层次结构，能帮助你快速定位异常的类型和处理方式。

#### 重点记忆常见异常
优先记住开发中高频出现的异常，例如：
1. `NullPointerException`（空指针）
2. `ArrayIndexOutOfBoundsException`（数组越界）
3. `IllegalArgumentException`（非法参数）
4. `IOException`（输入输出错误）
5. `SQLException`（数据库错误）

这些异常涵盖了大部分常见错误场景，熟悉它们就能应对大多数情况。

#### 根据场景记忆
异常通常与特定操作相关联。例如：
- 操作数组时，可能遇到 `ArrayIndexOutOfBoundsException`。
- 访问数据库时，可能遇到 `SQLException`。
- 处理文件时，可能遇到 `IOException`。

在实际开发中，结合上下文理解异常的触发条件，记忆会更自然。

#### 查阅文档和实践
- **Java 自带异常**：查阅 Java API 文档（如 `java.lang.RuntimeException` 的子类列表）。
- **框架异常**：使用框架时，阅读其官方文档，了解特定异常及其处理建议。
- **实践**：通过编写代码、调试和处理异常，积累经验，逐渐熟悉。

#### 不必强行记住所有异常
异常种类繁多，且新框架可能引入新异常。关键是掌握异常处理的基本原则（捕获、抛出、日志记录等），遇到不熟悉的异常时，知道如何查资料和解决问题。

---

### 总结
- 抛出 `RuntimeException` 的代码是：`throw new RuntimeException("Oops!");`。
- 常见异常包括 `NullPointerException`、`ArrayIndexOutOfBoundsException`、`IOException` 等。
- 异常既有 Java 自带的，也有框架定义的。
- 记忆异常的策略：理解分类、重点记忆常见异常、结合场景、查阅文档并通过实践积累经验。

希望这些解答对你有帮助！如果还有疑问，欢迎继续提问。



### 如何找异常

以下面这段代码为例，当我们使用第三方库、调用别人提供的方法时，经常会遇到一个问题：我怎么知道它可能抛出哪些异常？而我又该怎么处理这些异常？
![](source/_posts/笔记：Java%20异常处理/image-20250513192340373.png)


其实非常简单，我们只需要把鼠标悬停在方法名上，IDEA 会自动提示这个方法可能抛出的异常类型：
![](source/_posts/笔记：Java%20异常处理/image-20250513192712609.png)


如果你想进一步确认，还可以按住 `Ctrl` 并点击方法名，进入该方法的源码，查看它**显式抛出了哪些异常**：
![](source/_posts/笔记：Java%20异常处理/image-20250513192840306.png)


当你搞清楚了可能出现的异常，就可以进行异常处理。当然，也不用自己去写一堆 try-catch：只要选中方法，按下 `Ctrl + Alt + T`，IDEA 会自动帮你生成完整的 try-catch 模板：

要一整行都选中才可以
![](source/_posts/笔记：Java%20异常处理/image-20250513193033722.png)


然后我们的代码就可以像这样，每种异常你都可以单独处理，灵活掌控：
![](source/_posts/笔记：Java%20异常处理/image-20250513193745547.png)

不过，如果你暂时不打算对每个异常做精细化区分，也可以只 catch 顶层的 `Exception`，然后简单打印下异常信息作为兜底处理：
```
@GetMapping("/bucketExists")
public Boolean bucketExists(@RequestParam String bucketName) {
    try {
        return minioClient.bucketExists(
            BucketExistsArgs.builder().bucket(bucketName).build()
        );
    } catch (Exception e) {
    
        // 通知、告警
        System.out.println("查询 bucket 时出错：" + e.getMessage());

        // 返回一个安全默认值
        return false;
    }
}
```

---







我调用别人的方法，别人的方法把异常都抛出来了，我知道应该去处理哪些异常，可是我肯定不能一直调用别人的方法，自己不写方法吧，那我自己封装方法，我怎么知道可能会出现哪些异常、怎么抛异常、怎么知道我抛出的异常是不是和系统中已存在的异常交叠











我们可以就是把这些异常都以一个异常跑出来：
```
try {  
    boolean bucketExists = minioClient.bucketExists(BucketExistsArgs.builder().bucket(bucketName).build());  
} catch (ErrorResponseException e) {  
    throw new RuntimeException(e);  
} catch (InsufficientDataException e) {  
    throw new RuntimeException(e);  
} catch (InternalException e) {  
    throw new RuntimeException(e);  
} catch (InvalidKeyException e) {  
    throw new RuntimeException(e);  
} catch (InvalidResponseException e) {  
    throw new RuntimeException(e);  
} catch (IOException e) {  
    throw new RuntimeException(e);  
} catch (NoSuchAlgorithmException e) {  
    throw new RuntimeException(e);  
} catch (ServerException e) {  
    throw new RuntimeException(e);  
} catch (XmlParserException e) {  
    throw new RuntimeException(e);  
}
```
然后再去处理这一个异常
![](image-20250708180126262.png)

或者直接每个都去处理

![](image-20250708180155666.png)














































