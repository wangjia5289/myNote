---
title: 笔记：RabbitMQ
date: 2025-04-23
categories:
  - 数据管理
  - 消息队列
  - RabbitMQ
tags: 
author: 霸天
layout: post
---













![](PixPin_2025-05-09_15-09-55.png)

| 特性    | RabbitMQ   | Kafka          | RocketMQ               | ActiveMQ       |
| ----- | ---------- | -------------- | ---------------------- | -------------- |
| 类型    | 消息队列（AMQP） | 分布式日志系统        | 分布式消息中间件               | 传统 JMS 消息队列    |
| 语言    | Erlang     | Scala/Java     | Java                   | Java           |
| 吞吐    | 中等         | 极高             | 高                      | 中等偏低           |
| 延迟    | 低          | 中              | 低                      | 中等             |
| 消息顺序  | 支持（某种程度）   | 分区内有序          | 原生强顺序（可配置）             | 有序（但不稳定）       |
| 消息追踪  | 无原生支持      | 有 offset 可控    | ✅ 原生消息轨迹追踪             | ❌ 不完善          |
| 消息回溯  | 复杂         | ✅ 易（offset 回溯） | ✅ 类似 Kafka 的 offset 模型 | ❌ 基本不支持        |
| 分布式支持 | 集群可配但不天然   | ✅ 天然分布式        | ✅ 天然支持                 | ❌ 集群弱，主备为主     |
| 死信队列  | ✅ 支持       | 需自建逻辑          | ✅ 原生支持                 | ✅ 支持           |
| 社区活跃度 | 🔥🔥🔥     | 🔥🔥🔥🔥🔥     | 🔥🔥                   | ❄️（Apache 里冷门） |



# 实操


### 1. 单机测试环境搭建

#### 1.1. 安装 RabbitMQ

```
docker run -d \
--name rabbitmq \
-p 5672:5672 \
-p 15672:15672 \
-e RABBITMQ_DEFAULT_USER=guest \
-e RABBITMQ_DEFAULT_PASS=123456 \
rabbitmq:3.13-management
```

---


#### 1.2. 访问 RabbitMQ 控制台

访问 RabbitMQ 控制台： http://192.168.136.7:15672
![](image-20250510173743688.png)

---


### 2. 高可用集群搭建

#### 2.1. 架构说明

---


#### 2.2. 环境准备

---


#### 2.3. 节点列表

| IP             | 主机名        |
| -------------- | ---------- |
| 192.168.136.8  | rbmq-node1 |
| 192.168.136.9  | rbmq-node2 |
| 192.168.136.10 | rbmq-node3 |

---

#### 2.4. 时间同步

---


#### 2.5. 开放 5672、15672、4369、25672 TCP端口

```
sudo ufw allow 5672/tcp && sudo ufw allow 15672/tcp && sudo ufw allow 4369/tcp && sudo ufw allow 25672/tcp
```

> [!NOTE] 注意事项
> 1. TCP 端口就是我们常说的 HTTP 端口

---

#### 2.6. 设置主机名、主机名互相解析

必须设置，因为后续节点加入集群，需要根据主机名找到中间人，不能根据 IP 地址
```
# 1. 设置主机名
# 1.1. 192.168.136.8
sudo hostnamectl set-hostname rbmq-node1

# 1.2. 192.168.136.9
sudo hostnamectl set-hostname rbmq-node2

# 1.3. 192.168.136.10
sudo hostnamectl set-hostname rbmq-node3


# 2. 设置主机名互相解析
sudo vim /etc/hosts
"""
192.168.136.8 rbmq-node1
192.168.136.9 rbmq-node2
192.168.136.10 rbmq-node3
"""
```

---


#### 2.7. 安装 RabbitMQ

根据 [RabbitMQ 安装文档](https://www.rabbitmq.com/docs/install-debian)，选择合适的版本（3.13），并在每个节点上进行安装。安装方式可选用 `Apt with Cloudsmith Mirrors: a Quick Start Script` 或 `Using Apt with Cloudsmith Mirrors`。随后，使用 `Debian Package Version and Repository Pinning` 限定只从 RabbitMQ 提供的 Cloudsmith 镜像源安装 Erlang，并锁定特定版本的 Erlang 和 RabbitMQ。

在开始之前，我们先设置一下局部代理，然后开启一下 VM NAT：
```
export http_proxy="http://192.168.68.4:7890" && export https_proxy="http://192.168.68.4:7890" && export no_proxy="localhost,127.0.0.1,.svc,.cluster.local,192.168.136.0/24,10.96.0.1,10.244.0.0/16" && export HTTP_PROXY=$http_proxy && export HTTPS_PROXY=$https_proxy && export NO_PROXY=$no_proxy
```

然后删除一下之前的 `packagecloud.io` 和 `erlang-solutions.com` 的源：
```
# 1. 查找所有涉及 packagecloud.io 和 erlang-solutions.com 的源
grep -R "packagecloud\.io\|erlang-solutions\.com" /etc/apt/sources.list /etc/apt/sources.list.d/*.list
"""
root@user-VMware-Virtual-Platform:~# grep -R "packagecloud\.io\|erlang-solutions\.com" /etc/apt/sources.list /etc/apt/sources.list.d/*.list
/etc/apt/sources.list.d/erlang.list:deb https://packages.erlang-solutions.com/ubuntu noble contrib
/etc/apt/sources.list.d/rabbitmq_erlang.list:deb https://packagecloud.io/rabbitmq/erlang/ubuntu/ jammy main
/etc/apt/sources.list.d/rabbitmq_server.list:deb https://packagecloud.io/rabbitmq/rabbitmq-server/ubuntu/ jammy main
"""

# 2. 删除这些源
sudo rm /etc/apt/sources.list.d/erlang.list \
        /etc/apt/sources.list.d/rabbitmq_erlang.list \
        /etc/apt/sources.list.d/rabbitmq_server.list
```

然后进行 RabbitMQ 的安装：
![](image-20250513090533397.png)

以下情况，无需在意，实际上它已经创建了主目录：
![](image-20250513100716597.png)

---


#### 2.8. 启用管理界面插件

就是启用我们的后台管理： http://192.168.136.8:15672
```
rabbitmq-plugins enable rabbitmq_management
```

---


#### 2.9. 启动第一个节点

##### 2.9.1. 启动第一个节点

第一个节点非常重要，后续节点可以通过这个节点作为中间人，从而加入到集群：
```
# 1. 启动 RabbitMQ 服务
systemctl start  rabbitmq-server


# 2. 开机自启动 RabbitMQ 服务
systemctl enable rabbitmq-server
```

---


##### 2.9.2. 进行 RabbitMQ 集群基础配置

```
# 1. 新增登录账号密码
rabbitmqctl add_user batian wq666666
"""
注意：默认账号密码是 guest，不过出于安全考虑，它在配置里被限制为 只能通过 localhost（也就是 127.0.0.1 或 ::1）来连接，所以我们需要新建登录账号密码
"""

# 2. 设置登录账号权限
rabbitmqctl set_user_tags batian administrator

rabbitmqctl set_permissions -p / batian ".*" ".*" ".*"
"""
1. set_user_tags：
	1. 给 batian 这个用户打上一个或多个 标签（tags）。
	2. administrator：
		1. 管理员权限，能在 Management UI 和 HTTP API 中进行所有管理操作（增删用户、队列、交换机、权限、策略等）
	3. monitoring：
		1. 监控权限，只能查看运行时状态（队列长度、连接、通道、消息速率等），无法做写入或配置操作
	4. policymaker：
		1. 策略管理权限，允许创建、修改和删除策略（policies），但不具备完整的管理或监控权限。
2. set_permissions：
	1. 给用户在某个 “虚拟主机”（vhost）上的 权限。
	2. -p /：
		1. 表示默认的 “根” vhost（大多数场景都使用 `/`）
	3. ".*" 三个正则：
		1. configure-regex：
			1. 控制“声明”操作（declare/删除队列、交换机、绑定、quorum queue 等）
			2. 例如 ^amq\. 禁止用户操作内建的 amq.* 资源，或者 ^$ 直接不允许任何声明
		2. write-regex：
			1. 控制“写入”操作（publish 消息到交换机或默认交换机）
			2. 例如 ^logs\. 只允许向以 logs. 开头的交换机发送消息
		3. read-regex：
			1. 控制“读取”操作（从队列消费消息、使用 HTTP API 读取队列/交换机信息）
			2. 例如 ^tasks$ 只允许从名为 tasks 的队列中读取消息。
"""


# 3. 启用所有稳定功能 flag 启用
rabbitmqctl enable_feature_flag all


# 4. 重启RabbitMQ服务生效
systemctl restart rabbitmq-server
```

---


#### 2.10. 启动其他节点

##### 2.10.1. 查看集群内正常节点的 Erlang Cookie 值 并记录

若后续需要将新节点加入集群，只需从当前集群中的任一正常节点获取 Erlang Cookie，并在新节点中使用相同的 Cookie 即可完成身份验证。

这里我们集群中只有第一个节点，所以查看第一个节点的 Erlang Cookie 值
```
# 1. 查看 Cookie 值
vim /var/lib/rabbitmq/.erlang.cookie 


# 2. 记录 Cookie 值
VAWCBDXAIOWOXLZSBRYS
```

---


##### 2.10.2. 加入集群并启动节点

```
# 1. 如果开启了 RabbitMQ 服务，先关闭服务
sudo systemctl stop rabbitmq-server


# 2. 修改 Erlang Cookie 值，强制保存（w! 和 q!）
vim /var/lib/rabbitmq/.erlang.cookie
"""
输入刚才记录的 Cookie 值：VAWCBDXAIOWOXLZSBRYS
"""


# 2. 启动 RabbitMQ 服务
sudo systemctl start rabbitmq-server


# 2. 停止、重置、加入、启动
# 2.1. 停止
rabbitmqctl stop_app

# 2.2. 重置
rabbitmqctl reset

# 2.3. 加入
rabbitmqctl join_cluster rabbit@rbmq-node1

# 2.4. 启动
rabbitmqctl start_app
"""
1. rabbitmqctl stop_app：
	1. 停止 RabbitMQ 的应用（注意不是停止整个服务），防止在运行中加入集群导致冲突
2. rabbitmqctl reset：
	1. 清空本节点的元数据，包括已有的集群状态，这一步是必须的
3. rabbitmqctl join_cluster rabbit@rbmq-node1
	1. 将当前节点加入名为 rabbit@node01 的 RabbitMQ 集群
	2. rbmp-node1 为中间人，只能写主机名，不能写 IP：
4. rabbitmqctl start_app：
	1. 启动 RabbitMQ 应用，正式生效，开始作为集群成员工作
"""


# 3. 开机自启动 RabbitMQ 服务
systemctl enable rabbitmq-server
```

---


##### 2.10.3. 检查集群状态

```
rabbitmqctl cluster_status
```

---




















# ----

#### 0.1. 虚拟空间


![](image-20250511112907036.png)


![](image-20250511112921622.png)








不同消息队列根据路由键进行分配到队列












起名：
exchange.direct.order

exchange.direct.order.backup
queue.direct.order.backup






这个能直接发布信息
![](image-20250510215227990.png)


对这个图
![](image-20250510215558278.png)



![](image-20250510220812312.png)












![](image-20250510194722644.png)





#### 0.2. 环境要求

1. ==系统要求==：
	1. Ubuntu 22.04
	2. 开放 5672、15672、4369、25672 TCP端口
2. ==软件要求==：
	1. RabbitMQ 4.0.1
	2. Erlang 26.2

---


#### 0.3. 
---


### 1. 安装 Erlang（RabbitMQ 的运行强依赖）

```
# 1. 设置临时代理
export http_proxy="http://172.20.10.3:7890" && export https_proxy="http://172.20.10.3:7890" && export no_proxy="localhost,127.0.0.1,.svc,.cluster.local,192.168.136.0/24,10.96.0.1,10.244.0.0/16" && export HTTP_PROXY=$http_proxy && export HTTPS_PROXY=$https_proxy && export NO_PROXY=$no_proxy


# 2. 安装必要工具
sudo apt-get install -y curl gnupg apt-transport-https


# 3. 添加 RabbitMQ 包签名
curl -fsSL https://packagecloud.io/rabbitmq/erlang/gpgkey | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/rabbitmq_erlang.gpg && curl -fsSL https://packagecloud.io/rabbitmq/rabbitmq-server/gpgkey | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/rabbitmq_server.gpg


# 4. 添加 APT 源
echo "deb https://packagecloud.io/rabbitmq/erlang/ubuntu/ $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/rabbitmq_erlang.list && echo "deb https://packagecloud.io/rabbitmq/rabbitmq-server/ubuntu/ $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/rabbitmq_server.list


# 5. 更新索引
sudo apt-get update



# 2. 添加 Erlang 仓库
wget -O- https://packages.erlang-solutions.com/ubuntu/erlang_solutions.asc | sudo apt-key add -
echo "deb https://packages.erlang-solutions.com/ubuntu $(lsb_release -sc) contrib" | sudo tee /etc/apt/sources.list.d/erlang.list


# 2. 更新并安装 Erlang
sudo apt update && sudo apt install erlang -y
```




# 三、工具

### 1. RabbitMQ 控制台

#### 1.1. 访问方式

当我们部署完 RabbitMQ 后，可以访问服务器节点： http://192.168.136.7:15672
![](image-20250510173743688.png)

---











# 四、补充：

### 1. 相关网站

1. ==Erlang 与 RabbitMQ 版本兼容==
	1. https://www.rabbitmq.com/docs/which-erlang





![](image-20250511104732641.png)



### 2. **消息可以被其他消费者读取吗？**

- **默认行为**：如果消费者没有确认消息（没有发送 ACK），那么消息会继续被认为是处于未确认（unacked）状态。在 RabbitMQ 中，这条消息不会被其他消费者读取，直到消息超时或者消费者确认（ACK）或者拒绝（NACK）该消息。
    
- **消息重试**：如果消费者崩溃或连接断开，RabbitMQ 会检测到该消费者没有确认消息（ACK），然后将该消息重新放回队列，待其他消费者处理。这种机制保证了消息的可靠性，即使消费者出现问题，消息也不会丢失。
    

### 3. **如何控制消息的重新投递？**

- **消息超时机制**（`time-to-live`）：RabbitMQ 不会自动对未确认消息进行超时处理，除非设置了超时机制。但可以设置 **消费者确认超时** 或通过某些机制来控制消息的重试。











# 补充

### 1. 常见网站

1. RabbitMQ 官网
2. RabbitMQ 安装文档
	1. https://www.rabbitmq.com/docs/install-debian

### 2. 交换机、队列命名

命名模版：`[项目名].[业务名].[功能名].[类型]

常见类型：
1. 主交换机：`exchange`
2. 主队列：`queue`
3. 死信交换机：`dlx.exchange`
4. 死信队列：`dlx.queue`
5. 备份交换机：`backup.exchange`
6. 备份队列：`backup.queue`

举例说明：
1. 支付创建：`payment.create.exchange`
2. 支付超时取消：`payment.timeout.exchange`
3. 支付回调通知：`payment.notify.exchange`



































































































































































































































































































































