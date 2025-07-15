---
title: 笔记：Minio 工具
date: 2025-07-14
categories:
  - 数据管理
  - 数据的组织方式
  - 对象存储
  - Minio
  - Minio 工具
tags: 
author: 霸天
layout: post
---
## mc 命令行客户端

### mc 命令行客户端概述

`mc` 是一个命令行客户端，只需连接 MinIO 集群中的任意一个节点，便可统一管理整个集群，包括所有节点的桶、对象、用户、策略、健康状态等内容。

----


### mc 命令行客户端安装

参考 [MinIO 官方下载页面](https://min.io/open-source/download?platform=linux) 进行安装，安装过程会将 `mc` 文件下载到**当前目录**，文件名即为 `mc`（没有扩展名）：
![](image-20250714170717756.png)

> [!NOTE] 注意事项
> 1. 会话级别以太网临时代理：
```
export http_proxy="http://172.20.10.3:7890" && export https_proxy="http://172.20.10.3:7890" && export no_proxy="localhost,127.0.0.1,.svc,.cluster.local,192.168.136.0/24,10.96.0.1,10.244.0.0/16" && export HTTP_PROXY=$http_proxy && export HTTPS_PROXY=$https_proxy && export NO_PROXY=$no_proxy
```

---


### 将 mc 添加到环境变量

如果你不想每次都写 `./mc`，可以把 `mc` 加到 PATH 中：
```
cd /mystudy/mc


sudo cp ./mc /usr/local/bin/
```