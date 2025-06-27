---
title: 笔记：Python
date: 2025-05-12
categories:
  - Python
tags: 
author: 霸天
layout: post
---
```
C:\Users\ASUS\AppData\Local\Programs\Python\Python39
```

更换 Python 版本可以通过多种方式实现，具体取决于你的操作系统和个人偏好。以下是几种常见的方法：

### 方法一：使用 `pyenv` 管理多个 Python 版本

`pyenv` 是一个非常流行的工具，可以让你在同一台机器上轻松切换不同的 Python 版本。

#### 安装 `pyenv`

- **Windows**: 使用 [`pyenv-win`](https://github.com/pyenv-win/pyenv-win)。
  - 可以通过 PowerShell 或者命令提示符运行以下命令进行安装：
    ```powershell
    Invoke-WebRequest -UseBasicParsing -Uri "https://raw.githubusercontent.com/pyenv-win/pyenv-win/master/pyenv-win/install-pyenv-win.ps1" -OutFile "./install-pyenv-win.ps1"; &"./install-pyenv-win.ps1"
    ```
- **macOS/Linux**: 使用 Homebrew（macOS）或直接从源码安装。
  - macOS:
    ```bash
    brew update
    brew install pyenv
    ```
  - Linux:
    ```bash
    curl https://pyenv.run | bash
    ```

#### 安装特定版本的 Python

```
# 列出所有可用的 Python 版本
pyenv install --list

# 安装你需要的版本，例如 Python 3.9.7
pyenv install 3.9.7
```

#### 设置全局或局部 Python 版本

- **全局设置**（影响整个系统）：
  ```bash
  pyenv global 3.9.7
  ```
- **局部设置**（仅对当前目录及其子目录有效）：
  ```bash
  pyenv local 3.9.7
  ```

### 方法二：手动下载并安装不同版本的 Python

你可以直接从 [Python 官方网站](https://www.python.org/downloads/) 下载你想要的 Python 版本的安装包，然后像安装普通软件一样进行安装。

- **Windows**: 在安装过程中确保勾选“Add Python to PATH”选项。
- **macOS/Linux**: 可能需要手动解压和配置环境变量。

### 方法三：使用 Anaconda/Miniconda 管理环境

Anaconda 和 Miniconda 提供了方便的方式来管理不同的 Python 版本和依赖库。

1. 下载并安装 [Anaconda](https://www.anaconda.com/products/distribution) 或 [Miniconda](https://docs.conda.io/en/latest/miniconda.html)。
2. 创建一个新的环境并指定 Python 版本：
   ```bash
   conda create --name myenv python=3.9
   ```
3. 激活新创建的环境：
   ```bash
   conda activate myenv
   ```

### 总结

根据你的需求选择最适合的方法。如果你需要频繁切换 Python 版本或者维护多个项目，推荐使用 `pyenv` 或者 Anaconda/Miniconda。对于只需要临时更改 Python 版本的情况，直接下载安装所需版本也是一种简单快捷的方法。请根据自己的实际情况选择合适的方法进行操作。






你已经成功安装了 `pyenv-win`，接下来需要按照以下步骤操作：

---

### **1. 验证安装是否成功**
在 PowerShell 中运行以下命令，检查 `pyenv` 是否可用：
```powershell
pyenv --version
```
如果看到类似 `pyenv 2.4.3` 的输出，说明安装成功。

---

### **2. 安装特定版本的 Python**
使用 `pyenv` 安装你需要的 Python 版本（例如 `3.9.7` 或 `3.10.13`）：
```powershell
pyenv install 3.9.7
```
- **等待下载和编译**：这个过程可能需要几分钟，取决于你的网络速度和电脑性能。
- **查看可用版本**：如果不确定具体版本号，可以先运行：
  ```powershell
  pyenv install --list
  ```

---

### **3. 设置全局或局部 Python 版本**
#### **全局设置**（所有项目使用同一版本）：
```powershell
pyenv global 3.9.7
```

#### **局部设置**（仅当前目录及其子目录生效）：
```powershell
pyenv local 3.9.7
```
> 这会在当前目录生成一个 `.python-version` 文件，记录使用的 Python 版本。

---

### **4. 验证 Python 版本**
运行以下命令确认当前 Python 版本：
```powershell
python --version
```
输出应为 `Python 3.9.7`（或其他你安装的版本）。

---

### **5. 创建虚拟环境（可选但推荐）**
建议为项目创建独立的虚拟环境，避免依赖冲突：
```powershell
python -m venv myenv
```
激活虚拟环境：
```powershell
.\myenv\Scripts\Activate.ps1
```
激活后，`python` 和 `pip` 会使用虚拟环境中的版本。

---

### **6. 安装依赖库**
在虚拟环境中安装你需要的库（例如 `gensim==3.8.3` 和 `scipy==1.2.1`）：
```powershell
pip install gensim==3.8.3 scipy==1.2.1
```

---

### **常见问题排查**
#### **1. 命令未生效？**
- **重启 PowerShell**：某些更改需要重启终端才能生效。
- **检查环境变量**：确保 `pyenv` 的路径已添加到系统 `PATH` 中。

#### **2. 安装 Python 版本失败？**
- **网络问题**：尝试使用国内镜像加速下载：
  ```powershell
  pyenv install -v 3.9.7 --mirror https://npm.taobao.org/mirrors/python
  ```
- **权限问题**：以管理员身份运行 PowerShell。

#### **3. 需要卸载旧版本？**
不需要手动卸载旧版本 Python，`pyenv` 会管理多个版本共存。

---

### **总结**
你现在可以：
1. 使用 `pyenv` 管理多个 Python 版本。
2. 切换到兼容 `gensim==3.8.3` 的 Python 3.9/3.10。
3. 在虚拟环境中安全地安装依赖。

如果仍有问题，请告诉我具体错误信息，我会进一步协助！ 😊







如果你想删除之前通过 `pyenv` 安装的所有 Python 版本，可以按照以下步骤操作：

1. **查看已安装的 Python 版本**：  
    打开终端，输入以下命令查看当前已安装的 Python 版本：
    
    ```bash
    pyenv versions
    ```
    
2. **删除所有版本**：  
    可以逐一删除每个 Python 版本，使用以下命令：
    
    ```bash
    pyenv uninstall <version>
    ```
    
    例如，如果你想删除 Python 3.8.5 和 3.9.1，你可以运行：
    
    ```bash
    pyenv uninstall 3.8.5
    pyenv uninstall 3.9.1
    ```
    
    如果你想删除所有安装的版本，可以手动执行删除命令，或者使用以下命令删除已安装的所有版本：
    
    ```bash
    pyenv versions --bare | xargs -I {} pyenv uninstall -f {}
    ```
    
    这条命令会列出所有安装的版本（不包含版本号的标记），并使用 `xargs` 将每个版本传递给 `pyenv uninstall` 命令进行删除。
    
3. **确认删除**：  
    删除完后，再次使用 `pyenv versions` 查看，确保没有残留的版本。
    

这样，你就可以清理掉通过 `pyenv` 安装的所有 Python 版本了。




遇到下载超时的问题通常是由于网络连接不稳定或者 Python 镜像站点的响应时间过长。你提到你已经开了代理，但仍然出现超时错误。这里有几个解决方法可以尝试：

### 1. **确认代理设置**：

如果你是通过代理来访问网络，确保你的代理设置正确。在 Windows 上，`pyenv-win` 会自动使用你系统的代理配置，但你可以显式地设置代理来确保连接稳定。

你可以在终端中设置代理环境变量：

```bash
set http_proxy=http://your_proxy_address:port
set https_proxy=https://your_proxy_address:port
```

替换 `your_proxy_address:port` 为你的代理地址和端口。

### 2. **使用国内镜像源**：

如果你在中国大陆，使用默认的 Python 镜像站点时可能会遇到速度慢或者超时的问题。你可以使用国内的镜像源来下载 Python 版本。例如，可以使用阿里云的镜像源。

以下是如何配置 `pyenv` 使用国内镜像：

你可以通过设置环境变量来更改下载源：

```bash
set PYTHON_BUILD_MIRROR=https://mirrors.aliyun.com/python/
```

然后重新运行 `pyenv install`：

```bash
pyenv install 3.9.7
```

### 3. **手动下载并安装**：

如果代理设置正确、镜像源也不行，你也可以尝试手动下载 Python 安装包并安装。

- 访问 [Python 3.9.7 下载页面](https://www.python.org/ftp/python/3.9.7/)。
    
- 下载适合你操作系统的安装包（Windows 版本通常是 `.exe` 文件）。
    
- 手动安装 Python 3.9.7 后，使用 `pyenv` 注册这个版本：
    

```bash
pyenv rehash
```

然后通过以下命令查看安装的版本：

```bash
pyenv versions
```

### 4. **提高超时时间**：

你还可以尝试增加 `pyenv` 下载操作的超时时间（如果可能的话），以避免下载超时。虽然 `pyenv` 默认没有提供直接修改超时时间的选项，但通过设置较低的并发下载速度可能有助于稳定下载。

希望这些方法能够帮助你解决下载超时的问题。如果问题仍然存在，可能需要进一步检查代理配置或网络连接情况。