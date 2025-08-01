# 一、理论



1. **初次同步时**，主机会执行一次 RDB 快照，生成 `.rdb` 文件；
    
2. 这个 `.rdb` 文件会通过网络发送给从节点；
    
3. 从节点拿到这个文件后会加载它，恢复数据；
    
4. 主节点上的这个临时 `.rdb` 文件会被保留下来，保存在主节点的磁盘上，**哪怕这个 Redis 实例本身并不持久化。**


我们控制的就是这个 rdb文件是不是被保留

从节点接收到的 RDB 文件会**临时保存在磁盘**，加载数据后就删除，不会像主节点那样长期保留。




![](image-20250601150551372.png)


### 1.2. 数据持久化 

#### 1.2.1. RDB 方式

##### 1.2.1.1. RDB 概述

RDB 是 Redis 默认使用的数据持久化方式，通过将某个时刻的内存数据快照保存为一个二进制文件（默认名为 `dump.rdb`），每次保存的是整个数据库的完整状态。  

它的优点是：
1. 生成的文件体积较小且经过压缩，适合长期存储
2. 恢复时只需加载单个 RDB 文件，速度非常快，特别适合快速恢复大量数据。  

它的缺点是：
1. 如果 Redis 异常宕机，最近几分钟的数据可能会丢失；
2. 此外，生成快照时会 fork 出子进程，快照过程开销较大，数据量大时可能会对性能产生一定影响。  

> [!NOTE] 注意事项
> 1. RDB 是默认开启的
> 2. RDB 在保存 RDB 文件时，父进程唯一需要做的就是 fork 出一个子进程，接下来的工作全部由子进程来错，所以 RDB 持久化方式可以最大化 Redis 的性能

![](image-20250530141401641.png)

---


##### RDB 配置

```
################################ SNAPSHOTTING ################################
save 3600 1 300 100 60 10000
"""
1. 3600秒内（1小时）有≥1次写操作，进入 RDB 快照保存
2. 300秒内（5分钟）有≥100次写操作，进入 RDB 快照保存
3. 60秒内有≥10000次写操作，进入 RDB 快照保存
4. 可以写成 save "" 表示禁用 RDB 快照保存
"""


stop-writes-on-bgsave-error
"""
1. 是否当 Redis 在后台保存快照失败时停止接受写操作，目的是：强行提醒你 “写入的数据没有成功持久化”，别以为还很安全
2. yes（默认值）：
	1. 启用
3. no：
	1. 如果你已经有很完善的监控，也可以改成 no，Redis 会继续接受写入请求
"""


rdbcompression
"""
1. 是否在保存 .rdb 文件时会使用 LZF 算法压缩字符串对象
2. yes（默认值）：
	1. 能够节省磁盘空间，代价是保存过程稍微多耗些 CPU
3. no：
	1. 更省 CPU，尤其适合 CPU 紧张的环境，但文件会变大
"""


rdbchecksum
"""
1. 是否在 RDB 文件末尾添加一个 CRC64 校验和
2. yes（默认值）：
	1. 能够提高文件的可靠性，防止损坏导致读取出错
3. no：
	1. 节省一点性能（10%左右），但读取时不会校验是否损坏
""""


sanitize-dump-payload
"""
1. 控制 Redis 在加载 RDB 文件或执行 `RESTORE` 命令时，是否对压缩结构（如 ziplist、listpack）做完整性检查，防止出现不一致或崩溃。
2. no：
	1. 不检查，性能最高，但风险高
3. yes：
	1. 总是检查，最安全
4. clients（默认）：
	1. 只对客户端连接恢复的数据检查，推荐使用
"""


dbfilename
"""
1. 保存快照的文件名，默认是 dump.rdb
"""


rdb-del-sync-files
"""
1. 当 Redis 实例既没有开启 RDB，也没有开启 AOF 时，是否要删除用于主从同步的临时 RDB 文件
2. yes：
	1. 删除在主从同步过程中创建的 RDB 文件（就是我作为主节点，生成的用于发送给从节点并供其加载的那个 RDB 文件）
3. no（默认）：
	1. RDB 同步文件保留在磁盘，安全风险略高
"""


dir 
"""
1. 保存 dump.rdb（和 AOF 文件）的工作目录（工作目录，不包含文件名）
2. 默认是 ./，即当前目录
"""
```

---


##### 触发 RDB

RDB 快照的生成会依据配置项 `save 3600 1 300 100 60 10000` 自动触发。但除了自动触发，我们也可以通过命令手动触发保存。
```
# 1. 同步阻塞 
SAVE


# 2. 异步非阻塞
BGSAVE
```

> [!NOTE] 注意事项
> 1. 除上两种情况，还有几种情况会进行触发 RDB 持久化：
> 	1. 执行 `SHUTDOWN` 并且没有设置开启 AOF 持久化时
> 	2. 主从复制的时候，主节点自动触发

----

##### 其他 RDB 相关命令

```
# 1. 获取最后一次成功执行快照的时间
LASTSAVE


# 2. 检查和修复 dump.rdb 文件
redis-check-rdb dump6379.rdb
```

----



#### AOF 方式

##### AOF 概述

AOF（Append Only File）会将 Redis 所执行的每一条写命令（即修改数据的命令）按顺序追加记录到日志文件中（默认文件名为 `appendonly.aof`），通过“日志回放”的方式来实现数据恢复。

它的优点是：
1. 支持配置 `appendfsync everysec`，实现每秒写盘一次，即使 Redis 崩溃，最多只会丢失 1 秒内的数据，适合对数据安全性要求较高的场景
2. AOF 文件是纯文本格式，可读性强，便于审计和故障排查；

它的缺点是：
1. 每一条写操作都会记录，长时间运行后日志文件会持续增长，占用磁盘；
2. 恢复数据需要**回放所有写操作**，数据量大时恢复速度较慢；
3. 为防止日志无限膨胀，Redis 需要定期对 AOF 文件进行重写（rewrite），压缩内容，仅保留恢复当前数据状态所需的最小命令集

> [!NOTE] 注意事项
> 1. AOF 默认是关闭的，需要在配置文件中手动开启
> 2. 实际部署中推荐 RDB 与 AOF 一起开启，以兼顾恢复速度和数据安全，唯一的代价是：对性能的开销会比单独使用一种方式略大一些。

---














































# 二、实操

## 1. 全场把控

### 1.1. 线程特性


### Redis 主从复制是怎么同步数据的？




---


### 1.2. 数据持久化

#### 1.2.1. RDB 方式

##### 1.2.1.2. RDB 操作方式

RDB 的触发方式包括自动触发和手动触发两种：
==1.自动触发==


==2.手动触发==
```
# 1. 同步阻塞 
SAVE


# 2. 异步非阻塞
BGSAVE
```



----


#### 1.2.2. AOF 方式

##### 1.2.2.1. AOF 概述


----


##### 1.2.2.2. AOF 操作方式

---


#### 1.2.3. 纯缓存方式

即只在缓存中存数据，不进行持久化，只需要我们同时关闭 RDB 和 AOF

---


### 1.3. 数据同步

#### 1.3.1. 单体一致性



---


#### 1.3.2. 分布式一致性

Redis 本身并未内建分布式一致性机制。在常规的 Redis 主从集群中，虽然通过异步复制可以在节点之间同步数据，但这种同步仅能实现“最终一致性”，无法保障严格意义上的“分布式强一致性”。

若希望在多个 Redis 实例之间实现强一致性，可考虑以下方案：
1. ==使用 Redis Raft 模块==：
	1. [RedisRaft](https://github.com/RedisLabs/redisraft) 是由 RedisLabs 开源的一个模块，简单来说就是 Redis + Raft 算法
	2. 其旨在为 Redis 提供强一致性、故障自动恢复能力以及真正的高可用集群模式（即强同步复制），适用于对一致性要求较高的场景，如银行账务系统、事务处理系统等。

---













# 二、实操

## 1. 环境搭建

### 1.1. 单机测试环境搭建

### 1.2. 其他业务

#### 1.2.1. Redis 持久化


















# 工具

### 0.1. Redis Insight



### 0.2. Redis Commander

这是一个轻量级的 Redis Web 管理器，支持通过 Docker 快速启动。启动后，只需打开浏览器访问 http://192.168.136.8:8081 ，配置 Redis 地址，即可轻松实现可视化操作。
```
docker run -d --name redis-commander -p 8081:8081 rediscommander/redis-commander
```

---





# 补充

### 0.1. 相关网站

1. Redis、Redis Insight 下载
	1. https://redis.io/downloads/#software
2. Redis Raft
	1. https://github.com/RedisLabs/redisraft
