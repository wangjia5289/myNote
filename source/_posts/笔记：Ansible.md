---
title: 笔记：Ansible
date: 2025-03-15
categories:
  - 自动化
  - Ansible
tags: 
author: 霸天
layout: post
---
### 1. Ansible 概述

Ansible 是一款强大的自动化运维工具，提供配置管理、应用部署和任务执行等多种功能。通过 Ansible，我们可以高效地管理和部署多个主机，实现例如一键配置 YUM 源、自动安装服务、快速部署 Docker 等操作。

----


### 2. 环境准备

完成 Ubuntu 操作系统的安装和基本配置见：[Ubuntu 操作系统的安装和基本配置](https://blog.wangjia.xin/2025/03/15/%E7%AC%94%E8%AE%B0%EF%BC%9ALinux/)

---


### 3. 配置 SSH 密钥对认证

#### 3.1. SSH 密钥对认证概述

在使用 Ansible 管理远程主机之前，推荐在控制节点上配置 SSH 密钥对认证，这样 Ansible 在自动化操作时就无需每次输入密码，确保了免密登录。

---


#### 3.2. 生成 SSH 密钥对

在控制机上执行以下命令生成 SSH 密钥对（如果尚未存在密钥对）。默认情况下，私钥和公钥分别保存在 `~/.ssh/id_rsa` 和 `~/.ssh/id_rsa.pub`
```
sudo ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```
![](image-20250315161627569.png)

1. `ssh-keygen -t rsa -b 4096 -C "wangjia5289@163.com"`：
	1. 这个命令生成一个 RSA 类型的 SSH 密钥对，密钥长度为 4096 位，并为密钥加上注释 `"wangjia5289@163.com"`，通常用于标识密钥所属的用户
2. `Enter file in which to save the key (/root/.ssh/id_rsa)`：
	1. 系统询问你希望将密钥保存到哪个文件路径。默认情况下，私钥将保存在 `/root/.ssh/id_rsa`，公钥保存在 `/root/.ssh/id_rsa.pub`
	2. 如果你直接按回车，密钥将保存在默认路径。如果你输入其他路径，则会将密钥保存到你指定的位置
3. `Enter passphrase (empty for no passphrase)`：
	1. 系统询问你是否要为私钥设置密码短语（passphrase）
	2. 如果设置了密码短语，每次使用密钥时都需要输入该密码，增加了安全性
	3. 自动化场景会不设密码，可以直接按回车（即留空），表示没有密码
4. `Enter same passphrase again`：
	1. 系统要求你再输入一次密码短语，以确保你输入正确

---

#### 受控主机指纹追加写进 SSH 的信任列表

提前告诉 SSH，这个主机是可信的，别再问我 yes/no 了
```
ssh-keyscan -H 192.168.136.9 >> ~/.ssh/known_hosts && ssh-keyscan -H 192.168.136.10 >> ~/.ssh/known_hosts
```

---


#### 3.3. 将公钥复制到受控主机

使用 `ssh-copy-id` 将公钥传送到受控主机（例如 IP 为 192.168.136.9）：
```
ssh-copy-id -i ~/.ssh/id_rsa.pub root@192.168.136.9

ssh-copy-id -i ~/.ssh/id_rsa.pub root@192.168.136.10
```
![](image-20250516130541453.png)

---


#### 3.4. 测试 SSH 免密登录

测试控制机和受控主机之间是否能免密连接，若能直接登录则说明 SSH 密钥对认证配置成功。
```
# 1. 测试免密连接到受控主机
ssh root@192.168.136.9


# 2. 退出
exit
```
![](image-20250315163806382.png)

> [!NOTE] 注意事项
> 1. `ssh root@192.168.136.9` 是登录到该服务器，测试没问题要 `exit` 退出

---


### 4. 安装 Ansible

在我们的控制机上安装 Ansible：
```
# 1. 安装 Ansible
sudo apt install ansible -y


# 2. 验证安装
ansible --version
```

---


### 5. 配置 Ansible Inventory（主机清单）

编辑控制机的 `/etc/ansible/hosts`（默认主机清单文件），把你的服务器信息加进去：
```
[servers]
server1 ansible_host=192.168.136.9 ansible_user=root
server2 ansible_host=192.168.136.10 ansible_user=root
```

1. `server1`：
	1. 为主机指定别名，在执行 Ansible 任务时，可以使用 `server1` 代替 `192.168.1.101`
	2. 这样做的好处是提升可读性，如果 IP 发生变化，只需修改对应的 IP 地址即可
2. `ansible_host=192.168.1.101`：
	1. 填写受控主机的 IP 地址
3. `ansible_user=root`：
	1. 指定 SSH 的连接用户，这里是 `root` 用户
	2. 如果受控主机不允许 `root` 直接 SSH 登录，你可能需要改成普通用户，比如 `ansible_user=admin`

> [!NOTE] 注意事项
> 1. `[servers]` 是组名，你可以用组来管理不同的服务器集群，例如：
```
[webservers]
server1 ansible_host=192.168.1.101 ansible_user=root
server2 ansible_host=192.168.1.102 ansible_user=root

[dbservers]
server3 ansible_host=192.168.1.201 ansible_user=root
```

---


### 6. 测试 SSH 连接

使用 Ansible 内置的 `ping` 模块测试所有主机的连通性：
```
ansible all -i /etc/ansible/hosts -m ping -u root
```
![](image-20250315174245349.png)

---


### 7. 远程执行命令
```
ansible serverName -m shell -a "各种 shell 命令"
```

> [!NOTE] 注意事项
> 1. 这个命令仅在 `serverName` 组内的受控主机上执行，操作机自身不会执行。若希望操作机也参与执行，可使用 `"localhost:serverName"` 指定目标主机。
> 2. 如果有多个命令要执行，可以通过 && 来连接：
```
ansible servers -m shell -a "shell命令1 && shell命令2 && shell命令3"
```

---


### 8. 使用 Ansible 为多服务器配置 YUM 源
```
# 为 Centos7 受控主机配置阿里 YUM 源
ansible servers -m shell -a "sudo mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak"


# 为 Centos7 受控主机创建新的 YUM 源并配置为阿里云的 YUM 镜像源
ansible servers -m shell -a "echo -e '[base]\nname=CentOS-7 - Base - mirrors.aliyun.com\nbaseurl=http://mirrors.aliyun.com/centos/7/os/x86_64/\ngpgcheck=1\ngpgkey=http://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7\n\n[updates]\nname=CentOS-7 - Updates - mirrors.aliyun.com\nbaseurl=http://mirrors.aliyun.com/centos/7/updates/x86_64/\ngpgcheck=1\ngpgkey=http://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7\n\n[extras]\nname=CentOS-7 - Extras - mirrors.aliyun.com\nbaseurl=http://mirrors.aliyun.com/centos/7/extras/x86_64/\ngpgcheck=1\ngpgkey=http://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7' | sudo tee /etc/yum.repos.d/CentOS-Base.repo"
```

---


### 9. 使用 Ansible 为多服务器安装 Docker

```
# 为受控主机安装国内 Docker YUM 源
ansible servers -m shell -a "sudo yum install -y yum-utils && sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo"


# 受控主机安装最新版本的 Docker
ansible servers -m shell -a "yum install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin"


# 检查 Docker 的版本
ansible servers -m shell -a "docker --version"


# 启动 Docker 并设置开机自启
ansible servers -m shell -a "systemctl enable --now docker"

# 配置 Docker 加速器
ansible servers -m shell -a "sudo mkdir -p /etc/docker && echo '{\"registry-mirrors\": [\"https://dh-mirror.gitverse.ru\"]}' | sudo tee /etc/docker/daemon.json && sudo systemctl daemon-reload && sudo systemctl restart docker && docker info"
```

---
