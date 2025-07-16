---
title: 笔记：搭建 Minio 环境
date: 2025-07-08
categories: 
tags: []
author: 霸天
layout: post
---
## 1. 单机测试环境

### 1.1. 创建宿主机数据挂载目录

```
mkdir -p /mystudy/data/minio
```

---


### 1.2. 启动 Minio 容器

```
docker run -d \
  --name minio-test \
  -p 9000:9000 \
  -p 9001:9001 \
  -e "MINIO_ROOT_USER=root" \
  -e "MINIO_ROOT_PASSWORD=wq666666" \
  -v /mystudy/data/minio:/data \
  minio/minio:RELEASE.2025-06-13T11-33-47Z \
  server /data --console-address ":9001"
```

> [!NOTE] 注意事项
> 1. 密码长度至少 8 个字符
> 2. `server /data --console-address ":9001"` 是 MinIO 启动命令中的一个子命令，用来启动 MinIO 对象存储服务的：
> 	1. server：
> 		1. 启动 MinIO 存储服务
> 	2. /data
> 		1. MinIO 存储数据的根目录
> 	3. --console-address ":9001"
> 		1. 启动控制台 Web UI，监听 9001 端口
> 3. 9000 端口是 Minio 端口，9001 端口是 Web UI 端口
> 4. 会话级别以太网临时代理：
```
export http_proxy="http://172.20.10.3:7890" && export https_proxy="http://172.20.10.3:7890" && export no_proxy="localhost,127.0.0.1,.svc,.cluster.local,192.168.136.0/24,10.96.0.1,10.244.0.0/16" && export HTTP_PROXY=$http_proxy && export HTTPS_PROXY=$https_proxy && export NO_PROXY=$no_proxy
```

---

### 1.3. 访问 Minio Web UI
 
访问： http://192.168.136.8:9001

----


## 2. Minio 分布式集群

### 环境特性

#### 节点关系







### 2.1. 环境要求

#### 2.1.1. 节点要求

官方建议 MinIO 分布式集群至少部署 4 个节点。这是因为纠删码模式运行时最少需要 4 块磁盘才能生效，而如果只提供 4 块磁盘，那么需要每个节点都挂载一块磁盘（不推荐采用一个节点挂载 4 块磁盘，或两个节点共计 4 块磁盘的方式部署，因为这种架构下单点故障风险较高，无法实现真正意义上的数据高可用与分布式冗余）

节点数量是奇数还是偶数并不关键，关键在于保证最终的磁盘总数为偶数。例如，7 个节点每个挂载 2 个磁盘，总磁盘数为 14 个。通常推荐使用偶数个节点，主要是基于每个节点仅有一块磁盘的情况，但我们一般都是部署偶数个节点。

在早期版本中，MinIO 集群中建议最多使用不超过 32 块磁盘。即使每个节点仅挂载一块磁盘，那么集群也最多不能超过 32 个节点。

MinIO 集群中，虽然节点数量的增加有助于提升容量和并发性能，但同时也带来一系列挑战与潜在问题：
1. 元数据一致性成本上升：
	1. MinIO 采用去中心化架构，所有节点共同管理元数据（如 bucket 配置、版本控制、对象锁等）；
		1. 元数据由节点负责，而非磁盘维护，相关信息以 Key-Value 形式存储于每个节点本地的 `.minio.sys` 目录中。
	2. 随着节点增多，元数据的广播与同步耗时显著增加，尤其在网络存在延迟或抖动时更为明显。
	3. 节点数量增加，网络负载成倍上升，易形成“网状拓扑”瓶颈，影响集群整体性能。
2. 运维成本提升：
	1. 节点越多，需监控的服务日志、性能指标与健康状态也随之增多，运维复杂度提升。
3. 故障排查难度加大：
	1. 一次写入失败，可能涉及多个节点的 I/O、网络或锁状态，排查路径长，定位难度显著上升。
4. 启动与恢复耗时增长：
	1. MinIO 启动时需要加载全部节点信息、初始化纠删码映射、执行健康检查
	2. 节点越多，启动时间越长，若某些节点响应缓慢，可能导致整个集群初始化受阻或卡顿。

在新版本中，MinIO 已取消对最大磁盘数 32 的限制，单个集群可支持无限数量的磁盘。也就是说，单个集群可支持无限数量的节点，但出于性能、稳定性和运维复杂度的综合考虑，仍推荐将节点数控制在 32 个以内。

一般来说，4 到 8 个节点属于中型部署规模，10 到 16 个节点则适用于高并发、大容量的场景，而超过 16 个节点的部署则属于超大规模集群，常见于企业级云平台、公有云提供商。

当需要构建大规模分布式存储系统时，不建议持续扩展单个 MinIO 集群的节点数量，而是推荐通过部署多个独立的 MinIO 集群，并借助分布式协调机制，将多个集群组成联邦架构，实现更灵活的横向扩展与统一管理。

> [!NOTE] 注意事项
> 1. 推荐每个节点仅运行一个 Minio 实例，以确保集群稳定性

---


#### 2.1.2. 磁盘要求

MinIO 要求至少使用 4 块磁盘，这是因为纠删码模式在运行时至少需要 4 块磁盘才能正常工作。推荐每台机器挂载 2 块磁盘，不建议超过 4 块磁盘。

推荐每台机器挂在的磁盘数量一致，并且总磁盘数为偶数个，这是因为偶数数量更有利于提升纠删码的存储效率、容错能力与扩展时的平衡性，这一经验来自实际部署中的性能表现。

在早期版本中，MinIO 集群建议最多使用不超过 32 块磁盘。新版本已取消对最大磁盘数 32 的限制，单个集群可支持无限数量的磁盘，但仍建议控制在 32 块以内。

> [!NOTE] 注意事项
> 1. 文件系统推荐使用 XFS 或 ext4

---


#### 2.1.3. CPU 要求

纠删码计算和数据加密对 CPU 负载较大，CPU 性能直接影响整体吞吐和响应延迟。建议至少配置 4 核 CPU 起步，以更好地支持并发请求及加密计算（如 TLS/SSL 加解密和纠删码编码/解码）。

一般而言，小型或中型集群以 4 核 CPU 起步，大型高并发集群则建议配备 8 核或以上的 CPU。

---


#### 2.1.4. 内存要求

较大内存有助于提升整体性能和响应速度，尤其在高负载场景下效果更为明显。一般而言，小型或中型集群以 8GB 内存起步，大型高并发集群则建议配备 16GB 或以上的内存。

如果启用了大量加密（如 SSE-C、SSE-KMS）或复杂的纠删码配置，建议适当增加内存；此外，在使用 Kubernetes Operator 或多租户环境时，内存需求也会相应提升。

---


#### 2.1.5. 网络要求

一般而言，小型或中型集群以千兆（1 Gbps）网络带宽起步，大型高并发集群则建议配备 10 Gbps 或以上的网络带宽。

单个集群中的所有节点必须部署在同一局域网内，确保低延迟、低丢包率。不建议跨机房部署同一个集群，因为网络延迟和不确定性将严重影响纠删码的数据一致性与性能。

---


#### 2.1.6. 时间要求

MinIO 分布式集群中，各节点之间必须保持时间同步。

> [!NOTE] 注意事项
> 1. 在跨多集群时，时间同步更加敏感，必须进行时间同步。

---


#### 2.1.7. 用户要求

在生产环境中，建议新建一个专门的系统用户（例如命名为 `minio`）来运行 MinIO 服务，不建议赋予其过高权限（如 root 权限），以降低安全风险。通过限制权限，可以有效防止因服务漏洞导致整个系统被攻破。

该用户需对存储数据目录具备读写权限，并能够访问 MinIO 监听的端口。

----


#### 2.1.8. 端口要求

| 端口   | 类型  | 说明        |
| ---- | --- | --------- |
| 9000 | TCP | HTTP 服务端口 |
| 9001 | TCP | Web 控制台端口 |

----


#### 2.1.9. 扩容要求

扩容时应尽量保证最终的磁盘总数为偶数。建议每次增加偶数个节点（不是强制要求）

当需要构建大规模分布式存储系统时，不建议持续扩展单个 MinIO 集群的节点数量，而是推荐通过部署多个独立的 MinIO 集群，并借助分布式协调机制，将多个集群组成联邦架构，实现更灵活的横向扩展与统一管理。

---


#### 2.1.10. Swap 分区要求

swap 分区，简单来说，是操作系统在系统内存不足时，将某些程序不活跃或暂时不需要的数据移动到 swap 分区（位于硬盘），以释放内存资源供当前活跃程序使用。

由于 swap 分区依赖硬盘，其读写速度远低于内存。如果系统频繁使用 swap，会造成大量磁盘 I/O，严重影响系统性能，甚至导致系统响应迟缓或无响应。因此，在性能敏感的场景（如数据库、实时计算、大内存服务器）中，通常建议禁用 swap，确保数据始终在高速内存中运行。

我们 MinIO 要求关闭 swap 分区。

----


### 2.2. 环境准备

#### 2.2.1. Ubuntu 版本（22.04）

```
lsb_release -a
```

----


#### 2.2.2. 节点列表

| IP             | 磁盘                 | CPU | 内存  |
| -------------- | ------------------ | --- | --- |
| 192.168.136.8  | Root 盘、四块 SCSI 数据盘 | 2 核 | 4GB |
| 192.168.136.9  | Root 盘、四块 SCSI 数据盘 | 2 核 | 4GB |
| 192.168.136.10 | Root 盘、四块 SCSI 数据盘 | 2 核 | 4GB |
| 192.168.136.11 | Root 盘、四块 SCSI 数据盘 | 2 核 | 4GB |

-----


#### 2.2.3. 时间同步

```
command -v chrony >/dev/null 2>&1 || sudo apt-get install -y chrony


sudo systemctl enable chrony && sudo systemctl start chrony
```

> [!NOTE] 注意事项
> 1. 会话级别以太网临时代理：
```
export http_proxy="http://172.20.10.3:7890" && export https_proxy="http://172.20.10.3:7890" && export no_proxy="localhost,127.0.0.1,.svc,.cluster.local,192.168.136.0/24,10.96.0.1,10.244.0.0/16" && export HTTP_PROXY=$http_proxy && export HTTPS_PROXY=$https_proxy && export NO_PROXY=$no_proxy
```

----


#### 2.2.4. 开放端口

```
sudo ufw allow 9000/tcp


sudo ufw allow 9001/tcp
```

> [!NOTE] 注意事项
> 1. 如果使用云服务器，同样需要开放安全组

---


#### 2.2.5. 关闭 Swap 分区

```
# 1. 编辑 /etc/fstab
vim /etc/fstab
"""
# 将此内容进行注释（/swap 开头的）
# /swap.img       none    swap    sw      0       0
"""


# 2. 立即关闭 Swap 分区
swapoff -a
```

----


#### 2.2.6. 安装需要的工具

```
// 1. dos2unix，用于将文件转为 Unix 格式
command -v dos2unix >/dev/null 2>&1 || sudo apt-get install -y dos2unix


// 2. xfsprogs，用于将磁盘块设备格式化为 xfs 文件系统
command -v xfsprogs >/dev/null 2>&1 || sudo apt install -y xfsprogs
```

> [!NOTE] 注意事项
> 1. 会话级别以太网临时代理：
```
export http_proxy="http://172.20.10.3:7890" && export https_proxy="http://172.20.10.3:7890" && export no_proxy="localhost,127.0.0.1,.svc,.cluster.local,192.168.136.0/24,10.96.0.1,10.244.0.0/16" && export HTTP_PROXY=$http_proxy && export HTTPS_PROXY=$https_proxy && export NO_PROXY=$no_proxy
```

----


### 2.3. 环境搭建

#### 2.3.1. 创建软件安装路径

```
mkdir -p /mystudy/minio
"""
1. -p：
	1. 如果 /mystudy 目录本身不存在，可以加上 -p 参数，递归创建所有父目录
"""
```

----


#### 2.3.2. 安装 Minio Server

```
cd /mystudy/minio
```

接着参考 [MinIO 官方下载页面](https://min.io/open-source/download?platform=linux) 进行安装，安装过程会将 `minio` 文件下载到当前目录，文件名即为 `minio`（没有扩展名）。

需要注意的是，此处下载的是 MinIO Server，以下是 MinIO 的几个组件区别说明：
1. MinIO Server：
	1. 用于部署 MinIO 服务端时下载，用来搭建对象存储系统。
2. MinIO Client：
	1. 一款命令行工具，用于管理 MinIO 或兼容的 AWS S3 服务，支持上传、下载、创建桶、设置权限等操作。
	2. 如果你希望通过命令行操作 MinIO 或 AWS S3，就需要安装它！
3. MinIO SDK
	1. 面向开发者，用于在代码中操作 MinIO（如上传、下载、授权等），需根据所用编程语言选择对应版本（如 Java、Python、Go）
![](image-20250715220301330.png)

> [!NOTE] 注意事项
> 1. 安装 amd64 还是 arm64，要看系统 CPU 架构
```
uname -m
“”“
1. x86_64：
	1. amd64
2. aarch64：
	1. arm64
”“”
```

----


#### 2.3.3. 磁盘格式化与挂载

```
# 1. 创建磁盘挂载目录
mkdir -p /mystudy/minio/disk{1..4}


# 2. 列出所有块设备
lsblk


# 3. 格式化块（需要格式化为 XFS 或 ext4），这里格式化为 XFS
mkfs.xfs /dev/sdb

mkfs.xfs /dev/sdc

mkfs.xfs /dev/sdd

mkfs.xfs /dev/sde


# 4. 将磁盘挂载到挂载目录
mount /dev/sdb /mystudy/minio/disk1

mount /dev/sdc /mystudy/minio/disk2

mount /dev/sdd /mystudy/minio/disk3

mount /dev/sde /mystudy/minio/disk4


# 5. 再次列出所有块设备，查看是否挂载成功
lsblk
```

----


#### 2.3.4. 编写 Shell 启动脚本

```
# 1. 进入 Minio 目录
cd /mystudy/minio


# 2. 创建 Shell 启动脚本
vim start-minio.sh
"""
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

"""

// 2. 将脚本转为 Unix 格式
dos2unix /mystudy/minio/start-minio.sh


# 3. 添加可执行权限
chmod +x start-minio.sh
```

> [!NOTE] 注意事项
> 1. Shell 启动脚本中的最后一个空白行并非笔误，它是必须添加的内容。

----


#### 2.3.5. 启动分布式集群

```
/mystudy/minio/start-minio.sh
```

> [!NOTE] 注意事项
> 1. 每个节点都要启动

----






















