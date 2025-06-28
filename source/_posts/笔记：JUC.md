---
title: 笔记：JUC
date: 2025-05-18
categories:
  - Java
  - Java 基础
  - Java Util Concurrent（JUC）
tags: 
author: 霸天
layout: post
---




### 1. 线程安全问题

#### 1.1. 线程安全问题概述

在 Spring 中，我们通常会将某个类声明为 Bean，并通过依赖注入的方式在其他地方调用它的方法。也就是说，不管有多少个调用，最终操作的都是这个单例 Bean 的同一个实例。那么现在问题来了，如果我们将下面这段代码声明为 Bean：
```
@Service  
public class CounterService {  
  
    // 成员变量 totalCount
    private int totalCount = 0;  

	// 参数 count
    public int addCount(int count) {  
	    // 局部变量 c
	    int c = 10;
        totalCount += count;  
        return totalCount;  
    }  
}
```


并在其他组件中注入并调用它的方法：
```
@RestController
@RequestMapping("/counter")
public class CounterController {
	
    @Autowired
    private CounterService counterService;

    @GetMapping("/add")
    public int addCount(@RequestParam int value) {
        return counterService.addCount(value);
    }
}
```

在传统的 Tomcat 服务器中，每个 HTTP 请求会由一个独立的线程处理，当这个线程调用 `addCount()` 方法时：
1. 方法参数 `count` 和局部变量 `c` 是线程私有的，分配在各自线程的栈内存中，每个线程都有独立副本，互不干扰，因此天然线程安全
2. 而 `totalCount` 是 Bean 的成员变量，存放在堆内存中，为所有线程所共享，因此存在并发读写的风险。

因此，一旦多个请求同时调用该方法，就会触发**竞态条件**：线程们在没有排队的情况下同时抢着修改 `totalCount`，导致返回的结果不可靠、不确定、不可预测，产生严重的数据错误。

举个例子：A 线程读取到 `totalCount = 10`，打算加 5；B 线程同样读取到 10，打算加 7。它们各自完成加法并写回后，可能的结果包括：A 返回 15，B 返回 17；或者 A 返回 17，B 返回 15；甚至 A 返回 17，B 返回 22。注意，这还只是两个线程的情况，在高并发下，`totalCount` 的最终值将更加不可预测，容易造成严重的数据混乱。

<font color="#ff0000">所以这就是为什么我们的大多数服务和组件都设计为无状态的——也就是说，不在 Bean 中存放可变的成员变量。（不是不放成员变量，而是不放可变的成员变量）</font>

---


#### 1.2. 解决普通服务线程安全问题

为了解决线程安全问题，最理想的做法就是避免在 Bean 中存放可变的成员变量，而是将变量限定为线程私有。但如果业务上确实需要使用成员变量，我们也可以采取以下措施：
==1.使用线程安全的数据结构（推荐）==
这些数据结构内部采用了复杂的机制，保证每次更新都是原子操作，同时避免线程阻塞和锁竞争，性能远超 `synchronized`。
```
@Service
public class CounterService {

    // 使用 AtomicInteger 保证线程安全
    private AtomicInteger totalCount = new AtomicInteger(0);

    public int addCount(int count) {
        return totalCount.addAndGet(count);
    }
}
```


==2.方法加锁==
给方法加 `synchronized` 锁后，所有调用都会排队，导致性能下降。比如下面这个例子：
```
@Service  
public class CounterService {  
  
    private int totalCount = 0;  

    // 给方法加 synchronized 锁住整个方法
    public synchronized int addCount(int count) {  
        int c = 10;
        totalCount += count;  
        return totalCount;  
    }  
}
```
当一个线程调用 `addCount` 方法时，整个实例对象（即 `this`）会被锁定。也就是说，该对象内所有被 `synchronized` 修饰的实例方法都会被阻塞，直到锁释放。但如果对象内存在未加锁的方法，则不会受到影响，仍可正常调用。

因此，我们建议只对修改共享成员变量的公共方法加锁，不涉及共享状态的其他方法则尽量避免加锁，以减少性能损耗。

> [!NOTE] 注意事项
> 1. 普通实例方法加 `synchronized` 时，锁的是当前对象实例（`this`）
> 2. 静态方法加 `synchronized static` 时，锁的是该类的 Class 对象，所有实例共享同一把锁，所有实例都被锁了，但没加锁的方法，都不会被阻塞，照样可以调用


==3.代码块加锁==
它不是给整个方法加锁，而是在方法内部用一对大括号包裹的代码块，加上 `synchronized` 关键字，只锁本对象的代码块
```
@Service  
public class CounterService {  
  
    private int totalCount = 0;  

    private final Object lock = new Object();  // 自定义锁对象

    public int addCount(int count) {  
        int c = 10;

        synchronized (lock) {  // 只锁住对共享变量的操作
            totalCount += count;  
        }

        return totalCount;  
    }  
}
```


==4.可变状态放到外部系统（推荐）==
我们将可变状态甩给外部系统，比如数据库、Redis、消息队列等，从而避免在 Java 服务内部处理线程安全问题。你可能会疑惑：把状态交给它们后，难道外部系统就不会有线程安全问题吗？

实际上，这些外部系统天生具备强大的并发控制能力，能够帮我们确保状态的一致性。以数据库为例，它天然支持事务和行级锁，即使有上千个线程同时写入，也能通过行锁、MVCC（多版本并发控制）和隔离级别等机制，有效地管理并发访问，保证数据安全和一致。

---


#### 1.3. 解决外部连接服务线程安全问题

上文我们讲完了普通服务中的线程安全问题和解决方案，但如果该服务涉及外部连接（如数据库、FTP、Redis 等），情况就会变得更复杂，比如：
```
@Component
public class SharedJedisClient {

    private Jedis jedis = new Jedis("localhost", 6379);

    public String getValue(String key) {
        return jedis.get(key);
    }

    public void setValue(String key, String value) {
        jedis.set(key, value);
    }
}
```

这个 Bean 在初始化时就创建了一个 TCP 连接，之后的方法直接使用这个连接执行操作。表面上看，这个 Bean 并没有定义可变的成员变量，连接对象也没在方法中被修改，似乎一切很安全对吧？

但其实存在严重的线程安全隐患。因为多个线程如果同时调用这个 Bean 中的方法，它们会共享同一个连接对象，导致线程之间对连接的操作交叉进行，可能发生数据错乱、粘包拆包、读写冲突等问题。

你可能会想，那我不共享连接，每个方法里都 `new` 一个连接，岂不就天然线程安全了？确实，这样做可以保证线程隔离，但代价也很高。频繁地创建和销毁连接（尤其是 TCP 连接）会造成巨大的性能开销，严重影响系统吞吐量，并不是理想的做法。
```
@Component
public class NewPerCallJedisClient {

    public String getValue(String key) {
        try (Jedis jedis = new Jedis("localhost", 6379)) {
            return jedis.get(key);
        }
    }

    public void setValue(String key, String value) {
        try (Jedis jedis = new Jedis("localhost", 6379)) {
            jedis.set(key, value);
        }
    }
}
```

如果我们尝试套用前面解决普通服务线程安全问题的那四种方法，虽然能避免线程冲突，但往往会带来额外的性能开销，尤其是在涉及连接操作时，代价更高。

其实，我们还有一种更高效的解决方案 —— 连接池。连接池被我们设置为公用的成员变量，所有调用该 Bean 的线程都可以使用它。在每次方法调用时，线程会从连接池中借出一个连接，执行完操作后再归还池中。

这样既能做到多线程各用自己的连接，避免线程安全问题，又能通过连接复用来减少频繁创建/销毁连接的开销，兼顾了线程安全与系统性能。
```
@Component
public class PooledJedisClient {
    // 在应用启动时创建连接池
    private final JedisPool pool = new JedisPool("localhost", 6379);

    public String getValue(String key) {
        // 从池里借一个 Jedis 实例，操作完自动归还
        try (Jedis jedis = pool.getResource()) {
            return jedis.get(key);
        }
    }

    public void setValue(String key, String value) {
        try (Jedis jedis = pool.getResource()) {
            jedis.set(key, value);
        }
    }
}
```


> [!NOTE] 注意事项
> 1. `Netty` 好像也能处理这个事情，Redis 就是使用的这个方式，还维护自己的一套nid 线程池？不理解不理解
























