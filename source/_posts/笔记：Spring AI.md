---
title: 笔记：Spring AI
date: 2025-03-12
categories:
  - Java
  - Spring 生态
  - Spring AI
tags: 
author: 霸天
layout: post
---
# 一、理论

### 0、导图：[Map：Spring AI](../../maps/Map：SpringAI.xmind)

---


### 1、Spring AI 概述

Spring AI 是一个旨在简化 Java 开发者集成 AI 功能的框架，其**提供统一的 API 接口**，便于调用多种 AI 模型。它支持对话生成、文本生成、向量化处理、多模态交互等功能。

---


### 2、核心模型 API

#### 2.1、前言：了解 Prompt

`Prompt` 封装了一系列消息（`Message`），包括**角色定义**、**用户指令**和**约束条件**，并允许设置**模型参数**。通过这种方式，我们可以将 Prompt 以参数的形式传递给大模型，从而精确控制对话上下文，并有效引导 AI 模型生成预期的响应。
```
/**
* Prompt 的构造方法：
* 
* 1. Prompt(String)
* 2. Prompt(Message)
* 3. Prompt(List<Message>)
* 4. Prompt(String,ChatOptions)
* 5. Prompt(Message,ChatOptions)
* 6. Prompt(List<Message>,ChatOptions)
*/

// 以 Prompt(List<Message>,ChatOptions)为例：

// 1. 配置模型参数
ChatOptions chatOptions = ChatOptionsBuilder.builder()  
		.withModel("qwen-max-latest")  
		.withTemperature(0.7)  
		.build();  
		
// 2. 定义消息序列
List<Message> messages = new ArrayList<Message>();  
messages.add(new SystemMessage("你是一位资深Java架构师")); // 角色定义，设定模型的角色和背景，例如：你是一位 Java 架构师
messages.add(new UserMessage(message)); // 用户指令，传达具体的任务或需求，例如：请设计一个高并发订单系统
messages.add(new AssistantMessage("输出需包含架构图和核心代码")); // 用户指令，传达具体的任务或需求，例如：请设计一个高并发订单系统

// 3.将消息序列和模型参数封装为 prompt
Prompt prompt = new Prompt(messages,chatOptions);  
```

> [!NOTE] 注意事项
> 1. 消息序列通常并非硬编码在程序中，而是通过动态注入的方式生成，以便根据不同的需求灵活调整对话内容和模型响应

---


#### 2.2、对话模型 API

##### 2.2.1、ChatModel
```
/**
* chatModel 的 call 方法：
* 
* 1. call(Prompt prompt)：该方法接受一个封装了消息序列的 Prompt 对象，并允许配置模型参数，适用于需要复杂对话管理的场景，例如多轮对话、角色扮演，或对AI模型行为的细致调控。
* 2. call(String message)：最简化的调用方式，直接传入一条字符串作为用户的提问。内部会自动将该字符串转为包含单一用户消息的 Prompt 对象并发送给AI模型。适合基础的交互需求，例如简单的问答功能，且无需处理复杂的对话逻辑或个性化设置。
*/

@RestController  
@RequestMapping("/ai")  
public class ChatModelTest {  
	// 1. 注入 ChatModel
    private final ChatModel chatModel;  
  
    public ChatModelTest(ChatModel chatModel) {  
        this.chatModel = chatModel;  
    }  
  
    @GetMapping("/chatModel")  
    public String chat(@RequestParam String question) {  
	    // 配置模型参数
        ChatOptions chatOptions = ChatOptionsBuilder.builder()  
                .withModel("qwen-max-latest")  
                .withTemperature(0.7)  
                .build();  
                
		// 定义消息序列
        List<Message> messages = new ArrayList<Message>();  
        messages.add(new SystemMessage("你是一位资深Java架构师")); // 角色定义  
        messages.add(new UserMessage(question)); // 用户指令，例如：请设计一个高并发订单系统  
        messages.add(new AssistantMessage("输出需包含架构图和核心代码")); // 约束条件  
        
		// 2.将消息序列和模型参数封装为 prompt
        Prompt prompt = new Prompt(messages,chatOptions);  
        
        // 3. 调用 chatModel 的 call 方法，并对返回结果进行处理
        ChatResponse chatResponse = chatModel.call(prompt);  
        return chatResponse.getResult().getOutput().getContent();  
    }  
}
```

> [!NOTE] 注意事项
> 1. `ChatModel` 来自 `org.springframework.ai.chat.model` 
> 2. `ChatOptionsBuilder` 来自 `org.springframework.ai.chat.prompt`
> 3. `Message` 来自 `org.springframework.ai.chat.messages`

---


##### 2.2.2、StreamingChatModel

```
/**
* StreamingChatModel 的 stream 方法：
* 
* 1. stream(Prompt prompt)：该方法接受一个封装了消息序列的 Prompt 对象，并允许配置模型参数，适用于需要复杂对话管理的场景，例如多轮对话、角色扮演，或对AI模型行为的细致调控。
* 2. stream(String message)：最简化的调用方式，直接传入一条字符串作为用户的提问。内部会自动将该字符串转为包含单一用户消息的 Prompt 对象并发送给AI模型。适合基础的交互需求，例如简单的问答功能，且无需处理复杂的对话逻辑或个性化设置。
*/

@RestController  
@RequestMapping("/ai")  
public class StreamingChatModelTest {  
    // 1. 注入 StreamingChatModel    
    private final StreamingChatModel streamingChatModel;  
  
    public StreamingChatModelTest(StreamingChatModel streamingChatModel) {  
        this.streamingChatModel = streamingChatModel;  
    }  
  
    @GetMapping("/streamingChatModel")  
    public Flux<String> chat(@RequestParam String question) {  
        // 配置模型参数  
        ChatOptions chatOptions = ChatOptionsBuilder.builder()  
                .withModel("qwen-max-latest")  
                .withTemperature(0.7)  
                .build();  
  
        // 定义消息序列  
        List<Message> messages = new ArrayList<Message>();  
        messages.add(new SystemMessage("你是一位资深Java架构师")); // 角色定义  
        messages.add(new UserMessage(question)); // 用户指令，例如：请设计一个高并发订单系统  
        messages.add(new AssistantMessage("输出需包含架构图和核心代码")); // 约束条件  
  
        // 2.将消息序列和模型参数封装为 prompt       
		Prompt prompt = new Prompt(messages,chatOptions);  
  
        // 3. 调用 StreamingchatModel 的 stream 方法，并对返回结果进行处理  
        Flux<ChatResponse> stream = streamingChatModel.stream(prompt);  
        return stream.map(chunk -> chunk.getResult().getOutput().getContent());  
  
    }  
}
```

> [!NOTE] 注意事项
> 1. `StreamingChatModel` 来自 `org.springframework.ai.chat.model` 
> 2. `ChatOptionsBuilder` 来自 `org.springframework.ai.chat.prompt`
> 3. `Message` 来自 `org.springframework.ai.chat.messages`
> 4. 关于 Flux 的操作，可以参考 `Spring WebFlux` 的相关内容：[[笔记：Spring Web#4、Spring WebFlux|Spring WebFlux]]

---


##### 2.2.3、ChatClient

==1.非流式对话==
```
/**
* chatClientRequestSpec 的 user 方法
* 
* 1. user(Prompt prompt)：该方法接受一个封装了消息序列的 Prompt 对象，并允许配置模型参数，适用于需要复杂对话管理的场景，例如多轮对话、角色扮演，或对AI模型行为的细致调控。
* 2. user(String message)：最简化的调用方式，直接传入一条字符串作为用户的提问。内部会自动将该字符串转为包含单一用户消息的 Prompt 对象并发送给AI模型。适合基础的交互需求，例如简单的问答功能，且无需处理复杂的对话逻辑或个性化设置。
*/

@RestController  
@RequestMapping("/ai")  
public class ChatClientTest {  
  
    // 1. 注入 ChatClient    
    private final ChatClient chatClient;  
  
    public ChatClientTest(ChatClient.Builder builder) {  
        // 创建 ChatClient 的方法：ChatClient.Builder 对象的 build() 方法  
        this.chatClient = builder.build();  
    }  
    @GetMapping("/chatClient")  
    public String chat(@RequestParam String question) {  
        // 配置模型参数  
        ChatOptions chatOptions = ChatOptionsBuilder.builder()  
                .withModel("qwen-max-latest")  
                .withTemperature(0.7)  
                .build();  
                
        // 定义消息序列，将用户指令在 user 方法处传入，而不是这里  
        List<Message> messages = new ArrayList<Message>();  
        messages.add(new SystemMessage("你是一位资深Java架构师")); // 角色定义  
        messages.add(new AssistantMessage("输出需包含架构图和核心代码")); // 约束条件  
  
        // 2.将消息序列和模型参数封装为 prompt        
        Prompt prompt = new Prompt(messages,chatOptions);  
        
        // 3. 调用 chatClient 的 prompt 方法，并对返回结果进行处理  
        ChatClient.ChatClientRequestSpec chatClientRequestSpec = chatClient.prompt(prompt);  
        return chatClientRequestSpec.user(question).call().content();  
    }  
}
```

==2.流式对话==
```
/**
* chatClientRequestSpec 的 user 方法
* 
* 1. user(Prompt prompt)：该方法接受一个封装了消息序列的 Prompt 对象，并允许配置模型参数，适用于需要复杂对话管理的场景，例如多轮对话、角色扮演，或对AI模型行为的细致调控。
* 2. user(String message)：最简化的调用方式，直接传入一条字符串作为用户的提问。内部会自动将该字符串转为包含单一用户消息的 Prompt 对象并发送给AI模型。适合基础的交互需求，例如简单的问答功能，且无需处理复杂的对话逻辑或个性化设置。
*/

@RestController  
@RequestMapping("/ai")  
public class ChatClientTest {  
  
    // 1. 注入 ChatClient    
    private final ChatClient chatClient;  
  
    public ChatClientTest(ChatClient.Builder builder) {  
        // 创建 ChatClient 的方法：ChatClient.Builder 对象的 build() 方法  
        this.chatClient = builder.build();  
    }  
    @GetMapping("/chatClient")  
    public Flux<String> chat(@RequestParam String question) {  
        // 配置模型参数  
        ChatOptions chatOptions = ChatOptionsBuilder.builder()  
                .withModel("qwen-max-latest")  
                .withTemperature(0.7)  
                .build();  
                
        // 定义消息序列，将用户指令在 user 方法处传入，而不是这里  
        List<Message> messages = new ArrayList<Message>();  
        messages.add(new SystemMessage("你是一位资深Java架构师")); // 角色定义  
        messages.add(new AssistantMessage("输出需包含架构图和核心代码")); // 约束条件  
  
        // 2.将消息序列和模型参数封装为 prompt        
        Prompt prompt = new Prompt(messages,chatOptions);  
        
        // 3. 调用 chatClient 的 prompt 方法，并对返回结果进行处理  
        ChatClient.ChatClientRequestSpec chatClientRequestSpec = chatClient.prompt(prompt);  
        return chatClientRequestSpec.user(question).stream.content();  
    }  
}
```

> [!NOTE] 注意事项
> 1. `ChatClient` 来自 `org.springframework.ai.chat.client`
> 2. `ChatClient.Builder` 来自 `org.springframework.ai.chat.client.ChatClient`
> 3. `ChatOptionsBuilder` 来自 `org.springframework.ai.chat.prompt`
> 4. `Message` 来自 `org.springframework.ai.chat.messages`

---


##### 2.2.4、对话模型参数调优
```
/**
* 对话模型常用参数：
* 
* 1. withModel(String model)：指定使用的 AI 模型名称。
* 2. withTemperature(Double temperature)：控制输出的随机性程度。较低的值（接近0）会使输出更加确定性和重复，而较高的值（接近1或以上）则会增加创造力和多样性。
* 3. withMaxTokens(Integer maxTokens): 指定最大令牌数。从而限制回复的最大长度，避免过长的回答消耗过多资源或时间，在一些场景下非常适用。
*/

// 使用 ChatOptions 进行参数调优
ChatOptions chatOptions = ChatOptionsBuilder.builder()  
		.withModel("qwen-max-latest")  
		.withTemperature(0.7)  
		.build();  
```

> [!NOTE] 注意事项
> 1. 如果希望查看更多的参数信息，可以利用 IDEA 提供的代码提示功能。
> 2. `ChatOptionsBuilder` 来自 `org.springframework.ai.chat.prompt`

---


#### 2.3、图像模型 API

##### 2.3.1、ImageModel

```
/**
* imageModel 的 call 方法
* 1. call(ImagePrompt imageprompt)：此方法接收一个封装了生成图像所需的各种参数或描述信息的 ImagePrompt 对象。通过调用此方法，您可以向AI模型发送请求，从而基于所提供的提示生成相应的图像。
*/

@RestController  
@RequestMapping("/ai")  
public class ImageModelTest {  
    // 1. 注入 ImageModel    private final ImageModel imageModel;  
  
    public ImageClient(ImageModel imageModel) {  
        this.imageModel = imageModel;  
    }  
  
    @GetMapping("/imageModel")  
    public String GetImage(@RequestParam String question) {  
        // 配置模型参数  
        ImageOptions imageOptions = ImageOptionsBuilder.builder()  
                .withModel("wanx2.1-t2i-turbo")  
                .build();  
  
        // 2. 将用户指令和模型参数封装为 prompt        ImagePrompt imagePrompt = new ImagePrompt(question,imageOptions);  
  
        // 3. 调用 imageModel 的 call 方法，并对返回结果进行处理  
        ImageResponse imageResponse = imageModel.call(imagePrompt);  
        return imageResponse.getResult().getOutput().getUrl(); // 这里返回图片的临时 URL    }  
}
```

> [!NOTE] 注意事项
> 1. `ImageModel` 来自 `org.springframework.ai.chat.client`
> 2. `ImageOptionsBuilder` 来自 `org.springframework.ai.image`
> 3. `Message` 来自 `org.springframework.ai.chat.messages

---


##### 2.3.2、图像模型参数调优
```
/**
* 图像模型常用参数：
* 
* 1. withModel(String model)：指定使用的 AI 模型名称
* 2. withHeight(int height) ：指定生成图像的高度，常见：256、512、1024
* 3. withWidth(int width)：指定生成图像的宽度，常见：256、512、1024
* 4. withN(int n)：指定希望生成的图像数量，必须是一个 1~10 的整数。（DALL·E 3模型，仅支持 n = 1）
* 5. withStyle(String style)：仅适用于DALL·E 3模型，允许用户选择生成图像的艺术风格
* 6. withResponseFormat(String format)：指定接收到的数据格式是URL还是base64编码的JSON字符串(b64_json)
*    6.1 URL：每个生成的图像将由一个可以直接访问的链接表示，一般为默认选项；
*    6.2 b64_json：选择 b64_json 适合那些需要在本地进一步处理图像数据的应用程序
*/

// 使用 ImageOptionsBuilder 进行参数调优
ImageOptions imageOptions = ImageOptionsBuilder.builder()  
        .withModel("wanx2.1-t2i-turbo")  
        .build();
```

> [!NOTE] 注意事项
> 1. `ImageOptionsBuilder` 来自 `org.springframework.ai.image`

---


# 二、实操

### 1、Spring AI 集成 OpenAI

#### 1.1、环境准备
2. <font color="#00b0f0">JDK 版本</font>：使用 JDK 17 或更高版本
3. <font color="#00b0f0">Spring Boot 版本</font>：Spring AI 需要 Spring Boot 3.x 支持

---

#### 1.2、创建 Spring Boot 项目
见 `Categories/Java/常见 Java 项目创建方式`

---


#### 1.3、引入 OpenAI 依赖

引入 [OpenAI 起步依赖](https://mvnrepository.com/artifact/org.springframework.ai/spring-ai-openai) 
```
<dependencies>
	<dependency>
	    <groupId>org.springframework.ai</groupId>
	    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
	    <version>1.0.0-M6</version>
	</dependency>
</dependencies>
```

---

#### 1.4、配置 API Key
```
spring:  
  ai:  
    openai:  
      api-key: XXXXXXX
```

> [!NOTE] 注意事项
> 1. 关于如何安全配置 API Key，见 `Categories/Java/安全秘钥管理`

---


### 2、Spring AI 集成阿里云百炼

#### 2.1、环境准备
4. <font color="#00b0f0">JDK 版本</font>：
5. <font color="#00b0f0">Spring Boot 版本</font>：阿里云百炼对 Spring Boot 版本的要求较高，使用 Spring Boot 3.3.x 或更高版本。
6. <font color="#00b0f0">API Key</font>：在 [阿里云百炼](https://bailian.console.aliyun.com/?spm=a2c4g.11186623.0.0.266b2562wXkaxn&accounttraceid=f0f25be1960e4f2dace7763a9466fc65rjlo#/model-market) 控制台开通服务并获取 API Key

---


#### 2.2、创建 Spring Boot 项目
见 `Categories/Java/常见 Java 项目创建方式`

---


#### 2.3、引入阿里云起步依赖
```
<!-- 该依赖尚未发布到 Maven 中央仓库，需要添加 Spring Milestones 仓库。 -->
<repositories>
    <repository>
        <id>spring-milestones</id>
        <url>https://repo.spring.io/milestone</url>
    </repository>
</repositories>

<dependencies>
	<dependency>
	    <groupId>com.alibaba.cloud.ai</groupId>
	    <artifactId>spring-ai-alibaba-starter</artifactId>
	    <version>1.0.0-M3.1</version>
	</dependency>
</dependencies>
```

---


#### 2.4、配置 API Key
```
spring:  
  ai:  
    dashscope:  
      api-key: XXXXXXX
```

> [!NOTE] 注意事项
> 1. 关于如何安全配置 API Key，见 `Categories/Java/安全秘钥管理`

---


