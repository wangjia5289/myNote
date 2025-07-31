---
title: 笔记：JVM
date: 2025-05-29
categories:
  - Java
  - Java 基础
  - Java Virtual Machine（JVM）
tags: 
author: 霸天
layout: post
---
# JVM 基础

## 学习 JVM 的核心目标

1. 防止出现 OOM
2. 出现 OOM 后，解决 OOM
3. 减少 Full GC 出现的平吕

---


## 1. 类编译过程

---


## 2. 类加载过程

### 2.1. 类加载过程一览图

![](image-20250728104904949.png)

---


### 2.2. 类加载概述

**类加载**是指 JVM 将类的 `.class` 字节码文件中的二进制数据读取到内存，为该类在 JVM 内部构建一个对应的 `java.lang.Class` 实例，并依次完成对该类的结构信息的加载、验证、准备、解析和初始化，确保类可以被程序在运行时安全、合法地使用。

---


### 2.3. 类加载的时机

JVM（Java 虚拟机）在运行时并不会一次性加载所有的类，而是采取**按需加载**的策略，这被称为**动态加载**。类加载的时机主要分为两种：**启动时加载**和**运行时加载**。
1. 启动时加载
	1. JVM 启动时，会加载启动类，然后执行其 main 方法
		1. 不会在堆内存中创建启动类的实例对象
		2. 因为 `main` 方法是一个静态方法，它不需要一个类的实例就可以被调用
	2. 同时，JVM 会加载运行程序所需的核心类，例如 `java.lang.Object`、`java.lang.String` 以及其他在启动过程中直接或间接被引用的类。
2. 运行时加载
	1. 这是类加载最常见的场景，常见触发情况有：
	2. 创建类的实例时：
		1. 当你使用 `new` 关键字创建一个类的对象时，如果这个类还没有被加载，JVM 会立即加载它。
		2. 例如：`MyClass obj = new MyClass();` // 第一次使用 MyClass 时，会被加载
	3. 访问类的静态成员时：
		1. 当你访问一个类的静态变量（`static field`）或者调用一个类的静态方法（`static method`）时，如果该类还没有被加载，JVM 会加载它
		2. 例如：`MyClass.staticMethod();`
	4. 使用反射时
		1. 当你通过 `java.lang.reflect` 包中的方法，例如 `Class.forName("com.example.MyClass")` 来获取一个类的 `Class` 对象时，会触发该类的加载
		2. 例如：`Class<?> clazz = Class.forName("com.example.MyClass");`
	5. 子类被加载时，父类也会被加载
		1. 当一个类的子类被初始化时，它的父类如果尚未加载，则会首先被加载
		2. 这是因为子类继承了父类的成员，需要确保父类已经就绪
	6. 使用 ClassLoader.loadClass()
		1. 根据给定的类全名（例如 `java.lang.String` 或 `com.example.MyClass`）来加载对应的类
		2. 这是更底层、更灵活的加载方式，通常在自定义类加载器时使用

---

#### 2.3.1. JVM 类加载的时机

JVM（Java Virtual Machine）中类的加载是指将类的 `.class` 文件中的二进制数据读取到内存中，并为之创建对应的 `java.lang.Class` 对象的过程。这个过程并非在 JVM 启动时一次性完成所有类的加载，而是根据需要动态地进行。类加载的时机主要分为以下几个阶段：

#### 2.3.2. 加载 (Loading)

**“加载”是类加载过程的第一个阶段。** 在这个阶段，JVM 完成以下三件事：

- **通过类的完全限定名获取定义此类的二进制字节流。** 这一步可以通过多种方式实现，例如从本地文件系统（最常见）、网络、JAR 包、运行时计算生成（如动态代理）等。
    
- **将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构。** 方法区是存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据的区域。
    
- **在内存中生成一个代表这个类的 `java.lang.Class` 对象。** 这个对象作为方法区这个类的各种数据的访问入口。
    

**何时开始加载？**

加载阶段通常在以下情况发生：

- 当遇到 `new`、`getstatic`、`putstatic` 或 `invokestatic` 这四条字节码指令时，如果类没有进行过初始化，则需要先触发其初始化。在这之前，自然需要先完成加载和验证、准备阶段。
    
- 使用 `java.lang.reflect` 包的方法对类进行反射调用时。
    
- 当初始化一个类时，如果其父类还没有被初始化，则需要先初始化其父类。
    
- 当虚拟机启动时，用户需要指定一个要执行的主类（包含 `main()` 方法的那个类），虚拟机会先加载并初始化这个主类。
    
- 当使用 `JDK 7` 的动态语言支持时，如果一个 `java.lang.invoke.MethodHandle` 实例最后的解析结果是 `REF_getStatic`、`REF_putStatic`、`REF_invokeStatic`、`REF_newInvokeSpecial` 四种类型的方法句柄，并且这个方法句柄对应的类没有进行过初始化，则需要先触发其初始化。

        

**何时触发初始化？**

JVM 规范严格规定了有且只有以下五种情况（在 JDK 7 中增加了第六种情况）必须立即对类进行初始化（而加载、验证、准备自然要在此之前完成）：

1. 遇到 `new`、`getstatic`、`putstatic` 或 `invokestatic` 这四条字节码指令时，如果类没有进行过初始化，则需要先触发其初始化。这四条指令分别对应：
    
    - 使用 `new` 关键字实例化对象。
        
    - 读取一个类的静态字段（被 `final` 修饰，已在编译期把结果放入常量池的静态字段除外）。
        
    - 设置一个类的静态字段。
        
    - 调用一个类的静态方法。
        
2. 使用 `java.lang.reflect` 包的方法对类进行反射调用时，如果类没有进行过初始化，则需要先触发其初始化。
    
3. 当初始化一个类时，如果其父类还没有被初始化，则需要先初始化其父类。
    
4. 当虚拟机启动时，用户需要指定一个要执行的主类（包含 `main()` 方法的那个类），虚拟机会先加载并初始化这个主类。
    
5. 当使用 `JDK 7` 的动态语言支持时，如果一个 `java.lang.invoke.MethodHandle` 实例最后的解析结果是 `REF_getStatic`、`REF_putStatic`、`REF_invokeStatic`、`REF_newInvokeSpecial` 四种类型的方法句柄，并且这个方法句柄对应的类没有进行过初始化，则需要先触发其初始化。
    
6. 当一个接口中定义了 `JDK 8` 新加入的默认方法（`default` 方法）时，如果有这个接口的实现类发生了初始化，那该接口要在其之前被初始化。
    

这些情况被称为 **“主动引用”**。除此之外的所有引用类的方式，都不会触发类的初始化，这被称为 **“被动引用”**。

是初始化还是类加载？？？？？？///

---


### 2.4. 加载阶段

#### 2.4.1. 加载阶段概述

加载阶段是类加载过程的第一个阶段，JVM 会在该阶段完成以下三件事：
1. 通过类的完全限定名获取定义此类的二进制字节流
	1. 这一步可以通过多种方式实现，常见有：
		1. 从本地文件系统直接加载
			1. 根据类名查找磁盘上的 `.class` 文件
			2. 例如：org.example.test.MyClass → org/example/test/MyClass.class
			3. 场景：本地 Java 应用
		2. 通过网络加载
			1. 类的字节码通过 HTTP 或 FTP 等协议远程获取
			2. 场景：分布式类加载
		3. 运行时动态生成字节码
			1. 程序运行时动态创建字节数组作为类定义内容
			2. 场景：
				1. 动态代理
				2. 字节码增强技术
		4. 从其他文件类型转换生成
			1. `.jsp` 被转换为 `.java`，再编译成 `.class` 并加载
			2. 场景：
				1. JSP
2. 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构
3. 在内存中生成一个代表这个类的 `java.lang.Class` 对象
	1. 这个对象作为方法区这个类的各种数据的访问入口

![](image-20250728230311438.png)

> [!NOTE] 注意事项
> 1. JVM 验证的并不是 `.class` 字节码文件本身，而是将其加载并解析后生成的**结构化类元信息**，因此验证过程发生在加载过程之后
> 2. 一个类被加载后，既会在方法区（元空间）中构建对应的运行时数据结构，同时还会在内存中生成一个 `java.lang.Class` 实例，该对象内部包含指向元信息的引用，我们可以通过其提供的方法访问和获取该类的详细信息。

---


#### 2.4.2. 类加载器

---


### 2.5. 验证阶段

验证阶段是链接阶段中的第一个阶段，其主要包含：
1. 文件格式验证
	1. 检查字节码是否符合 Class 文件格式的规范，是否以魔数 `0xCAFEBABE` 开头。
	2. 确保主版本号和副版本号在 JVM 支持的范围内。
	3. 检查文件结构，如常量池、字段表、方法表等是否符合规范。
2. 元数据验证
	1. 这个类是否有父类
		1. 除了 `java.lang.Object`，所有类都应该有父类
		2. 注意接口 `interface` 也没有父类
	2. 这个类的父类是否是 `final` 的（`final` 类不允许被继承）
	3. 这个类是否实现了它所声明实现的所有接口的方法
	4. 类中的字段和方法是否与父类或接口的规范保持一致
	5. 确保类、字段和方法访问权限的正确性（例如，不能覆盖一个 `final` 方法）
3. 字节码验证
	1. 控制流分析
		1. 确保方法体内的控制流是合法的，例如，不会出现跳转到方法体外的指令，或者指令序列是连贯的
	2. 类型检查
		1. 确保在操作数栈上的操作是合法的，例如，不会将一个整数类型的值赋值给一个引用类型的变量，或者对一个非对象类型进行方法调用
	3. 栈帧验证
		1. 验证在方法执行过程中，栈帧（Stack Frame）的操作是正确的，例如，操作数栈的深度不会溢出，局部变量表的索引是合法的
	4. 方法调用验证
		1. 确保方法调用时参数的数量和类型都与目标方法的签名匹配
	5. 异常处理验证
		1. 确保异常处理器（exception handler）是合法的，并且能够正确处理抛出的异常
4. 符号引用验证
	1. 符号引用中描述的全限定名是否能找到对应的类、字段或方法。
	2. 符号引用中描述的类、字段或方法是否具有访问权限（例如，是否尝试访问一个 `private` 成员）
	3. 引用的字段或方法是否与被引用的类兼容（例如，对一个接口调用其未定义的方法）

> [!NOTE] 注意事项
> 1. 实际上，文件格式验证发生在加载阶段之前，而符号引用验证则是在解析阶段进行的。

---


### 2.6. 准备阶段

#### 2.6.1. 准备阶段概述

准备阶段是链接阶段的第二个子阶段，在这一阶段，JVM 的主要任务是：
1. 为类的静态变量（static 变量）分配内存
	1. 内存空间会被分配在方法区中
2. 并为类的静态变量赋予默认初始值
	1. 整型（byte、short、int、long）
		1. 0
	2. 浮点型（float、double）
		1. 0.0
	3. char
		1. \u0000（即 0）
	4. boolean
		1. false
	5. 引用类型
		1. null

例如这个代码：
```
public class Test {

    // 1. 静态基本数据类型变量
    public static int staticBasicTypeValue = 50000;

    // 2. 静态字符串类型变量 1
    public static String staticStringTypeValue1 = "abc";

    // 3. 静态字符串类型变量 2
    public static String staticStringTypeValue2 = new String("def");

    // 4. 静态字符串类型变量 3
    public static String staticStringTypeValue3 = staticStringTypeValue2.intern();

    // 5. 静态引用类型变量
    public static Demo staticReferenceTypeValue = new Demo();


    public static void main(String[] args) {
        Test test = new Test();

        System.out.println("1. staticBasicTypeValue = " + Test.staticBasicTypeValue); // 50000
        System.out.println("2. staticStringTypeValue1 = " + Test.staticStringTypeValue1); // abc
        System.out.println("3. staticStringTypeValue2 = " + Test.staticStringTypeValue2); // def
        System.out.println("4. staticStringTypeValue3 = " + Test.staticStringTypeValue3); // def
        System.out.println("5. staticReferenceTypeValue = " + Test.staticReferenceTypeValue); // org.example.test.Demo@58372a00
    }
}
```

在准备阶段时为：
![](image-20250729130341869.png)

> [!NOTE] 注意事项
> 1. 准备阶段的作用是为静态变量提供一个 “安全” 的初始状态，防止出现野指针或非法内存访问
> 2. 对于静态常量（static 和 final 同时修饰），并且是基本数据类型和字符串类型的静态常量，其值在编译期就已确定，因此在准备阶段就会被直接初始化为其编译期常量值，而不是默认值
```
// 静态基本数据类型常量
public static final int staticBasicTypeValue = 50000;


// 静态字符串类型常量 
public static final String staticStringTypeValue1 = "abc";
```

---


#### 2.6.2. 静态常量处理

静态常量（全局常量）与静态变量的区别在于：对于基本数据类型和字符串类型的静态常量（如 1、2），它们是**编译期常量**，其值在编译阶段就已确定，因此在 JVM 的**准备阶段**就会被直接初始化为最终值，而不是默认值

而对于引用类型的静态变量（如 3、4、5），它们在准备阶段会先被赋予默认值，随后在初始化阶段通过执行 `clinit` 方法完成真正的赋值。

以如下代码为例：
```
public class Test {

    // 1. 静态基本数据类型常量
    public static final int staticFinalBasicTypeValue = 50000;

    // 2. 静态字符串类型常量 1
    public static final String staticFinalStringTypeValue1 = "abc";

    // 3. 静态字符串类型常量 2
    public static final String staticFinalStringTypeValue2 = new String("def");

    // 4. 静态字符串类型常量 3
    public static final String staticFinalStringTypeValue3 = staticFinalStringTypeValue2.intern();

    // 5. 静态引用类型常量
    public static final Demo staticFinalReferenceTypeValue = new Demo();

    public static void main(String[] args) {
        Test test = new Test();

        System.out.println("1. staticFinalBasicTypeValue = " + Test.staticFinalBasicTypeValue); // 50000
        System.out.println("2. staticFinalStringTypeValue1 = " + Test.staticFinalStringTypeValue1); // abc
        System.out.println("3. staticFinalStringTypeValue2 = " + Test.staticFinalStringTypeValue2); // def
        System.out.println("4. staticFinalStringTypeValue3 = " + Test.staticFinalStringTypeValue3); // def
        System.out.println("5. staticFinalReferenceTypeValue = " + Test.staticFinalReferenceTypeValue); // org.example.test.Demo@58372a00
    }
}
```

其 `clinit` 字节码是：
```
 0 new #45 <java/lang/String>
 3 dup
 4 ldc #47 <def>
 6 invokespecial #49 <java/lang/String.<init> : (Ljava/lang/String;)V>
 9 putstatic #26 <org/example/test/Test.staticFinalStringTypeValue2 : Ljava/lang/String;>
12 getstatic #26 <org/example/test/Test.staticFinalStringTypeValue2 : Ljava/lang/String;>
15 invokevirtual #51 <java/lang/String.intern : ()Ljava/lang/String;>
18 putstatic #34 <org/example/test/Test.staticFinalStringTypeValue3 : Ljava/lang/String;>
21 new #55 <org/example/test/Demo>
24 dup
25 invokespecial #57 <org/example/test/Demo.<init> : ()V>
28 putstatic #38 <org/example/test/Test.staticFinalReferenceTypeValue : Lorg/example/test/Demo;>
31 return
```

其 `init` 字节码是：
```
0 aload_0
1 invokespecial #1 <java/lang/Object.<init> : ()V>
4 return
```

其 `main` 字节码是：
```
 0 new #7 <org/example/test/Test>
 3 dup
 4 invokespecial #9 <org/example/test/Test.<init> : ()V>
 7 astore_1
 8 getstatic #10 <java/lang/System.out : Ljava/io/PrintStream;>
11 ldc #16 <1. staticFinalBasicTypeValue = 50000>
13 invokevirtual #18 <java/io/PrintStream.println : (Ljava/lang/String;)V>
16 getstatic #10 <java/lang/System.out : Ljava/io/PrintStream;>
19 ldc #24 <2. staticFinalStringTypeValue1 = abc>
21 invokevirtual #18 <java/io/PrintStream.println : (Ljava/lang/String;)V>
24 getstatic #10 <java/lang/System.out : Ljava/io/PrintStream;>
27 getstatic #26 <org/example/test/Test.staticFinalStringTypeValue2 : Ljava/lang/String;>
30 invokedynamic #30 <makeConcatWithConstants, BootstrapMethods #0>
35 invokevirtual #18 <java/io/PrintStream.println : (Ljava/lang/String;)V>
38 getstatic #10 <java/lang/System.out : Ljava/io/PrintStream;>
41 getstatic #34 <org/example/test/Test.staticFinalStringTypeValue3 : Ljava/lang/String;>
44 invokedynamic #37 <makeConcatWithConstants, BootstrapMethods #1>
49 invokevirtual #18 <java/io/PrintStream.println : (Ljava/lang/String;)V>
52 getstatic #10 <java/lang/System.out : Ljava/io/PrintStream;>
55 getstatic #38 <org/example/test/Test.staticFinalReferenceTypeValue : Lorg/example/test/Demo;>
58 invokedynamic #42 <makeConcatWithConstants, BootstrapMethods #2>
63 invokevirtual #18 <java/io/PrintStream.println : (Ljava/lang/String;)V>
66 return
```

在该阶段的静态常量为：
![](image-20250729191420526.png)

其执行结果为：
![](image-20250729191932730.png)

---


### 2.7. 解析阶段

#### 2.7.1. 解析阶段概述

解析阶段的主要任务是将运行时常量池中的符号引用（符号形式的文本信息）解析为直接引用（可定位到内存的真实地址）。通常，这一阶段主要针对以下四类常量池元素进行解析：
1. CONSTANT_Class_info
	1. 类或接口符号引用
2. CONSTANT_Fieldref_info
	1. 字段的符号引用
3. CONSTANT_Methodref_info
	1. 类中方法的符号引用
4. CONSTANT_InterfaceMethodref_info
	1. 接口中方法的符号引用

而在 Java 7 引入 invokedynamic 指令之后，常量池中增加了三种与动态调用相关的符号引用：
1. CONSTANT_MethodType_info
	1. 方法类型的符号引用
2. CONSTANT_MethodHandle_info
	1. 方法句柄的符号引用
3. CONSTANT_InvokeDynamic_info
	1. 动态调用点的符号引用

下图是在**假设所有符号引用都已完成解析**（未加载的类都已经发生了类加载，包括加载、链接、初始化）的前提下绘制的示意图，**这在实际执行过程中是逐步完成的**，而不是在解析阶段一次性完成的，**这里只是为了方便理解而统一展示**。

另外，图中**只列出了一些具有代表性的常量池元素**，实际上运行时常量池中还有很多常量没有在图中展示出来。比如 `#1` 实际上引用了 `#2` 和 `#3`，但出于图示空间限制并未展开。

这些常量池元素在经过解析之后，都会**指向具体的内存地址（方法区内）**；例外的是像 `Integer` 这样的基本类型常量，它们不是符号引用类型，而是**字面量值**，不涉及地址指向，因此仍以值的形式存在；而像 `String` 这样的类型，是由 JVM 通过特定机制在字符串常量池（String Table）中查找对应的地址，我们可以简单地理解为：`String` 类型的变量指向的是字符串常量池中的条目。
![](image-20250729162455405.png)

> [!NOTE] 注意事项
> 1. 不同 JVM 对解析阶段的处理方式可能存在差异：
> 	1. 一些 JVM 实现会在解析阶段，遍历运行时常量池中的所有符号引用，并尝试将其立即解析为直接引用；
> 	2. 也有一些 JVM 实现会在解析阶段，选择只解析其中**关键或高频使用的符号引用**，比如 java.lang.Object 这类核心类;
> 	3. HotSpot JVM 采用的是延迟解析策略
> 		1. 在解析阶段，HotSpot 会遍历并识别出所有类和接口的符号引用（CONSTANT_Class_info）
> 		2. 对于像 java.lang.Object 这样的基础核心类，JVM 会尝试解析它
> 			1. 如果发现尚未加载，就会立即进行加载（加载、链接、初始化），然后将其转化为直接引用
> 		3. 而大多数的 类和接口的符号引用（包括其父类在内的 CONSTANT_Class_info）、字段（CONSTANT_Fieldref_info）、类中方法（CONSTANT_Methodref_info）、接口方法（CONSTANT_InterfaceMethodref_info）的符号引用，通常采用**按需解析**，即在首次使用时才触发解析
> 		4. 比如在初始化阶段执行 clinit 方法时，或在运行 init、main 方法，甚至其他方法的字节码时，如果某个符号引用还未被解析，就会在那一刻解析
> 		5. 总结来说：符号引用将在 **解析阶段** **或者** **程序运行时** 被解析为直接引用。
> 2. 除了上述的符号引用类型，常量池中还包含一些字面量，它们不需要经过解析过程，因为它们本身就是具体的值，可以直接使用：
> 	1. CONSTANT_Utf8_info
> 		1. UTF-8 编码的字符串
> 	2. CONSTANT_Integer_info
> 		1. 整数常量
> 	3. CONSTANT_Float_info
> 		1. 浮点数常量
> 	4. CONSTANT_Long_info
> 		1. 长整数常量
> 	5. CONSTANT_Double_info
> 		1. 双精度浮点数常量
> 	6. CONSTANT_String_info
> 		1. 引用一个 CONSTANT_Utf8_info 项，用于表示字符串的文本内容；
> 		2. 最终在运行时会解析为一个 java.lang.String 对象的直接引用；
> 		3. 它更侧重于对字符串字面量的处理，而不属于传统意义上的符号引用解析（如类、字段、方法等）
> 	7. CONSTANT_NameAndType_info
> 		1. 同样引用两个 CONSTANT_Utf8_info 项，分别表示名称和描述符（Descriptor）；
> 		2. 用于将一个 字段或方法的名称 和它的 描述符（Descriptor） 组合在一起
> 		3. 主要被 CONSTANT_Fieldref_info、CONSTANT_Methodref_info 和 CONSTANT_InterfaceMethodref_info 所引用，用于完成字段和方法的精确标识
> 3. boolean、byte、char、short 没有各自对应的常量池元素
> 	1. 我们知道，在操作数栈和局部变量表中，这几种类型在 JVM 中统一按 int 类型进行存储和处理；
> 	2. 对于 boolean，数值 1 表示 true，0 表示 false；
> 	3. 对于 char，例如：char staticChar = 'a'，Java 中的 char 是一个 16 位无符号整数，存储的是字符的 Unicode 编码，字符 'a' 的 Unicode 值是 97，所以它实际存储的就是数值 97；
> 	4. 因此这些值在编译期间就已确定，编译器会直接使用 iconst、bipush、sipush 等指令将它们压入操作数栈，无需通过 ldc 从运行时常量池中加载，从而提升了运行效率。

---


#### 2.7.2. 深入理解 CONSTANT_Utf8_info

`CONSTANT_Utf8_info` 用于存储 `.class` 文件中**以 UTF-8 编码表示的各种文本信息**，例如类名、字段名、方法名、描述符、字符串字面量等。

例如，在 `Test` 类的常量池中，就可能存在一个 `CONSTANT_Class_info` 项，它会进一步引用一个 `CONSTANT_Utf8_info`，例如：
![](image-20250729104232958.png)

`cp_info #4` 项内部指向一个 `CONSTANT_Utf8_info`，而该 `Utf8_info` 项中存储的内容就是该类的全限定名字符串，例如：
![](image-20250729104338743.png)



这个引用的字符串（如 `"java/lang/Object"`）本质上只是提供了类的全限定名信息，用于告诉 JVM “应该去哪里找这个类”。例如：JVM 可能会根据这个类名去查找对应的 `java/lang/Object.class` 文件，如果该类尚未被加载，就会触发对它的加载流程（加载 → 验证 → 准备 → 解析 → 初始化）。

而运行时常量池中的 `#2 CONSTANT_Class_info` 项，最终会转换为指向方法区中 `java.lang.Object` 的类元数据的直接引用。所以你可以这样理解：这个字符串信息只是 “引导解析” 的依据，真正完成解析之后，引用该指向哪里还是指向哪里。也就是说，它的作用只是 “告诉你去哪里找”，而不是 “最终指向哪里”；

需要注意的是，在准备阶段，JVM 会在方法区中创建该类的元数据结构，这个过程中实际上也会读取 `CONSTANT_Utf8_info` 中的相关文本信息来初始化类的结构描述。

---


#### 2.7.3. 深入理解 CONSTANT_String_info


运行时常量池中的 `CONSTANT_String_info` 指向的是一个 `CONSTANT_Utf8_info` 项，例如我们在代码中写了：
```
public static String staticStringTypeValue1 = "abc";
```

这时，在运行时常量池中就会存在一个 `CONSTANT_String_info` 常量项：
![](image-20250729105950541.png)

它指向一个 `CONSTANT_Utf8_info`：
![](image-20250729110014360.png)


当我们调用某个方法（如 `clinit`）并执行对应字节码时：
```
ldc #57 <def>
```

JVM 会通过 `CONSTANT_String_info` 的索引，定位到对应的 `CONSTANT_Utf8_info`，提取其中的 UTF-8 字节序列（例如 `'a'`、`'b'`、`'c'` 的 UTF-8 编码）

接着，JVM 内部的字符串常量池机制会通过 `StringTable` 进行处理：
1. 如果 `StringTable` 中已经存在指向 `"abc"` 的 `String` 对象的引用，JVM 就直接返回该对象的堆内存地址；
2. 如果不存在，JVM 会在 堆内存中创建一个新的 `java.lang.String` 对象，其内部值为 `"abc"`，然后将该对象的引用（即内存地址）缓存进 `StringTable`。

无论是复用已有对象还是新建对象，JVM 最终都会获得该字符串对象的堆内存地址
![](image-20250729111925100.png)

> [!NOTE] 注意事项
> 1. **`StringTable`** 是 JVM 堆区中维护的一个哈希表，用于存储的是对堆中 `String` 对象的引用（可能是弱引用或强引用）。
> 	1. 它不是单独划分在哪个内存区，如 Eden 区、Survivor 区，而是可以理解为 JVM 在堆上管理的一个内部结构。
> 2. intern() 方法的作用是：
> 	1. 如果 `StringTable` 中已经存在内容相同的 `String` 对象引用，就直接返回该引用
> 	2. 如果没有，则将当前 `String` 对象的引用加入 `StringTable` 中，并返回它的引用
> 3. 为什么执行 System.out.println(instanceStringTypeValue) 时，打印出的内容是字符串，而按理说应该像 instanceReferenceTypeValue 那样打印引用地址才对？
> 	1. 这是因为 System.out.println() 实际上是先调用对象的 toString() 方法
> 	2. 而 String 类重写了 toString()，直接返回自身（即 return this），所以直接打印字符串内容。
> 	3. 其他对象如果没有重写 toString()，则打印的是类似 @1a2b3c4d 这样的内存地址形式
```
// 源码
public String instanceStringTypeValue1 = "abc";
System.out.println("2. instanceStringTypeValue1 = " + test.instanceStringTypeValue1);


// 等价于
System.out.println("2. instanceStringTypeValue1 = " + test.instanceStringTypeValue1.toString());


// 但是 String 对象重写 toString() 方法
public String toString() {
    return this;
}
```

---


### 2.8. 初始化阶段

#### 2.8.1. 初始化阶段概述

Java 虚拟机（JVM）的初始化阶段是类加载过程的最后一个阶段，也是极其关键的一步。在这个阶段，**JVM 会执行类的初始化方法，即 `clinit` 方法**。

在该方法中，JVM 会执行 `clinit` 字节码，对所有的静态变量（`static int staticVar = 10`）进行赋值，并执行静态代码块（`static { ... }`）中的代码（考虑到按需解析的机制，在执行过程中，若需要用到其他类，则按需加载、按需初始化）

为了确保类的初始化过程具备 **原子性** 和 **只执行一次** 的语义，JVM 会为每个 `Class` 对象维护一个内部锁（该锁对 Java 程序员不可见），因此，`clinit` 方法是线程安全的，一个类的 `clinit` 方法在整个生命周期中只会执行一次。

需要注意的是，类加载不一定会进行初始化，JVM 规范严格定义了以下几种情况，一个类或接口**必须**进行初始化：
1. 遇到 `new`、`getstatic`、`putstatic` 或 `invokestatic` 字节码指令时
	1. 使用 `new` 关键字实例化对象
	2. 访问类的静态字段时
		1. 被 `final` 修饰的静态字段，如果其值已在编译期确定，则不会触发初始化
	3. 访问类的静态方法时
2. 使用 `java.lang.reflect` 包的方法对类型进行反射调用时
	1. 例如，使用 `Class.forName("com.example.MyClass")` 或 `MyClass.class.newInstance()`
3. 初始化一个类的子类时
	1. 如果一个类还没有初始化，它的任何一个子类在初始化之前，父类都必须先初始化。
4. JVM 启动时指定的主类
	1. 包含 `main()` 方法的那个类，JVM 在启动时会先初始化这个类
5. JDK 7 引入的动态语言支持
	1. 当一个 `java.lang.invoke.MethodHandle` 实例的解析结果是 `REF_getStatic`、`REF_putStatic`、`REF_invokeStatic`、`REF_newInvokeSpecial` 四种类型的方法句柄时，并且这个方法句柄对应的类没有初始化时，需要先触发其初始化

下面几种情况不会触发类的初始化，这被称为被动引用：
1. 通过子类引用父类的静态字段
	1. 例如，`System.out.println(SubClass.parentStaticField);`
	2. 这只会初始化父类，而不会初始化子类
2. 通过数组定义来引用类
	1. 例如，`MyClass[] mcArray = new MyClass[10];`
	2. 这并不会触发 `MyClass` 类的初始化，它只是创建了一个 `MyClass` 类型的数组对象
3. 引用类的常量
	1. 例如，`System.out.println(MyClass.MY_CONSTANT);`
	2. 如果常量在编译阶段就被放入了调用类的常量池中（例如被 `final` 修饰的基本类型或字符串常量），那么本质上并没有直接引用定义常量的类，因此不会触发该类的初始化

> [!NOTE] 注意事项
> 1. `init` 方法是在你调用 `new` 指令的那个线程中执行的
> 2. 但是 `clinit` 方法通常是主线程或第一个引用类的线程执行的。在大多数情况下，如果一个类是应用程序的主类（包含 `main` 方法），或者它是第一个被某个特定线程引用的类，那么 `clinit` 方法就会由启动该应用程序的线程（通常是主线程）或第一个引用该类的线程来执行

---


#### 2.8.2. 静态变量处理

例如如下代码：
```
public class Test {

    // 1. 静态基本数据类型变量
    public static int staticBasicTypeValue = 50000;

    // 2. 静态字符串类型变量 1
    public static String staticStringTypeValue1 = "abc";

    // 3. 静态字符串类型变量 2
    public static String staticStringTypeValue2 = new String("def");

    // 4. 静态字符串类型变量 3
    public static String staticStringTypeValue3 = staticStringTypeValue2.intern();

    // 5. 静态引用类型变量
    public static Demo staticReferenceTypeValue = new Demo();


    public static void main(String[] args) {
        Test test = new Test();

        System.out.println("1. staticBasicTypeValue = " + Test.staticBasicTypeValue); // 50000
        System.out.println("2. staticStringTypeValue1 = " + Test.staticStringTypeValue1); // abc
        System.out.println("3. staticStringTypeValue2 = " + Test.staticStringTypeValue2); // def
        System.out.println("4. staticStringTypeValue3 = " + Test.staticStringTypeValue3); // def
        System.out.println("5. staticReferenceTypeValue = " + Test.staticReferenceTypeValue); // org.example.test.Demo@58372a00
    }
}
```

其 `clinit` 字节码是：
```
/**
 * ============================================
 * 1. 处理 staticBasicTypeValue
 * --------------------------------------------
 * ldc #53
 * - 将运行时常量池中索引 52 处的 int 值 50000 推入操作数栈
 *
 * putstatic #16
 * - 将栈顶的 50000 存入 Test 类的静态 int 字段 staticBasicTypeValue
 * ============================================
 */
 0 ldc #52 <50000>
 2 putstatic #16 <org/example/test/Test.staticBasicTypeValue : I>


/**
 * ============================================
 * 2. 处理 staticStringTypeValue1
 * --------------------------------------------
 * ldc #53
 * - 将运行时常量池中索引 53 处的 "abc" String 对象的引用推入操作数栈
 *
 * putstatic #30
 * - 将栈顶的 "abc" String 对象的引用存入 Test 类的静态 String 字段 staticStringTypeValue1
 * ============================================
 */
 5 ldc #53 <abc>
 7 putstatic #30 <org/example/test/Test.staticStringTypeValue1 : Ljava/lang/String;>


/**
 * ============================================
 * 3. 处理 staticStringTypeValue2
 * --------------------------------------------
 * new #55
 * - 在堆上为 java/lang/String 对象分配空间（仅分配内存并赋默认值，不执行接下来的对象实例化步骤）
 * - 然后把那个刚分配的对象引用压入操作数栈
 *
 * dup
 * - 复制栈顶的对象引用
 * - 现在操作数栈中有两份对象引用：[ uninitRef, uninitRef ]
 * 
 * ldc #57
 * - 将运行时常量池中索引 57 处的 "def" String 对象的引用推入操作数栈
 * - 现在操作数栈的内容有：[ uninitRef, uninitRef, defRef ]
 * 
 * 
 * invokespecial #59
 * - 调用 java/lang/String 的构造方法，消费最顶上的 [ uninitRef, defRef ] 两个操作数
 * - 然后初始化这个 String 对象，即第一个 uninitRef
 * - 初始化后只留下一个已经构造好的对象引用：[ initRef ]
 * 
 * putstatic #37
 * - 将栈顶的 String 对象的引用存入 Test 类的静态 String 字段 staticStringTypeValue2
 * ============================================
 */
10 new #55 <java/lang/String>
13 dup
14 ldc #57 <def>
16 invokespecial #59 <java/lang/String.<init> : (Ljava/lang/String;)V>
19 putstatic #37 <org/example/test/Test.staticStringTypeValue2 : Ljava/lang/String;>


// 4. 处理staticStringTypeValue3
22 getstatic #37 <org/example/test/Test.staticStringTypeValue2 : Ljava/lang/String;>
25 invokevirtual #61 <java/lang/String.intern : ()Ljava/lang/String;>
28 putstatic #41 <org/example/test/Test.staticStringTypeValue3 : Ljava/lang/String;>


// 5. 处理 staticReferenceTypeValue
31 new #65 <org/example/test/Demo>
34 dup
35 invokespecial #67 <org/example/test/Demo.<init> : ()V>
38 putstatic #45 <org/example/test/Test.staticReferenceTypeValue : Lorg/example/test/Demo;>

41 return
```

其 `init` 字节码是：
```
0 aload_0
1 invokespecial #1 <java/lang/Object.<init> : ()V>
4 return
```

其 `main` 字节码是：
```
// 1. Test test = new Test();
 0 new #7 <org/example/test/Test>
 3 dup
 4 invokespecial #9 <org/example/test/Test.<init> : ()V>
 7 astore_1


// 2. System.out.println("1. staticBasicTypeValue = " + Test.staticBasicTypeValue);
 8 getstatic #10 <java/lang/System.out : Ljava/io/PrintStream;>
11 getstatic #16 <org/example/test/Test.staticBasicTypeValue : I>
14 invokedynamic #20 <makeConcatWithConstants, BootstrapMethods #0>
19 invokevirtual #24 <java/io/PrintStream.println : (Ljava/lang/String;)V>


// 3. System.out.println("2. staticStringTypeValue1 = " + Test.staticStringTypeValue1);
22 getstatic #10 <java/lang/System.out : Ljava/io/PrintStream;>
25 getstatic #30 <org/example/test/Test.staticStringTypeValue1 : Ljava/lang/String;>
28 invokedynamic #34 <makeConcatWithConstants, BootstrapMethods #1>
33 invokevirtual #24 <java/io/PrintStream.println : (Ljava/lang/String;)V>


// 4. System.out.println("3. staticStringTypeValue2 = " + Test.staticStringTypeValue2);
36 getstatic #10 <java/lang/System.out : Ljava/io/PrintStream;>
39 getstatic #37 <org/example/test/Test.staticStringTypeValue2 : Ljava/lang/String;>
42 invokedynamic #40 <makeConcatWithConstants, BootstrapMethods #2>
47 invokevirtual #24 <java/io/PrintStream.println : (Ljava/lang/String;)V>


// 5. System.out.println("4. staticStringTypeValue3 = " + Test.staticStringTypeValue3);
50 getstatic #10 <java/lang/System.out : Ljava/io/PrintStream;>
53 getstatic #41 <org/example/test/Test.staticStringTypeValue3 : Ljava/lang/String;>
56 invokedynamic #44 <makeConcatWithConstants, BootstrapMethods #3>
61 invokevirtual #24 <java/io/PrintStream.println : (Ljava/lang/String;)V>


// 6. System.out.println("5. staticReferenceTypeValue = " + Test.staticReferenceTypeValue);
64 getstatic #10 <java/lang/System.out : Ljava/io/PrintStream;>
67 getstatic #45 <org/example/test/Test.staticReferenceTypeValue : Lorg/example/test/Demo;>
70 invokedynamic #49 <makeConcatWithConstants, BootstrapMethods #4>
75 invokevirtual #24 <java/io/PrintStream.println : (Ljava/lang/String;)V>
78 return
```

在该阶段的静态变量为：
![](image-20250729170742826.png)

其执行结果为：
![](image-20250729191615514.png)

---


## 3. 对象实例化

### 3.1. 对象实例化概述

在对象实例化时，JVM 会在堆内存中为该对象分配空间，所有成员变量都会分配内存并赋予默认值（**默认初始化**）。随后，开始执行对象的初始化逻辑，即调用其 `<init>` 方法，具体包括以下步骤：
1. 调用父类的构造器
	1. 如果存在父类（除了 `java.lang.Object` 之外，所有类都应有父类）
	2. 注意接口 `interface` 也没有父类
2. 执行实例变量、普通常量的赋值（**显示初始化**）
3. 执行实例代码块
	1. 例如 `{xxxx}`
4. 执行构造器中的代码（**构造器初始化**）

---


### 3.2. 实例变量处理

以如下代码为例：
```
public class Test {
		
    // 1. 实例基本数据类型变量
    public int instanceBasicTypeValue = 50000;
		
    // 2. 实例字符串类型变量 1
    public String instanceStringTypeValue1 = "abc";
		
    // 3. 实例字符串类型变量 2
    public String instanceStringTypeValue2 = new String("def");
		
    // 4. 实例字符串类型变量 3
    public String instanceStringTypeValue3 = instanceStringTypeValue2.intern();
		
    // 5. 实例引用类型变量
    public Demo instanceReferenceTypeValue = new Demo();
		
    public static void main(String[] args) {
        Test test = new Test();
		
        System.out.println("1. instanceBasicTypeValue = " + test.instanceBasicTypeValue); // 5
        System.out.println("2. instanceStringTypeValue1 = " + test.instanceStringTypeValue1); // abc
        System.out.println("3. instanceStringTypeValue2 = " + test.instanceStringTypeValue2); // def
        System.out.println("4. instanceStringTypeValue3 = " + test.instanceStringTypeValue3); // def
        System.out.println("5. instanceReferenceTypeValue = " + test.instanceReferenceTypeValue); // org.example.test.Demo@7699a589
		
    }
}
```

其 `init` 字节码是：
```
 0 aload_0
 1 invokespecial #1 <java/lang/Object.<init> : ()V>

/**
 * ============================================
 * 1. 处理 instanceBasicTypeValue
 * --------------------------------------------
 * aload_0
 * - 将当前对象的引用（this）压入操作数栈
 *
 * ldc #7
 * - 将运行时常量池中索引 7 处的 int 值 50000 推入操作数栈
 * - 对于较小的数值，JVM 不需要通过 ldc 指令加载，而是直接使用 iconst、bipush 或 sipush 等指令直接将常量压入操作数栈，从而加快了效率。
 *
 * putfield #8
 * - 将栈顶的 50000 赋值给当前对象（this）的 int 字段 instanceBasicTypeValue
 * ============================================
 */
 4 aload_0
 5 ldc #7 <50000>
 7 putfield #8 <org/example/test/Test.instanceBasicTypeValue : I>

 
/**
 * ============================================
 * 2. 处理 instanceStringTypeValue1
 * --------------------------------------------
 * aload_0
 * - 将当前对象的引用（this）压入操作数栈
 *
 * ldc #14
 * - 将运行时常量池中索引 14 处的 "abc" String 对象的引用推入操作数栈
 *
 * putfield #16
 * - 将栈顶的 "abc" String 对象的引用赋值给当前对象（this）的 String 字段 instanceBasicTypeValue1
 * ============================================
 */
10 aload_0
11 ldc #14 <abc>
13 putfield #16 <org/example/test/Test.instanceStringTypeValue1 : Ljava/lang/String;>


/**
 * ============================================
 * 3. 处理 instanceStringTypeValue2
 * --------------------------------------------
 * aload_0
 * - 将当前对象的引用（this）压入操作数栈
 *
 * new #20
 * - 在堆上为 java/lang/String 对象分配空间（仅分配内存并赋默认值，不执行接下来的对象实例化步骤）
 * - 然后把那个刚分配的对象引用压入操作数栈
 * - 为什么能恰到好处为 String 对象分配空间？
 *      - JVM 的一个基本原则是：在创建对象时，必须明确知道对象的大小，以便在堆中分配合适的内存块。这个大小信息在类的元数据中就已经确定了。
 *      - 你可能会好奇：如果对象中有一个 List 属性，那在对象创建后我不断向这个 List 添加元素，JVM 怎么知道我最终需要多少内存？
 *      - 别忘了，我们在对象中保存的只是对 List 实例的引用，这个引用的大小是固定不变的。那你又好奇了：为什么 List 能动态扩容？
 *      - 其实本质是：当容量不足时，会新建一个更大的数组对象，把原有内容复制进去，然后让 elementData 字段指向新的数组。

 *
 * dup
 * - 复制栈顶的对象引用
 *
 * ldc #22
 * - 将运行时常量池中索引 22 处的 "def" String 对象的引用推入操作数栈
 *
 * invokespecial #24
 * - 调用 java/lang/String 的构造方法，消费最顶上的 [ uninitRef, defRef ] 两个操作数
 * - 然后初始化这个 String 对象，即第一个 uninitRef
 *
 * putfield #27
 * - 将栈顶的 新创建的 String 实例的引用 赋值给当前对象（this）的 String 字段 instanceBasicTypeValue2 字段
 * ============================================
 */
16 aload_0
17 new #20 <java/lang/String>
20 dup
21 ldc #22 <def>
23 invokespecial #24 <java/lang/String.<init> : (Ljava/lang/String;)V>
26 putfield #27 <org/example/test/Test.instanceStringTypeValue2 : Ljava/lang/String;>


// 4. 处理 instanceStringTypeValue3
29 aload_0
30 aload_0
31 getfield #27 <org/example/test/Test.instanceStringTypeValue2 : Ljava/lang/String;>
34 invokevirtual #30 <java/lang/String.intern : ()Ljava/lang/String;>
37 putfield #34 <org/example/test/Test.instanceStringTypeValue3 : Ljava/lang/String;>


/**
 * ============================================
 * 5. 处理 instanceReferenceTypeValue
 * --------------------------------------------
 * aload_0
 * - 将当前对象的引用（this）压入操作数栈
 *
 * new #37
 * - 在堆上为 org/example/test/Demo 对象分配空间（仅分配内存并赋默认值，不执行接下来的对象实例化步骤）
 * - 然后把那个刚分配的对象引用压入操作数栈
 *
 * dup
 * - 复制栈顶的对象引用
*
 * invokespecial #39
 * - 调用 Demo 构造方法，在堆内存中创建新的 Demo 实例。
 *
 * putfield #40
 * - 将栈顶的 新创建的 Demo 实例的引用 赋值给当前对象（this）的 String 字段 instanceReferenceTypeValue
 * ============================================
 */
40 aload_0
41 new #37 <org/example/test/Demo>
44 dup
45 invokespecial #39 <org/example/test/Demo.<init> : ()V>
48 putfield #40 <org/example/test/Test.instanceReferenceTypeValue : Lorg/example/test/Demo;>

51 return
```

其 `main` 字节码是：
```
 0 new #9 <org/example/test/Test>
 3 dup
 4 invokespecial #44 <org/example/test/Test.<init> : ()V>
 7 astore_1
 8 getstatic #45 <java/lang/System.out : Ljava/io/PrintStream;>
11 aload_1
12 getfield #8 <org/example/test/Test.instanceBasicTypeValue : I>
15 invokedynamic #51 <makeConcatWithConstants, BootstrapMethods #0>
20 invokevirtual #55 <java/io/PrintStream.println : (Ljava/lang/String;)V>
23 getstatic #45 <java/lang/System.out : Ljava/io/PrintStream;>
26 aload_1
27 getfield #16 <org/example/test/Test.instanceStringTypeValue1 : Ljava/lang/String;>
30 invokedynamic #60 <makeConcatWithConstants, BootstrapMethods #1>
35 invokevirtual #55 <java/io/PrintStream.println : (Ljava/lang/String;)V>
38 getstatic #45 <java/lang/System.out : Ljava/io/PrintStream;>
41 aload_1
42 getfield #27 <org/example/test/Test.instanceStringTypeValue2 : Ljava/lang/String;>
45 invokedynamic #63 <makeConcatWithConstants, BootstrapMethods #2>
50 invokevirtual #55 <java/io/PrintStream.println : (Ljava/lang/String;)V>
53 getstatic #45 <java/lang/System.out : Ljava/io/PrintStream;>
56 aload_1
57 getfield #34 <org/example/test/Test.instanceStringTypeValue3 : Ljava/lang/String;>
60 invokedynamic #64 <makeConcatWithConstants, BootstrapMethods #3>
65 invokevirtual #55 <java/io/PrintStream.println : (Ljava/lang/String;)V>
68 getstatic #45 <java/lang/System.out : Ljava/io/PrintStream;>
71 aload_1
72 getfield #40 <org/example/test/Test.instanceReferenceTypeValue : Lorg/example/test/Demo;>
75 invokedynamic #65 <makeConcatWithConstants, BootstrapMethods #4>
80 invokevirtual #55 <java/io/PrintStream.println : (Ljava/lang/String;)V>
83 return
```

在该阶段的实例变量为：
![](image-20250729184405223.png)

其执行结果为：
![](image-20250727163945675.png)

---


### 3.3. 普通常量处理

普通常量的赋值方式与实例变量相同，而 `final` 则额外引入了 “只能赋值一次” 的限制。`final` 变量一旦初始化，其值就被锁定。如果是引用类型，锁定的是引用本身，而不是引用所指向对象的内容。

以如下代码为例：
```
public class Test {

    // 1. 基本数据类型常量
    public final int finalBasicTypeValue = 15;
	
    // 2. 字符串类型常量 1
    public final String finalStringTypeValue1 = "abc";
	
    // 3. 字符串类型常量 2
    public final String finalStringTypeValue2 = new String("def");
	
    // 4. 字符串类型常量 3
    public final String finalStringTypeValue3 = finalStringTypeValue2.intern();
	
    // 5. 引用类型常量
    public final Demo finalReferenceTypeValue = new Demo();
	
    public static void main(String[] args) {
        Test test = new Test();
	
        System.out.println("1. finalBasicTypeValue = " + test.finalBasicTypeValue); // 15
        System.out.println("2. finalStringTypeValue1 = " + test.finalStringTypeValue1); // abc
        System.out.println("3. finalStringTypeValue2 = " + test.finalStringTypeValue2); // def
        System.out.println("4. finalStringTypeValue3 = " + test.finalStringTypeValue3); // def
        System.out.println("5. finalReferenceTypeValue = " + test.finalReferenceTypeValue); // org.example.test.Demo@4dd8dc3
    }
}
```

其 `init` 字节码是：
```
 0 aload_0
 1 invokespecial #1 <java/lang/Object.<init> : ()V>
 4 aload_0
 5 bipush 15
 7 putfield #7 <org/example/test/Test.finalBasicTypeValue : I>
10 aload_0
11 ldc #13 <abc>
13 putfield #15 <org/example/test/Test.finalStringTypeValue1 : Ljava/lang/String;>
16 aload_0
17 new #19 <java/lang/String>
20 dup
21 ldc #21 <def>
23 invokespecial #23 <java/lang/String.<init> : (Ljava/lang/String;)V>
26 putfield #26 <org/example/test/Test.finalStringTypeValue2 : Ljava/lang/String;>
29 aload_0
30 aload_0
31 getfield #26 <org/example/test/Test.finalStringTypeValue2 : Ljava/lang/String;>
34 invokevirtual #29 <java/lang/String.intern : ()Ljava/lang/String;>
37 putfield #33 <org/example/test/Test.finalStringTypeValue3 : Ljava/lang/String;>
40 aload_0
41 new #36 <org/example/test/Demo>
44 dup
45 invokespecial #38 <org/example/test/Demo.<init> : ()V>
48 putfield #39 <org/example/test/Test.finalReferenceTypeValue : Lorg/example/test/Demo;>
51 return
```

其执行结果为：
![](image-20250728115451360.png)

---


### 3.4. 补充：方法参数和局部变量处理

方法参数和局部变量的处理并不是在对象实例化的过程中完成的，而是在对象方法被调用时进行的。具体来说：当某个线程调用对象方法时，JVM 会在该线程的虚拟机栈中为该方法创建一个栈帧，并开始执行方法字节码。

需要注意的是，局部变量和方法参数不会像静态变量、静态常量、实例变量、普通常量那样自动初始化默认值。所以如果你只是声明了一个局部变量（如 `int localVar;`），但没有显式赋值（如 `localVar = 5`），那么就会编译报错。

以如下代码为例：
```
public class Test {

    // 方法参数
    public void method(int param) {

        // 局部变量
        int localVar = 5;
        
        System.out.println("param: " + param);
        System.out.println("localVar: " + localVar);
    }

    public static void main(String[] args) {
        Test test = new Test();
        test.method(10);
    }
}
```

其 `method` 字节码是：
```
 0 iconst_5
 1 istore_2
 2 getstatic #7 <java/lang/System.out : Ljava/io/PrintStream;>
 5 iload_1
 6 invokedynamic #13 <makeConcatWithConstants, BootstrapMethods #0>
11 invokevirtual #17 <java/io/PrintStream.println : (Ljava/lang/String;)V>
14 getstatic #7 <java/lang/System.out : Ljava/io/PrintStream;>
17 iload_2
18 invokedynamic #23 <makeConcatWithConstants, BootstrapMethods #1>
23 invokevirtual #17 <java/io/PrintStream.println : (Ljava/lang/String;)V>
26 return
```

其 `main` 字节码是：
```
 0 new #24 <org/example/test/Test>
 3 dup
 4 invokespecial #26 <org/example/test/Test.<init> : ()V>
 7 astore_1
 8 aload_1
 9 bipush 10
11 invokevirtual #27 <org/example/test/Test.method : (I)V>
14 return
```

其 `init` 字节码是：
```
0 aload_0
1 invokespecial #1 <java/lang/Object.<init> : ()V>
4 return
```

---


## 4. 运行时数据区

### 4.1. 运行时数据区一览图

![](image-20250724223952607.png)

从线程是否共享的角度又可以为：
![](image-20250726201657760.png)

----


### 4.2. 程序计数器（PC 寄存器）

在 CPU 中，寄存器专门用于存储指令执行相关的现场信息，CPU 只有将数据加载到寄存器后才能执行指令。而在 JVM 中，PC 寄存器的全称是 Program Counter Register，中文通常译为 “程序计数器”，它是对物理寄存器的一种抽象。尽管名称相似，但它与物理 CPU 寄存器在功能上并不相同。

根据 JVM 规范，**每个线程都拥有独立的、私有的程序计数器**，其生命周期与线程一致，**用于记录当前线程所执行方法中下一条将要执行的 JVM 字节码指令的地址（偏移量）**，由执行引擎负责读取指令并执行。

PC 寄存器是一块极小的内存区域，几乎可以忽略不计，因为它只需保存一个字节码指令的地址。在 32 位 JVM 中，通常使用 `int` 类型，占 4 个字节；而在 64 位 JVM 中，通常使用 `long` 类型，占 8 个字节。

需要注意的是，如果执行的是本地方法（native），PC 的值为 `undefined`。本地方法是指用 Java 以外的语言（如 C、C++）编写，并编译成平台相关的机器码的方法，它们通过 Java 本地接口（JNI）被 Java 代码调用，但它实际上是离开了 JVM 的控制，转而去执行操作系统底层的机器码

![](image-20250722190040146.png)

> [!NOTE] 注意事项
> 1. 程序计数器不会发生 GC，也不会抛出 OOM 或其他错误
> 2. PC 寄存器为什么不存在 GC？
> 	1. 在 JVM 创建线程时，会分配一块连续的内存区域，称为线程执行上下文，用于维护该线程的完整运行状态，包括 PC 寄存器、虚拟机栈、本地方法栈、线程本地存储以及线程状态等
> 	2. 当线程结束时，JVM 和操作系统会释放该线程所有相关的资源
> 3. PC 寄存器为什么不存在 OOM？
> 4. 为什么使用 PC 寄存器记录对应线程的执行地址？
> 	1. 由于 CPU 需要频繁切换不同线程，切换回来时必须知道从哪里继续执行，因此通过 PC 寄存器保存线程的执行地址，实现精确恢复
> 5. PC 寄存器中只记录偏移量，但你却说它 “用于记录当前线程所执行方法中下一条将要执行的 JVM 字节码指令的地址（偏移量）”，那它是如何确保这个偏移量一定对应的是 “当前方法” 的指令？
> 	1. 程序计数器只记录字节码数组中的偏移量，但它会配合 JVM 虚拟机栈中当前线程的栈顶栈帧，因为每个方法对应一个栈帧，最顶层的栈帧代表当前正在执行的方法。

---


### 4.3. 虚拟机栈

#### 4.3.1. 虚拟机栈一览图

![](图像清晰化.png)

----


#### 4.3.2. 虚拟机栈概述


> [!NOTE] 注意事项
> 1. 虚拟机栈不会发生 GC，但会抛出 OOM 或 StackOverflowError，常见有：
> 	1. OOM
> 		1. OutOfMemoryError: unable to create new native thread
> 			1. JVM 尝试创建新线程时，系统无法分配栈内存
> 	2. StackOverflowError
> 		1. 栈帧太多（比如递归没结束），导致栈空间“满了”


----




#### 4.3.3. 局部变量表

局部变量表是一个一维的数字数组，用于保存**方法参数**和**方法体内部定义的局部变量**。它可以存储多种数据类型，包括基本数据类型和对象引用（reference），在 JDK5 之前还包含一种 returnAddress 类型（了解即可）。

![](image-20250723142505659.png)

局部变量表中，最基本的存储单元称为变量槽（Slot）。JVM 会为每一个 Slot 分配一个访问索引（从 index 0 开始），通过这个索引可以访问到局部变量表中的变量。需要注意的是，索引不一定是连续递增 1，例如 long、double 类型会占用两个槽位。

其中，32 位及以内的数据类型（如：`reference`、`returnAddress`、`byte`、`short`、`char`、`int`、`float`、`boolean`）占用一个 Slot；**64 位的数据类型**（如：`long` 和 `double`）则占用两个 Slot。

![](image-20250723130746731.png)

> [!NOTE] 注意事项
> 1. `byte`、`short`、`char` 在存储进局部变量表前，会被自动转换为 `int` 类型，对应关系有：
> 	1. 原始类型 ---- 在局部变量表的类型
> 	2. byte ---- int
> 	3. short ---- int
> 	4. char ---- int
> 	5. int ---- int
> 	6. boolean ---- int
> 	7. long ----- long
> 	8. float ---- float
> 	9. double ---- double
> 2. 在栈帧中，与性能调优关系最为密切的部分就是局部变量表
> 3. 当方法调用结束后，该方法对应的栈帧会被销毁，局部变量表也随之销毁。
> 4. 局部变量表所需的容量大小在编译期间就已经确定，运行时无法更改。
> 5. 如果当前线程调用的方法是构造方法或实例方法，那么代表当前对象的引用 `this`（即堆中对象的引用地址）会默认存放在局部变量表 index 为 0 的槽中。这也解释了为什么我们在构造方法和实例方法中可以使用 `this`，而在 `static` 方法中却不能使用：
> 	1. 在下面的示例中，当执行 `main` 方法时，会创建一个栈帧；
> 	2. 执行 `Demo.staticMethod();` 时，由于是静态方法，属于类级别，可以直接调用；
> 	3. 执行 `Demo demo = new Demo();` 时，调用的是构造方法 `<init>`，此时会创建一个新的栈帧，并自动将 `this` 引用压入 index 为 0 的槽位中；当 `<init>` 方法执行完毕，会返回 `this`，赋值给变量 `demo`；
> 	4. 调用 `demo.instanceMethod();` 时，同样会创建一个栈帧，并将 `this` 引用压入 index 为 0 的槽位。
> 	5. 你可能会疑惑：调用 `new Demo()` 和 `demo.instanceMethod()` 的时候，代码里并没有传递 `this`，为什么 `this` 也能压入槽？其实这是 Java 的一种语法糖 + 字节码优化机制。虽然源码中没写，但在编译成字节码时，JVM 会自动将 `this` 作为第一个参数加入，如果方法声明为：`method(int a, int b, int c)`，实际上在实例方法中对应的局部变量表会是：`[this, a, b, c]`
```
public class Demo {

    public static void main(String[] args) {
        Demo.staticMethod();
        Demo demo = new Demo();
        demo.instanceMethod();
    }

    public static void staticMethod() {
        System.out.println("Static");
    }

    public void instanceMethod() {
        System.out.println("Instance");
    }
}
```

> [!NOTE] 注意事项
>4. 槽位是可以复用的。当某个局部变量生命周期结束（超出作用域）后，在其作用域之外声明的新局部变量很可能会复用这个已释放的槽位，从而达到节省局部变量表空间的目的
>	1. 例如，在下面的示例代码中，`b` 会复用 `a` 所占用的槽位。因此，整个方法的局部变量表最大只需要 2 个槽位
```
public class Test {  
    public static void main(String[] args) {  
        {  
            int a = 10;  
            System.out.println(a);  
        }  
  
        {  
            int b = 20;  
            System.out.println(b);  
        }  
    }  
}
```


----


#### 4.3.4. 操作数栈

##### 4.3.4.1. 操作数栈概述

操作数栈是 JVM 执行引擎的一个工作区，会根据方法中的字节码指令，向栈中写入数据或提取数据，这些操作被称为入栈和出栈。它主要用于保存计算过程中的中间结果，同时作为变量的临时存储空间。

**我们常说 Java 虚拟机的解释引擎是基于栈的执行引擎，这里的 “栈” 指的就是操作数栈**。

需要注意的是，操作数栈是通过数组实现的栈结构，而**不是链表**，而且虽然它是基于数组实现的，但我们在设计时不应使用数组的随机访问等方法，而应遵循栈的操作特性，仅使用入栈和出栈操作。

下面用详细代码进行举例：
```
public class Test {

    public static void main(String[] args) {
        int i = 15;
        int j = 8;
        int k = i + j;
    }
	
}
```


<font color="#92d050">1. 指令地址 0</font>
![](image-20250723180139005.png)

> [!NOTE] 注意事项
> 1. 数据压入操作数栈后，类型依旧遵循以下规则：
> 	1. 原始类型 ---- 在局部变量表的类型
> 	2. byte ---- int
> 	3. short ---- int
> 	4. char ---- int
> 	5. int ---- int
> 	6. boolean ---- int
> 	7. long ----- long
> 	8. float ---- float
> 	9. double ---- double
> 2. 32 位及以内的数据类型（如：`reference`、`returnAddress`、`byte`、`short`、`char`、`int`、`float`、`boolean`）占用一个栈单位深度；64 位的数据类型（如：`long` 和 `double`）则占用两个栈单位深度。
> 3. 与局部变量表一致，操作数栈最大深度在编译期间就已经确定，运行时无法更改（栈顶缓存技术不会影响最大深度）


<font color="#92d050">2. 指令地址 2</font>
![](image-20250723180026214.png)


<font color="#92d050">3. 指令地址 3 </font>
![](image-20250723180216377.png)


<font color="#92d050">4. 指令地址 5</font>
![](image-20250723180252011.png)


<font color="#92d050">5. 指令地址 6</font>
![](image-20250723180343374.png)


<font color="#92d050">6. 指令地址 7</font>
![](image-20250723180532258.png)


<font color="#92d050">7. 指令地址 8</font>
![](image-20250723180621968.png)


<font color="#92d050">8. 指令地址 9</font>
![](image-20250723180737908.png)

> [!NOTE] 注意事项
> 1. 在该方法中，最多仅有两个操作数同时存在于操作数栈中，因此操作数栈最大深度为 2


<font color="#92d050">9. 最终效果</font>
![](image-20250723173022583.png)

---


##### 4.3.4.2. 栈顶缓存技术

JVM 是基于栈的架构，它不像物理 CPU 那样通过寄存器来存放操作数并参与计算，而是依赖操作数栈（Operand Stack）来完成大部分运算。因此，JVM 的指令通常不携带操作数，比如 `iadd`（整数加法）会默认从栈顶弹出两个数，相加后再将结果压回栈中，使字节码更加简洁紧凑。

这种设计虽然带来了跨平台性强、字节码体积小、指令格式简单等优点，但也引入了性能问题：所有操作都依赖栈，频繁的入栈、出栈操作会导致大量内存访问，从而拖慢指令执行速度。一次简单的加法可能需要经历 `load → push → push → iadd → store` 的多个步骤，而不像寄存器架构那样仅需一条指令。此外，每次入栈和出栈本质上都是对 JVM 内存结构（即栈帧中的操作数虚拟机栈域）的访问，而内存访问速度远不如 CPU 寄存器。

为了解决这一问题，HotSpot JVM 的设计者提出了栈顶缓存（ToS，Top-of-Stack Caching）技术，将操作数栈顶部的若干个元素缓存在 CPU 寄存器中，以减少对操作数栈内存的频繁访问，从而提升执行引擎的整体执行效率。

在解释执行阶段，字节码逐条执行，大部分值都直接读写内存中的操作数栈，寄存器的使用非常有限（仅用于短暂调度）。而当某个方法成为热点，触发 JIT 编译后，**JIT 编译器**会在生成本地机器码的过程中启用栈顶缓存技术，将栈顶的值尽可能缓存在寄存器中。

然而，由于硬件本身的限制，寄存器数量十分有限。以常见的 x86 架构为例，通用寄存器仅有 8 个（`eax`、`ebx`、`ecx`、`edx`、`esi`、`edi`、`ebp`、`esp`），即便是现代的 x86-64 架构也只有 16 个。这些寄存器还需要被用于局部变量、参数传递等其他任务，因此必须通过非常严格的寄存器分配与抢占策略进行管理。这也正是栈顶缓存技术只被应用于热点方法，而非所有方法的原因。并且寄存器能够缓存的数据非常有限，但通常只缓存 2~4 个槽位就已能显著提升性能。

需要注意的是，只有在必要的场景下，寄存器中的数据才会被同步回栈内存（即先存入 CPU 寄存器，必要时才写回内存中的操作数栈）：
1. 调用方法前
	1. 如果要调用某个方法，而该方法需要从栈顶获取参数，就必须先将寄存器中的值写回栈内存，确保方法调用机制能按照字节码协议正确读取参数
2. 操作数栈溢出时
	1. 当需要缓存的栈顶数据超过寄存器容量时，旧数据必须写回栈中，以腾出寄存器空间
3. 数据生命周期结束前的检查
	1. 如果某个值在之后不会再使用，为了保持字节码语义的正确性（如异常处理或 GC 根节点扫描），可能会提前将其写回栈中
4. 调试 / 异常机制介入时
	1. 当调试器需要访问栈上值，或在发生异常抛出、栈回溯等情况下，JVM 需还原执行现场，也必须将寄存器中的值同步回操作数栈

----


#### 4.3.5. 动态链接

---

#### 4.3.6. 虚拟机栈异常示例

---


### 4.4. 本地方法栈




---


### 4.5. 堆区

#### 4.5.1. 堆区一览图

![](image-20250725114229098.png)

----


#### 4.5.2. 堆区概述

堆区是 Java 内存管理的核心区域，也是整个内存结构最大的区域，几乎所有的对象实例都会被分配到堆上（需考虑逃逸分析的优化可能）。

堆区是所有线程共享的内存区域（但需考虑线程私有的 TLAB 缓冲区机制），其底层物理内存可以是非连续的，但在逻辑上应被视为连续的空间。

> [!NOTE] 注意事项
> 1. 堆区既会发生 GC，也会抛出 OOM，常见有：
> 	1. OutOfMemoryError: Java heap space
> 		1. 堆内存空间不足，无法为新对象分配内存
> 2. 堆区的大小可以通过 JVM 参数手动指定，包括初始空间和最大空间，且在运行时具有动态扩展能力
> 	1. -Xms512m：
> 		1. 用于设置堆区的初始内存大小
> 		2. 例如：java -Xms512m YourApp
> 		3. 默认值：物理内存的 1/64
> 		4. 常用单位：k、m、g
> 	2. -Xmx2048m：
> 		1. 用于设置堆区的最大内存大小
> 		2. 例如：java -Xmx2048m YourApp
> 		3. 默认值：物理内存的 1/4
> 	3. 实际使用中，通常将 -Xms 和 -Xmx 设置为相同的值，
> 		1. 为了避免 JVM 在运行期间，频繁触发 GC、堆空间扩缩容

---


#### 对象在堆区的内存分配过程

我们 new 的对象先放到 Eden 区：
1. 如果 Eden 区空间不足
	1. 





#### 4.5.3. 堆区异常示例

#####  OutOfMemoryError: Java heap space

```
public class HeapOOMExample {

    public static void main(String[] args) {
    
        List<byte[]> list = new ArrayList<>();
        System.out.println("尝试分配大量内存，观察 OOM...");
        while (true) {
            // 每次循环创建一个 5MB 的字节数组（数组也是对象），并添加到列表中
            // 列表持有对这些数组的引用，阻止它们被GC回收
            list.add(new byte[5 * 1024 * 1024]); // 5MB
        }
    }
}
```

> [!NOTE] 注意事项
> 1. `new byte[5 * 1024 * 1024]` 会在 Java 堆内存中开辟一块 **5MB** 的连续空间，用于存放这个 `byte` 数组的实际数据
> 2. `list.add()` 操作并不会把这 5MB 的数据复制到 `ArrayList` 内部，相反，`ArrayList` 会在其内部的数组中，存储这个 `byte` 数组的引用

----


##### OutOfMemoryError: GC overhead limit exceeded













#### 堆区 GC 过程

---

在 JVM 中，堆区的垃圾回收（GC）与方法区（PermGen/Metaspace）有相似之处，但也有关键的不同点。堆区是 Java 应用程序运行时对象实例的主要存储区域，其 GC 过程通常更为频繁和复杂。

##### 堆区的 GC 触发机制与过程

堆区主要关注的是**对象实例的回收**，而不是类的卸载。其垃圾回收主要分为以下几种类型：

##### 1. Minor GC (或 Young GC)

- **触发条件：** 当新生代（Young Generation）的 Eden 区空间不足，无法为新的对象分配内存时，会触发 Minor GC。
    
- **回收范围：** 主要回收新生代中的垃圾对象。
    
- **特点：** Minor GC 通常非常频繁，但由于新生代中的对象生命周期短，且回收效率高，所以它的暂停时间（Stop-The-World, STW）通常很短。
    

##### 2. Major GC (或 Old GC)

- **触发条件：**
    
    - **老年代空间不足：** 当老年代（Old Generation）空间不足，无法容纳新生代晋升过来的对象，或者无法为大对象分配内存时。
        
    - **CMS GC 失败：** 如果使用 CMS 垃圾收集器，在并发清理阶段预测失败，或者并发模式失败（Concurrent Mode Failure），也会触发 Full GC。
        
    - **其他收集器特定触发条件：** 例如 G1 收集器在 Mixed GC 之后，如果内存仍然不足，可能会退化到 Full GC。
        
- **回收范围：** 主要回收老年代中的垃圾对象。
    
- **特点：** Major GC 的暂停时间通常比 Minor GC 长，因为它需要扫描并回收整个老年代。
    

##### 3. Full GC

- **触发条件：**
    
    - **老年代空间严重不足：** 这是最常见的 Full GC 触发原因。
        
    - **方法区空间不足：** 如你所述，方法区触及高水位线时，JVM 会尝试触发 Full GC。
        
    - **System.gc() 调用：** 应用程序代码显式调用 `System.gc()`（不推荐，因为它只是一种建议，JVM 不保证执行，且可能引发不必要的 Full GC）。
        
    - **堆内存分配失败：** 在 Minor GC 或 Major GC 后，如果仍然无法分配内存。
        
    - **某些 JVM 参数设置：** 例如，`-XX:+HeapDumpOnOutOfMemoryError` 可能会在 OOM 前触发一次 Full GC 以生成堆转储文件。
        
- **回收范围：** **Full GC 会对整个堆（新生代、老年代）以及方法区进行垃圾回收。**
    
- **特点：** Full GC 通常会导致较长的暂停时间，对应用程序的性能影响最大。应尽量避免 Full GC 的频繁发生。
    

---

##### 堆区的 OOM (Out Of Memory) 异常

当堆区的内存不足，且 JVM 无法通过垃圾回收（包括 Full GC）释放足够的空间来满足新的内存分配请求时，就会抛出 `OutOfMemoryError: Java heap space` 异常。

这表示 JVM 已经用尽了 `-Xmx` 参数设置的最大堆内存，并且无法再分配任何新的对象。

---

##### 总结

与方法区的 GC 主要关注**类的卸载**不同，堆区的 GC 主要关注**对象实例的回收**。堆区的 GC 更为频繁，且根据不同的区域（新生代、老年代）有不同的回收策略（Minor GC, Major GC）。当这些回收机制无法满足内存需求时，最终会触发影响最大的 Full GC，如果仍然无法解决，便会抛出 `OutOfMemoryError: Java heap space` 错误。

希望这能帮你理解堆区的 GC 机制！你对方法区和堆区的 GC 机制还有其他疑问吗？





#### 堆区常用 JVM 参数

1. 设置堆区的初始内存大小
	1. -Xms
		1. 例如：java -Xms512m -jar MyApp.jar
		2. 默认值：物理内存的 1/64
		3. 常用单位有：k、m、g
2. 设置堆区的最大内存大小
	1. -Xmx
		1. 例如：java -Xmx2048m -jar MyApp.jar
		2. 默认值：物理内存的 1/4
3. 设置新生代与老年代在堆区的占比
	1. -XX:NewRatio=2
		1. 例如：java -XX:NewRatio=2 -jar MyApp.jar，表示新生代占堆区的 1/3，老年代占堆区的 2/3
		2. 默认值：新生代占堆区的 1/3，老年代占堆区的 2/3
4. 设置 Eden 区与 Survivor 区在新生代的占比
	1. -XX:SurvivorRatio=8
		1. 例如：java -Xmn100m -XX:SurvivorRatio=8 -jar MyApp.jar，表示 Eden : Survivor0 : Survivor1 = 8 : 1 : 1
		2. 默认值：Eden : Survivor0 : Survivor1 = 8 : 1 : 1
5. 设置对象存活年龄
	1. -XX:MaxTenuringThreshold=15
		1. 例如：java -XX:MaxTenuringThreshold=15 -jar MyApp.jar
		2. 默认值：15
6. 关闭 TLAB
	1. -XX:-UseTLAB
		1. 例如：java -XX:-UseTlab -jar MyApp.jar
		2. 默认值：开启 TLAB
7. 打印 TLAB 分配和使用情况
	1. -XX:+PrintTLAB
		1. JDK 8 及以前
		2. 例如：java -XX:+PrintTLAB -jar MyApp.jar
	2. -Xlog:gc+tlab=trace
		1. JDK 9 及以后
		2. 例如：java -Xlog:gc+tlab=trace -jar MyApp.jar
8. 设置 TLAB 剩余空间率
	1. -XX:TLABWasteTargetPercent=50
		1. 例如：java -XX:TLABWasteTargetPercent=50 -jar MyApp.jar
		2. 默认值：1

---



























































































































### 4.6. 方法区

#### 4.6.1. 方法区一览图

![](image-20250729130316083.png)

> [!NOTE] 注意事项
> 1. 每一个在堆中的实例对象，都会持有一个指向其类元数据（位于方法区）的指针。我们知道，在对象头中存在一个名为 `Klass Word` 的字段，它正是用于存储这个类元数据的指针。

---


#### 4.6.2. 方法区概述

方法区是用来存储已经被 JVM 加载的各种类的元数据信息、类的静态变量，以及即时编译器（JIT）编译后生成的代码缓存。

方法区和堆区一样，都是所有线程共享的内存区域。其底层物理内存可以是非连续的，但在逻辑上应被视为连续的空间。

JDK1.7 之前的永久代、JDK1.8 之后的元空间，都是对 JVM 规范中方法区的实现，其最大的区别在于：元空间不再使用 JVM 虚拟机设置的堆内存，而是改为使用本地内存，因此更少发生 OOM异常。
1. 永久代（PermGen） 是 JVM 内存的一部分，大小由 `-XX:PermSize` 和 `-XX:MaxPermSize` 限制，默认值往往较小（比如几十 MB），而且很容易超过上限，从而导致 OOM
2. 元空间（Metaspace） 使用的是 本地内存（Native memory），大小可以远远超过 JVM 堆的限制，甚至可以动态扩展至整个物理内存的上限

> [!NOTE] 注意事项
> 2. 方法区既会发生 GC，也会抛出 OOM，常见有：
> 	1. OutOfMemoryError: Metaspace
> 		1. 元空间内存不足
> 	2. OutOfMemoryError: PermGen space
> 		1. 永久代内存不足
> 3. 方法区的大小可以通过 JVM 参数手动指定，包括初始空间和最大空间，且在运行时具有动态扩展能力
> 	1. 永久代
> 		1. -XX:PermSize=64m
> 			1. 用于设置方法区的初始内存大小
> 			2. 例如：java -XX:PermSize=64m -XX:MaxPermSize=128m -jar MyApp.jar
> 			3. 默认值：20.75m
> 			4. 常用单位：k、m、g
> 		2. -XX:MaxPermSize=128m
> 			1. 用于设置方法区的最大内存大小
> 			2. 例如：java -XX:PermSize=64m -XX:MaxPermSize=128m -jar MyApp.jar
> 			3. 默认值：32 位机器默认是 64 M，64 位机器默认是 82 M
> 	2. 元空间
> 		1. -XX:MetaspaceSize=128m
> 			1. 用于设置方法区的初始内存大小
> 			2. 例如：java -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=256m -jar MyApp.jar
> 			3. 默认值：21m
> 			4. 注意事项：
> 				1. 如果初始高水位线（-XX:MetaspaceSize）设置过低，可能会导致 Full GC 频繁触发，建议将其设置为一个相对较高的值以减少不必要的 GC 开销。
> 		2. -XX:MaxMetaspaceSize=256m
> 			1. 用于设置方法区的最大内存大小
> 			2. 例如：java -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=256m -jar MyApp.jar
> 			3. 默认值：-1，既没有限制

---


#### 4.6.3. 类元信息

以下是类元信息中通常包含的关键内容：
1. 类的全限定名
	1. 例如：`java/lang/String`
	2. 注意事项：
		1. 当我们日常编写时，类的全限定名是用点号，例如：`java.lang.String`
		2. 但是在 JVM 内部、字节码文件（.class 文件）以及 JNI 描述符中，是使用斜杠 **/**
2. 直接父类的全限定名
	1. 除了 `java.lang.Object`，所有类都应该有父类
	2. 注意接口 `interface` 也没有父类
3. 直接实现的接口的全限定名列表
	1. 因为可能实现很多接口，所以这里用列表进行记录
4. 类的修饰符
	1. 访问修饰符
		1. public
			1. 任何地方都可以访问
		2. protected
			1. 同一个包内可以访问
			2. 不同包的**子类**可以访问
		3. package-private（默认）
			1. 只能在同一个包内访问
			2. 如果不指定任何访问修饰符，则为默认访问级别
		4. private
			1. 只能在定义它的类内部访问
			2. 这也是为什么当我们为字段添加 `private` 修饰符时，需要编写相应的 Getter 和 Setter 方法，因为除了该类自身，其他类根本无法直接访问这个字段。
	2. 非访问修饰符
		1. final
			1. 最终类，不能被继承
		2. static
			1. 静态嵌套类
			2. 这个修饰符不能直接用于外部类，只能用于**嵌套类 (Nested Classes)**
		3. abstract
			1. 抽象类，不能被实例化，必须被继承。
	3. 注意事项：
		1. 此外，还有类型本身的种类标识符，用于 JVM 判断该类到底是普通类、接口、枚举类，还是注解类型：
			1. class
			2. interface
			3. enum
			4. @interface
		2. 虽然我们通常认为 `enum` 是一种特殊的 `class`，`@interface` 是一种特殊的 `interface`，但本质上它们都是处于同一级别的关键字，并在字节码层面**通过不同的访问标志加以区分**
	4. 这些信息在 `.class` 文件中都包含在 `access_flags` 字段。
5. 类加载器引用
	1. 指向加载这个类的那个类加载器实例的引用

> [!NOTE] 注意事项
> 1. 无论是类、接口还是枚举、注解，在 JVM 的底层实现和运行层面，它们最终都被表示为 **Class**，因此都会在类的元数据信息中占有一席之地。

---

#### 4.6.4. 域信息（字段信息）

1. 字段名称
2. 字段类型
3. 字段修饰符
	1. 访问修饰符
		1. public
		2. protected
		3. package-private（默认）
		4. private
	2. 非访问修饰符
		1. final
			1. 常量字段，其值一旦初始化就不能修改
		2. static
		3. volatile
			1. 用于 JUC
		4. transient
			1. 用于 Java 序列化机制
	3. 注意事项：
		1. 此外，还有一些注解信息，如果注解使用了 `@Retention(RetentionPolicy.RUNTIME)`，那么它就会被保留到运行时，并存储元数据中。
		2. 例如 `deprecated`，它是通过 `@Deprecated` 注解标注的字段，表示该字段已被废弃，不推荐使用。
		3. 虽然这不属于传统意义上的关键字修饰符，但它确实会被记录在元数据中。

---


#### 4.6.5. 方法信息

1. 方法名称
2. 方法返回类型
3. 方法的修饰符
	1. 访问修饰符
		1. public
		2. protected
		3. package-private（默认）
		4. private
	2. 非访问修饰符
		1. final
			1. 该方法不能被子类重写（oberride）
		2. static
			1. 静态方法，属于类本身，可以直接通过类名调用
		3. abstract
			1. 抽象方法，只有方法签名，没有方法体
			2. 非抽象子类必须提供其实现
		4. synchronized
		5. native
			1. 表示该方法是用**非 Java 语言**（如 C/C++）实现的
			2. 方法体在外部定义，通常通过 Java Native Interface (JNI) 调用
	3. 注意事项：
		1. 此外，还有一些注解信息，如果注解使用了 `@Retention(RetentionPolicy.RUNTIME)`，那么它就会被保留到运行时，并存储元数据中。
		2. 此外还有 `deprecated`，它是通过 `@Deprecated` 注解标注的方法，表示该方法已被废弃，不推荐使用。
		3. 虽然这不属于传统意义上的关键字修饰符，但它确实会被记录在元数据中。
4. 方法的字节码
	1. 这是方法实际执行逻辑的体现
	2. JVM 会直接解释或即时编译并执行这些指令
5. 操作数栈最大深度
	1. JVM 需要知道执行方法时需要分配多少操作数栈空间
6. 局部变量表大小
	1. JVM 需要知道执行方法时需要分配多少局部变量表空间
7. 异常表
	1. 它包含了 `try-catch-finally` 块的范围、捕获的异常类型以及处理异常代码的起始位置
	2. 当程序抛出异常时，JVM 会根据异常表来查找对应的异常处理器

---


#### 4.6.6. 运行时常量池

存储了各种 **字面量（如字符串常量、数值常量）** 和**符号引用**，符号引用将在解析阶段或者程序运行时被解析为直接引用。

---












#### 4.6.7. 方法区 GC 过程

当方法区（Metaspace 或 PermGen）的使用量，触及初始高水位线（即默认的 -XX:MetaspaceSize）时，JVM 会尝试触发一次 Full GC

Full GC 会对新生代、老年代、方法区进行垃圾回收，其中在方法区中主要涉及到类的卸载
	1. 类的卸载比较严格，当满足以下三个条件时，**一个类及与其关联的数据**才有可能被卸载：
		1. 该类的所有实例都已被回收
		2. 加载该类的 `ClassLoader` 已经被回收
		3. 该类对应的 `java.lang.Class` 对象没有在任何地方被引用
	2. 多数情况下，一个类的 `ClassLoader` 和 `Class` 对象都会被长期引用

触发 Full GC 后，高水位线会被动态调整，新的高水位线取决于 GC 后释放了多少空间：
1. 如果释放的空间不足，在不超过 MaxMetaspaceSize 限制时，会适当提高高水位线
2. 如果释放的空间过多，则会适当降低高水位线

如果 Full GC 后空间依然不足，且超过了 MaxMetaspaceSize 限制，则 JVM 会抛出 OOM 异常：
1. OutOfMemoryError: PermGen space
	1. 永久代内存不足
2. OutOfMemoryError: Metaspace
	1. 元空间内存不足

> [!NOTE] 注意事项
> 1. 如果初始高水位线（-XX:MetaspaceSize）设置过低，可能会导致 Full GC 频繁触发，建议将其设置为一个相对较高的值以减少不必要的 GC 开销。

---


#### 4.6.8. 方法区常用 JVM 参数

1. 设置方法区的初始内存大小
	1. 永久代
		1. -XX:PermSize=64m
			1. 例如：java -XX:PermSize=64m -XX:MaxPermSize=128m -jar MyApp.jar
			2. 默认值：20.75m
			3. 常用单位：k、m、g
	2. 元空间
		1. -XX:MetaspaceSize=128m
			1. 例如：java -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=256m -jar MyApp.jar
			2. 默认值：21m
			3. 注意事项：
				1. 如果初始高水位线（-XX:MetaspaceSize）设置过低，可能会导致 Full GC 频繁触发，建议将其设置为一个相对较高的值以减少不必要的 GC 开销。
2. 用于设置方法区的最大内存大小
	1. 永久代
		1. -XX:MaxPermSize=128m
			1. 例如：java -XX:PermSize=64m -XX:MaxPermSize=128m -jar MyApp.jar
			2. 默认值：32 位机器默认是 64 M，64 位机器默认是 82 M
	2. 元空间
		1. -XX:MaxMetaspaceSize=256m
			1. 用于设置方法区的最大内存大小
			2. 默认值：-1，既没有限制
3. 禁止方法区中类的卸载
	1. -Xnoclassgc
		1. 例如：java -Xnoclassgc -jar MyApp.jar
4. 在类被加载时输出详细日志
	1. -XX:+TraceClassLoading
		1. JDK 8 及以前
	2. -Xlog:class+load
		1. JDK 9 及以后
5. 在类被卸载时输出详细日志
	1. -XX:+TraceClassUnloading
		1. JDK 8 及以前
	2. -Xlog:class+unload
		1. JDK 9 及以后

---




















---


## 5. 垃圾回收（GC）

### 5.1. 垃圾回收概述

垃圾回收是指在程序运行过程中，对堆区中没有任何指针指向的对象进行回收，对在方法区中主要涉及到常量池回收和类的卸载。

对方法区的回收，主要涉及到：
1. 常量池的回收
2. 类的卸载
	1. 类的卸载比较严格，当满足以下三个条件时，一个类及与其关联的数据才有可能被卸载：
		1. 该类的所有实例都已被回收
		2. 加载该类的 `ClassLoader` 已经被回收
		3. 该类对应的 `java.lang.Class` 对象没有在任何地方被引用
	2. 多数情况下，一个类的 `ClassLoader` 和 `Class` 对象都会被长期引用

但是，堆区是垃圾收集器的主要工作区域，这里我们主要研究堆区的垃圾收集，从回收频率上看，通常遵循以下规律：
1. 频繁收集新生代
2. 较少收集老年代
3. 基本不动永久代（方法区、元空间）

---


### 5.2. 垃圾标记算法

#### 5.2.1. 垃圾标记概述

在堆中存放着几乎所有的 Java 对象实例，在 GC 执行垃圾回收之前，首先需要区分内存中哪些是存活的对象，哪些是已经死亡的对象。

只有被标记为死亡的对象，GC 在执行回收时才会释放它们所占用的内存空间，因此，这一过程被称为垃圾标记阶段。

---


#### 5.2.2. 引用计数法

引用计数算法是为每一个对象维护一个整型的引用计数器**属性**，用于记录该对象被引用的次数。例如，对于一个对象 A，只要有其他任何对象引用了 A，则 A 的引用计数器就加 1；当引用失效时，计数器减 1。当 A 的引用计数器为 0 时，说明该对象已经无法再被访问，可以被回收。

其优点是：
1. 实现简单
2. 垃圾对象易于识别
3. 判定效率高，回收无需等待

其缺点是：
1. 增加了一定的存储空间的开销
2. 每次赋值都需要更新计数器，伴随着加法和减法操作，带来一定的时间开销
3. 前两点尚可接受，但引用计数法有一个致命缺陷：**无法处理循环引用的问题**，这正是 Java 的垃圾回收器没有采用该算法的根本原因

下面的代码是一个典型的循环引用示例：
```
public class RefCountGC {
		
    private byte[] bigSize = new byte[5 * 1024 * 1024]; //5MB
    Object reference = null;
		
    public static void main(String[] args) {
	    
        RefCountGC obj1 = new RefCountGC();
        RefCountGC obj2 = new RefCountGC();
		
        obj1.reference = obj2;
        obj2.reference = obj1;
		
        obj1 = null;
        obj2 = null;
        
        // 显式执行 Full GC，若使用引用计数法，obj1 和 obj2 将无法被回收
        System.gc();
    }
}
```

其图示如下：
![](image-20250727083631864.png)

> [!NOTE] 注意事项
> 1. 由于 Java 并未采用引用计数算法，所以在面试中回答 “什么会导致内存泄漏” 这类问题时，应避免以“循环引用”作为主要答案

---


#### 5.2.3. 可达性分析（根搜索方法、追踪性垃圾收集）

##### 5.2.3.1. 可达性分析概述

可达性分析算法是以根对象集合（GC Roots，一组始终处于活跃状态的引用）为起始点，自上而下地搜索与这些根对象相关联的目标对象是否可达，内存中的所有存活对象，都会被根对象集合直接或间接连接。

搜索过程中所经过的路径被称为引用链（Reference Chain），如果某个目标对象在内存中没有任何引用链与 GC Roots 相连，那么它就是不可达的，也就意味着该对象已死亡，可被标记为垃圾对象。

> [!NOTE] 注意事项
> 1. 如果要使用可达性分析算法判断内存是否可回收，分析工作必须在一个能够**保证内存状态一致性的快照中进行**。
> 2. 如果无法满足这一点，分析结果的准确性就无法保证，这也是导致 GC 进行时必须执行 “Stop The World” 的一个重要原因。
> 3. 即便是号称几乎不会停顿的 CMS 收集器，在枚举根节点阶段，也必须暂停所有应用线程（主线程、工作线程、守护线程）

---


##### 5.2.3.2. 常见 GC Roots

<font color="#92d050">1. 虚拟机栈中引用的对象</font>





---


##### 5.2.3.3. 查看 GC Roots

---


### 5.3. 垃圾清除算法

#### 5.3.1. 标记-清除算法

标记-清除算法是垃圾回收中最基本和最古老的一种算法，也是许多现代垃圾回收算法的基础。它的工作原理正如其名称所示，分为两个主要阶段：**标记 (Mark)** 和 **清除 (Sweep)**
1. 标记：
	1. 垃圾回收器从一组被称为 **GC Roots** 的对象出发，遍历所有从 GC Roots 可达的对象，并**标记**这些对象为“存活”或“in use”（通常通过在对象头中设置一个标记位 `mark bit` 来实现）
	2. 这个遍历过程通常采用递归的深度优先搜索（DFS）或广度优先搜索（BFS）方式，沿着对象引用链不断向下，直到所有可达对象都被标记完毕。
2. 清楚：
	1. 垃圾回收器会遍历整个堆内存。
	2. 对于那些在标记阶段**未被标记**的对象（即不可达对象），它们的内存会被回收，并把这些地址加入到空闲内存列表中，供后续的新对象分配使用
	3. 对于已标记为存活的对象，它们的标记位会在此阶段被清除，为下一次垃圾回收做准备。

![](image-20250728174155278.png)

其优点是：
1. 实现简单：
	1. 标记-清除算法逻辑比较直观，容易实现
2. 能够处理循环引用

其缺点是：
1. 内存碎片化：
	1. 回收后的内存空间不连续，产生大量的内存碎片
	2. 需要额外维护一个空闲列表用于记录空闲地址，增加内存开销
	3. 当后续需要分配较大对象时，即使总的空闲内存足够，也可能因为没有足够大的连续空间而导致分配失败
2. 回收效率中等：
	1. 标记阶段的工作量，与**存活对象的数量及其引用关系复杂程度**成正比，如果堆中绝大部分是垃圾，那么标记的开销相对较小（这个工作量，所有垃圾清除算法都一样）
	2. 但清除阶段的工作量，与**整个堆内存的大小**成正比，无论堆中有多少存活对象，清除阶段都需要线性扫描整个堆，从而导致效率不算高

> [!NOTE] 注意事项
> 1. 无论采用哪种垃圾回收算法，也无论使用哪种垃圾回收器，都不可避免地会存在 STW（Stop The World），即暂停所有应用线程以执行垃圾回收操作。
> 2. 一个优秀的垃圾回收器，必须在吞吐量与响应速度之间取得良好平衡，既能高效回收内存，又不会严重影响程序的响应性能。

---


#### 5.3.2. 复制算法

复制算法同样基于可达性分析，从 From 空间中识别出所有存活对象。一旦找到这些对象，GC 会将它们复制到 To 空间中，并按顺序紧凑排列，从而自然地消除了内存碎片。并且在复制的同时更新所有引用，确保指向已移动对象的指针保持正确。

复制完成后，From 空间中未被复制的对象（即不可达对象）被视为垃圾，整个 From 空间会被整体清空。下一次垃圾回收时，From 和 To 空间将交换角色，继续重复同样的回收流程。

总结来说，复制算法以牺牲空间换取了时间，并彻底解决了内存碎片问题。
![](image-20250728181214127.png)

其优点是：
1. 实现简单：
	1. 相比于复杂的标记-整理算法，复制算法的基本逻辑相对直观，容易实现
2. 消除内存碎片化
3. 回收效率最高：
	1. 标记阶段的工作量，与**存活对象的数量及其引用关系复杂程度**成正比，如果堆中绝大部分是垃圾，那么标记的开销相对较小（这个工作量，所有垃圾清除算法都一样）
	2. 复制阶段的工作量，涉及到对象的复制和引用的更新，与**存活对象的数量**成正比
	3. 简单来说，复制算法的 “清除” 是批量化的，一旦存活对象被复制到新的空间，旧空间中的所有对象（包括大量垃圾）就被直接整块废弃，无需逐个遍历和回收。
	4. 因此其适用于**垃圾对象占多数**的场景，需要移动的存活对象越少，其开销就越低；当存活对象较多时，复制的开销会显著增加
4. 能够处理循环引用

其缺点是：
1. 内存空间利用率低：
	1. 这是复制算法最大的缺点，它需要将可用的内存空间一分为二，每次只使用一半。
2. 不适用于存活对象占多数的场景
	1. 需要移动的存活对象越少，其开销就越低
	2. 当存活对象较多时，复制的开销会显著增加

---


#### 5.3.3. 标记-整理算法

标记-整理算法是垃圾回收领域中一种重要的算法，它融合了标记-清除算法的优点，同时解决了其所带来的**内存碎片问题**。因此，它常被视为标记-清除算法的升级版，即“标记-清除 plus”。

在标记阶段，标记-整理算法与标记-清除算法一样，**基于可达性分析**，对所有存活对象进行标记。标记完成后，垃圾回收器会遍历整个堆，将所有标记为存活的对象**按顺序移动到堆的一端**（通常是起始地址），在移动过程中同步**更新所有引用**，确保指针始终指向对象的新位置。

当对象移动完成后，堆的另一端（尚未占用的区域）就成为**连续的空闲内存空间**，原位置中未被移动的对象（即不可达对象）也随之被 “清除”。
![](image-20250728182811254.png)

其优点是：
1. 消除内存碎片化
2. 内存利用率高：
	1. 不像复制算法需要牺牲一半的内存空间
	2. 标记-整理算法可以充分利用整个堆空间进行对象分配
3. 能够处理循环引用

其缺点是：
1. 实现复杂：
	1. 相比标记-清除或复制算法，标记-整理算法的实现复杂性更高
2. 回收效率最低：
	1. 标记阶段的工作量，与**存活对象的数量及其引用关系复杂程度**成正比，如果堆中绝大部分是垃圾，那么标记的开销相对较小（这个工作量，所有垃圾清除算法都一样）
	2. 整理阶段的工作量，涉及到对象的复制和引用的更新，与**存活对象的数量**成正比

---


### 5.4. 垃圾回收思想

#### 5.4.1. 分代思想

分代思想的提出基于两个重要的对象生命周期假说：
1. 弱代假说（Weak Generational Hypothesis）：
	1. 绝大多数对象都是朝生夕死的  
2. 强代假说（Strong Generational Hypothesis）： 
	1. 熬过越多次垃圾回收过程的对象就越难以被回收

基于这两个假说，如果每次垃圾回收都针对整个堆，效率会非常低下。因此，分代思想将 Java 堆内存划分为几个不同的区域，每个区域根据其对象的特点（生命周期）采用最适合的 GC 算法，避免了 “大而全” 的低效全堆回收

> [!NOTE] 注意事项：跨代引用



#### 5.4.2. 分区思想

---


#### 5.4.3. 增量收集思想

---







### 5.5. 对象的 finalization 机制

---

































### 5.6. GC 常用 JVM 参数

> [!NOTE] 注意事项
> 1. GC 常用的 JVM 参数有：
> 	1. -XX:±PrintGCDetails

----


## 6. 字节码详解

### 6.1. 符号引用

JVM 在解析它后会把符号引用转换成 直接引用


|类型|常量池项类型|描述|举例|
|---|---|---|---|
|类或接口引用|`CONSTANT_Class_info`|类的完全限定名|`java/lang/String`|
|字段引用|`CONSTANT_Fieldref_info`|类 + 字段名 + 描述符|`Test.instanceBasicTypeValue : I`|
|方法引用|`CONSTANT_Methodref_info`|类 + 方法名 + 描述符|`Object.<init>()V`|
|接口方法引用|`CONSTANT_InterfaceMethodref_info`|接口 + 方法名 + 描述符|`Comparable.compareTo(Object)`|
虽然 `CONSTANT_String_info` 本身不是“符号引用”，但 JVM 在解析它后，会把运行时常量池对应槽位替换成 `"def"` 的 String 对象引用，因此你最终看到的是 Java 对象引用。

| 常量池项类型                       | 是否符号引用类型 | 运行时是否会变成对象引用 | 举例说明                               |
| ---------------------------- | -------- | ------------ | ---------------------------------- |
| `CONSTANT_String_info`       | ❌ 否      | ✅ 会          | 转成 `"abc"` 的 `String` 对象引用         |
| `CONSTANT_Integer_info`      | ❌ 否      | ✅ 会          | 转成 `java.lang.Integer` 对象或直接 int 值 |
| `CONSTANT_Float_info`        | ❌ 否      | ✅ 会          | 类似 float 42.0f，装箱为 Float           |
| `CONSTANT_Long_info`         | ❌ 否      | ✅ 会          | long 类型字面量，值保存在堆中或局部变量表中           |
| `CONSTANT_Double_info`       | ❌ 否      | ✅ 会          | 同上，double 值也装箱                     |
| `CONSTANT_Class_info`        | ✅ 是      | ✅ 会          | 转成 `java.lang.Class` 对象            |
| `CONSTANT_MethodHandle_info` | ✅ 是      | ✅ 会          | 转成 `MethodHandle` 对象               |
| `CONSTANT_Dynamic_info`      | ✅ 是      | ✅ 会          | 通过 bootstrap 方法调用结果填入常量池槽位         |

| 包装类型        | 实际使用的常量池项类型             | 装箱后是否成为对象引用       |
| ----------- | ----------------------- | ----------------- |
| `Integer`   | `CONSTANT_Integer_info` | ✅                 |
| `Float`     | `CONSTANT_Float_info`   | ✅                 |
| `Long`      | `CONSTANT_Long_info`    | ✅                 |
| `Double`    | `CONSTANT_Double_info`  | ✅                 |
| `Boolean`   | ❌（用 iconst 指令代替）        | ✅（有缓存 true/false） |
| `Byte`      | `CONSTANT_Integer_info` | ✅                 |
| `Short`     | `CONSTANT_Integer_info` | ✅                 |
| `Character` | `CONSTANT_Integer_info` | ✅                 |

![](image-20250727223437242.png)


---




### 6.2. iconst_n

把小整数常量（-1 ~ 5）推入栈顶，例如：
1. iconst_m1
	1. 推 -1
2. iconst_0
	1. 推 0
3. iconst_1
	1. 推 1
4. iconst_2
	1. 推 2
5. iconst_3
	1. 推 3
6. iconst_4
	1. 推 4
7. iconst_5
	1. 推 5

---


### 6.3. bipush

把一个 8 位带符号整数常量（byte 类型，-128 ~ 127）推入栈顶，例如：
1. binpush -30
	1. 推 -30

---


### 6.4. sipush

把一个 16 位带符号整数常量（short 类型，-32768 ~ 32767）推入栈顶，例如：
1. sipush 20000
	1. 推 20000

> [!NOTE] 注意事项
> 1. 数值更大，就不是直接推入栈顶了，而是字面量保存在方法区，通过 ldc 进行加载了，就像
```
// 实例变量
public int instanceBasicTypeValue = 32768;


// 字节码
ldc #7 <32768>
putfield #8 <org/example/test/Test.instanceBasicTypeValue : I>
```

---


# JVM 工具

## JConsole





---


## VisualVM

### 软件安装

#### 软件安装

安装地址： https://visualvm.github.io/download.html
![](image-20250730205937863.png)

---


#### 安装插件

![](image-20250730214613670.png)

---


#### 集成 IDEA

<font color="#92d050">1. 下载 VisualVM Launcher 插件</font>
![](image-20250730210053342.png)


<font color="#92d050">2. 进行插件配置</font>
![](image-20250730214937797.png)


<font color="#92d050">3. 使用插件</font>
![](image-20250730214912560.png)

---


### 远程连接

<font color="#92d050">1. 启动远程 Java 程序</font>
这里使用 JMX 进行远程连接，你需要在远程服务器上的 Java 程序加上这些 JVM 参数：
2. -Dcom.sun.management.jmxremote
	1. 启用 JMX 远程管理
3. -Djava.rmi.server.hostname=192.168.136.8
	1. 远程服务器 IP
4. -Dcom.sun.management.jmxremote.port=9999
	1. JMX 监听的端口号
	2. 需要开发防火墙和安全组，确保我们能够正常连接
5. -Dcom.sun.management.jmxremote.ssl=false
	1. 禁用 SSL 加密
	2. 在生产环境中强烈建议启用 SSL 加密
6. -Dcom.sun.management.jmxremote.authenticate=false
	1. 禁用身份验证
	2. 在生产环境中强烈建议启用身份验证


<font color="#92d050">2. 添加远程主机</font>
![](image-20250731092919578.png)


<font color="#92d050">3. 添加 JMX 连接</font>
![](image-20250731093012593.png)

![](image-20250731093109639.png)

---


### 主要功能

#### 生成 / 读取 dump 快照

<font color="#92d050">1. 生成 dump 快照</font>
![](image-20250731093332816.png)


<font color="#92d050">2. 保存 dump 快照</font>
![](image-20250731093412812.png)


<font color="#92d050">3. 读取内存快照</font>
![](image-20250731093617255.png)

---











































































































