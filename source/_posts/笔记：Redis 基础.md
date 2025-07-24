---
title: 笔记：Redis 基础
date: 2025-05-17
categories:
  - 数据管理
  - 非关系型数据库
  - 键值型
  - Redis
  - Redis 基础
tags: 
author: 霸天
layout: post
---
# 理论

## 1. 导图：[Map：Redis](Map：Redis.xmind)

---


## 2. Redis 语法

### 2.1. 系统命令

```
# 1. 心跳命令（检测该客户端与 Redis 的连接是正常）
ping
```


### 集群命令

```

主从集群的哦，其他的无效
可以动态设置，比如在 Redis 启动后执行：
replicaof 192.168.1.100 6379

要取消从属关系，恢复成主节点，执行：
replicaof no one
```



---


### 2.2. 字符串键值对命令

==1.插入字符串键值对==
```
# 1. 设置单个键值对
set key value [NX|XX] [GET] [EX seconds|PX milliseconds|EXAT unix-time-seconds|PXAT unix-time-milliseconds|KEEPTTL]
"""
1. NX：
	1. 仅在键不存在时设置，等价于插入操作
2. XX：
	1. 仅在键已存在时设置，等价于更新操作
3. GET: 
	1. 设置新值前，返回键当前的值（若不存在返回 `nil`），便于获取旧值做对比
4. EX seconds：
	1. 设置键在指定 “秒数” 后过期（相对时间）
5. PX milliseconds：
	1. 设置键在指定 “毫秒数” 后过期（相对时间）
6. EXAT timestamp: 
	1. 设置键在指定 “秒级 Unix 时间戳” 时过期（绝对时间）
7. PXAT milliseconds-timestamp: 
	1. 设置键在指定 “毫秒级 Unix 时间戳” 时过期（绝对时间）
8. KEEPTTL: 
	1. 在更新键值时保留其原有 TTL（生存时间）不变
	2. 不能与任何过期时间参数（如 EX、PX、EXAT、PXAT）同时使用——要么保留现有 TTL，要么设置新的过期时间，二者不可兼得。
9. 举例：
	1. SET mykey "hello" NX EX 10
	2. SET session_token "abc123" EXAT 1717155600
	3. SET cache_result "ok" PXAT 1717155600123
"""


# 2. 批量设置多个键值对
MSET key1 value1 key2 value2 key3 value3
"""
1. 举例：
	1. MSET user:1 "Alice" user:2 "Bob" user:3 "Carol"
2. 注意事项：
	1. 不同于 Redis 管道（Pipeline），其是多条命令批量发送，非原子的
	2. 而此命令是原子性的，一条命令设置多个键值对，所有键值对要么全部设置成功，要么全部失败
	3. 当你使用 MSET 设置多个键时，不能使用 [NX|XX|GET|EX|PX|EXAT|PXAT|KEEPTTL] 这些选项
"""


# 3. 批量设置多个键值对（所有键必须不存在）
MSETNX key1 val1 key2 val2 key3 val3
"""
1. 注意事项：
	1. 所有键都不存在时才执行，否则一个都不设置
"""
```


==2.删除字符串键值对==


==3.修改字符串键值对==
```
# 1. 数值操作（必须是数值字符串）
SET visits 100    # 初始值：100
INCR visits       # 数字递增：101
INCRBY visits 50  # 增加指定的整数：151
DECR visits       # 数字递减：150
DECRBY visits 20  # 减少指定的整数：130


# 2. 字符串追加
APPEND mykey "world"  # 在末尾追加 "world"
```


==4.查询字符串键值对==
```
# 1. 查询单个值
get key
"""
1. 举例：
	1. get name
"""


# 2. 批量查询多个值
MGET key1 key2 key3


# 3. 获取字符串长度
STRLEN mykey


# 4. 查询剩余生存时间（以秒为单位）
System.out.println(Long.toString(System.currentTimeMillis()/1000L));
```


==5.其他命令==
```
# 1. 先 get 再 set
getset key value
```

----


## 3. Redis 管道

Redis 管道是 Redis 提供的一种批量发送命令的机制，其核心目的是减少客户端与服务器之间的网络往返次数，从而显著提升性能。

比如执行如下命令，其处理流程是：客户端发送请求 ➜ 服务器接收并处理 ➜ 服务器返回结果 ➜ 客户端等待接收：
```
SET name "Alice"
```

如果你连续发送 100 条命令，这个过程就得重复 100 次，网络延迟就会成为严重瓶颈。而使用管道机制，我们可以一次性把这 100 条命令打包发送，Redis 也会一口气处理完所有请求，然后将结果一并返回，效率大幅提升。

我们常常在 Java 代码中使用 Redis 管道操作（Pipelining）来提升性能：
```
// 以 Jedis 为例
Jedis jedis = new Jedis("localhost");
Pipeline pipeline = jedis.pipelined();

pipeline.set("key1", "value1");
pipeline.set("key2", "value2");
pipeline.get("key1");

List<Object> results = pipeline.syncAndReturnAll();
```

> [!NOTE] 注意事项
> 1. 管道不同于事务机制，它不具备原子性，也不能回滚，仅仅是提高通信效率、批量发送的一种手段

---














# 实操

## 1. 服务把控

### 1.1. 线程特性

Redis 采用的是 “**主线程执行命令 + 多线程处理部分辅助功能**” 的架构。在数据读写方面，Redis 始终坚持使用一个主线程来执行命令。由于 Redis 直接操作的是内存，访问速度极快，即使只用单线程处理，也能获得非常高的性能。更关键的是，单线程模式无需考虑线程安全问题，不需要加锁，从而避免了锁竞争等开销，反而执行效率更高。

而针对一些可能拖慢主线程性能的任务，Redis 引入了 **多线程或后台子进程** 来“分担杂活”，提高整体响应能力，例如：
1. ==网络 I/O 处理==：
	1. 从 Redis 6.0 起，可在配置文件中通过 `io-threads` 参数启动若干专用网络线程（默认关闭）。
	2. 这些线程负责客户端的请求、建立连接、协议解析和回复发送，真正的主线程只调度命令执行，IO 线程直接做数据收发，避免高并发时主线程在网络上“排队”。
2. ==异步删除大 Key==：
	1. 当删除非常大的对象（如超长列表、巨型哈希）时，命令执行后主线程只标记删除，并将释放内存的任务提交给后台“懒卸载”线程（通过 `lazyfree-threads` 配置）。
	2. 这些线程在空闲时异步清理对象，避免主线程因一次性大规模内存释放而卡顿。
3. ==集群通信、数据同步相关的任务==：
	1. 在集群模式下，节点间心跳维护、槽迁移、故障检测等通信都由集群总线线程异步处理，主线程仅负责本地命令执行。
	2. 在主从模式下，主节点有专门的 I/O 线程向从节点推送复制流，从节点也有 I/O 线程接收。
4. ==持久化==：
	1. 虽然 Redis 把数据放在内存中，但它也会周期性地将数据持久化到磁盘，比如通过 RDB 快照或 AOF 日志
	2. 这些写磁盘的操作相对耗时，因此 Redis 主进程 `fork()` 出一个子进程，该子进程负责异步进行持久化

---











## 2. 环境搭建

### 2.1. 单机测试

#### 2.1.1. 集群特性

##### 2.1.1.1. 事务支持

Redis 的事务机制本质上是**单机级别的**，在 Redis Cluster 模式下，其使用受到一定限制，不能保证跨节点的原子性

当我们调用 `MULTI` 时，表示开启一个事务，接下来的命令将依次进入事务队列，此时 Redis 不执行这些命令，而是返回 `QUEUED` 表示排队成功。直到我们执行 `EXEC`，所有排队命令才会被按顺序执行，并且执行期间不会被其他客户端打断，从而实现 “顺序一致、不可中断” 的效果
```
WATCH account_A balance_B         # 开始监视一个或多个 key，直到事务执行（EXEC）或被显式放弃（DISCARD）
val_A = GET account_A
val_B = GET balance_B

if val_A >= 100:                   
    MULTI                         # 开启事务
    DECRBY account_A 100          # 发送多个命令（排队）
    INCRBY balance_B 100
    EXEC                          # 提交事务
else:
    DISCARD                       # 余额不足，放弃事务
"""
1. 注意事项：
	1. 如果在 WATCH 到 EXEC 之间，监视的任意 key被其他客户端修改了，事务提交时会自动失败（EXEC 返回 nil），但事务队列不会自动取消（简单来说：只是提交失败，事务不会自动取消），而 DISCARD 是让我们主动放弃事务 + 取消监视
	2. 事务不是数据库那种全回滚事务，而是一旦某条命令出错，其前后命令仍然会照常执行，不会回滚。
"""
```

---


#### 2.1.2. Docker 环境部署

==1.创建宿主机数据挂载目录==
```
mkdir -p /mystudy/data/redis
```


==2.启动 Redis 容器==
```
docker run -d \
  --name redis \
  -p 6379:6379 \
  -v /mystudy/data/redis:/data \
  redis:latest \
  redis-server --appendonly yes --bind 0.0.0.0 --requirepass wq666666
"""
1. --appendonly yes
	1. 开启 AOF（Append Only File）持久化模式，简单可靠，适合这个场景
2. --bind 0.0.0.0
	1. 监听所有网卡
	2. Redis 默认只监听 127.0.0.1，也就是说只有通过本机地址 127.0.0.1:6379 才能访问它，哪怕你用本机的其他 IP（比如 192.168.136.8:6379）也无法连接。
"""
```

---


### 主从集群

#### 集群概述

主从集群是 Redis 最经典的集群模式，通常为一主多从架构：数据仅写入唯一的主节点，从节点通过复制机制同步主节点的数据，整个集群中只有一条主从复制链：

![](image-20250601162731128.png)

即使构建出多个主从组合的“多主多从”结构，其本质仍是伪多主：各主节点之间没有任何协调机制，彼此独立，数据完全隔离，甚至不具备 Redis Cluster 所支持的数据分片能力。
![](image-20250601162633830.png)

---


#### 集群特性

##### 读写分离

---


























### 2.2. Redis Cluster 集群

#### 2.2.1. 集群概述

Redis Cluster 是 Redis 官方提供的开源集群解决方案，通常采用主从架构搭建，并需要我们手动进行运维管理。它的最大特点在于，数据在写入时会被自动按哈希槽进行分区，直接分配到不同的主节点，随后，各主节点的数据由对应的从节点进行同步,而非像传统的大数据系统那样先进行分库分表处理。

---


#### 2.2.2. 集群特性

##### 2.2.2.1. 数据同步（单体一致性）

采用的是异步复制——主节点在写入成功后会立即返回；如果此时主节点宕机，而从节点尚未完成同步却被提升为主节点，就会导致数据丢失。这与 MySQL 类似，同样最多只能保障 **单体一致性**，实现分布式一致性中的 **最终一致性**，无法满足真正的强一致性的要求。

尽管在很多使用场景下，相比 MySQL，我们对 Redis 中数据的“是否存在”或“是否最新”已经更宽容一些，但对于诸如金融交易、秒杀等对一致性要求极高的业务场景，这种宽容是不可接受的，我们仍然必须追求严格的分布式一致性。









---


##### 2.2.2.2. 数据分布

Redis Cluster 采用虚拟槽分区，所有键通过哈希函数（`CRC16[key] & 16383`）映射到编号为 0 到 16383 的槽内，共计 16384 个槽（编号从 0 开始）
![](image-20250530162454124.png)

> [!NOTE] 注意事项
> 1. 数据仅分布在主节点上，不直接分区到从节点；从节点仅负责从对应主节点同步数据

----


##### 2.2.2.3. 事务支持

Redis Cluster 只支持同一个节点（slot）内的事务操作，跨节点时会立即报错

这是因为 Redis Cluster 采用了**虚拟槽位（hash slot）分区机制**：每个 key 会通过哈希函数映射到某个槽位（slot），再由系统将槽位分配给不同的 Redis 节点。例如，以下命令：
```
MSET user:1:name Alice user:2:name Bob
```

如果 `user:1:name` 和 `user:2:name` 分别落在不同的槽位（slot），即分布在不同的节点上，Redis 会直接返回错误：`CROSSSLOT Keys in request don't hash to the same slot`

如果项目对性能要求不高但需要强一致性的事务，使用单节点 Redis 即可满足需求；但如果系统规模较大且需要支持分布式事务，就需要考虑其他架构方案，而不是依赖 Redis Cluster 来解决（或许 Redis 初衷就算不是强一致性）

---


##### 2.2.2.4. 集群限制




在使用 Redis Cluster 时，也只能使用 DB 0












#### 2.2.3. 一主多从架构

##### 2.2.3.1. 节点列表

| IP             | 主机名 | 角色     |
| -------------- | --- | ------ |
| 192.168.136.8  |     | master |
| 192.168.136.9  |     | slave  |
| 192.168.136.10 |     | slave  |

---


##### 2.2.3.2. 目录结构

```
/mystudy/redis/
├── src/              # 源码和编译目录（解压后）
├── bin/              # 可执行文件（redis-server、redis-cli）
├── conf/             # 配置文件（redis7000.conf 等）
├── data/             # 数据目录（按端口分子目录）
│   ├── 7000/
│   └── 7001/
├── logs/             # 日志目录（按端口分子目录）
│   ├── 7000/
│   └── 7001/
└── redis-7.4.0.tar.gz         # 原始 tar.gz 文件


/mystudy/redis /
├── redis /
├── redis-7.4.0.tar.gz




```

---


##### 2.2.3.3. 环境准备

###### 2.2.3.3.1. 查看 Ubuntu 版本

```
lsb_release -a
```

----


###### 2.2.3.3.2. 下载软件

在 [Redis 官方下载地址](https://redis.io/downloads/#software) 下载 Redis 安装包（`redis-7.4.0.tar.gz`）
![](image-20250530104631796.png)

----


###### 2.2.3.3.3. 相关工具

```
sudo apt install -y build-essential pkg-config tcl make libjemalloc-dev
```

---


###### 2.2.3.3.4. 时间同步

```
# 1. 安装 chrony
sudo apt install -y chrony


# 2. 启动并开启自启动 chrony
sudo systemctl enable chrony && sudo systemctl start chrony
```

----


###### 2.2.3.3.5. 开放端口

```
# 1. 开放 6379 ~ 6381、16379 ~ 16381 TCP 端口
sudo ufw allow proto tcp from 192.168.136.0/24 to any port 6379:6381,16379:16381
"""
1. proto tcp from 192.168.136.0/24
	1. 仅允许来自 192.168.136.0/24 网段的机器访问这些端口
"""
```

> [!NOTE] 注意事项
> 1. 由于我们只有三台服务器，因此只需开放端口至 6381 和 16381。若有四台，则需开放至 6382 和 16382，依此类推

---


###### 2.2.3.3.6. 关闭 Swap 分区

```
# 1. 将 内容注释
vim /etc/fstab
"""
# 将此内容进行注释
# /swap.img       none    swap    sw      0       0
"""


# 2. 立即关闭 Swap 分区
swapoff -a
```

---


##### 2.2.3.4. 安装软件

==1.创建 /mystudy/redis 目录并进入==
```
mkdir -p /mystudy/redis


cd /mystudy/redis
```


==2.上传安装包==
上传 `redis-7.4.0.tar.gz` 到 `/mystudy/redis`


==3.解压安装包==
```
sudo tar -xvf redis-7.4.0.tar.gz


mv redis-7.4.0 redis
```


==4.编译第三方库==
```
cd /mystudy/redis/redis/deps


make hiredis linenoise lua hdr_histogram fpconv jemalloc
```


==5.安装软件==
```
cd /mystudy/redis/redis


make && make install


# 查看 redis-server 是否安装并可执行
which redis-server
```

> [!NOTE] 注意事项
> 1. Redis 最终被安装到 `/mystudy/redis/redis` 目录下，但 `make && make install` 会把一些 可执行文件  安装到 `/usr/local/bin/` 下：
> 	1. redis-server：
> 		1. Redis 服务器启动命令
> 	2. redis-cli：
> 		1. Redis 客户端工具
> 	3. redis-benchmark：
> 		1. Redis 性能测试工具
> 	4. redis-sentinel：
> 		1. Redis 集群相关
> 	5. redis-check-aof：
> 		1. 修复有问题的 AOF 文件
> 	6. redis-check-dump：
> 		1. 修复有问题的 dump.rdb 文件
> 2. 如果编译失败，请先执行以下命令清理旧文件，然后重新安装： 
```
cd /mystudy/redis/redis


make distclean
```

---


##### 2.2.3.5. 创建相关目录（配置、数据、日志、pid）

```
sudo mkdir -p /mystudy/redis/conf /mystudy/redis/data /mystudy/redis/logs /mystudy/redis/pid


sudo cp /mystudy/redis/redis/redis.conf /mystudy/redis/conf
```

---


























# 补充

## 1. Redis 配置文件

### 1.1. Redis 配置文件位置

Redis 可以在 `/mystudy/reids/reidis/redis.conf` 中进行配置

---


### 网络配置（NETWORK）

```
################################## NETWORK #####################################
bind
"""
1. Redis 监听的网络接口地址（网卡地址）
2. 127.0.0.1 -::1（默认）：
	1. 127.0.0.1 表示 IPV4 回环地址
	2. - 表示即使该地址当前在网卡上不可见，也不会导致启动失败
	3. ::1 表示 IPV6 回环地址
	4. 即仅允许来自本机的 IPv4 和 IPv6 回环地址访问，适合本地开发或非集群部署（客户端程序 与 Redis 在同一台服务器上，否则无法通过网络访问 Redis 服务）
3. 其他 具体 IP：
	1. 如 192.168.136.8
4. 0.0.0.0：
	1. 所有地址
"""


protected-mode
"""
1. 是否启用保护模式
2. yes（默认）：
	1. 表示 启用保护模式
	2. 当默认用户（default）未设置任何身份验证（即没有密码，也没有 ACL 授权）时，Redis 会仅接收来自本机回环地址（127.0.0.1、::1）或 Unix 文件通信接口的连接请求，拒绝任何非本机的 TCP 连接（外部连接会被拒绝）
	3. 即便我们没使用默认用户，而是其他用户（设置密码或 ACL 授权），也会被拒绝
3. no：
	1. 表示 不启用保护模式
"""


enable-protected-configs
"""
1. 是否允许修改某些高风险配置项，如 dir、dbfilename、save、appendonly、bind、requirepass、user 等
2. yes：
	1. 允许
3. no（默认）：
	1. 不允许
4. local：
	1. 仅允许来自本机的连接执行
"""


enable-debug-command
"""
1. 是否允许使用 DEBUG 命令族，如 DEBUG SEGFAULT（可以让 Redis 故意崩溃）、DEBUG RELOAD、DEBUG HTSTATS 等。这些命令多数是为开发和调试设计的，在生产环境中很危险
2. yes：
	1. 允许
3. no（默认）：
	1. 不允许
4. local：
	1. 仅允许来自本机的连接执行
"""


enable-module-command
"""
1. 是否允许使用 MODULE 命令族，如 MODULE LOAD、MODULE UNLOAD 等。加载模块的权限相当于动态向 Redis 注入 C 语言代码，安全风险极高。
2. yes：
	1. 允许
3. no（默认）：
	1. 不允许
4. local：
	1. 仅允许来自本机的连接执行
"""


port
"""
1. 监听的端口号
2. 6379（默认）：
	1. 单实例情况下，通常是 6379
3. 6380 ~ ...：
	1. 单机多实例情况下，通常是从 6380 开始，一个实例一个端口
4. 0：
	1. 关闭所有 TCP 监听，一般用于纯 Unix socket 模式（了解就好，集群搭建不能用 Unix socket 模式，只适用于单机模式）
"""


tcp-backlog
"""
1. TCP 监听队列的大小。当客户端连接 Redis 时，如果 TCP 三次握手完成，但 Redis 还没来得及处理这个连接，且 Redis 正在忙着处理其他请求，这些连接就会被放入 backlog 队列等待。如果 backlog 队列满了，新来的连接将被操作系统直接拒绝
2. 注意事项：
	1. 即便我们配置了 tcp-backlog 为 10000，也不代表真的能排队 10000 个连接，因为 Linux 会对这个值进行隐性限制，主要由两个系统参数控制：
		1. /proc/sys/net/core/somaxconn：
			1. 定义内核中 TCP 监听队列（backlog）的最大长度
			2. 查看该值：
				1. cat /proc/sys/net/core/somaxconn 
			3. 修改该值：
				1. 临时修改，重启失效：
					1. sudo sysctl -w net.core.somaxconn=1024
				2. 永久修改，重启生效：
					1. 编辑 /etc/sysctl.conf
					2. 加入 net.core.somaxconn = 1024
					3. 执行 sudo sysctl -p
		2. /proc/sys/net/ipv4/tcp_max_syn_backlog：
			1. 定义 TCP 半连接队列的最大长度
			2. 即 TCP 三次握手尚未完成的连接数，通常指服务器收到客户端 SYN 后回复 SYN+ACK，但尚未收到客户端最后 ACK 的连接，处于半连接状态，这些连接会被放到该队列中
			3. 查看该值：
				1. cat /proc/sys/net/ipv4/tcp_max_syn_backlog
			4. 修改该值：
				1. 临时修改，重启失效：
					1. sudo sysctl -w net.ipv4.tcp_max_syn_backlog=2048
				2. 永久修改，重启生效：
					1. 编辑 /etc/sysctl.conf
					2. 加入 net.ipv4.tcp_max_syn_backlog = 2048
					3. 执行 sudo sysctl -p
	2. 如果 tcp-backlog 设置为 1000，但 /proc/sys/net/core/somaxconn 是 500，那么 TCP 监听队列最大只能是 500。也就是说，要想支持更大的连接排队数，必须同时调整这几个参数，才能获得更高性能。
	3. Redis 和 MySQL 都有连接池，连接池维护的是已经建立好的 “长连接”，我们复用这些连接。你可能会问：既然连接已经建立好并复用，为什么还需要配置 tcp-backlog？难道后续还会建立新连接？答案是：当业务瞬时产生大量连接时，如果连接池内的连接不够用，且连接池还没达到最大连接数限制，会创建新的连接；另外，短连接模式下频繁建立连接也会产生新连接，部分客户端也可能不支持连接池。
	4. 当使用连接池时，客户端等待可用连接时，会在应用层维护一个等待队列（如线程等待或请求排队），这部分与内核的 tcp-backlog、somaxconn 等参数无关，属于应用层面的管理
	5. 那我们直接配置 /proc/sys/net/core/somaxconn 就好了，为什么还要配置 tcp-backlog 呢？这是因为：
		1. tcp-backlog 是 Redis 向内核 申请的监听队列长度，表示它期望内核为自己保留多少排队的连接数；
		2. 而 /proc/sys/net/core/somaxconn 是 操作系统内核的全局上限，用于限制所有程序（包括 Redis）最大能使用多少连接排队空间
"""


unixsocket
"""
1. 配置 Redis 是否通过 Unix socket 接收连接，不是通过 TCP 连接（了解就好）
2. Unix socket 只适用于本机通信，分布式或者 java 程序与 Redis 不在同机，不适合用这个，还是使用 tcp
"""


timeout
"""
1. 客户端空闲超时时间，以秒为单位
2. 0（默认）：
	1. 永不超时
3. 其他 具体值：
	1. 如 300 代表 5 分钟
"""


tcp-keepalive
"""
1. 每隔 多长时间 向客户端发送一次 TCP keepalive 包，以秒为单位。能有效防止死连接，例如有时客户端异常断开，但服务器端没收到关闭通知，以为连接还在使用，这时候连接变成 “死连接”，我们使用这种方式，就能探测客户端是否还在线，防止大量无效连接导致资源浪费
2. 300（默认）：
	1. 每隔 5 分钟 向客户端发送一次 TCP keepalive 包 
3. 其他 具体值：
	1. 如 60 代表 1 分钟
"""


socket-mark-id
"""

"""
```

---


### TLS / SSL 配置（TLS/SSL）

```

```

---


### 通用配置（GENERAL）

```
daemonize
"""
1. no：
	1. Redis 以前台方式运行（前台方式运行，就是不以守护进行运行）
2. yes：
	1. Redis 以后台方式运行，并创建 pid 文件
		1. 以后台方式运行，就是以守护进程运行
		2. pid 就是 process ID（进程编号）的缩写，每个正在运行的进程在操作系统里都有一个唯一的数字标识，叫做 PID。
"""


supervised
"""
1. Redis 如何与操作系统的 服务管理器 配合工作，主要是针对 Linux 下常见的两种服务管理工具：
	1. Upstart：
		1. 较老的服务管理系统
	2. systemd：
		1. 现代主流服务管理系统
2. 其实就是决定 Redis 启动时，是否以及如何向服务管理器报告“我已经准备好了”，从而实现与服务管理器的配合，让它们准确掌握 Redis 的运行状态，方便后续管理和监控。
	1. 重点是针对这种情况：如果不汇报状态，操作系统无法准确判断 Redis 是否启动成功。比如系统启动时，需要先启动 Redis，再启动依赖它的应用，但操作系统不知道 Redis 什么时候真正可用，可能会提前启动依赖服务，导致运行错误
	2. 这和进程本身无关，仅仅是告诉操作系统 “我准备好了” 与否。无论是否汇报，只要 Redis 启动了，你都能找到它的进程 ID，并通过该 PID kill 进程
3. no（默认）：
	1. 不启用任何服务管理器的交互，不汇报状态
4. upstart：
	1. 适配 Upstart 服务管理系统
5. systemd：
	1. 适配 systemd 服务管理系统
6. auto：
	1. 自动检测当前环境变量（如通过 UPSTART_JOB 或 NOTIFY_SOCKET 判断），自动选择合适的 supervision 方法。
"""


pidfile
"""
1. Redis 启动后，除了由操作系统分配进程号（PID），还会将该 PID 写入配置指定的 pidfile 文件中，主要用于后续进程管理和关闭操作。
2. 需要注意，即使不写入 pidfile，Redis 进程本身依然会正常启动和运行，你仍然可以通过 ps aux | grep redis 等方式查到它。pidfile 的意义在于：当我们编写 shell 脚本来启动或停止 Redis 时，通常通过 cat redis.pid 获取 PID 后执行 kill 命令；或者在一台服务器上同时运行多个 Redis 实例时，通过不同的 pidfile 区分各自的进程。
3. 另外，不需要提前手动创建 pid 文件，Redis 会在启动过程中自动生成该文件，只要确保其所在目录存在并具有写权限即可。
4. /var/run/redis_6379.pid（默认）：
	1. 默认路径
5. 其他 路径：
	1. 如 /mystudy/redis/pid/redis.pid
"""


loglevel
"""
1. 指定日志等级
2. debug：
	1. 最详细的级别，适合调试
3. verbose：
4. notice：
	1. 输出适度的重要消息，推荐用于生产环境
5. warning：
	1. 只记录关键或严重问题
6. nothing：
	1. 不记录任何日志（几乎没人用，除非你追求极致的性能或又其他监控机制）
"""


logfile
"""
1. 指定日志文件的位置
2. ""（默认）：
	1. 代表日志输出到标准输出
3. 其他 路劲：
	1. 例如 /mystudy/redis/logs/redis.log
4. 我们可以指定其他路径，如果你启用了 daemonize yes，而又没有指定日志文件，日志会被丢弃到 /dev/null。
	1. daemonize 和我们的日志有什么关系？
	2. 服务前台运行时，stdout 是终端，可以看到日志
	3. 服务后台运行时，stdout 被重定向到 /dev/null，除非我们设置了 loglevel
"""


syslog-enabled


syslog-ident


syslog-facility


crash-log-enabled


crash-memcheck-enabled


databases
"""
1. 数据库的数量
2. 可选值：
	1. 16（默认）：
		1. Redis 默认 16 个数据库
	2. 其他 数值：
		1. 例如 8
3. 注意事项：
	1. 虽然 Redis 提供了多个逻辑库，但很多实际项目中只使用 DB 0
	2. 在使用 Redis Cluster 时，也只能使用 DB 0
"""


always-show-logo
"""
1. 是否在 Redis 启动时，显示 ASCII 艺术字 logo
2. 可选值：
	1. yes
	2. no
"""


hide-user-data-from-log


set-proc-title
"""
1. 是否修改 Redis 进程名，以便在系统工具如 top、ps aux 中显示更丰富的运行时信息，方便管理员快速了解进程状态
2. 可选值：
	1. yes（默认）：
		1. Redis 会修改当前进程的标题（进程名），具体能修改成什么，看下面 proc-title-template
	2. no：
		1. Redis 进程名保持为启动时的原始名称，不显示附加的运行时状态信息
"""


proc-title-template
"""
1. 设置进程标题的模板
2. 可选值：
	1. {title} {listen-addr} {server-mode}（默认）：
		1. 默认模板
3. 候选值：
	1. {title}：
		1. 进程名称，如果是主进程就是执行命令名，子进程则显示子进程类型
	2. {listen-addr}：
		1. 监听的地址
	3. {server-mode}：
		1. 服务器特殊模式，比如 [sentinel] 表示哨兵模式，[cluster] 表示集群模式
	4. {port}：
		1. TCP 监听端口，若未开启则为 0
	5. {tls-port}：
		1. TLS 监听端口，若未开启则为 0
	6. {unixsocket}：
		1. Unix 域套接字路径，未使用则为空字符串。
	7. {config-file}：
		1. Redis 使用的配置文件名称
"""

locale-collate

```

---


### 1.2. RDB 配置（SNAPSHOTTING）

```
save 3600 1 300 100 60 10000
"""
1. 3600 秒内（1小时）有 ≥ 1 次写操作，进入 RDB 快照保存
2. 300 秒内（5分钟）有 ≥ 100 次写操作，进入 RDB 快照保存
3. 60 秒内有 ≥ 10000 次写操作，进入 RDB 快照保存
4. 可以写成 save "" 表示禁用 RDB 快照保存
"""


stop-writes-on-bgsave-error
"""
1. 是否当 Redis 在后台保存快照失败时停止接受写操作，目的是：强行提醒你 “写入的数据没有成功持久化”，别以为还很安全
2. 可选值：
	1. yes（默认）
	2. no：
		1. 如果你已经有很完善的监控，也可以改成 no
"""


rdbcompression
"""
1. 是否在保存 .rdb 文件时会使用 LZF 算法压缩字符串对象
2. 可选值：
	1. yes（默认）：
		1. 能够节省磁盘空间，代价是保存过程稍微多耗些 CPU
	2. no：
		1. 更省 CPU，尤其适合 CPU 紧张的环境，但文件会变大
"""


rdbchecksum
"""
1. 是否在 RDB 文件末尾添加一个 CRC64 校验和
2. 可选值：
	1. yes（默认）：
		1. 能够提高文件的可靠性，防止损坏导致读取出错
	2. no：
		1. 节省一点性能（10% 左右），但读取时不会校验是否损坏
""""


sanitize-dump-payload
"""
1. 控制 Redis 在加载 RDB 文件或执行 RESTORE 命令时，是否对压缩结构（如 ziplist、listpack）做完整性检查，防止出现不一致或崩溃。
2. 可选值：
	1. clients（默认）：
		1. 只对客户端连接恢复的数据检查，推荐使用
	2. yes：
		1. 总是检查，最安全
	3. no：
		1. 不检查，性能最高，但风险高
"""


dbfilename
"""
1. 保存快照的文件名
2. 可选值：
	1. dump.rdb（默认）
	2. 其他 文件名
"""


rdb-del-sync-files
"""
1. 当 Redis 实例既没有开启 RDB，也没有开启 AOF 时，是否要删除用于主从同步的临时 RDB 文件
2. 可选值：
	1. yes：
		1. 删除在主从同步过程中创建的 RDB 文件（就是我作为主节点，生成的用于发送给从节点并供其加载的那个 RDB 文件）
	2. no（默认）：
		1. RDB 同步文件保留在磁盘
3. 注意事项：
	1. 你可能会好奇，既然我没有开启持久化，为什么主从同步还能正常进行？其实，主从同步的核心是数据复制，这个过程并不依赖于持久化文件，而是基于内存中的数据快照来传输。
"""


dir 
"""
1. 保存 dump.rdb（和 AOF 文件）的工作目录（工作目录，不包含文件名）
2. 可选值：
	1. ./（默认）：
		1. 即当前目录
		2. 注意不是配置文件 redis.conf 所在的目录，而是指你启动 Redis 服务器时所在的那个目录
	2. 其他 目录：
		1. 例如 /mystudy/redis/data
"""
```

---


### 主从复制配置（REPLICATION）

```
replicaof
"""
1. 设置 当前 Redis 实例 为某个主节点的副本（从节点）
2. 可选值：
	1. 具体 IP 和 端口：
		1. 例如 replicaof 192.168.1.10 6379
3. 注意事项， 
	1. 只适用于传统的主从集群，Redis Cluster 集群下，Redis会忽略掉 replicaof 指令，因为 Redis Cluster 会维护自己的节点信息表（nodes.conf），主从关系记录在这里，而不是来自 replicaof 的配置。
"""


masteruser
masterauth
"""
1. ACL 的用户名和密码
2. 注意事项：
	1. 这是基于 ACL 的，在 Redis 6 之前，只需要写 masterauth ，里面写主节点的密码即可
"""


replica-serve-stale-data
"""
1. 是否当副本即使与主节点失联了，也继续响应客户端的读请求
2. 可选值：
	1. yes（默认）：
		1. 可能会返回旧数据
	2. no：
		1. 会拒绝大多数数据访问命令，只允许执行部分命令（如 INFO、REPLICAOF、AUTH、CONFIG 等管理命令）
"""


replica-read-only
"""
1. 副本是否是只读的（读写分离）
2. 可选值：
	1. yes（默认）：
		1. 读写分离
	2. no
"""




```





















### 安全配置










## 2. 常见分区方案

哈西分区、一致性哈希分区、节点取余分区

| 分区方式 | 适用场景        | 优点    | 缺点        |
| ---- | ----------- | ----- | --------- |
| 范围分区 | 按时间、数字范围查询多 | 查询区间快 | 数据分布可能不均  |
| 列表分区 | 离散字段（国家、类型） | 灵活    | 维护复杂，数据倾斜 |
| 哈希分区 | 数据均匀分布      | 负载均衡好 | 不适合范围查询   |
| 复合分区 | 复杂查询和负载均衡要求 | 灵活兼顾  | 复杂难管理     |
| 键值分区 | Redis、分布式缓存 | 自动透明  | 受限于哈希算法   |


### 2.1. 节点取余分区

直接对某个字段（如用户ID）的数值进行取余运算（mod），根据节点总数确定数据存放的位置。举例来说，4个节点时，用户ID为12345，则计算：
```
12345 % 4 = 1  → 数据落在节点 1
```

该方法实现简单，定位快速，只需知道节点数量和 key 即可完成映射。数据在节点间分布较均匀，有效避免热点节点。

但其缺点也较明显：一旦节点数发生变化（如从4扩容到5），几乎所有 key 的映射结果都会改变，导致大量数据迁移。此外，均匀分布虽好，但会导致连续范围的数据被分散到不同节点，降低范围查询效率。

----


### 2.2. 哈希分区

与节点取余分区类似，但不同之处在于，先对 key 进行哈希，无论 key 是数字还是字符串，都能统一处理。示例：
```
hash("user:123") % 3 = 1  → 数据落在节点 1（key 是 user:123）
hash("user:456") % 3 = 0  → 数据落在节点 0
```

此方式简单高效，如果哈希函数的质量很高，可以实现良好的数据均匀分布。缺点与节点取余分区相同，即节点数变化时大规模数据重映射，以及范围查询效率受限。

---


### 2.3. 一致性哈希分区

把 key 和节点都映射到一个“虚拟哈希环”上，哈希空间范围从 0 到 2³²-1。每个 key 会被分配到它在环上顺时针遇到的第一个节点上。下面是一个典型的哈希环示意，key1 对应到 Node1：

![](image-20250530160330698.png)

这种设计的好处是，当你增加或减少节点时，只有少量的 key 需要重新分配，避免了大量数据迁移的问题。不过需要注意的是，节点的位置也是通过哈希算出来的，分布不一定均匀，可能会导致某些节点压力比较大。比如，添加一个节点后，节点的分布可能像这样：

![](image-20250530160555827.png)

----


### 2.4. 虚拟一致性哈希分区

因为传统一致性哈希会出现有些节点“管得多”，有些“管得少”的情况，导致负载不平衡，所以引入了“虚拟节点”这个技巧。简单来说，就是给每个物理节点分配多个虚拟节点，让它们均匀分布在哈希环上，确保负载更平均。

![](image-20250530161002181.png)

---


## 文件结构树

使用 Alt + 数字键：
1. ├：Alt + 195
2. ─：Alt + 196
3. └：Alt + 192








