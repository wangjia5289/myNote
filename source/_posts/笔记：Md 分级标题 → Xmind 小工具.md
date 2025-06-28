---
title: 笔记：Md 分级标题 → Xmind 小工具
date: 2025-05-12
categories:
  - 自研小工具
  - Md 分级标题 → Xmind 小工具
tags: 
author: 霸天
layout: post
---
### 编写 Py 代码

创建 `markdown_title_extractor.py` 并编写代码：
```
import tkinter as tk
from tkinter import filedialog, messagebox
import os

class MarkdownTitleExtractor:
    def __init__(self, root):
        self.root = root
        self.root.title("Markdown 标题提取器")
        self.root.geometry("620x150")
        # 居中窗口
        self.root.eval('tk::PlaceWindow . center')

        # 默认输出目录
        self.default_output_dir = r"D:\文件集合"

        # 输入文件选择
        tk.Label(root, text="输入 Markdown 文件：").grid(row=0, column=0, padx=10, pady=10, sticky='e')
        self.input_entry = tk.Entry(root, width=50)
        self.input_entry.grid(row=0, column=1, padx=10, pady=10)
        tk.Button(root, text="选择文件", command=self.select_input).grid(row=0, column=2, padx=10)

        # 输出文件选择
        tk.Label(root, text="输出 Markdown 文件：").grid(row=1, column=0, padx=10, pady=10, sticky='e')
        self.output_entry = tk.Entry(root, width=50)
        # 初始默认输出文件
        initial_out = os.path.join(self.default_output_dir, "output.md")
        self.output_entry.insert(0, initial_out)
        self.output_entry.grid(row=1, column=1, padx=10, pady=10)
        tk.Button(root, text="选择文件", command=self.select_output_file).grid(row=1, column=2, padx=10)

        # 提取标题按钮
        tk.Button(root, text="提取标题", command=self.extract_titles, width=20).grid(row=2, column=0, columnspan=3, pady=20)

    def select_input(self):
        # 默认打开指定目录
        default_dir = r"E:\1、办公\4、博客\myNote\source\_posts"
        file_path = filedialog.askopenfilename(
            title="选择输入 Markdown 文件",
            initialdir=default_dir,
            filetypes=[("Markdown 文件", "*.md"), ("所有文件", "*.*")]
        )
        if file_path:
            # 更新输入路径
            self.input_entry.delete(0, tk.END)
            self.input_entry.insert(0, file_path)
            # 根据输入文件名生成输出文件默认路径
            filename = os.path.basename(file_path)
            output_path = os.path.join(self.default_output_dir, filename)
            self.output_entry.delete(0, tk.END)
            self.output_entry.insert(0, output_path)

    def select_output_file(self):
        file_path = filedialog.asksaveasfilename(
            title="选择输出 Markdown 文件",
            initialdir=self.default_output_dir,
            defaultextension=".md",
            filetypes=[("Markdown 文件", "*.md"), ("所有文件", "*.*")]
        )
        if file_path:
            self.output_entry.delete(0, tk.END)
            self.output_entry.insert(0, file_path)

    def extract_titles(self):
        input_path = self.input_entry.get().strip()
        output_file = self.output_entry.get().strip()

        if not input_path or not output_file:
            messagebox.showerror("错误", "请同时选择输入和输出文件路径！")
            return

        try:
            with open(input_path, 'r', encoding='utf-8') as infile, \
                    open(output_file, 'w', encoding='utf-8') as outfile:
                outfile.write("# Map:\n\n")
                in_code_block = False
                for line in infile:
                    stripped = line.strip()
                    if stripped.startswith('```'):
                        in_code_block = not in_code_block
                        continue
                    if not in_code_block and stripped.startswith('#'):
                        outfile.write(line)
            messagebox.showinfo("成功", f"标题提取成功！\n输出文件已保存到：{output_file}")
        except Exception as e:
            messagebox.showerror("错误", f"发生错误：{e}")

if __name__ == "__main__":
    root = tk.Tk()
    app = MarkdownTitleExtractor(root)
    root.mainloop()
```

---


### 打包为可执行文件

```
# 1. 安装打包工具
pip install pyinstaller


# 2. 打包代码
pyinstaller --onefile --windowed --distpath D:\文件集合\ D:\文件集合\markdown_title_extractor.py
"""
1. --onefile
	1. 打包为单个 .exe 文件。
2. --windowed
	1. 确保 GUI 程序无控制台窗口
3. --distpath D:\文件集合\
	1. 指定输出位置
"""
```
