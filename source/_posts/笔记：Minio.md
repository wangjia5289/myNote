---
title: 笔记：Minio
date: 2025-05-13
categories:
  - 数据管理
  - 对象存储
  - Minio
tags: 
author: 霸天
layout: post
---
# 一、理论


### 1. Minio 核心概念

==1.Bucket==
Bucket 是存储 Object 的逻辑空间，每个 Bucket 之间的数据是相互隔离的，对用户而言，相当于存放文件的顶层**文件夹**


==2.Object==
Object 是存储到 Minio 中的基本对象，对用户而言，相当于**文件**

----


# 二、实操

### 把控全场

每台机器至少挂在2块磁盘，格式话为 XFS/ext4

至少4台节点、

扩容时最好一次性加偶数个磁盘/节点

---



### 1. 环境搭建

#### 1.1. 单机测试环境搭建

##### 1.1.1. 安装 Minio

```
docker run -d \
  --name minio-test \
  -p 9000:9000 \
  -p 9001:9001 \
  -e "MINIO_ROOT_USER=admin" \
  -e "MINIO_ROOT_PASSWORD=admin123" \
  minio/minio server /data --console-address ":9001"
"""
注意事项：密码长度至少 8 个字符
"""
```

---


##### 1.1.2. 访问 Minio 控制台

访问 Minio 控制台： http://192.168.136.8:9001

---


#### 分布式集群环境搭建

##### 架构说明

采用 多驱动＋纠删码 的架构，每个节点上挂载多块驱动器（存储目录），利用纠删码将对象数据拆分成「数据分片」和「校验分片」，从而实现冗余存储和容错能力。

简单来说，在一个 12 驱动器集群中，一个对象将被切分为 6 个数据分片和 6 个校验分片。即便任意 6 个驱动器（无论是数据分片还是校验分片）丢失，你仍可从剩余分片中完整恢复该对象。

> [!NOTE] 注意事：驱动器到底是什么？
> 1. 驱动器就是**存储目录**。记住了，驱动器 ≈ 存储目录，没别的玄学。
> 2. 不推荐将存储目录放在 Root（系统）盘下。原因很简单：MinIO 是高 IO、高吞吐的服务，频繁写入会让系统盘压力山大。如果你把数据写到系统盘（例如 `/data`、`/mnt` 下的目录），一旦空间爆满或磁盘故障，不仅存储服务会挂，可能连系统都崩，SSH 也登不上去了，连喊“救命”都来不及
> 3. 我们强烈推荐驱动器使用独立的磁盘或分区。什么意思呢？简单说，不要在一个已有的挂载目录下再创建子目录，而是直接将整块磁盘或一个分区挂载到驱动器目录本身。也就是说，存储目录就是磁盘或分区挂载点本身，而不是在挂载点下再套一个子目录。
> 4. 为什么这么推荐？虽然你技术上可以在一个磁盘或分区挂载的目录下创建多个子目录，并把这些子目录作为多个驱动器使用，但别忘了，MinIO 是为数据冗余和容错设计的。如果你把 12 个驱动器目录全放在同一个盘下，这个盘一旦挂了，12 个驱动器全都陪葬，数据恢复无从谈起。相反，如果每个驱动器目录都是独立挂载的磁盘或分区，就算某一块盘出问题，其他驱动器依旧健康，你的数据照样能恢复，稳得很。

---


##### 环境要求

1. ==硬件要求==：
	1. 4 台服务器
	2. 5 块磁盘
		1. Root 盘
		2. 四块数据盘（类型不限，推荐 SCSI）
2. ==系统要求==：
	1. Ubuntu 22.04
	2. 时间同步
	3. 开放 9000、9001 TCP端口
3. ==软件要求==：


---


##### 节点列表

Ø准备4台机器；（根据MinIO的架构设计，至少需要4个节点来构建集群，这是因为在一个N节点的分布式MinIO集群中，只要有N/2节点在线，数据就是安全的，同时，为了确保能够创建新的对象，需要至少有N/2+1个节点，因此，对于一个4节点的集群，即使有两个节点宕机，集群仍然是可读的，但需要有3个节点才能写数据；）

---



##### 时间同步



开放 9000、9001 端口

---



##### 安装 Minio Server

首先，在系统中创建目录 `/mystudy/minio`，并进入到该目录：
```
# 1. 创建目录
mkdir -p /mystudy/minio
"""
1. -p：
	1. 如果 /mystudy 目录本身不存在，可以加上 -p 参数，递归创建所有父目录
"""

# 2. 进入该目录
cd /mystudy/minio
```

接着，参考 [MinIO 官方下载页面](https://min.io/open-source/download?platform=linux) 进行安装，安装过程会将 `minio` 文件下载到**当前目录**，文件名即为 `minio`（没有扩展名）。

注意：此处下载的是 MinIO Server，不是其他工具，以下是 MinIO 的几个组件区别说明：
1. ==MinIO Server==：
	1. 要搭建 MinIO 服务端时，下载这个。
2. ==MinIO Client==：
	1. 命令行工具，用来管理 MinIO 或 AWS S3，例如上传、下载、建桶、设权限等。
	2. 想用命令行操作 MinIO（或 AWS S3）存储，就得下载它！
3. ==MinIO SDK==
	1. 代程序员在代码中操作 MinIO（上传/下载/授权等）时使用，根据语言选择对应的 SDK（如 Java、Python、Go）
![](image-20250514104455366.png)

> [!NOTE] 注意事项
> 1. 我们可以先在一台服务器上先下载 minio，然后拷贝到其他服务器上
> 2. 但是不要忘记执行另外两步操作

---


##### 磁盘格式化与挂载

```
# 1. 创建磁盘挂载目录
mkdir -p /mystudy/minio/disk{1..4}


# 2. 安装 xfsprogs 工具
sudo apt install -y xfsprogs


# 3. 列出所有块设备
lsblk


# 4. 格式化块（需要格式化为 XFS 或 ext4），这里格式化为 XFS
mkfs.xfs /dev/sdb

mkfs.xfs /dev/sdc

mkfs.xfs /dev/sdd

mkfs.xfs /dev/sde


# 5. 将磁盘挂载到挂载目录
mount /dev/sdb /mystudy/minio/disk1

mount /dev/sdc /mystudy/minio/disk2

mount /dev/sdd /mystudy/minio/disk3

mount /dev/sde /mystudy/minio/disk4


# 6. 再次列出所有块设备，查看是否挂载成功
lsblk
```

---


##### 创建 Shell 启动脚本

```
# 1. 进入 Minio 目录
cd /mystudy/minio


# 2. 创建 Shell 启动脚本
vim start-minio.sh
"""
1. 脚本内容：
#!/bin/bash

# 设置根账户和密码
export MINIO_ROOT_USER=admin
export MINIO_ROOT_PASSWORD=password

# 多机分布式模式
nohup /mystudy/minio/minio server \
  --config-dir /etc/minio \
  --address :9000 \
  --console-address :9001 \
  http://192.168.136.8/mystudy/minio/disk1  http://192.168.136.8/mystudy/minio/disk2  \
  http://192.168.136.8/mystudy/minio/disk3  http://192.168.136.8/mystudy/minio/disk4  \
  http://192.168.136.9/mystudy/minio/disk1  http://192.168.136.9/mystudy/minio/disk2  \
  http://192.168.136.9/mystudy/minio/disk3  http://192.168.136.9/mystudy/minio/disk4  \
  http://192.168.136.10/mystudy/minio/disk1 http://192.168.136.10/mystudy/minio/disk2 \
  http://192.168.136.10/mystudy/minio/disk3 http://192.168.136.10/mystudy/minio/disk4 \
  http://192.168.136.11/mystudy/minio/disk1 http://192.168.136.11/mystudy/minio/disk2 \
  http://192.168.136.11/mystudy/minio/disk3 http://192.168.136.11/mystudy/minio/disk4 \
  > /var/log/minio.log 2>&1 &

echo "MinIO 多机分布式集群已启动，后台运行…"

2.
"""


# 3. 添加可执行权限
chmod +x start-minio.sh
```

---


##### 启动分布式集群

```
# 1. 执行 Shell 启动脚本（每个节点都要启动）
/mystudy/minio/start-minio.sh


# 2. 补充：若提示 /mystudy/minio/start-minio.sh: cannot execute: required file not found。
# 2.1. 原因
可能是因为，我们在编辑器里创建／修改过这个脚本，行尾可能包含 \r\n（CRLF），而 Linux 期望的是 \n（LF），我们可以这样做

# 2.2. 安装 dos2unix 工具
apt-get install -y dos2unix

# 2.3. 将脚本转为 Unix 格式
dos2unix /mystudy/minio/start-minio.sh
```

---











# 补充

### 1. 相关网站

1. ==Minio 官方文档、官网==
	1. https://min.io/docs/minio/linux/developers/minio-drivers.html
2. ==Minio 官方下载页面==
	1. https://min.io/open-source/download?platform=linux

---


### 2. Minio 命名规范

1. ==Root 用户密码==：
	1. 密码长度至少 8 个字符
2. ==存储桶名==：
	1. 长度至少 3 个字符

---


##### 









# 数据一致

不是的，MinIO 并不采用 Raft 协议来同步各节点的元数据，而是依赖自有的“分布式锁 + 多数派（quorum）”机制来保证一致性：

1. **元数据存储在对象存储层**  
    MinIO 没有单独的元数据库，所有的元数据（bucket 列表、对象索引、版本信息等）都直接以对象形式写入磁盘。各节点启动时会基于一致性哈希（consistent hashing）探测到同一份磁盘集群，然后通过文件系统进行读写操作，实现了天然的扩展能力和高吞吐。([MinIO Blog](https://blog.min.io/minio-versioning-metadata-deep-dive/?utm_source=chatgpt.com "MinIO Versioning, Metadata and Storage Deep Dive"))
    
2. **dsync：轻量级的分布式锁**  
    MinIO 在内部使用一个名为 `dsync` 的库来实现跨节点的分布式锁。它的原理类似于 Raft，但更简单，旨在做锁服务：
    
    - 每个节点都和其它所有节点保持连接。
        
    - 发起锁请求时，会广播给所有节点，等待半数以上（`n/2+1`）节点响应成功才算获取到锁。
        
    - 释放锁时再广播一次，告知其它节点可以释放。  
        这种设计既能保证在网络分区或节点故障时仍能达成多数派共识，又避免了 Raft 那样的日志复制与状态机恢复开销。([GitHub](https://github.com/minio/dsync/blob/master/README.md?utm_source=chatgpt.com "README.md - minio/dsync - GitHub"))
        
3. **读写流程中的多数派确认**
    
    - **写入**：当客户端 PUT/DELETE 操作元数据或对象时，MinIO 会把数据切分（若启用了纠删码）、并行写入多块磁盘，再等待多数块写成功后才返回成功给客户端。
        
    - **读取**：同样，从多数块中重建或直接读取，不依赖某个“主”节点。
        
4. **为何不用 Raft？**  
    Raft 虽然可提供强一致性，但在需要超大规模（上千节点、上万磁盘）的高并发环境下，日志复制和领导者选举的开销会显著影响性能。MinIO 更追求极简、高效的读写路径，因此选择了基于 dsync 的 quorum 方案，而不是 Raft。
    

---

**小结**：

- MinIO 的元数据同步并非通过 Raft 日志复制，而是把元数据当成普通对象写入磁盘，并依赖 `dsync` 分布式锁＋多数派确认来保证并发一致性。
    
- 不同节点之间无领导者（leader）；每个节点都可接受请求，只要多数派达成共识即可。
    

这样既避免了单点，也能在极大规模下保持高性能。希望能解答你的疑惑！


















