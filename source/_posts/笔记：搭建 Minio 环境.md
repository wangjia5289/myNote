---
title: 笔记：搭建 Minio 环境
date: 2025-07-08
categories: 
tags: []
author: 霸天
layout: post
---
## 单机测试环境

### Docker 单机部署 Minio

<font color="#92d050">1. 创建宿主机数据挂载目录</font>
```
mkdir -p /mystudy/data/minio
```


<font color="#92d050">2. 启动 Minio 容器</font>
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

 
<font color="#92d050">3. 访问 Minio 控制台</font>

访问： http://192.168.136.8:9001

----


## Minio 分布式集群










