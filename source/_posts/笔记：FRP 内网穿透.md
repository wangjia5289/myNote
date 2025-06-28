---
title: 笔记：FRP 内网穿透
date: 2025-06-24
categories:
  - 网络
  - 内网穿透
  - FRP 内网穿透
tags: 
author: 霸天
layout: post
---
![](image-20250624100920194.png)

![](image-20250624100936818.png)






## FRPS 端（云服务器端）

### 创建 mystudy/frp 目录

```
mkdir -p /mystudy/frp
```

---

### 下载并安装 FRP Linux

<span style="background:#fff88f">1. 下载 FRP 服务端</span>
从 [FRP 服务端下载地址](https://github.com/fatedier/frp/releases)下载 FRP 服务端安装包：
![](image-20250624101633939.png)

> [!NOTE] 注意事项
> 1. 下载前请确认服务器架构，x86_64 需下载 linux_amd64，aarch64 需下载 linux_arm64
```
uname -m
```


<span style="background:#fff88f">2. 安装 FRP Linux</span>
```
# 1. 将 FRP 服务端安装包上传至 /mystudy/frp 目录


# 2. 进入 /mystudy/fro 目录
cd /mystudy/frp


# 3. 删除已存在 FRP 目录
rm -rf /mystudy/frp/frp


# 4. 解压
tar -zxvf  frp_0.62.1_linux_amd64.tar.gz -C /mystudy/frp


# 5. 重命名
mv frp_0.62.1_linux_amd64 frp
```

----


### 启动 FRPS 端

#### 关闭防火墙 / 开启 7000 端口

##### 关闭本地防火墙

```
sudo ufw disable
```

---


##### 关闭云服务商防火墙
![](image-20250624104543612.png)

---


##### 测试能否联通
```
# 确认 frps 真正启动并监听在 0.0.0.0:7000，有返回结果就行
sudo netstat -tulpn | grep :7000


# 1. 本机连通性
telnet 127.0.0.1 7000

# 2. 公网是否可达
telnet 117.72.211.221 7000
```

---

#### 配置 FRPS 配置文件：frps.toml

```
# 1. 修改 FRPS 配置文件
vim /mystudy/frp/frp/frps.toml
"""
# frps.toml (v0.62.1+ 推荐最简配置)

# 监听客户端连接的地址与端口
bindAddr = "0.0.0.0"
bindPort = 7000

# 如果要开启 Web Dashboard，则如下配置
# Dashboard 会在 127.0.0.1:7500 上提供状态页面
webServer.addr = "127.0.0.0"
webServer.port = 7500
webServer.user = "admin"
webServer.password = "admin123"

# 日志输出到文件，并设定日志级别
log.to = "./frps.log"
log.level = "info"

"""
```

----


#### 启动 FRPS 端

```
# 1. 进入 FRP 目录
cd /mystudy/frp/frp


# 2. 启动 FRPS 端
./frps -c frps.toml
```

---

#### 设置 FRPS 开机自启动

```
sudo tee /etc/systemd/system/frps.service > /dev/null << 'EOF'
[Unit]
Description=FRP Server Service
After=network.target

[Service]
Type=simple
User=root
# 请改成你的 frps 可执行文件所在绝对路径
ExecStart=/mystudy/frp/frp/frps -c /mystudy/frp/frp/frps.toml
Restart=on-failure
RestartSec=5s

# 日志输出到 journal
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF
```

```
# 让 systemd 识别新服务
sudo systemctl daemon-reload

# 开机自动启动
sudo systemctl enable frps.service

# 立刻启动
sudo systemctl start frps.service

# 查看服务是否正常运行
sudo systemctl status frps.service

```



## FRPC 端（本地电脑端）

### 下载并安装 FRP Windows

<span style="background:#fff88f">1. 下载 FRP Windows</span>
从 [FRP 服务端下载地址](https://github.com/fatedier/frp/releases)下载 FRP 服务端安装包：
![](image-20250624102659401.png)

> [!NOTE] 注意事项
> 1. 下载前请确认服务器架构，AMD64 需下载 windows_amd64，ARM64 需下载 windows_arm64
```
echo $env:PROCESSOR_ARCHITECTURE
```


<span style="background:#fff88f">2. 解压 FRP Windows</span>

----


### 启动 FRPC 端

#### 配置 FRPC 配置文件：frpc.toml

```
# 1. 修改 FRPC 配置文件
编辑 frpc.toml
"""
[common]
server_addr = "117.72.211.221"
server_port = 7000
[socks5]
type       = "tcp"
local_ip   = "127.0.0.1"
local_port = 7890
remote_port= 7890
"""
```
你在 `local_port` 和 `remote_port` 的 **字段后面直接加了注释**，这虽然在某些宽容的 TOML 解析器中可以工作，但 `frp v0.62.1+` 使用的是严格模式，**不允许字段后紧跟中文注释或注释前无空格**。
![](image-20250624110113741.png)

----


#### 启动 FRPC 端
进入命令行（CMD）运行：
```
.\frpc.exe -c frpc.toml
```


---

## 云服务器

访问
```
sudo netstat -tulpn | grep :7890


127.0.0.1:7890

export http_proxy="http://127.0.0.1:7890"
export https_proxy="http://127.0.0.1:7890"
export no_proxy="localhost,127.0.0.1,.svc,.cluster.local,192.168.136.0/24,10.96.0.1,10.244.0.0/16"
export HTTP_PROXY=$http_proxy
export HTTPS_PROXY=$https_proxy
export NO_PROXY=$no_proxy

```