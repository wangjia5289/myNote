---
title: 笔记：Obsidian 插件制作（自用）
date: 2025-05-13
categories:
  - 博客
tags: 
author: 霸天
layout: post
---
### 新建 Obsidian 仓库

我们先创建一个叫“插件”的新仓库，专门用来做测试，避免误操作影响原有内容。此外别忘了在设置中启用“开发者模式”（设置 > 第三方插件 > 关闭安全模式）。

---


### 克隆 Obsidian 插件模板

创建 `ob_plugin` 目录，并在该目录执行以下代码后，使用 VS 打开 `my-obsidian-plugin` 目录：
```
git clone https://github.com/obsidianmd/obsidian-sample-plugin.git my-obsidian-plugin
```

----


### 安装 TypeScript、Obsidian 依赖

Obsidian官方更支持使用 TypeScrip 制作插件，所以我们这里使用 typescript 进行制作
```
npm install -g typescript 

npm install --save-dev obsidian
```

---


























编写一个自用的Obsidian插件需要了解Obsidian的插件API、JavaScript/TypeScript开发以及插件的基本结构。以下是一个简要的指南，帮助你快速入门并创建一个简单的Obsidian插件（以添加自定义命令为例）。我假设你有基本的JavaScript知识，并且熟悉VS Code等开发环境。

### 规划
1. **目标**：创建一个简单的Obsidian插件，添加一个命令，当触发时在当前笔记中插入当前时间戳。
2. **技术栈**：使用TypeScript（推荐，Obsidian官方更支持TypeScript），并基于Obsidian插件模板。
3. **步骤**：
   - 设置开发环境。
   - 使用Obsidian插件模板初始化项目。
   - 编写插件逻辑（注册命令，插入时间戳）。
   - 打包和测试插件。

### 开发环境设置
1. 安装Node.js（建议最新LTS版本）。
2. 安装VS Code或其他代码编辑器。
3. 全局安装TypeScript：`npm install -g typescript`。
4. 确保你的Obsidian已启用“开发者模式”（设置 > 第三方插件 > 关闭安全模式）。

### 实现步骤
以下是一个简单的插件示例，包含项目结构和代码。

#### 1. 初始化项目
Obsidian官方提供了一个插件模板，可以快速开始：
- 克隆Obsidian插件模板：
  ```bash
  git clone https://github.com/obsidianmd/obsidian-sample-plugin.git my-obsidian-plugin
  cd my-obsidian-plugin
  ```
- 安装依赖：
  ```bash
  npm install
  ```

#### 2. 修改插件代码
我们将创建一个名为“Timestamp Inserter”的插件，添加一个命令来插入当前时间戳。

编辑以下文件：
- **`main.ts`**: 插件的主入口文件，包含插件逻辑。
- **`manifest.json`**: 插件的元数据文件，描述插件信息。
- **`package.json`**: Node.js项目配置文件。

#### 3. 代码实现
以下是具体的代码：

```typescript
import { Plugin } from 'obsidian';

export default class TimestampPlugin extends Plugin {
  async onload() {
    // 注册一个命令
    this.addCommand({
      id: 'insert-timestamp',
      name: 'Insert Current Timestamp',
      editorCallback: (editor, view) => {
        // 获取当前时间并格式化为字符串
        const timestamp = new Date().toLocaleString();
        // 在光标处插入时间戳
        editor.replaceSelection(`[${timestamp}] `);
      },
    });
  }

  onunload() {
    // 插件卸载时的清理逻辑（如果需要）
  }
}
```

```json
{
  "id": "timestamp-inserter",
  "name": "Timestamp Inserter",
  "version": "1.0.0",
  "minAppVersion": "0.15.0",
  "description": "A simple plugin to insert the current timestamp into your notes.",
  "author": "Your Name",
  "isDesktopOnly": false
}
```

```json
{
  "name": "timestamp-inserter",
  "version": "1.0.0",
  "description": "A simple Obsidian plugin to insert the current timestamp.",
  "main": "main.js",
  "scripts": {
    "dev": "node esbuild.config.mjs",
    "build": "tsc -noEmit -skipLibCheck && node esbuild.config.mjs production",
    "version": "node version-bump.mjs",
    "test": "jest"
  },
  "keywords": [],
  "author": "Your Name",
  "license": "MIT",
  "devDependencies": {
    "@types/node": "^16.11.6",
    "@typescript-eslint/eslint-plugin": "5.29.0",
    "@typescript-eslint/parser": "5.29.0",
    "builtin-modules": "3.3.0",
    "esbuild": "0.17.3",
    "obsidian": "latest",
    "tslib": "2.4"
  }
}
```

#### 4. 编译和测试
- 运行开发模式（自动编译）：
  ```bash
  npm run dev
  ```
- 编译后的文件（`main.js`、`manifest.json`等）会出现在项目根目录。
- 将这些文件复制到Obsidian的插件目录：
  - 路径：`你的Obsidian Vault/.obsidian/plugins/timestamp-inserter/`
  - 确保目录名为`timestamp-inserter`，与`manifest.json`中的`id`一致。
- 在Obsidian中启用插件：
  - 打开Obsidian设置 > 第三方插件，找到“Timestamp Inserter”并启用。
- 测试插件：
  - 打开命令面板（Ctrl/Cmd + P），输入“Insert Current Timestamp”，执行命令。
  - 当前时间戳（如`[2025-05-13 14:30:00]`）应插入到光标位置。

### 总结与使用说明
- **功能**：此插件添加了一个命令，允许用户在笔记中插入当前时间戳。
- **扩展**：你可以进一步扩展插件，例如：
  - 添加设置界面（使用`SettingTab`类）。
  - 支持自定义时间戳格式。
  - 添加热键绑定（通过`this.addCommand`的`hotkeys`属性）。
- **调试**：在Obsidian中按Ctrl/Cmd + Shift + I打开开发者工具，查看控制台日志。
- **发布**：如果想分享插件，可以提交到Obsidian社区插件仓库（参考Obsidian官方文档）。

如果需要更复杂的功能（例如自定义UI、处理Markdown文件等），请提供更多细节，我可以进一步扩展代码！