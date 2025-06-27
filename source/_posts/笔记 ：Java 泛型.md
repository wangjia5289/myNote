---
title: 笔记 ：Java 泛型
date: 2025-05-06
categories:
  - Java
  - Java 基础
  - Java 泛型
tags: 
author: 霸天
layout: post
---
### 1. 四大函数对比表

| 接口              | 方法签名               | 接收参数 | 返回值 | 可抛异常 | 用途描述               |
| --------------- | ------------------ | ---- | --- | ---- | ------------------ |
| `Runnable`      | `void run()`       | ❌ 否  | ❌ 否 | ❌ 否  | 执行一个任务，不需要输入，也不返回  |
| `Callable<R>`   | `T call()`         | ❌ 否  | ✅ 是 | ✅ 是  | 执行一个任务，返回结果 R，可抛异常 |
| `Consumer<T>`   | `void accept(T t)` | ✅ 是  | ❌ 否 | ❌ 否  | 消费一个输入值 T，不返回结果    |
| `Function<T,R>` | `R apply(T t)`     | ✅ 是  | ✅ 是 | ❌ 否  | 把输入 T 转换成输出 R      |

---


### 常见泛型

1. `mono.map(Function<T, R> mapper)`：
	1. `Function`：
		1. Java 中的函数式接口（能接受参数，有返回值，不可抛异常的任务）
	2. `T`：
		1. 输入泛型占位符，代表任意一种输入类型。
	3. `R`：
		1. 输出泛型占位符，代表任意一种返回类型。
	4. `mapper`：
		1. 函数名，表示你传入的具体转换函数。
	5. `Function<T, R> mapper`：
		1. 名为 mapper 的函数接口，输入 T，返回 R。

---


2. `Mono.just(T data)`：
	1. `data`：
		1. 变量名，表示要包装进 Mono 的那个具体值。
	2. `T data`：
		1. 表示一个 T 类型的输入值

---


3. `mono.cast(Class<R> type)`
	1. `Class`：
		1. Java 关键字，用于获取某个类的 `Class` 对象。
	2. `type`：
		1. 变量名，表示目标类型的 `Class` 实例。
	3. `Class<R> type`：
		1. 表示 R 类型的运行时类型信息

---


4. `Mono.fromCallable(Callable<R>)` 
	1. `Callable`：
		1. Java 中的函数式接口（不能接受参数，无返回值，不可抛异常的任务）
	2. `Callable<R>`：
		1. 表示一个执行后返回 R 类型结果的任务
	3. 与 `Runnable` 相对，该接口表示不能抛异常、不返回结果的任务

---

5. `mono.flatMap(Function<T, Mono<R>> mapper)`
	1. `Function<T, Mono<R>>`：
		1. 输入 T，返回 `Mono<R>` 的函数。

---


6. `mono.switchIfEmpty(Mono<? extends T> fallback)`
	1. `? extends T`：
		1. T 类型或 T 的子类型

---

7. `mono.onErrorResume(Function<Throwable, Mono<? extends T>> fallback)`
	1. `Throwable`：
		1. `Throwable` 是 Java 中的所有异常和错误的根类，它分为两大类：
			1. **`Error`**：通常用于表示 JVM 内部的严重错误，如 `OutOfMemoryError`，这些一般不被捕获和处理。
			2. **`Exception`**：表示程序中可以捕获和处理的异常。包括受检异常（如 `IOException`）和运行时异常（如 `NullPointerException`）。
		2. 你可以把 `Throwable` 看作一个包含所有错误和异常的容器。

---

8. `map(Function<? super T, ? extends R> mapper)`
	1. `? super T`
		1. 表示「T 类型或 T 的**父类型**」。

