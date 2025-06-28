---
title: 笔记：Docker
date: 2025-03-12
categories:
  - 容器化
  - Docker
  - Docker 基础
tags: 
author: 霸天
layout: post
---
### 1. 导图：[Map：Docker](Map：Docker.xmind)

---


### 2. Docker 概述

Docker 是一种**容器化技术**，能够**跨平台**的**加速**项目的**构建**、**分享**、**运行**。

Docker 致力于以下三个方面：
1. **解决“运行环境不一致”问题**
	- 应用开发中常见一句话：**“在我电脑上没问题！”**。这是因为不同环境下（操作系统、库版本）可能导致程序无法正常运行。而 Docker 的容器包含了应用运行的全部环境，保证在开发、测试、生产等环境中都一致。
2. **轻量、快速启动**
	- Docker 容器很轻，占用资源少，启动速度快。和传统的虚拟机相比，它更高效。
3. **方便部署和迁移**
	- 通过 Docker，你可以把应用快速部署到任何支持 Docker 的服务器或云平台上，而无需关心底层环境的差异。

![](source/_posts/笔记：Docker%20基础/image-20250313103856946.png)

---


### 3. Docker 核心组件图

![](source/_posts/笔记：Docker%20基础/image-20250313090125273.png)

---


### 4. Docker Engine（引擎）

#### 4.1. Docker Engine 概述

Docker 引擎是运行和管理 Docker 容器的核心程序，它包括 Docker Daemon（服务端，一个守护进程，负责构建、运行容器），和 Docker Client（客户端，提供与用户交互的命令行工具）

我们平常所说的安装 Docker 本质上就是在安装的 Docker Engine。

---


#### 4.2. 安装 Docker（Docker Engine）

安装 Docker 的方法有很多，详细安装方法可以查看 [Docker Engine 安装文档](https://docs.docker.com/engine/install/) ，这里选择在 Ubuntu 中进行安装
```
# 1. 移除旧版本 Docker
sudo apt remove docker docker-engine docker.io containerd runc

sudo apt autoremove


# 2. 安装必要的依赖包
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common


# 3. 添加 Docker 的官方 GPG 密钥（使用国内CDN）
# 3.1. 创建存放 GPU 秘钥的目录
sudo mkdir -p /etc/apt/keyrings

# 3.2. 获取 GPU 秘钥
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg


# 4. 添加 Docker 的 APT 仓库（中科大源）
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null


# 5. 更新 APT 包索引，让修改生效
sudo apt update


# 6. 安装 Docker Engine
sudo apt install -y docker-ce docker-ce-cli containerd.io


# 7. 检查 Docker
docker --version
或
docker info


# 7. 启动并设置开机自启动
sudo systemctl start docker && sudo systemctl enable docker
```

> [!NOTE] 注意事项
> 1. 如果启动 docker 时出现错误，我们可以：
```
# 1. 找出是哪个依赖挂了
systemctl list-dependencies docker.service
"""
root@user-virtual-machine:~# systemctl list-dependencies docker.service
docker.service
● ├─containerd.service
× ├─docker.socket              # 如果是这里出现错误，可能是在安装脚本时没有创建 docker 用户组
● ├─system.slice
● ├─network-online.target
● │ └─NetworkManager-wait-online.service
● └─sysinit.target
●   ├─apparmor.service
●   ├─dev-hugepages.mount
●   ├─dev-mqueue.mount
●   ├─keyboard-setup.service
●   ├─kmod-static-nodes.service
●   ├─plymouth-read-write.service
●   ├─plymouth-start.service
●   ├─proc-sys-fs-binfmt_misc.automount
●   ├─setvtrgb.service
●   ├─sys-fs-fuse-connections.mount
●   ├─sys-kernel-config.mount
●   ├─sys-kernel-debug.mount
●   ├─sys-kernel-tracing.mount
○   ├─systemd-ask-password-console.path
●   ├─systemd-binfmt.service
○   ├─systemd-boot-system-token.service
●   ├─systemd-journal-flush.service
●   ├─systemd-journald.service
○   ├─systemd-machine-id-commit.service
●   ├─systemd-modules-load.service
○   ├─systemd-pstore.service
●   ├─systemd-random-seed.service
●   ├─systemd-sysctl.service
●   ├─systemd-sysusers.service
"""


# 2. 创建 docker 用户组
sudo groupadd docker


# 3. 启动 docker.socket
sudo systemctl start docker.socket


# 4. 重启和开机自启动 docker
sudo systemctl start docker && sudo systemctl enable docker
```


---


#### 4.3. 配置 Docker（Docker Engine）

我们可以**为 Docker 配置代理**，或者在Docker 1.10.0之后的版本，用以下方式为 Docker 配置国内镜像加速器，以提高 Docker 镜像的拉取速度。
```
# 1. 创建 Docker 配置目录（如果不存在）
sudo mkdir -p /etc/docker


# 2. 配置 Docker 镜像加速器
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://dh-mirror.gitverse.ru"]
}
EOF


# 3. 重新加载 Docker 配置并重启服务
sudo systemctl daemon-reload && sudo systemctl restart docker


# 4. 检查是否配置成功
sudo docker info
```

> [!NOTE] 注意事项
> 1. <font color="#00b0f0">加速器的原理</font>：
> 	- 当你从 Docker Hub 拉取镜像时，加速器会自动将你请求的镜像（如果它之前没有缓存）从 Docker Hub 拉取到加速器服务器，并且在加速器缓存中存储该镜像。这样，其他使用该加速器的用户会更快地下载到相同的镜像。
> 1. <font color="#00b0f0">常用 Docker 加速器</font>：
> 	- GitVerse：
> 		- 加速器地址：https://dh-mirror.gitverse.ru    （慢）
> 		- 配置文档：[GiteVerse 加速器配置文档](https://gitverse.ru/docs/artifactory/registry-mirrors/dh-mirror/)
> 	- DaoCloud：
> 		- 加速器地址：https://docker.m.daocloud.io    （快）
> 	- 社区提供：
> 		-  https://hub.rat.dev/    （慢）
> 		-  https://docker.1panel.live/    （慢）
> 1. <font color="#00b0f0">加速器变动问题</font>：
> 	- 您可能会发现，许多加速器已发生变动，无法继续使用（如阿里云加速器、腾讯云加速器）。这通常是由于运营商网络等不稳定因素导致镜像加速器无法成功拉取指定版本的容器镜像。因此，建议在生产环境中谨慎依赖 Docker Hub 提供的容器镜像。
> 	- 为了确保稳定性和安全性，阿里云推荐使用其[制品中心](https://help.aliyun.com/zh/acr/user-guide/artifact-center)提供的官方支持的容器基础镜像。
> 1. <font color="#00b0f0">实际开发中发现的问题</font>：
> 	- 配置加速器后，虽然能加速 `docker pull` 命令，但是仍旧无法正常执行 `docker search` 命令

---


#### 4.4. Docker Engine 相关命令

```
# 1. Docker 的版本
docker --version


# 2. Docker 的详细信息（包括容器数、镜像数、运行状态等）
docker info
```

---


### 5. Docker Image（镜像）

#### 5.1. Docker Image 概述

镜像是一个只读的“程序模板”，用于创建 Docker 容器。它包含了应用运行所需的所有内容，包括操作系统环境、应用程序、库文件等，由镜像创建容器，就像是根据模板制造实际的产品（一个镜像可以制作很多很多的容器）

---


#### 5.2. Docker Image 命令

==1.查看镜像==
```
# 1. 查看 Docker 主机存在的所有镜像
docker images


# 2. 查看指定镜像
docker images zookeeper


# 3. 仅显示镜像 ID（常用于一键删除所有镜像）
docker images -q
```
![](source/_posts/笔记：Docker%20基础/image-20250313102408127.png)
1. ==REPOSITORY==：
	1. 镜像的名称
2. ==TAG==：
	1. 镜像的标签，一般指版本号，如 `v1.0`、`1.0`、`latest` 等等
3. ==IMAGE ID==：
	1. 镜像的唯一 ID（截断版，前12位，后续引用时可简写，只要能唯一区分）
4. ==CREATED==：
	1. 镜像的创建时间
5. ==SIZE==：
	1. 镜像的大小


==2.搜索镜像==
```
# 1. 命令行搜索（来源于DockerHub）
docker search nginx


# 2. 直接上 DockerHub 官网搜索


# 3. 直接上其他镜像仓库搜索（例如阿里云）
```
![](source/_posts/笔记：Docker%20基础/image-20250313102619922.png)

> [!NOTE] 注意事项：上官网搜索的好处
> 1. 当我们可以查看某镜像特定版本还可以查阅相关文档


==3.拉取镜像==
```
# 1. 从 DockerHub 中拉取镜像（未指定 image-tag，默认拉取 latest）
docker pull <image-name>:<image-tag>


# 2. 从其他镜像仓库拉取镜像（以阿里云为例）
docker pull crpi-eq36m90bmg934mtm.cn-beijing.personal.cr.aliyuncs.com/wangjia5289/first_test:[镜像版本号]
```

> [!NOTE] 注意事项
> 1. 下载公共镜像无需登录账户（`docker login`），但若下载私有镜像，则需先使用个人账号登录，无论是 DockerHub 还是阿里云仓库，均遵循此规则


==4.加载镜像==
将一个通过 `docker save` 命令导出的 Docker 镜像 tar 包加载到本地 Docker 环境中
```
# 加载镜像 tar 包
docker load -i mynginx-1.0.tar
```


==5.推送镜像==
见下文：Docker Registry（仓库）


==6.删除镜像==
```
# 1. 删除指定镜像（不能删除正在运行容器的镜像）
docker rmi <image-name>:<image-tag> / <image-id>


# 2. 强制删除镜像
docker rmi -f <image-name>:<image-tag> / <image-id>


# 3. 删除所有镜像
docker rmi -f ${docker images -q}


# 4. 删除所有未运行容器的镜像
docker image prune
```


==7.保存镜像==
```
# 1. 将镜像推送到镜像仓库
见下文：Docker Registry（仓库）


# 2. 将镜像导出为 tar 包（需要时使用 docker load 加载）
docker save -o /home/user/backups/mynginx-1.0.tar mynginx:v1.0
```
1. ==-o /home/user/backups/mynginx-1.0.tar==：
	1. 指定导出文件的路径和文件名
	2. 如仅填写 `mynginx-1.0.tar`，表示将文件保存在当前目录下
2. ==mynginx:v1.0==：
	1. 镜像的版本标签（如省略，默认保存带有 `latest` 标签的镜像）。


==8.构建镜像==
见下文：Docker Image（镜像） / Docker Image 构建方法

---


#### 5.3. Docker Image 构建方法

##### 5.3.1. 从现有容器构建 Docker 镜像
```
# 1. 将一个运行中的容器保存为一个新的镜像，从而捕获容器内的所有更改。
docker commit -a "LeiFeng" -m "update index.html" mynginx_Container mynginx:v1.0
```
1. ==-a "LeiFeng"==：
	1. 指定作者信息
2. ==-m "update index.html"==：
	1. 提交说明，类似我们提交 Git 时的说明 "first commit"
3. ==mynginx_Container==:
	1. 要提交的容器，可以使用`<container-name> / <container-id>`
4. ==mynginx:v1.0==：
	1. <font color="#00b0f0">mynginx</font>：
		1. 新镜像的名称
	2. <font color="#00b0f0">v1.0</font>：
		1. 镜像的版本标签（如果省略，Docker 默认会使用 `latest` 标签）。

> [!NOTE] 注意事项
> 1. 提交时会暂停容器运行

---


##### 5.3.2. 通过 Docker File 构建 Docker 镜像

###### 5.3.2.1. Docker File 概述

`Dockerfile` 是一个用于定义 Docker 容器镜像的文本文件，其中包含一组指令，指示 Docker 如何根据 `Dockerfile` 中的步骤构建一个 Docker 镜像

`Dockerfile` 就是一个普通的文本文件，保存在哪里都可以，只要在构建镜像时能访问到它就行

---


###### 5.3.2.2. Docker File 基础指令

```
# FROM：基于哪个镜像构建新镜像，一个阶段只能有一个 FROM（AS builder 表示此阶段命名为 builder，后续阶段可以使用 --from=builder 引用该阶段容器）
FROM ubuntu:20.04 AS builder


# LABEL：元数据，如作者、版本等信息。
LABEL maintainer="yourname@example.com"                             # 维护者
LABEL version="1.0"


# SHELL：指定当前阶段后续命令的 shell。默认使用轻量级的 /bin/sh -c，但在需要更强大功能（如复杂脚本、命令扩展、管道、数组操作等）时，可以切换为 /bin/bash -c
SHELL ["/bin/bash", "-c"] 


# WORKDID：设置容器中的工作目录。后续的 RUN, Copy, CMD, ENTRYPOINT 等指令都会在这个目录下执行（若不存在此目录，将自动创建）
WORKDIR /app 


# RUN：执行 Shell 命令，通常用来在容器内安装软件包、配置文件、下载依赖等（容器里安装的软件和宿主机互不影响）
RUN apt-get update && apt-get install -y curl vim 


# COPY：将宿主机中的文件或目录复制到镜像中的指定位置。
COPY . /app                              # 将 DocekerFile 所在目录的所有文件复制到镜像中的 /app 目录


# ADD：类似于 COPY，但是 ADD 除了可以复制文件，还能解压 tar 文件、下载网络资源（URL）。
ADD myapp.tar.gz /app                    # 将本地 tar 文件解压到镜像的 /app 目录


# VOLUME：创建挂载点，它会在容器运行时创建一个目录（通常是空目录），并且可以通过 Docker 挂载到主机上。
VOLUME ["/data"]                        # 创建一个挂载点 /data


# ENTRYPOINT：指定容器启动后的执行命令，不可被覆盖，通常与 CMD 一起使用
# CMD：指定容器启动后的执行命令，可以在容器运行时被覆盖（CMD 可以有多个，但只会执行最后一个 CMD）
ENTRYPOINT ["python3"] 
CMD ["app.py"]
```

> [!NOTE] 注意事项
> 1. 通常将 `ENTRYPOINT` 和 `CMD` 结合使用，虽然可以直接在 `ENTRYPOINT` 中固定执行逻辑，比如 `ENTRYPOINT ["python3", "app.py"]`，但这种方式缺乏灵活性，无法轻松更换运行的文件。
> 2. 如果采用 `ENTRYPOINT + CMD` 的方式，运行容器时可以轻松覆盖默认执行文件。例如：`docker run my-app other-app.py`，这里用 `other-app.py` 替代了默认的 `app.py`。
> 3. 为什么能替换执行文件：这其实是 Docker 的一个非常灵活的特性。我们默认的执行文件确实是 `app.py`，但是如果 `other-app.jar` 必须在镜像内已经存在，或者通过挂载（`-v`）从宿主机提供。如果我们使用 `run my-app other-app.py` ，Docker 容器会在运行时接收参数，这些参数会覆盖 `CMD` 的默认值。

---


###### 5.3.2.3. 基于 Spring Boot 项目的构建示例

==1.补充：多阶段构建==
在 Dockerfile 中，我们可以定义多个构建阶段（`FROM` 指令可以出现多次），每个阶段拥有独立的环境和依赖。最终，只将构建产物（如 JAR 包等可执行文件）拷贝到最终的运行镜像中。

以 Spring Boot 项目为例，通常使用 Maven 进行构建和依赖管理。构建阶段需要依赖大量的工具和库（如 Maven、JDK 和构建工具等），但这些内容在生产环境中是不必要的。

通过多阶段构建，在构建阶段配置合适的环境和依赖以生成可执行 JAR 包，而在运行阶段，只将构建生成的 JAR 包拷贝到镜像中。此阶段的镜像仅包含运行时环境（如 JRE），不包含任何构建工具，进一步精简镜像内容，从而有效减小镜像体积。

==2.创建 Docker File==
```
# 1. 创建存放 Docker File 的目录
mkdir -p /etc/temp


# 2. 创建一个空的 Docker File 文件
touch /etc/temp/Dockerfile


# 3. 进入 /etc/temp 目录
cd /etc/temp
```


==2.编辑 Docker File==
```
# builder 阶段

# maven:3.8.7-eclipse-temurin-17 它包含了 Maven 和 JDK 17 的环境
FROM maven:3.8.7-eclipse-temurin-17 AS builder


# 设置工作目录
WORKDIR /app


# 将 DockerFile 所在目录中项目的 pom.xml 文件复制到容器内的 /app 目录。
COPY pom.xml /app                # 或者直接写 .


# 下载 pom.xml 中所指定的项目的所有依赖（包括插件等），-B 参数是指批处理模式（非交互式）
RUN mvn dependency:go-offline -B


# 将 DockerFile 所在目录中项目的源代码 src 文件复制到容器内的 /app/src 目录。
COPY src /app/src                # 或者直接写 ./src


# 运行 mvn package 命令，构建项目并生成 JAR 包。-DskipTests 表示跳过单元测试的执行
RUN mvn package -DskipTests





# 生产阶段

# eclipse-temurin:17-jre-alpine，是一个轻量级的 JRE 镜像，适合运行 Java 应用程序。
FROM eclipse-temurin:17-jre-alpine


# 设置元数据
LABEL maintainer="dev@example.com"
LABEL version="1.0"


# 创建 springboot 用户（非 root 用户，Docker 安全相关知识）
RUN addgroup -S springboot && adduser -S springboot -G springboot


# 设置工作目录
WORKDIR /app


# 从 builder 阶段复制 JAR 文件（注意 JAR 名称需与 pom.xml 中配置一致）
COPY --from=builder /app/target/*.jar app.jar


# 更改 /app 目录及其中所有文件的拥有者为 springboot 用户和组
RUN chown -R springboot:springboot /app


# 切换到 springboot 用户运行后续命令，避免以 root 用户执行。
USER springboot


# 声明容器将监听的端口。Spring Boot 默认在 8080 端口提供服务，因此这里暴露了 8080 端口。
EXPOSE 8080

# 健康检查（Docker 安全相关知识）
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget -q -O /dev/null http://localhost:8080/actuator/health || exit 1


# 启动命令，-Dspring.profiles.active=prod 是 Boot 运行的相关参数，指定是在生产环境
ENTRYPOINT ["java", "-jar", "-Dspring.profiles.active=prod", "app.jar"]
```

> [!NOTE] 注意事项：为什么不直接在项目中打包 JAR 文件，而要在 Docker 中构建？
> 1. 确实可以在本地环境中直接进行构建和打包，但在 **Docker** 中构建项目有其独特的优势。
> 2. Docker 提供了一个一致的运行环境，通过在 Docker 中构建应用，可以确保构建环境与生产环境完全一致，避免由于不同开发机器上的环境差异（如 JDK 版本、依赖版本等）带来的问题，不会受到宿主机上已安装工具或库的影响。

> [!NOTE] 注意事项：`RUN mvn package -DskipTests` 命令
> 1. 默认情况下，当你执行 `mvn package` 时，Maven 只会将项目编译后的 `.class` 文件和 `resources` 文件打包进生成的 JAR 文件。外部依赖（如 Spring Boot、第三方库等）不会自动被打包进 JAR 文件，所以，打出来的 JAR 文件无法独立运行，除非目标机器也具备所需依赖。
> 2. 如果希望在没有外部依赖环境的情况下运行应用，就需要将所有外部依赖一起打包进 JAR 文件中，这样生成的 JAR 就可以独立运行，无需额外配置依赖。这时，我们需要在 `pom.xml` 中配置相应的插件来进行这一实现：
> 	- `spring-boot-maven-plugin`（适用于 Spring Boot 项目）
> 	- `maven-shade-plugin`（适用于任何 Java 项目）


==3.将项目代码复制到 /etc/temp==
大致结构可能为：
```
/etc/temp
|
|-- Dockerfile
|
|-- src /
|
|-- pom.xml
```


==5.使用 Docker File 构建镜像==
```
# 1. 模版
docker build \
  --build-arg http_proxy=http://<代理IP>:<端口> \
  --build-arg https_proxy=http://<代理IP>:<端口> \
  -t <your-image-name>:<your-image-tag> .
  

# 2. 以太网
docker build \
  --build-arg http_proxy=http://172.20.10.3:7890 \
  --build-arg https_proxy=http://172.20.10.3:7890 \
  -t <your-image-name>:<your-image-tag> .


# 3. 局域网
docker build \
  --build-arg http_proxy=http://182.32.38.66:7890 \
  --build-arg https_proxy=http://182.32.38.66:7890 \
  -t <your-image-name>:<your-image-tag> .
```
1. ==-t your-image-name:your-image-tag==
	1. 你的镜像名称和标签，例如： my-spring-app:v1.0
2. ==(.)==
	1. Dockerfile 的位置，这里表示当前目录，即 Dockerfile 在当前目录

---


##### 5.3.3. 第三方自动构建 Docker 镜像

以下是基于 阿里云容器镜像服务 +  GitHub 自动构建 Docker 镜像的全过程，能够当代码提交到 GitHub 仓库后，自动构建镜像：

==1.创建 GitHub 项目==
创建一个空的 GitHub 仓库


==2.推送代码==
然后将我们的代码推送到该仓库中。
```
# 1. 初始化本地 Git 仓库
git init


# 2. 创建 README 文档
echo "# automotic" > README.md


# 3. 将 README 稳定添加到缓冲取
git add README.md


# 4. 进行第一次推送（务必进行，相当于点火器）
git commit -m "first commit"


# 5. 将当前分支强制重命名为 main
git branch -M main


# 6. 本地仓库与远程仓库进行关联。
git remote add origin https://github.com/wangjia5289/xxxxxx.git


# 7. 进行分支关联（使用 -u 选项，将本地 main 分支与远程 main 分支关联，之后可以直接使用 git push 和 git pull，而无需每次指定远程仓库和分支。）
git push -u origin main


# 8. 后续推送
我们可以通过命令行或使用 GitHub Desktop 等工具来推送代码。
```


==3.创建并编辑 Docker File==
在 GitHub 项目根目录下创建 `Dockerfile` 文件，内容根据需求自定义


==4.阿里云创建镜像仓库==
在创建镜像仓库时，将其与我们之前创建的 GitHub 仓库进行关联。
![](source/_posts/笔记：Docker%20基础/image-20250406135036107.png)


==5.添加构建规则==
![](source/_posts/笔记：Docker%20基础/image-20250406140638445.png)

==6.编辑构建规则==
![](source/_posts/笔记：Docker%20基础/image-20250406140437388.png)


### 6. Docker Container（容器）

#### 6.1. Docker Container 概述

Docker Container 就是从镜像启动的“运行实例”，它是一个独立的小盒子，应用就在里面跑，可以理解为一个大 Linux 系统中的小 Linux 系统。容器可以随时启动、停止、销 毁，互不干扰。

<font color="#ff0000">Docker 官方推荐一个容器运行一个程序</font>，这样可以保持容器的简单性和高效性
![](source/_posts/笔记：Docker%20基础/image-20250313103856946.png)

---


#### 6.2. Docker Container 命令

==1.查看容器==
```
# 1. 查看运行中的容器
docker ps
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Ports}}"

# 2. 查看所有容器，包括已停止的容器
docker ps -a


# 3. 查看所有容器，并只打印 IMAGE ID（常用于一键删除所有容器）
docker ps -a -q


# 4. 查看容器详细信息
docker inspect <container-id> / <container-name> 


# 5. 查看容器日志
docker logs <container-id> / <container-name> 
```
![](source/_posts/笔记：Docker%20基础/image-20250313104542342.png)
1. ==CONTAINER ID==：
	1. 容器的唯一 ID（截断版，前12位，后续引用时可简写，只要能唯一区分）
2. ==IMAGE==：
	1. 容器所基于的镜像名称
3. ==COMMAND==：
	1. 启动容器时执行的命令（截断显示）
4. ==CREATED==：
	1. 容器创建的时间
5. ==STATUS==：容器的当前状态。
	1. <font color="#00b0f0">Up</font>：容器正在运行，`Up 2 hours` 代表已运行 2小时
	2. <font color="#00b0f0">Exited</font>：容器已停止
	3. <font color="#00b0f0">Restarting</font>：容器正在重启
	4. <font color="#00b0f0">Paused</font>：容器已暂停
	5. <font color="#00b0f0">Created</font>：容器已创建但未启动
6. ==PORTS==：
	1. 容器端口映射，`80/tcp` 代表容器暴露了 80 端口，使用的是 TCP 协议
7. ==NAME==：
	1. 容器的名称（若未手动指定则自动生成）

---


==2.启动容器==
```
# 1. 启动一个新的容器
docker run <选项> <image-name>:<image-tag>


# 2. 启动一个停止的容器
docker start <container-id> / <container-name>


# 3. 重启一个容器
docker restart <container-id> / <container-name>
```

> [!NOTE] 注意事项
> 1. 如果宿主机（主机）关闭或重启，正在运行的 Docker 容器会自动停止。这是因为容器依赖宿主机的资源（如 CPU、内存等）运行。若希望容器在宿主机重启时自动启动，需要在启动容器时或修改容器的基本信息时配置 `--restart` 重启策略
> 2. 当通过 `docker run` 启动一个新容器时，首先会在本地查找指定的镜像。如果镜像已存在，则直接启动容器。如果镜像不存在本地，则会从镜像中心（如 Docker Hub）查找。如果镜像中心存在该镜像，它会被下载到本地并启动容器；如果镜像中心也没有该镜像，则会报错。
> 3. `docker run` 命令的常用选项：
> 	- <font color="#00b0f0">-d</font>：
> 		- 在后台启动容器并返回容器完整 ID，可以使用 `docker exec` 命令进入容器
> 		- 该选项非常重要，否则容器只会在当前终端的前台运行，退出终端时容器会终止
> 	- <font color="#00b0f0">-p</font>：
> 		- 用于端口映射，如 `-p 8080:80`，将宿主机的 8080 端口映射到容器的 80 端口
> 	- <font color="#00b0f0">--name</font>：
> 		- 用于为容器指定名称，如 `--name mynginx`。如果不指定名称，容器将随机分配一个名称
> 	- <font color="#00b0f0">--restart</font>
> 		- <font color="#7030a0">--restart unless-stopped</font>:
> 			- 容器会在 Docker 服务启动时自动启动，除非容器被手动停止（如 `docker stop`）。
> 			- 手动停止后，容器不会自动重启，直到手动启动。
> 		- <font color="#7030a0">--restart always</font>:
> 			- 容器总是会在 Docker 服务启动时自动启动


==3.停止容器==
```
docker stop <container-id> / <container-name>
```


==4.删除容器==
```
# 1. 删除容器（需先停止容器）
docker rm <container-id> / <container-name>


# 2. 强制删除容器（无需停止容器）
docker rm -f <container-id> / <container-name>


# 3. 一键强制删除所有容器
docker rm -f $(docker ps -aq)
```

> [!NOTE] 注意事项
> 1. 删除镜像是 `docker rmi` 命令，而删除容器是 `docker rm` 命令


==4.更新容器==
```
# 1. 修改容器的名称
docker rename my_old_container my_new_container


# 2. 修改容器的重启策略
docker update --restart=unless-stopped my_container
docker update --restart=always my_container
```

> [!NOTE] 注意事项
> 1. 已创建的容器无法直接挂载数据卷或修改数据卷映射。如果有这种需求，可以基于原容器的配置重新创建一个容器，并在新容器中挂载所需的数据卷或数据卷映射。


==5.进入 / 退出容器==
```
# 1. 进入容器，并以 bash 命令行交互模式（最常用）
docker exec -it <container-id or name> /bin/bash


# 2. 退出容器
exit
```


==6.容器与宿主机之间的文件拷贝==
```
# 1. 宿主机文件拷贝到容器
docker cp /path/on/host your-container-name:/path/in/container

# 2. 容器文件拷贝到宿主机
docker cp your-container-name:/path/in/container /path/on/host
```

---


### 6.3. Docker Registry（仓库）

##### 6.3.1. Docker Registry 概述

Docker Registry 是用于存储和分发 Docker 镜像的仓库，允许开发者可以方便地分享和下载容器镜像

----


##### 6.3.2. DockerHub

[DockerHub](https://hub.docker.com/) 是 Docker 官方出品的公共仓库，如果要像 DockerHub 上推送镜像或拉取私人镜像，可以使用以下步骤：

==1.登录 DockerHub==
在将镜像推送到 Docker Hub 前，必须先登录你的 Docker Hub 账户，登录成功后，你就拥有了在 Docker Hub 上执行诸如推送镜像的权限。
```
docker login
```


==2.给镜像重新打标签（Tag）==
```
docker tag mynginx:latest leifengyang/mynginx:v1.0
```
1. ==docker tag==：
	1. 给已有的镜像打一个新的标签
2. ==mynginx:latest==：
	1. 本地已有的镜像名和版本
3. ==leifengyang/mynginx:v1.0==：
	1. Docker Hub 对镜像的命名有固定规范，格式为：`用户名/镜像名:版本号`

> [!NOTE] 注意事项
> 1. 为什么需要重打标签？
> 	- Docker Hub 要求镜像名称必须带有用户名前缀
> 	- 如果直接推送本地镜像（如 `mynginx:latest`），由于缺少用户名前缀，Docker Hub 无法识别该镜像属于哪个用户或命名空间
> 1. 重打标签后，镜像的 ID 不变，说明什么？
> 	- 重打标签本质上只是为已有镜像创建一个新的“别名”，即使镜像拥有多个标签，它们本质上都指向同一个镜像实例，共享同一个镜像 ID


==3.推送镜像到 Docker Hub==
```
docker push leifengyang/mynginx:v1.0
```


==4.再推送一个最新版本的镜像==
```
docker tag mynginx:latest leifengyang/mynginx:latest
docker push leifengyang/mynginx:latest
```

> [!NOTE] 注意事项：为什么需要再推送一个最新版本的镜像？
> 1. `latest` 是 Docker 的一个惯例标签，通常表示某个镜像的最新稳定版本，使用者在拉取镜像时，如果没有指定版本标签，Docker 默认会拉取 `latest`

---


##### 6.3.3. 阿里云容器 仓库

[阿里云容器镜像服务](https://cr.console.aliyun.com/cn-hangzhou/instances)是阿里云提供的高效、可靠的镜像托管平台，具备镜像构建、安全扫描等实用功能。在国内访问速度较快，尤其适合已深度使用阿里云容器服务、弹性计算等云产品的企业。选择阿里云搭建私有镜像仓库，可以在集成性和管理便利性上获得显著优势。

具体使用步骤可以参考阿里云容器镜像服务 ACR，操作直观，无需过多说明。

需要注意的是，阿里云镜像仓库支持存储**同一应用**的**多个版本镜像**，而非多个应用。用户应根据业务需求按应用划分仓库，并通过标签管理不同版本的镜像。

---


### 补充：常用网站

1. Docker Hub：
	1. https://hub.docker.com/
2. Docker 官网：
	1. https://www.docker.com/