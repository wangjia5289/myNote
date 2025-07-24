---
title: 笔记：Hexo + Obsidian +GitHub Pages
date: 2025-03-10
categories:
  - 博客建站
  - Hexo + Obsidian +GitHub Pages
tags: 
author: 霸天
layout: post
password: wq666666
---
## 1. 导图：[Map：Hexo + Obsidian + GitHub Pages](Map：Hexo+Obsidian+GitPage.xmind)

---


## 2. 环境准备

1. 安装 Git
2. 安装 Node.js
3. 安装 PowerShell

---


## 3. 创建 Hexo 项目

### 3.1. 初始化 Hexo 项目

<font color="#92d050">1. 创建文件夹</font>
这里我命名为 `myNote`（名称可自定义）
![](image-20250722220723829.png)


<font color="#92d050">2. 使用 PowerShell 打开文件夹</font>
![](image-20250722220832053.png)


<font color="#92d050">3. 全局安装 hexo</font>
```
npm install -g hexo-cli
```


<font color="#92d050">4. 初始化 Hexo 项目</font>
```
hexo init
```


<font color="#92d050">5. 安装相关依赖</font>
```
npm install
```

---


### 3.2. 配置 Hexo 元数据

<font color="#92d050">1. 使用 VS Code 打开 myNote 文件夹</font>
![](image-20250722221155776.png)


<font color="#92d050">2. 配置 Hexo 元数据</font>
在系统 `myNote/_config.yml` 文件中配置元数据：
![](image-20250310173524507.png)

---


### 3.3. 启动 Hexo，查看效果

<font color="#92d050">1. 使用 PowerShell 打开文件夹</font>
![](image-20250722220832053.png)


<font color="#92d050">2. 依次执行命令</font>
```
# 1. 清除现存的 Hexo 的静态文件
hexo clean


# 2. 生成新的 Hexo 静态文件
hexo g


# 3. 启动本地服务器，预览生成的网页效果，便于本地调试
hexo s
```
![](image-20250722221458392.png)


<font color="#92d050">3. 查看 Hexo 效果</font>
查看： http://localhost:4000/

---


### 3.4. 配置 Hexo 主题

上面展示的 Hexo 默认样式较为简陋，视觉效果不佳。你可以在 [Hexo主题库](https://hexo.io/themes/) 中挑选一个自己喜欢的主题，这里我选择 [Fluid](https://github.com/fluid-dev/hexo-theme-fluid) 作为我的主题。

<font color="#92d050">1. 下载 fluid 安装包</font>
![](image-20250722221633784.png)


<font color="#92d050">2. 解压缩到 myNote/theme 目录下，并将其重命名为 fluid</font>
![](image-20250722221726737.png)


<font color="#92d050">3. 配置使用 fluid 主题</font>
![](image-20250722221831285.png)


<font color="#92d050">4. 使用 PowerShell 打开文件夹</font>
![](image-20250722220832053.png)


<font color="#92d050">5. 添加 about 页面</font>
```
hexo new page about
```


<font color="#92d050">6. 设置 about/index.md 的头部属性</font>
![](image-20250722225302802.png)


<font color="#92d050">7. 进行 Hexo 主题配置</font>
根据需要在主题的 `myNote/themes/fluid/config.yml` 文件中进行相应的配置，具体怎么配，看你自己喜欢就行。


<font color="#92d050">8. 启动 Hexo，查看效果</font>
```
# 1. 清除现存的 Hexo 的静态文件
hexo clean


# 2. 生成新的 Hexo 静态文件
hexo g


# 3. 启动本地服务器，预览生成的网页效果，便于本地调试
hexo s
```

---


## 4. 集成 GitHub Pages

### 4.1. 进行 Git 代理配置

由于需要将静态页面部署到 GitHub Pages，并将整个 Obsidian 仓库推送到 MyNote 仓库，我们先设置 Git 代理，以提升网络传输速度

<font color="#92d050">1. 检查是否配置过 Git 代理</font>
```
git config --global http.proxy


git config --global https.proxy
```


<font color="#92d050">2. 如果存在代理，进行代理取消</font>
```
git config --global --unset http.proxy


git config --global --unset https.proxy
```


<font color="#92d050">3. 重新配置 Git 代理</font>
```
git config --global https.proxy 127.0.0.1:7890


git config --global http.proxy 127.0.0.1:7890
```

> [!NOTE] 注意事项
> 1. 此处的端口号需根据你本地 Clash 的配置进行调整，填写你实际使用的代理端口。

![](image-20250722220538295.png)

---


### 4.2. 创建 GitHub Pages

首先，登录 GitHub 并创建一个名为 `<your_id>.github.io` 的仓库，并确保将其设置为 Public（公开）
![](image-20250310162727945.png)

---


### 4.3. Hexo 项目关联到 GitHub Pages

<font color="#92d050">1. 安装 deploy 插件</font>
```
npm install hexo-deployer-git --save
```


<font color="#92d050">2. 进行 myNote/config.yml 配置</font>
```
deploy:
  type: git
  repo: https://github.com/wangjia5289/wangjia5289.github.io.git
  branch: main
```

---


### 4.4. Hexo 项目推送到 GitHub Pages

```
# 1. 清除现存的Hexo的静态文件
hexo clean


# 2. 生成新的 Hexo 静态文件
hexo g


# 3. 启动本地服务器，预览生成的网页效果，便于本地调试
hexo s


# 4. 将 Hexo 项目推送到 GitHub Pages
hexo d
```

---


### 4.5. 查看 GitHub Pages 效果

查看： https://github.com/wangjia5289/wangjia5289.github.io.git

----


## 5. 集成 GitHub Repository

<font color="#92d050">1. 创建 GitHub Repository</font>
我们额外创建一个名为 `myNote`（名称可自定义）的仓库，并确保将其设置为 Private（私有）用于备份所有文件，以防数据丢失
![](image-20250310164200172.png)


<font color="#92d050">2. 依次运行命令</font>
```
# 1. 初始化本地 Git 仓库
git init


# 2. 创建 README 文档
echo "随便写点东西好了" > README.md


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

---


## 6. 集成 Obsidian

### 6.1. 创建模版

在Hexo中，为确保每篇文章都能以预设格式和样式展示，通常需要在文章头部设置特定的属性。然而，手动设置这些属性对于每一次撰写新文章来说可能显得繁琐。为简化这一过程，可以创建一个模板。这样，每次撰写新文章时，只需导入该模板，即可自动包含所需的默认属性设置。

<font color="#92d050">1. 创建模版</font>
![](image-20250310171119367.png)

<font color="#92d050">2. 设置模版</font>
![](image-20250310171208901.png)

> [!NOTE] 注意事项：Hexo 页面加锁
```
# 1. 安装 hexo-blog-encrypt
npm install --save hexo-blog-encrypt


## 2. 在文章头部添加 password
password: xxxxxx
```

----


### 6.2. Custom Attachment Location 插件

<font color="#92d050">1. 修改 Obsidian 配置</font>
![](image-20250310171539936.png)


<font color="#92d050">2. 安装 Custom Attachment Location 插件</font>
![](image-20250310171747324.png)


<font color="#92d050">3. 进行插件配置</font>
![](image-20250310171902656.png)


<font color="#92d050">4. 进行 myNote/config.yml 配置</font>
```
# 1. 将 post_asset_foler 设置为 true
post_asset_folder: true


# 2. 添加下述内容
marked: 
  prependRoot: true
  postAsset: true
```

---


### 6.3. File Explorer++ 插件

<font color="#92d050">1. 安装 File Explorer++ 插件</font>
![](image-20250310171951376.png)


<font color="#92d050">2. 进行插件配置</font>
![](image-20250310172043190.png)

---


### 6.4. Git 插件

<font color="#92d050">1. 安装 Git 插件</font>
![](image-20250310172338749.png)


<font color="#92d050">2. 配置插件</font>
![](image-20250310172624881.png)


<font color="#92d050">3. 手动实现推送到 Repository</font>
![](image-20250310172731797.png)

---


## 7. Page 与域名绑定（可选）

<font color="#92d050">1. 登录 GitHub，在您的 id.github.io 项目中，通过 Settings选项进行域名配置</font>
![](image-20250310175254198.png)


<font color="#92d050">2. 在 public 文件夹下创建一个名为 CNAME 的文件，并在其中保存您希望绑定的域名</font>
![](image-20250310174709288.png)


<font color="#92d050">3. 为域名添加 CNAME 记录并指向 id.github.io</font>
![](image-20250310174924876.png)

---


## 8. 项目发布到云服务器（可选）

### 8.1. 开放云服务器 6666 端口和安全组

![](image-20250722231904464.png)

-----


### 8.2. 上传 /public 文件夹到 /mystudy/hexo-blog 文件夹下

```
// 1. 创建 /mystudy/hexo-blog 文件夹
mkdir -p /mystudy/hexo-blog


// 2. 上传 /public 文件夹到 /mystudy/hexo-blog 文件夹下


// 3. 进入 /mystudy/hexo-blog 文件夹
cd /mystudy/hexo-blog/public


// 4. 使用 python 启动 web 服务
// 4.1. 前台启动
python3 -m http.server 6666

// 4.2. 后台启动
nohup python3 -m http.server 6666 > server.log 2>&1 &

// 5. 测试访问 web 服务
访问：http://117.72.211.221:6666


// 6. 将该服务做成系统 service
sudo vim /etc/systemd/system/hexo-blog.service
“”“
[Unit]
Description=Hexo Blog Python HTTP Server
After=network.target

[Service]
User=root                                                                       # 这里需要改成你的用户名
WorkingDirectory=/mysudy/hexo-blog/public              # 这里需要改成你的 public 路径
ExecStart=/usr/bin/python3 -m http.server 666666
Restart=always
StandardOutput=append:/home/ubuntu/hexo-blog/http.log
StandardError=append:/home/ubuntu/hexo-blog/http.err.log

[Install]
WantedBy=multi-user.target
”“”


7. 重新加载 systemd 配置
sudo systemctl daemon-reexec && sudo systemctl daemon-reload


8. 设置该 web 服务开机自启动
sudo systemctl enable hexo-blog


// 9. 启动该 web 服务
sudo systemctl start hexo-blog


// 10. 查看该 web 服务运行状态
sudo systemctl status hexo-blog
```






