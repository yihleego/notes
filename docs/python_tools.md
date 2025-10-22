# Python 工具

## UV

### 一、uv 的常用用法

#### 1. 安装与初始化

```
pip install uv
```

或使用 curl 安装最新版本：

```
curl -LsSf https://astral.sh/uv/install.sh | sh
```

安装后，可以直接用：

```
uv --version
```

#### 2. 依赖管理

安装依赖

```
uv add requests
```

类似于 pip install，但会自动更新 pyproject.toml 和 uv.lock。

移除依赖

```
uv remove requests
```

更新依赖

```
uv update requests
```

同步依赖

```
uv sync

```

确保虚拟环境与 uv.lock 中的版本完全一致。

#### 3. 环境与运行

创建虚拟环境

```
uv venv
```

激活虚拟环境

```
source .venv/bin/activate
```

运行脚本（自动选择虚拟环境）

```
uv run main.py
```

临时运行依赖

```
uv run --with requests python script.py
```

（不写入依赖文件，临时使用）

### 二、uv 的高级用法

#### 1. 多 Python 版本管理

uv 内置支持管理多个 Python 版本：

```
uv python install 3.12
uv python list
uv run --python 3.12 main.py
```

类似 pyenv 的功能，但与依赖管理结合得更紧密。

#### 2. 高效缓存与并行安装

uv 基于 Rust，安装包时会并行下载和编译，速度远超 pip。

使用缓存避免重复下载：

```
uv sync --frozen
```

（严格按照 lock 文件，不会触发更新）

#### 3. 与 Docker 集成

uv 能快速生成 Docker 镜像：

```
FROM python:3.12-slim
RUN pip install uv
WORKDIR /app
COPY . .
RUN uv sync --frozen
CMD ["uv", "run", "main.py"]
```

#### 4. 跨项目运行（类似 pipx）

可以全局运行某个包的 CLI 工具：

```
uv tool install black
uvx black .
```

(uvx 类似 npx，运行时自动下载并缓存工具)

#### 5. 高级依赖解析

可选依赖组

```
[project.optional-dependencies]
dev = ["pytest", "black"]
docs = ["mkdocs"]
```

安装方式：

```
uv add -g dev
```

锁定生产环境

```
uv lock --frozen
```

只允许安装完全匹配 lock 文件的依赖。

### 三、总结

常用操作：uv add/remove/sync/run

高级特性：多 Python 版本管理、跨项目运行工具、极快依赖解析、Docker 集成

对比传统工具：比 pip + venv 更快，比 poetry 更轻量，比 pipx 更通用。

## Ruff

### 一、常用用法

#### 1. 安装

```
pip install ruff
```

或使用 uv / poetry / pipx 等工具。

#### 2. 基本检查

对某个文件或目录进行检查：

```
ruff check .
ruff check myscript.py
```

默认会输出类似 flake8 风格的警告，比如未使用的导入、多余空行等。

#### 3. 自动修复

Ruff 可以自动修复大多数问题：

```
ruff check . --fix
```

例如：

自动移除未使用的导入

自动排序导入（相当于 isort）

自动格式化部分代码风格问题

#### 4. 配置文件

在项目根目录创建 pyproject.toml：

```
[tool.ruff]
line-length = 100
target-version = "py39"
select = ["E", "F", "I"]  # 启用的规则集
ignore = ["E501"]         # 忽略某些规则
fix = true                # 自动修复
```

这样执行 ruff check . 时会按照配置运行。

#### 5. 作为代码格式化工具

Ruff 也支持类似 Black 的格式化：

```
ruff format .
```

这会统一代码风格（缩进、换行、字符串引号等）。

### 二、高级用法

#### 1. 替代 isort

Ruff 内置 I 规则集，可以自动排序导入：

```
ruff check . --select I --fix
```

无需额外安装 isort。

#### 2. Pylint/flake8 插件兼容

Ruff 模拟了 大量 flake8 插件（如 flake8-bugbear, flake8-comprehensions, flake8-docstrings 等），你可以通过配置启用：

```
[tool.ruff]
select = ["E", "F", "B", "C4", "D"]  # 包括 bugbear, comprehensions, docstrings
```

#### 3. 忽略特定文件/代码

忽略整个目录：

```
[tool.ruff]
exclude = ["migrations", "build"]
```

在代码中忽略某一行：

```
import os  # noqa: F401
```

#### 4. 与 Pre-commit 集成

在 .pre-commit-config.yaml 中：

```
repos:
- repo: https://github.com/astral-sh/ruff-pre-commit
  rev: v0.6.2
  hooks:
    - id: ruff
      args: [--fix]
    - id: ruff-format
```

这样在提交代码时会自动 lint & fix。

#### 5. 高级配置：规则选择

Ruff 的规则集非常多，可以细粒度控制：

* E：pycodestyle 错误
* F：pyflakes
* I：import 排序
* B：bugbear
* C4：列表推导优化
* D：docstring 规则
* UP：pyupgrade (自动升级到更现代语法)

例如：

```
[tool.ruff]
select = ["E", "F", "I", "B", "UP"]
ignore = ["B008"]   # 忽略某个 bugbear 规则
```

#### 6. 性能优化

Ruff 的速度非常快，但可以进一步优化：

启用缓存（默认启用，存储在 .ruff_cache/）

大项目中使用并行检查：

```
ruff check . --jobs auto
```

#### 7. 编辑器集成

Ruff 提供 LSP (Language Server Protocol)，可以与 VSCode、PyCharm、Neovim 等集成，实时提示问题并自动修复。

### 三、常见场景

1. 日常开发：`ruff check . --fix`
2. 自动化 CI：在 GitHub Actions 或 GitLab CI 中添加 Ruff 检查
3. 代码格式化：`ruff format .`
4. 导入管理：自动排序、移除未使用导入
5. 升级代码：通过 UP 规则集自动升级到 f-string、现代语法



