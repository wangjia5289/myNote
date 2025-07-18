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
## 导图

![](source/_posts/笔记：JUC/Map：JUC.xmind)

----


## 查看线程

### Windows

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


### Linux

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


### Java 专属命令

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


### 创建线程

#### 直接使用 Thread

```
// 1. 创建线程对象
Thread myThread = new Thread("myThread") {
	@Override
	public void run() {
		// 本线程要执行的任务
	}
};

// 2. 启动线程
myThread.start();
```

----


#### 使用 Runnable + Thread

直接使用 `Thread` 相当于将线程控制与具体任务耦合在一起，为了具有更好的灵活性，也为了更容易与线程池等高级并发 API 配合使用，我们可以使用 `Runnable` 实现了线程与任务的分离。

`Runnable` 是一个接口，其源码如下所示：
```
@FunctionalInterface
public interface Runnable {
    public abstract void run();
}
```

接下来我们可以通过如下方式创建一个线程：
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
> 1. 在 Java 中，只包含一个抽象方法的接口可以加上 `@FunctionalInterface` 注解，表示该接口是一个函数式接口，从而可以使用 lambda 表达式进行简洁书写。
> 2. 如果 lambda 表达式中有多条语句，必须使用花括号 `{}` 包裹；如果只有一条语句，则可以省略花括号。
```
// 1. Runnable 源码
@FunctionalInterface
public interface Runnable {
    public abstract void run();
}


// 2. lambda 表达式创建 Runnable
Runnable runnable = () -> {
	// 本 Runnable 要执行的任务
};
```
>3. Runnable 是一个接口，而创建 Runnable 接口通常有三种常见玩法：
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

----


#### 使用 FutureTask + Thread

##### Runnable 的缺陷

无论是使用 Runnable + Thread，还是直接使用 Thread，我们都会发现只能执行无返回值的方法。也就是说，方法执行完成后无法获取返回值，而有时我们确实需要返回值来进行错误处理。

除此之外，我们也发现，使用 Runnable 时也无法抛出受检异常（checked Exception）。

---


##### Callable 的引入

Callable 和 Runnable 类似，都是用来定义任务的接口。不同的是，Callable 定义了带返回值且可抛异常的 `V call()` 方法。

但需要注意的是，Thread 构造方法只能接收 Runnable 类型的对象，因此 Callable 不能像 Runnable 那样直接传给 Thread 使用。
```
@FunctionalInterface
public interface Callable<V> {
    V call() throws Exception;
}
```

---


#### FutureTask + Thread 的使用

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

// 3. 创建线程对象，传入 FutureTask 对象。线程将指向 Callable 中指定的 call 方法，并能调用 FutureTask 的方法进行相关操作
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


## 线程的状态

### 操作系统层面

![](image-20250717222336285.png)

----


### Java 层面

![](image-20250717201013757.png)


---


## 线程的常用方法

### start() 方法

| 方法名     | static | 说明                         | 注意事项                                                                                                                                                                                                                        |
| ------- | ------ | -------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| start() |        | 启动一个新线程，在新的线程运行 run 方法中的代码 | 1. start 方法只是让线程从初始状态进入就绪状态，里面代码不一定立刻运行（需要CPU 将时间片分配给它）<br><br>2. 每个线程对象的 start 方法只能调用一次（即使线程已终止），如果调用多次会出现 IllegalThreadStateException<br><br>3.`static` 方法是通过类名调用，如 `Thread.xxx`；非 `static` 方法是通过实例对象调用，如 `myThread.xxx`。 |

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

----


### sleep(long) 方法

| 方法名         | static | 说明                                                             | 注意事项                                                                                                                                                                                                                                                                                              |
| ----------- | ------ | -------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| sleep(long) | static | 让当前执行的线程休眠 n 毫秒（从运行状态进入 Timed Waiting 状态），休眠时会让出 cpu 的时间片给其它线程 | 1. 其他线程可以通过 `interrupt` 方法打断正在睡眠的线程，此时 `sleep` 方法会抛出 `InterruptedException`。如果你未捕获并处理该异常，线程可能会直接终止；<br><br>2. 如果线程在睡眠结束或被中断（并正确处理了异常），将会从 `TIMED_WAITING` 状态被唤醒，转入就绪状态，等待 CPU 分配时间片继续执行。<br><br>3. 建议使用 TimeUnit 的 sleep 代替 Thread 的 sleep，两者功能相同，但前者可读性更强，单位更清晰。<br><br>4. 以 毫秒 为单位，1000 毫秒是 1 秒 |

```
public class Main {
    public static void main(String[] args) throws ExecutionException, InterruptedException {

        Callable<Integer> task = new Callable<Integer>() {
            @Override
            public Integer call() throws Exception {
            
                // 执行本方法的线程 sleep
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
> 1. 使用 TimeUnit：
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

---


### yield() 方法

| 方法名     | static | 说明                                   | 注意事项                                                                                                                                                                                     |
| ------- | ------ | ------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| yield() | static | 提示线程调度器让出当前线程对 CPU 的使用，线程从运行状态进入就绪状态 | 1. 主要是为了测试和调试<br><br>2. yield 方法仅是建议性提示，是否生效取决于具体的调度策略；<br><br>3. 与 `sleep()` 方法的区别在于：`yield()` 让出时间片后，线程会处于就绪状态，如果没有其他可运行的线程，当前线程仍有可能被继续调度执行；而 `sleep()` 会使线程进入`TIMED_WAITING`，不能随即就被调度 |

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


### setPriority(int) 方法

| 方法名              | static | 说明      | 注意事项                                                                                                                                                     |
| ---------------- | ------ | ------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| setPriority(int) |        | 修改线程优先级 | 1. Java 中规定线程优先级为 1~10 之间的整数，数值越大，表示该线程被 CPU 调度的概率越高；<br><br>2. 该优先级仅是建议性提示，是否生效取决于具体的调度策略；<br><br>3. 当 CPU 忙碌时，高优先级线程通常会获得更多时间片；但在 CPU 空闲时，优先级对调度几乎无影响。 |
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

### join()、join(long) 方法

| 方法名        | static | 说明                 | 注意事项                                                  |
| ---------- | ------ | ------------------ | ----------------------------------------------------- |
| join()     |        | 等待线程运行结束           |                                                       |
| join(long) |        | 等待线程运行结束，最多等待 n 毫秒 | 1. 超时后，代码不再等待，继续向下执行；<br><br>2. 以 毫秒 为单位，1000 毫秒是 1 秒 |
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














dafd













# ------------------
## 自己写的，超级不错



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

----






































