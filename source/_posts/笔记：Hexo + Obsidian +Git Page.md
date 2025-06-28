---
title: 笔记：Hexo + Obsidian +Git Page
date: 2025-03-10
categories:
  - 博客建站
  - Hexo + Obsidian + GitPage
tags: 
author: 霸天
layout: post
password: wq666666
---
### 1. 0、导图：[Map：Hexo + Obsidian + Git Page](../../maps/Map：Hexo+Obsidian+GitPage.xmind)

---


### 2. 环境准备
1. 安装 Git
2. 安装 Node.js
3. 安装 PowerShell


---

### 3. 创建 Git Page

首先，登录 GitHub 并创建一个名为 `id.github.io` 的仓库，并确保将其设置为 Public（公开）。如果我们在后面执行了 `hexo d`（即 `hexo deploy`）命令，此时 Hexo 会将 `public/` 文件夹中生成的静态文件上传至该仓库。上传完成后，访问 `id.github.io` 就能看到我们的页面了，当然这都是后话了，当先任务是创建 Git Page 如下图所示：
![](image-20250310162727945.png)

---


### 4. 创建 Git Repository

由于 GitHub Pages 仅存储 `/page` 目录中的数据，无法确保文件丢失后能够直接恢复，因此，我们额外创建一个名为 `myNote`（名称可自定义）的仓库，并确保将其设置为 Private（私有）用于备份所有文件，以防数据丢失
![](image-20250310164200172.png)

---


### 5. 进行 Git 代理配置

1. ==检查是否配置过 Git 代理==
```
git config --global http.proxy
git config --global https.proxy
```

2. ==如果存在代理，进行代理取消==
```
git config --global --unset http.proxy
git config --global --unset https.proxy
```

3. ==重新配置 Git 代理==
```
git config --global https.proxy 127.0.0.1:7890
git config --global http.proxy 127.0.0.1:7890
```

---


### 6. 创建 Hexo 项目

在本地创建一个名为 `myNote`（名称可自定义）的文件夹，然后点击该文件夹右键选择“使用 PowerShell 打开”，并运行以下命令：
1. ==全局安装 hexo==
```
# 全局安装 Hexo，以便在任何位置都能运行 Hexo 命令
npm install -g hexo-cli
```

2. ==初始化 Hexo 项目==
```
hexo init
```

3. ==安装相关依赖==
```
npm install
```

---


### 7. 配置元数据

在系统 `_config.yml` 文件中配置元数据：
![](image-20250310173524507.png)

---


### 8. 配置主题

你可以在 [Hexo主题库](https://hexo.io/themes/) 中挑选一个自己喜欢的主题。我选择 [Fluid](https://github.com/fluid-dev/hexo-theme-fluid) 作为我的主题。

1. ==下载 fluid 安装包==
![](image-20250310190105538.png)


2. ==解压缩到 theme 目录下，并将其重命名为 fluid==
![](image-20250310190142750.png)


3. ==进行系统 config.yml 配置==
```
# 使用 fluid 主题
theme: fluid
```


4. ==添加 about 页面==
```
hexo new page about
```


5. ==设置 about/index.md 的头部属性==
![](image-20250310190542177.png)


6. ==进行主题 config.yml 配置==
根据需要在主题的 `config.yml` 文件中进行相应的配置。

---


### 9. 给 Hexo 加个锁
```
# 1. 安装 hexo-blog-encrypt
npm install --save hexo-blog-encrypt


# 2. 在文章头部使用 password
password: xxxxxx
```

---


### 10. 项目关连到 Repository

在 PowerShell 中运行以下命令，将其与 Git Repository 关联：

1. ==初始化本地 Git 仓库==
```
git init
```


2. ==与 Repository 关连==
```
# 本地仓库与远程仓库进行关联。
git remote add origin https://github.com/wangjia5289/myNote.git

# 将当前分支强制重命名为 main
git branch -M main

# 进行第一次推送，使用 -u 选项，将本地 main 分支与远程 main 分支关联，之后可以直接使用 git push 和 git pull，而无需每次指定远程仓库和分支。
git push -u origin main
```

---


### 11. 集成 Obsidian

#### 11.1. 创建模版

在Hexo中，为确保每篇文章都能以预设格式和样式展示，通常需要在文章头部设置特定的属性。然而，手动设置这些属性对于每一次撰写新文章来说可能显得繁琐。为简化这一过程，可以创建一个模板。这样，每次撰写新文章时，只需导入该模板，即可自动包含所需的默认属性设置。

1. ==创建模版==
![](image-20250310171119367.png)

2. ==设置模版==
![](image-20250310171208901.png)


#### 11.2. 图片互通

1. ==修改 Obsidian 配置==
![](image-20250310171539936.png)


2. ==安装 Custom Attachment Location 插件==
![](image-20250310171747324.png)


3. ==进行插件配置==
![](image-20250310171902656.png)


4. ==进行系统 config.yml 配置==
```
# 1. 将 post_asset_foler 设置为 true
post_asset_folder: true


# 2. 添加下述内容
marked: 
  prependRoot: true
  postAsset: true
```


---


#### 11.3. 标题栏过滤

1. ==安装 File Explorer++== 
![](image-20250310171951376.png)


2. ==进行插件配置==
![](image-20250310172043190.png)

---


#### 11.4. Obsidian 中推送到 Repository

目标是能够在 Obsidian 中直接以手动或定时的方式将内容推送到 Repository，而无需通过命令行操作。

1. ==安装 Git 插件==
![](image-20250310172338749.png)


2. ==配置插件==
![](image-20250310172624881.png)


3. ==手动实现推送到 Repository==
![](image-20250310172731797.png)

---


### 12. 项目关连 Page

点击 `myNote` 文件夹右键选择“使用 PowerShell 打开”，并运行以下命令：

1. ==安装 deploy 插件==
```
npm install hexo-deployer-git --save
```


2. ==进行系统 config.yml 配置==
```
deploy:
  type: git
  repo: https://github.com/wangjia5289/wangjia5289.github.io.git
  branch: main
```

---


### 13. 项目发布到 Page

在 PowerShell 中运行以下命令：
```
# 1. 清除缓存和生成的静态文件
hexo clean

# 2. 根据当前的配置和文章，生成静态网页文件
hexo g

# 3. 将生成的静态网页发布到指定的 Page
hexo d

# 4. 补充：启动本地服务器，预览生成的网页效果，便于本地调试（http:localhost:4000）
hexo s
```

---


### 14. Page 与域名绑定

1. ==登录 GitHub，在您的 id.github.io 项目中，通过 Settings选项进行域名配置==
![](image-20250310175254198.png)


2. ==在 public 文件夹下创建一个名为 CNAME 的文件，并在其中保存您希望绑定的域名==
![](image-20250310174709288.png)


2. ==为域名添加 CNAME 记录并指向 id.github.io==
![](image-20250310174924876.png)

---


### 补充：相关网站

1. Hexo 主题库：
	1. https://hexo.io/themes/

