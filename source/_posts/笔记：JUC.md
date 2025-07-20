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


## 3. 创建线程

### 3.1. 直接使用 Thread

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


### 3.2. 使用 Runnable + Thread

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


### 3.3. 使用 FutureTask + Thread

#### 3.3.1. Runnable 的缺陷

无论是使用 Runnable + Thread，还是直接使用 Thread，我们都会发现只能执行无返回值的方法。也就是说，方法执行完成后无法获取返回值，而有时我们确实需要返回值来进行错误处理。

除此之外，我们也发现，使用 Runnable 时也无法抛出受检异常（checked Exception）。

---


#### 3.3.2. Callable 的引入

Callable 和 Runnable 类似，都是用来定义任务的接口。不同的是，Callable 定义了带返回值且可抛异常的 `V call()` 方法。

但需要注意的是，Thread 构造方法只能接收 Runnable 类型的对象，因此 Callable 不能像 Runnable 那样直接传给 Thread 使用。
```
@FunctionalInterface
public interface Callable<V> {
    V call() throws Exception;
}
```

---


### 3.4. FutureTask + Thread 的使用

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


## 4. 线程的状态

### 4.1. 操作系统层面

![](image-20250717222336285.png)

----


### 4.2. Java 层面

![](image-20250717201013757.png)


---


## 5. JUC 常用方法

### 5.1. 常用方法一览表

| 方法名                         | static | 说明                                                             | 注意事项                                                                                                                                                                                                                                                                                                |
| --------------------------- | ------ | -------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Thread static 方法            |        |                                                                |                                                                                                                                                                                                                                                                                                     |
| Thread.sleep(long)          | static | 让当前执行的线程休眠 n 毫秒（从运行状态进入 Timed Waiting 状态），休眠时会让出 cpu 的时间片给其它线程 | 1. 其他线程可以通过 `interrupt` 方法打断当前正在睡眠的线程，此时 `sleep` 方法会抛出 `InterruptedException`。如果你未捕获并处理该异常，线程可能会直接终止；<br><br>2. 如果线程在睡眠结束或被中断（并正确处理了异常），将会从 `TIMED_WAITING` 状态被唤醒，转入就绪状态，等待 CPU 分配时间片继续执行。<br><br>3. 建议使用 TimeUnit 的 sleep 代替 Thread 的 sleep，两者功能相同，但前者可读性更强，单位更清晰。<br><br>4. 以 毫秒 为单位，1000 毫秒是 1 秒 |
| Thread.yield()              | static | 提示线程调度器让出当前执行的线程对 CPU 的使用，线程从运行状态进入就绪状态                        | 1. 主要是为了测试和调试<br><br>2. yield 方法仅是建议性提示，是否生效取决于具体的调度策略；<br><br>3. 与 `sleep()` 方法的区别在于：`yield()` 让出时间片后，线程会处于就绪状态，如果没有其他可运行的线程，当前线程仍有可能被继续调度执行；而 `sleep()` 会使线程进入`TIMED_WAITING`，不能随即就被调度                                                                                                            |
| Thread 实例方法                 |        |                                                                |                                                                                                                                                                                                                                                                                                     |
| xxThread.start()            |        | 启动一个新线程，在新的线程运行 run 方法中的代码                                     | 1. start 方法只是让线程从初始状态进入就绪状态，里面代码不一定立刻运行（需要CPU 将时间片分配给它）<br><br>2. 每个线程对象的 start 方法只能调用一次（即使线程已终止），如果调用多次会出现 IllegalThreadStateException<br><br>3.`static` 方法是通过类名调用，如 `Thread.xxx`；非 `static` 方法是通过实例对象调用，如 `myThread.xxx`。                                                                         |
| xxThread.setPriority(int)   |        | 修改某线程的优先级                                                      | 1. Java 中规定线程优先级为 1~10 之间的整数，数值越大，表示该线程被 CPU 调度的概率越高；<br><br>2. 该优先级仅是建议性提示，是否生效取决于具体的调度策略；<br><br>3. 当 CPU 忙碌时，高优先级线程通常会获得更多时间片；但在 CPU 空闲时，优先级对调度几乎无影响。                                                                                                                                            |
| xxThread.join()             |        | 等待某线程运行结束                                                      |                                                                                                                                                                                                                                                                                                     |
| xxThread.join(long)         |        | 等待某线程运行结束，最多等待 n 毫秒                                            | 1. 超时后，代码不再等待，继续向下执行；<br><br>2. 以 毫秒 为单位，1000 毫秒是 1 秒                                                                                                                                                                                                                                               |
| xxThread.setDaemon(boolean) |        | 将某线程（用户线程）设置为守护线程                                              | 1. 主线程和我们创建的线程默认都是用户线程。很多人误以为“主线程一结束，JVM 就会退出”，但实际上并非如此，即使主线程结束，只要还有其他用户线程存活，JVM 就不会退出。<br><br>2. 而守护线程的行为则不同，当所有非守护线程（即用户线程）都结束后，JVM 会自动退出，无需等待守护线程执行完毕。<br><br>3. 守护线程常用于后台服务，例如垃圾回收、心跳监控、日志清理等任务                                                                                                 |
| Object 实例方法                 |        |                                                                |                                                                                                                                                                                                                                                                                                     |
|                             |        |                                                                |                                                                                                                                                                                                                                                                                                     |
|                             |        |                                                                |                                                                                                                                                                                                                                                                                                     |
|                             |        |                                                                |                                                                                                                                                                                                                                                                                                     |
|                             |        |                                                                |                                                                                                                                                                                                                                                                                                     |

----


### 5.2. Thread static 方法

#### 5.2.1. sleep(long) 方法

| 方法名                | static | 说明                                                             | 注意事项                                                                                                                                                                                                                                                                                                |
| ------------------ | ------ | -------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Thread.sleep(long) | static | 让当前执行的线程休眠 n 毫秒（从运行状态进入 Timed Waiting 状态），休眠时会让出 cpu 的时间片给其它线程 | 1. 其他线程可以通过 `interrupt` 方法打断当前正在睡眠的线程，此时 `sleep` 方法会抛出 `InterruptedException`。如果你未捕获并处理该异常，线程可能会直接终止；<br><br>2. 如果线程在睡眠结束或被中断（并正确处理了异常），将会从 `TIMED_WAITING` 状态被唤醒，转入就绪状态，等待 CPU 分配时间片继续执行。<br><br>3. 建议使用 TimeUnit 的 sleep 代替 Thread 的 sleep，两者功能相同，但前者可读性更强，单位更清晰。<br><br>4. 以 毫秒 为单位，1000 毫秒是 1 秒 |

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

----


#### 5.2.2. yield() 方法

| 方法名            | static | 说明                                      | 注意事项                                                                                                                                                                                     |
| -------------- | ------ | --------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Thread.yield() | static | 提示线程调度器让出当前执行的线程对 CPU 的使用，线程从运行状态进入就绪状态 | 1. 主要是为了测试和调试<br><br>2. yield 方法仅是建议性提示，是否生效取决于具体的调度策略；<br><br>3. 与 `sleep()` 方法的区别在于：`yield()` 让出时间片后，线程会处于就绪状态，如果没有其他可运行的线程，当前线程仍有可能被继续调度执行；而 `sleep()` 会使线程进入`TIMED_WAITING`，不能随即就被调度 |

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


### 5.3. Thread 实例方法

#### 5.3.1. start() 方法

| 方法名              | static | 说明                         | 注意事项                                                                                                                                                                                                                        |
| ---------------- | ------ | -------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| xxThread.start() |        | 启动一个新线程，在新的线程运行 run 方法中的代码 | 1. start 方法只是让线程从初始状态进入就绪状态，里面代码不一定立刻运行（需要CPU 将时间片分配给它）<br><br>2. 每个线程对象的 start 方法只能调用一次（即使线程已终止），如果调用多次会出现 IllegalThreadStateException<br><br>3.`static` 方法是通过类名调用，如 `Thread.xxx`；非 `static` 方法是通过实例对象调用，如 `myThread.xxx`。 |

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


#### 5.3.2. setPriority(int) 方法

| 方法名                       | static | 说明        | 注意事项                                                                                                                                                     |
| ------------------------- | ------ | --------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| xxThread.setPriority(int) |        | 修改某线程的优先级 | 1. Java 中规定线程优先级为 1~10 之间的整数，数值越大，表示该线程被 CPU 调度的概率越高；<br><br>2. 该优先级仅是建议性提示，是否生效取决于具体的调度策略；<br><br>3. 当 CPU 忙碌时，高优先级线程通常会获得更多时间片；但在 CPU 空闲时，优先级对调度几乎无影响。 |
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


#### 5.3.3. join()、join(long) 方法

| 方法名                 | static | 说明                  | 注意事项                                                  |
| ------------------- | ------ | ------------------- | ----------------------------------------------------- |
| xxThread.join()     |        | 等待某线程运行结束           |                                                       |
| xxThread.join(long) |        | 等待某线程运行结束，最多等待 n 毫秒 | 1. 超时后，代码不再等待，继续向下执行；<br><br>2. 以 毫秒 为单位，1000 毫秒是 1 秒 |
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


#### 5.3.4. setDaemon(boolean)

| 方法名                         | static | 说明                | 注意事项                                                                                                                                                                                                |
| --------------------------- | ------ | ----------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| xxThread.setDaemon(boolean) |        | 将某线程（用户线程）设置为守护线程 | 1. 主线程和我们创建的线程默认都是用户线程。很多人误以为“主线程一结束，JVM 就会退出”，但实际上并非如此，即使主线程结束，只要还有其他用户线程存活，JVM 就不会退出。<br><br>2. 而守护线程的行为则不同，当所有非守护线程（即用户线程）都结束后，JVM 会自动退出，无需等待守护线程执行完毕。<br><br>3. 守护线程常用于后台服务，例如垃圾回收、心跳监控、日志清理等任务 |
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


### 5.4. Object 实例方法

#### 5.4.1. wait() 方法

| 方法名               | static | 说明                                                                                                      | 注意事项                                                                                                                                                                                                     |
| ----------------- | ------ | ------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Object.wait()     |        | 使当前持有该对象锁的线程进入该对象监视器（Monitor）中的 WaitSet 中等待，同时释放对象锁和 CPU 执行权                                            | 1. 与 `sleep()` 的区别是，`sleep()` 使线程进入 TIMED_WAITING 状态，但不会释放所持有的对象锁；而 `wait()` 会使线程进入 WAITING / TIMED_WAITING 状态，并且会释放对象锁。<br><br>2. `wait()` 一定要放在 `synchronized` 中，否则会抛异常；<br><br>3. 必须获得此对象的锁，才能调用这几个方法 |
| Object.wait(long) |        | 使当前持有该对象锁的线程进入该对象监视器（Monitor）中的 WaitSet 中等待，同时释放对象锁和 CPU 执行权<br><br>最多等待 n 毫秒后自动醒来，不论是否被 `notify()` 唤醒。 | 1. 单位是毫秒，1000 毫秒是 1 秒                                                                                                                                                                                    |

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

----


#### 5.4.2. notify()、notifyAll() 方法

| 方法名                | static | 说明                                | 注意事项                                                                                                                                                                                                                          |
| ------------------ | ------ | --------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Object.notify()    |        | 唤醒该对象监视器（Monitor）中 WaitSet 中的一个线程 | 1. 被唤醒的线程进入可运行状态，但尚未持有对象锁，只有在获得 CPU 时间片并成功获取该对象锁后才能继续执行<br><br>2. `notify()` 一定要放在 `synchronized` 中，否则会抛异常。<br><br>3. 被唤醒的线程是此前因调用 `wait()` 而释放该对象锁、并进入等待集的线程；而当前执行 `notify()` 的线程是持有同一个对象锁的线程。<br><br>4. 必须获得此对象的锁，才能调用这几个方法 |
| Object.notifyAll() |        | 唤醒该对象监视器（Monitor）中 WaitSet 中的全部线程 |                                                                                                                                                                                                                               |
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


### 6.5. 阻塞式解决方案（悲观锁思想）

#### 锁的分类

##### 可重入锁、不可重入锁

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


##### 乐观锁、悲观锁

乐观锁认为冲突是小概率事件，访问资源时不加锁，只有在更新时才进行校验。如果检测到资源已被其他线程修改，则放弃本次修改并重试，适用于读多写少的场景。Java 中提供的 CAS 操作，是乐观锁的一种实现方式，而 `Atomic` 原子类就是基于 CAS 实现的。

而悲观锁基于一种保守假设，只要存在多线程操作共享资源，就极有可能产生冲突。因此，每次访问资源前，线程都会先获取锁，以独占方式持有资源，直到操作完成后释放锁，其它线程需等待锁被释放后才能继续访问。

当无法获取到锁资源时，当前线程会被挂起（进入 BLOCKED 或 WAITING 状态）。线程挂起与唤醒涉及用户态与内核态之间的切换，这种上下文切换是较为耗费系统资源的。
1. 用户态：
	1. 指 JVM 能独立完成的操作，不需要操作系统介入。
2. 内存态：
	1. 指必须由操作系统参与、通过系统调用才能完成的操作。

Java 中提供的 `synchronized`、`ReentrantLock`、`ReentrantReadWriteLock` 都属于悲观锁的实现。

----


##### 公平锁和非公平锁

公平锁是指，当线程 A 获取到锁资源后，线程 B 竞争失败，便进入等待队列排队。此时如果线程 C 也来竞争锁，它会直接排在 B 的后面，只有当 B 获取到锁或取消等待后，C 才有机会尝试获取锁。即，先来先服务，遵循排队原则，防止线程饥饿

而非公平锁是指，线程 A 获取到锁资源，线程 B 竞争失败后进入等待队列。这时线程 C 到来，不会直接排队，而是会先尝试竞争一波：
1. 若成功获取锁：
	1. 插队成功，直接执行，破坏了队列公平性。
2. 若竞争失败：
	1. 才会排到 B 后面，继续等待获取锁或等 B 放弃后再次尝试。

Java 中的 `synchronized` 只能实现非公平锁，而 `ReentrantLock` 和 `ReentrantReadWriteLock` 则支持公平锁和非公平锁两种模式。

----


##### 互斥锁、共享锁

互斥锁是指，在同一时刻，只有一个线程能够持有该锁，其它线程必须等待锁被释放后才能竞争获取。

而共享锁是指，在同一时刻，该锁可以被多个线程同时持有，多个线程可以并发访问共享资源（通常是只读操作）。

Java 中的 `synchronized`、`ReentrantLock` 只能实现互斥锁，而 `ReentrantReadWriteLock` 则支持互斥锁和共享锁两种模式。

----

#### 6.5.2. synchronized

##### 6.5.2.1. synchronized 基本使用

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


##### 6.5.2.2. synchronized 底层原理

###### 6.5.2.2.1. Java 对象头

通常我们的一个 Java 对象，他在内存中由两部分组成，一部分是 Java 对象头，一部分是对象中的一些成员变量，对于 对象头而言，以 32 位虚拟机为例：

<font color="#92d050">1. 普通对象</font>
![](image-20250718231947171.png)


<font color="#92d050">2. 数组对象</font>
![](image-20250718232007146.png)


<font color="#92d050">3. Mark Word</font>
![](image-20250719085830451.png)

----


###### 6.5.2.2.2. Monitor

Monitor（可译为 “监视器” 或 “管程”）是 JVM 内部专门用于实现重量级锁的结构。当一个 Java 对象升级为重量级锁时，JVM 会为其关联一个 Monitor 对象，用于协调多线程之间对该对象的互斥访问。
![](image-20250719102443800.png)

当 Thread 1 执行完临界区代码后，会根据线程中保存的指向 Object 的地址找到该对象，再通过对象头中指向的 Monitor 地址找到 Monitor 对象，将其 Owner 设置为 null，并唤醒 EntryList 中所有等待的线程，这些线程随后开始竞争锁的拥有权。

----


###### 6.5.2.2.3. 轻量级锁

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


###### 6.5.2.2.4. 偏向锁

偏向锁是在轻量级锁基础上的进一步优化，其设计初衷是为了在无竞争的场景下提升锁操作的性能。

然而，随着硬件性能提升和虚拟机其他优化手段的发展，偏向锁带来的性能收益已逐渐减弱，同时其实现的复杂性也成为阻碍 JVM 进一步优化的负担。因此，Oracle 在 JDK 15 中将偏向锁标记为废弃，并在 JDK 17 中将其彻底移除。若使用的是 JDK8，偏向锁默认是开启的，我们需要通过在 JVM 启动参数中添加配置来显式禁止偏向锁。
```
-XX:-UseBiasedLocking
```

---


##### synchronized 高级使用

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


#### ReentrantLock

##### ReentrantLock 与 synchronized 的区别

























### 非阻塞式解决方案（乐观锁思想）

#### 原子变量

------







































## 线程的活跃性

### 死锁

#### 死锁概述

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


#### 死锁定位

##### jconsole 定位死锁
![](image-20250719183232957.png)

---


#### 死锁解决方案

---


### 活锁

#### 活锁概述

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


#### 活锁解决方案


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


# ----------


## 并发编程三大特性

### 原子性

原子性是指，保证指令不会受到线程上下文切换的影响。

----


### 可见性

#### 可见性概述

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


#### 可见性问题解决方案

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
> 2. 除了加 volatile 之外，还可以使用 synchronized 关键字来确保可见性，因为持有锁的线程会在释放锁之前将所有修改的共享变量同步至主内存中；在获取锁时，会从主内存中重新加载所有与这个锁相关的共享变量

----











### 有序性

有序性是指，程序执行的顺序按照代码的先后顺序执行，保证指令不会受 cpu 指令重排的影响。

---







































