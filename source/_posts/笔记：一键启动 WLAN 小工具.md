---
title: 笔记：一键启动 WLAN 小工具
date: 2025-07-06
categories:
  - 自研小工具
  - 一键启动 WLAN 小工具
tags: 
author: 霸天
layout: post
---
## 1. Py 代码

创建 `一键启动 WLAN 小工具.py`，并编写代码：
```
import subprocess

def check_service_status(service_name):
    result = subprocess.run(["sc", "query", service_name], capture_output=True, text=True)
    return "RUNNING" in result.stdout

def start_wifi_service():
    service_name = "WlanSvc"
    if check_service_status(service_name):
        print(f"ℹ️ 服务 '{service_name}' 已在运行中")
        return

    try:
        subprocess.run(["sc", "start", service_name], check=True)
        print(f"✅ 服务 '{service_name}' 已成功启动")
    except subprocess.CalledProcessError:
        print(f"❌ 启动服务 '{service_name}' 失败，请以管理员权限运行")

if __name__ == "__main__":
    start_wifi_service()
```

----


## 2. 打包为 exe

```
# 1. 安装打包工具
pip install pyinstaller


# 2. 打包代码
pyinstaller --onefile --windowed --icon="D:\文件集合\ICO\一键启动 WLAN 小工具.ico" --distpath "D:\文件集合" "D:\文件集合\一键启动 WLAN 小工具.py"
"""
1. --onefile
	1. 打包为单个 .exe 文件。
2. --windowed
	1. 确保 GUI 程序无控制台窗口
"""
```


