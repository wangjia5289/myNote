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





关于 Java 中“内部类”的声明方式，这是一个非常重要、经常考的知识点，涵盖了 Java 语法、访问权限、类加载等多个层面。我们来系统、通俗地梳理一下：

---

## ✅ 什么是内部类？

**内部类（Inner Class）**：就是**定义在另一个类内部的类**。

Java 中的内部类可以分为以下几种类型，每种都有独特的用法和限制。

---

## 🧭 内部类的五种分类 + 示例

| 类型            | 声明位置           | 是否 static | 是否能访问外部实例变量                          |
| ------------- | -------------- | --------- | ------------------------------------ |
| 1. 成员内部类      | 类中，方法外         | ❌ 非静态     | ✅ 可以                                 |
| 2. 静态内部类      | 类中，方法外         | ✅ 静态      | ❌ 不可以                                |
| 3. 局部内部类      | 方法/代码块内部       | ❌ 非静态     | ✅ 但只能访问 final 或 effectively final 变量 |
| 4. 匿名内部类      | 临时写在表达式中的类实现   | ❌ 非静态     | ✅ 规则同上                               |
| 5. Lambda 表达式 | 仅限函数式接口的匿名实现方式 | N/A       | ✅ 相似于匿名内部类                           |

---

## 🔎 一、成员内部类（非静态内部类）

```java
public class Outer {
    class Inner {
        void show() {
            System.out.println("我是成员内部类");
        }
    }

    void test() {
        Inner inner = new Inner(); // 需要先有外部类实例
        inner.show();
    }
}
```

- **需要通过外部类实例来创建**：`new Outer().new Inner()`
    
- 可以访问外部类的所有成员，包括 `private`
    

---

## 🔎 二、静态内部类（static 修饰）

```java
public class Outer {
    static class StaticInner {
        void show() {
            System.out.println("我是静态内部类");
        }
    }
}
```

- **和外部类没有实例绑定**，可以直接用：`new Outer.StaticInner()`
    
- 只能访问外部类的 `static` 成员
    

---

## 🔎 三、局部内部类（定义在方法中）

```java
public class Outer {
    void outerMethod() {
        int x = 10; // 必须是 final 或有效 final
        class LocalInner {
            void show() {
                System.out.println("局部内部类，x = " + x);
            }
        }
        LocalInner inner = new LocalInner();
        inner.show();
    }
}
```

- 定义在方法或代码块内
    
- 生命周期受限于方法作用域
    
- 只能访问 **final 或 effectively final** 的局部变量
    

---

## 🔎 四、匿名内部类（实现接口或继承类）

```java
Runnable r = new Runnable() {
    @Override
    public void run() {
        System.out.println("匿名内部类");
    }
};
```

或

```java
Thread t = new Thread() {
    @Override
    public void run() {
        System.out.println("匿名继承 Thread");
    }
};
```

- 没有类名，**只能用一次**
    
- 编译后会生成类似 `Outer$1.class` 的 class 文件
    
- 可以访问外部 final 或 effectively final 的变量
    

---

## 🔎 五、Lambda（函数式接口的简写）

```java
Runnable r = () -> System.out.println("lambda 形式");
```

- 只能用于函数式接口
    
- 更轻量、不生成额外 `.class` 文件
    

---

## 📌 总结一张速查表：

|类型|是否 static|作用范围|可访问外部成员|创建方式|
|---|---|---|---|---|
|成员内部类|否|类中方法外|✅（所有成员）|`outer.new Inner()`|
|静态内部类|是|类中方法外|❌（只能访问 static）|`new Outer.StaticInner()`|
|局部内部类|否|方法内|✅（final 或 e.final）|方法内 new|
|匿名内部类|否|表达式中|✅（final 或 e.final）|写在 `new Interface() {}` 中|
|Lambda 表达式|N/A|表达式中|✅（final 或 e.final）|`() -> {}` 形式，函数式接口才可使用|

---

如你想了解这些类在字节码层面的表现（是否生成独立 `.class`、类名命名规则、class 文件结构差异等），我可以带你用 `javac` + `javap` 分析，非常实用。是否要继续深入？



















很好，我们现在来聊聊 Java 中的「向上转型」和「向下转型」，它们是多态的核心机制之一。为了更清晰，我们分几个小节讲清楚：

---

### ✅ 一、什么是「向上转型」（Upcasting）

**定义**：  
把子类对象赋值给父类类型的引用，这叫向上转型。

```java
Animal a = new Dog();
```

**特点**：

- 是**自动的**，不需要强制类型转换。
    
- 只能访问父类中定义的属性和方法，**即使子类有重写，也只看类型定义的部分**。
    
- 实际运行时，如果调用了被子类重写的方法，还是会执行子类版本（这是多态）。
    

---

### ✅ 二、什么是「向下转型」（Downcasting）

**定义**：  
把父类引用强制转换成子类类型的引用，这叫向下转型。

```java
Animal a = new Dog();  // 向上转型
Dog d = (Dog) a;       // 向下转型
```

**特点**：

- 必须**强制类型转换**。
    
- 如果 `a` 本身不是 `Dog` 类型对象，运行时会抛出 `ClassCastException`。
    
- 向下转型的目的是使用子类中扩展的方法和字段。
    

---

### ✅ 三、例子对比说明：

```java
class Animal {
    void speak() {
        System.out.println("Animal speaks");
    }
}

class Dog extends Animal {
    void speak() {
        System.out.println("Dog barks");
    }

    void sniff() {
        System.out.println("Dog sniffs");
    }
}
```

```java
public class Demo {
    public static void main(String[] args) {
        Animal a = new Dog();  // 向上转型
        a.speak();             // 输出 Dog barks（多态）

        // a.sniff();  // ❌ 报错：Animal 中没有 sniff 方法

        Dog d = (Dog) a;       // 向下转型
        d.sniff();             // ✅ 可以调用 Dog 特有的方法
    }
}
```

---

### ✅ 四、你应该注意的问题：

1. 向上转型 **安全可靠**，用于“抽象编程”。
    
2. 向下转型 **要判断类型**，可以配合 `instanceof`：
    

```java
if (a instanceof Dog) {
    Dog d = (Dog) a;
    d.sniff();
}
```

---

### ✅ 五、与接口的关系：

接口也可以参与转型：

```java
Runnable r = new MyRunnable();  // 向上转型到接口
MyRunnable mr = (MyRunnable) r; // 向下转型回具体类
```

---

需要我再用更具体的场景（比如线程、集合等）来说明转型吗？

















这是个关键的问题。**向上转型不是为了少用，而是为了写出更灵活、可扩展的代码。**

---

## ✅ 一、向上转型到底能干什么？

### 1. **实现多态的基础**

向上转型后，即便引用类型是父类，但调用的是子类的重写方法，实现了“**编译看父类，运行走子类**”的行为。

```java
Animal a = new Dog(); // 向上转型
a.speak();            // 调用的是 Dog 的 speak 方法，而不是 Animal 的
```

👉 这是 Java 多态最核心的机制。

---

### 2. **统一处理不同子类对象**

向上转型可以**让你写出统一的逻辑代码**，例如：一个方法接收父类类型参数，就可以传入任意子类对象。

```java
void makeItSpeak(Animal animal) {
    animal.speak(); // 会自动调用实际子类的 speak 方法
}

makeItSpeak(new Dog());
makeItSpeak(new Cat());
```

👉 你不需要重载 `makeItSpeak(Dog)`、`makeItSpeak(Cat)`，一个父类参数就搞定。

---

### 3. **配合接口编程，提高灵活性**

例如线程的写法：

```java
Runnable task = new MyRunnable(); // 向上转型
Thread t = new Thread(task);      // 接收 Runnable 类型
```

接口 + 向上转型，让你可以随时更换实现类、改动更小，解耦性更强。

---

### 4. **减少依赖具体实现，代码更加抽象通用**

如果一个方法接收的是父类或接口类型，那未来无论你传入哪种实现，它都能兼容，不需要修改方法签名：

```java
List<String> list = new ArrayList<>();
List<String> list2 = new LinkedList<>();
```

你只依赖 `List` 接口，就能任意切换实现，向上转型起到桥梁作用。

---

## ✅ 二、向上转型后有什么限制？

- **无法访问子类扩展的方法或属性**，因为引用类型是父类。
    
    ```java
    Animal a = new Dog();
    a.sniff(); // ❌ 报错，Animal 中没有 sniff 方法
    ```
    
- 如果你想访问子类特有的方法，只能向下转型回来：
    
    ```java
    if (a instanceof Dog) {
        Dog d = (Dog) a;
        d.sniff(); // ✅
    }
    ```
    

---

## ✅ 三、什么时候该用向上转型？

- 想统一处理所有子类的情况（多态）
    
- 想解耦具体实现，只依赖接口/抽象类
    
- 想封装共同行为，简化逻辑处理
    
- 想在集合中存放父类对象，统一遍历处理子类对象（经典用法）
    

---

需要我总结一个「该用 / 不该用向上转型」的决策清单吗？




这是接口的，错的很多奥，第一个就是匿名类实现接口懂吗，还差个向下转型，然后适用匿名类实现接口，就是可以用 lambda 简化写法（不能用lambda简化匿名类继承类），但是Lambda 在编译器层面和匿名类略有不同，不会生成独立 class，而是通过 invokedynamic 延迟绑定，非常高效（有兴趣我可以展开讲）
```
// 1. 直接 new 出接口对象
Runnable runnable = new Runnable() {
    @Override
    public void run() {
        System.out.println("运行中");
    }
};


// 2. 向上转型 new 出接口对象
class MyRunnable implements Runnable {
    @Override
    public void run() {
        System.out.println("运行中");
    }
}

Runnable runnable = new MyRunnable();


// 3. new 出实现类（MyRunnable）类型的对象
class MyRunnable implements Runnable {  
    @Override  
    public void run() {  
        System.out.println("运行中");  
    }  
}  
  
MyRunnable myRunnable = new MyRunnable();
```








无线循环：
while (true){
	if(Thread.currentThread().isInterrupted()){
		break;
	}
}

for(;;){
xxx
xxx
xxx}_