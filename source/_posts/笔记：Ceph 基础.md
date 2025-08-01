---
title: 笔记：Ceph 基础
date: 2025-04-08
categories:
  - 数据管理
  - 数据的组织方式
  - 统一存储
  - Ceph
  - Ceph 基础
tags: 
author: 霸天
layout: post
---
### Geph 组件

1. ==Monitors（监管）==：
	1. Monitor负责监控集群状态并保持集群的一致性，确保每个节点（例如 OSD 或 MDS）都正常工作，如果出现问题，它会警告Managers，并协助修复。
	2. 它们维护一个关于整个集群的状态数据，保存关于集群配置、节点状态等信息。
	3. - Monitor 保存集群的状态信息（如各 OSD 节点、PG（Placement Group）和 CRUSH map 等），并确保集群的一致性。
	4. Ceph Monitor (守护进程 ceph-mon) 维护集群状态的映射，包括监视器映射、管理器映射、OSD 映射、MDS 映射和 CRUSH 映射。这些映射是 Ceph 守护进程相互协调所需的关键集群状态。监视器还负责管理守护进程和客户端之间的身份验证。通常至少需要三个监视器才能实现冗余和高可用性。基于 paxos 协议实现节点间的信息同步。
2. ==Managers（管理）==：
	1. **Manager**（简称 MGR）是 Ceph 集群中的管理组件，提供集群的**管理、监控、统计和配置功能**。
	2. 它们监控集群的性能、资源利用情况，并且管理集群的一些高级功能，比如扩展、修复和优化。
	3. 通过 **Manager**，管理员可以查看集群的健康状态，并作出调整，甚至可以通过 **Web 界面**进行集群管理。
	4. Ceph 管理器 (守护进程 ceph-mgr) 负责跟踪运行时指标和 Ceph 集群的当前状态，包括存储利用率、当前性能指标和系统负载。Ceph 管理器守护进程还托管基于 Python 的模块来管理和公开 Ceph 集群信息，包括基于 Web 的 Ceph 仪表板和 REST API。高可用性通常至少需要两个管理器。基于 raft 协议实现节点间的信息同步。
3. ==OSDs（存储工人）==：
	1. 是负责宿主机上实际的数据存储和管理，其主要任务是存储数据对象并处理对这些数据对象的读写操作。
	2. 还负责数据的副本管理、数据恢复、数据重新平衡等工作。
	3. - **OSD**（Object Storage Daemon）是 Ceph 存储的核心组件，负责数据的**存储、读取、复制和恢复**。
	- 每个 OSD 进程管理一个磁盘（或磁盘的某一部分），并且会维护数据的副本，确保数据的冗余性和可靠性。
	- 当某个 OSD 发生故障时，其他 OSD 会自动修复数据的丢失，保持数据的高可用性。
	- Ceph OSD (对象存储守护进程，ceph-osd) 存储数据，处理数据复制、恢复、重新平衡，并通过检查其他 Ceph OSD 守护进程的心跳来向 Ceph 监视器和管理器提供一些监控信息。通常至少需要 3 个 Ceph OSD 来实现冗余和高可用性。本质上 osd 就是一个个 host 主机上的存储磁盘。
4. ==MDSs（文件管理员）==
	1. **MDS** 是 Ceph 文件系统（CephFS）的核心组件，负责存储和管理**文件系统的元数据**（如文件和目录的结构、权限等）。
	2. 它确保 CephFS 文件系统的路径、文件名、目录结构和权限等信息的快速访问。
	3. 每个 MDS 处理 CephFS 中的一部分元数据。多个 MDS 可以共同工作，来提高文件系统的性能。
	4. 如果你把 Ceph 看作一个超级大文件系统，**MDS** 就像是**文件管理员**，负责管理文件目录的“标签”和“标签夹”，确保你能快速找到需要的文件并操作它们。
	5. Ceph 元数据服务器（MDS [Metadata Server]、ceph-mds）代表 Ceph 文件系统存储元数据。Ceph 元数据服务器允许 POSIX（为应用程序提供的接口标准）文件系统用户执行基本命令（如 ls、find 等），而不会给 Ceph 存储集群带来巨大负担。
5. ==PGs（数据分组）==
	1. **PG**（Placement Group）是 Ceph 中用于数据分布的基本单位。它们将存储数据的对象划分成多个组，每个 PG 包含多个数据对象
	2. 每个 PG 会将数据分布到多个 OSD 上，这样可以提高存储的效率和容错性。
	3. PG 负责决定每个数据对象存储在哪些 OSD 上，并处理这些对象的副本。
	4. 你可以把 PG 想象成 **Ceph 中的数据分类箱**。每个数据对象会被分配到一个或多个 PG 里面，然后这些 PG 会负责把数据分发到不同的 OSD 上。这样，数据既能高效存储，又能确保在故障时不丢失。
	5. 简单来说，一个 PG 包含多个 OSD
	6. PG 全称 Placement Groups，是一个逻辑的概念，一个 PG 包含多个 OSD。引入 PG 这一层其实是为了更好的分配数据和定位数据。写数据的时候，写入主 osd，冗余两份。
	7. PG 会对数据自动进行备份？


### 数据存储流程
![](image-20250409101551563.png)



![](image-20250409101918985.png)


### Ceph 数据冗余机制

![](image-20250409101837245.png)
PG 下的 OSD 也会选主，主要是为了磁盘之间的数据一致性同步和像 mon 报告自身状态，使用的是 Hash 算法将数据 object 对象到 PG，将 PG里面的数据分散到不同的OSD



#### Ceph 存储类型
![](image-20250409102406023.png)




### Ceph 存储原理



### ceph 版本选择

cpeh 只有三种版本：x.0.z是开发版，x.1.z是候选版，x.2.z（稳定、修正版）





### Ceph 实战
#### 前言
在Ceph系统的搭建过程中，会出现各种意想不到或者预想到问题，其主要原因就是由于 ceph 快速更新和 底层操作系统的库文件、内核、gcc 版本之间的兼容性之间的问题，其实对 Ubuntu 的兼容性还号就算整个过程中每一步都没问题，还是会出现各
种问题，这些问题不仅仅在网上找不到，甚至在官网中找不到，甚至玩ceph数年的人都解决不了。

尤其是，就算你第一次成功后，第二次重试就会出现问题。所以，如果出现问题怎么办？一步一步踏踏实实的进行
研究，分析解决问题，并进行总结并梳理成册就可以了。

ceph的环境部署是非常繁琐的，所以，官方帮我们提供了很多的快捷部署方式。
https://docs.ceph.com/en/pacific/install/










![](image-20250409110737249.png)


![](image-20250409110924759.png)

























