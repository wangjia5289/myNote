---
title: 笔记：Markdown图片路径修复小工具
date: 2025-07-23
categories:
  - 自研小工具
  - Markdown 图片路径修复小工具
tags: 
author: 霸天
layout: post
---
## 1. Py 代码

创建 `Markdown 图片路径修复小工具.py`，并编写代码：
```
import os
import re
import urllib.parse
import tkinter as tk
from tkinter import filedialog, messagebox
from tkinter import ttk

def process_file(file_path):
    with open(file_path, 'r', encoding='utf-8') as f:
        content = f.read()

    # 匹配 ![](image-xxx.png)
    pattern = r'!\[\]\(source/_posts/[^)]*?/((image-[^)]*?))\)'

    # 替换为 ![](image-xxx.png)
    new_content = re.sub(pattern, r'![](\1)', content)

    # 解码   为 空格 等
    new_content = urllib.parse.unquote(new_content)

    if content != new_content:
        with open(file_path, 'w', encoding='utf-8') as f:
            f.write(new_content)
        return True
    return False

def select_folder_and_fix():
    folder_path = filedialog.askdirectory(title="选择包含 Markdown 文件的文件夹")
    if not folder_path:
        return

    btn.config(state='disabled')
    status_label.config(text="正在处理...")
    root.update()

    changed_files = []
    for root_dir, dirs, files in os.walk(folder_path):
        for file in files:
            if file.endswith('.md'):
                full_path = os.path.join(root_dir, file)
                if process_file(full_path):
                    changed_files.append(full_path)

    btn.config(state='normal')
    if changed_files:
        status_label.config(text=f"已修复 {len(changed_files)} 个文件。")
        messagebox.showinfo("处理完成", f"已修复 {len(changed_files)} 个文件。")
    else:
        status_label.config(text="没有发现需要修复的路径。")
        messagebox.showinfo("无需修改", "没有发现需要修复的路径。")

# UI 界面
root = tk.Tk()
root.title("Markdown 图片路径修复工具")
root.geometry("500x250")
root.resizable(False, False)
root.configure(background='lightgray')

style = ttk.Style()
style.theme_use('clam')
style.configure('TLabel', font=('Arial', 12), background='lightgray')
style.configure('TButton', font=('Arial', 12), foreground='white', background='blue')

title_label = ttk.Label(root, text="Markdown 图片路径修复工具", font=('Arial', 16, 'bold'))
title_label.grid(row=0, column=0, pady=10)

label = ttk.Label(root, text="点击下方按钮选择 Markdown 文件夹进行处理：")
label.grid(row=1, column=0, pady=10)

btn = ttk.Button(root, text="选择文件夹并开始修复", command=select_folder_and_fix, padding=10)
btn.grid(row=2, column=0, pady=10)

status_label = ttk.Label(root, text="")
status_label.grid(row=3, column=0, pady=10)

root.columnconfigure(0, weight=1)
root.rowconfigure(0, weight=1)
root.rowconfigure(1, weight=1)
root.rowconfigure(2, weight=1)
root.rowconfigure(3, weight=1)

root.mainloop()
```

---


## 2. 打包为 exe

```
# 1. 安装打包工具
pip install pyinstaller


# 2. 打包代码
pyinstaller --onefile --windowed --icon="D:\文件集合\ICO\Markdown 图片路径修复小工具.ico" --distpath "D:\文件集合" "D:\文件集合\Markdown 图片路径修复小工具.py"
"""
1. --onefile
	1. 打包为单个 .exe 文件。
2. --windowed
	1. 确保 GUI 程序无控制台窗口
"""
```

