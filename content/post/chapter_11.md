---
title: 第十一章：项目分析与动态配置
date: 2026-01-11 00:00:00
categories: Agents
tags: [AI, Agent, Python, Claude, SDK]
---

# 第十一章：项目分析与动态配置

> "一个优秀的智能体系统不应该对每个项目都使用相同的配置。它应该像一位经验丰富的工程师：进入新项目前先观察环境，然后根据所见调整自己的行为方式。"

---

## 11.1 动态配置的必要性

考虑 Auto-Claude 面临的一个根本挑战：它需要运行在任意项目上——可能是 Python FastAPI 后端、React 前端、Go 微服务、Rust 系统工具，或者一个包含所有这些的 Monorepo。

如果使用固定的安全配置，有两种糟糕的结果：
- **配置太严格**：禁止了项目实际需要的命令（如 `npm run test` 对于前端项目），导致智能体无法正常工作
- **配置太宽松**：允许了项目不应该执行的危险命令（如 `dropdb` 在没有数据库的项目中毫无意义），增大攻击面

理想方案是：**在运行时分析项目，生成专属于这个项目的最小必要命令集**。这正是 `ProjectAnalyzer` 要做的事。

---

## 11.2 `SecurityProfile`：一切分析的产出

`apps/backend/project/models.py` 中的 `SecurityProfile` 是分析系统的核心数据结构：

```python
# apps/backend/project/models.py（简化）

@dataclass
class SecurityProfile:
    """项目的完整安全配置。"""

    # 四层命令集合（集合求并集 = 完整白名单）
    base_commands: set[str]    # 所有项目通用的基础命令（cat, ls, echo...）
    stack_commands: set[str]   # 根据技术栈添加（npm, pytest, cargo...）
    script_commands: set[str]  # 项目自定义脚本（package.json scripts, Makefile targets）
    custom_commands: set[str]  # 手动添加的额外命令

    # 检测到的技术信息
    detected_stack: TechnologyStack  # 语言、框架、数据库等
    custom_scripts: CustomScripts    # npm scripts, make targets 等

    # 元数据
    project_hash: str    # 关键配置文件的哈希，用于变更检测
    inherited_from: str  # Worktree 继承：指向父项目路径

    def get_all_allowed_commands(self) -> set[str]:
        """返回完整白名单（四层并集）。"""
        return (self.base_commands | self.stack_commands
                | self.script_commands | self.custom_commands)
```

`TechnologyStack` 则是分析结果的容器：

```python
@dataclass
class TechnologyStack:
    languages: list[str]           # ["python", "typescript"]
    package_managers: list[str]    # ["npm", "pip"]
    frameworks: list[str]          # ["react", "fastapi"]
    databases: list[str]           # ["postgres", "redis"]
    infrastructure: list[str]      # ["docker"]
    cloud_providers: list[str]     # ["aws"]
    code_quality_tools: list[str]  # ["ruff", "eslint"]
    version_managers: list[str]    # ["nvm", "pyenv"]
```

---

## 11.3 技术栈检测：文件证据链

`StackDetector` 通过检查文件的存在性来推断技术栈——这是一种"从证据推断事实"的侦探式逻辑：

```python
# apps/backend/project/stack_detector.py（简化）

class StackDetector:
    def detect_languages(self) -> None:
        """通过文件特征检测使用的编程语言。"""
        # Python：存在 *.py 文件，或存在 pyproject.toml、requirements.txt
        if self.parser.file_exists(
            "*.py", "**/*.py", "pyproject.toml",
            "requirements.txt", "setup.py", "Pipfile"
        ):
            self.stack.languages.append("python")

        # TypeScript：存在 *.ts/*.tsx 文件，或存在 tsconfig.json
        if self.parser.file_exists("*.ts", "*.tsx", "**/*.ts", "**/*.tsx", "tsconfig.json"):
            self.stack.languages.append("typescript")

        # Rust：存在 Cargo.toml 或 *.rs 文件
        if self.parser.file_exists("Cargo.toml", "*.rs", "**/*.rs"):
            self.stack.languages.append("rust")

        # Go：存在 go.mod 或 *.go 文件
        if self.parser.file_exists("go.mod", "*.go", "**/*.go"):
            self.stack.languages.append("go")

        # 共支持 15+ 种语言...
```

注意这里使用的是**多重证据**：`TypeScript` 的判定检查 `*.ts`、`*.tsx` 和 `tsconfig.json` 三种证据之一。这比单独检查文件扩展名更可靠——一个纯配置项目可能只有 `tsconfig.json` 而没有 `.ts` 文件，但它仍然是 TypeScript 项目。

框架检测则更进一步，通过解析 `package.json` 的依赖项：

```python
# apps/backend/project/framework_detector.py（概念化）

class FrameworkDetector:
    def detect_all(self) -> list[str]:
        frameworks = []
        # 读取 package.json 的 dependencies 和 devDependencies
        pkg_deps = self.parser.get_package_json_deps()

        if "react" in pkg_deps:
            frameworks.append("react")
        if "next" in pkg_deps or "next.js" in pkg_deps:
            frameworks.append("nextjs")
        if "@angular/core" in pkg_deps:
            frameworks.append("angular")
        if "vue" in pkg_deps:
            frameworks.append("vue")

        # 读取 pyproject.toml 的依赖
        py_deps = self.parser.get_python_deps()
        if "fastapi" in py_deps:
            frameworks.append("fastapi")
        if "django" in py_deps:
            frameworks.append("django")
        if "flask" in py_deps:
            frameworks.append("flask")

        return frameworks
```

`★ Insight ─────────────────────────────────────`
证据链越长，结论越可靠。检测到 `Cargo.toml` 说明是 Rust 项目，但还需确认目录中实际有 `.rs` 文件，才能排除"空壳项目"。这种**多重证据验证**（Multiple Evidence Corroboration）在分析不可控外部数据时极为重要。
`─────────────────────────────────────────────────`

---

## 11.4 命令注册表：技术到命令的映射

检测到技术栈之后，`_build_stack_commands()` 将其转化为允许的命令集合：

```python
# apps/backend/project/analyzer.py（简化）

def _build_stack_commands(self) -> None:
    """根据检测到的技术栈构建允许命令集合。"""
    stack = self.profile.detected_stack
    commands = self.profile.stack_commands   # 要填充的集合

    # 语言命令
    for lang in stack.languages:
        if lang in LANGUAGE_COMMANDS:
            commands.update(LANGUAGE_COMMANDS[lang])
            # 例如：python → {"python", "python3", "pip", "pip3", "pytest", "uv", ...}

    # 包管理器命令
    for pm in stack.package_managers:
        if pm in PACKAGE_MANAGER_COMMANDS:
            commands.update(PACKAGE_MANAGER_COMMANDS[pm])
            # 例如：npm → {"npm", "npx", "node", ...}

    # 框架命令
    for fw in stack.frameworks:
        if fw in FRAMEWORK_COMMANDS:
            commands.update(FRAMEWORK_COMMANDS[fw])

    # 数据库命令
    for db in stack.databases:
        if db in DATABASE_COMMANDS:
            commands.update(DATABASE_COMMANDS[db])

    # 基础设施、云提供商、代码质量工具、版本管理工具...
```

命令注册表本身（`command_registry/`）是一系列精心维护的映射字典。以语言命令为例：

```python
# apps/backend/project/command_registry/languages.py（概念化）

LANGUAGE_COMMANDS = {
    "python": {"python", "python3", "pip", "pip3", "pytest",
               "uv", "uvicorn", "alembic", "mypy", "ruff"},
    "typescript": {"tsc", "ts-node", "tsx"},
    "rust": {"cargo", "rustc", "rustup", "rust-analyzer"},
    "go": {"go", "gofmt", "golint", "govulncheck"},
    # ...
}
```

这种数据驱动的设计使添加新技术支持只需要添加一个字典条目，无需修改主逻辑。

---

## 11.5 哈希增量检测：避免重复分析

项目分析是一个相对耗时的操作（需要递归扫描目录、解析配置文件）。如果每次启动都完整分析，会明显拖慢工作流。`ProjectAnalyzer` 使用哈希增量检测来避免不必要的重新分析：

```python
# apps/backend/project/analyzer.py（简化）

def compute_project_hash(self) -> str:
    """
    计算关键项目文件的哈希，用于检测变更。

    哈希的输入：各配置文件的修改时间和大小
    （而非文件内容——避免读取大文件的 I/O 开销）
    """
    hash_files = [
        # JavaScript/TypeScript
        "package.json", "package-lock.json", "yarn.lock",
        # Python
        "pyproject.toml", "requirements.txt", "Pipfile",
        # Rust
        "Cargo.toml", "Cargo.lock",
        # Go
        "go.mod", "go.sum",
        # 基础设施
        "Dockerfile", "docker-compose.yml",
        # ...共 20+ 种配置文件
    ]

    hasher = hashlib.md5(usedforsecurity=False)
    for filename in hash_files:
        filepath = self.project_dir / filename
        if filepath.exists():
            stat = filepath.stat()
            # 使用文件名 + 修改时间 + 大小作为哈希输入
            # 比读取文件内容快几个数量级
            hasher.update(f"{filename}:{stat.st_mtime}:{stat.st_size}".encode())

    return hasher.hexdigest()


def should_reanalyze(self, profile: SecurityProfile) -> bool:
    """检查项目是否自上次分析以来发生了变化。"""
    # 继承的配置（来自 Worktree 的父项目）永不重新分析
    if profile.inherited_from:
        # 但需要验证继承链的合法性
        parent = Path(profile.inherited_from)
        if (parent.exists() and parent.is_dir()
                and self._is_descendant_of(self.project_dir, parent)
                and (parent / self.PROFILE_FILENAME).exists()):
            return False  # 合法继承，跳过重新分析

    # 比较当前哈希与存储的哈希
    current_hash = self.compute_project_hash()
    return current_hash != profile.project_hash


def analyze(self, force: bool = False) -> SecurityProfile:
    """执行项目分析，支持缓存。"""
    existing = self.load_profile()
    if existing and not force and not self.should_reanalyze(existing):
        print(f"Using cached security profile (hash: {existing.project_hash[:8]})")
        return existing  # 直接使用缓存，跳过分析

    print("Analyzing project structure for security profile...")
    # ... 执行完整分析
```

这里有两个值得关注的细节：

1. **哈希的输入是文件元数据（mtime + size），而非文件内容**。这意味着即使文件内容没有实质性变化，只要修改时间变了（如 `touch package.json`），也会触发重新分析。这是故意的——保守比激进更安全：宁可多分析一次，也不要错过真正的变更。

2. **使用 `hashlib.md5(usedforsecurity=False)`**。参数 `usedforsecurity=False` 告诉 Python 这个 MD5 不用于加密/安全目的，仅用于内容比较。某些 FIPS 合规系统（如某些企业 Linux）会禁止 MD5 用于安全用途，这个参数绕过了该限制。

---

## 11.6 Worktree 配置继承

Auto-Claude 在第五章介绍的 Worktree 系统中，每个 Worktree 是父项目的一个分支副本。对这些 Worktree 项目进行重新分析是浪费的——它们的技术栈与父项目完全相同。

`SecurityProfile.inherited_from` 字段实现了配置继承：

```python
# apps/backend/project/analyzer.py（简化）

def should_reanalyze(self, profile: SecurityProfile) -> bool:
    if profile.inherited_from:
        parent = Path(profile.inherited_from)

        # 继承验证（三个条件全部满足才接受）：
        # 1. 父路径必须存在且是目录
        # 2. 当前项目必须是父项目的子孙（防止任意路径注入）
        # 3. 父目录必须有有效的安全配置文件
        if (parent.exists()
                and parent.is_dir()
                and self._is_descendant_of(self.project_dir, parent)
                and (parent / self.PROFILE_FILENAME).exists()):
            return False  # 跳过重新分析，使用继承配置

    # ...
```

继承路径的安全验证是关键：`self._is_descendant_of(child, parent)` 检查 `child` 必须是 `parent` 的子孙路径，防止配置文件中的 `inherited_from` 字段指向任意系统路径（可能是 Worktree 配置文件被篡改的攻击向量）。

```python
def _is_descendant_of(self, child: Path, parent: Path) -> bool:
    """检查 child 是否是 parent 的子孙路径。"""
    try:
        child.resolve().relative_to(parent.resolve())
        return True
    except ValueError:  # relative_to() 失败表示不是子孙
        return False
```

`★ Insight ─────────────────────────────────────`
**继承的同时必须验证来源**。允许配置文件指定"从哪里继承"是方便的，但也可能是注入点——恶意构造的 `inherited_from` 路径可能绕过安全限制。`_is_descendant_of()` 强制要求继承路径必须是当前项目的"祖先"，形成单向树状信任链，有效防止这种攻击。
`─────────────────────────────────────────────────`

---

## 11.7 从分析到安全策略：完整数据流

将本章内容串联起来，描绘从"陌生项目"到"定制化安全策略"的完整数据流：

```
ProjectAnalyzer.analyze()
    │
    ├── 1. load_profile() → 检查是否有缓存
    │   └── 若缓存有效且哈希未变 → 直接返回 ✓
    │
    ├── 2. StackDetector.detect_all()
    │   ├── detect_languages()  → Python, TypeScript, Go...
    │   ├── detect_package_managers() → npm, pip, cargo...
    │   ├── detect_databases()  → postgres, redis...
    │   ├── detect_infrastructure() → docker...
    │   └── detect_cloud_providers() → aws, gcp...
    │
    ├── 3. FrameworkDetector.detect_all()
    │   └── 解析 package.json/pyproject.toml 依赖
    │       → react, fastapi, django...
    │
    ├── 4. StructureAnalyzer.analyze()
    │   ├── 解析 package.json scripts → npm run test, npm run dev
    │   ├── 解析 Makefile targets → make build, make test
    │   └── 读取 .auto-claude/allowed_commands.json（自定义白名单）
    │
    ├── 5. _build_stack_commands()
    │   └── TechnologyStack → LANGUAGE_COMMANDS[lang]
    │                       | PACKAGE_MANAGER_COMMANDS[pm]
    │                       | FRAMEWORK_COMMANDS[fw]
    │                       | DATABASE_COMMANDS[db]
    │                       → SecurityProfile.stack_commands
    │
    └── 6. save_profile() → 持久化 .auto-claude-security.json
        └── 返回 SecurityProfile
            ↓
            被 security/hooks.py 使用
            ↓
            PreToolUse 钩子：bash_security_hook()
                └── 检查命令是否在 profile.get_all_allowed_commands() 中
```

---

## 11.8 配置文件的位置策略

`get_profile_path()` 展示了一个简洁的存储位置策略：

```python
def get_profile_path(self) -> Path:
    if self.spec_dir:
        # 有 spec 目录时（任务执行阶段），存在 spec 目录中
        return self.spec_dir / self.PROFILE_FILENAME
    # 否则存在项目根目录
    return self.project_dir / self.PROFILE_FILENAME
```

这意味着每个任务可以有自己的安全配置，与项目级配置相互独立。任务完成后，spec 目录连同其配置一起归档，不会影响其他任务。

---

## 本章要点回顾

| 概念 | 实现位置 | 关键点 |
|------|---------|-------|
| SecurityProfile 四层命令集 | `project/models.py` | base + stack + script + custom |
| 文件证据链检测 | `project/stack_detector.py` | 多重文件特征推断语言 |
| 依赖解析框架检测 | `project/framework_detector.py` | 解析 package.json 依赖 |
| 命令注册表 | `project/command_registry/` | 技术→命令集的映射字典 |
| mtime+size 哈希 | `analyzer.py: compute_project_hash()` | 快速变更检测，不读取文件内容 |
| 继承链验证 | `analyzer.py: _is_descendant_of()` | 防止任意路径注入 |
| Worktree 继承 | `SecurityProfile.inherited_from` | 子项目复用父项目配置 |

---

## Lab 11.1：迷你项目分析器

本 Lab 实现一个简化的项目分析器，能够检测技术栈并生成安全配置。

### Part A：技术栈检测器（必做）

```python
# lab_11_project_analyzer.py
from __future__ import annotations

import hashlib
import json
from dataclasses import dataclass, field
from pathlib import Path


@dataclass
class TechnologyStack:
    languages: list[str] = field(default_factory=list)
    package_managers: list[str] = field(default_factory=list)
    frameworks: list[str] = field(default_factory=list)
    databases: list[str] = field(default_factory=list)


@dataclass
class SecurityProfile:
    allowed_commands: set[str] = field(default_factory=set)
    detected_stack: TechnologyStack = field(default_factory=TechnologyStack)
    project_hash: str = ""

    def to_dict(self) -> dict:
        return {
            "allowed_commands": sorted(self.allowed_commands),
            "detected_stack": {
                "languages": self.detected_stack.languages,
                "package_managers": self.detected_stack.package_managers,
                "frameworks": self.detected_stack.frameworks,
                "databases": self.detected_stack.databases,
            },
            "project_hash": self.project_hash,
        }


# 基础命令（所有项目通用）
BASE_COMMANDS = {
    "ls", "cat", "echo", "pwd", "cd", "mkdir", "touch",
    "cp", "mv", "find", "grep", "head", "tail", "wc",
    "git", "curl", "wget",
}

# 技术到命令的映射
COMMAND_REGISTRY = {
    # 语言
    "python": {"python", "python3", "pip", "pip3", "pytest", "uv", "ruff", "mypy"},
    "javascript": {"node", "npm", "npx"},
    "typescript": {"tsc", "ts-node", "tsx"},
    "go": {"go", "gofmt"},
    "rust": {"cargo", "rustc"},
    # 包管理器
    "yarn": {"yarn"},
    "pnpm": {"pnpm"},
    # 框架
    "react": {"react-scripts"},
    "django": {"django-admin", "manage.py"},
    "fastapi": {"uvicorn"},
    "nextjs": {"next"},
    # 数据库
    "postgres": {"psql", "pg_dump", "pg_restore"},
    "redis": {"redis-cli"},
    "sqlite": {"sqlite3"},
}


def detect_stack(project_dir: Path) -> TechnologyStack:
    """
    通过检查文件存在性检测项目技术栈。

    TODO: 实现检测逻辑（30-50 行）：

    语言检测（多重证据）：
    - Python: 任一存在 *.py 文件 / pyproject.toml / requirements.txt / setup.py
    - JavaScript: package.json 或 *.js 文件
    - TypeScript: tsconfig.json 或 *.ts 文件
    - Go: go.mod 或 *.go 文件
    - Rust: Cargo.toml 或 *.rs 文件

    包管理器检测：
    - npm: package-lock.json
    - yarn: yarn.lock
    - pnpm: pnpm-lock.yaml

    框架检测（解析 package.json 依赖）：
    - react / nextjs / vue / svelte / angular
    - 解析 pyproject.toml 或 requirements.txt 检测 django / fastapi / flask

    数据库检测（在配置文件中搜索关键词）：
    - 在 *.env / docker-compose.yml 中搜索 postgres / redis / sqlite / mysql

    提示：
    - 使用 list(project_dir.glob("*.py")) 检查文件是否存在
    - 解析 package.json 时使用 json.loads()
    - 返回填充好的 TechnologyStack 对象
    """
    stack = TechnologyStack()
    # YOUR CODE HERE
    return stack


def build_security_profile(stack: TechnologyStack) -> SecurityProfile:
    """
    根据检测到的技术栈构建安全配置。

    TODO: 实现命令集构建逻辑（10-15 行）：

    1. 从 BASE_COMMANDS 开始
    2. 对 stack 中检测到的每种技术，查找 COMMAND_REGISTRY
    3. 将所有相关命令添加到允许集合

    注意：stack.languages + stack.package_managers + stack.frameworks + stack.databases
    都需要查找

    提示：
    allowed = set(BASE_COMMANDS)  # 不要直接修改 BASE_COMMANDS
    allowed.update(COMMAND_REGISTRY.get("python", set()))
    """
    # YOUR CODE HERE
    pass


def compute_hash(project_dir: Path) -> str:
    """
    计算项目关键配置文件的哈希（使用文件元数据，非内容）。

    TODO: 实现哈希计算逻辑（10-15 行）：

    1. 定义要检查的配置文件列表
       ["package.json", "pyproject.toml", "requirements.txt",
        "Cargo.toml", "go.mod", "Dockerfile", "docker-compose.yml"]
    2. 对每个存在的文件，将 "{filename}:{mtime}:{size}" 加入哈希
    3. 返回 MD5 十六进制字符串

    提示：
    hasher = hashlib.md5(usedforsecurity=False)
    stat = filepath.stat()
    hasher.update(f"{name}:{stat.st_mtime}:{stat.st_size}".encode())
    """
    # YOUR CODE HERE
    pass
```

### Part B：分析器主类（必做）

```python
# 接续 lab_11_project_analyzer.py

import os
import tempfile


class MiniProjectAnalyzer:
    """
    简化的项目分析器，支持缓存。
    """

    PROFILE_FILE = ".mini-security-profile.json"

    def __init__(self, project_dir: Path):
        self.project_dir = Path(project_dir).resolve()
        self._profile_path = self.project_dir / self.PROFILE_FILE

    def load_cached(self) -> SecurityProfile | None:
        """加载缓存的安全配置（如果存在）。"""
        if not self._profile_path.exists():
            return None
        try:
            data = json.loads(self._profile_path.read_text())
            profile = SecurityProfile(
                allowed_commands=set(data.get("allowed_commands", [])),
                project_hash=data.get("project_hash", ""),
            )
            stack_data = data.get("detected_stack", {})
            profile.detected_stack = TechnologyStack(
                languages=stack_data.get("languages", []),
                package_managers=stack_data.get("package_managers", []),
                frameworks=stack_data.get("frameworks", []),
                databases=stack_data.get("databases", []),
            )
            return profile
        except (json.JSONDecodeError, KeyError):
            return None

    def save(self, profile: SecurityProfile) -> None:
        """持久化安全配置。"""
        self._profile_path.write_text(
            json.dumps(profile.to_dict(), indent=2)
        )

    def analyze(self, force: bool = False) -> SecurityProfile:
        """
        分析项目并返回安全配置。支持缓存。

        TODO: 实现分析逻辑（12-18 行）：

        1. 如果 force=False，尝试加载缓存
        2. 如果缓存存在，计算当前哈希
        3. 如果哈希相同，直接返回缓存（打印"Using cached profile"）
        4. 否则，执行完整分析：
           a. detect_stack(self.project_dir)
           b. build_security_profile(stack)
           c. profile.project_hash = compute_hash(self.project_dir)
           d. self.save(profile)
           e. 打印"Analysis complete: {len(profile.allowed_commands)} commands allowed"
        5. 返回 profile

        提示：缓存命中时检查 cached.project_hash == compute_hash(self.project_dir)
        """
        # YOUR CODE HERE
        pass


# === 测试辅助：创建临时项目 ===

def create_test_project(structure: dict) -> Path:
    """
    在临时目录中创建测试项目。

    Args:
        structure: 文件名到内容的映射，例如：
                   {"package.json": '{"dependencies": {"react": "18.0.0"}}'}
    """
    tmpdir = Path(tempfile.mkdtemp())
    for filename, content in structure.items():
        filepath = tmpdir / filename
        filepath.parent.mkdir(parents=True, exist_ok=True)
        filepath.write_text(content)
    return tmpdir


# === 验证测试 ===

def test_python_project():
    """测试：Python 项目检测。"""
    project = create_test_project({
        "requirements.txt": "fastapi==0.100.0\nredis==4.6.0",
        "main.py": "from fastapi import FastAPI\napp = FastAPI()",
    })

    analyzer = MiniProjectAnalyzer(project)
    profile = analyzer.analyze()

    assert "python" in profile.detected_stack.languages, "Should detect Python"
    assert "python" in profile.allowed_commands or "python3" in profile.allowed_commands
    print(f"[PASS] Python project: {profile.detected_stack.languages}, "
          f"{len(profile.allowed_commands)} commands")
    return project


def test_nodejs_project():
    """测试：Node.js 项目检测。"""
    pkg_json = json.dumps({
        "dependencies": {"react": "18.0.0", "next": "13.0.0"},
        "devDependencies": {},
        "scripts": {"dev": "next dev", "build": "next build"}
    })
    project = create_test_project({
        "package.json": pkg_json,
        "package-lock.json": "{}",
    })

    analyzer = MiniProjectAnalyzer(project)
    profile = analyzer.analyze()

    assert "javascript" in profile.detected_stack.languages or \
           "typescript" in profile.detected_stack.languages
    assert "npm" in profile.detected_stack.package_managers
    assert "npm" in profile.allowed_commands
    print(f"[PASS] Node.js project: {profile.detected_stack.languages}, "
          f"frameworks={profile.detected_stack.frameworks}")
    return project


def test_cache_works():
    """测试：缓存机制正常工作。"""
    import time

    project = create_test_project({
        "pyproject.toml": "[tool.poetry]\nname = 'myapp'\n[tool.poetry.dependencies]\npython = '^3.11'"
    })

    analyzer = MiniProjectAnalyzer(project)

    # 第一次分析
    profile1 = analyzer.analyze()
    hash1 = profile1.project_hash

    # 第二次分析（应使用缓存）
    profile2 = analyzer.analyze()

    assert profile2.project_hash == hash1, "Hash should be the same"
    assert "python" in profile2.detected_stack.languages

    # 修改配置文件，触发重新分析
    time.sleep(0.01)  # 确保 mtime 变化
    (project / "pyproject.toml").touch()

    profile3 = analyzer.analyze()
    # 配置文件内容相同，但哈希因 mtime 变化而重新计算
    print(f"[PASS] Cache: hash1={hash1[:8]}, hash3={profile3.project_hash[:8]}")


def main():
    print("=== Part A: 技术栈检测测试 ===")
    test_python_project()
    test_nodejs_project()

    print("\n=== Part B: 缓存机制测试 ===")
    test_cache_works()

    print("\n全部测试通过！")


if __name__ == "__main__":
    main()
```

### Part C：Worktree 继承（进阶）

```python
# 接续 lab_11_project_analyzer.py

def create_worktree_profile(
    parent_dir: Path,
    child_dir: Path,
) -> SecurityProfile | None:
    """
    为 Worktree 子目录创建继承自父项目的安全配置。

    TODO: 实现继承逻辑（15-20 行）：

    1. 检查父项目是否有安全配置文件（.mini-security-profile.json）
    2. 验证 child_dir 是 parent_dir 的子孙路径
       （使用 child_dir.resolve().relative_to(parent_dir.resolve())）
    3. 如果验证通过：
       a. 加载父项目的安全配置
       b. 在子目录中创建配置文件，添加 "inherited_from" 字段
       c. 返回配置对象
    4. 如果验证失败（非子孙路径）：
       返回 None，并打印警告

    安全要求：必须验证继承路径，防止任意路径注入
    """
    # YOUR CODE HERE
    pass
```

### 验证标准

- `detect_stack()` 正确检测 Python 项目（`requirements.txt`）和 Node.js 项目（`package.json`）
- `build_security_profile()` 包含 BASE_COMMANDS + 技术专属命令
- `analyze()` 第二次调用时使用缓存（不重复分析）
- `create_worktree_profile()` 拒绝非子孙路径

### 进阶挑战

1. **框架置信度评分**：为检测到的每个框架添加置信度（如"在 dependencies 中找到" → 0.95，"只找到 tsx 文件" → 0.6），并在置信度低于阈值时输出警告
2. **增量更新**：实现 `update_profile()`，只重新检测发生变化的技术维度（如 package.json 变了就只重新检测 JS/TS 相关命令），而非完整重新分析

---

*下一章，我们转向 Auto-Claude 的后台服务与恢复编排系统——多个长时间运行的任务如何被协调、监控，以及在崩溃后如何自动恢复。*
