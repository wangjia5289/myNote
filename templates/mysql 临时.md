


# 实操

## 1. 全场把控

### 1.1. 数据同步

#### 1.1.1. 单体一致性

MySQL 中的 binlog（二进制日志），可以形象地理解为数据库的“操作流水账”或“录音带”，它详细记录了所有修改数据的操作，比如 `INSERT`、`UPDATE`、`DELETE`（`DML` 语句）、数据库结构的变更（`DDL` 语句）、用户和角色的管理（`DCL` 语句）。当数据库意外挂掉或崩溃时，binlog 允许我们将之前的操作“回放”一遍，从而恢复数据到崩溃前的状态。

在传统的主从集群中，主库会将所有写操作写入 binlog，从库再按顺序读取并执行这些日志，实现数据同步，其详细步骤如下：
1. 客户端发起 `INSERT`、`UPDATE`、`DELETE` 或其他修改数据或结构的 SQL 请求到 MySQL 主库。
2. MySQL 执行该 SQL 语句时，并将操作写入事务日志（`redo log` 和 `undo log`），以确保事务的原子性与持久性。
3. 当事务提交成功后，MySQL 会将这次操作对应的事件记录写入 binlog 文件。
4. 从库通过 IO 线程 与主库建立连接，并向主库请求 binlog 数据。
5. 主库将 binlog 中的新内容发送给从库，从库收到后先写入本地的 relay log（中继日志） 文件。
6. 从库的 SQL 线程 读取 relay log 中的内容，按顺序执行其中的操作事件，更新自身数据。
7. 通过不断地拉取和重放 binlog，从库的数据最终与主库保持一致，实现主从同步。
![](downloadImage.png)

> [!NOTE] 注意事项
> 1. “单体一致性” 问题：
> 	1. binlog 复制最多只能算是 **“单体一致性”** ，因为 binlog 的生成完全发生在主库本地，它准确记录了主库上的数据变更顺序和状态，也就是说，主库写入 binlog 就算提交成功了。
> 	2. 而对于分布式集群来说，虽然从库可以通过 binlog 跟进主库的变更，但这最多只能实现 **“最终一致性”** ，无法达到分布式强一致性的要求。这是因为：分布式一致性要求在任何时间，从任何一个健康节点读取的数据都是相同的结果，但 MySQL 只能保证数据“迟早”会一致。
> 	3. 举个例子：主节点写入 binlog 并返回客户端后，异步地发送给从库；如果此时主库突然宕机，而从库还没来得及复制这条数据，当该从库顶替为新主库时，所有从库都应该读取这个新主库的日志文件，那么这条数据就会丢失。虽然概率较低，但确实存在数据丢失的风险。
> 	4. 相比之下，Raft 这种强一致性协议要求超过半数节点确认写入后，事务才算成功提交，不同于 MySQL 写入 binlog 即算成功的机制。因此，binlog 更适合称为单体一致性，而非分布式一致性。
> 2. “单体一致性” 是什么：
> 	1. 指的是本地节点内的数据操作具备强一致性，而跨节点的数据同步情况则不受本地节点控制
> 3. 仍然可以搭建主从集群：
> 	1. 需要注意的是，尽管如此，binlog 复制依然非常强大，也是最主流的数据同步方式之一。我们可以基于它搭建主从集群，数据同步的一般流程是先进行一次全量复制，然后再通过 binlog 实现增量同步，也就是主库新增一条数据，从库就跟着同步一条。
> 	2. 如果是金融交易、秒杀场景，是绝对不能出现假数据的，那我们必须要追求分布式一致性，否则，直接搭建主从集群足以满足我们99%的需求了

---


#### 1.1.2. 分布式一致性

MySQL 本身并不内建分布式一致性机制。虽然在常规的 MySQL 主从集群中，binlog 复制可以实现集群节点间的数据同步，但这种同步只能达到“最终一致性”，无法保证严格的“分布式一致性”。举例来说，在主从架构中，binlog 是异步复制到从库的，如果主库突然宕机，从库可能还没来得及同步最新数据，导致数据不一致。

如果想在多个 MySQL 实例之间实现强一致性，可以考虑以下方案：
1. ==搭建 Galera Cluster==：
	1. 这是一个支持 MySQL 的多主集群方案，通过写集复制（Write Set Replication）实现同步复制，保证节点间数据的一致性。

也可以使用其他分布式数据库：
1. ==TiDB==：
	1. 兼容 MySQL 协议的分布式数据库，内部用 Raft 协议来实现分布式一致性。

---


## 2. 环境搭建

### 2.1. 单机环境

==1.创建宿主机数据挂载目录==
```
mkdir -p /mystudy/data/mysql
```


==2.启动 MySQL 容器==
```
docker run -d \
  --name my_mysql \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=wq666666 \
  -v /mystudy/data/mysql:/var/lib/mysql \
  mysql:8.0 \
  --character-set-server=utf8mb4 \
  --collation-server=utf8mb4_unicode_ci
```

---


### 2.2. 主从集群

#### 2.2.1. 主从集群概述

主从集群下的所有架构都是基于 binlog 复制实现的：主库写 binlog，从库实时追踪并同步。

---


#### 2.2.2. 一主多从架构

##### 2.2.2.1. 架构说明


无法自动进行故障转移

---


##### 节点列表

| IP             | 主机名         | 角色  |
| -------------- | ----------- | --- |
| 192.168.136.8  | mysql-node1 | 主库  |
| 192.168.136.9  | mysql-node2 | 从库  |
| 192.168.136.10 | mysql-node3 | 从库  |

---


##### 环境准备

==1.硬件要求==


==2.确定 Ubuntu 版本（22.04）==
```
lsb_release -a
```


==3.安装相关工具==
```
# 1. dos2unix（若执行 Shell 脚本）
command -v dos2unix >/dev/null 2>&1 || sudo apt-get install -y dos2unix


# 2. chrony（时间同步）
command -v chrony >/dev/null 2>&1 || sudo apt-get install -y chrony
```


==4.时间同步==
```
# 1. 安装 chrony
sudo apt install -y chrony


# 2. 启动并开启自启动 chrony
sudo systemctl enable chrony && sudo systemctl start chrony
```


==5.开放 3306 TCP 端口==
```
sudo ufw allow 3306/tcp
```

---


##### 安装 MySQL

```
sudo apt update && sudo apt install -y mysql-server
```




# 工具

## 1. MyCat2

```
启动一个 mysql


# 1. 创建 /mystudy/mycat2 目录
mkdir -p /mystudy/mycat2


# 1. 将 ES 安装包上传至 /mystudy/es 目录


# 2. 进入 /mystudy/es 目录
cd /mystudy/mycat2


# 3. 删除已存在 ES 目录
rm -rf /mystudy/mycat2/mycat


# 4. 解压
unzip mycat2-install-template-1.21.zip -d /mystudy/mycat2

rm -rf mycat2-install-template-1.21.zip

cd /mystudy/mycat2/mycat/bin

chmod +x *

mv /mystudy/mycat2/mycat2-1.22-release-jar-with-dependencies.jar /mystudy/mycat2/mycat/lib



```


```
vim /mystudy/mycat2/mycat/conf/datasources/prototypeDs.datasource.json
```

### 1.1. 数据源配置
```
{
	"dbType":"mysql",
	"idleTimeout":60000,
	"initSqls":[],
	"initSqlsGetConnection":true,
	"instanceType":"READ_WRITE",
	"maxCon":1000,
	"maxConnectTimeout":3000,
	"maxRetryCount":5,
	"minCon":1,
	"name":"prototypeDs",
	"password":"123456",
	"type":"JDBC",
	"url":"jdbc:mysql://localhost:3306/mysql?useUnicode=true&serverTimezone=Asia/Shanghai&characterEncoding=UTF-8",
	"user":"root",
	"weight":0
}
```



### 1.2. MyCat2 语法

#### 1.2.1. MyCat2 语法概述

MyCat2 的配置本质上就是编辑配置文件，完全可以手动完成。但为了更方便，我们也可以使用注释配置语法，让配置过程更直观高效。

---


#### 1.2.2. 重置配置

```
/*+ mycat:resetConfig{} */
```

----

#### 1.2.3. 用户相关

==1.创建用户==
```
/*+ mycat:createUser{
  "username":"user",
  "password":"",
  "ip":"127.0.0.1",
  "transactionType":"xa"
} */
```


==2.删除用户==
```
/*+ mycat:dropUser{
  "username":"user"} */
```


==3.查询用户==
```
# 1. 查询所有用户
/*+ mycat:showUsers */
```

---


#### 1.2.4. 数据源相关

==1.创建数据源==
```
/*+ mycat:createDataSource{
  "dbType":"mysql",
  "idleTimeout":60000,
  "initSqls":[],
  "initSqlsGetConnection":true,
  "instanceType":"READ_WRITE",
  "maxCon":1000,
  "maxConnectTimeout":3000,
  "maxRetryCount":5,
  "minCon":1,
  "name":"dc1",
  "password":"123456",
  "type":"JDBC",
  "url":"jdbc:mysql://127.0.0.1:3306?useUnicode=true&serverTimezone=UTC&characterEncoding=UTF-8",
  "user":"root",
  "weight":0
} */;
```


==2.删除数据源==
```
/*+ mycat:dropDataSource{
  "dbType":"mysql",
  "idleTimeout":60000,
  "initSqls":[],
  "initSqlsGetConnection":true,
  "instanceType":"READ_WRITE",
  "maxCon":1000,
  "maxConnectTimeout":3000,
  "maxRetryCount":5,
  "minCon":1,
  "name":"dc1",
  "type":"JDBC",
  "weight":0
} */;
```


==3.查询数据源==
```
# 1. 查询所有数据源
/*+ mycat:showDataSources{} */
```














### 1.3. MyCat2 配置文件

#### 1.3.1. 用户

用户配置文件位于 `/mystudy/mycat2/mycat/conf/users` 目录，每个用户对应一个独立的配置文件，内容示例如下：
```
{
	"dialect":"mysql",
	"ip":null,
	"password":"123456",
	"transactionType":"proxy",
	"username":"root"，
	"isolation":3
}
"""
1. dialect：
	1. 数据库类型
2. ip：
	1. IP 白名单，一般写 null（能连接到 MyCat 的 IP 白名单）
3. password：
	1. MyCat 用户密码
4. transactionType：
	1. 事务模式，可选值：
	2. proxy（默认）：
		1. 本地事务，适用于涉及多个数据库的事务，提交阶段失败可能导致不一致，但兼容性最佳。
	3. xa：
		1. 分布式事务，需确认存储节点是否支持 XA。
5. username：
	1. MyCat 用户名
6. isolation：
	1. 事务隔离级别，可选值：
	2. 1：读未提交
	3. 2：读已提交
	4. 3（默认）：可重复读
	5. 4：可串行化
"""
```

---


#### 1.3.2. 数据源

数据源配置文件位于 `/mystudy/mycat2/mycat/conf/datasources` 目录，每个数据源对应一个独立的配置文件，内容示例如下：
```
{
	"dbType":"mysql",
	"idleTimeout":60000,
	"initSqls":[],
	"initSqlsGetConnection":true,
	"instanceType":"READ_WRITE",
	"maxCon":1000,
	"maxConnectTimeout":3000,
	"maxRetryCount":5,
	"minCon":1,
	"name":"prototypeDs",
	"password":"wq666666",
	"type":"JDBC",
	"url":"jdbc:mysql://192.168.136.8:3306/?useUnicode=true&serverTimezone=Asia/Shanghai&characterEncoding=UTF-8",
	"user":"root",
	"weight":0,
	"queryTimeout":30,//mills
}
"""
1. dbType：  
     1. 数据库类型
2. idleTimeout：  
     1. 空闲连接超时时间（单位：毫秒）。  
     2. 达到超时未使用的连接会被关闭，节省资源。
3. initSqls：  
     1. 初始化 SQL 语句列表。  
     2. 一般用于连接创建后执行一些预设命令，通常为空数组。
4. initSqlsGetConnection：  
     1. 是否在每次获取连接时执行 initSqls。  
     2. true 表示每次取连接都执行；false 只在初始化时执行。
5. instanceType：  
     1. 实例读写类型，可选值：  
     2. READ：只读  
     3. READ_WRITE（默认）：读写
6. maxCon：  
     1. 最大连接数。  
     2. 控制连接池中最多能开的连接数量。
7. maxConnectTimeout：  
     1. 建立连接的最大超时时间（单位：毫秒）。  
     2. 超过此时间视为连接失败。
8. maxRetryCount：  
     1. 连接失败时的最大重试次数。
9. minCon：  
     1. 最小连接数。  
     2. 启动时会预热并保持至少这么多连接。
10. name：  
     1. 数据源的逻辑名称。  
     2. 在 MyCAT 配置中用于标识该实例。
11. password：  
     1. 数据库角色登录密码（不是 MyCat 角色）
12. type：  
     1. 数据源连接类型，常用值为 JDBC。
13. url：  
     1. 数据库连接地址（JDBC URL），常用值为 jdbc:mysql://127.0.0.1:3306/mysql?useUnicode=true&serverTimezone=Asia/Shanghai&characterEncoding=UTF-8"
14. user：  
     1. 数据库用户名（不是 MyCat 角色）
15. weight：  
     1. 负载均衡权重。  
     2. 为 0 表示该节点不参与负载均衡。
16. queryTimeout：  
     1. SQL 查询超时时间（单位：毫秒）。  
     2. 超过时间则中断当前查询。
"""
```

---







