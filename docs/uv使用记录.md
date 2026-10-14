# uv 使用记录

## uv 简介

`uv` 是 Rust 编写的高性能 Python 包管理工具，可替代 `pip`、`pip-tools`、`venv`、`poetry` 等工具。核心命令包括 `uv init`、`uv add`、`uv run` 等。

---

## Python 版本管理

uv 内置 Python 版本管理功能，无需手动下载安装。

### 查看当前 Python 版本

```bash
# 查看项目虚拟环境使用的 Python 版本
uv run python --version

# 查看系统默认 Python 版本
python --version

# 查找当前项目实际使用的 Python 解释器路径
uv python find

# 列出所有已安装的 Python 版本
uv python list
```

### 安装指定版本的 Python

```bash
# 安装 Python 3.12（自动安装最新的 3.12.x）
uv python install 3.12

# 安装精确版本
uv python install 3.12.5

# 同时安装多个版本
uv python install 3.11 3.12 3.13

# 安装最新的稳定版
uv python install
```

### 切换项目使用的 Python 版本

```bash
# 将项目固定到指定版本（更新 .python-version 文件）
uv python pin 3.12

# 用指定版本重新创建虚拟环境
uv venv --python 3.12

# 或在初始化时指定版本
uv init --python 3.12
```

> **说明**：`uv python pin` 会修改项目根目录的 `.python-version` 文件，之后 `uv run` 等命令会自动使用该版本。

---

## uv init

初始化一个新的 Python 项目，生成 `pyproject.toml`、`README.md`、`.python-version` 等文件，并自动创建虚拟环境（`.venv`）。

### 基本用法

```bash
# 在当前目录初始化项目
uv init

# 在指定目录初始化项目
uv init my-project

# 创建一个应用项目（默认）
uv init --app

# 创建一个库项目
uv init --lib

# 指定 Python 版本
uv init --python 3.12
```

### 说明

- 执行后会生成 `pyproject.toml`，其中包含项目元数据（名称、版本、作者、依赖等）。
- 自动创建 `.venv` 虚拟环境，后续命令默认使用该环境。
- 可通过 `--name`、`--description`、`--readme` 等参数自定义生成的内容。
- 如果当前目录已存在项目文件，`uv init` 会提示或跳过。

### 应用项目 vs 库项目

`uv init` 默认创建**应用项目**（`--app`），使用 `--lib` 可创建**库项目**。两者的核心区别：

| 对比项 | 应用项目 (`--app`，默认) | 库项目 (`--lib`) |
|--------|--------------------------|------------------|
| **用途** | Web 服务器、脚本、命令行界面（CLI）等可直接运行的程序 | 提供函数/对象供其他项目导入使用，目的是构建分发（如上传 PyPI） |
| **CLI 入口点** | 包含 `[project.scripts]`，定义命令行入口（如 `example-app = "example_app:main"`） | 无 CLI 入口点 |
| **额外文件** | 无特殊标记文件 | 包含 `py.typed` 标记文件，告知消费方该库支持类型注解 |
| **`__init__.py` 内容** | 定义 `main()` 函数（作为 CLI 入口） | 定义 `hello()` 等 API 函数（供其他项目 `import`） |
| **运行方式** | `uv run example-app`（直接执行 CLI 命令） | `uv run python -c "import example_lib; example_lib.hello()"` |

#### 核心差异总结

1. **定位不同**：应用是「可执行程序」，库是「被导入的代码包」
2. **CLI 入口点**：只有应用项目会生成 `[project.scripts]`，可以直接通过命令名运行
3. **`py.typed` 标记**：只有库项目会生成该文件，表明库是类型安全的，供下游消费者读取类型提示
4. **发布目标**：库项目通常会 `uv build` 后发布到 PyPI，应用项目则通常部署运行

#### 如何判断当前项目类型？

查看 `pyproject.toml` 中是否包含 `[project.scripts]` 节：
- 包含 → 应用项目
- 不包含，且有 `py.typed` 文件 → 库项目

> 本项目包含 `[project.scripts]`（`langchain-mika = "langchain_mika:main"`），属于**应用项目**。

---

## uv add

向项目添加依赖包，自动更新 `pyproject.toml` 中的 `dependencies` 并安装到虚拟环境。

### 基本用法

```bash
# 添加一个包
uv add langchain

# 添加多个包
uv add langchain langchain-openai langchain-community

# 添加开发依赖（写入 [dependency-groups]）
uv add --dev pytest ruff
```

### 版本约束

```bash
# 精确版本（等于）
uv add "langchain==0.3.0"

# 最低版本（大于等于）
uv add "langchain>=0.3.0"

# 兼容版本（语义化版本，>=0.3.0,<0.4.0）
uv add "langchain~=0.3.0"

# 区间版本
uv add "langchain>=0.3.0,<0.4.0"
```

> **Windows PowerShell 注意**：包含特殊字符（如 `>=`、`~=`、`==`）的版本约束需要用双引号包裹，避免被 shell 解析。

### 其他选项

| 选项 | 说明 |
|------|------|
| `--dev` | 作为开发依赖添加 |
| `--optional <group>` | 添加到指定的可选依赖组 |
| `--editable` | 以可编辑模式安装本地包 |
| `--python <version>` | 指定 Python 版本 |
| `--no-sync` | 只更新 `pyproject.toml`，不安装到环境 |

### 查看依赖当前版本

```bash
# 方式一：用 uv 列出指定包信息（推荐）
uv pip show langchain

# 方式二：用 Python 直接打印版本（需包支持 __version__）
uv run python -c "import langchain; print(langchain.__version__)"

# 方式三：列出所有已安装的依赖及其版本
uv pip list

# 方式四：查看 pyproject.toml 中声明的版本约束
grep -i langchain pyproject.toml
```

> **说明**：
> - `uv pip show <pkg>` 输出包含 Version、Location、Requires 等详细信息，最全面。
> - `uv run python -c` 直接读取包内的 `__version__` 属性，最接近运行时实际版本。
> - `uv pip list` 会列出所有已安装包，适合排查依赖冲突。
> - 用 `uv pip show` 或 `uv pip list` 时，会显示虚拟环境中实际安装的精确版本号。

---

## 常用 uv 命令速查

| 命令 | 说明 |
|------|------|
| `uv init` | 初始化项目 |
| `uv add <pkg>` | 添加依赖 |
| `uv remove <pkg>` | 移除依赖 |
| `uv sync` | 同步依赖到虚拟环境 |
| `uv run <cmd>` | 在虚拟环境中运行命令 |
| `uv lock` | 更新 `uv.lock` 锁文件 |
| `uv venv` | 手动创建虚拟环境 |
| `uv python list` | 列出已安装的 Python 版本 |
| `uv python install <ver>` | 安装指定版本的 Python |
| `uv python pin <ver>` | 固定项目使用的 Python 版本 |
| `uv python find` | 查找当前项目使用的 Python 解释器 |
| `uv pip install <pkg>` | 类似 pip 的安装命令（不修改 pyproject.toml） |
