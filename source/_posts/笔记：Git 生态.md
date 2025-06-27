---
title: 笔记：Git 生态
date: 2025-04-13
categories:
  - 版本控制
  - Git 生态
tags: 
author: 霸天
layout: post
---



进行 Git 代理配置

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




第一次
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
git remote add origin https://github.com/wangjia5289/myNote.git

# 7. 进行分支关联（使用 -u 选项，将本地 main 分支与远程 main 分支关联，之后可以直接使用 git push 和 git pull，而无需每次指定远程仓库和分支。）
git push -u origin main


# 8. 后续推送
我们可以通过命令行或使用 GitHub Desktop 等工具来推送代码。
```










