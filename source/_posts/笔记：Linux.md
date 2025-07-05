---
title: 笔记：Linux
date: 2025-03-15
categories:
  - 操作系统
  - Linux
tags: 
author: 霸天
layout: post
---
## 1. 导图：[Map：Linux](../../maps/Map：Linux.xmind)

---


## 2. Ubuntu 操作系统的安装和基本配置

### 2.1. 安装 Ubuntu 操作系统

这里选择从 [清华大学开源软件镜像站](https://mirrors.tuna.tsinghua.edu.cn/ubuntu-releases/) 下载并安装 `ubuntu-22.04-desktop-amd64.iso`

---


### 2.2. 网络设置

#### 2.2.1. Windows 启动 VMware NAT Service 服务（☆）

![](image-20250315102024363.png)

> [!NOTE] 注意事项
> 1. 以后发现问题不对，就来找这个，很可能是因为这个东西未启动

---


#### 2.2.2. 配置 VMware 软件的虚拟网络编译器

1.==点击 编辑 ->->-> 虚拟网络编译器==
![](image-20250315102345834.png)


==2.记住此处子网IP==
![](image-20250315102523602.png)


==3.记住 NAT 的网关 IP==
![](image-20250315102727828.png)

![](image-20250315102806921.png)


==4.配置 NAT 模式==
![](image-20250315102923234.png)

![](image-20250315103044276.png)

---


#### 2.2.3. 配置虚拟机的网络适配器

![](image-20250315103315818.png)
![](image-20250315103335885.png)


---


#### 2.2.4. Windows 配置 VMware Network Adapter VMnet8 

![](image-20250315103512620.png)



![](image-20250315104102514.png)
是网关地址，2 而不是255把


---


#### 2.2.5. 配置虚拟机的 Network

![](image-20250315104515867.png)

![](image-20250315121549400.png)

----


### 2.3. 设置代理

==1.查询代理服务器的 IP==
![](image-20250404204208673.png)

> [!NOTE] 注意事项
> 1. 由于连接不同的 Wi-Fi 或热点时 IP 地址会发生变化，因此我在下文建议使用临时代理。


==2.设置 图形界面应用 代理==
![](image-20250404165010104.png)


==3.设置 命令行工具 代理==
```
# 1. 编辑 ~/.bashrc（用户级别、系统级别）
vim ~/.bashrc


# 2. 添加内容
# 2.1. 以太网
export http_proxy="http://172.20.10.3:7890"
export https_proxy="http://172.20.10.3:7890"
export no_proxy="localhost,127.0.0.1,.svc,.cluster.local,192.168.136.0/24,10.96.0.1,10.244.0.0/16"
export HTTP_PROXY=$http_proxy
export HTTPS_PROXY=$https_proxy
export NO_PROXY=$no_proxy

# 2.2. WLAN
export http_proxy="http://182.32.37.244:7890"
export https_proxy="http://182.32.37.244:7890"
export no_proxy="localhost,127.0.0.1,.svc,.cluster.local,192.168.136.0/24,10.96.0.1,10.244.0.0/16"
export HTTP_PROXY=$http_proxy
export HTTPS_PROXY=$https_proxy
export NO_PROXY=$no_proxy


no_proxy 是不走代理的网络地段，就是访问这些地址的时候，不走代理，本机直连

# 3. 使命令生效（有时候需要重启 NAT Service）
source ~/.bashrc


# 4. 检测能否访问外网
curl www.google.com


# 5. 补充：临时代理（会话级别）
# 2.1. 以太网
export http_proxy="http://172.20.10.3:7890" && export https_proxy="http://172.20.10.3:7890" && export no_proxy="localhost,127.0.0.1,.svc,.cluster.local,192.168.136.0/24,10.96.0.1,10.244.0.0/16" && export HTTP_PROXY=$http_proxy && export HTTPS_PROXY=$https_proxy && export NO_PROXY=$no_proxy

# 2.2. WLAN
export http_proxy="http://182.32.37.244:7890" && export https_proxy="http://182.32.37.244:7890" && export no_proxy="localhost,127.0.0.1,.svc,.cluster.local,192.168.136.0/24,10.96.0.1,10.244.0.0/16" && export HTTP_PROXY=$http_proxy &&export HTTPS_PROXY=$https_proxy && export NO_PROXY=$no_proxy


# 6. 补充：临时禁用代理（会话级别）
unset http_proxy
unset https_proxy
unset no_proxy
```

> [!NOTE] 注意事项
> 1. `NO_PROXY` 配置指定了在请求发送到这些地址时，流量将直接访问目标服务器，不经过代理服务器。在 Kubernetes 环境中，某些特定的 IP 地址（如 Master 节点和虚拟 IP）应禁用代理。
> 2. 推荐仅为 root 用户配置代理，或者**仅使用临时代理**
> 3. 命令行工具虽然能够访问外网，但 Kubernetes 的容器运行时（如 containerd 或 Docker）并不会自动读取继承这些环境变量，导致容器无法通过代理访问外网，从而无法拉取镜像。为了解决这个问题，我们可以设置 containerd 代理
> 4. 我发现设置完临时代理后，都要再启动一下 VM NAT 服务（`services.msc`）


==4.设置 containerd 代理（K8s 需要）==
```
# 1. 创建代理配置目录
sudo mkdir -p /etc/systemd/system/containerd.service.d


# 2. 生成并配置代理文件
# 2.1. 以太网
cat <<EOF | sudo tee /etc/systemd/system/containerd.service.d/proxy.conf
[Service]
Environment="HTTP_PROXY=http://172.20.10.3:7890"
Environment="HTTPS_PROXY=http://172.20.10.3:7890"
NO_PROXY="localhost,127.0.0.1,.svc,.cluster.local,192.168.136.0/24,10.96.0.1,10.244.0.0/16"
EOF


# 2.2. WLAN
cat <<EOF | sudo tee /etc/systemd/system/containerd.service.d/proxy.conf
[Service]
Environment="HTTP_PROXY=http://182.32.38.66:7890"
Environment="HTTPS_PROXY=http://182.32.38.66:7890"
Environment="NO_PROXY=localhost,127.0.0.1,10.96.0.1,192.168.136.0/24,10.244.0.0/16,.svc,.cluster.local"
EOF


# 3. 重载配置并重启服务
sudo systemctl daemon-reload && sudo systemctl restart containerd


# 4. 验证环境变量是否注入成功
sudo systemctl show --property Environment containerd
```

> [!NOTE] 注意事项
> 1. 命令 `cat <<EOF | sudo tee /etc/systemd/system/containerd.service.d/proxy.conf` **是覆盖写入**，如果需要更换代理，直接运行命令即可。


==5.设置 Docker 代理==
```
# 1. 创建代理配置目录
sudo mkdir -p /etc/systemd/system/docker.service.d


# 2. 生成并配置代理文件
# 2.1. 以太网
cat <<EOF | sudo tee /etc/systemd/system/docker.service.d/proxy.conf
[Service]
Environment="HTTP_PROXY=http://172.20.10.3:7890"
Environment="HTTPS_PROXY=http://172.20.10.3:7890"
Environment="NO_PROXY=localhost,127.0.0.1,.svc,.cluster.local,192.168.136.0/24,10.96.0.1,10.244.0.0/16"
EOF


# 2.2. WLAN
cat <<EOF | sudo tee /etc/systemd/system/docker.service.d/proxy.conf
[Service]
Environment="HTTP_PROXY=http://113.128.71.31:7890"
Environment="HTTPS_PROXY=http://113.128.71.31:7890"
Environment="NO_PROXY=localhost,127.0.0.1,10.96.0.1,192.168.136.0/24,10.244.0.0/16,.svc,.cluster.local"
EOF


# 3. 重载配置并重启服务
sudo systemctl daemon-reload && sudo systemctl restart docker


# 4. 验证环境变量是否注入成功
sudo systemctl show --property Environment docker
```


==6.补充：环境变量相关命令==
```
# 1. 设置系统环境变量（用户级别、系统级别）
vim ~/.bashrc


# 2. 设置临时环境变量（会话级别，防止暴露）
export DOCKER_REGISTRY_SERVER="your-docker-registry-server"


# 3. 清除临时环境变量（会话级别）
unset DOCKER_REGISTRY_SERVER


# 4. 输出环境变量（变量引用加 "" 是最佳实践）
echo "DOCKER_REGISTRY_SERVER="

"$DOCKER_REGISTRY_SERVER"
```

---


### 2.4. 更新系统软件包

更新系统软件包可以确保所有软件包都是最新的，减少漏洞。
```
sudo apt update && sudo apt upgrade -y
```

---


### 2.5. 安装常用工具 / 服务

```
sudo apt install -y vim curl wget git htop net-tools unzip zip tree dos2unix openssl chrony nmap openssh-server telnet
```
- `curl/wget`：用于下载文件
- `git`：版本控制
- `htop`：进程管理
- `net-tools`：查看网络信息（如 `ifconfig`）
- `unzip/zip`：解压缩工具
- `tree`：查看目录结构
- `dos2unix`：将 shell 转为 Unix 格式
- `chrony`：时间同步
- `telnet`：测试节点间是否联通的

---


### 2.6. 安装 JDK

```
# 1. 安装 JDK 8
sudo apt install -y openjdk-8-jdk


# 2. 安装 JDK 17
sudo apt install -y openjdk-17-jdk
```

---


### 2.7. 进行时间同步

```
# 1. 启动并开机自启动 chrony
sudo systemctl enable chrony && sudo systemctl start chrony


# 2. 查看当前时间同步源（NTP 服务器）的状态
chronyc sources
```

---


### 2.8. 开启 SSH 服务

```
# 1. 启动并开机自启动 SSH
sudo systemctl start ssh
sudo systemctl enable ssh


# 2. 启用 SSH 端口（默认端口为 22）
sudo ufw allow 22/tcp


# 3. 检查 SSH 服务是否正常运行
sudo systemctl status sshd
```

---


### 2.9. 设置主机名

==1.设置主机名==
```
sudo hostnamectl set-hostname myserver
```


==2.修改本地 DNS==
修改本地 DNS 后，本机请求中的`myserver` 就会被解析为 `localhost`，指向本机地址。
```
# 1. 编辑 /etc/hosts
sudo vim /etc/hosts


# 2. 修改本地 DNS
127.0.0.1 localhost myserver
```

---


### 2.10. 防火墙配置

```
# 1. 启用防火墙
sudo ufw enable


# 2. 关闭防火墙
sudo ufw disable


# 3. 查看防火墙状态
sudo ufw status


# 4. 开放端口（推荐显式指定协议，否则同时处理 TCP 和 UDP 数据）
# 4.1. 开放 HTTP 端口（如果需要）
sudo ufw allow 80/tcp

# 4.2. 开放 HTTPS 端口（如果需要）
sudo ufw allow 443/tcp

# 4.3. 开放 SSH 端口
sudo ufw allow 22/tcp
```

---


### 2.11. 启用 root 用户

Ubuntu 默认禁用了 `root` 用户，因为 Ubuntu 不推荐直接以 `root` 账户登录，而是鼓励用户使用 `sudo` 提升权限来执行管理员任务，如果我们有特殊需求，如某些环境必须使用 root 登录，可以选择启用 root 用户
```
sudo passwd root

su root
```

> [!NOTE] 补充：
> 1. Ubuntu 出于安全性考虑，默认禁用了 `root` 用户的 SSH 远程登录，如果你确实需要 `root` 通过 SSH 登录，可以手动修改 SSH 配置
```
# 1. 编辑 /etc/ssh/sshd_config
sudo vim /etc/ssh/sshd_config


# 2. 修改 PermitRootLogin
PermitRootLogin prohibit-password   ->   PermitRootLogin yes   # 如果 PermitRootLogin 被注解，直接添加


# 3. 重启 SSH
sudo systemctl restart ssh
```

> [!NOTE] 注意事项：SFTP 子系统申请已拒绝
```
# 1. 添加以下内容
Subsystem sftp internal-sftp


# 2. 重启 ssh
sudo systemctl restart ssh
```

---


## 3. 查看 CPU 架构

在下面的命令中，`x86_64`：就是 amd64，`aarch64`：就是 arm64
```
uname -m
```

---


## 4. 重启电脑

```
sudo reboot
```

---


## 5. 创建目录、文件

```
# 1. 创建目录
mkdir -p 


# 2. 创建文件
touch


# 3. 删除目录 / 文件
rm -rf 
```

---


## 6. vim

```
# 1. 使用 vim 编辑
vim 


# 2. 保存
wq


# 3. 强制保存
:w!


# 4. 强制退出
q!
```

---



## 7. 追加

![](image-20250514173030553.png)


下面是覆盖
![](image-20250514173059391.png)



```
# 7. 安装 dos2unix 工具（如果没有就安装）
command -v dos2unix >/dev/null 2>&1 || sudo apt-get install -y dos2unix
"""
为什么在 Shell 脚本中不推荐使用这种简洁写法？因为这一行执行时，用户根本不知道你做了什么，除非主动去翻源码才能搞清楚。而采用更明确的写法则更友好：比如在执行脚本的过程中，你可以清楚看到“正在检查哪个工具”，也能知道“这个工具是已安装的”还是“刚刚被安装的”。这种信息透明的方式在编写运维脚本或部署脚本时尤其重要，因为这类脚本通常需要被他人复用。
"""

# 8. 将脚本转为 Unix 格式
dos2unix es-shell.sh
```

---


## 8. Linux 脚本（ES 脚本（注释版）.sh）

![](image-20250516154950604.png)

![](image-20250516154744807.png)

创建目录，先想想是不是需要检查并删除旧的

```
#!/bin/bash

# -------------------------------- 开启严格模式 ---------------------------------------------
set -euo pipefail

# -------------------------------- 检查是否以 root 权限运行 ---------------------------------------------
if [ "$(id -u)" != "0" ]; then
    echo "这个脚本需要以 root 用户运行" >&2
    exit 1
fi

# -------------------------------- 安装 openssl ---------------------------------------------
echo "开始安装 openssl"
echo "检查 openssl 是否已安装"
if command -v openssl >/dev/null 2>&1; then
    echo "openssl 已安装，跳过安装"
else
    echo "未检测到 openssl，正在安装..."
    max_retries=3
    retry_count=0
    until apt-get install -y openssl; do
        retry_count=$((retry_count + 1))
        if [ "$retry_count" -ge "$max_retries" ]; then
            echo "安装 openssl 已尝试 ${retry_count} 次，仍然失败" >&2
            exit 1
        fi
        echo "安装失败，第 ${retry_count} 次重试，立即重新尝试..."
    done
    echo "openssl 安装完成"
fi
"""
1. command -v openssl：
	1. command -v openssl 用于检测 openssl 命令是否存在于当前系统的环境变量 PATH 指定的路径中
	2. 如果找到了，会输出完整路径（例如 /usr/bin/openssl），并返回退出码 0（true），表示命令执行成功、任务也成功（即找到了）
	3. 如果找不到，不会输出任何内容，但命令本身仍然成功执行，只是任务失败（未找到命令），这时返回退出码为 1（false）
2. >/dev/null
	1. > 是输出重定向符，默认只作用于标准输出（stdout）。
	2. /dev/null 是 Linux 中的“黑洞”文件，任何输出重定向到这里都会被吞掉，相当于“我不想看到这个输出”。
	3. 因此，>/dev/null 表示：将命令的标准输出重定向到黑洞中，不显示在终端上。
3. 2>&1
	1. 1 表示标准输出，2 表示标准错误输出
	2. 2>&1 的意思是：“将标准错误重定向到标准输出的输出位置上”
	3. 因为前面已经执行了 >/dev/null，所以标准输出已经被扔进黑洞了，这时候标准错误也跟着一起被重定向到黑洞。
4. if...then...else...fi
5. max_retries=3
	1. 最大重试次数，总共 3 + 1 次
6. retry_count=0
	1. 已重试的次数
7. until apt-get install -y openssl; do
	1. 执行 apt-get install -y openssl; 成功就退出循环，不成功就继续循环
"""

# -------------------------------- 安装 dos2unix ---------------------------------------------
echo "开始安装 dos2unix"
echo "检查 dos2unix 是否已安装"
if command -v dos2unix >/dev/null 2>&1; then
    echo "dos2unix 已安装，跳过安装"
else
    echo "未检测到 dos2unix，正在安装..."
    max_retries=3
    retry_count=0
    until apt-get install -y dos2unix; do
        retry_count=$((retry_count + 1))
        if [ "$retry_count" -ge "$max_retries" ]; then
            echo "安装 dos2unix 已尝试 ${retry_count} 次，仍然失败" >&2
            exit 1
        fi
        echo "安装失败，第 ${retry_count} 次重试，立即重新尝试..."
    done
    echo "dos2unix 安装完成"
fi

# -------------------------------- 安装 chrony ---------------------------------------------
echo "开始安装 chrony"
echo "检查 chrony 是否已安装"
if command -v chronyd >/dev/null 2>&1; then
    echo "chrony 已安装，跳过安装"
else
    echo "未检测到 chrony，正在安装..."
    max_retries=3
    retry_count=0
    until apt-get update && apt-get install -y chrony; do
        retry_count=$((retry_count + 1))
        if [ "$retry_count" -ge "$max_retries" ]; then
            echo "安装 chrony 已尝试 ${retry_count} 次，仍然失败" >&2
            exit 1
        fi
        echo "安装失败，第 ${retry_count} 次重试，立即重新尝试..."
        sleep 2
    done
    echo "chrony 安装完成"
fi

# -------------------------------- 时间同步 ---------------------------------------------
echo "开始时间同步"
echo "正在启动 chrony 服务..."
if systemctl enable chrony && systemctl start chrony; then
    echo "chrony 服务已启动"
else
    echo "启动 chrony 失败" >&2
fi

# -------------------------------- 创建 es 用户 ---------------------------------------------
echo "开始创建 es 用户"
echo "删除已有 es 用户（包括其主目录）"
if id "es" &>/dev/null; then
    echo "检测到已有 es 用户，正在删除..."
    userdel -r es
    echo "已删除旧的 es 用户"
fi
echo "新增 es 用户"
useradd -m -s /bin/bash es
echo "为 es 用户设置密码（wq666）"
passwd es
echo "创建 es 用户完成"
"""
1. id "es"
	1. 用于检查 es 用户是否存在
	2. 如果存在，会输出例如：uid=1001(es) gid=1001(es) groups=1001(es)
2. userdel -r es
	1. 删除用户和其主目录
"""

# -------------------------------- 关闭 Swap 分区 ---------------------------------------------
echo "开始关闭 Swap 分区"
echo "将 内容注释"
sed -i '/^[^#].*\bswap\b/ s/^/#/' /etc/fstab
echo "立即关闭 Swap 分区"
swapoff -a
echo "关闭 Swap 分区完成"

# -------------------------------- 开放 9200、9300 TCP 端口 ---------------------------------------------
echo "开始开放 9200、9300 TCP 端口"
ufw allow 9200/tcp && ufw allow 9300/tcp
echo "开放 9200、9300 TCP 端口完成"

# -------------------------------- 设置主机名、主机名互相解析 ---------------------------------------------
echo "开始设置主机名、主机名互相解析"
echo "获取本机内网 IP 地址"
INTERFACE=$(ip link | grep -oP '^[0-9]+: \K[^:]+' | grep -v lo | head -1)
LOCAL_IP=$(ip addr show "$INTERFACE" | grep -oP 'inet \K[\d.]+' | head -1)
if [ -z "$LOCAL_IP" ]; then
    echo "不能获取本机 IP 地址" >&2
    exit 1
fi
echo "本机内网 IP 地址获取成功：$LOCAL_IP"
echo "设置本机主机名"
case "$LOCAL_IP" in
    "192.168.136.8")
        HOSTNAME="es-node1"
        ;;
    "192.168.136.9")
        HOSTNAME="es-node2"
        ;;
    "192.168.136.10")
        HOSTNAME="es-node3"
        ;;
    *)
        echo "IP $LOCAL_IP does not match any configured hostname" >&2
        exit 1
        ;;
esac
hostnamectl set-hostname "$HOSTNAME"
echo "设置主机名互相解析"
HOSTS_CONTENT="
192.168.136.8   es-node1
192.168.136.9   es-node2
192.168.136.10  es-node3"
echo "检查 /etc/hosts 是否已包含指定追加内容"
if grep -Fx "$HOSTS_CONTENT" /etc/hosts > /dev/null; then
    echo "主机名解析记录已存在，跳过添加"
else
    echo "$HOSTS_CONTENT" >> /etc/hosts
    echo "主机名解析记录已添加"
fi
echo "设置主机名、主机名互相解析完成"

# -------------------------------- 安装 ES ---------------------------------------------
echo "开始安装 ES"
echo "进入 /mystudy/es 目录"
cd /mystudy/es
echo "删除已存在 ES 目录"
if [ -d "/mystudy/es/elasticsearch" ]; then
    echo "检测到已存在 ES 目录，准备删除..."
    rm -rf /mystudy/es/elasticsearch
    echo "ES 目录已删除"
fi
echo "检查是否存在 ES 安装包"
if [ ! -f "elasticsearch-8.18.0-linux-x86_64.tar.gz" ]; then
    echo "ES 安装包不存在" >&2
    exit 1
fi
echo "解压"
tar -zxvf elasticsearch-8.18.0-linux-x86_64.tar.gz -C /mystudy/es
echo "重命名"
mv elasticsearch-8.18.0 elasticsearch
echo "安装 ES 完成"
"""
1. [ -d "path" ]:
	1. path 是一个存在的目录吗？
	2. 类似的有：
		1. -f "path"：
			1. path 是一个存在的文件吗？
		2. -e "path"：
			1. path 是一个存在的路径吗？（文件或目录）
"""

# -------------------------------- 创建存放 ES 数据的目录 ---------------------------------------------
echo "开始创建存放 ES 数据的目录"
mkdir -p /mystudy/es/elasticsearch/data
echo "创建存放 ES 数据的目录完成"

# -------------------------------- 创建存放证书的目录 ---------------------------------------------
echo "开始创建存放证书的目录"
mkdir -p /mystudy/es/elasticsearch/config/certs
echo "创建存放证书的目录完成"

# -------------------------------- 生成 CA 证书、CA 公钥（192.168.136.8） ---------------------------------------------
if [ "$LOCAL_IP" = "192.168.136.8" ]; then
    echo "开始在 192.168.136.8 上生成 CA 证书、CA 公钥"
    echo "进入 ES 目录"
    cd /mystudy/es/elasticsearch
    echo "签发 ca 证书"
    bin/elasticsearch-certutil ca
    echo "导出 CA 公钥"
    openssl pkcs12 -in elastic-stack-ca.p12 -nokeys -out ca.crt || { echo "导出 CA 公钥失败" >&2; exit 1; }
    echo "将 CA 证书、CA 公钥放到存放证书目录下"
    mv elastic-stack-ca.p12 ca.crt /mystudy/es/elasticsearch/config/certs/ || { echo "移动证书文件失败" >&2; exit 1; }
    echo "生成 CA 证书、CA 公钥完成"
fi
# -------------------------------- 签发节点证书（192.168.136.8） ---------------------------------------------
if [ "$LOCAL_IP" = "192.168.136.8" ]; then
    echo "开始在 192.168.136.8 上签发节点证书"
    echo "进入 ES 目录"
    cd /mystudy/es/elasticsearch
    echo "批量签发节点证书"
    echo "创建并编辑 instances.yml"
    cat > instances.yml <<EOF
instances:
  - name: es-node1
    ip:
      - "192.168.136.8"
  - name: es-node2
    ip:
      - "192.168.136.9"
  - name: es-node3
    ip:
      - "192.168.136.10"
EOF
    echo "进行批量签发"
    bin/elasticsearch-certutil cert \
        --silent \
        --in instances.yml \
        --ca config/certs/elastic-stack-ca.p12 \
        --pem \
        --out certs.zip
    echo "将 certs.zip 放到存放证书目录下"
    mv certs.zip config/certs/

    echo "签发节点证书完成"
fi
# -------------------------------- 签发 HTTPS 证书（192.168.136.8） ---------------------------------------------
if [ "$LOCAL_IP" = "192.168.136.8" ]; then
    echo "开始在192.168.136.8 上签发 HTTPS 证书"
    echo "进入 ES 目录"
    cd /mystudy/es/elasticsearch
    echo "批量签发 HTTPS 证书"
    echo "创建并编辑 http-instances.yml"
    cat > http-instances.yml << EOF
instances:
  - name: es-node1-http
    ip:
      - "192.168.136.8"
  - name: es-node2-http
    ip:
      - "192.168.136.9"
  - name: es-node3-http
    ip:
      - "192.168.136.10"
EOF
    echo "进行批量签发"
    bin/elasticsearch-certutil cert \
      --silent \
      --in http-instances.yml \
      --ca  config/certs/elastic-stack-ca.p12 \
      --pem \
      --out http-certs.zip || { echo "生成 HTTPS 证书失败" >&2; exit 1; }

    mv http-certs.zip config/certs/ || { echo "移动 http-certs.zip 失败" >&2; exit 1; }
    echo "签发 HTTPS 证书完成"
fi

# -------------------------------- 分发证书（192.168.136.8） ---------------------------------------------
if [ "$LOCAL_IP" = "192.168.136.8" ]; then
    echo "开始在 192.168.136.8 上分发证书"
    cd /mystudy/es/elasticsearch/config/certs
    echo "分发节点证书"
    unzip certs.zip 
    mv /mystudy/es/elasticsearch/config/certs/es-node1/es-node1.{crt,key} /mystudy/es/elasticsearch/config/certs/
    scp /mystudy/es/elasticsearch/config/certs/es-node2/es-node2.{crt,key} \
        root@192.168.136.9:/mystudy/es/elasticsearch/config/certs/
    scp /mystudy/es/elasticsearch/config/certs/es-node3/es-node3.{crt,key} \
        root@192.168.136.10:/mystudy/es/elasticsearch/config/certs/
    echo "分发HTTPS 证书"
    unzip http-certs.zip
    mv /mystudy/es/elasticsearch/config/certs/es-node1-http/es-node1-http.{crt,key} /mystudy/es/elasticsearch/config/certs/
    scp /mystudy/es/elasticsearch/config/certs/es-node2-http/es-node2-http.{crt,key} \
        root@192.168.136.9:/mystudy/es/elasticsearch/config/certs/
    scp /mystudy/es/elasticsearch/config/certs/es-node3-http/es-node3-http.{crt,key} \
        root@192.168.136.10:/mystudy/es/elasticsearch/config/certs/
    echo "分发 CA 公钥"
    scp /mystudy/es/elasticsearch/config/certs/ca.crt \
        root@192.168.136.9:/mystudy/es/elasticsearch/config/certs/
    scp /mystudy/es/elasticsearch/config/certs/ca.crt \
        root@192.168.136.9:/mystudy/es/elasticsearch/config/certs/
    rm -rf es-node1 es-node1-http es-node2 es-node2-http es-node3 es-node3-http certs.zip http-certs.zip
    echo "分发证书完成"
fi

# -------------------------------- 修改 ES 文件拥有者为 es ---------------------------------------------
echo "开始修改 ES 文件拥有者为 es"
chown -R es:es /mystudy/es/elasticsearch
echo "修改 ES 文件拥有者为 es 完成"
echo "本节点脚本执行结束"
```

---


## 查看 Ubuntu 版本

```
lsb_release -a
"""
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 24.04.2 LTS
Release:        24.04
Codename:       noble
"""
```

---




