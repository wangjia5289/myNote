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
## 1. 导图

![](source/_posts/笔记：JUC/Map：JUC.xmind)

----


## 3. 创建线程

----


## 6. 线程安全问题

### 6.1. 临界区概述

当一段代码块中存在对共享资源的多线程读写操作，这段代码就称为**临界区**。例如：
```
static int counter = 0;

public static void main(String[] args) throws InterruptedException {

	Thread t1 = new Thread(() -> {
		for (int i = 0; i < 5000; i++) 
		// 临界区
		{	
			counter++;
		}
	}, "t1");

	Thread t2 = new Thread(() -> {
		for (int i = 0; i < 5000; i++) 
		// 临界区
		{
			counter--;
		}
	}, "t2");

	t1.start();
	t2.start();
	t1.join();
	t2.join();
	System.out.println(counter);
}
```

又比如：
```
public class Counter {

    private int count = 0;

    public void increment() 
    // 临界区
    { 
        count++;
    }
}
```

---


### 6.2. 线程安全问题概述

现在有这样一个代码：
```
static int counter = 0;

public static void main(String[] args) throws InterruptedException {
	Thread t1 = new Thread(() -> {
		for (int i = 0; i < 5000; i++) {
			counter++;
		}
	}, "t1");

	Thread t2 = new Thread(() -> {
		for (int i = 0; i < 5000; i++) {
			counter--;
		}
	}, "t2");

	t1.start();
	t2.start();
	t1.join();
	t2.join();
	System.out.println(counter);
}
```

一个线程对 `counter` 执行 5000 次加法，另一个线程对 `counter` 执行 5000 次减法。**按理来说，最终结果应该是 0**，但实际上输出可能是正数、负数，甚至恰好为零。为什么会这样呢？

这是因为 Java 中对静态变量的自增、自减操作**不是原子性的**。要彻底理解这个问题，我们需要先了解 Java 的内存模型（JMM）：
![](image-20250718153733803.png)

然后我们要从 字节码层面分析自增、自减的完整过程。以 `counter++` 为例，Java 编译器会生成以下 JVM 字节码指令：
```
getstatic       counter           // 获取静态变量 counter 的值
iconst_1                               // 准备常量 1
iadd                                     // 自增
putstatic       counter           // 将修改后的值存入静态变量 counter
```

而 `counter--` 则会生成类似的指令：
```
getstatic       counter          // 获取静态变量 counter 的值
iconst_1                              // 准备常量 1
isub                                    // 自减
putstatic       counter         // 将修改后的值存入静态变量 counter
```

假设 `counter++` 的线程执行到了 `iadd`，此时 `counter` 的值为 5，准备通过 `putstatic` 写入主内存，但就在这一步之前，CPU 的时间片耗尽发生了线程上下文切换。

接着，另一个执行 `counter--` 的线程获得了 CPU 时间并执行了多次减法操作，将 `counter` 的值减到 2，并把 2 写回了主内存。之后 `counter++` 线程恢复执行，继续完成刚才的 `putstatic`，把原本准备写的 **5** 写回了主内存，覆盖了之前减到 2 的结果，导致最终结果不正确。

----


### 6.3. 线程安全问题分析

一个程序中运行多个线程本身没有问题，关键在于多个线程访问共享资源时的情况：
1. 多个线程只读共享资源通常不会有问题；
2. 多个线程中存在读写混合时，就容易出现指令交错，导致读取的数据不是最新的，写入的数据也可能被覆盖，从而引发错误。

-----


### 6.4. 阻塞式解决方案（悲观锁思想）

#### 6.4.1. 锁的分类

##### 6.4.1.1. 可重入锁、不可重入锁

可重入锁是指当前线程获取到 A 锁，在获取之后尝试再次获取 A 锁是可以直接拿到的。Java 中提供的 synchronized、ReentrantLock、ReentrantReadWriteLock 都属于可重入锁的实现。
```
public class MiTest {

    private static volatile MiTest test;

    private MiTest() {}

    public static MiTest getInstance() {
        if (test == null) {
            synchronized (MiTest.class) {
                if (test == null) {
                    test = new MiTest();
                }
            }
        }
        return test;
    }
}
```

----


##### 6.4.1.2. 乐观锁、悲观锁

乐观锁认为冲突是小概率事件，访问资源时不加锁，只有在更新时才进行校验。如果检测到资源已被其他线程修改，则放弃本次修改并重试，适用于读多写少的场景。Java 中提供的 CAS 操作，是乐观锁的一种实现方式，而 `Atomic` 原子类就是基于 CAS 实现的。

而悲观锁基于一种保守假设，只要存在多线程操作共享资源，就极有可能产生冲突。因此，每次访问资源前，线程都会先获取锁，以独占方式持有资源，直到操作完成后释放锁，其它线程需等待锁被释放后才能继续访问。

当无法获取到锁资源时，当前线程会被挂起（进入 BLOCKED 或 WAITING 状态）。线程挂起与唤醒涉及用户态与内核态之间的切换，这种上下文切换是较为耗费系统资源的。
1. 用户态：
	1. 指 JVM 能独立完成的操作，不需要操作系统介入。
2. 内存态：
	1. 指必须由操作系统参与、通过系统调用才能完成的操作。

Java 中提供的 `synchronized`、`ReentrantLock`、`ReentrantReadWriteLock` 都属于悲观锁的实现。

----


##### 6.4.1.3. 公平锁和非公平锁

公平锁是指，当线程 A 获取到锁资源后，线程 B 竞争失败，便进入等待队列排队。此时如果线程 C 也来竞争锁，它会直接排在 B 的后面，只有当 B 获取到锁或取消等待后，C 才有机会尝试获取锁。即，先来先服务，遵循排队原则，防止线程饥饿

而非公平锁是指，线程 A 获取到锁资源，线程 B 竞争失败后进入等待队列。这时线程 C 到来，不会直接排队，而是会先尝试竞争一波：
1. 若成功获取锁：
	1. 插队成功，直接执行，破坏了队列公平性。
2. 若竞争失败：
	1. 才会排到 B 后面，继续等待获取锁或等 B 放弃后再次尝试。

Java 中的 `synchronized` 只能实现非公平锁，而 `ReentrantLock` 和 `ReentrantReadWriteLock` 则支持公平锁和非公平锁两种模式。

----


##### 6.4.1.4. 互斥锁、共享锁

互斥锁是指，在同一时刻，只有一个线程能够持有该锁，其它线程必须等待锁被释放后才能竞争获取。

而共享锁是指，在同一时刻，该锁可以被多个线程同时持有，多个线程可以并发访问共享资源（通常是只读操作）。

Java 中的 `synchronized`、`ReentrantLock` 只能实现互斥锁，而 `ReentrantReadWriteLock` 则支持互斥锁和共享锁两种模式。

----

#### 6.4.2. synchronized

##### 6.4.2.1. synchronized 基本使用

```
public class Room {

    private int counter = 0;

    public synchronized void increment() {
        counter++;
    }

    public synchronized void decrement() {
        counter--;
    }

    public synchronized int getCounter() {
        return counter;
    }
}
```

> [!NOTE] 注意事项
> 1. 静态方法加 `synchronized static` 时，锁的是该类的 Class 对象
> 2. 普通方法加 `synchronized` 时，锁的是当前对象（`this`），其可以简写为：
```
// 1. 写法 1
public void increment() {
	synchronized (this) {
		counter++;
	}
}


// 2. 写法 2
public synchronized void increment() {
	counter++;
}
```

----


##### 6.4.2.2. synchronized 底层原理

###### 6.4.2.2.1. Java 对象头

通常我们的一个 Java 对象，他在内存中由两部分组成，一部分是 Java 对象头，一部分是对象中的一些成员变量，对于 对象头而言，以 32 位虚拟机为例：

<font color="#92d050">1. 普通对象</font>
![](image-20250718231947171.png)


<font color="#92d050">2. 数组对象</font>
![](image-20250718232007146.png)


<font color="#92d050">3. Mark Word</font>
![](image-20250719085830451.png)

----


###### 6.4.2.2.2. Monitor

Monitor（可译为 “监视器” 或 “管程”）是 JVM 内部专门用于实现重量级锁的结构。当一个 Java 对象升级为重量级锁时，JVM 会为其关联一个 Monitor 对象，用于协调多线程之间对该对象的互斥访问。
![](image-20250719102443800.png)

当 Thread 1 执行完临界区代码后，会根据线程中保存的指向 Object 的地址找到该对象，再通过对象头中指向的 Monitor 地址找到 Monitor 对象，将其 Owner 设置为 null，并唤醒 EntryList 中所有等待的线程，这些线程随后开始竞争锁的拥有权。

----


###### 6.4.2.2.3. 轻量级锁

考虑到重量级锁的性能问题，锁竞争时线程需要挂起与唤醒，会触发操作系统层面的上下文切换，代价极高，动辄消耗数万 CPU 周期。同时，频繁的线程阻塞还可能带来调度延迟、缓存失效等额外副作用。

而现实中，大多数锁的竞争程度其实并不高：很多临界区的代码执行极快，线程间访问呈错峰分布，根本不会产生真正的冲突；即便发生竞争，通常也只是短暂的、少数线程的重叠访问。这种情况下，一个线程已迅速完成执行，而另一个线程却仍在挂起、唤醒、重新调度的过程中。更何况，即使没有阻塞与唤醒，重量级锁的加锁与解锁过程本身也不轻。因此，为了提升并发性能，在这类 “低冲突、短执行” 的场景中，Java 官方将重量级锁优化为轻量级锁，语法不变
```
public class Room {

    private int counter = 0;

    public synchronized void increment() {
        counter++;
    }

    public synchronized void decrement() {
        counter--;
    }

    public synchronized int getCounter() {
        return counter;
    }
}
```

现在有线程去调用 synchronized 的方法，会在其线程栈帧中创建一个锁记录（Lock Record）。
![](image-20250719100014731.png)

接着，线程尝试给对象加轻量级锁，操作流程是：先让 Object reference 指向对象地址，并使用 CAS（原子操作）尝试将锁记录的地址与对象头中的 Mark Word 交换。
![](image-20250719100114713.png)

如果 CAS 替换成功，对象头中就存储了锁记录的地址（因为需要记录是哪个线程拥有该对象的锁），此时 Object 的状态为 00，表示对象持有轻量级锁。当临界区代码执行完毕，锁记录会被交换回对象头。
![|850](image-20250719100159695.png)

如果 CAS 失败，但，是当前线程在执行 synchronized 发生重入，会添加一条新的锁记录作为重入计数。锁记录数量表示该线程对该对象加锁的次数，例如：
```
static final Object obj = new Object();

public static void method1() {
    synchronized(obj) {
        method2();
    }
}

public static void method2() {
    synchronized(obj) {
    }
}
```

![](image-20250719100555573.png)

如果 CAS 失败且发生线程竞争，当前线程会进行多次自旋尝试（赌其他线程能快速释放锁，避免像重量级锁一样直接挂起带来的性能损耗）。若多次尝试仍未成功，则发生锁膨胀，锁会升级为重量级锁。

假设 Thread 0 已持有轻量级锁，但 Thread 1 多次自旋均未成功，
![](image-20250719101118338.png)

这时 JVM 会为该对象分配一个 Monitor 对象，将对象头中原先指向锁记录的 Mark Word 替换为指向 Monitor 的指针，Thread 1 被加入 Monitor 的 EntryList 队列挂起，Monitor 的 Owner 指向 Thread 0。
![](image-20250719101521799.png)

当 Thread 0 执行完临界区代码，需要解锁时，会先尝试通过 CAS 将锁记录恢复到对象头（Mark Word），肯定发生失败，进入重量级锁解锁流程。  

解锁时，线程会根据线程中保存的指向 Object 的地址找到该对象，再通过对象头中指向的 Monitor 地址找到 Monitor 对象，将其 Owner 设置为 null，并唤醒 EntryList 中所有等待的线程，这些线程随后开始竞争锁的拥有权。
    
虽然轻量级锁相比不加锁确实带来了一些额外成本（但共享资源操作必须加锁），例如每次进入锁时需要创建锁记录、执行 CAS 操作，并且存在短暂的自旋，可能会消耗一些 CPU 周期；

但轻量级锁有效避免了线程阻塞和唤醒的开销，在大多数 “无锁竞争” 或 “低冲突” 的场景下，能够以极低的成本完成加锁与释放，因此整体性能远远优于重量级锁。

----


###### 6.4.2.2.4. 偏向锁

偏向锁是在轻量级锁基础上的进一步优化，其设计初衷是为了在无竞争的场景下提升锁操作的性能。

然而，随着硬件性能提升和虚拟机其他优化手段的发展，偏向锁带来的性能收益已逐渐减弱，同时其实现的复杂性也成为阻碍 JVM 进一步优化的负担。因此，Oracle 在 JDK 15 中将偏向锁标记为废弃，并在 JDK 17 中将其彻底移除。若使用的是 JDK8，偏向锁默认是开启的，我们需要通过在 JVM 启动参数中添加配置来显式禁止偏向锁。
```
-XX:-UseBiasedLocking
```

---


##### 6.4.2.3. synchronized 高级使用

对于下面的代码：
```
public class Room {

    public void sleep() {
        synchronized (this) {
            System.out.println("sleeping 2 小时");
        }
    }

    public void study() {
        synchronized (this) {
            System.out.println("study 1 小时");
        }
    }
}
```

虽然 `study()` 方法和 `sleep()` 方法本身没有逻辑上的交集，但如果某个线程调用了带有 `synchronized` 的 `sleep()` 方法，就会导致该对象，甚至可能是整个类被加锁。这样一来，其他线程即使只是想调用 `study()` 方法，也必须等到前一个线程释放锁后才能执行。这种做法不利于并发性能的提升，因此我们可以通过引入多把锁来进行优化：
```
public class Room {

    private final Object studyRoom = new Object();
    
    private final Object bedRoom = new Object();

    public void sleep(){
        synchronized (bedRoom) {
            System.out.println("sleeping 2 小时");
        }
    }

    public void study() {
        synchronized (studyRoom) {
            System.out.println("study 1 小时");
        }
    }
}
```

----


#### 6.4.3. ReentrantLock

##### 6.4.3.1. ReentrantLock 与 synchronized 的区别

























### 6.5. 非阻塞式解决方案（乐观锁思想）

#### 6.5.1. CAS

CAS，全称为 Compare-And-Swap，中文意为“比较并交换”，是并发编程中最基础且关键的原子操作之一。通俗地说，CAS 的含义是：先检查当前主内存中的值是否与我预期的一致，如果一致，就将其更新为新值；否则，说明有其他线程修改过该值，此时我不会进行任何操作。

CAS 操作涉及三个核心参数：
1. V：
	1. 当前主内存中的实际值（Current value）
2. E：
	1. 线程期望该变量拥有的值（Expected value）
3. N：
	1. 需要更新的新值（New value）

下面的代码展示了一个典型的 CAS 操作过程：
```
public void withdraw(Integer amount) {

    while (true) {
    
	    // 将当前主内存中的值加载到工作内存
        int prev = balance.get();
        
        // 在工作内存中计算新的结果
        int next = prev - amount;
        
        // 比较并尝试设置结果到主内存，设置失败将重新循环
        if (balance.compareAndSet(prev, next)) {
            break;
        }
    }
}
```

这段代码的关键在于 `compareAndSet` 方法。它是一个原子操作，也必须具备原子性，才能保证在多线程环境下数据更新的正确性。其底层依赖于硬件提供的原子指令（如 x86 架构中的 `cmpxchg`），从而实现了线程安全的变量更新逻辑。它所完成的功能是：
![](image-20250720093607520.png)

> [!NOTE] 注意事项
> 1. CAS 必须结合 `volatile` 使用。Java 提供的各类原子变量（如 `AtomicInteger`）底层都通过 `volatile` 修饰共享变量。这是因为无论是读取当前值，还是进行比较并交换操作，都必须确保读取到的是主内存中的最新值，才能保证线程之间的可见性和操作的正确性
> 2. 由于 CAS 并未使用 `synchronized`，线程在竞争时不会陷入 `BLOCKED` 状态，也不会触发锁的升级过程，这是它能够提升并发性能的重要原因之一。
>    然而，CAS 更适用于线程数量较少、CPU 核心数较多的场景，因为它采用的是 `while` 自旋的方式不断重试，会让 CPU 保持持续高频运转。一旦线程数过多，反复重试拉满 CPU，造成大量 CPU 资源浪费，进而导致性能大幅下降

---


#### 6.5.2. 原子变量

##### 6.5.2.1. AtomicBoolean

----


##### 6.5.2.2. AtomicInteger

```
public class Main {

    static AtomicInteger atomicInteger = new AtomicInteger(0);

    public static void main(String[] args) {

        System.out.println(atomicInteger.incrementAndGet()); // ++i，先自增，然后返回结果

        System.out.println(atomicInteger.getAndIncrement()); // i++，先返回当前值，然后再自增

        System.out.println(atomicInteger.decrementAndGet()); // --i

        System.out.println(atomicInteger.getAndIncrement()); // i--

        atomicInteger.updateAndGet(value -> value * 10); // 先执行操作，然后返回结果

        atomicInteger.getAndUpdate(value -> value * 10); // 先返回当前值，然后再执行操作

        System.out.println(atomicInteger.get()); // 读取当前值
    }
}
```

----


##### 6.5.2.3. AtomicIntegerArray

---


##### 6.5.2.4. AtomicIntegerFieldUpdater

---


##### 6.5.2.5. AtomicLong


----


##### 6.5.2.6. AtomicLongArray

---


##### 6.5.2.7. AtomicLongFieldUpdater

---


## 7. 线程池

### 7.1. 线程池概述






































##### 7.1.1.1. AtomicReference

----


##### 7.1.1.2. AtomicReferenceArray

----



##### 7.1.1.3. AtomicReferenceFieldUpdater

----


##### 7.1.1.4. AtomicMarkableReference

----


##### 7.1.1.5. AtomicStampedReference

------


## 8. 线程的活跃性

### 8.1. 死锁

#### 8.1.1. 死锁概述

当一个线程需要同时获取多把锁时，就容易发生死锁。例如，线程 t1 先获得了 A 对象的锁，接下来试图获取 B 对象的锁；与此同时，线程 t2 已经获得了 B 对象的锁，并准备去获取 A 对象的锁。此时两个线程互相等待对方释放锁，便形成了死锁。
```
public class Test {

    private static final Object A = new Object();
    private static final Object B = new Object();

    private static void runT1() {
        synchronized (A) {
            System.out.println("t1 获得 A");
            sleep(1);
            synchronized (B) {
                System.out.println("t1 获得 B");
                System.out.println("t1 执行操作...");
            }
        }
    }

    private static void runT2() {
        synchronized (B) {
            System.out.println("t2 获得 B");
            sleep(500);
            synchronized (A) {
                System.out.println("t2 获得 A");
                System.out.println("t2 执行操作...");
            }
        }
    }

    private static void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
	
    public static void main(String[] args) {
        Thread t1 = new Thread(Test::runT1, "t1");
        Thread t2 = new Thread(Test::runT2, "t2");

        t1.start();
        t2.start();
    }
}
```

----


#### 8.1.2. 死锁定位

##### 8.1.2.1. jconsole 定位死锁
![](image-20250719183232957.png)

---


#### 8.1.3. 死锁解决方案

---


### 8.2. 活锁

#### 8.2.1. 活锁概述

活锁虽然名字中带有“锁”，但它并不是由我们显式加上的锁造成的，而是指两个线程不断互相改变对方的终止条件，导致最终双方都无法完成或结束的情况。
```
public class TestLiveLock {

    static volatile int count = 10;
    
    private static void sleep(double seconds) {
        try {
            Thread.sleep((long) (seconds * 1000));
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    public static void main(String[] args) {
        new Thread(() -> {
            // 期望减到 0 退出循环
            while (count > 0) {
                sleep(0.2);
                count--;
                log.debug("count: {}", count);
            }
        }, "t1").start();
        
        new Thread(() -> {
            // 期望超过 20 退出循环
            while (count < 20) {
                sleep(0.2);
                count++;
                log.debug("count: {}", count);
            }
        }, "t2").start();
    }
}
```

---


#### 8.2.2. 活锁解决方案


----















































# ------------------
## 1. 自己写的，超级不错



### 1.1. 线程安全问题

#### 1.1.1. 线程安全问题概述

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


#### 1.1.2. 解决普通服务线程安全问题

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


#### 1.1.3. 解决外部连接服务线程安全问题

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

----



## 2. 查看线程

### 2.1. Windows

<font color="#92d050">1. 任务管理器</font>
快捷键为：Ctrl + Shift + Esc


<font color="#92d050">2. 命令行工具</font>
```
// 1. 查看进程
tasklist
"""
1. 注意事项
	1. 可以使用 tasklist | findstr <进程名> 来筛选特定进程
"""


// 2. 杀死进程
taskkill /F /PID <进程 ID>
```

----


### 2.2. Linux

<font color="#92d050">1. ps 命令</font>
```
// 1. 查看所有进程
ps -ef


// 2. 查看某个进程（PID）下的所有线程
ps -fT -p <进程 ID>
```


<font color="#92d050">2. top 命令</font>
```
// 1. 实时查看进程资源占用情况
top
"""
1. 注意事项：
	1. 按下大写 H 可切换是否显示线程视图
"""


// 2. 实时查看某个进程（PID）下的线程资源占用
top -H -p <进程 ID>
```


<font color="#92d050">3. kill 命令</font>
```
// 1. 请求进程优雅退出（发送 SIGTERM）
kill <进程 ID>
"""
1. 注意事项：
	1. 进程可捕获此信号，用于执行资源释放、状态保存等清理操作
"""


// 2. 强制终止进程（发送 SIGKILL）
kill -9 <进程ID>
"""
1. 注意事项：
	1. 用于强制终止卡死、无法响应的进程，不会触发清理逻辑
"""
```

----


### 2.3. Java 专属命令

```
// 1. 查看所有 Java 进程
jps


// 2. 查看某个 Java 进程（PID）下的所有线程
jstack <Java 进程 ID>
```


> [!NOTE] 注意事项
> 1. Windows 系统中的 `jconsole` 是一个图形化工具，可用于查看 Java 进程中各线程的运行状态与资源使用情况
> 2. 该工具默认只能查看本地 Windows 机器上的 Java 进程线程。如果要远程监控，需要用以下命令启动目标机器上的 jar 包：
```
java -Djava.rmi.server.hostname='<ip 地址>' -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.port='<连接端口>' -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -jar yourapp.jar
```

----





# ----------


## 1. 并发编程三大特性

### 1.1. 原子性

原子性是指，保证指令不会受到线程上下文切换的影响。

----


### 1.2. 可见性

#### 1.2.1. 可见性概述

可见性是指，当一个线程修改了共享变量的值，其他线程能够立即知道这个修改，保证指令不会受 cpu 缓存的影响。首先我们来看一个代码：
```
static boolean run = true;

public static void main(String[] args) throws InterruptedException {
    new Thread(() -> {
        while (run) {
            // 业务逻辑
        }
    }).start();

    Thread.sleep(1000);

    run = false;
}
```

按理来说，`run = false` 应该能使线程退出循环，但是实际上，线程并不会立即退出循环，而是会继续执行循环体中的代码。这是因为，`run` 变量被修改后，线程并不知道这个变化，它仍然使用的是旧值，所以会继续执行循环体中的代码。

其实本质原因是，在线程初始化时，会将 `run` 变量的值从主内存复制到线程的工作内存中：
![](image-20250719223150888.png)

执行一次就要从主内存中读取一次，应为线程频繁从主内存中读取run 的值，JIT 会将 run 变量的值缓存到线程的工作内存中的高速缓存中，避免每次都从主内存中读取，从而提高效率。
![](image-20250719223516927.png)

这种优化本意是好的，但是也带来了可见性问题：main 线程修改了 `run` 变量的值，并同步至主内存，但是线程并不知道这个变化，它仍然使用的是旧值，所以会继续执行循环体中的代码。

![](image-20250719223757282.png)

---


#### 1.2.2. 可见性问题解决方案

##### 1.2.2.1. volatile 解决方案

想解决这个问题其实很简单，我们只需为共享变量添加一个新的修饰符 `volatile`。这个关键字的含义是“易变的”，也就是说，加上 `volatile` 之后，变量的值不能再从线程的本地缓存中读取，而是每次都必须从主内存中读取最新的值。
```
static volatile boolean run = true;

public static void main(String[] args) throws InterruptedException {
    new Thread(() -> {
        while (run) {
            // 业务逻辑
        }
    }).start();

    Thread.sleep(1000);

    run = false;
}
```

虽然这样做在性能上会有一定的损失，但它确保了多个线程之间对共享变量的可见性，也就是说，一个线程修改了变量的值，其他线程能够及时看到这个变化。

> [!NOTE] 注意事项
> 1. volatile 可以用来修饰成员变量和静态变量，不用用来修饰局部变量
> 2. volatile 的本质是基于读写屏障来保证可见性的：
```
/**
 * ============================================
 * 写屏障（sefence）
 * --------------------------------------------
 * 当对被 volatile 修饰的成员变量或静态变量进行写操作时
 * 编译器会在该语句之后插入一条写屏障指令
 * 写屏障用于确保在该屏障之前，对所有共享变量的修改
 * 都被刷新到主内存中，从而保证写操作对其他线程可见。
 * ============================================
 */
public void actor1() {
    num = 1; // num 是普通的共享变量，也会被同步到主内存
    ready = true; // ready 是 volatile 变量
    // 写屏障
}


/**
 * ============================================
 * 读屏障（lfence）
 * --------------------------------------------
 * 当对被 volatile 修饰的成员变量或静态变量进行读操作时
 * 编译器会在该语句之前插入一条读屏障指令
 * 读屏障用于确保在该屏障之后，对所有共享变量的读取操作
 * 都能从主内存中获取最新的数据，避免读取到过期值
 * ============================================
 */
public void actor2() {
    // 读屏障
    // ready 是 volatile 变量
    if (ready) {
        num = num + num; // num 是普通的共享变量，也加载的是主内存中的最新数据
    }
}
```

----


##### 1.2.2.2. synchronized 解决方案

synchronized 解决可见性的原理是：
1. 加锁：
   1. 线程在获取锁之前，会先从主内存中读取共享变量的值，然后将其复制到线程的工作内存中，保证了线程获取锁时，使用的是主内存中的最新值。
2. 解锁：
   1. 线程在释放锁之前，会将线程的工作内存中所有修改的共享变量刷新到主内存中。
    
```
static boolean run = true;

// 锁对象
final static Object Lock = new Object();

public static void main(String[] args) throws InterruptedException {
    Thread t = new Thread(() -> {
        while (true) {
            // ...
            synchronized (Lock) {
                if (!run) {
                    break;
                }
            }
        }
    });
    t.start();

    sleep(1);
    log.debug("停止 t");
    synchronized (Lock) {
        run = false; // 线程t不会如预想的停下来
    }
}
```

> [!NOTE] 注意事项
> 1. synchronized 既能保证可见性，又能保证原子性，但缺点是 synchronized 是重量级操作，性能相对更低

----





###### 1.2.2.2.1. ✅ 2. **ReentrantLock + 显式内存语义**

- 虽然 `ReentrantLock` 本身也保证了加锁/解锁的**可见性**，但更重要的是它可以通过 `lock()` 和 `unlock()` 方法建立**happens-before** 关系。
    

> 所以只要线程获取了锁，就可以看到**之前持有锁的线程对共享变量的修改**。

---

###### 1.2.2.2.2. ✅ 3. **final 变量的安全发布**

- `final` 本身不保证可见性，但通过**构造函数安全发布**的方式（如通过不可变对象发布到线程共享结构中），可以避免重排序导致的可见性问题。
    

---

###### 1.2.2.2.3. ✅ 4. **原子类（例如 AtomicInteger）**

- `AtomicXXX` 类底层通过 `volatile` 实现可见性，同时通过 **CAS + 内存屏障** 实现原子性。
    
- **适用场景**：对数值进行原子更新时需要可见性
    

---

###### 1.2.2.2.4. ✅ 5. **线程通信方法：wait()/notify()、park()/unpark()**

- `wait()` 会让线程释放锁并进入等待状态，`notify()` 唤醒线程。
    
- 这些方法**依赖 synchronized 的可见性语义**，所以线程醒来之后能看到共享变量的最新值。
    

---

###### 1.2.2.2.5. 小结对比：

|方法|可见性|原子性|有序性|是否加锁|
|---|---|---|---|---|
|`volatile`|✅|❌|✅（禁止指令重排）|❌|
|`synchronized`|✅|✅|✅|✅|
|`ReentrantLock`|✅|✅|✅|✅|
|`AtomicInteger`|✅|✅|✅|❌（基于 CAS）|
|`wait/notify`|✅|✅|✅|✅|

如果你关心的是**写了之后别人能看到**，`volatile`、锁、CAS 都是解决方案，但要选哪种，还得看是否也需要原子性、性能等其他考量。

需要我根据某种特定场景来推荐选择哪种方式吗？








### 1.3. 有序性

#### 1.3.1. 有序性概述

有序性是指，程序的执行顺序与代码的先后顺序一致，以防止 CPU 进行指令重排所带来的不确定性。因为 JVM 在不影响程序最终正确性的前提下，可能会调整语句的实际执行顺序。例如，下面这段代码：
```
static int a = 0;
static int b = 1;

static void main() {
    a = 1;
    b = 2;
}
```

从语义上看，先执行 `a = 1`，还是先执行 `b = 2`，并不会改变程序的最终结果。因此，这段代码的实际执行顺序既可以是：
```
a = 1
b = 2
```

也可以是：
```
b = 2
a = 1
```

这种行为称为**指令重排**。其本质目的是通过指令的重排序与组合，提升执行效率，实现指令级并行（流水线技术）。在单线程环境下，指令重排不会改变程序的执行结果，因此通常是安全的。

但在多线程场景下，指令重排可能会导致线程之间观察到的执行结果不一致，进而影响程序的正确性。因此，在并发编程中，我们需要确保指令重排不会破坏程序逻辑和线程之间的数据可见性，例如：
```
// 可以重排的列子
int a = 10;
int b = 20;
System.out.println(a + b);


// 不能重排的例子
public class ConcurrencyTest {

    int num = 0;
    
    volatile boolean ready = false;
    
	// 一个线程执行这个
    public void actor1(I_Result r) {
        if (ready) {
            r.r1 = num + num;
        } else {
            r.r1 = 1;
        }
    }

	// 一个线程执行这个
    public void actor2(I_Result r) {
        num = 3;
        ready = true;
    }
}
```

> [!NOTE] 注意事项
> 1. 上述代码在某些情况下可能输出结果为 0，这种现象通常只有在进行大量并发测试时才更容易复现，因此需要借助压测工具辅助观察。其根本原因在于，actor2 中的指令发生了重排，导致执行顺序被打乱：
```
ready = true;
num = 3;
```

---


#### 1.3.2. 有序性问题解决方案

##### 1.3.2.1. volatile 解决方案

同样是为共享变量添加 volatile 关键字，既能够保证可见性，同时也能够禁止指令重排，其同样是基于读写屏障实现的。
```
/**
 * ============================================
 * 写屏障（sefence）
 * --------------------------------------------
 * 当对被 volatile 修饰的成员变量或静态变量进行写操作时
 * 编译器会在该语句之后插入一条写屏障指令
 * 写屏障用于确保指令重排序时，不会将写屏障之前的代码排在写屏障之后
 * ============================================
 */
public void actor1() {
    num = 1; 
    ready = true; // ready 是 volatile 变量
    // 写屏障
}


/**
 * ============================================
 * 读屏障（lfence）
 * --------------------------------------------
 * 当对被 volatile 修饰的成员变量或静态变量进行读操作时
 * 编译器会在该语句之前插入一条读屏障指令
 * 读屏障用于确保指令重排序时，不会将读屏障之后的代码排在都屏障之前
 * ============================================
 */
public void actor2() {
    // 读屏障
    // ready 是 volatile 变量
    if (ready) {
        num = num + num;
    }
}
```

----

# JUC 基础

## 线程基础

### 串行、并发、并行

<font color="#92d050">1. 串行</font>
串行是指任务按顺序一个接一个地执行，只有当前一个任务执行完成，后一个任务才会开始执行，严格遵循任务提交的先后顺序。即便系统拥有多个 CPU 核心，在串行模式下，任意时刻也只会有一个线程在运行。


<font color="#92d050">2. 并发</font>
并发是指多个线程看起来像是在同时运行，其是通过时间片轮转机制实现，通过快速切换线程，让每个线程都获得运行机会，因此并发特性在单核环境中体现得尤为明显。

而在多核 CPU 上，线程有可能被分配到不同的核心上并行执行，但当线程数量多于核心数量时，CPU 仍需通过时间片轮转进行调度，以确保所有线程都能获得执行机会。


<font color="#92d050">3. 并行</font>
并行是指多个线程在真正意义上同时运行，分别占用不同的 CPU 核心，在同一时刻执行各自的任务，体现出真正的同时处理能力。

---


### 同步、异步、阻塞、非阻塞

我们通常将操作分为下面的操作：
1. 同步操作
	1. 同步简单操作
	2. 同步阻塞操作

2. ==同步操作==：
	1. <font color="#00b0f0">同步简单操作</font>：
		1. 快速执行、无明显耗时、不涉及复杂计算或 I/O
	2. <font color="#00b0f0">同步阻塞操作</font>：
		1. 设计耗时计算或潜在阻塞操作（如 I/O），但是在当前线程同步执行
	3. 同步非阻塞操作
3. ==异步操作==：
	1. <font color="#00b0f0">异步阻塞操作</font>：
		1. 将同步阻塞操作通过异步机制（如线程池）执行，“伪非阻塞” 操作
	2. <font color="#00b0f0">异步非阻塞操作</font>：
		1. 完全非阻塞，通常是基于时间驱动或回调，依赖外部异步机制（如 `WebClient`、 `R2DBC`、`CompletableFuture`）
4. ==补充：I/O==：
	1. I/O，即输入/输出（Input/Output），是指计算机与外部世界的信息交换过程，包括但不限于：
		1. <font color="#00b0f0">网络请求</font>：
			1. 如调用其他 HTTP API 接口（RestTemplate、WebClient）、JDBC、R2DBC 等等
		2. <font color="#00b0f0">文件操作</font>：
			1. 读取或写入硬盘上的文件
		3. <font color="#00b0f0">用户交互</font>：
			1. 键盘、鼠标等输入设备的信号获取，以及向显示器等输出设备发送信息。
		4. <font color="#00b0f0">硬件通信</font>：
			1. 与打印机、扫描仪等外部设备的数据交换





这是一个很好的问题，因为这正是 Java 线程状态最容易让人感到困惑的地方。我们来总结一下，哪些常见的操作会导致线程在**操作系统（OS）层面处于阻塞状态**，但在 **Java 层面却依然是 `RUNNABLE`**。

这些操作通常都与 **I/O (输入/输出)** 有关，因为 I/O 操作需要等待外部设备（如硬盘、网卡）的响应，而这个等待过程是由操作系统管理的。

---

### 常见的 I/O 阻塞操作总结

#### 1. 文件 I/O

当线程执行文件读写操作时，如果数据不能立即就绪，就会发生阻塞。

- **操作**：
    
    - `FileInputStream.read()`：从文件中读取数据。
        
    - `FileOutputStream.write()`：向文件中写入数据。
        
- **底层行为**：Java 的 `native` 方法会向操作系统发出系统调用，请求读写磁盘。如果硬盘正在忙碌或数据量大，OS 会将线程挂起，直到 I/O 完成。
    
- **Java 线程状态**：**`RUNNABLE`**。
    

#### 2. 网络 I/O

网络通信是 I/O 阻塞最常见的场景之一。

- **操作**：
    
    - `Socket.accept()`：等待客户端连接。
        
    - `Socket.connect()`：连接远程服务器。
        
    - `Socket.read()` / `Socket.write()`：从网络中读取或写入数据。
        
    - `DatagramSocket.receive()`：接收 UDP 数据包。
        
- **底层行为**：`native` 方法会向操作系统请求网络资源。当线程在等待网络连接建立、或者等待数据从网络到达时，OS 会将线程挂起。
    
- **Java 线程状态**：**`RUNNABLE`**。
    

#### 3. 数据库 I/O

与数据库的交互本质上也是网络或文件 I/O。

- **操作**：
    
    - `Connection.createStatement()`
        
    - `Statement.executeQuery()`
        
    - `ResultSet.next()`
        
- **底层行为**：JDBC 驱动通过网络向数据库服务器发送请求。线程需要等待服务器处理请求并返回结果。
    
- **Java 线程状态**：**`RUNNABLE`**。
    

---

### 为什么这些操作会这样？

核心原因在于 **Java `Thread.State` 的设计哲学**。

- `BLOCKED`, `WAITING`, `TIMED_WAITING` 这些状态，是专门为了反映 **Java 语言层面的同步和等待机制**。它们是 JVM 精心管理和控制的状态，比如等待 `synchronized` 锁或调用 `wait()` 方法。
    
- **I/O 阻塞**是一个**操作系统层面的行为**。当 Java 调用 `native` 方法进行 I/O 时，它把控制权交给了 OS。从 JVM 的角度看，线程只是在执行一个 `native` 方法，它并没有通过 Java 自己的机制进入等待。一旦 OS 的 I/O 操作完成，这个 `native` 方法就会返回，线程就能继续执行 Java 代码。因此，JVM 认为这个线程一直都是**可运行**的。
    

通过理解这个区别，你就能更好地诊断和分析 Java 多线程程序中的性能瓶颈，区分是由于锁竞争（Java `BLOCKED`）还是由于 I/O 等待（Java `RUNNABLE`，OS 阻塞）造成的了。

---


### 线程的创建

#### 继承 Thread 类，重写 run 方法

此景难得，我们就以这个场景为例，来举一举类的几种常见玩法：

<font color="#92d050">1. 使用匿名内部类的方式继承 Thread 类，并重写 run 方法</font>
```
public class Main {  
  
    public static void main(String[] args) {  
    
		// 创建线程对象
        Thread myThread = new Thread("myThread") {  
            @Override  
            public void run() {  
                // 本线程要执行的任务  
            }  
        };  
          
        // 启动线程  
        myThread.start();  
    }  
}
```

类只能被继承，接口只能被实现。在这种写法中，看似直接 new 了一个 Thread 对象，实际上这是典型的匿名内部类（四种内部类之一）的声明方式。匿名内部类是专门用来继承某个类或实现某个接口，我们省去了单独定义一个类（因为它是匿名的），代码更简洁。

匿名内部类的本质是：在编译时，编译器会自动生成两个字节码文件，一个是 `Demo.class`，另一个是 `Demo$1.class`。这个自动生成的类继承了 Thread 类，并重写了 `run` 方法。

当执行到 `Thread myThread = new Thread("myThread") { ... };` 时，JVM 会像处理普通类一样，将 `Demo$1.class` 加载到方法区，并在堆区创建这个对象实例，赋值给变量 `myThread`。

> [!NOTE] 注意事项
> 1. 使用匿名类创建了临时的类对象，方法执行完后如果栈帧中没有对该对象的引用，它可能会在下一次 GC 时就被回收
> 2. 但它对应的类信息仍然会加载到方法区，而且我们知道方法区中的类卸载条件非常严格，所以必须明确匿名类的适用范围：
> 	1. 如果只是临时封装逻辑，使用匿名类是合适的
> 		1. 但如果需要在两处及以上复用同一段逻辑，匿名类就不太合适
> 	2. 匿名类不支持灵活传参，若需要传参灵活，建议避免使用匿名类
> 	3. 在事件驱动或回调场景中，匿名类则非常适用


<font color="#92d050">2. 使用显示声明类的方式继承 Thread 类，并重写 run 方法</font>
显式声明类，是指无论你是直接定义一个类对象，还是在某个类中编写成员内部类、静态内部类、局部内部类等嵌套类，总之你都是明确地定义了一个具名类
```
public class MyThread extends Thread{  
  
    @Override  
    public void run() {  
        // 本线程要执行的任务  
  
    }  
}
```

然后我们去创建线程对象：
```
public class Main {  
  
    public static void main(String[] args) {  
  
        // 1. 正常创建 MyThread 类型的线程对象  
        MyThread myThread1 = new MyThread();  
  
        // 2. 向上转型创建 Thread 类型的线程对象
        Thread myThread2 = new MyThread();  
  
        // 启动线程  
        myThread1.start();  
        myThread2.start();  
    }  
  
}
```

> [!NOTE] 注意事项
> 1. 这个示例不太适合作为演示 “向下转型” 的例子，详见笔记：Java SE
> 2. 我们在继承类并重写方法的时候，通常会自动写一个 super.run()，这个是指调用父类的 run() 方法例如：
```
public class MyThread extends Thread{  
  
    @Override  
    public void run() {  
        super.run();
    }  
}
```

---


#### 3.2. 使用 Runnable + Thread

直接使用 `Thread` 相当于将线程控制与具体任务耦合在一起，为了具有更好的灵活性，也为了更容易与线程池等高级并发 API 配合使用，我们可以使用 `Runnable` 实现了线程与任务的分离。

`Runnable` 是一个接口，其源码如下所示：
```
@FunctionalInterface
public interface Runnable {
    public abstract void run();
}
```

接下来我们可以通过如下方式创建一个线程对象：
```
// 1. 创建 Runnable 接口对象
Runnable runnable = new Runnable() {
	@Override
	public void run() {
		// 本 Runnable 要执行的任务
	}
};

// 2. 创建线程对象，传入 Runnable 接口对象。线程将执行 Runnable 中指定的 run 方法
Thread myThread = new Thread(runnable, "myThread");

// 3. 启动线程
myThread.start();
```

> [!NOTE] 注意事项
> 1. 在 Java 中，只包含一个抽象方法的接口可以加上 `@FunctionalInterface` 注解，表示该接口是一个函数式接口，从而可以使用 lambda 表达式进行简洁书写（只有一个抽象方法，但还可能存在几个 default 方法）
> 2. 需要注意的是，如果 lambda 表达式中有多条语句，必须使用花括号 `{}` 包裹；如果只有一条语句，则可以省略花括号。
```
// lambda 表达式创建 Runnable
Runnable runnable = () -> {
	// 本 Runnable 要执行的任务
};
```

---


#### 3.3. 使用 FutureTask + Thread

##### 3.3.1. Runnable 的缺陷

无论是使用 Runnable + Thread，还是直接使用 Thread，我们都会发现只能执行无返回值的方法。也就是说，方法执行完成后无法获取返回值，而有时我们确实需要返回值来进行错误处理。

除此之外，我们也发现，使用 Runnable 时也无法抛出受检异常（checked Exception）。

---


##### 3.3.2. Callable 的引入

Callable 和 Runnable 类似，都是用来定义任务的接口。不同的是，Callable 定义了带返回值且可抛异常的 `V call()` 方法。

但需要注意的是，Thread 构造方法只能接收 Runnable 类型的对象，因此 Callable 不能像 Runnable 那样直接传给 Thread 使用。
```
@FunctionalInterface
public interface Callable<V> {
    V call() throws Exception;
}
```

---


##### 3.4. 使用 FutureTask + Thread 

关于 Runnable、Callable、Future 的详细解释，详见下文：异步编程。这里我们先了解如何通过这种方式来创建线程

```
// 1. 创建 Callable 接口对象
Callable<Integer> task = new Callable<Integer>() {
	@Override
	public Integer call() throws Exception {
		return 1 + 2;
	}
};

// 2. 创建 FutureTask 对象，传入 Callable 接口对象。
FutureTask<Integer> futureTask = new FutureTask<>(task);

// 3. 创建线程对象，传入 FutureTask 对象。线程将执行 Callable 中指定的 call 方法，并能调用 FutureTask 的方法进行相关操作
Thread myThread = new Thread(futureTask, "myThread");

// 4. 启动线程
myThread.start();

// 5. 调用 FutureTask  相关方法
try {
    Integer result = futureTask.get();  // 阻塞等待任务执行完成，获取返回值
    System.out.println("任务返回结果: " + result);
} catch (InterruptedException | ExecutionException e) {
    e.printStackTrace();
}
```

> [!NOTE] 注意事项
> 1. 创建 FutureTask 对象时，可直接使用 Lambda 表达式，无需单独创建 Callable 接口对象
```
FutureTask<Integer> futureTask = new FutureTask<>(() -> 1 + 2);

Thread myThread = new Thread(futureTask, "myThread");

myThread.start();
```

----


#### 基于线程池

这部分内容，详见下文：线程池

---


### 线程的状态

#### 4.1. 操作系统层面

操作系统层面的线程状态，是**操作系统内核对线程的实际调度反映**
![](image-20250717222336285.png)

----


#### 4.2. Java 层面

Java 层面的线程状态，是 JVM 对线程生命周期的抽象和管理，主要**反映线程在 Java 内存模型中的行为**：
![](image-20250717201013757.png)
1. 线程状态
	1. NEW
		1. 指线程对象已创建，但尚未调用 `start()` 方法处于未启动状态
		2. 在这一阶段，Java 中的线程对象尚未与底层操作系统的原生线程建立关联
	2. RUNNABLE
		1. 可运行状态
			1. 线程已经准备好运行，但能否真正运行，还要取决于操作系统的调度
		2. 运行状态
		3. 阻塞状态
			1. 一般是我们执行的某些操作，涉及到操作系统层面的线程阻塞，例如 I/O 操作时
			2. 虽然其真实情况是线程进入阻塞状态，CPU 不再调用这个线程
			3. 但是从 JVM 的角度看，执行 IO 操作的线程通常仍然停留在 `RUNNABLE` 状态
			4. 而我们 Java 层面的 Bolcked、WAITING、TIMED_WAITING 主要是我们去调用了 `wait()`、`sleep()`、`join()` 等方法进入的
			5. 你可以简单理解为，只要不是 `wait()`、`sleep()`、`join()` 等情况，其他的虽然是进入了阻塞状态，但是在 java 层面还是认为是 `RUNNABLE` 状态
			6. 常见的阻塞 I/O 有：
				1. 文件 I/O
				2. 网络 I/O
				3. 数据库 I/O
				4. 用户交互
				5. 需要注意的是，传统的这些操作，都是阻塞 I/O，但随着技术的发展，java 提供了这些操作的 BIO、NIO、AIO 实现。
	3. BLOCKED
	4. WAITING
	5. TIMED_WAINTING
	6. TERMINATED
2. 线程状态转换
	1. NEW ---> 可运行状态
		1. 当线程对象调用 `start()`方法，Java 中的线程对象就与底层操作系统的原生线程建立关联，就进入了可运行状态
	2. 运行状态 ---> WAITING
		1. 调用 Object.wait()
			1. 当线程适用 synchronized(obj)获取对象锁后，调用 obj.wait()方法时，线程会从运行状态 ---> WAITING / TIMED_WAITING
			2. 补充：当我们调用 obj.notify()、obj.notifyAll()、xxThread.interrupt() 时，线程会尝试竞争锁：
				1. 竞争锁成功，线程从 WAITING / TIMED_WAITING ---> 可运行状态
				2. 竞争锁失败，线程从 WAITING / TIMED_WAITING ---> BOLCKED（线程从 WaitSet 进入了 EntryList）
		2. 调用 xxThread.join()
	3. 运行状态 ---> TIMED_WAITING
		1. 调用 Object.wait(long)
		2. 调用 xxThread.join(long)
		3. 调用 xxThread.sleep(long)
	4. WAININT <---> 可运行状态
		1. 调用 Object.notify()、Object.notifyAll()、xxThread.interrupt()
			1. 当线程适用 synchronized(obj)获取对象锁后，调用 obj.wait()方法时，线程会从运行状态 ---> WAITING / TIMED_WAITING


---




---


### 线程的常用方法

#### 5.2. Thread 静态方法

##### 5.2.1. sleep(long)

###### sleep(long) 概述

| 方法名                | static | 说明                                       | 注意事项                                                                                                                                                                                                                                                                                                                                                     |
| ------------------ | ------ | ---------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Thread.sleep(long) | static | 让**当前执行的线程**休眠 n 毫秒，休眠时会让出 cpu 的时间片给其它线程 | 1. 从运行状态 ---> TIMED_WAITING<br><br>2. 其他线程可以通过 `xxThread.interrupt()` 方法打断某个正在睡眠的线程，此时 `Thered.sleep()` 方法会抛出 `InterruptedException`。如果你未捕获并处理该异常，线程可能会直接终止；<br><br>3. 如果线程在睡眠结束或被中断（并正确处理了异常），将会从 `TIMED_WAITING` ---> 可运行状态，等待 CPU 分配时间片继续执行。<br><br>4. 建议使用 TimeUnit 的 sleep 代替 Thread 的 sleep，两者功能相同，但前者可读性更强，单位更清晰。<br><br>5. 以 毫秒 为单位，1000 毫秒是 1 秒 |

```
public class Main {
    public static void main(String[] args) throws ExecutionException, InterruptedException {

        Callable<Integer> task = new Callable<Integer>() {
            @Override
            public Integer call() throws Exception {
            
                // 执行本方法的线程进行 sleep
                try {
                    Thread.sleep(50000);
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
                return 1 + 2;
            }
        };

        FutureTask<Integer> futureTask = new FutureTask<>(task);

        Thread myThread = new Thread(futureTask, "myThread");

        myThread.start();

        // Main 线程 sleep
        Thread.sleep(1000);
        
        myThread.interrupt();
    }
}
```

> [!NOTE] 注意事项
> 1. 使用 TimeUnit 的 sleep 代替 Thread 的 sleep
```
public class Main {

    public static void main(String[] args) throws ExecutionException, InterruptedException {

        Callable<Integer> task = new Callable<Integer>() {
            @Override
            public Integer call() throws Exception {
            
                // 执行本方法的线程 sleep
                try {
                    TimeUnit.HOURS.sleep(5);
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
                return 1 + 2;
            }
        };

        FutureTask<Integer> futureTask = new FutureTask<>(task);

        Thread myThread = new Thread(futureTask, "myThread");

        myThread.start();

        // Main 线程 sleep
        TimeUnit.MILLISECONDS.sleep(5000);

        myThread.interrupt();
    }
}
```

----


###### sleep(long) 使用案例

<font color="#92d050">1. 防止 CPU 空转</font>
CPU 空转是指**线程持续进行无意义的轮询操作**：既没有执行有价值的计算，也不是在等待某个条件或事件的变化，仅仅是在占用 CPU 做无用功。在当前时间片内，CPU 使用率被拉满但是却没做任何有意义的事情，像这样毫无意义地持续运行，就是典型的 CPU 空转行为。相比之下，更合理的做法是使用 `Thread.yield()` 或 `Thread.sleep()` 主动让出 CPU，给其他线程或程序使用资源。

以如下代码为例：
```
public class Main {  
  
    public static void main(String[] args) {  
  
        while (true) {  
            System.out.println("程序正在运行中~~~");  
        }  
    }  
}
```

这段代码会导致某个线程在当前时间片内持续占用 CPU，但实际上进行的是无意义的 “空转”，尤其在单核 CPU 情况下影响更大。我们只需简单地在循环中加入 `Thread.sleep(50)` 或者 `Thread.yield()`，就能显著降低 CPU 占用率，避免资源浪费，同时不会影响程序的逻辑流程
```
public class Main {  
  
    public static void main(String[] args) {  
  
        for (;;) {  
            Thread.sleep(50);
            System.out.println("程序正在运行中~~~");  
        }  
    }  
}
```

> [!NOTE] 注意事项
> 1. `Object.wait()` 或者条件变量也能达到类似的效果
> 2. 不过这两种方式都需要在加锁的前提下使用，并依赖其他线程进行唤醒，通常适用于涉及线程协作的同步场景
> 3. 而 `Thread.sleep(50)` 和 `Thread.yield()` 则无需锁和同步，适合单线程控制节奏、减少空转的简单场景

---


##### 5.2.2. yield()

| 方法名            | static | 说明                                          | 注意事项                                                                                                                                                                                                    |
| -------------- | ------ | ------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Thread.yield() | static | 提示线程调度器让出**当前执行的线程**对 CPU 的使用，线程从运行状态进入就绪状态 | 1. 运行状态 ---> 可执行状态<br><br>2. 这个方法主要是为了测试和调试，并且仅是建议性提示，是否生效取决于具体的调度策略；<br><br>3. 与 `sleep()` 方法的区别在于：`yield()` 让出时间片后，线程会处于可运行状态，如果没有其他可运行的线程，当前线程仍有可能被继续调度执行；而 `sleep()` 会使线程进入`TIMED_WAITING`，不能随即就被调度 |

```
public class Main {
    public static void main(String[] args) throws ExecutionException, InterruptedException {

        Runnable runnable1 = new Runnable() {
            @Override
            public void run() {
                Thread.yield();
                while (true){
                    System.out.println("----->1");
                }
            }
        };
        Runnable runnable2 = new Runnable() {
            @Override
            public void run() {
                while (true){
                    System.out.println("       ----->2");
                }
            }
        };

        Thread myThread1 = new Thread(runnable1, "myThread1");

        Thread myThread2 = new Thread(runnable2, "myThread2");
        
        myThread1.start();
        
        myThread2.start();
    }
}
```

> [!NOTE] 注意事项
> 1. 该代码在多核处理器上效果可能不明显，但在单核环境中其调度行为将更为明显

---


#### 5.3. Thread 实例方法

##### 5.3.1. start()

| 方法名              | static | 说明                                                           | 注意事项                                                                                                |
| ---------------- | ------ | ------------------------------------------------------------ | --------------------------------------------------------------------------------------------------- |
| xxThread.start() |        | 启动 xxThread 线程对象，使其与底层操作系统的原生线程建立关联，并在新线程中执行 `run()` 方法中的代码。 | 1. NEW ---> 可运行状态<br><br>2. 每个线程对象的 start 方法只能调用一次（即使线程已终止），如果调用多次会出现 `IllegalThreadStateException` |

```
public class Main {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        
        Callable<Integer> task = new Callable<Integer>() {
            @Override
            public Integer call() throws Exception {
                return 1 + 2;
            }
        };
        
        FutureTask<Integer> futureTask = new FutureTask<>(task);
        
        Thread myThread = new Thread(futureTask, "myThread");
        
        myThread.start();
        
    }
}
```

---


##### interrupt()

| 方法名                  | static | 说明  | 注意事项                                                                                                                                                                                                                                                                               |
| -------------------- | ------ | --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| xxThread.interrupt() |        |     | 1. TIMED_WAITINT ---> 可运行状态 / 程序终止<br><br>2. interrupt() 方法主要用于打断处于等待状态（WAITING / TIMED_WAITING）状态的线程，被打断的线程将处于可运行状态<br><br>3. 如果被打断的线程是处于运行状态的线程，interrupt() 不会中断线程的执行，但是会设置线程的中断标志位（isInterrupted() 为 true），如果线程代码中主动检查了中断状态（例如通过 `Thread.interrupted()`），就可以自行做出响应，比如提前退出或者其他逻辑 |

| interrupt() |     | 打断线程 | 如果被打断线程正在 sleep, wait, join 会导致被打断的线程抛出 InterruptedException，并清除打断标记；如果打断的正在运行的线程，则会设置打断标记；park 的线程被打断，也会设置打断标记 |
| ----------- | --- | ---- | --------------------------------------------------------------------------------------------------------------- |

---


##### 5.3.2. setPriority(int)

| 方法名                       | static | 说明                 | 注意事项                                                                                                                                                     |
| ------------------------- | ------ | ------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| xxThread.setPriority(int) |        | 修改 xxThread 线程的优先级 | 1. Java 中规定线程优先级为 1~10 之间的整数，数值越大，表示该线程被 CPU 调度的概率越高；<br><br>2. 该优先级仅是建议性提示，是否生效取决于具体的调度策略；<br><br>3. 当 CPU 忙碌时，高优先级线程通常会获得更多时间片；但在 CPU 空闲时，优先级对调度几乎无影响。 |
```
public class Main {
    public static void main(String[] args) throws ExecutionException, InterruptedException {

        Runnable runnable1 = new Runnable() {
            @Override
            public void run() {
                while (true){
                    System.out.println("----->1");
                }
            }
        };
        Runnable runnable2 = new Runnable() {
            @Override
            public void run() {
                while (true){
                    System.out.println("       ----->2");
                }
            }
        };

        Thread myThread1 = new Thread(runnable1, "myThread1");

        Thread myThread2 = new Thread(runnable2, "myThread2");
        
        // 设置线程优先级
        myThread1.setPriority(Thread.MAX_PRIORITY);

        myThread1.start();
        
        myThread2.start();
    }
}
```

----


##### 5.3.3. join()、join(long)

| 方法名                 | static | 说明                               | 注意事项                                                                                                                                                                                                                                     |
| ------------------- | ------ | -------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| xxThread.join()     |        | 让当前线程等待 xxThread 线程运行结束          | 1. 运行状态 ---> WAITING<br><br>2. 其他线程可以通过 `xxThread.interrupt()` 方法打断某个线程继续等待其他线程运行结束，此时 `xxThread.join()` 方法会抛出 `InterruptedException`，如果你未捕获并处理该异常，线程可能会直接终止；<br><br>3. 线程 b 正在等待线程 a 执行完毕，这时可以在 线程 c 中调用 `b.interrupt()` 来打断线程 b 的等待操作。 |
| xxThread.join(long) |        | 让当前线程等待 xxTread 线程运行结束，最多等待 n 毫秒 | 1. 运行状态 ---> TIMED_WAITING<br><br>2. 超时后，代码不再等待，继续向下执行；<br><br>3. 以 毫秒 为单位，1000 毫秒是 1 秒                                                                                                                                                  |

```
public class Main {
    static int repertoryNumber  = 0;

    public static void main(String[] args) throws ExecutionException, InterruptedException {

        Runnable runnable = new Runnable() {
            @Override
            public void run() {
                try {
                    // 模拟从 MySQL 中读取数据，假设独到的是 1500
                    TimeUnit.MINUTES.sleep(1);
                    repertoryNumber = 1500;
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
            }
        };

        Thread myThread = new Thread(runnable, "myThread");

        myThread.start();

        myThread.join();

        System.out.println("从数据库中读取的数据为："+ repertoryNumber);
    }
}
```

> [!NOTE] 注意事项
> 1. 这里是被 `xxThread.interrupt()` 打断的示例代码：
```
public class Main {

    public static void main(String[] args) throws InterruptedException {
        Thread myThread1 = new Thread(() -> {
            try {
                Thread.sleep(50000);
                System.out.println("myThread1 执行结束");
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }

        });

        Thread myThread2 = new Thread(() -> {
            try {
                System.out.println("myThread2 等待 myThread1 执行结束");
                myThread1.join();
                System.out.println("myThread2 等待 myThread1 执行结束");
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        });

        myThread1.start();
        myThread2.start();

        // 主线程 sleep 2 秒后打断 myThread2
        Thread.sleep(2000);
        myThread1.interrupt();
    }
}
```

----


##### 5.3.4. setDaemon(boolean)

| 方法名                         | static | 说明                         | 注意事项                                                                                                                                                                                                 |
| --------------------------- | ------ | -------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| xxThread.setDaemon(boolean) |        | 将 xxThread 线程（用户线程）设置为守护线程 | 1. 主线程和我们创建的线程默认都是用户线程。很多人误以为 “主线程一结束，JVM 就会退出”，但实际上并非如此，即使主线程结束，只要还有其他用户线程存活，JVM 就不会退出。<br><br>2. 而守护线程的行为则不同，当所有非守护线程（即用户线程）都结束后，JVM 会自动退出，无需等待守护线程执行完毕。<br><br>3. 守护线程常用于后台服务，例如垃圾回收、心跳监控、日志清理等任务 |
```
public class Main {  
    static int repertoryNumber  = 0;  
  
    public static void main(String[] args) throws ExecutionException, InterruptedException {  
  
        Runnable runnable = new Runnable() {  
            @Override  
            public void run() {  
                while (true){  
                    if(Thread.currentThread().isInterrupted()){  
                        break;  
                    }  
                }  
            }  
        };  
  
        Thread myThread = new Thread(runnable, "myThread");  
  
        // 设置为守护线程  
        myThread.setDaemon(true);  
  
        myThread.start();  
  
        System.out.println("从数据库中读取的数据为："+ repertoryNumber);  
    }  
}
```

> [!NOTE] 注意事项
> 1. 上面的代码如果没有设置为守护线程，即使 `main` 方法执行完毕，JVM 仍不会停止运行。

----










| 方法名              | static | 说明                                                             | 注意事项                                                                                                                                    |
| ---------------- | ------ | -------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| join()           |        | 等待线程运行结束                                                       |                                                                                                                                         |
| join(long n)     |        | 等待线程运行结束，最多等待 n 毫秒                                             |                                                                                                                                         |
| start()          |        | 启动一个新线程，在新的线程运行 run 方法中的代码                                     | 1. start 方法只是让线程从初始状态进入就绪状态，里面代码不一定立刻运行（需要CPU 将时间片分配给它）<br><br>2. 每个线程对象的 start 方法只能调用一次（即使线程已终止），如果调用多次会出现 IllegalThreadStateException |
| sleep(long n)    | static | 让当前执行的线程休眠n 毫秒（从 Running 进入 Timed Waiting），休眠时让出 cpu 的时间片给其它线程 |                                                                                                                                         |
| yield()          | static | 提示线程调度器让出当前线程对CPU的使用                                           | 主要是为了测试和调试                                                                                                                              |
| setPriority(int) |        | 修改线程优先级                                                        | java中规定线程优先级是1~10 的整数，较大的优先级能提高该线程被 CPU 调度的机率                                                                                           |
| getId()          |        | 获取线程长整型的 id                                                    | id 唯一                                                                                                                                   |
| getName()        |        | 获取线程名                                                          |                                                                                                                                         |
| setName(String)  |        | 修改线程名                                                          |                                                                                                                                         |
| getPriority()    |        | 获取线程优先级                                                        |                                                                                                                                         |
| getState()       |        | 获取线程状态                                                         | Java 中线程状态是用 6 个 enum 表示，分别为：NEW, RUNNABLE, BLOCKED, WAITING, TIMED_WAITING, TERMINATED                                                 |
| isInterrupted()  |        | 判断是否被打断                                                        | 不会清除打断标记                                                                                                                                |
| isAlive()        |        | 线程是否存活（还没有运行完毕）                                                |                                                                                                                                         |
| interrupt()      |        | 打断线程                                                           | 如果被打断线程正在 sleep, wait, join 会导致被打断的线程抛出 InterruptedException，并清除打断标记；如果打断的正在运行的线程，则会设置打断标记；park 的线程被打断，也会设置打断标记                         |
| interrupted()    | static | 判断当前线程是否被打断                                                    | 会清除打断标记                                                                                                                                 |
| currentThread()  | static | 获取当前正在执行的线程                                                    |                                                                                                                                         |

----


#### 5.4. Object 实例方法

##### 5.4.1. wait() 方法

| 方法名                 | static | 说明                                                                     | 注意事项                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| ------------------- | ------ | ---------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| xxObject.wait()     |        | 使当前持有该对象锁的线程进入该对象监视器（Monitor）中的 WaitSet 中等待，同时释放对象锁和 CPU 执行权           | 1. 运行状态 ---> WAITING（虽然处于可运行状态的线程也可能持有锁，但调用 `xxObject.wait()` 方法时，线程一定处于运行状态）<br><br>2. 与 `Thread.sleep()` 的区别是，`Thread.sleep()` 使线程进入 TIMED_WAITING 状态，但不会释放所持有的对象锁<br><br>3. 而 `xxObject.wait()` 会使线程进入 WAITING 状态，并且会释放所持有的对象锁<br><br>4. `xxObject.wait()` 一定要放在 `synchronized` 加锁代码块中，否则会抛异常<br><br>5. 需要考虑虚假唤醒情况<br><br>6. 其他线程可以通过 `xxThread.interrupt()` 方法打断某个线程的 wait 状态，此时 `xxObject.wait()` 方法会抛出 `InterruptedException`，如果你未捕获并处理该异常，线程可能会直接终止；<br><br>7. 被 `Thread.interrupt()` 的线程同样由 WAITING ---> 可运行状态，但同样需要获得 CPU 时间片并成功获取该对象锁后才能继续执行后续代码（包括异常处理的 catch 代码块） |
| xxObject.wait(long) |        | 使当前持有该对象锁的线程进入该对象监视器（Monitor）中的 WaitSet 中等待，同时释放对象锁和 CPU 执行权，最多等待 n 毫秒 | 1. 运行状态 ---> TIMED_WAITING<br><br>2. 超时后，线程自动醒来，无论是否被 `xxObject.notify()` 唤醒<br><br>3. 单位是毫秒，1000 毫秒是 1 秒                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |

```
public class WaitNotifyDemo {

    private static final Object lock = new Object();
    
    private static boolean hasData = false;

    // 消费者线程
    static class Consumer extends Thread {
        @Override
        public void run() {
            synchronized (lock) {
                while (!hasData) {
                    try {
                        System.out.println("消费者：没有数据，等待...");
                        lock.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                System.out.println("消费者：拿到了数据，消费完成");
                hasData = false;
                lock.notify();  // 通知生产者可以继续生产
            }
        }
    }

    // 生产者线程
    static class Producer extends Thread {
        @Override
        public void run() {
            synchronized (lock) {
                while (hasData) {
                    try {
                        System.out.println("生产者：数据未被消费，等待...");
                        lock.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                System.out.println("生产者：生产了数据");
                hasData = true;
                lock.notify();  // 通知消费者可以消费
            }
        }
    }

    public static void main(String[] args) {
        new Consumer().start();
        new Producer().start();
    }
}
```

> [!NOTE] 注意事项
> 1. 这里是被 `xxThread.interrupt()` 打断的示例代码：
```
public class Main {  
  
    public static void main(String[] args) throws InterruptedException {  
  
        Object object = new Object();  
  
        Thread myThread1 = new Thread(() -> {  
            synchronized (object) {  
                System.out.println("开始等待...");  
                try {  
                    object.wait();  
                } catch (InterruptedException e) {  
                    System.out.println("myThread1 的 wait() 被 interrupt 打断");  
                }  
                System.out.println("代码继续执行...");  
            }  
        });  
  
        myThread1.start();  
  
        // 主线程 sleep 2 秒后打断 myThread1        
        Thread.sleep(2000);  
        myThread1.interrupt();  
    }  
}
```

----


##### 5.4.2. notify()、notifyAll() 方法

| 方法名                | static | 说明                                                  | 注意事项                                                                                                                                                                                    |
| ------------------ | ------ | --------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| xxObject.notify()  |        | 当前持有该对象锁的线程去唤醒在<br>该对象监视器（Monitor）中 WaitSet 中随机一个线程 | 1. 被唤醒的线程由 WAITING / TIMED_WAITING ---> 可运行状态，但不要忘记，这些方法都是在 `synchronized` 代码块中的，也就是说还需要再获得 CPU 时间片并成功获取该对象锁后才能继续执行后续代码<br><br>2. `xxObject.notify()` 一定要放在 `synchronized` 代码块中，否则会抛异常。 |
| Object.notifyAll() |        | 当前持有该对象锁的线程去唤醒唤醒该对象监视器（Monitor）中 WaitSet 中的全部线程     |                                                                                                                                                                                         |

```
public class WaitNotifyDemo {
    private static final Object lock = new Object();
    private static boolean hasData = false;

    // 消费者线程
    static class Consumer extends Thread {
        @Override
        public void run() {
            synchronized (lock) {
	            // 防止线程虚假唤醒
                while (!hasData) {
                    try {
                        System.out.println("消费者：没有数据，等待...");
                        lock.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                System.out.println("消费者：拿到了数据，消费完成");
                hasData = false;
                lock.notify();  // 通知生产者可以继续生产
            }
        }
    }

    // 生产者线程
    static class Producer extends Thread {
        @Override
        public void run() {
            synchronized (lock) {
	            // 防止线程虚假唤醒
                while (hasData) {
                    try {
                        System.out.println("生产者：数据未被消费，等待...");
                        lock.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                System.out.println("生产者：生产了数据");
                hasData = true;
                lock.notify();  // 通知消费者可以消费
            }
        }
    }

    public static void main(String[] args) {
        new Consumer().start();
        new Producer().start();
    }
}

```

> [!NOTE] 注意事项：虚假唤醒
> 1. 虚假唤醒是指线程在没有被明确唤醒（即没有收到 `notify()` / `notifyAll()` 调用）、也没有超时（对 `wait(timeout)` 而言）的情况下恢复到了可运行状态。
> 2. 这并非 Bug，而是 JVM 设计上的一种容忍，但我们为了防止线程在资源尚未准备好时误以为条件满足，我们通常使用 `while` 循环反复检查条件，从而有效避免虚假唤醒带来的问题。

----


#### LockSupport 静态方法

##### park()

| 方法名                | static | 说明     | 注意事项                 |
| ------------------ | ------ | ------ | -------------------- |
| LockSupport.park() | static | 暂停当前线程 | 1. 运行状态 ---> WAITING |

---


##### unpark()

| 方法名                  | static | 说明        | 注意事项                  |
| -------------------- | ------ | --------- | --------------------- |
| LockSupport.unpark() | static | 恢复某个线程的运行 | 1. WAITING ---> 可运行状态 |




































