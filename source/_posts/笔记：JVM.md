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

## 1. 运行时数据区

### 1.1. 运行时数据区一览图

![](image-20250724223952607.png)

从线程是否共享的角度又可以为：
![](image-20250726201657760.png)

----


### 1.2. 程序计数器（PC 寄存器）

在 CPU 中，寄存器专门用于存储指令执行相关的现场信息，CPU 只有将数据加载到寄存器后才能执行指令。而在 JVM 中，PC 寄存器的全称是 Program Counter Register，中文通常译为 “程序计数器”，它是对物理寄存器的一种抽象。尽管名称相似，但它与物理 CPU 寄存器在功能上并不相同。

根据 JVM 规范，每个线程都拥有独立的、私有的程序计数器，其生命周期与线程一致，用于记录当前线程所执行方法中下一条将要执行的 JVM 字节码指令的地址（偏移量），由执行引擎负责读取指令并执行。如果执行的是本地方法（native），PC 的值为 `undefined`。

PC 寄存器是一块极小的内存区域，几乎可以忽略不计，因为它只需保存一个字节码指令的地址。在 32 位 JVM 中，通常使用 `int` 类型，占 4 个字节；而在 64 位 JVM 中，通常使用 `long` 类型，占 8 个字节。

![](image-20250722190040146.png)

- [ ] 
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


### 1.3. 虚拟机栈

#### 1.3.1. 虚拟机栈一览图

![](图像清晰化.png)

----


#### 1.3.2. 虚拟机栈概述


> [!NOTE] 注意事项
> 1. 虚拟机栈不会发生 GC，但会抛出 OOM 或 StackOverflowError，常见有：
> 	1. OOM
> 		1. OutOfMemoryError: unable to create new native thread
> 			1. JVM 尝试创建新线程时，系统无法分配栈内存
> 	2. StackOverflowError
> 		1. 栈帧太多（比如递归没结束），导致栈空间“满了”


----




#### 1.3.3. 局部变量表

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


#### 1.3.4. 操作数栈

##### 1.3.4.1. 操作数栈概述

操作数栈是 JVM 执行引擎的一个工作区，会根据方法中的字节码指令，向栈中写入数据或提取数据，这些操作被称为入栈和出栈。它主要用于保存计算过程中的中间结果，同时作为变量的临时存储空间。

我们常说 Java 虚拟机的解释引擎是基于栈的执行引擎，这里的 “栈” 指的就是操作数栈。

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


##### 1.3.4.2. 栈顶缓存技术

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


#### 1.3.5. 动态链接

---

#### 1.3.6. 虚拟机栈异常示例

---


### 1.4. 本地方法栈




---


### 1.5. 堆区

#### 1.5.1. 堆区一览图

![](image-20250725114229098.png)

----


#### 1.5.2. 堆区概述

在一个 JVM 实例中，只存在一块堆内存，堆是 Java 内存管理的核心区域，也是整个内存结构中最大的一块，几乎所有的对象实例和数组，在运行时都会被分配到堆上（需考虑逃逸分析的优化可能），Java 堆是所有线程共享的（但需考虑线程私有的 TLAB 缓冲区机制）。



有意思：数组和对象可能永远不会存储在栈上，因为栈帧中保存引用，这个引用指向对象或者数组在堆中的位置。


> [!NOTE] 注意事项
> 1. 堆区既会发生 GC，也会抛出 OOM，常见有：
> 	1. OutOfMemoryError: Java heap space
> 		1. 堆内存空间不足，无法为新对象分配内存
> 	2. OutOfMemoryError: GC overhead limit exceeded
> 		1. 垃圾回收耗时过长且回收效果极差，JVM 为避免长时间卡顿而主动抛出异常
> 	3. OutOfMemoryError: Requested array size exceeds VM limit
> 		1. 尝试分配超出虚拟机限制的超大数组，导致分配失败
> 2. 为什么要在堆区分代？
> 	1. IBM 公司的专门研究表明，新生代中 80% 的对象都是 “朝生夕死” 的
> 3. 虽然堆在物理上可以是不连续的内存空间，但从逻辑上应视为一个连续的整体。
> 4. Java 堆区在 JVM 启动时被创建，其空间大小也在此时初始化。但不同于局部变量表和操作数栈，堆区的大小可以通过 JVM 参数手动指定，包括初始空间和最大空间，且在运行时具有动态扩展能力：当前堆空间不足，但尚未达到最大限制时，JVM 会尝试扩大堆空间，同时通常也会触发一次垃圾回收（GC），GC 部分详见下文：垃圾回收（GC）
> 	1. -Xms：用于设置堆区的初始内存大小。例如：java -Xms512m YourApp
> 		1. 常用单位有：k、m、g
> 	2. -Xmx：用于设置堆区的最大内存大小。例如：java -Xmx2048m YourApp
> 	3. 实际使用中，通常将 -Xms 和 -Xmx 设置为相同的值，目的是为了避免 JVM 在运行期间，频繁触发 堆空间扩容与内存结构重计算以及其触发的GC
> 		1. 堆空间扩容与内存结构重计算
> 			1. 当 GC 之后依然无法满足内存申请，JVM 会尝试扩展堆内存至更高值，同时可能重新计算内存布局（如 Eden/Survivor 的比例划分），这是一种昂贵的操作
> 		2. 垃圾回收（GC）
> 			1. 在堆空间不足时触发，试图清理无用对象以释放内存。
> 			2. 详见下文：垃圾回收（GC）
> 	4. 默认情况下（以 HotSpot JVM 为例）
> 		1. 初始堆内存大小（-Xms）默认为物理内存的 1/64；
> 		2. 最大堆内存大小（-Xmx）默认为物理内存的 1/4；
> 5. 堆区常用的 JVM 参数有：
> 	1. -Xms
> 		1. 用于设置堆区的初始内存大小。
> 		2. 例如：java -Xms512m YourApp
> 		3. 常用单位有：k、m、g
> 	2. -Xmx
> 		1. 用于设置堆区的最大内存大小。
> 		2. 例如：java -Xmx2048m YourApp
> 		3. 实际使用中，通常将 -Xms 和 -Xmx 设置为相同的值
> 	3. -XX:NewRatio=2
> 		1. 用于配置新生代与老年代在堆区的占比。
> 		2. 例如：java -XX:NewRatio=2 MyApp，表示新生代占堆区的 1/3，老年代占堆区的 2/3
> 	4. -XX:SurvivorRatio=8
> 		1. 用于配置 Eden 区与 Survivor 区在新生代的占比。
> 		2. 例如：java -Xmn100m -XX:SurvivorRatio=8 MyApp，表示 Eden : Survivor0 : Survivor1 = 8 : 1 : 1
> 	5. -XX:MaxTenuringThreshold=15
> 		1. 用于配置对象存活年龄
> 			1. 例如：java -XX:MaxTenuringThreshold=15。
> 	6. -XX:±UseTLAB
> 		1. 用于配置是否开启 TLAB
> 		2. 例如：java -XX:+UseTLAB -XX:+PrintTLAB MyApp
> 	7. -XX:±PrintTLAB
> 		1. 用于打印 TLAB 分配和使用情况
> 		2. 例如：java -XX:+UseTLAB -XX:+PrintTLAB MyApp
> 	8. -XX:TLABWasteTargetPercent=50




#### 1.5.3. 堆区异常示例

----


### 1.6. 方法区

#### 方法区一览图

---


#### 方法区概述



下面是方法区（Metaspace）中存储的**各种类型信息的详细分类**，每一项都是 JVM 加载类时产生并保存在方法区中的元数据。

|区域/结构|存储内容说明|示例/说明|
|---|---|---|
|**1. 类结构信息**|每个类或接口的定义信息|类的名称、访问标志、父类、接口、字段、方法等|
|├─ 类的全限定名|如 `java/lang/String`||
|├─ 类加载器引用|负责加载该类的 ClassLoader 对象引用|用于类的隔离和卸载|
|├─ 父类引用|超类元数据的引用|支持继承关系|
|├─ 接口列表|当前类实现的所有接口|存储为接口元数据的引用|
|├─ 访问标志（修饰符）|`public`、`abstract`、`final`、`interface` 等|决定类或接口的属性|
|├─ 源文件名|用于调试信息：来自 `SourceFile` 常量|如 `MyClass.java`|

---

|区域/结构|存储内容说明|示例/说明|
|---|---|---|
|**2. 字段信息**|每个字段的结构定义（不含实际值）|类型、名称、访问标志、偏移量等|
|├─ 名称、描述符|如 `name : Ljava/lang/String;`|描述字段类型|
|├─ 访问修饰符|`private`、`static`、`final` 等|编译器生成的字段限制|
|├─ 静态字段默认值|例如 `static final int VALUE = 10` 的常量|编译期已知，存入常量池，运行时初始化|
|├─ 字段偏移量|用于在实例对象中快速定位字段|优化字段访问|

---

|区域/结构|存储内容说明|示例/说明|
|---|---|---|
|**3. 方法信息**|方法签名、修饰符、异常表、字节码指针、栈帧大小等|每个方法都作为一个独立结构存在|
|├─ 方法签名|`main([Ljava/lang/String;)V`|方法名+参数+返回值组合|
|├─ 访问修饰符|`public static`、`private final` 等||
|├─ 异常表|`throws Exception`|声明性异常信息|
|├─ Code属性指针|指向实际的字节码段|存放方法体实际执行内容|
|├─ 栈帧需求（最大栈深、局部变量表大小）|虚拟机执行该方法所需资源|用于栈帧构建|

---

|区域/结构|存储内容说明|示例/说明|
|---|---|---|
|**4. 常量池（Runtime Constant Pool）**|字面量和符号引用常量池（每个类一个）|含字符串字面量、类引用、方法引用、字段引用等|
|├─ 字符串字面量引用|`"abc"`、`"hello"`|实际字符串对象可能位于字符串常量池|
|├─ 符号引用|对类、方法、字段的符号表示|运行时解析为直接引用|
|├─ 常量值（如 int、float）|编译器产生的值|`ldc` 指令加载使用|

---

|区域/结构|存储内容说明|示例/说明|
|---|---|---|
|**5. 方法句柄/动态调用点**|invokedynamic 所需的 BootstrapMethods 表，CallSite 信息等|支持 Lambda、String Concat、动态语言|
|├─ BootstrapMethods|指向用于动态链接的引导方法|比如 `StringConcatFactory`|
|├─ 动态常量池项（JEP 309）|`CONSTANT_Dynamic_info`|支持延迟解析常量|

---

|区域/结构|存储内容说明|示例/说明|
|---|---|---|
|**6. 注解元数据**|类/方法/字段上的注解信息|`@Override`, `@MyCustomAnnotation`|
|├─ RuntimeVisibleAnnotations|编译后仍保留在运行时的注解|用于反射等功能|
|├─ RuntimeInvisibleAnnotations|编译器处理但运行时不可见的注解|编译提示、框架配置|

---

|区域/结构|存储内容说明|示例/说明|
|---|---|---|
|**7. 调试信息**|行号表、局部变量表、源文件、参数名等|`LineNumberTable`, `LocalVariableTable`|
|├─ 行号表（LineNumberTable）|字节码 → 源代码行映射|支持调试|
|├─ 局部变量表|每个局部变量在局部变量表中的位置、作用范围|IDE 调试时显示变量名|

---


#### 常量池




















```
public class Test {

    public static Object instance = new Object();
    public static final int num = 1;
    
    public int compute(int a,int b){
        return a * b - num;
    }
    
    public static void main(String[] args) {
        Test test = new Test();
        int a = 4;
        int b = 5;
        int c = test.compute(a,b);
        System.out.println(c);
    }
}
```


```
PS D:\文件集合\summer\JUC\src\main\java\org\example\test> javap -l -c -v Test.class
Classfile /D:/文件集合/summer/JUC/src/main/java/org/example/test/Test.class
  Last modified 2025年7月26日; size 666 bytes
  SHA-256 checksum 0e59b8b62b24b374efb47f1d8cf4c65013ac16b888c0c3e89bad4957c9266e31
  Compiled from "Test.java"
public class org.example.test.Test
  minor version: 0
  major version: 61
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #7                          // org/example/test/Test
  super_class: #2                         // java/lang/Object
  interfaces: 0, fields: 2, methods: 4, attributes: 1
Constant pool:
   #1 = Methodref          #2.#3          // java/lang/Object."<init>":()V
   #2 = Class              #4             // java/lang/Object
   #3 = NameAndType        #5:#6          // "<init>":()V
   #4 = Utf8               java/lang/Object
   #5 = Utf8               <init>
   #6 = Utf8               ()V
   #7 = Class              #8             // org/example/test/Test
   #8 = Utf8               org/example/test/Test
   #9 = Methodref          #7.#3          // org/example/test/Test."<init>":()V
  #10 = Methodref          #7.#11         // org/example/test/Test.compute:(II)I
  #11 = NameAndType        #12:#13        // compute:(II)I
  #12 = Utf8               compute
  #13 = Utf8               (II)I
  #14 = Fieldref           #15.#16        // java/lang/System.out:Ljava/io/PrintStream;
  #15 = Class              #17            // java/lang/System
  #16 = NameAndType        #18:#19        // out:Ljava/io/PrintStream;
  #17 = Utf8               java/lang/System
  #18 = Utf8               out
  #19 = Utf8               Ljava/io/PrintStream;
  #20 = Methodref          #21.#22        // java/io/PrintStream.println:(I)V
  #21 = Class              #23            // java/io/PrintStream
  #22 = NameAndType        #24:#25        // println:(I)V
  #23 = Utf8               java/io/PrintStream
  #24 = Utf8               println
  #25 = Utf8               (I)V
  #26 = Fieldref           #7.#27         // org/example/test/Test.instance:Ljava/lang/Object;
  #27 = NameAndType        #28:#29        // instance:Ljava/lang/Object;
  #28 = Utf8               instance
  #29 = Utf8               Ljava/lang/Object;
  #30 = Utf8               num
  #31 = Utf8               I
  #32 = Utf8               ConstantValue
  #33 = Integer            1
  #34 = Utf8               Code
  #35 = Utf8               LineNumberTable
  #36 = Utf8               main
  #37 = Utf8               ([Ljava/lang/String;)V
  #38 = Utf8               <clinit>
  #39 = Utf8               SourceFile
  #40 = Utf8               Test.java
{
  public static java.lang.Object instance;
    descriptor: Ljava/lang/Object;
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC

  public static final int num;
    descriptor: I
    flags: (0x0019) ACC_PUBLIC, ACC_STATIC, ACC_FINAL
    ConstantValue: int 1

  public org.example.test.Test();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 2: 0

  public int compute(int, int);
    descriptor: (II)I
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=3
         0: iload_1
         1: iload_2
         2: imul
         3: iconst_1
         4: isub
         5: ireturn
      LineNumberTable:
        line 7: 0

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=3, locals=5, args_size=1
         0: new           #7                  // class org/example/test/Test
         3: dup
         4: invokespecial #9                  // Method "<init>":()V
         7: astore_1
         8: iconst_4
         9: istore_2
        10: iconst_5
        11: istore_3
        12: aload_1
        13: iload_2
        14: iload_3
        15: invokevirtual #10                 // Method compute:(II)I
        18: istore        4
        20: getstatic     #14                 // Field java/lang/System.out:Ljava/io/PrintStream;
        23: iload         4
        10: return
      LineNumberTable:
        line 3: 0
}
SourceFile: "Test.java"
```




---




















方法区用于存储已被虚拟机加载的 类型信息、域信息、方法信息、

<font color="#92d050">1. 类型信息</font>

|  信息内容   | 说明  | 注意事项 |
| :-----: | :-: | :--: |
|  类的名称   |     |      |
| 类加载器的引用 |     |      |
|         |     |      |





```
public class Example {

    // 1. 静态常量：属于类，编译期常量，可能会放入常量池
    public static final String CONST = "Hello, JVM";

    // 2. 静态变量：属于类，变量的值存储在堆中
    public static int staticCount = 42;

    // 3. 实例变量：属于对象，完全存储在堆中
    private int instanceValue = 100;

    // 4. 实例方法：属于类结构的一部分，方法信息存储在方法区
    public void printInstanceValue() {
        System.out.println("Instance value: " + instanceValue);
    }

    // 5. 静态方法：方法结构存在方法区，不属于某个对象
    public static void printStatic() {
        System.out.println("Static count: " + staticCount);
    }

    public static void main(String[] args) {
        // 类加载时，类的结构被加载到方法区
        Example obj = new Example(); // 对象在堆中分配
        obj.printInstanceValue();    // 调用实例方法
        Example.printStatic();       // 调用静态方法
        System.out.println(CONST);   // 访问静态常量
    }
}

```





永久代、元空间

使用本地内存，就更不容易出现 OOM

JVM 的虚拟内存难道不是使用的本地内存嘛？


> [!NOTE] 注意事项
> 1. 方法区既会发生 GC，也会抛出 OOM，常见有：
> 	1. java.lang.OutOfMemoryError: Metaspace
> 		1. 
> 	2. java.lang.OutOfMemoryError: PermGen space
> 		1. 
> 2. JDK1.7 之前的永久代、JDK1.8 之后的元空间，都是对 JVM 规范中方法区的实现，其最大的区别在于：元空间不再使用 JVM 虚拟机设置的堆内存，而是改为使用本地内存，因此更少发生 OOM异常。
> 	1. 永久代（PermGen） 是 JVM 内存的一部分，大小由 `-XX:PermSize` 和 `-XX:MaxPermSize` 限制，默认值往往较小（比如几十 MB），而且很容易超过上限，从而导致 OOM
> 	2. 元空间（Metaspace） 使用的是 本地内存（Native memory），大小可以远远超过 JVM 堆的限制，甚至可以动态扩展至整个物理内存的上限
> 3. 方法区在 JVM 启动时被创建，其实际的物理内存可以是不连续的内存空间
> 4. 方法区的大小可以通过 JVM 参数手动指定，包括初始空间和最大空间，且在运行时具有动态扩展能力
> 	1. JDK1.7 之前
> 		1. -XX:PermSize=64m
> 			1. 用于设置方法区的初始内存大小
> 			2. 例如：java -XX:PermSize=64m -XX:MaxPermSize=128m -jar MyApp.jar
> 			3. 默认值：20.75m
> 			4. 常用单位：k、m、g
> 		2. -XX:MaxPermSize=128m
> 			1. 用于设置方法区的最大内存大小
> 			2. 例如：java -XX:PermSize=64m -XX:MaxPermSize=128m -jar MyApp.jar
> 			3. 默认值：32 位机器默认是 64 M，64 位机器默认是 82 M
> 	2. JDK1.8 之前
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
> 5. 方法区的 GC 过程：
> 	1. 当方法区（Metaspace 或 PermGen）的使用量，触及初始高水位线（即默认的 -XX:MetaspaceSize）时，JVM 会尝试触发一次 Full GC（不会触发 Minor GC）
> 	2. Full GC 会对新生代、老年代、方法区进行垃圾回收，其中在方法区中会卸载那些对应类加载器不再存活的类
> 	3. 触发 GC 后，高水位线会被动态调整，新的高水位线取决于 GC 后释放了多少空间：
> 		1. 如果释放的空间不足，在不超过 MaxMetaspaceSize 限制时，会适当提高高水位线
> 		2. 如果释放的空间过多，则会适当降低高水位线
> 	4. 如果 Full GC 后空间依然不足，且超过了 MaxMetaspaceSize 限制，则 JVM 会抛出 OOM 异常
> 	5. 如果初始高水位线（-XX:MetaspaceSize）设置过低，可能会导致 Full GC 频繁触发，建议将其设置为一个相对较高的值以减少不必要的 GC 开销。

---


















## 2. 垃圾回收（GC）

### 2.1. 垃圾回收概述

垃圾回收是指在程序运行过程中，对堆区中没有任何指针指向的对象进行回收，对在方法区中卸载那些其对应类加载器已不再存活的类。

其中，堆区是垃圾收集器的主要工作区域，从回收频率上看，通常遵循以下规律：
1. 频繁收集新生代
2. 较少收集老年代
3. 基本不懂永久代（方法区、元空间）

---


### 垃圾标记算法

#### 垃圾标记概述

在堆中存放着几乎所有的 Java 对象实例，在 GC 执行垃圾回收之前，首先需要区分内存中哪些是存活的对象，哪些是已经死亡的对象。

只有被标记为死亡的对象，GC 在执行回收时才会释放它们所占用的内存空间，因此，这一过程被称为垃圾标记阶段。

---


#### 引用计数法

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


#### 可达性分析算法（根搜索方法、追踪性垃圾收集）

##### 可达性分析算法概述

可达性分析算法是以根对象集合（GC Roots，一组始终处于活跃状态的引用）为起始点，自上而下地搜索与这些根对象相关联的目标对象是否可达，内存中的所有存活对象，都会被根对象集合直接或间接连接。

搜索过程中所经过的路径被称为引用链（Reference Chain），如果某个目标对象在内存中没有任何引用链与 GC Roots 相连，那么它就是不可达的，也就意味着该对象已死亡，可被标记为垃圾对象。

> [!NOTE] 注意事项
> 1. 如果要使用可达性分析算法判断内存是否可回收，分析工作必须在一个能够**保证内存状态一致性的快照中进行**。
> 2. 如果无法满足这一点，分析结果的准确性就无法保证，这也是导致 GC 进行时必须执行 “Stop The World” 的一个重要原因。
> 3. 即便是号称几乎不会停顿的 CMS 收集器，在枚举根节点阶段，也必须暂停所有应用线程（主线程、工作线程、守护线程）

---


##### 常见 GC Roots

---


##### 查看 GC Roots

---


### 垃圾清除算法

---


### 对象的 finalization 机制

---

































### 2.2. GC 常用 JVM 参数

> [!NOTE] 注意事项
> 1. GC 常用的 JVM 参数有：
> 	1. -XX:±PrintGCDetails

----







## 3. 对象实例化流程

### 实例

![](image-20250727210546488.png)







> [!NOTE] 注意事项
>5. Java 类里各种变量的 “诞生” 与 “存活” 流程
>	1. 在下面的例子中，变量会经历以下几个阶段：
>		1. 类加载阶段
>			1. 准备阶段
>				1. JVM 将 `Demo.class` 文件加载到内存，并为静态变量 `staticVar` 分配内存，并赋予默认值 0
>				2. 该静态变量存储在方法区（元空间、Metaspace）（坏事了，jdk1.7 之前确实是方法区，jdk 1.8 之后是放到堆内存，直接放到老年代，那准备阶段对象都还没创建呢，啊对，静态变量与对象根本没关系，而是而是属于类本身的，不用等对象）
>				3. 准备阶段不执行任何代码，也不调用任何方法，只是为静态变量提供一个 “安全” 的初始状态，避免野指针或未定义行为。
>			2. 初始化阶段
>				1. 执行静态变量的显式赋值语句（如 `staticVar = 10;`）和静态代码块（`static {}`）中的代码。
>		2. 对象创建阶段
>			1. JVM 在堆内存中为对象 `demo` 分配空间，所有成员变量（如 `instanceVar`）都被分配内存并赋予默认值（0、false、null）
>			2. 然后执行赋值？、、、显构造方法，可以对成员变量进行显式初始化或逻辑处理。
>			3. 简单来说就是属性的默认初始化、显示初始化、构造器初始化
>		3. 方法调用阶段
>			1. JVM 在栈内存中为方法调用创建一个新的栈帧
>			2. 因为是实例方法，槽位 0 用于存放 `this` 引用，指向当前调用方法的 `demo` 对象。
>			3. 槽位 1 用于存放方法参数 `param`，槽位 2 存放局部变量 `localVar`。
>			4. 需要注意的是，局部变量和方法参数不会像静态变量或成员变量那样自动初始化默认值。
>			5. 如果你只是声明了一个局部变量（如 `int localVar;`），但没有显式赋值（如 `localVar = 5`），那么就会编译报错。
```
public class Demo {

    // 静态变量（类变量）
    static int staticVar = 10;

    // 成员变量（实例变量）
    int instanceVar;
    
	// 方法参数
    public void method(int param) {
    
	    // 局部变量
        int localVar = 5;
        
        System.out.println(localVar);
    }
}
```

---




### 代码大全

```
public class Test {

    // 1. 实例基本数据类型变量
    public int instanceBasicTypeValue = 5;

    // 2. 实例字符串类型变量 1
    public String instanceStringTypeValue1 = "abc";

    // 3. 实例字符串类型变量 2
    public String instanceStringTypeValue2 = new String("def");

    // 4. 实例字符串类型变量 3
    public String instanceStringTypeValue3 = instanceStringTypeValue2.intern();

    // 5. 实例引用类型变量
    public Demo instanceReferenceTypeValue = new Demo();

    // 6. 静态基本数据类型变量
    public static int staticBasicTypeValue = 10;

    // 7. 静态字符串类型变量 1
    public static String staticStringTypeValue1 = "abc";

    // 8. 静态字符串类型变量 2
    public static String staticStringTypeValue2 = new String("def");

    // 9. 静态字符串类型变量 3
    public static String staticStringTypeValue3 = staticStringTypeValue2.intern();

    // 10. 静态引用类型变量
    public static Demo staticReferenceTypeValue = new Demo();

    // 11. 基本数据类型常量
    public final int finalBasicTypeValue = 15;

    // 12. 字符串类型常量 1
    public final String finalStringTypeValue1 = "abc";

    // 13. 字符串类型常量 2
    public final String finalStringTypeValue2 = new String("def");

    // 14. 字符串类型常量 3
    public final String finalStringTypeValue3 = finalStringTypeValue2.intern();

    // 15. 引用类型常量
    public final Demo finalReferenceTypeValue = new Demo();

    // 16. 静态基本数据类型常量
    public static final int staticFinalBasicTypeValue = 20;

    // 17. 静态字符串类型常量 1
    public static final String staticFinalStringTypeValue1 = "abc";

    // 18. 静态字符串类型常量 2
    public static final String staticFinalStringTypeValue2 = new String("def");

    // 19. 静态字符串类型常量 3
    public static final String staticFinalStringTypeValue3 = staticFinalStringTypeValue2.intern();

    // 20. 静态引用类型常量
    public static final Demo staticFinalReferenceTypeValue = new Demo();

    // 方法参数
    public void method(int param) {
        // 局部变量
        int localValue = 25;
        System.out.println(localValue);
    }

    public static void main(String[] args) {
        Test test = new Test();

        System.out.println("instanceBasicTypeValue = " + test.instanceBasicTypeValue); // 5
        System.out.println("instanceStringTypeValue1 = " + test.instanceStringTypeValue1); // abc
        System.out.println("instanceStringTypeValue2 = " + test.instanceStringTypeValue2); // def
        System.out.println("instanceStringTypeValue3 = " + test.instanceStringTypeValue3); // def
        System.out.println("instanceReferenceTypeValue = " + test.instanceReferenceTypeValue); // org.example.test.Demo@7699a589

        System.out.println("staticBasicTypeValue = " + Test.staticBasicTypeValue); // 10
        System.out.println("staticStringTypeValue1 = " + Test.staticStringTypeValue1); // abc
        System.out.println("staticStringTypeValue2 = " + Test.staticStringTypeValue2); // def
        System.out.println("staticStringTypeValue3 = " + Test.staticStringTypeValue3); // def
        System.out.println("staticReferenceTypeValue = " + Test.staticReferenceTypeValue); // org.example.test.Demo@58372a00

        System.out.println("finalBasicTypeValue = " + test.finalBasicTypeValue); // 15
        System.out.println("finalStringTypeValue1 = " + test.finalStringTypeValue1); // abc
        System.out.println("finalStringTypeValue2 = " + test.finalStringTypeValue2); // def
        System.out.println("finalStringTypeValue3 = " + test.finalStringTypeValue3); // def
        System.out.println("finalReferenceTypeValue = " + test.finalReferenceTypeValue); // org.example.test.Demo@4dd8dc3

        System.out.println("staticFinalBasicTypeValue = " + Test.staticFinalBasicTypeValue); // 20
        System.out.println("staticFinalStringTypeValue1 = " + Test.staticFinalStringTypeValue1); // abc
        System.out.println("staticFinalStringTypeValue2 = " + Test.staticFinalStringTypeValue2); // def
        System.out.println("staticFinalStringTypeValue3 = " + Test.staticFinalStringTypeValue3); // def
        System.out.println("staticFinalReferenceTypeValue = " + Test.staticFinalReferenceTypeValue); // org.example.test.Demo@6d03e736
    }
}
```


### 实例变量

以如下代码为例：
```
public class Test {
		
    // 1. 实例基本数据类型变量
    public int instanceBasicTypeValue = 5000;
		
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

在 `Main` 线程的 `main` 方法中执行 `Test test = new Test();` 时，`new Test()` 这一操作对应的字节码是：
```
 0 aload_0
 1 invokespecial #1 <java/lang/Object.<init> : ()V>
 4 aload_0
 5 ldc #7 <50000>
 7 putfield #8 <org/example/test/Test.instanceBasicTypeValue : I>
10 aload_0
11 ldc #14 <abc>
13 putfield #16 <org/example/test/Test.instanceStringTypeValue1 : Ljava/lang/String;>
16 aload_0
17 new #20 <java/lang/String>
20 dup
21 ldc #22 <def>
23 invokespecial #24 <java/lang/String.<init> : (Ljava/lang/String;)V>
26 putfield #27 <org/example/test/Test.instanceStringTypeValue2 : Ljava/lang/String;>
29 aload_0
30 aload_0
31 getfield #27 <org/example/test/Test.instanceStringTypeValue2 : Ljava/lang/String;>
34 invokevirtual #30 <java/lang/String.intern : ()Ljava/lang/String;>
37 putfield #34 <org/example/test/Test.instanceStringTypeValue3 : Ljava/lang/String;>
40 aload_0
41 new #37 <org/example/test/Demo>
44 dup
45 invokespecial #39 <org/example/test/Demo.<init> : ()V>
48 putfield #40 <org/example/test/Test.instanceReferenceTypeValue : Lorg/example/test/Demo;>
51 return
```

其详细流程为：
![](image-20250727224126808.png)

![](image-20250727224146335.png)

字节码详细解析为：
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
 * - 从运行时常量池中加载整数常量 5000，并将其压入操作数栈
 *
 * putfield #8
 * - 将栈顶的 5000 赋值给当前对象（this）的 instanceBasicTypeValue 字段
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
 * - 从运行时常量池加载常量池中的 "abc" String 对象的引用并压入操作数栈
 *
 * putfield #16
 * - 将栈顶的 "abc" String 对象的引用赋值给当前对象（this）的 instanceBasicTypeValue1 字段
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
 * - 在堆区分配一个 String 对象的空间（此时仅分配内存，尚未初始化，类型由方法区的元数据 #20 指定为 String 类型）
 * - JVM 的一个基本原则是：在创建对象时，必须明确知道对象的大小，以便在堆中分配合适的内存块。这个大小信息在类的元数据中就已经确定了。
 * - 你可能会好奇：如果对象中有一个 List 属性，那在对象创建后我不断向这个 List 添加元素，JVM 怎么知道我最终需要多少内存？
 * - 别忘了，我们在对象中保存的只是对 List 实例的引用，这个引用的大小是固定不变的。那你又好奇了：为什么 List 能动态扩容？
 * - 其实本质是：当容量不足时，会新建一个更大的数组对象，把原有内容复制进去，然后让 elementData 字段指向新的数组。
 *
 * dup
 * - 复制该 String 对象的引用以备后续初始化。
 *
 * ldc #22
 * - 从运行时常量池加载 "def" String 对象的引用并压入操作数栈
 *
 * invokespecial #24
 * - 调用 String 构造方法，在堆内存中创建新的 String 实例。
 *
 * putfield #27
 * - 将栈顶的 新创建的 String 实例的引用 赋值给当前对象（this）的 instanceBasicTypeValue2 字段
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
 * - 在堆区分配一个 Demo 对象的空间（此时仅分配内存，尚未初始化，类型由方法区的元数据 #37 指定为 Demo 类）
 *
 * dup
 * - 复制该 Demo 对象的引用以备后续初始化。
*
 * invokespecial #39
 * - 调用 Demo 构造方法，在堆内存中创建新的 Demo 实例。
 *
 * putfield #40
 * - 将栈顶的 新创建的 Demo 实例的引用 赋值给当前对象（this）的 instanceReferenceTypeValue 字段
 * ============================================
 */
40 aload_0
41 new #37 <org/example/test/Demo>
44 dup
45 invokespecial #39 <org/example/test/Demo.<init> : ()V>
48 putfield #40 <org/example/test/Test.instanceReferenceTypeValue : Lorg/example/test/Demo;>

51 return
```

最终执行结果为：
![](image-20250727163945675.png)

> [!NOTE] 注意事项
> 1. intern() 方法的作用是，返回字符串常量池中 “内容相同” 的 String 对象的引用，如果常量池中没有这个内容，就把当前 String 对象加入常量池，并返回它的引用（不会让当前的 String 对象消失）
> 2. 为什么执行 System.out.println(instanceStringTypeValue) 时，打印出的内容是字符串，而按理说应该像 instanceReferenceTypeValue 那样打印引用地址才对？
> 	1. 这是因为 System.out.println() 先调用对象的 toString() 方法
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


### 静态变量

以如下代码为例：
```
public class Test {

    // 1. 静态基本数据类型变量
    public static int staticBasicTypeValue = 10;

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

        System.out.println("1. staticBasicTypeValue = " + Test.staticBasicTypeValue); // 10
        System.out.println("2. staticStringTypeValue1 = " + Test.staticStringTypeValue1); // abc
        System.out.println("3. staticStringTypeValue2 = " + Test.staticStringTypeValue2); // def
        System.out.println("4. staticStringTypeValue3 = " + Test.staticStringTypeValue3); // def
        System.out.println("5. staticReferenceTypeValue = " + Test.staticReferenceTypeValue); // org.example.test.Demo@58372a00
    }
}
```

在 `Main` 线程的 `main` 方法中执行 `Test test = new Test();` 时，`new Test()` 这一操作对应的字节码是：
```
0 aload_0
1 invokespecial #1 <java/lang/Object.<init> : ()V>
4 return
```

其详细流程为：















## 字节码

### 符号引用

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




### iconst_n

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


### bipush

把一个 8 位带符号整数常量（byte 类型，-128 ~ 127）推入栈顶，例如：
1. binpush -30
	1. 推 -30

---


### sipush

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











































