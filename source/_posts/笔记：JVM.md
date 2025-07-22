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

## 运行时数据区

### 一览图

![](提高图片清晰度%20(1).png)

----


### 程序计数器（PC 寄存器）

在 CPU 中，寄存器专门用于存储指令执行相关的现场信息，CPU 只有将数据加载到寄存器后才能执行指令。而在 JVM 中，PC 寄存器的全称是 Program Counter Register，中文通常译为 “程序计数器”，它是对物理寄存器的一种抽象。尽管名称相似，但它与物理 CPU 寄存器在功能上并不相同。

根据 JVM 规范，每个线程都拥有独立的、私有的程序计数器，其生命周期与线程一致，用于记录当前线程所执行方法中下一条将要执行的 JVM 字节码指令的地址（偏移量），由执行引擎负责读取指令并执行。如果执行的是本地方法（native），PC 的值为 `undefined`。

PC 寄存器是一块极小的内存区域，几乎可以忽略不计，因为它只需保存一个字节码指令的地址。在 32 位 JVM 中，通常使用 `int` 类型，占 4 个字节；而在 64 位 JVM 中，通常使用 `long` 类型，占 8 个字节。

需要注意的是，程序计数器不存在 GC，也不存在 OOM

![](image-20250722190040146.png)

以如下 Java 代码举例：
```
public class PCExample {
    public static void main(String[] args) {
        int result = add(5, 7);
        if (result > 10) {
            printResult(result);
        } else {
            System.out.println("Too small.");
        }
    }

    public static int add(int a, int b) {
        return a + b;
    }

    public static void printResult(int value) {
        System.out.println("Result is: " + value);
    }
}
```


它的 class 文件如下：
```
// main 方法字节码
 0: bipush        5          // 把 5 压栈
 2: bipush        7          // 把 7 压栈
 4: invokestatic  #2         // 调用 add(int, int)，此时 PC → add 方法入口
 7: istore_1                  // 把结果存入本地变量 result

 8: iload_1                   // 加载 result
 9: bipush       10           // 加载常量 10
11: if_icmple    21           // 若 result <= 10，跳转到 21（else 分支）

14: iload_1
15: invokestatic #3           // 调用 printResult，PC → printResult 方法入口
18: goto          26          // 跳过 else 分支，直接 return

21: getstatic     #4          // System.out
24: ldc           #5          // 加载 "Too small."
26: invokevirtual #6          // 调用 println(String)
29: return


// add 字节码
 0: iload_0       // 加载第一个参数 a
 1: iload_1       // 加载第二个参数 b
 2: iadd          // 相加
 3: ireturn       // 返回结果，PC 返回 main 中调用位置之后


// printResult 字节码
 0: getstatic     #4          // System.out
 3: new           #7          // new StringBuilder
 6: dup
 7: invokespecial #8          // StringBuilder.<init>()

10: ldc           #9          // "Result is: "
12: invokevirtual #10         // StringBuilder.append(String)
15: iload_0
16: invokevirtual #11         // append(int)
19: invokevirtual #12         // toString
22: invokevirtual #6          // println(String)
25: return
```


- [ ] 
> [!NOTE] 注意事项：
> 1. PC 寄存器为什么不存在 GC？
> 	1. 在 JVM 创建线程时，会分配一块连续的内存区域，称为线程执行上下文，用于维护该线程的完整运行状态，包括 PC 寄存器、虚拟机栈、本地方法栈、线程本地存储以及线程状态等
> 	2. 当线程结束时，JVM 和操作系统会释放该线程所有相关的资源
> 2. PC 寄存器为什么不存在 OOM？
> 3. 为什么将 PC 寄存器设置为线程私有？
> 	1. 为了准确记录每个线程当前正在执行的字节码指令地址，最合理的做法是为每个线程分配独立的 PC 寄存器
> 4. 为什么使用 PC 寄存器记录对应线程的执行地址？
> 	1. 由于 CPU 需要频繁切换不同线程，切换回来时必须知道从哪里继续执行，因此通过 PC 寄存器保存线程的执行地址，实现精确恢复
> 5. PC 寄存器中只记录偏移量，但你却说它 “用于记录当前线程所执行方法中下一条将要执行的 JVM 字节码指令的地址（偏移量）”，那它是如何确保这个偏移量一定对应的是 “当前方法” 的指令？
> 	1. 程序计数器只记录字节码数组中的偏移量，但它会配合 JVM 虚拟机栈中当前线程的栈顶栈帧，因为每个方法对应一个栈帧，最顶层的栈帧代表当前正在执行的方法。
















