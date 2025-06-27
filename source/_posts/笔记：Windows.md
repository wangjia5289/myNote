---
title: 笔记：Windows
date: 2025-04-05
categories:
  - 操作系统
  - Windows
tags: 
author: 霸天
layout: post
---
## 1. 调整电源状态

### 1.1. 常见电源状态

|状态|屏幕|程序|网络连接|是否继续运行任务|
|---|---|---|---|---|
|正常运行|亮|是|是|是|
|仅息屏|关|是|是|是 ✅|
|睡眠（Sleep）|关|否（挂起）|否|否 ❌|
|休眠（Hibernate）|关|否（写入硬盘）|否|否 ❌|

---


### 1.2. 修改电源状态

控制面板 → 硬件和声音 → 电源选项 → 更改计划设置
    
---



## 查看 CPU 架构

打开 PowerShell，输入下面的指令，`AMD64`：就是 amd64，`ARM64`：就是 arm64
```
echo $env:PROCESSOR_ARCHITECTURE
```

----


## 解决网页残留问题

1. 打开注册表（Win + R ，输入 `regedit`），
2. 找到 `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\Dwm`
3. 新增 OverlayTestMode 的DWORD 项，将其值修改为 `5`，
4. 重启电脑
![](image-20250503160117848.png)

---


## 查看端口是否监听

```
PS C:\Users\ASUS> netstat -ano | Select-String "7890"

返回以下结果，说明 7890 端口确实被一个进程 PID 15144 占用了
  TCP    0.0.0.0:7890           0.0.0.0:0              LISTENING       15144
  TCP    [::]:7890              [::]:0                 LISTENING       15144
```

---


## 软件添加到 Win 

创建软件快捷方式，拖进：`C:\ProgramData\Microsoft\Windows\Start Menu\Programs`

---


## 重置网络

```
netsh winsock reset

netsh int ip reset
```

---


### 查看核数

```
Get-WmiObject -Class Win32_Processor | Select-Object Name, NumberOfCores, NumberOfLogicalProcessors
- **NumberOfCores** 是物理核数
    
- **NumberOfLogicalProcessors** 是逻辑核数（启用了超线程会是核数的 2 倍）
```
我们可以简单理解为 VM分配的核心是逻辑核心



## 测磁盘读写

很好，CrystalDiskMark 是 Windows 系统下最流行、操作最简单的硬盘读写速度测试工具之一。下面我一步一步教你怎么用：

---

## ✅ 一、下载安装

1. **访问官网下载地址**：
    
    - 官网主页：[https://crystalmark.info/en/software/crystaldiskmark/](https://crystalmark.info/en/software/crystaldiskmark/)
        
    - 选择适合你系统的版本：
        
        - 通常选“**Standard Edition**”
            
        - 如果不想安装，也可以点“**ZIP版（免安装）**”，解压后直接运行。
            
2. **安装或解压后打开**程序。
    

---

## ✅ 二、界面参数说明（见下图）

界面上方有几个关键参数，按需设置：

|参数|作用说明|推荐设置|
|---|---|---|
|① 测试次数|默认是 5 次，取平均值。可改成 3 次测试更快。|建议保持默认|
|② 测试文件大小|单次测试的数据量，单位 GiB|通常选 1GiB 或 4GiB|
|③ 测试磁盘|选择你要测试的磁盘分区（如 C: D: E:）|选你要测试的硬盘|

---

## ✅ 三、开始测试

点击中间绿色的 **“All” 按钮**，系统会依次执行 4 个读写测试：

|测试项|含义说明|
|---|---|
|Seq1M Q8T1|顺序读写（大文件），多个线程并发访问|
|Seq1M Q1T1|顺序读写（大文件），单线程访问|
|RND4K Q32T16|随机读写（小文件），高并发场景|
|RND4K Q1T1|随机读写（小文件），真实日常使用最接近|

每项测试会分别显示：

- **Read（读取速度）**
    
- **Write（写入速度）**  
    单位：MB/s（越高越好）
    

---

## ✅ 四、如何看结果

- `Seq1M` 的结果：代表你硬盘的**最高理论顺序读写速度**。
    
- `RND4K Q1T1` 的结果：代表日常使用时（比如开网页、打开软件）**平均性能表现**。
    
- 机械硬盘 `RND4K` 通常只有个位数 MB/s，NVMe SSD 能有几百甚至上千。
    

---

## ✅ 五、补充说明

- 不建议在 **系统盘（C盘）使用过程中测试**，会有干扰。
    
- 如果硬盘有缓存，第一次和第二次结果可能会差别较大（可多跑几次看波动）。
    
- 如果你想测试实际复制大文件的速度，也可以手动复制 10GB 文件试试，但这不如 CrystalDiskMark 标准化。
    

---

你要不要我帮你分析一下测试结果？你测完后发截图或结果数值就行。



![](image-20250619203524550.png)

![](image-20250619203539052.png)



![](image-20250619203359869.png)