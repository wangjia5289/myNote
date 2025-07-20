---
title: 笔记：AutoHotKey
date: 2025-07-20
categories:
  - 操作系统
  - Windows
  - AutoHotKey
tags: 
author: 霸天
layout: post
---
## 使用流程

### 安装 AutoHotKey

打开官网：[https://www.autohotkey.com/](https://www.autohotkey.com/)，下载最新版

---


### 创建脚本

任意位置右键，新建AutoHotkey Script：
![](image-20250720134431254.png)

命名为：`jump_to_line_end.ahk`，并输入以下内容：
```
; Ctrl + ; 让光标跳到行尾（模拟按 End 键）
^;::
Send, {End}
Return
```

---


### 运行脚本

保存后，双击该 .ahk 文件运行，你就会在右下角托盘看到绿色 H 图标

---


### 设置开机自动运行

Win + R 输入 `shell:startup` 打开 “启动” 文件夹，然后将你的 `jump_to_line_end.ahk` 文件（或它的快捷方式）复制到这个目录下，这样每次开机就自动运行这个脚本了。

----


## 常用脚本

### ctrl + ; 跳到行尾（模拟按 End 键）

```
; Ctrl + ; 让光标跳到行尾（模拟按 End 键）
^;::
Send, {End}
Return
```

----








