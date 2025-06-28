---
title: 笔记：Spring WebFlux
date: 2025-05-03
categories:
  - Java
  - Spring 生态
  - Spring WebFlux
tags: 
author: 霸天
layout: post
---
# 一、理论

### 1. Spring WebFlux 概述

==1.现有问题==
随着互联网应用的日益复杂和用户对实时性要求的提高，传统的基于**阻塞 I/O** 的Web框架在处理高并发和大数据流时逐渐显现出瓶颈。


==2.响应式编程解决方案==
基于以上问题，响应式编程模型应运而生，响应式编程是一种**处理异步数据流**、**变化传播**、**非阻塞**的编程范式，在响应式编程中，数据被建模为流，操作以非阻塞方式执行，从而保持应用程序在高负载下的响应性。


==3.Spring WebFlux 是什么==
Spring WebFlux，作为 Spring Framework 的响应式Web框架，基于 Reactive Streams 规范构建，采用事件驱动 + 非阻塞 I/O 的架构设计，特别适合处理高并发、低延迟和 I/O 密集型任务。

---


### 深入理解传统 Spring Web（Servlet 栈）

Spring Web 采用传统的阻塞式编程模型，基于 Servlet API，默认使用 Servlet 容器（如 Tomcat）。其线程模型为每个请求分配一个线程，并在请求处理期间阻塞该线程，直到响应生成完毕（线程阻塞）。其缺乏内建的流式数据处理支持，且依赖多线程调度，其内存开销较大。在高并发场景下，受限于线程池大小和阻塞模型，吞吐量较低。然而，其延迟较低，适合传统的请求/响应模型，尤其是 CPU 计算密集型 或 请求阻塞不明显 的传统 Web 应用开发。

对线程阻塞的深入理解：
1. 一个请求通常由一个线程全程处理。当程序执行到某个耗时操作（如阻塞的网络请求、阻塞的数据库查询、文件读写、或 CPU 密集计算）时，这个线程会**停下来等待**，直到结果返回后才能继续往下执行。
2. 在这段等待时间里，线程既不做其他事，又无法释放，造成**严重的资源浪费**，系统吞吐量受限。
3. 虽然我们可以通过**异步线程池**来“解耦”这些耗时操作，把任务转交给其他线程执行，释放主线程，看起来像是实现“非阻塞”，但这其实只是**线程的转移**，本质上仍是阻塞，只不过阻塞发生在别的线程上。更糟的是：如果主线程还需要等待异步结果返回，等于主线程和工作线程**两个线程都被卡住了**，反而加重了系统开销。这种方式可以称为“**伪非阻塞**”。

----


### 深入理解Spring WebFlux（Reactive 栈）

==1.Spring WebFlux（Reactive 栈）概述==
Spring WebFlux 基于 **Reactive Streams 规范**构建，采用**事件驱动 + 非阻塞 I/O** 的架构设计，默认使用**非阻塞 I/O 服务器**（默认 Netty），借助 Reactor 库 **（Mono 和 Flux）** 实现异步操作，能够以较少的线程处理更多请求，资源消耗显著降低。在 I/O 密集型 场景下，其吞吐量和响应速度表现优异，尽管响应链较长可能导致延迟略高。Spring WebFlux 特别适合 高并发 和 大量 I/O 请求 的应用场景，例如聊天系统、实时通知和数据流处理等。


==2.Reactive Streams 规范==
既然 Spring WebFlux 是基于 Reactive Streams 规范构建的，我们若想真正搞懂 WebFlux，就绕不开对 Reactive Streams 规范的理解。而这套规范简单来说就是定义了四个核心接口：
1. <font color="#00b0f0">Publiser</font>：
	1. 也就是负责发布数据流的数据源。我们平时使用的 Flux 和 Mono 就是 Publisher 接口的实现类。
	2. 因此可以理解为：哪里创建了 Flux 或 Mono，哪里就是一个 Publisher。
2. <font color="#00b0f0">Subscriber</font>：
	1. 也就是消费数据流的“消费者”，负责处理接收到的数据。
	2. 在学习 Flux 和 Mono 时，我们接触了它们的 `subscribe` 方法，只有调用这个方法，数据流才会真正开始流动。所以可以理解为：哪里调用了 `subscribe` 方法，哪里就是 Subscriber。
	3. 常见的 Subscriber 示例有：
		1. WebClient：
			1. 发起请求并消费响应数据流；
		2. WebFlux 框架本身：
			1. 当调用 API 时，WebFlux 框架会在响应阶段自动订阅 Reactive 类型的返回值，将数据转换为客户端需要的格式并发送出去。
			2. 不要误以为客户端是 Subscriber，其实是 WebFlux 帮你完成了订阅和数据推送。
3. <font color="#00b0f0">Subscription</font>：
	1. 是 Publisher 与 Subscriber 之间的桥梁，负责建立连接并控制数据流速。
	2. 它可以防止 Publisher 生产太快、而 Subscriber 消费太慢，从而导致 Publisher 缓存堆积、内存暴涨，最终可能引发内存溢出。
	3. 这正是我们常说的“**背压（Backpressure）控制**”，具体我们可以看下文的被压部分。
4. <font color="#00b0f0">Processor</font>：
	1. 是一种同时实现了 Publisher 和 Subscriber 的组件，承担中间处理的角色。
	2. 它能够接收上游的数据，进行加工处理后，再将结果发布给下游。


==3.I/O 是什么==
I/O，即输入/输出（Input/Output），是指计算机与外部世界的信息交换过程，包括但不限于：
1. <font color="#00b0f0">网络请求</font>：
	1. 如调用其他 HTTP API 接口（RestTemplate、WebClient）、JDBC、R2DBC 等等
2. <font color="#00b0f0">文件操作</font>：
	1. 读取或写入硬盘上的文件
3. <font color="#00b0f0">用户交互</font>：
	1. 键盘、鼠标等输入设备的信号获取，以及向显示器等输出设备发送信息。
4. <font color="#00b0f0">硬件通信</font>：
	1. 与打印机、扫描仪等外部设备的数据交换


==4.事件驱动 + 非阻塞 I/O 是什么==
“非阻塞 I/O”与“阻塞 I/O”是一对相对概念。阻塞 I/O 指的是：主线程在执行某个依赖外部响应的操作时（比如使用 `RestTemplate` 调用 HTTP 接口，或通过 JDBC 查询数据库），会**卡在原地等待**数据返回，不能继续执行其它任务。

而非阻塞 I/O 的模式是：主线程在发起操作后不会等待结果，而是转身继续处理别的任务，等到数据返回时，再通过某种机制“通知”主线程回来继续处理。这种“通知”机制，就是事件驱动模型的核心所在（理想状态，Spring WebFlux + 异步阻塞操作、异步非阻塞操作）

举个例子，当主线程遇到一个耗时操作，如果这个操作是异步非阻塞的（比如非阻塞网络请求、非阻塞数据库查询），主线程执行之后，不会呆等结果返回，而是注册一个事件及其对应的回调函数，然后立刻去处理其他任务。一旦有数据返回，比如 R2DBC 查询返回结果，就会触发事件并执行回调。如果操作未完全完成，只返回了部分数据，主线程会继续处理其他任务，等下次数据返回后再继续回调、推进后续业务流程。

这是 Spring WebFlux 碰到异步非阻塞操作结合后展现的强大优势，但当遇到本质上是同步阻塞的操作时，情况就不同了。比如同步简单操作或者同步阻塞操作，主线程依然需要等待它们执行完才能继续处理其他任务。然而，这种做法显然不合理，因为主 Reactive 线程需要同时处理成千上万个请求。如果某个操作长时间阻塞，主线程将无法高效处理其他任务，这显然是不可接受的。<font color="#ff0000">因此，我们约定：在主 Reactive 流（如 `flatMap`、`doOnNext` 等操作符内部）中，禁止执行任何严重阻塞操作，允许出现三类操作：同步简单操作、异步阻塞操作、异步非阻塞操作</font>。

对于执行同步阻塞操作这类场景，WebFlux 推荐将其转移到一个专门的 “阻塞友好线程池” 中去执行。执行完成后再将结果返回主线程。和传统的阻塞模型不同的是，虽然线程池中的线程仍是阻塞的，但主线程不会被挂起，而是继续处理其他任务，等结果回来后再处理该请求。这有效避免了主 Reactive 线程的长时间占用，使其保持轻量、高效。

这样一来，除了主 Reactive 线程在执行同步简单操作时略有耗时，其它包括异步阻塞、异步非阻塞的操作都不会阻塞主线程。主线程可以继续处理更多任务，从而实现以少量线程支撑大量并发请求的目标。

综上所述，我们可以总结出以下规律：
1. Spring WebFlux 虽然提供了非阻塞的能力，但能否真正实现非阻塞，取决于你的技术栈：
	1. Spring WebFlux + 异步非阻塞操作 = 完全非阻塞操作（王炸组合、理想组合）
	2. Spring WebFlux + 异步阻塞操作 = 主线程非阻塞、异步线程池阻塞（虽然稍逊，但比同步阻塞要强的多）
	3. Spring WebFlux + 同步简单操作 = 主线程阻塞（主线程短暂阻塞（通常可接受）
	4. Spring WebFlux + 同步阻塞操作 = 主线程严重阻塞主线程严重阻塞（不可接受，必须避免）
2. 我们约定：在主 Reactive 流（如 `flatMap`、`doOnNext` 等操作符内部）中，禁止执行任何严重阻塞操作，允许出现三类操作：同步简单操作、异步阻塞操作、异步非阻塞操作


==5.非阻塞 I/O 服务器==
其实，Spring WebFlux 程序并非只能运行在非阻塞 I/O 服务器上，它同样也可以部署在 Servlet 容器中。虽然在不同类型的服务器上，开发者编写的代码几乎一致，但它们在底层的执行机制上却截然不同，这一点我们必须清楚。

在上文中我们提到，Spring WebFlux 的主线程在遇到耗时操作时会“放下”当前任务，去处理其他请求，看起来好像是 WebFlux 本身很智能，但其实这并非 WebFlux 的功劳，而是它底层所依赖的非阻塞 I/O 服务器（如 Netty）在发挥作用。只有在使用非阻塞服务器时，请求才能由单线程事件循环异步处理，从而实现真正的高并发与资源高效利用。

然而，如果将 WebFlux 部署在传统的 Servlet 容器上（例如 Tomcat），底层仍旧是“一请求一线程”的处理模型。这种线程池的方式违背了响应式架构的核心理念。以如下示例代码为例：
```
@RestController
public class MyController {
    
    @GetMapping("/flux")
    public Mono<String> flux() { 
        return Mono.just("WebFlux");
    }
}
```
尽管我们在接口中返回了 `Mono<String>`，但 Tomcat 并不具备真正的响应式、非阻塞能力，它仅通过 Servlet 3.1 的 Async API 对“异步”进行了模拟，流程上还是阻塞的。这种模拟虽可正常运行，但性能上无法真正发挥响应式编程的优势。

下图展示了 Servlet 与非阻塞服务器在模型上的根本差异：

| 对比点    | 阻塞 I/O（如 Tomcat） | 非阻塞 I/O（如 Netty）  |
| ------ | ---------------- | ----------------- |
| 请求处理模型 | 每个请求对应一个线程       | 单线程事件循环，异步处理所有连接  |
| I/O 行为 | 等数据可用或写入完成前线程阻塞  | 不等，先注册事件，数据到来再处理  |
| 并发连接能力 | 线程资源受限（线程多了开销大）  | 资源消耗极低，能处理非常多连接   |
| 内存压力   | 线程多了容易爆内存        | 单线程或少量线程，内存更可控    |
| 编程模型   | 同步、阻塞调用          | 响应式、异步调用          |
| 响应式支持  | 无响应式支持，流程阻塞      | 原生响应式，支持流式数据处理    |
| 吞吐量    | 中等，受限于线程池        | 高，利用非阻塞机制提高并发处理能力 |
| 延迟     | 低，线程直通，响应快速      | 略高，响应链长，调度复杂      |

此外，在高并发场景下，Spring WebFlux 在不同服务器上的性能差异也非常明显：

| 服务器    | 最大并发量（4核8G） | 最大并发量（4核16G） |
| ------ | ----------- | ------------ |
| Tomcat | 5000 连接     | 10000 连接     |
| Netty  | 50000 连接    | 100000 连接    |


==6.Mono 和 Flux==
`Mono` 和 `Flux` 是 Reactor 的核心组件，Spring WebFlux 强烈建议我们所有的方法返回值都使用 `Mono` 或 `Flux`。简单来说，如果你希望执行异步非阻塞操作，就必须将其封装在 `Mono` 或 `Flux` 中，从而在响应式流中执行。

需要注意的是，即便你没有显式返回 `Mono` 或 `Flux`，比如直接写 `return "data"`，Spring WebFlux 也会自动将你的返回值包装成 `Mono` 或 `Flux`。虽然这样看似返回了 `Mono` 或 `Flux`，但这通常意味着你的代码逻辑仍是**同步阻塞的**（因为你无法使用异步非阻塞操作），无法真正利用 Spring WebFlux 提供的非阻塞能力和异步执行的优势，只是将结果进行了一层华丽的包装。例如，下面的代码就是一个典型的阻塞调用：
```
@RestController
public class UserController {

    @GetMapping("/users/lazy")
    public List<User> getUsersLazily() {
        List<User> allUsers = new ArrayList<>();
        int pageSize = 10;
        for (int page = 0; page < 100; page++) {
            int offset = page * pageSize;
            List<User> pageUsers = jdbcTemplate.query(
                "SELECT * FROM users LIMIT ? OFFSET ?",
                new Object[]{pageSize, offset},
                (rs, rowNum) -> new User(
                    rs.getInt("id"),
                    rs.getString("name")
                )
            );
            if (pageUsers.isEmpty()) {
                break;
            }
            allUsers.addAll(pageUsers);
        }

        return allUsers;
    }
}
```
如上所示，尽管 Spring WebFlux 会将返回值包装成 `Mono` 或 `Flux`，但由于内部仍使用了阻塞式的 `JDBC`，因此代码的执行依然是同步阻塞的，未能发挥出 WebFlux 的非阻塞特性。

<font color="#ff0000">因此，我们约定：在 Spring WebFlux 中，所有方法的返回值应当返回 `Mono` 或 `Flux`，确保所有操作都在响应式流中异步执行</font>。

---






















### 2. 对 Spring WebFlux 深入理解

==1.Spring Web（Servlet 栈）==
Spring Web 采用传统的阻塞式编程模型，基于 Servlet API，默认使用 Servlet 容器（如 Tomcat）。其线程模型为每个请求分配一个线程，并在请求处理期间阻塞该线程，直到响应生成完毕（线程阻塞）。其缺乏内建的流式数据处理支持，且依赖多线程调度，其内存开销较大。在高并发场景下，受限于线程池大小和阻塞模型，吞吐量较低。然而，其延迟较低，适合传统的请求/响应模型，尤其是 CPU 计算密集型 或 请求阻塞不明显 的传统 Web 应用开发。

对线程阻塞的深入理解：
1. 一个请求通常由一个线程全程处理。当程序执行到某个耗时操作（如阻塞的网络请求、阻塞的数据库查询、文件读写、或 CPU 密集计算）时，这个线程会**停下来等待**，直到结果返回后才能继续往下执行。
2. 在这段等待时间里，线程既不做其他事，又无法释放，造成**严重的资源浪费**，系统吞吐量受限。
3. 虽然我们可以通过**异步线程池**来“解耦”这些耗时操作，把任务转交给其他线程执行，释放主线程，看起来像是实现“非阻塞”，但这其实只是**线程的转移**，本质上仍是阻塞，只不过阻塞发生在别的线程上。更糟的是：如果主线程还需要等待异步结果返回，等于主线程和工作线程**两个线程都被卡住了**，反而加重了系统开销。这种方式可以称为“**伪非阻塞**”。


==2.Spring WebFlux（Reactive 栈）==
Spring WebFlux 基于 Reactive Streams 规范构建，采用事件驱动 + 非阻塞 I/O 的架构设计，使用非阻塞 I/O 服务器（默认 Netty），借助 Reactor 库（Mono 和 Flux）实现异步操作，能够以较少的线程处理更多请求，资源消耗显著降低。在 I/O 密集型 场景下，其吞吐量和响应速度表现优异，尽管响应链较长可能导致延迟略高。Spring WebFlux 特别适合 高并发 和 大量 I/O 请求 的应用场景，例如聊天系统、实时通知和数据流处理等。

对非阻塞 I/O 的深入理解：
1. Spring WebFlux 的非阻塞 I/O 模型有所不同。它不是通过线程切换来“回避阻塞”，而是以事件驱动 + 回调机制的方式从根本上规避阻塞。需要特别注意的是：“<font color="#ff0000">Spring WebFlux 仅提供非阻塞的能力，是否真正实现非阻塞，取决于你的整体技术栈，简单来说，Spring WebFlux + 异步非阻塞操作 = 王炸</font>”。
2. 举个例子，当主线程遇到一个耗时操作，如果这个操作是异步非阻塞的（比如非阻塞网络请求、非阻塞数据库查询），主线程不会呆等结果返回，而是注册一个事件及其对应的回调函数，然后立刻去处理其他任务。一旦有数据返回，比如 R2DBC 查询返回结果，就会触发事件并执行回调。如果操作未完全完成，只返回了部分数据，主线程会继续处理其他任务，等下次数据返回后再继续回调、推进后续业务流程。
3. 但如果这个操作是同步简单操作或同步阻塞操作，主线程仍然必须等它执行完才能继续。要注意的是，主 Reactive 线程承担着成千上万个请求，如果它被某个操作长时间阻塞，显然是不可接受的。因此，<font color="#ff0000">我们约定：在主 Reactive 流（如 `flatMap`、`doOnNext` 等操作符内部）中，禁止执行任何阻塞操作，只允许出现两类操作：同步简单操作、异步非阻塞操作</font>。
4. 那么问题来了，如果必须执行同步阻塞操作怎么办？对于这类场景，WebFlux 支持将其转移到一个专门的“阻塞友好线程池”中去执行。执行完成后再将结果返回主线程。和传统的阻塞模型不同的是，虽然线程池中的线程仍是阻塞的，但主线程不会被挂起，而是继续处理其他任务，等结果回来后再处理该请求。这有效避免了主 Reactive 线程的长时间占用，使其保持轻量、高效。
5. 这样一来，除了主 Reactive 线程在执行同步简单操作时略有耗时，其它包括异步阻塞、异步非阻塞的操作都不会阻塞主线程。主线程可以继续处理更多任务，从而实现以少量线程支撑大量并发请求的目标。


==3.两者对比表==

| 项目              | Spring Web（Tomcat）                | Spring WebFlux（Netty）          |
| --------------- | --------------------------------- | ------------------------------ |
| **并发模型**        | 每请求一个线程（阻塞）                       | 单线程事件驱动，非阻塞处理                  |
| **适用场景**        | 计算密集型 / 阻塞不明显的中低并发场景              | 高并发、I/O 密集、实时响应                |
| **资源占用**        | 多线程调度，线程上下文开销大                    | 少量线程处理大量请求，资源占用低               |
| **响应式支持**       | 无响应式支持，流程阻塞                       | 原生响应式，支持流式数据处理                 |
| **返回 JSON**     | 使用 Jackson + HttpMessageConverter | 使用 Jackson + HttpMessageWriter |
| **编程风格**        | 命令式，流程清晰，开发简单                     | 响应式，链式调用，开发思维门槛高               |
| **学习曲线**        | 低，传统经验易迁移                         | 高，需要理解响应式思想                    |
| **吞吐量**         | 中等，受限于线程池                         | 高，利用非阻塞机制提高并发处理能力              |
| **延迟（Latency）** | 低，线程直通，响应快速                       | 响应链长，调度复杂，延迟略高                 |

---


### 3. 对 “流” 深入理解

Reactor、Spring WebFlux 等框架的核心就是通过返回流（`Mono` 或 `Flux`）来管理异步操作的控制流。即便你在内部做的是阻塞操作，返回一个 `Mono` 或 `Flux` 也能保持与其他异步操作的协作性和合适的线程调度。

也就是，能返回流就返回流，不管你是执行异步阻塞操作，还是异步非阻塞









在 Spring WebFlux 中，真正的流式处理主要涉及两个方面：
1. ==服务器接受流式数据==：
	1. 通常涉及数据库查询、网络请求等外部数据源返回数据给服务器。对于这类操作，我们不能依赖同步阻塞或异步阻塞机制。比如，JDBC 会一次性拉取所有数据并返回，HttpClient 调用外部服务时也会等待所有响应返回，这种方式属于“阻塞”模式，不适合流式数据处理。
	2. 为了实现真正的流式输出，我们需要采用异步非阻塞操作。例如，R2DBC 以逐行的方式从数据库返回数据，WebClient 也采用类似的机制。这也是为什么我们说：“Spring WebFlux + 异步非阻塞操作 = 王炸”。 
2. ==服务器发送流式数据给客户端==：
	1. 默认情况下，Spring WebFlux 的返回并不是流式的，要实现真正的流式响应，必须满足以下四个条件：
	2. 使用 `Flux<T>` 返回多个元素
	3. `Flux` 必须懒生成（☆☆☆）
	4. 设置正确的 `Content-Type`（例如 `text/event-stream` 或 `application/stream+json`）
	5. 客户端需要支持 chunked transfer 或 SSE（服务器推送事件）
	6. 如果缺少任何一个条件，流式效果无法实现。例如，即使你使用了 `Flux`、懒加载，并且客户端支持，但如果没有正确设置 `Content-Type`，客户端会一直处于加载状态，直到最终一次性返回所有数据，这时数据会“哐哐”全部输出。

> [!NOTE] 注意事项
> 1. <font color="#00b0f0">懒加载</font>：
> 	1. Flux 或 Mono 在没有人订阅（subscribe）之前，它**啥都不干**
> 	2. 只有当你调用 `.subscribe()` 的时候，它才真的发数据，这叫懒加载
> 2. <font color="#00b0f0">懒生成</font>：
> 	1. 数据不是一次性准备好的，而是**边订阅边生成**的。
> 	2. 以 `Flux.just("A", "B", "C")` 为例，一旦你订阅，虽然会按照 A -> B -> C 依次发出，但它把所有数据都已经准备好了，不是懒生成。
> 	3. 再以 `Flux.interval(Duration.ofSeconds(1))` 为例，是边订阅边生成的：它每秒生成一个数，是真·懒生成。


Flux 依次发出数据

---


### 4. 高并发环境下 WebFlux 性能表现



----


### 5. Mono

#### 5.1. Mono 概述

`Mono` 表示一个可能包含 0 或 1 个元素 的异步数据流。它可以承载一个元素或一个空值，例如下面代码中， `Hello, World!` 就是一个元素；
```
Mono<String> mono = Mono.just("Hello, World!");
```

> [!NOTE] 注意事项
> 1. “元素”的概念是基于 `Mono<T>` 中的泛型类型来理解的。比如 `Mono<String>` 中一个 `String` 类型就是一个元素；如果是 `Mono<Object>`，那一个 `Object` 类型就是一个元素。

----


#### 5.2. Mono 相关方法

##### 5.2.1. 前言

我们约定：在主 Reactive 流（如 `flatMap`、`doOnNext` 等操作符内部）中，不得放置任何阻塞操作，只允许包含两类操作：
1. 同步简单操作
2. 异步非阻塞操作

对于 同步阻塞操作，必须通过 `mono.subscribeOn` 将其调度到 “阻塞友好线程” 中，因此，以后我们提到 “异步阻塞操作” ，实际上是指：**将原本是同步阻塞的操作，调度到阻塞友好线程中执行**，从而避免阻塞 Reactor 的工作线程，保持响应式流的非阻塞特性。

-----


##### 5.2.2. Mono 实例创建方法

1. `Mono.just(T data)`：
	1. 从一个确定且非 `null` 的值创建 Mono（可以为空字符串 `""`），直接发出该值
2. `Mono.justOrEmpty(T data)`：
	1. 从一个可能为 `null` 的值创建 Mono，若为 `null` 则返回空 Mono
3. `Mono.empty()`：
	1. 创建不发出任何元素且立即完成的空 Mono
4. `Mono.fromCallable(Callable<R>)`：
	1. 从 `Callable` 创建 Mono，无论 `Callable` 返回值是什么，最终都将被包装为一个 Mono，适用于封装同步简单操作、异步阻塞操作、异步非阻塞操作
	2. 这里异步阻塞操作指的是结合 `subscribeOn` 异步化阻塞操作（使成为异步阻塞操作），避免占用主 Reactive 线程过多时间
```
@RestController
@RequestMapping("/api")
public class MonoExamples {

    // 1. Mono.just(T data)
    @GetMapping("/just")
    public Mono<String> just() {
        Mono<String> mono = Mono.just("Hello");
        return mono;
    }

    // 2. Mono.justOrEmpty(T data)
    @GetMapping("/justOrEmpty")
    public Mono<String> justOrEmpty(@RequestParam(required = false) String input) {
        Mono<String> mono = Mono.justOrEmpty(input);
        return mono;
    }

    // 3. Mono.empty()
    @GetMapping("/empty")
    public Mono<String> empty() {
        Mono<String> mono = Mono.empty();
        return mono;
    }

    // 4. Mono.fromCallable(Callable<R>)
    @GetMapping("/fromCallable")
    public Mono<String> fromCallable() {
        Mono<String> mono = Mono.fromCallable(() -> {
	        // 模拟阻塞操作
            Thread.sleep(1000); 
            return "data from callable";
        }).subscribeOn(Schedulers.boundedElastic());
        return mono;
    }
}
```

---


##### 5.2.3. mono 对象实例方法

###### 5.2.3.1. 实例方法一览图

| **实例方法**                                                                                          | **功能描述**                                                                                                                                      |
| ------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| **转换与映射方法**                                                                                       |                                                                                                                                               |
| `map(Function<? super T, ? extends R> mapper)`                                                    | 对流中每个元素应用提供的函数（同步简单操作），将它从类型 T 变换成类型 R。<br>与 `flatMap` 不同点在于：`map` 是同步操作，结果通常是原始类型转换，没有异步流。                                                   |
| `flatMap(Function<? super T, ? extends Publisher<? extends R>> mapper)`                           | 对每个元素执行一个异步操作（异步阻塞、异步非阻塞），返回一个新的异步流，并将所有这些异步流与元素合并成一个流。<br>“展平”的意思是把多个 Publisher 的流合成一个统一的 Flux。                                              |
| `cast(Class<R> clazz)`                                                                            | 将流中元素强制转换为类型 R；若元素类型与 R 无继承或实现关系（即类型不兼容），将抛出 `ClassCastException` 异常。                                                                         |
| `ofType(Class<U> clazz)`                                                                          | 仅保留属于指定类型 U 的元素，过滤掉其他类型的元素。                                                                                                                   |
| `switchIfEmpty(Publisher<? extends T> alternate)`                                                 | 如果 Mono 为空（没有发出任何元素就完成），则切换到指定的备用 Publisher（如另一个 Flux 或 Mono）继续发出元素。<br>                                                                      |
| `defaultIfEmpty(T defaultValue)`                                                                  | 如果 Mono 为空（没有发出任何元素就完成），则发出指定的默认值（可以是流，也可以不是流）并完成。<br>与 `switchIfEmpty` 不同点在于：`defaultIfEmpty` 提供的是一个固定的同步值，而不是另一个异步流。适用于简单场景，如为空时返回默认字符串或对象。 |
| **过滤方法**                                                                                          |                                                                                                                                               |
| `filter(Predicate<? super T> predicate)`                                                          | 对每个元素执行断言函数，仅保留返回 `true` 的元素，返回 `false` 的元素将被过滤掉（满足条件的保留）。                                                                                    |
| **合并与组合方法**                                                                                       |                                                                                                                                               |
| `zipWith(Mono<? extends U>other，BiFunction<T，U，R>combiner)`                                       | 将当前 `Mono<T>` 与另一个 `Mono<U>` 组合，通过 `BiFunction` 合并为结果 `R`，返回 `Mono<R>`<br/>任一 Mono 为空则返回空 Mono                                                |
| `zip(Mono<? extends T1> m1, Mono<? extends T2> m2, ...)`                                          | 同上，打包多个 Mono 的结果为 `Tuple`，返回 `Mono<TupleN>`<br/>任一 Mono 为空则返回空 Mono                                                                           |
| **错误处理方法**                                                                                        |                                                                                                                                               |
| `onErrorReturn(T fallback)`                                                                       | 一旦上游发生错误，就发出一个默认值（可以是流也可以不是流），不再传递错误信号。                                                                                                       |
| `onErrorMap(Function<Throwable, ? extends Throwable> mapper)`                                     | 捕获异常并将其包装或转换为另一种更具语义的异常类型，然后重新发出错误信号，便于统一上层错误处理策略。简单来说：异常 A -> 异常 X，上层统一处理异常 X。                                                               |
| `onErrorResume(Function<Throwable, ? extends Publisher<? extends T>> fallback)`                   | 错误发生时，切换到备用的 Publisher 继续发出元素，避免整个流被中断，可用于降级、重试或备用数据源。                                                                                        |
| `onErrorContinue(BiConsumer<Throwable, Object> errorConsumer)`                                    | 对单个元素处理出错时只执行指定副作用（如记录日志），并跳过该元素继续后续处理，下游不会收到错误；相当于「有容错的流水线」。                                                                                 |
| `retry(long times)`                                                                               | 出错时将重试整个 Mono 上游 `times` 次（总共尝试 1 + times 次）。                                                                                                 |
| **延迟与调度方法**                                                                                       |                                                                                                                                               |
| `delayElements(Duration delay)`                                                                   | 给每个元素之间插入固定延迟，模拟节流或人为降速，就像每隔一秒钟才放一个包裹。                                                                                                        |
| `subscribeOn(Scheduler scheduler)`                                                                | 指定整个订阅过程在哪个调度器线程上运行（影响源的执行线程）<br>注意：多个 `subscribeOn` 只会生效第一个                                                                                  |
| `publishOn(Scheduler scheduler)`                                                                  | 指定 `publishOn` 之后的操作在那个线程池执行                                                                                                                  |
| `timeout(Duration timeout)`                                                                       | 设置超时时间，如果某个元素未在规定时间内到达，就抛出 `TimeoutException`，用于超时控制                                                                                          |
| **其他实用方法**                                                                                        |                                                                                                                                               |
| `doOnNext(Consumer<? super T> onNext)`                                                            | 每当流发出一个元素时执行指定副作用（如日志、指标统计），但不改变元素本身；就像在传送带上做一次检查打标。                                                                                          |
| `doOnError(Consumer<? super Throwable> onError)`                                                  | 流遇到错误时执行副作用（如记录日志、告警），然后再把错误继续抛给下游；适合在错误点插入埋点或监控。                                                                                             |
| `doOnComplete(Runnable onComplete)`                                                               | 在流正常走完所有元素后执行一次副作用，如资源释放或完成通知；就像闭环操作中的收尾仪式。                                                                                                   |
| `doOnSubscribe(Consumer<? super Subscription> onSubscribe)`                                       | 订阅者发起订阅时触发，用于记录或处理初次订阅行为，比如打日志或初始化状态。                                                                                                         |
| `doFinally(Consumer<SignalType> onFinally)`                                                       | 在流终止时（无论正常完成、取消、还是出错）都执行指定副作用，相当于 finally 块，常用于收尾清理等场景。                                                                                       |
| `log()`                                                                                           | 自动在控制台打印 Mono 的信号轨迹，包括订阅、请求、发出元素、完成和错误，帮你像放监控摄像头一样随时看流水线状态。                                                                                   |
| `cache()`                                                                                         | 将流转换为可重放的热流，缓存所有元素，后续订阅者会立刻看到之前的数据，无需再次执行上游逻辑；适合昂贵操作的结果复用。                                                                                    |
| `share()`                                                                                         | 也会把 Cold Mono 变成 Hot Mono，但不缓存旧数据，所有订阅者共享同一份数据流，适合广播式分发；就像把演唱会直播送给多个观众同时观看。                                                                   |
| **订阅方法**                                                                                          |                                                                                                                                               |
| `subscribe()`                                                                                     | 触发流的执行，**不处理任何信号**（元素、错误、完成），                                                                                                                 |
| `subscribe(Consumer<? super T> onNext)`                                                           | 触发流的执行，只处理正常发出的元素，不管错误或完成                                                                                                                     |
| `subscribe(Consumer<? super T> onNext, Consumer<? super Throwable> onError, Runnable onComplete)` | 触发流的执行，分别处理元素、错误、完成 三种信号                                                                                                                      |

---


###### 5.2.3.2. 转换与映射方法

```
@RestController
@RequestMapping("/api")
public class MonoOperators {
	
    // 1. map(Function<T, R> mapper)
    @GetMapping("/map")
    public Mono<String> map() {
        Mono<String> mono = Mono.just("hello")
                                 .map(value -> value.toUpperCase()); // 转为大写
        return mono;
    }

    // 2. flatMap(Function<T, Mono<R>> mapper)
    @GetMapping("/flatMap")
    public Mono<String> flatMap() {
        Mono<String> mono = Mono.just("hello")
                                 .flatMap(value -> Mono.just(value + " world"));
        return mono;
    }

    // 3. cast(Class<R> type)
    @GetMapping("/cast")
    public Mono<Number> cast() {
        Mono<Number> mono = Mono.just(123)
                                 .cast(Number.class); 
        return mono;
    }

    // 4. ofType(Class<R> type)
	@GetMapping("/ofType")  
	public Mono<String> ofType() {  
	    Mono<String> mono = Mono.just("I'm a string")  
	            .ofType(String.class);  
	    return mono;  
	}

    // 5. switchIfEmpty(Mono<? extends T> fallback)
    @GetMapping("/switchIfEmpty")
    public Mono<String> switchIfEmpty() {
        Mono<String> mono = Mono.empty()
                                .switchIfEmpty(Mono.just("Fallback value")); 
        return mono;
    }

    // 6. defaultIfEmpty(T defaultValue)
    @GetMapping("/defaultIfEmpty")
    public Mono<String> defaultIfEmpty() {
        Mono<String> mono = Mono.<String>empty()
                                 .defaultIfEmpty("Default value"); 
        return mono;
    }
}
```

---


###### 5.2.3.3. 过滤方法

```
@RestController  
@RequestMapping("/api")  
public class ReactiveExample {  
  
    // 1. filter(Predicate<? super T> predicate)  
    @GetMapping("/filter")  
    public Mono<String> filterExample() {  
        return Mono.just("admin")  
                .filter(role -> role.equals("admin"))  // 等于 admin 才能通过
                .map(r -> "权限验证通过：" + r)  
                .defaultIfEmpty("无权限，拒绝访问");  
    }  
  
}
```

---


###### 5.2.3.4. 合并与组合方法

```
@RestController  
@RequestMapping("/api")  
public class ReactiveExample {  
  
    // 1. zipWith(Mono<? extends U>other，BiFunction<T，U，R>combiner)  
    @GetMapping("/zipWith")  
    public Mono<String> zipWithExample() {  
        Mono<String> usernameMono = Mono.just("Alice");  
        Mono<Integer> ageMono = Mono.just(30);  
  
        return usernameMono.zipWith(ageMono, (name, age) -> name + " 的年龄是 " + age); // 输出：Alice 的年龄是 30    }  
  
    // 2. zip(Mono<? extends T1> m1, Mono<? extends T2> m2, ...)  
    @GetMapping("/zip")  
    public Mono<String> zipExample() {  
        Mono<String> name = Mono.just("Bob");  
        Mono<Integer> score = Mono.just(95);  
        Mono<String> level = Mono.just("A");  
  
        return Mono.zip(name, score, level)  
                .map(tuple -> {  
                    String n = tuple.getT1();  
                    Integer s = tuple.getT2();  
                    String l = tuple.getT3();  
                    return n + " 的成绩是 " + s + "，评级为 " + l;  
                });  
    }  
}
```

---


###### 5.2.3.5. 错误处理方法

```
@RestController
@RequestMapping("/api")
public class ErrorHandlingExamples {

    // 1. onErrorReturn(T fallback)
    @GetMapping("/onErrorReturn")
    public Mono<String> onErrorReturnExample() {
        return Mono.<String>error(new RuntimeException("出错了"))
                .onErrorReturn("返回默认值"); // 一旦发生错误，发出默认值
    }

    // 2. onErrorMap(Function<Throwable, ? extends Throwable> mapper)
    @GetMapping("/onErrorMap")
    public Mono<String> onErrorMapExample() {
        return Mono.<String>error(new IllegalArgumentException("原始异常"))
                .onErrorMap(e -> new CustomAppException("转换后的业务异常", e)); // 异常转换
    }

    // 3. onErrorResume(Function<Throwable, ? extends Publisher<? extends T>> fallback)
    @GetMapping("/onErrorResume")
    public Mono<String> onErrorResumeExample() {
        return Mono.<String>fromCallable(() -> {
                    throw new RuntimeException("数据库挂了");
                })
                .onErrorResume(e -> {
                    System.out.println("发生错误，切换数据源：" + e.getMessage());
                    return Mono.just("来自备用数据源的数据");
                });
    }

    // 4. onErrorContinue(BiConsumer<Throwable, Object> errorConsumer)
    @GetMapping("/onErrorContinue")
    public Flux<String> onErrorContinueExample() {
        return Flux.just("A", "B", "error", "C")
                .map(value -> {
                    if ("error".equals(value)) {
                        throw new RuntimeException("处理失败：" + value);
                    }
                    return value.toLowerCase();
                })
                .onErrorContinue((e, val) ->
                        System.out.println("跳过出错元素：" + val + "，错误：" + e.getMessage()));
    }

    // 5. retry(long times)
    @GetMapping("/retry")
    public Mono<String> retryExample() {
        return Mono.<String>fromCallable(() -> {
                    System.out.println("尝试一次");
                    throw new RuntimeException("临时异常");
                })
                .retry(3) // 总共尝试4次
                .onErrorReturn("尝试失败，返回默认");
    }
}

```

---


###### 5.2.3.6. 延迟与调度方法

```
@RestController
@RequestMapping("/api")
public class ReactiveControlOperators {

    // 1. delayElements(Duration delay)
    @GetMapping("/delay")
    public Flux<String> delayExample() {
        return Flux.just("A", "B", "C")
                .delayElements(Duration.ofSeconds(1))  // 每个元素延迟 1 秒发出
                .map(s -> "发出元素：" + s);
    }

    // 2. subscribeOn(Scheduler scheduler)
    @GetMapping("/subscribeOn")
    public Mono<String> subscribeOnExample() {
        return Mono.fromCallable(() -> {
                    System.out.println("执行线程：" + Thread.currentThread().getName());
                    return "任务完成";
                })
                .subscribeOn(Schedulers.boundedElastic()); // 指定任务在线程池中执行
    }

    // 3. publishOn(Scheduler scheduler)
    @GetMapping("/publishOn")
    public Mono<String> publishOnExample() {
        return Mono.just("原始数据")
                .doOnNext(data -> System.out.println("步骤1线程：" + Thread.currentThread().getName()))
                .publishOn(Schedulers.parallel())  // 后续操作切换线程
                .map(data -> {
                    System.out.println("步骤2线程：" + Thread.currentThread().getName());
                    return "处理后的数据：" + data;
                });
    }

    // 4. timeout(Duration timeout)
    @GetMapping("/timeout")
    public Mono<String> timeoutExample() {
        return Mono.delay(Duration.ofSeconds(3)) // 故意延迟
                .map(time -> "数据来了")
                .timeout(Duration.ofSeconds(2))  // 设置超时时间
                .onErrorReturn("请求超时，返回默认值");
    }
}

```

---


###### 5.2.3.7. 其他实用方法

| `doOnNext(Consumer<? super T> onNext)`                      | 每当流发出一个元素时执行指定副作用（如日志、指标统计），但不改变元素本身；就像在传送带上做一次检查打标。                        |
| ----------------------------------------------------------- | --------------------------------------------------------------------------- |
| `doOnError(Consumer<? super Throwable> onError)`            | 流遇到错误时执行副作用（如记录日志、告警），然后再把错误继续抛给下游；适合在错误点插入埋点或监控。                           |
| `doOnComplete(Runnable onComplete)`                         | 在流正常走完所有元素后执行一次副作用，如资源释放或完成通知；就像闭环操作中的收尾仪式。                                 |
| `doOnSubscribe(Consumer<? super Subscription> onSubscribe)` | 订阅者发起订阅时触发，用于记录或处理初次订阅行为，比如打日志或初始化状态。                                       |
| `doFinally(Consumer<SignalType> onFinally)`                 | 在流终止时（无论正常完成、取消、还是出错）都执行指定副作用，相当于 finally 块，常用于收尾清理等场景。                     |
| `log()`                                                     | 自动在控制台打印 Mono 的信号轨迹，包括订阅、请求、发出元素、完成和错误，帮你像放监控摄像头一样随时看流水线状态。                 |
| `cache()`                                                   | 将流转换为可重放的热流，缓存所有元素，后续订阅者会立刻看到之前的数据，无需再次执行上游逻辑；适合昂贵操作的结果复用。                  |
| `share()`                                                   | 也会把 Cold Mono 变成 Hot Mono，但不缓存旧数据，所有订阅者共享同一份数据流，适合广播式分发；就像把演唱会直播送给多个观众同时观看。 |

```
@RestController
@RequestMapping("/api")
public class SideEffectOperators {

    // 1. doOnNext(Consumer<? super T> onNext)
    @GetMapping("/doOnNext")
    public Flux<String> doOnNextExample() {
        return Flux.just("A", "B", "C")
                .doOnNext(val -> System.out.println("元素通过检查：" + val))
                .map(String::toLowerCase);
    }

    // 2. doOnError(Consumer<? super Throwable> onError)
    @GetMapping("/doOnError")
    public Mono<String> doOnErrorExample() {
        return Mono.<String>error(new RuntimeException("模拟错误"))
                .doOnError(err -> System.out.println("记录错误日志：" + err.getMessage()));
    }

    // 3. doOnComplete(Runnable onComplete)
    @GetMapping("/doOnComplete")
    public Flux<String> doOnCompleteExample() {
        return Flux.just("1", "2", "3")
                .doOnComplete(() -> System.out.println("流处理完毕，收工！"));
    }

    // 4.doOnSubscribe(Consumer<? super Subscription> onSubscribe)
    @GetMapping("/doOnSubscribe")
    public Mono<String> doOnSubscribeExample() {
        return Mono.just("准备启动流程")
                .doOnSubscribe(sub -> System.out.println("收到订阅请求，开始处理"));
    }

    // 5. doFinally(Consumer<SignalType> onFinally)
    @GetMapping("/doFinally")
    public Mono<String> doFinallyExample() {
        return Mono.just("最终操作")
                .map(val -> {
                    if (val.equals("最终操作")) throw new RuntimeException("模拟终止");
                    return val;
                })
                .doFinally(signal -> System.out.println("流终止信号类型：" + signal));
    }

    // 6. log()
    @GetMapping("/log")
    public Flux<String> logExample() {
        return Flux.just("X", "Y", "Z")
                .log(); // 控制台打印信号日志（订阅、请求、发出、完成等）
    }

    // 7. cache()
    Mono<String> cachedMono = Mono.fromSupplier(() -> {
        System.out.println("执行昂贵操作...");
        return "昂贵结果";
    }).cache();

    @GetMapping("/cache")
    public Mono<String> cacheExample() {
        return cachedMono; // 不会再次执行昂贵操作
    }

    // 8. share()
    Flux<Long> sharedFlux = Flux.interval(Duration.ofSeconds(1))
            .map(val -> val + 1)
            .doOnSubscribe(sub -> System.out.println("开始生成直播数据"))
            .share();

    @GetMapping("/share")
    public Flux<Long> shareExample() {
        return sharedFlux.take(5); // 每个订阅者看到的是“正在直播”的状态
    }
}
```

---


###### 5.2.3.8. 订阅方法

```
@RestController
@RequestMapping("/api")
public class SubscribeVariants {

    // 1. subscribe()
    @GetMapping("/subscribe-empty")
    public String subscribeEmptyExample() {
        Flux<String> flux = Flux.just("A", "B", "C")
                .doOnNext(val -> System.out.println("处理元素：" + val));

        flux.subscribe(); // 什么都不处理，但触发了执行
        return "subscribe() 执行完毕（请看控制台）";
    }

    // 2. subscribe(Consumer<? super T> onNext)
    @GetMapping("/subscribe-onNext")
    public String subscribeOnNextExample() {
        Flux<String> flux = Flux.just("X", "Y", "Z");

        flux.subscribe(val -> System.out.println("收到元素：" + val));
        return "subscribe(onNext) 执行完毕（请看控制台）";
    }

    // 3. subscribe(Consumer<? super T> onNext, Consumer<? super Throwable> onError, Runnable onComplete)
    @GetMapping("/subscribe-full")
    public String subscribeFullExample() {
        Flux<String> flux = Flux.just("1", "2", "error", "3")
                .map(val -> {
                    if (val.equals("error")) {
                        throw new RuntimeException("遇到非法值：" + val);
                    }
                    return val;
                });

        flux.subscribe(
                val -> System.out.println("接收到：" + val),
                err -> System.out.println("发生错误：" + err.getMessage()),
                () -> System.out.println("流处理完成！")
        );

        return "subscribe(onNext, onError, onComplete) 执行完毕（请看控制台）";
    }
}
```

---


### 6. Flux

#### 6.1. Flux 相关方法

##### 6.1.1. Flux 实例创建方法

==1.从已有数据构建流==
当你手头已经有数据（数组、集合、Stream 等），直接封装成 Flux 发出去。
1. `Flux.just(T... data)`
	1. 从确定且非 `null` 的值创建 Flux（可以为空字符串 `""`），**依次**发出各值所有元素，适合少量数据
2. `Flux.fromArray(T[] array)`
	1. 从 `Array` 数组创建 Flux
3. `Flux.fromIterable(Iterable<T> it)`：
	1. 从 `Iterable`创建 Flux（List、Set、Queue），依次发出所有元素
```
@RestController
@RequestMapping("/api")
public class ReactiveExample {

    // 1. Flux.just(T... data)
    @GetMapping("/just")
    public Flux<String> just() {
        return Flux.just("A", "B", "C");
    }

    // 2. Flux.fromArray(T[] array)
    @GetMapping("/fromArray")
    public Flux<String> fromArray() {
        String[] arr = {"X", "Y", "Z"};
        return Flux.fromArray(arr);
    }

    // 3. Flux.fromIterable(Iterable<T> it)
    @GetMapping("/fromIterable")
    public Flux<Integer> fromIterable() {
        List<Integer> list = Arrays.asList(1, 2, 3);
        return Flux.fromIterable(list);
    }
}
```



==2.按规则动态生成数据==
不依赖已有数据，而是代码里一个个“造”出来，适合序列、状态机等。
1. `Flux.range(int start, int count)`
	1. 创建一个从 start 开始，连续发出 count 个整数的 Flux。
2. `Flux.generate(Callable<S> stateSupplier，BiFunction<S，SynchronousSink<T>，S>generator)`
	1. 动态生成元素，适合需要状态控制的场景。
3. `Flux.defer(Supplier<? extends Publisher<T>> supplier)`
	1. 延迟创建 `Flux`，每次订阅时重新生成数据
```
@RestController
@RequestMapping("/api")
public class ReactiveExample {

    // 1. Flux.range(int start, int count)
    @GetMapping("/range")
    public Flux<Integer> range() {
        return Flux.range(1, 5);
    }

    // 2. Flux.generate(Callable<S> stateSupplier, BiFunction<S, SynchronousSink<T>, S> generator)
    @GetMapping("/generate")
    public Flux<Integer> generate() {
        return Flux.generate(
            () -> 0,
            (state, sink) -> {
                sink.next(state);
                if (state >= 4) sink.complete();
                return state + 1;
            }
        );
    }

    // 3. Flux.defer(Supplier<? extends Publisher<T>> supplier)
    @GetMapping("/defer")
    public Flux<Long> defer() {
        return Flux.defer(() -> Flux.just(System.currentTimeMillis()));
    }
}

```


==3.基于异步回调的数据流==
数据来自外部事件、回调、异步任务，推模式为主。
1. `Flux.create(Consumer<FluxSink<T>>emitter)`
	1. 提供更灵活的控制，适合异步或外部事件驱动的场景。
```
@RestController
@RequestMapping("/api")
public class ReactiveExample {

    // 1. Flux.create(Consumer<FluxSink<T>> emitter)
    @GetMapping("/create")
    public Flux<String> create() {
        return Flux.create(emitter -> {
            new Thread(() -> {
                emitter.next("async-1");
                emitter.next("async-2");
                emitter.complete();
            }).start();
        });
    }
}

```


==4.基于时间调度创建流==
按照固定时间间隔发出数据，用于轮询、心跳、定时任务等。
1. `Flux.interval(Duration period)`
	1. 按固定时间间隔发出递增的长整型数字（从 0 开始）
2. `Flux.interval(Duration delay, Duration period)`
	1. 指定初始延迟和间隔时间
```
@RestController
@RequestMapping("/api")
public class ReactiveExample {

    // 1. Flux.interval(Duration period)
    @GetMapping(value = "/interval", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<Long> interval() {
        return Flux.interval(Duration.ofSeconds(1));
    }

    // 2. Flux.interval(Duration delay, Duration period)
    @GetMapping(value = "/intervalWithDelay", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<Long> intervalWithDelay() {
        return Flux.interval(Duration.ofSeconds(3), Duration.ofSeconds(1));
    }
}

```


==5.控制型流（空流、异常、无限挂起）==
不发真实数据，用于流程控制、占位、异常测试等。
1. `Flux.empty()`
	1. 创建一个不发出任何元素直接完成的 `Flux
2. `Flux.error(Throwable throwable)`
	1. 创建一个发出错误信号的 `Flux`。
3. `Flux.never()`
	1. 创建一个不发出任何信号（不发出元素、不完成、不出错）的 `Flux`。
```
@RestController
@RequestMapping("/api")
public class ReactiveExample {

    // 1. Flux.empty()
    @GetMapping("/empty")
    public Flux<String> empty() {
        return Flux.empty();
    }

    // 2. Flux.error(Throwable throwable)
    @GetMapping("/error")
    public Flux<String> error() {
        return Flux.error(new RuntimeException("Something went wrong"));
    }

    // 3. Flux.never()
    @GetMapping("/never")
    public Flux<String> never() {
        return Flux.never();
    }
}

```


==6.组合多个流==
不生成新数据，而是对多个已有 Flux 进行组合或转换。
1. `Flux.from(Publisher<? extends T> source)`
	1. 从另一个 `Publisher`（如 `Mono` 或其他 `Flux`）创建 `Flux`。
2. `Flux.concat(Publisher<? extends T>... sources)`
	1. 按顺序连接多个 `Publisher`，依次发出它们的元素
3. `Flux.merge(Publisher<? extends T>... sources)`
	1. 合并多个 `Publisher`，异步发出元素（不保证顺序)）
4. `Flux.zip(Publisher<? extends T1> source1, Publisher<? extends T2> source2, ...)`
	1. 将多个 `Publisher` 的元素组合成元组。
```
@RestController
@RequestMapping("/api")
public class ReactiveExample {

    // 1. Flux.from(Publisher<? extends T> source)
    @GetMapping("/fromPublisher")
    public Flux<String> fromPublisher() {
        Mono<String> mono = Mono.just("Mono data");
        return Flux.from(mono);
    }

    // 2. Flux.concat(Publisher<? extends T>... sources)
    @GetMapping("/concat")
    public Flux<String> concat() {
        Flux<String> flux1 = Flux.just("A", "B");
        Flux<String> flux2 = Flux.just("C", "D");
        return Flux.concat(flux1, flux2);
    }

    // 3. Flux.merge(Publisher<? extends T>... sources)
    @GetMapping("/merge")
    public Flux<String> merge() {
        Flux<String> flux1 = Flux.just("A", "B");
        Flux<String> flux2 = Flux.just("C", "D");
        return Flux.merge(flux1, flux2);
    }

    // 4. Flux.zip(Publisher<? extends T1> source1, Publisher<? extends T2> source2, ...)
    @GetMapping("/zip")
    public Flux<String> zip() {
        Flux<String> flux1 = Flux.just("A", "B");
        Flux<String> flux2 = Flux.just("1", "2");
        return Flux.zip(flux1, flux2, (s1, s2) -> s1 + s2);
    }
}
```

---

  
##### 6.1.2. flux 对象实例方法

###### 6.1.2.1. 实例方法一览表

| **实例方法**                                                                                          | **功能描述**                                                                                                                                      |
| ------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| **转换与映射方法**                                                                                       |                                                                                                                                               |
| `map(Function<? super T, ? extends R> mapper)`                                                    | 对流中每个元素应用提供的函数（同步简单操作），将它从类型 T 变换成类型 R。<br>与 `flatMap` 不同点在于：`map` 是同步操作，结果通常是原始类型转换，没有异步流。                                                   |
| `flatMap(Function<? super T, ? extends Publisher<? extends R>> mapper)`                           | 对每个元素执行一个异步操作（异步阻塞、异步非阻塞），返回一个新的异步流，并将所有这些异步流与元素合并成一个流。<br>“展平”的意思是把多个 Publisher 的流合成一个统一的 Flux。                                              |
| `cast(Class<R> clazz)`                                                                            | 将流中元素强制转换为类型 R；若元素类型与 R 无继承或实现关系（即类型不兼容），将抛出 `ClassCastException` 异常。                                                                         |
| `ofType(Class<U> clazz)`                                                                          | 仅保留属于指定类型 U 的元素，过滤掉其他类型的元素。                                                                                                                   |
| `switchIfEmpty(Publisher<? extends T> alternate)`                                                 | 如果 Flux 为空（没有发出任何元素就完成），则切换到指定的备用 Publisher（如另一个 Flux 或 Mono）继续发出元素。<br>注意：是整个 Flux 为空，而不是某个元素为空                                              |
| `defaultIfEmpty(T defaultValue)`                                                                  | 如果 Flux 为空（没有发出任何元素就完成），则发出指定的默认值（可以是流，也可以不是流）并完成。<br>与 `switchIfEmpty` 不同点在于：`defaultIfEmpty` 提供的是一个固定的同步值，而不是另一个异步流。适用于简单场景，如为空时返回默认字符串或对象。 |
| **过滤方法**                                                                                          |                                                                                                                                               |
| `filter(Predicate<? super T> predicate)`                                                          | 对每个元素执行断言函数，仅保留返回 `true` 的元素，返回 `false` 的元素将被过滤掉（满足条件的保留）。                                                                                    |
| `distinct()`                                                                                      | 去重操作，移除重复元素，确保每个元素在流中只出现一次（对整个流去重，不论是否分批加载）。<br>核心机制：维护一个 `HashSet` 来记录已发出的元素。每当一个新元素准备推送时，都会先检查是否已存在于集合中。若处理的数据流非常大（如百万级），这个集合可能迅速膨胀，占用大量内存。 |
| **收集与聚合方法**                                                                                       |                                                                                                                                               |
| `collectList()`                                                                                   | 等到整个流执行完毕（无论是否分批加载），所有元素会被收集进一个 `List`，并最终通过 `Mono<List>` 一次性发送。例如：`Flux.just("A", "B", "C").collectList();` 会返回 `["A", "B", "C"]`。           |
| `collectMultimap(Function<? super T, ? extends K> keyMapper)`                                     | 使用 keyMapper 为每个元素生成键，将元素本身作为值，汇聚到一个 `Map<K, Collection<T>>` 里，并封装成 `Mono<Map<K, Collection<T>>>` 返回，就像按标签分门别类地整理档案。                          |
| `reduce(BiFunction<T, T, T> aggregator)`                                                          | 对流中元素进行两两累积，将前一次计算结果与下一个元素交给 aggregator 处理，最终合成一个值并通过 Mono 发出。常用于求和、拼接字符串、数据汇总等。                                                              |
| `scan(BiFunction<T, T, T> accumulator)`                                                           | 类似 reduce，但每遇到一个新元素就会「打卡」一次，立刻把当前的累积结果发出来——就好比你在跑步时，每跑一公里就看一次手表，实时掌握进度。                                                                       |
| **合并与组合方法**                                                                                       |                                                                                                                                               |
| `mergeWith(Publisher<? extends T> other)`                                                         | 并行地把另一个流的元素插入当前流，不保证顺序——就像把两条河的水汇入同一个大江，水流混杂而不分先后。                                                                                            |
| `concatWith(Publisher<? extends T> other)`                                                        | 先发完当前流所有元素，再按顺序接入另一个流，保证完全的先后顺序，就像礼貌地排队，前面的人走完后才轮到后面。                                                                                         |
| `zipWith(Publisher<? extends U> other)`                                                           | 两个流按索引位置逐个配对生成 `Tuple2<T, U>`，如 flux1 的第一个元素配 flux2 的第一个元素；当任一流提前结束，组合流也会结束。适用于必须一一对应的场景。                                                     |
| `combineLatest(Publisher<? extends U> other, BiFunction<T, U, R> combinator)`                     | 订阅两个流，并在任一流发出新元素时，取对方最新值一起合并发出。                                                                                                               |
| **限制与截取方法**                                                                                       |                                                                                                                                               |
| `take(long n)`                                                                                    | 只接收前 n 个元素，下游收到 n 个后就主动完成，适合取样或分页场景。                                                                                                          |
| `takeLast(int n)`                                                                                 | 缓存发出的所有元素，等整个流执行完毕才一次性发出最后 n 个元素；常用于需要尾部数据的情况，但要注意内存开销。                                                                                       |
| `skip(long n)`                                                                                    | 忽略前 n 个元素，直接跳过，再把剩余元素发给下游。                                                                                                                    |
| `skipLast(int n)`                                                                                 | 缓存发出的所有元素，等整个流执行完毕后丢弃最后 n 个元素，再把剩下的一起发出。                                                                                                      |
| **错误处理方法**                                                                                        |                                                                                                                                               |
| `onErrorReturn(T fallback)`                                                                       | 一旦上游发生错误，就发出一个默认值（可以是流也可以不是流），不再传递错误信号。                                                                                                       |
| `onErrorMap(Function<Throwable, ? extends Throwable> mapper)`                                     | 捕获异常并将其包装或转换为另一种更具语义的异常类型，然后重新发出错误信号，便于统一上层错误处理策略。简单来说：异常 A -> 异常 X，上层统一处理异常 X。                                                               |
| `onErrorResume(Function<Throwable, ? extends Publisher<? extends T>> fallback)`                   | 错误发生时，切换到备用的 Publisher 继续发出元素，避免整个流被中断，可用于降级、重试或备用数据源。                                                                                        |
| `onErrorContinue(BiConsumer<Throwable, Object> errorConsumer)`                                    | 对单个元素处理出错时只执行指定副作用（如记录日志），并跳过该元素继续后续处理，下游不会收到错误；相当于「有容错的流水线」。                                                                                 |
| `retry(long times)`                                                                               | 出错时将重试整个 Flux 上游 `times` 次（总共尝试 1 + times 次）。                                                                                                 |
| **延迟与调度方法**                                                                                       |                                                                                                                                               |
| `delayElements(Duration delay)`                                                                   | 给每个元素之间插入固定延迟，模拟节流或人为降速，就像每隔一秒钟才放一个包裹。                                                                                                        |
| `subscribeOn(Scheduler scheduler)`                                                                | 指定整个订阅过程在哪个调度器线程上运行（影响源的执行线程）<br>注意：多个 `subscribeOn` 只会生效第一个                                                                                  |
| `publishOn(Scheduler scheduler)`                                                                  | 指定 `publishOn` 之后的操作在那个线程池执行                                                                                                                  |
| `timeout(Duration timeout)`                                                                       | 设置超时时间，如果某个元素未在规定时间内到达，就抛出 `TimeoutException`，用于超时控制                                                                                          |
| **其他实用方法**                                                                                        |                                                                                                                                               |
| `doOnNext(Consumer<? super T> onNext)`                                                            | 每当流发出一个元素时执行指定副作用（如日志、指标统计），但不改变元素本身；就像在传送带上做一次检查打标。                                                                                          |
| `doOnError(Consumer<? super Throwable> onError)`                                                  | 流遇到错误时执行副作用（如记录日志、告警），然后再把错误继续抛给下游；适合在错误点插入埋点或监控。                                                                                             |
| `doOnComplete(Runnable onComplete)`                                                               | 在流正常走完所有元素后执行一次副作用，如资源释放或完成通知；就像闭环操作中的收尾仪式。                                                                                                   |
| `doOnSubscribe(Consumer<? super Subscription> onSubscribe)`                                       | 订阅者发起订阅时触发，用于记录或处理初次订阅行为，比如打日志或初始化状态。                                                                                                         |
| `doFinally(Consumer<SignalType> onFinally)`                                                       | 在流终止时（无论正常完成、取消、还是出错）都执行指定副作用，相当于 finally 块，常用于收尾清理等场景。                                                                                       |
| `log()`                                                                                           | 自动在控制台打印 Flux 的信号轨迹，包括订阅、请求、发出元素、完成和错误，帮你像放监控摄像头一样随时看流水线状态。                                                                                   |
| `cache()`                                                                                         | 将流转换为可重放的热流，缓存所有元素，后续订阅者会立刻看到之前的数据，无需再次执行上游逻辑；适合昂贵操作的结果复用。                                                                                    |
| `share()`                                                                                         | 也会把 Cold Flux 变成 Hot Flux，但不缓存旧数据，所有订阅者共享同一份数据流，适合广播式分发；就像把演唱会直播送给多个观众同时观看。                                                                   |
| **订阅方法**                                                                                          |                                                                                                                                               |
| `subscribe()`                                                                                     | 触发流的执行，**不处理任何信号**（元素、错误、完成），                                                                                                                 |
| `subscribe(Consumer<? super T> onNext)`                                                           | 触发流的执行，只处理正常发出的元素，不管错误或完成                                                                                                                     |
| `subscribe(Consumer<? super T> onNext, Consumer<? super Throwable> onError, Runnable onComplete)` | 触发流的执行，分别处理元素、错误、完成 三种信号                                                                                                                      |

---


###### 6.1.2.2. 转换与映射方法

```
@RestController
@RequestMapping("/api/operators")
public class OperatorExamples {

    // 1. map(Function<? super T, ? extends R> mapper)
    @GetMapping("/map")
    public Flux<Integer> mapExample() {
        return Flux.just("1", "2", "3")
                   .map(Integer::parseInt); // 将 String 类型转换成 Integer 类型
    }

    // 2. flatMap(Function<? super T, ? extends Publisher<? extends R>> mapper)
    @GetMapping("/flatMap")
    public Flux<String> flatMapExample() {
        return Flux.just("user1", "user2")
                   .flatMap(userId -> Mono.just("Profile of " + userId)); // 模拟异步调用
    }

    // 3. cast(Class<R> clazz)
    @GetMapping("/cast")
    public Flux<Number> castExample() {
        return Flux.just(1, 2.0, 3L)
                   .cast(Number.class); // 所有元素都可以被安全地转为 Number 类型
    }

    // 4. ofType(Class<U> clazz)
    @GetMapping("/ofType")
    public Flux<Integer> ofTypeExample() {
        return Flux.just(1, "two", 3, "four")
                   .ofType(Integer.class); // 只保留 Integer 类型的元素
    }
    
	// 5. switchIfEmpty(Publisher<? extends T> alternate)
    @GetMapping("/switchIfEmpty")
    public Flux<String> switchIfEmptyExample() {
        return Flux.<String>empty()
                   .switchIfEmpty(Flux.just("default1", "default2"));
    }

    // 6. defaultIfEmpty(T defaultValue)
    @GetMapping("/defaultIfEmpty")
    public Flux<String> defaultIfEmptyExample() {
        return Flux.<String>empty()
                   .defaultIfEmpty("default-value");
    }
}
```

---


###### 6.1.2.3. 过滤方法

```
@RestController
@RequestMapping("/api/operators")
public class OperatorExamples {

    // 1. filter(Predicate<? super T> predicate)
    @GetMapping("/filter")
    public Flux<String> filterExample() {
        return Flux.just("apple", "banana", "cherry", "date")
                   .filter(fruit -> fruit.startsWith("a")); // 只保留以 "a" 开头的元素
    }

    // 2. distinct()
    @GetMapping("/distinct")
    public Flux<String> distinctExample() {
        return Flux.just("apple", "banana", "apple", "cherry", "banana")
                   .distinct(); // 去除重复元素，确保每个元素只出现一次
    }
}
```

---


###### 6.1.2.4. 收集与聚合方法

```
@RestController
@RequestMapping("/api/operators")
public class OperatorExamples {

    // 1. collectList()
    @GetMapping("/collectList")
    public Mono<List<String>> collectListExample() {
        return Flux.just("A", "B", "C")
                   .collectList(); // 返回：["A","B","C"]
    }

    // 2. collectMultiMap(Function<? super T, ? extends K> keyMapper)
	@GetMapping("/collectMap")  
	public Mono<Map<Character, Collection<String>>> collectMapExample() {  
	    return Flux.just("apple", "banana", "apricot")  
            .collectMultimap(word -> word.charAt(0)); // 用每个单词的首字母作为 key，返回：{"a":["apple","apricot"],"b":["banana"]}
	}
	
    // 3. reduce(BiFunction<T, T, T> aggregator)
    @GetMapping("/reduce")
    public Mono<String> reduceExample() {
        return Flux.just("A", "B", "C")
                   .reduce((a, b) -> a + b); // 累加成一个字符串，返回："ABC"
    }

    // 4. scan(BiFunction<T, T, T> accumulator)
    @GetMapping("/scan")
    public Flux<String> scanExample() {
        return Flux.just("A", "B", "C")
                   .scan((a, b) -> a + b); // 实时输出，返回："A", "AB", "ABC"
    }
}
```

---


###### 6.1.2.5. 合并与组合方法

```
@RestController
@RequestMapping("/api/operators")
public class OperatorExamples {

    // 1. mergeWith(Publisher<? extends T> other)
    @GetMapping("/mergeWith")
    public Flux<String> mergeWithExample() {
        Flux<String> flux1 = Flux.just("A", "B").delayElements(Duration.ofMillis(100));
        Flux<String> flux2 = Flux.just("1", "2").delayElements(Duration.ofMillis(50));
        return flux1.mergeWith(flux2); // 并行混合输出，不保证顺序
    }

    // 2. concatWith(Publisher<? extends T> other)
    @GetMapping("/concatWith")
    public Flux<String> concatWithExample() {
        Flux<String> flux1 = Flux.just("A", "B");
        Flux<String> flux2 = Flux.just("1", "2");
        return flux1.concatWith(flux2); // 先发 flux1，再发 flux2，严格按顺序来
    }

    // 3. zipWith(Publisher<? extends U> other)
    @GetMapping("/zipWith")
    public Flux<String> zipWithExample() {
        Flux<String> flux1 = Flux.just("A", "B", "C");
        Flux<String> flux2 = Flux.just("1", "2");
        return flux1.zipWith(flux2) // 执行结果：[{"t1":"A","t2":"1"},{"t1":"B","t2":"2"}]
                    .map(tuple -> tuple.getT1() + tuple.getT2()); // 输出 "A1", "B2"
    }

    // 4. combineLatest(Publisher<? extends U> other, BiFunction<T, U, R> combinator)
    @GetMapping("/combineLatest")
    public Flux<String> combineLatestExample() {
        Flux<String> flux1 = Flux.just("A", "B").delayElements(Duration.ofMillis(100));
        Flux<String> flux2 = Flux.just("1", "2", "3").delayElements(Duration.ofMillis(50));
        return Flux.combineLatest(flux1, flux2, (s1, s2) -> s1 + s2); // 总是用“最新”组合，返回：A1A2A3B3
    }
}
```

---


###### 6.1.2.6. 限制与截取方法

```
@RestController
@RequestMapping("/api/operators")
public class OperatorExamples {

    // 1. take(long n)
    @GetMapping("/take")
    public Flux<Integer> takeExample() {
        return Flux.range(1, 10)
                   .take(3); // 只取前 3 个元素，输出：1, 2, 3
    }

    // 2. takeLast(int n)
    @GetMapping("/takeLast")
    public Flux<Integer> takeLastExample() {
        return Flux.range(1, 10)
                   .takeLast(3); // 等所有元素发完后再输出最后 3 个，输出：8, 9, 10
    }

    // 3. skip(long n)
    @GetMapping("/skip")
    public Flux<Integer> skipExample() {
        return Flux.range(1, 10)
                   .skip(4); // 跳过前 4 个，输出：5, 6, 7, 8, 9, 10
    }

    // 4. skipLast(int n)
    @GetMapping("/skipLast")
    public Flux<Integer> skipLastExample() {
        return Flux.range(1, 10)
                   .skipLast(4); // 丢弃最后 4 个，输出：1, 2, 3, 4, 5, 6
    }
}

```

---


###### 6.1.2.7. 错误处理

```
@RestController
@RequestMapping("/api/operators")
public class ErrorHandlingExamples {

    // 1. onErrorReturn(T fallback)
    @GetMapping("/onErrorReturn")
    public Flux<String> onErrorReturnExample() {
        return Flux.just("A", "B", "C")
                   .concatWith(Flux.error(new RuntimeException("Boom!")))
                   .onErrorReturn("Fallback"); // 出错时返回默认值："Fallback"
    }

    // 2. onErrorMap(Function<Throwable, ? extends Throwable> mapper)
    @GetMapping("/onErrorMap")
    public Flux<String> onErrorMapExample() {
        return Flux.just("A", "B", "C")
                   .concatWith(Flux.error(new IllegalArgumentException("Invalid input")))
                   .onErrorMap(e -> new CustomException("Mapped: " + e.getMessage()));
    }

    // 3. onErrorResume(Function<Throwable, ? extends Publisher<? extends T>> fallback)
    @GetMapping("/onErrorResume")
    public Flux<String> onErrorResumeExample() {
        return Flux.just("A", "B", "C")
                   .concatWith(Flux.error(new RuntimeException("Service down")))
                   .onErrorResume(e -> Flux.just("Switched to backup", "Another item")); // 切换到备用 Flux
    }

    // 4. onErrorContinue(BiConsumer<Throwable, Object> errorConsumer)
    @GetMapping("/onErrorContinue")
    public Flux<String> onErrorContinueExample() {
        return Flux.just("1", "two", "3")
                   .map(Integer::parseInt)
                   .onErrorContinue((e, value) ->
                       System.out.println("Skipping '" + value + "' due to: " + e.getMessage()));
        // "two" 转换失败被跳过，继续处理 "3"
    }

    // 5. retry(long times)
    @GetMapping("/retry")
    public Flux<String> retryExample() {
        return Flux.just("A", "B", "C")
                   .concatWith(Flux.error(new RuntimeException("Temporary failure")))
                   .retry(3) // 错误信号会被 retry(3) 捕获并重试最多 3 次
                   .onErrorReturn("After retries, fallback"); // 若仍然失败，则返回默认值
    }
}
```

---


###### 6.1.2.8. 延迟与调度方法

```
@RestController
@RequestMapping("/api/flux")
public class FluxThreadingAndTiming {

    // 1. delayElements(Duration delay)
    @GetMapping("/delayElements")
    public Flux<String> delayElementsExample() {
        return Flux.just("A", "B", "C")
                   .delayElements(Duration.ofSeconds(1)); // 每个元素之间间隔 1 秒发出
    }

    // 2. subscribeOn(Scheduler scheduler)
    @GetMapping("/subscribeOn")
    public Flux<String> subscribeOnExample() {
        return Flux.defer(() -> {
                    System.out.println("Subscribed on thread: " + Thread.currentThread().getName());
                    return Flux.just("data1", "data2");
                })
                .subscribeOn(Schedulers.boundedElastic()); // 指定「数据源」运行在哪个线程池
    }

    // 3. publishOn(Scheduler scheduler)
    @GetMapping("/publishOn")
    public Flux<String> publishOnExample() {
        return Flux.just("A", "B", "C")
                   .doOnNext(v -> System.out.println("Before publishOn: " + Thread.currentThread().getName()))
                   .publishOn(Schedulers.parallel()) // 从这里开始切换执行线程
                   .doOnNext(v -> System.out.println("After publishOn: " + Thread.currentThread().getName()));
    }
    
     // 4. timeout(Duration timeout)
    @GetMapping("/timeout")
    public Flux<String> timeoutExample() {
        return Flux.just("X", "Y", "Z")
                   .delayElements(Duration.ofSeconds(2)) // 每个元素延迟2秒
                   .timeout(Duration.ofSeconds(1))       // 如果超过1秒未发出新元素，则触发 TimeoutException
                   .onErrorResume(e -> Flux.just("超时fallback")); // 错误降级处理
    }
}
```

> [!NOTE] 注意事项
> 1. `.delayElements` 单位
> 	1. `Duration.ofNanos(long)`：
> 		1. 纳秒
> 	2. `Duration.ofMillis(long)`：
> 		1. 毫秒
> 	3. `Duration.ofSeconds(long)`：
> 		1. 秒
> 	4. `Duration.ofMinutes(long)`：
> 		1. 分钟
> 	5. `Duration.ofHours(long)`：
> 		1. 小时
> 	6. `Duration.ofDays(long)`：
> 		1. 天数
> 2. 常见线程
> 	1. `Schedulers.parallel()`
> 		1. 适用于 CPU 密集型任务，使用固定大小线程池
> 	2. `Schedulers.boundedElastic()`
> 		1. 适用于 I/O 密集型任务（如数据库、网络、文件操作）
> 	3. `Schedulers.immediate()`
> 		1. 在当前线程中执行
> 	4. `Schedulers.fromExecutor(Executor)`
> 		1. 使用自定义线程池调度任务

---


###### 6.1.2.9. 其他实用方法

```
@RestController
@RequestMapping("/api/flux")
public class FluxHooksController {

    // 1. doOnNext(Consumer<? super T> onNext)
    @GetMapping("/doOnNext")
    public Flux<String> doOnNextExample() {
        return Flux.just("apple", "banana", "orange")
                   .doOnNext(item -> System.out.println("处理前日志：" + item));
    }

    // 2. doOnError(Consumer<? super Throwable> onError)
    @GetMapping("/doOnError")
    public Flux<String> doOnErrorExample() {
        return Flux.just("1", "2")
                   .concatWith(Flux.error(new RuntimeException("故意错误")))
                   .doOnError(e -> System.out.println("错误发生：" + e.getMessage()));
    }

    // 3. doOnComplete(Runnable onComplete)
    @GetMapping("/doOnComplete")
    public Flux<String> doOnCompleteExample() {
        return Flux.just("X", "Y")
                   .doOnComplete(() -> System.out.println("流已完成"));
    }

    // 4. doOnSubscribe(Consumer<? super Subscription> onSubscribe)
    @GetMapping("/doOnSubscribe")
    public Flux<String> doOnSubscribeExample() {
        return Flux.just("start")
                   .doOnSubscribe(sub -> System.out.println("订阅开始：" + sub));
    }

    // 5. doFinally(Consumer<SignalType> onFinally)
    @GetMapping("/doFinally")
    public Flux<String> doFinallyExample() {
        return Flux.just("keep", "going")
                   .concatWith(Flux.error(new RuntimeException("Oops!")))
                   .doFinally(type -> System.out.println("流终止，原因：" + type));
    }

    // 6. log()
    @GetMapping("/log")
    public Flux<String> logExample() {
        return Flux.just("foo", "bar").log();
    }

    // 7. cache()
    private Flux<String> expensiveSource = Flux.just("A", "B", "C")
                                               .doOnSubscribe(sub -> System.out.println("上游被调用"))
                                               .cache();

    // 8. share()
    private Flux<String> sharedSource = Flux.interval(Duration.ofSeconds(1))
                                            .map(i -> "Tick-" + i)
                                            .take(5)
                                            .doOnSubscribe(sub -> System.out.println("订阅共享流"))
                                            .share();

    @GetMapping(value = "/share", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> shareExample() {
        return sharedSource;
    }
}
```

---


###### 6.1.2.10. 订阅方法

```
@RestController
@RequestMapping("/api/flux")
public class FluxHooksController {

	// 1. subscribe()
	@GetMapping("/subscribe")
	public Flux<String> subscribeExample() {
	    return Flux.just("A", "B", "C")
	               .subscribe();
	}
	
	// 2. subscribe(onNext)
	@GetMapping("/subscribeOnNext")
	public Flux<String> subscribeOnNextExample() {
	    return Flux.just("Item 1", "Item 2")
	               .subscribe(item -> System.out.println("接收到的元素: " + item));
	}

	// 3. subscribe(onNext, onError, onComplete)
	@GetMapping("/subscribeAll")
	public Flux<String> subscribeAllExample() {
	    return Flux.just("Task 1", "Task 2")
	               .concatWith(Flux.error(new RuntimeException("Error in task")))
	               .subscribe(
	                   item -> System.out.println("收到任务: " + item), 
	                   error -> System.out.println("发生错误: " + error.getMessage()), 
	                   () -> System.out.println("任务完成")
	               );
	}
}
```


> [!NOTE] 注意事项：已经有了 `subscribe(onNext, onError, onComplete)`，为什么还要使用 `doOnNext`、`doOnError`、`doOnComplete` 这些副作用操作
> 1. **`doOnNext`、`doOnError` 和 `doOnComplete`** 是为了在流处理过程中执行**副作用**（比如日志记录、统计、监控等），但它们不会直接影响流的最终输出。它们让你在流运行时插入额外的行为，而不改变流本身。
> 2. **`subscribe(onNext, onError, onComplete)`** 是你处理流结果的地方。这是最终的反应——你是流的消费者，订阅流并处理实际的数据、错误和完成状态。





# 二、实操



## 三、补充

#### 操作行为划分
1. ==同步操作==：
	1. <font color="#00b0f0">同步简单操作</font>：
		1. 快速执行、无明显耗时、不涉及复杂计算或 I/O
	2. <font color="#00b0f0">同步阻塞操作</font>：
		1. 设计耗时计算或潜在阻塞操作（如 I/O），但是在当前线程同步执行
2. ==异步操作==：
	1. <font color="#00b0f0">异步阻塞操作</font>：
		1. 将同步阻塞操作通过异步机制（如线程池）执行，“伪非阻塞” 操作
	2. <font color="#00b0f0">异步非阻塞操作</font>：
		1. 完全非阻塞，通常是基于时间驱动或回调，依赖外部异步机制（如 `WebClient`、 `R2DBC`、`CompletableFuture`）
3. ==补充：I/O==：
	1. I/O，即输入/输出（Input/Output），是指计算机与外部世界的信息交换过程，包括但不限于：
		1. <font color="#00b0f0">网络请求</font>：
			1. 如调用其他 HTTP API 接口（RestTemplate、WebClient）、JDBC、R2DBC 等等
		2. <font color="#00b0f0">文件操作</font>：
			1. 读取或写入硬盘上的文件
		3. <font color="#00b0f0">用户交互</font>：
			1. 键盘、鼠标等输入设备的信号获取，以及向显示器等输出设备发送信息。
		4. <font color="#00b0f0">硬件通信</font>：
			1. 与打印机、扫描仪等外部设备的数据交换

