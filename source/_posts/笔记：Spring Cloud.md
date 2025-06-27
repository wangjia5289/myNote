---
title: 笔记：Spring Cloud
date: 2025-03-14
categories:
  - Java
  - Spring 生态
  - Spring Cloud
tags: 
author: 霸天
layout: post
---
# 一、理论

### 1. 导图：[Map：Spring Cloud](../../maps/Map：SpringCloud.xmind)

---

### 2. Spring Cloud 组件图

![](image-20250427142937834.png)

---


### 3. Spring Cloud 流程图

- Spring Cloud LoadBalancer 是一种客户端负载均衡工具，用于在多个服务实例间分配请求。
- OpenFeign 是一种声明式的 Web 服务客户端，简化 REST API 调用，可与 LoadBalancer 集成以实现负载均衡。
- 两者可配合使用：OpenFeign 定义客户端接口，LoadBalancer 处理请求分发。

---


### 4. Consul

#### 4.1. Consul 概述

==1.现有问题==
在微服务 A 调用微服务 B 的过程中，如果把 B 的 IP 和端口写死在 A 的代码里，就容易遇到一堆大麻烦：
1. <font color="#00b0f0">服务发现困难</font>：
	1. 微服务 B 的 IP 或端口一旦变更，A 就懵了，找不到 B，调用失败。
2. <font color="#00b0f0">无法负载均衡</font>：
	1. 系统里明明有好几个 B，A 却只能连固定的一个，导致这个 B 被打爆，其他 B 闲得发慌。
3. <font color="#00b0f0">服务健康状态未知</font>：
	1. B 已经“阵亡”了，A 还傻乎乎地发请求过去，结果超时、报错、出事故，谁也不好受。

即便我们可能有一些解决方案，例如 `Keepalived + Haproxy`，也还会存在一些问题：
1. <font color="#00b0f0">配置管理混乱</font>：
	1. 服务多到飞起，每个服务都要配置一堆地址，配置文件满天飞，一改就头大，发布流程慢得像蜗牛。
	2. 有没有一种更清爽的方式？比如通用配置统一从一个固定位置拉取，大家只需要修改那个位置就行；而各服务的个性化配置则保留在各自的 `application.yml` 文件中，既集中管理，又保留灵活性。
2. <font color="#00b0f0">安全通信复杂</font>：
	1. 服务间通信加密、证书管理，麻烦得让人怀疑人生。


==2.Consul 解决方案==
Consul 是 HashiCorp 家出品的一款神器，把**服务发现**、**配置管理**和**服务网格**统统打包到一起，靠着 HTTP API、DNS 和灵活的健康检查机制，轻松搞定上述痛点：
1. <font color="#00b0f0">服务发现困难？</font>：
	1. 微服务一启动，就自己“报到”注册到 Consul，以后 A 直接问 Consul 要地址，妈妈再也不用担心我找不到 B！
2. <font color="#00b0f0">无法负载均衡？</font>：
	1. 搭配 Gateway、OpenFeign、LoadBalancer 等组件，可以优雅实现负载均衡，自动把请求合理分发给多个 B。
3. <font color="#00b0f0">服务健康状态未知？</font>：
	1. Consul 自带健康探测机制，服务挂了就“踢出群聊”，服务要注册就“过来拜码头”
4. <font color="#00b0f0">配置管理混乱？</font>：
	1. 配置集中托管在 Consul，微服务可以动态拉取或订阅更新，改配置就像改朋友圈状态一样轻松。
5. <font color="#00b0f0">安全通信复杂？</font>：
	1. Consul 内置证书颁发和管理，轻松搞定服务间的加密通信（Mutual TLS），安全又省心。

---


#### 4.2. 搭建 Consul 环境

##### 4.2.1. 高可用实现

##### 4.2.2. 单体测试环境搭建

==1.下载 Consul 安装包==
从 [Consul 下载地址](https://developer.hashicorp.com/consul/install) 下载 Consul 安装包：
![](image-20250427165021312.png)

> [!NOTE] 注意事项
> 1. 根据自己的 CPU 架构选择合适的下载方式
> 2. 查看 CPU 架构：`uname -m`


==2.将 Consul 安装包上传到服务器并解压==
在此步骤中，我将 Consul 安装包上传至 `/mystudy/consul` 目录，并直接在该目录中进行了解压。
```
# 1. 进入目录
cd /mystudy/consul


# 2. 解压
unzip consul_1.20.5_linux_amd64.zip
```


==3.启动 Consul==
```
# 1. 前台启动
./consul agent -dev -client 0.0.0.0


# 2. 后台启动（把所有日志（标准输出 + 错误输出）都写到 consul.log 文件）
# 2.1. 后台启动 Consul
nohup ./consul agent -dev -client 0.0.0.0 > consul.log 2>&1 &

# 2.2. 查询是否正确启动
ps -ef | grep consul


# 2.3. 查询是否监听 8500
ss -lntp | grep 8500
```


==4.访问 Consul UI==
```
http://192.168.136.200:8500
```


==5.补充：关闭 Consul==
```
# 1. 查看 Consul 进程号
ps -ef | grep consul


# 2. 杀死 Consul
kill -9 641001
```

---


##### 4.2.3. 高可用集群搭建（传统方案）

---


##### 4.2.4. 高可用集群搭建（K8S，推荐）

---


#### 4.3. 使用 Spring Cloud Consul

##### 4.3.1. 创建 Spring Boot Consul 项目

这里采用 IDEA 提供的脚手架创建 Spring Boot 项目，分别勾选：
1. ==Web==
	1. Spring Web
2. ==Spring Cloud Config==：
	1. Consul Configuration
3. ==Spring Cloud Discovery==
	1. Consul Discovery

> [!NOTE] 注意事项
> 1. <font color="#00b0f0">Consul Configuration</font>：
> 	1. 把应用的配置（比如数据库地址、密码、环境变量）存到 Consul，Spring Boot 启动时自动从 Consul 读配置，这样就不用放本地 `application.yml` 了
> 2. <font color="#00b0f0">Consul Discovery</font>：
> 	1. 把你的服务注册到 Consul，让别人发现你，同时你也可以发现别人

---


##### 4.3.2. 进行项目配置

###### 4.3.2.1. Spring Boot 如何加载配置文件

在项目配置之前，我们必须了解 Spring Boot 如何加载配置文件，详情见：

----


###### 4.3.2.2. 创建配置文件




```
spring:  
  application:  
    name: first_test                                # 项目名称，一般也作为 Consul 注册的名称
  cloud:  
    consul:  
      host: 192.168.136.8                           # Consul 节点的 IP 地址
      port: 8500                                    # Consul 节点的端口号
      discovery:  
        service-name: ${spring.application.name}    # 本服务在 Consul 注册的名称，一般与 application.name 一致
        prefer-ip-address: true                     # 以本服务所在 IP 地址优先，否则会自动使用主机名
      config:  
        enabled: false  
```
![|475](image-20250427192909664.png)

----


##### 4.3.3. 为主类标注 @EnableDiscoveryClient

`@EnableDiscoveryClient` 使得服务能够向注册中心（如 Eureka、Consul、Zookeeper 等）注册，并能够从注册中心发现其他服务。

当你启动这个 Spring Boot 应用时，它会自动将服务信息注册到 Consul 中，包括服务的名称、IP 地址和端口等。Consul 会维护这个服务的注册信息，其他服务可以通过 Consul 来发现该服务。

```
@SpringBootApplication  
@EnableDiscoveryClient  
public class CloudApplication {  
  
    public static void main(String[] args) {  
        SpringApplication.run(CloudApplication.class, args);  
    }  
  
}
```

---


### LoadBalancer + OpenFeign


#### 使用步骤

##### 创建并配置 OpenFeign 配置类

```

```





### 5. Spring Cloud LoadBalancer

#### 5.1. Spring Cloud LoadBalancer 概述

- Spring Cloud LoadBalancer 是一种客户端负载均衡工具，用于在多个服务实例间分配请求。
- OpenFeign 是一种声明式的 Web 服务客户端，简化 REST API 调用，可与 LoadBalancer 集成以实现负载均衡。
- 两者可配合使用：OpenFeign 定义客户端接口，LoadBalancer 处理请求分发。

----






































# 实操















# 补充




### 6.1.1. 相关网站

1. ==Consul 官网==：
	1. https://developer.hashicorp.com/consul
2. ==Consul 下载地址==：
	1. https://developer.hashicorp.com/consul/install

----


### 6.1.2. 常见注册中心

| 注册中心          | 开发语言 | CAP 架构 | 对外暴露接口 | 是否支持服务健康检查 | 是否与 Spring Cloud 继承 |
| ------------- | ---- | ------ | ------ | ---------- | ------------------- |
| **Eureka**    | Java | AP     | 可配置支持  | HTTP       | 已集成                 |
| **Consul**    | Go   | CP     | 支持     | HTTP/DNS   | 已集成                 |
| **Zookeeper** | Java | CP     | 支持     | 客户端        | 已集成                 |


