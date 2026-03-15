---
title: uv Quick Start
draft: "false"
tags:
  - uv
  - python
created: 2026-03-15 21:01
modified: 2026-03-15 21:01
---
uv 是一个用 **Rust 编写的 Python 包管理器和环境管理工具**, 速度比 pip 快很多, 并且可以替代：

- pip
- pip-tools
- virtualenv
- pyenv（部分功能）

一句话总结：

> uv = pip + venv + python installer（超级快版本）

---

# 1 安装 uv

Windows / Linux / macOS：

```bash
curl -Ls https://astral.sh/uv/install.sh | sh
```

Windows PowerShell：

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

验证：

```bash
uv --version
```

---

# 2 安装 Python

uv 可以直接管理 Python 版本：

```bash
uv python install 3.11
```

查看已安装版本：

```bash
uv python list
```

---

# 3 创建项目

创建新项目：

```bash
uv init my_project
```

目录结构：

```text
my_project/
 ├─ pyproject.toml
 ├─ README.md
 └─ src/
```

进入项目：

```bash
cd my_project
```

如果已经在 my_project/ 下, 则直接

```shell
uv init
```

---

# 4 创建虚拟环境

```bash
uv venv
```

会生成：

```text
.venv/
```

激活环境：

Windows：

```bash
.venv\Scripts\activate
```

Linux / macOS：

```bash
source .venv/bin/activate
```

---

# 5 安装依赖

安装包：

```bash
uv pip install fastapi
```

安装多个：

```bash
uv pip install numpy pandas matplotlib
```

---

# 6 从 requirements.txt 安装

```bash
uv pip install -r requirements.txt
```

---

# 7 导出依赖

生成 requirements：

```bash
uv pip freeze > requirements.txt
```

---

# 8 运行 Python

可以直接运行：

```bash
uv run python main.py
```

或：

```bash
uv run pytest
```

不需要手动 activate。

---

# 9 VSCode 配置

VSCode 会自动识别：

```
.venv
```

如果没有识别：

```
Ctrl + Shift + P
Python: Select Interpreter
```

选择：

```
.venv/bin/python
```

或：

```
.venv\Scripts\python.exe
```

---

# 推荐项目结构

```text
project/
 ├─ .venv/
 ├─ src/
 │   └─ app.py
 ├─ tests/
 ├─ pyproject.toml
 └─ README.md
```

---

# uv 常用命令

|功能|命令|
|---|---|
|安装 Python|`uv python install 3.11`|
|创建 venv|`uv venv`|
|安装包|`uv pip install pkg`|
|运行脚本|`uv run python script.py`|
|安装 requirements|`uv pip install -r requirements.txt`|

---

# uv 的优势

相比传统工具：

|工具|uv 替代|
|---|---|
|pip|uv pip|
|virtualenv|uv venv|
|pyenv|uv python|
|pip-tools|uv pip compile|

性能：

- 安装速度 **10~100x faster**
- 并行下载
- 全局缓存

---

# 推荐搭配工具

现代 Python 开发环境：

```
uv
ruff
pytest
pyright
```

VSCode 插件：

- Python
- Ruff
- Pylance

---

