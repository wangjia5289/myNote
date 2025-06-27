---
title: 笔记：NFS
date: 2025-03-22
categories:
  - 数据管理
  - 文件存储
  - NFS
tags: 
author: 霸天
layout: post
---
### 0. 导图：[Map：NFS](../../maps/Map：NFS.xmind)

---


### 1. NFS 概述

NFS（Network File System）是一种允许网络中的计算机共享文件的协议。<font color="#c00000">它允许客户端通过网络访问远程服务器的文件系统，像本地文件一样进行读写操作</font>。

---


### 2. NFS 服务器端配置

#### 2.1. NFS 服务器概述

NFS 服务器实际上是一个普通的服务器，区别在于它安装了 NFS 服务并对外提供 NFS 文件共享，因此我们称之为 NFS 服务器。

通常情况下，NFS 不需要由 Kubernetes 管理或配置，它是一个独立的外部服务，负责数据存储和共享。

---


#### 2.2. 安装 NFS 服务
```
# 1. 安装 NFS 服务（Ubuntu）
sudo apt-get install nfs-kernel-server   


# 2. 启动 NFS 服务
systemctl start nfs-kernel-server


# 3. 设置 NFS 服务开机自启动
systemctl enable nfs-kernel-server


# 4. 检查 NFS 服务状态
sudo systemctl status nfs-kernel-server


# 5. 开放 NFS 端口（Ubuntu）
sudo ufw allow 2049/tcp
sudo ufw allow 111/tcp
sudo ufw allow 111/udp
sudo ufw allow 20048/tcp
sudo ufw allow 20048/udp
sudo ufw allow 875/tcp
sudo ufw allow 875/udp


# 6. 重新加载防火墙
sudo ufw enable
sudo ufw reload


# 补充：关闭防火墙（永久禁用）
sudo ufw disable
```

---


#### 2.3. 创建普通目录
```
# 1. 创建一个普通目录
mkdir -p /k8s-nfs/redis/data


# 2. 设置目录权限，使得该目录可以由 NFS 服务以较低权限访问
sudo chown nobody:nogroup /k8s-nfs/redis/data
sudo chmod 755 /k8s-nfs/redis/data
```

---


#### 2.4. 配置共享目录

NFS 使用 `/etc/exports` 文件来定义共享目录以及访问权限。你需要在这个文件中添加一行，指定哪些客户端可以访问这个共享目录。
```
# 1. 编辑 /etc/exports
vim /etc/exports


# 2. 在 /etc/exports 文件中配置共享目录
/k8s-nfs/redis/data 192.168.126.0/24(rw,sync,no_subtree_check)


# 3. 重新导出所有共享目录，使新 NFS 配置生效
sudo exportfs -a
```

> [!NOTE] 注意事项：`/k8s-nfs/redis/data 192.168.126.0/24(rw,sync,no_subtree_check)`
> 1. `/k8s-nfs/redis/data`：NFS 共享的目录路径
> 2. `192.168.136.0/24`：允许访问该共享目录的客户端 IP 范围（此处是允许整个子网的客户端）
> 3. `(rw,sync,no_subtree_check)`：
> 	- 允许 NFS 客户端进行**读写访问**
> 	- 确保所有写操作同步完成后再返回结果
> 	- 禁用子目录检查，提高性能

---


### 3. NFS 客户端配置

#### 3.1. 安装 NFS 客户端服务

```
# 1. 安装 NFS 客户端服务（安装后通常会自动启动并设置为开机自启动，无需手动设置）
sudo apt-get install nfs-common                          # Ubuntu


# 2. 确保宿主机节点能通过 NFS 访问 NFS 服务器
showmount -e 192.168.126.112
```

---


#### 3.2. 设置临时目录挂载

临时目录挂载：电脑重启后不再进行目录挂载
```
# 1. 临时挂载
sudo mount 192.168.126.100:/nfs/redis/data /mnt/nfs-redis


# 2. 检查挂载是否成功
df -h
```
1. ==192.168.126.100:/nfs/redis/data==：
	1. 远程 NFS 服务器的共享路径
2. ==/mnt/nfs-redis==
	2. 本地挂载点，挂载后可以在这里访问远程共享的数据

---


#### 3.3. 设置自动目录挂载（可选）

自动目录挂载：希望在客户端重启后自动挂载 NFS 共享，可以编辑 `/etc/fstab` 文件，添加如下行：
```
# 1. 自动挂载
192.168.1.100:/nfs/redis/data  /mnt/nfs-redis  nfs  defaults  0  0


# 2. 重启主机


# 3. 检查挂载是否成功
df -h
```
1. ==nfs==:
	1. 文件系统类型，这里是 nfs
2. ==defaults==：
	1. 使用默认挂载参数，包括 rw、hard、intr 等
3. ==0==：
	1. 表示不需要 dump 备份
4. ==0==：
	1. 表示 fsck 在启动时不会检查这个挂载点

---


#### 3.4. 测试目录挂载是否有效

```
sudo touch /mnt/testfile                            # 读权限
sudo mkdir /mnt/testdir                             # 写权限
```

---


#### 3.5. 补充：卸载目录挂载
```
sudo umount /mnt/nfs-redis
```

---



