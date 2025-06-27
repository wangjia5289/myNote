---
title: 笔记：Webmin
date: 2025-04-05
categories:
  - 操作系统
  - Webmin
tags: 
author: 霸天
layout: post
---
# 一、理论

### 1. Webmin 概述

**Webmin** 是一个基于网页的系统管理工具，主要用于管理和监控 Unix-like 操作系统（如 Linux 和 FreeBSD）的各种服务和系统资源，例如可以监控和管理以下内容：
1. ==系统资源==：
    - CPU 使用率、内存、磁盘空间等。
    - 系统负载、进程和服务状态。
2. ==网络监控==：
    - 查看网络流量、配置网络接口、管理防火墙等。
    - 查看系统的 IP 地址、路由表和 DNS 设置。
3. ==磁盘管理==：
    - 管理硬盘分区、文件系统、挂载点等。
    - 查看磁盘使用情况和文件系统的健康状态。
4. ==服务和守护进程==：
    - 启动、停止或重启服务（如 Apache、Nginx、MySQL 等）。
    - 配置服务的启动项和日志设置。
5. ==用户和权限管理==：
    - 添加、删除、修改系统用户和组。
    - 设置权限、访问控制和 sudo 权限。
6. ==日志文件==：
    - 查看系统日志、服务日志。
    - 配置日志文件的轮转和备份策略。
7. ==安全管理==：
    - 配置防火墙（如 iptables 或 firewalld）。
    - 设置 SSH 访问和密钥管理。

---


# 二、实操

### 1. 安装 Webmin

以下是基于 Ubuntu 操作系统的安装步骤：
```
# 1. 安装必要依赖
sudo apt install -y perl libnet-ssleay-perl openssl libio-pty-perl libauthen-pam-perl


# 2. 下载 Webmin 安装包
wget https://sourceforge.net/projects/webadmin/files/webmin/2.303/webmin_2.303_all.deb


# 3. 安装 Webmin
sudo dpkg --install webmin_2.303_all.deb


# 4. 启动和设置开机自启动
sudo systemctl start webmin && sudo systemctl enable webmin


# 5. 开放 Webmin 端口（默认10000）
sudo ufw allow 10000/tcp       
```

> [!NOTE] 注意事项
> 1. 您可以访问 SourceForge 查找最新版本：[Webmin 下载页面](https://sourceforge.net/projects/webadmin/files/)。
> 2. 注意要下载 deb 后缀的文件

---


#### 2. 访问 Webmin

在 Windows 的浏览器访问：
```
https://<Linux-server-ip>:10000（192.168.136.8:10000）
```

> [!NOTE] 注意事项
> 1. 登录时使用的是 Ubuntu 系统中的用户名和密码。你可以使用不同的用户进行登录，例如 `root`、`user` 或 `admin`。




