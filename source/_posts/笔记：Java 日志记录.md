---
title: 笔记：Java 日志记录
date: 2025-03-11
categories:
  - Java
  - 日志记录
tags: 
author: 霸天
layout: post
---
### 0、导图：[Map：日志框架](Map：日志框架.xmind)

---


### 1、框架发展史
![](image-20250311223318675.png)

---


### 2、JUL 实现框架

#### 2.1、JUL 概述

JUL 全称 Java Util Logging，是 JDK 内置的日志记录框架，使用时无需额外引入第三方库，使用相对简单。虽然能满足一般应用的日志需求，但相比 Log4j、SLF4J 等第三方框架，其扩展性和高级特性较为有限，更适合中小型项目或对日志需求较为简单的场景。

---


#### 2.2、JUL 组件介绍

![|500](source/_posts/笔记：Java 日志记录/image-20250311134645238.png)

1. <font color="#00b0f0">Logger</font>：
	1. Logger（记录器）是日志框架的入口，我们通过调用 `Logger.getLogger(String name)` 获取 Logger 实例，并调用其方法记录日志信息。
		1. <font color="#6425d0">name</font>：`name` 作为 Logger 的唯一标识，指定获取的 Logger 的层级，通常使用 **全类名** 或 **包名** 进行命名，以便于**实现层级化日志管理**
	2. `Logger` 会根据本 `Logger` 的日志级别来记录需要的日志<font color="#ff0000">（简单理解为：第一层过滤</font>）。对于低于设定级别的日志，即使你调用了日志记录方法（如 `logger.config(xxx)`），这些日志也不会被记录。
2. <font color="#00b0f0">Filter</font>：
	1. Filter（过滤器） 用于在日志记录被发送到 Handler 之前进行筛选，可以根据定制决定哪些日志信息应该被接受，哪些应被过滤掉（<font color="#ff0000">简单理解为：第二层过滤</font>）
3. <font color="#00b0f0">Handler</font>：
	1. Handler（处理器），负责接收 Logger 记录的日志，并将日志消息输出到指定目标，常见的 Handler 包括：
		1. <font color="#7030a0">ConsoleHandler</font>：日志输出到控制台
		2. <font color="#7030a0">FileHandler</font>：日志写入文件
		3. <font color="#7030a0">SocketHandler</font>：日志发送至网络端点
		4. <font color="#7030a0">MemoryHandler</font>：日志暂存于内存，满足条件时批量输出
	2. 一个 Logger 可以关联多个 Handler，以便将日志信息分发到不同的输出目标
	3. `Handler` 会根据本 `Handler` 的日志级别来输出需要的日志（<font color="#ff0000">简单理解为：第三层过滤</font>）。对于低于设定级别的日志，即使 `Logger` 将其传递过来，这些日志也不会被输出。
4. <font color="#00b0f0">Formatter</font>：
	1. Formatter（格式化组件） 定义了日志消息的输出格式。JUL 提供了两种默认的 Formatter：
		1. <font color="#7030a0">SimpleFormatter</font>：简单、易读的文本格式
		2. <font color="#7030a0">XMLFormatter</font>：以 XML 格式输出日志
	2. 你可以通过扩展 `Formatter` 类来自定义输出格式
5. <font color="#00b0f0">Level</font>
	1. Level（验证级别）定义了日志级别，常见的级别顺序为：
		1. <font color="#7030a0">OFF</font>：禁止捕获所有级别的日志信息
		2. <font color="#7030a0">SEVERE</font>：错误信息，表示严重错误或异常情况，应用可能无法继续运行
		3. <font color="#7030a0">WARNING</font>：警告信息，提示潜在的问题，但应用仍然可以继续运行
		4. <font color="#7030a0">INFO</font>：输出常规运行信息，提供系统运行状态
		5. <font color="#7030a0">CONFIG</font>：输出配置项信息
		6. <font color="#7030a0">FINE</font>：调试信息（少），详细调试时使用
		7. <font color="#7030a0">FINER</font>：调试信息（中），详细调试时使用
		8. <font color="#7030a0">FINEST</font>：调试信息（多），详细调试时使用
		9. <font color="#7030a0">ALL</font>：捕获所有级别的日志信息

---


#### 2.3、JUL 特性介绍

JUL 是一个具备层次结构的日志记录器，采用父子关系进行组织。简单来说，`com` 是 `com.example` 的父级，而 `com.example` 又是 `com.example.test` 的父级。而最顶层的 `com` 则以根日志（ `RootLogger`） 作为其父级。

JUL 具有继承的特性。如果我们没有为子级 Logger 手动设置日志级别，它会默认继承父级 Logger 的日志级别。

JUL 具有日志向上传递的特性。每当子级 Logger（如 `com.example`）记录一条日志时，首先会通过自身配置的 `Handler` 输出。随后，这条日志会向上传递至父级 Logger，并通过父级的 `Handler` 输出（不会再经过父级 Logger 的日志级别过滤）。这个向上传递过程会逐级进行，最终到达最顶层的 `RootLogger`，并通过其 `Handler` 输出。除非我们显式禁用这种向上传递行为（通过 `setUseParentHandlers(false)`），否则日志始终会沿着层次结构向上传递。

> [!NOTE] 注意事项
> 1. 由于日志是逐条向上传递的，而不是等到底层 `Handler` 输出后再整体传递，因此可能会看到类似以下的日志输出：
```
3月 12, 2025 10:47:37 上午 com.example.test.Main main
严重: severe 信息
3月 12, 2025 10:47:37 上午 com.example.test.Main main
严重: severe 信息
3月 12, 2025 10:47:37 上午 com.example.test.Main main
严重: severe 信息
3月 12, 2025 10:47:37 上午 com.example.test.Main main
警告: warning 信息
3月 12, 2025 10:47:37 上午 com.example.test.Main main
警告: warning 信息
3月 12, 2025 10:47:37 上午 com.example.test.Main main
警告: warning 信息
3月 12, 2025 10:47:37 上午 com.example.test.Main main
信息: info 信息
3月 12, 2025 10:47:37 上午 com.example.test.Main main
信息: info 信息
3月 12, 2025 10:47:37 上午 com.example.test.Main main
信息: info 信息
```

----


#### 2.4、JUL 实战步骤

##### 2.4.1、配置根日志记录器（RootLogger）

找到并修改 `${JAVA_HOME}/conf/logging.properties` 以进行相关配置：
```
# 1. 指定 Root Logger 的日志级别
.level = INFO  

# 2. 指定 Root Logger 的日志处理器（Handler）
handlers = java.util.logging.ConsoleHandler, java.util.logging.FileHandler

# 3. 对 ConsoleHandler 日志处理器进行配置
java.util.logging.ConsoleHandler.level = INFO                                    # 处理器的日志级别
java.util.logging.ConsoleHandler.formatter = java.util.logging.SimpleFormatter   # 日志的输出格式

# 4. 对 FileHandler 日志处理器进行配置
java.util.logging.FileHandler.level = ALL                  # 处理器的日志级别
java.util.logging.FileHandler.pattern = D:\\EnglishDeployment\\logs\\JUL.%g.log   # 日志文件存放位置
java.util.logging.FileHandler.limit = 10485760             # 单个日志文件最大10485760字节（10MB）
java.util.logging.FileHandler.count = 5                    # 日志文件的最大数量
java.util.logging.FileHandler.append = true                # 以追加模式
java.util.logging.FileHandler.formatter = java.util.logging.SimpleFormatter  # 日志的输出格式（默认是 XML 方式）
```

> [!NOTE] 注意事项
> 1. 如果需要将日志写入文件（即使用 `FileHandler`），需要注意以下三点：
> 	- 由于 JUL 只创建文件而不会自动生成目录，因此必须手动创建 `logs` 目录，并确保程序具有相应的读写权限。
> 	- 此外，需要注意文件路径的格式应使用 `\\` 而非 `\`
> 	- 最后，需要了解 JUL 中的日志轮换机制：JUL 将日志记录在 `JUL.0.log` 文件中。当文件写满时，它会被重命名为 `JUL.1.log`，同时创建一个新的 `JUL.0.log` 继续记录。随着日志再次写满，`JUL.1.log` 会被重命名为 `JUL.2.log`，而 `JUL.0.log` 则再次重命名为 `JUL.1.log`，这样依次进行。当文件数量达到最大限制时，最早的日志文件（如 `JUL.4.log`）将被删除，以腾出空间（该机制与 Log4J2 的轮换机制不同，Log4J2 是数字越小，文件越旧）
> 1. 建议在实际配置时移除所有注释和空格，以避免注释中隐藏的特殊字符引起解析错误，以下是纯净版本：
```
.level = INFO
  
handlers = java.util.logging.ConsoleHandler, java.util.logging.FileHandler
  
java.util.logging.ConsoleHandler.level = INFO
java.util.logging.ConsoleHandler.formatter = java.util.logging.SimpleFormatter
  
java.util.logging.FileHandler.level = ALL
java.util.logging.FileHandler.pattern = D:\\EnglishDeployment\\logs\\JUL.%g.log
java.util.logging.FileHandler.limit = 10485760
java.util.logging.FileHandler.count = 5
java.util.logging.FileHandler.append = true
java.util.logging.FileHandler.formatter = java.util.logging.SimpleFormatter
```

---


##### 2.4.2、 代码中为某个特定包或类的 Logger 单独配置（可选）

对于某些包下的代码，如果我们不希望完全沿用 Root Logger 的配置，可以在代码中手动为这些包单独设置 Logger。

例如，在下面的例子中，我们为 `com.example.Main` 层级的 Logger 进行了独立配置，这样它就不再使用 Root Logger 的日志级别，也不会将日志向上传递到 Root Logger 的 Handler 进行处理。
```
public class Main {  
    public static void main(String[] args) {  
    
        // 1. 获取 ConsoleHandler 日志处理器  
        ConsoleHandler consoleHandler = new ConsoleHandler();  
        
        // 2. 获取 SimpleFormatter 格式化组件  
        SimpleFormatter simpleFormatter = new SimpleFormatter();  
        
        // 3， 将格式化组件设置到日志处理器中  
        consoleHandler.setFormatter(simpleFormatter);  
        
        // 4. 获取 /com/example/Main 层的 Logger        
        Logger logger = Logger.getLogger(Main.class.getName());  
        
        // 5. 设置该层 Logger 的属性  
        logger.setLevel(Level.INFO);  
        logger.setUseParentHandlers(false);  // 禁用向上传递
        logger.addHandler(consoleHandler);  
        
        // 6. 调用 Logger 记录日志  
        logger.severe("severe 信息");  
        logger.warning("warning 信息");  
        logger.info("info 信息");  
        logger.config("config 信息");  
        logger.fine("fine 信息");  
        logger.finer("finer 信息");  
        logger.finest("finest 信息");  
    }  
}
```

---


##### 2.3.2、调用 JUL-Logger 记录日志
```
public class Test {  
    public static void main(String[] args) {  
		Logger logger = Logger.getLogger(Main.class.getName());   // 获取指定层级的 Logger，这里获取的是 com.example.Main 层级的 Logger（可以传入 name.class.getName()，也可以直接传入全类名字符串："com.example.Main"）
		
		logger.severe("severe 信息");  
        logger.warning("warning 信息");  
        logger.info("info 信息");  
        logger.config("config 信息");  
        logger.fine("fine 信息");  
        logger.finer("finer 信息");  
        logger.finest("finest 信息");  
    }  
}
```

> [!NOTE] 注意事项
> 1. Logger 要来自 `Java.util.Logging` 包下
> 2. 由于我们没有为 Main 层的 Logger 手动设置日志级别，它会默认继承父 Logger（`Root Logger`）的日志级别。此外，因为 Main 层 Logger 也没有单独配置自己的 Handler，所以它会将日志向上传递，并使用父 Logger（`Root Logger`）的 Handler 进行输出
> 3. JUL 还提供另外一种记录方式，允许我们动态传递参数，**从而代替拼接字符串**：
```
String name = "Ba Tian";
int age = 18;
logger.log(Level.INFO,"学生的姓名为：{0}，年龄为：{1}",new Object[]{name,age});
```

---


### 3、SLF4J  门面框架

#### 3.1、SLF4J 概述

SLF4J（Simple Logging Facade for Java）是一种日志门面框架，它并不直接进行日志实现，而是<font color="#ff0000">为各种日志框架提供统一的抽象接口（API）</font>，让开发者在应用中只需依赖于这一套 API，而无需针对不同的日志实现学习各自的专有 API

> [!NOTE] 注意事项
> 1. 如果需要配置日志框架的底层实现，应遵循底层日志实现的配置方式。SLF4J 作为日志门面，本身不具备配置能力，具体的日志管理和配置仍需在底层日志实现中完成。

---


#### 3.2、SLF4J 支持的日志级别

SLF4J 仅支持以下标准级别：
1. <font color="#00b0f0">ERROR</font>：输出错误信息，表示严重错误或异常情况，应用可能无法继续进行
2. <font color="#00b0f0">WARN</font>：输出警告信息，提示潜在的问题（推荐）
3. <font color="#00b0f0">INFO</font>：输出常规运行信息，提供系统运行状态
4. <font color="#00b0f0">DEBUG</font>：输出调试信息，比如详细的配置解析过程
5. <font color="#00b0f0">TRACE</font>：输出最详细的信息，几乎是逐行解释发生了什么

> [!NOTE] 注意事项
> 1. SLF4J 本身只是个日志门面，不提供具体的日志实现，因此他没有默认的日志级别。实际的默认日志级别取决于我们绑定的底层日志实现：
> 	- <font color="#00b0f0">JUL</font>：默认日志级别是 INFO
> 	- <font color="#00b0f0">Log4J</font>：默认日志级别是 ERROR
> 	- <font color="#00b0f0">Log4J2</font>：默认日志级别是 ERROR
> 1. 如果需要调整日志级别（如开启或关闭日志，调整默认日志级别），必须在底层日志实现中进行具体配置。

---

#### 3.3、使用 SLF4J 框架

##### 3.3.1、引入 SLF4J 相关依赖

引入 [slf4j-api 依赖](https://mvnrepository.com/artifact/org.slf4j/slf4j-api/2.0.16)
```
<!-- SLF4J 日志门面依赖 -->
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
    <version>XXXXXX</version>
</dependency>
```

---


##### 3.3.2、引入底层日志实现相关依赖

根据不同的底层日志实现，应选择相应的依赖，我们需要注意以下两点：
1. SLF4J 之前的日志框架 并未遵循 SLF4J 标准，因此无法直接引入依赖后立即使用，需要额外引入适配器
2. 而 SLF4J 之后的日志框架，如果已遵循 SLF4J 规范，则只需直接引入相关依赖即可正常使用（如 `Log4J2` 就没有遵循SLF4J 规范，仍然需要引入适配器）

在此，我们选择引入 [log4j-slf4j2-impl 依赖](https://mvnrepository.com/artifact/org.apache.logging.log4j/log4j-slf4j2-impl) 作为 Log4J2 的 SLF4J 适配器，并引入 [slf4j-api 依赖](https://mvnrepository.com/artifact/org.slf4j/slf4j-api/2.0.16)
```
<!-- Log4J2 适配器 -->
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-slf4j2-impl</artifactId>
    <version>XXXXXX</version>
</dependency>

<!-- SLF4J 日志门面依赖 -->
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
    <version>XXXXXX</version>
</dependency>
```

---


##### 3.3.3、调用 SLF4J-Logger 记录日志

```
public class Main {  
    public static void main(String[] args) {  
        Logger logger = LoggerFactory.getLogger(Main.class.getName());  // 获取指定层级的 Logger，这里获取的是 com.example.Main 层级的 Logger（可以传入 name.class.getName()，也可以直接传入全类名字符串："com.example.Main"）
        logger.trace("trace 信息");  
        logger.debug("debug 信息");  
        logger.info("info 信息");  
        logger.warn("warn 信息");  
        logger.error("error 信息");  
    }  
}
```

> [!NOTE] 注意事项
> 1. Logger 要来自 `org.slf4j` 包下
> 2. SLF4J 还提供另外一种记录方式，允许我们动态传递参数，**从而代替拼接字符串**：
```
# 基础使用
String name = "Ba Tian";
int age = 18;
logger.log(Level.INFO,"学生的姓名为：{}，年龄为：{}",name,age);


# 进阶使用
try{
	......
}
catch(ClaassNotFoudException e){
	logger.info("xxx类中的xxx方法出现了异常，请及时关注相关信息");
	logger.info("具体错误是：{}",e);
}
```

---


### 4、SLF4J + Log4J2

#### 4.1、SLF4J + Log4J2 概述

使用 SLF4J 作为日志门面框架，搭配 Log4j2 的日志实现框架，绝对是当前市场上最强大、最灵活的日志功能实现方式。

---


#### 4.2、Log4J2 组件介绍

Log4j2 主要由 <font color="#ff0000">Loggers (日志记录器)</font>、<font color="#ff0000">Appenders（输出控制器）</font>和 <font color="#ff0000">Layout（日志格式化器）</font>组成：

1. <font color="#00b0f0">Loggers</font>：
	1. Loggers（日志记录器），负责记录日志，并控制日志的输出级别（☆☆☆）
	2. 与 JUL 不同，Log4j2 中的日志信息在 Logger 中确定输出级别后，不会再经过 Handler 的再次过滤。这意味着 Logger 配置的输出级别直接决定最终的日志输出级别（☆☆☆）
2. <font color="#00b0f0">Appenders</font>：
	1. Appenders（输出控制器），负责把日志输出到指定位置，常用的 Appenders 有：
		1. <font color="#7030a0">ConsoleAppender</font>：将日志输出到控制台
		2. <font color="#7030a0">FileAppender</font>：将日志写入到单一文件中
		3. <font color="#7030a0">RollingFileAppender</font>：将日志写入到文件中，支持按大小或时间滚动
		4. <font color="#7030a0">JDBCAppender</font>：将日志存储到关系型数据库中
3. <font color="#00b0f0">Layout</font>：
	1. Layout（日志格式化器），决定了日志的格式化方式，即日志输出的具体格式，常用的 Layout 有：
		1. <font color="#7030a0">HTMLLayout</font>：输出为 HTML 表格的格式
		2. <font color="#7030a0">JSONLayout</font>：输出为 JSON 格式
		3. <font color="#7030a0">XMLLayout</font>：输出为 XML 格式
		4. <font color="#7030a0">SimpleLayout</font>：简单的日志输出格式
		5. <font color="#7030a0">PatternLayout</font>：最强大的格式化组件，可以根据自定义日志输出格式（默认格式化器）

---


#### 4.3、Log4J2 支持的日志级别

Log4J2 支持以下日志级别：
1. <font color="#00b0f0">OFF</font>：禁止捕获所有级别的日志信息
2. <font color="#00b0f0">FATAL</font>：输出严重错误信息，应用可能无法继续运行
3. <font color="#00b0f0">ERROR</font>：输出错误信息，表示严重错误或异常情况，但应用仍然可以继续运行
4. <font color="#00b0f0">WARN</font>：输出警告信息，提示潜在的问题（推荐）
5. <font color="#00b0f0">INFO</font>：输出常规运行信息，提供系统运行状态
6. <font color="#00b0f0">DEBUG</font>：输出调试信息，比如详细的配置解析过程
7. <font color="#00b0f0">TRACE</font>：输出最详细的信息，几乎是逐行解释发生了什么
8. <font color="#00b0f0">ALL</font>：捕获所有级别的日志信息


无需担心 SLF4J 日志门面框架只有五个标准级别的限制。在配置层面，我们依然可以灵活设置日志级别，这是因为适配器在 SLF4J 和 Log4j2 之间建立了一套完整的映射关系。映射关系大致如下：
![](image-20250312195258282.png)


---


#### 4.3、SLF4J + Log4J2 实战步骤

##### 4.3.1、引入 SLF4J 相关依赖

引入 [slf4j-api 依赖](https://mvnrepository.com/artifact/org.slf4j/slf4j-api/2.0.16)
```
<!-- SLF4J 日志门面依赖 -->
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
    <version>XXXXXX</version>
</dependency>
```

---


##### 4.3.2、引入 Log4J2 相关依赖

引入 [log4j-api 依赖](https://mvnrepository.com/artifact/org.apache.logging.log4j/log4j-api/2.24.3)、[log4j-core 依赖](https://mvnrepository.com/artifact/org.apache.logging.log4j/log4j-core/2.24.3)、[disruptor 依赖](https://mvnrepository.com/artifact/com.lmax/disruptor/3.4.4)、 [log4j-slf4j2-impl 依赖](https://mvnrepository.com/artifact/org.apache.logging.log4j/log4j-slf4j2-impl) 
```
<!-- Log4J2 日志门面依赖 -->
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-api</artifactId>
    <version>2.24.3</version>
</dependency>

<!-- Log4J2 核心依赖 -->
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-core</artifactId>
    <version>2.24.3</version>
</dependency>

<!-- Log4J2 异步日志依赖 -->
<dependency>
    <groupId>com.lmax</groupId>
    <artifactId>disruptor</artifactId>
    <version>XXXXXX</version>
</dependency>

<!-- Log4J2 适配器 -->
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-slf4j2-impl</artifactId>
    <version>2.24.3</version>
    <scope>test</scope>
</dependency>
```

> [!NOTE] 注意事项
> 1. `log4j-api` 是 Log4j 的日志门面，为什么在已有 SLF4J 门面的情况下仍需要引入？
> 	- 虽然 SLF4J 是一个通用日志门面，但在使用 Log4j2 作为日志实现时，SLF4J 实际上会访问调用 `log4j-api`，再由它对接 Log4j2 的具体实现
> 1. 为什么引入的是 `Log4j2` 依赖，看起来却像是在引入 `Log4j` 依赖？
> 	- 如果依赖版本号为 2.x.x，说明确实是引入了 Log4j2 的依赖

---


##### 4.3.3、配置 Log4J2

Log4j2 会自动在类路径中查找配置文件，通常我们在项目的 `/src/main/resources` 目录下创建一个名为 `log4j2.xml` 的配置文件即可，完成下述配置即可：
```
<?xml version="1.0" encoding="UTF-8"?>  
<!-- 1. 根节点 -->
<Configuration status="WARN" monitorInterval="30">  
  
    <!-- 2. 定义全局属性 -->  
    <Properties>  
        <Property name="LOG_PATTERN">%d{yyyy-MM-dd HH:mm:ss} [%t] %-5level %logger{36} - %msg%n</Property>   <!-- 自定义日志的输出格式 -->
        <Property name="LOG_PATH">D:\\EnglishDeployment\\logs</Property>   <!-- 自定义文件的保存路径 -->
    </Properties>  
  
    <!-- 3. 定义日志输出器（Appenders） -->  
    <Appenders>  
        <!-- 3.1 配置 ConsoleAppender 控制器 -->  
        <Console name="ConsoleAppender" target="SYSTEM_OUT">  
            <PatternLayout pattern="${LOG_PATTERN}"/>  <!-- 使用自定义的日志输出格式 -->
        </Console>  
  
        <!-- 3.2 配置 FileAppender 控制器 -->  
        <File name="FileAppender" fileName="${LOG_PATH}/app.log">  
            <PatternLayout pattern="${LOG_PATTERN}"/> <!-- 使用自定义的日志输出格式 -->
        </File>  
  
        <!-- 3.3 配置 RollingFileAppender 控制 -->  
        <RollingFile name="RollingFileAppender" fileName="${LOG_PATH}/rolling.log"  
                     filePattern="${LOG_PATH}/rolling-%d{yyyy-MM-dd}-%i.log">
            <PatternLayout pattern="${LOG_PATTERN}"/>  
            <Policies>  
                <SizeBasedTriggeringPolicy size="10MB"/> <!-- 文件达到 10MB 时滚动一次 -->  
                <TimeBasedTriggeringPolicy/> <!-- 每天滚动一次 -->  
            </Policies>  
            <DefaultRolloverStrategy max="7"/> <!-- 最多保留 7 个历史文件 -->  
        </RollingFile>  
    </Appenders>  
  
    <!-- 4. 配置日志记录器（Loggers） -->  
    <Loggers>  
        <!-- 4.1 为某个特定包或类的 Logger 单独配置 -->  
        <Logger name="com.example.service" level="DEBUG" additivity="false">  
            <AppenderRef ref="FileAppender"/>   <!-- 添加 FileAppender 控制器 -->
        </Logger>  
  
        <!-- 4.2 配置根日志记录器 RootLogger -->  
        <Root level="INFO">  
            <AppenderRef ref="ConsoleAppender"/>  <!-- 添加 ConsoleAppender 控制器 -->
            <AppenderRef ref="RollingFileAppender"/>  <!-- 添加 RollingFileAppender 控制器 -->
        </Root>  
    </Loggers>  
  
</Configuration>
```

1. <font color="#00b0f0">Confiuration</font>：配置根节点
	1. <font color="#7030a0">status="WARN"</font>：Log4j2 自身日志输出级别，用来输出 Log4j2 在解析和加载配置文件时的日志信息，用于调试配置文件，生产环境中一般设为 `WARN` 或更高
	2. <font color="#7030a0">monitorInterval="30"</font>：每 30 秒检查一次配置文件是否有修改，如果修改则自动加载。
2. <font color="#00b0f0">Properties</font>：配置全局属性，定义可以复用的变量，后续可以使用 `${}` 引用
3. <font color="#00b0f0">Appenders</font>：配置日志输出器
	1. <font color="#7030a0">Console</font>：对应 ConsoleAppender 控制器
		- <font color="#9bbb59">name</font>：为这个输出器命名，后续引用
		- <font color="#9bbb59">target</font>：设置输出方式
			- SYSTEM_OUT：标准输出流，用来输出正常的信息
			- SYSTEM_ERR：标准错误流，用来输出错误或警告信息，通常在控制台上显示为红色
	2. <font color="#7030a0">File</font>：对应 FileAppender 控制器
		- <font color="#9bbb59">name</font>：为这个输出器命名，后续引用
		- <font color="#9bbb59">fileName</font>：指定日志文件的路径，Log4j2 会自动创建目录和文件，无需手动创建
	3. <font color="#7030a0">RollingFile</font>：对应 RollingFileAppender 控制器
		- <font color="#9bbb59">name</font>：为这个输出器命名，后续引用
		- <font color="#9bbb59">filePattern</font>：定义日志文件滚动时的命名规则
	4. <font color="#7030a0">JDBC</font>：对应 JDBCAppender 控制器
4. <font color="#00b0f0">Loggers</font>：配置日志记录器
	1. <font color="#7030a0">Logger</font>：为某个特定包或类设置单独的日志级别
		- <font color="#9bbb59">name</font>：指定该 Logger 的层级，通常为包名或类名
		- <font color="#9bbb59">level</font>：指定日志级别
		- <font color="#9bbb59">additivity</font>：设置是否向上传递日志信息
	2. <font color="#7030a0">Root</font>：设置根日志记录器
		- <font color="#9bbb59">level</font>：指定根日志记录器的日志级别

> [!NOTE] 注意事项：滚动的流程
> 1. <font color="#00b0f0">日志写入</font>：日志会首先写入到当前的 `rolling.log` 文件
> 2. <font color="#00b0f0">滚动触发</font>：
> 	- <font color="#7030a0">按时间滚动</font>：每天，Log4j2 会检查当前日期，并根据配置的 `filePattern`（如 `rolling-%d{yyyy-MM-dd}-%i.log`）将 `rolling.log` 重命名为带有当前日期的文件（例如 `rolling-2025-03-12-0.log`）。
> 	- <font color="#7030a0">按大小滚动</font>：当 `rolling.log` 文件大小达到指定的阈值（如 10MB）时，Log4j2 会根据配置的 `filePattern` 进行滚动，生成一个新文件，通常会加上序号（例如 `rolling-2025-03-12-1.log`），再次达到指定的阈值时，会生成 `rolling-2025-3-12-2.log`（该机制与 JUL 的轮换机制不同，JUL 是数字越打，文件越旧）
> 1. <font color="#00b0f0">历史文件管理</font>：根据配置的 `max` 属性，最多只会保留 `max` 个历史文件，超出部分的旧日志文件会被删除。

> [!NOTE] 注意事项：为什么 Log4j2 的配置可以完全在配置文件中完成，而 URL 的配置需要先在配置文件中配置 RootLogger，再在代码中按需为某个特定包或类设置单独的日志级别
> 1. <font color="#00b0f0">为什么 JUL 需要两处配置</font>：
> 	- 对于某些包下的代码，如果我们不希望完全沿用 Root Logger 的配置，需要再在代码中手动为这些层级单独定义 Logger
> 	- 在 JUL 中，虽然可以在配置文件中为特定层级的 Logger 单独设置日志级别，并选择是否禁止向上传递，但无法为这些 Logger 配置独立的 Handler，它们始终依赖父 Logger 的 Handler
> 	- 这可能导致 Logger 输出的信息因父 Handler 的日志级别过滤而丢失（例如，一个 Logger 配置为 CONFIG 级别，而其父 Handler 配置为 INFO 级别，结果 CONFIG 信息被过滤掉）
> 	- 因此，必须在代码中单独定义 Logger，而不能完全依赖配置文件
> 1. <font color="#00b0f0">为什么在 Log4J2 中只需要在一处配置</font>
> 	- 在 Log4j2 中，只需在配置文件中直接为 Logger 指定所需的日志级别，日志便能直接输出，无需经过 Handler 的额外过滤

---


##### 4.4.4、配置 Log4J2 异步日志（特色）

###### 4.4.4.1、异步日志的概念

异步日志的概念是将日志记录操作交由一个独立的异步线程执行，从而实现业务逻辑和日志记录的并行处理。这种方式能够减少日志记录对主线程性能的影响，提高系统的整体响应速度，例如下面的代码，你会看到它们交替执行：
```
public class Main {  
    public static void main(String[] args) {  
        Logger logger = LoggerFactory.getLogger(Main.class.getName());  
        
        // 记录日志  
        for (int i=0;i<2000;i++) {  
            logger.trace("trace 信息");  
            logger.debug("debug 信息");  
            logger.info("info 信息");  
            logger.warn("warn 信息");  
            logger.error("error 信息");  
        }  
  
        // 业务逻辑  
        for(int i=0;i<2000;i++) {  
            System.out.println("---------------------");  
        }  
          
    }  
}
```

---


###### 4.4.4.2、配置全局异步

要将所有日志记录操作统一设置为异步模式，只需在类路径下的 `resources` 目录中创建一个名为 `log4j2.component.properties` 的属性文件（文件名必须固定），并在其中添加以下配置：
```
Log4jContextSelector=org.apache.logging.log4j.core.async.AsyncLoggerContextSelector
```

---


###### 4.4.4.3、配置混合异步（推荐）

可以在应用中同时使用同步日志和异步日志。通常是在保持根日志记录器为同步的基础上，为某个特定包或类的 Logger 单独配置为异步模式。

其方法只需要在 `log4j2.xml` 配置文件中使用 `<AsyncLogger>` 配置日志记录器（`Loggers`）即可：
```
<?xml version="1.0" encoding="UTF-8"?>  
<Configuration status="WARN" monitorInterval="30">  
  
    <Properties>  
        <Property name="LOG_PATTERN">%d{yyyy-MM-dd HH:mm:ss} [%t] %-5level %logger{36} - %msg%n</Property>   
    </Properties>  
  
    <Appenders>  
        <Console name="ConsoleAppender" target="SYSTEM_OUT">  
            <PatternLayout pattern="${LOG_PATTERN}"/> 
        </Console>  
    </Appenders>  
  
    <!-- 配置日志记录器（Loggers） -->  
    <Loggers>  
        <!-- 4.1 为某个包或类的 Logger 单独配置 -->  
        <Logger name="com.example.service" level="DEBUG" additivity="false">  
            <AppenderRef ref="FileAppender"/>   <!-- 添加 FileAppender 控制器 -->
        </Logger>  
  
        <!-- 4.2 Root Logger，应用中所有日志的默认入口 -->  
        <Root level="INFO">  
            <AppenderRef ref="ConsoleAppender"/>  <!-- 添加 ConsoleAppender 控制器 -->
            <AppenderRef ref="RollingFileAppender"/>  <!-- 添加 RollingFileAppender 控制器 -->
        </Root>  
        
        <!-- 4.3 使用 AsyncLogger 为某个包或类的 Logger 单独配置为异步模式-->
        <AsyncLogger name="com.example.async" level="INFO" additivity="false" 
			        includeLocation="false">
	        <AppenderRef ref="ConsoleAppender"/>  <!-- 添加 ConsoleAppender 控制器 -->
        </AsyncLogger>
    </Loggers>  
  
</Configuration>
```

1. <font color="#00b0f0">AsynncLogger</font>：配置异步日志记录器
	- <font color="#7030a0">name</font>：指定该 Logger 的层级，通常为包名或类名
	- <font color="#7030a0">level</font>：指定日志级别
	- <font color="#7030a0">additivity</font>：设置是否向上传递日志信息
	- <font color="#7030a0">includeLocation</font>：控制是否在日志中包含行号信息。由于获取行号信息会影响日志记录性能，建议将其设置为 `false`（表示不记录行号信息）以提升效率

---


##### 4.4.5、调用 SLF4J-Logger 记录日志

```
public class Main {  
	Logger logger = LoggerFactory.getLogger(Main.class.getName());  
	logger.trace("trace 信息");  
	logger.debug("debug 信息");  
	logger.info("info 信息");  
	logger.warn("warn 信息");  
	logger.error("error 信息");  
}
```

---



