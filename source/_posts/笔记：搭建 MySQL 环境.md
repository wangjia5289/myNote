---
title: 笔记：搭建 MySQL 环境
date: 2025-06-29
categories:
  - 数据管理
  - 关系型数据库
  - MySQL
  - 搭建 MySQL 环境
tags: 
author: 霸天
layout: post
---
## 单机测试环境

### Docker 环境部署

<span style="background:#fff88f">1. 创建宿主机数据挂载目录</span>
```
mkdir -p /mystudy/data/mysql
```


<span style="background:#fff88f">2.启动 MySQL 容器</span>
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

> [!NOTE] 注意事项
> 1. 安装 Docker 时虽然已配置 Docker 代理，但这里任然需要配置命令行代理，因为需要与对应网站进行 TLS 握手
> 2. 不要忘记将网络设置为我们设置的 Docker 代理的网络
```
export http_proxy="http://172.20.10.3:7890" && export https_proxy="http://172.20.10.3:7890" && export no_proxy="localhost,127.0.0.1,.svc,.cluster.local,192.168.136.0/24,10.96.0.1,10.244.0.0/16" && export HTTP_PROXY=$http_proxy && export HTTPS_PROXY=$https_proxy && export NO_PROXY=$no_proxy
```

---



