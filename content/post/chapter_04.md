---
title: 第四章：Claude Agent SDK 深度解剖
date: 2026-01-04 00:00:00
categories: Agents
tags: [AI, Agent, Python, Claude, SDK]
---

# 第四章：Claude Agent SDK 深度解剖

> "框架的价值不在于它帮你做了什么，而在于它帮你挡住了什么。"

## 4.1 为什么需要专用 SDK？

在第一章，我们看到了直接调用 `anthropic.Anthropic()` 的代价：你要自己处理速率限制、自己管理工具权限、自己实现重试逻辑。当系统演化为多智能体时，这些问题成倍叠加。

Auto-Claude 的解决方案是统一的工厂函数：`create_client()`。

这个函数只有一个调用点——但它是整个系统最重要的 50 行代码。理解它，就理解了 Auto-Claude 的安全模型、性能策略和多智能体协调机制。

```python
# apps/backend/core/client.py

def create_client(
    phase: str,
    task_id: Optional[str] = None,
    worktree_path: Optional[str] = None,
    project_path: Optional[str] = None,
    subtask_index: Optional[int] = None,
    is_parallel: bool = False,
    spec_name: Optional[str] = None,
    spec_path: Optional[str] = None,
) -> ClaudeSDKClient:
    """
    工厂函数：构建一个完全配置的 SDK 客户端。
    所有 AI 交互必须经过此函数，绝不直接实例化 SDK。
    """
```

注意签名：没有 `api_key`，没有 `model`，没有 `temperature`。这些全是系统级决策，由 `create_client()` 根据上下文自动解析。调用者只需告诉它"我是哪个阶段、在哪个工作区工作"。

`★ Insight ─────────────────────────────────────`

**工厂模式的工程价值**：所有调用点共享同一配置逻辑。当你需要修改安全策略时，改一处即全局生效；当你需要 A/B 测试不同模型时，只需修改工厂，所有智能体无感切换。这是"开放封闭原则"在 AI 系统中的体现：对扩展开放，对修改封闭。

`─────────────────────────────────────────────────`

## 4.2 SDK 客户端的五层配置

`create_client()` 的内部逻辑可以分解为五个层次，每层解决一类问题：

```
┌─────────────────────────────────────────┐
│  Layer 5: MCP 服务器注册                 │  工具扩展
├─────────────────────────────────────────┤
│  Layer 4: 工具权限 + 文件系统许可        │  安全边界
├─────────────────────────────────────────┤
│  Layer 3: 模型 + 思考预算               │  性能配置
├─────────────────────────────────────────┤
│  Layer 2: 工作区路径解析                │  环境感知
├─────────────────────────────────────────┤
│  Layer 1: 项目数据缓存（线程安全）       │  效率基础
└─────────────────────────────────────────┘
```

让我们逐层深入。

## 4.3 第一层：线程安全的项目数据缓存

多智能体系统中，多个智能体可能同时调用 `create_client()`。如果每次调用都重新读取项目配置文件，既慢又有竞争条件的风险。

Auto-Claude 使用**双重检查锁定（Double-Checked Locking）**模式解决这个问题：

```python
# apps/backend/core/client.py（简化）

_project_cache: dict[str, Any] = {}
_cache_lock = threading.Lock()

def _get_cached_project_data(project_path: str) -> dict[str, Any]:
    """
    双重检查锁定：在不牺牲并发性能的前提下保证线程安全。
    """
    # 第一次检查：无锁快速路径
    if project_path in _project_cache:
        return _project_cache[project_path]

    # 获取锁
    with _cache_lock:
        # 第二次检查：防止另一个线程在我们等待锁时已经写入
        if project_path in _project_cache:
            return _project_cache[project_path]

        # 只有我们是第一个到达的线程，才执行昂贵的 I/O
        data = _load_project_data(project_path)
        _project_cache[project_path] = data
        return data
```

为什么需要两次检查？

想象两个智能体线程 A 和 B 同时发现缓存为空，同时尝试获取锁。线程 A 先拿到锁，写入缓存，释放锁。如果没有第二次检查，线程 B 拿到锁后会**再次执行昂贵的文件读取**，浪费资源；更严重的是，如果写入操作不幂等，可能导致数据不一致。

第二次检查确保：即使多个线程竞争，只有第一个线程执行实际工作，其余线程直接使用已缓存的结果。

```
时间轴：
线程A: 检查(空) → 等锁 → 拿锁 → 检查(空) → 读文件 → 写缓存 → 释放锁
线程B:            检查(空) → 等锁 →         → 拿锁 → 检查(有!) → 返回缓存 → 释放锁
线程C:                                                   检查(有!) → 直接返回（无锁）
```

`★ Insight ─────────────────────────────────────`

**并发编程中的"检查-执行"原子性**：Python 的 GIL（全局解释器锁）并不能保证复合操作的原子性。`if key in dict: dict[key] = value` 在 Python 里不是原子的——GIL 可能在 `if` 和赋值之间切换线程。双重检查锁定通过显式锁将"检查+写入"变成原子操作，这是多线程编程中处理"懒加载"的经典模式。

`─────────────────────────────────────────────────`

## 4.4 第二层：工作区路径解析

每个任务运行在独立的 git worktree 中。智能体需要知道自己的工作根目录是哪里，这影响：

- 文件系统许可范围
- 工具可访问的路径
- 相对路径的基准点

```python
# 路径解析逻辑（简化）

def _resolve_working_directory(
    worktree_path: Optional[str],
    project_path: Optional[str],
) -> str:
    """
    优先级：worktree_path > project_path > 当前目录
    """
    if worktree_path and os.path.exists(worktree_path):
        return worktree_path
    if project_path and os.path.exists(project_path):
        return project_path
    return os.getcwd()
```

这个优先级顺序体现了**最小权限原则**：在 worktree 中工作的智能体，其工作目录是 worktree，而不是整个项目根目录。这限制了智能体能直接访问的文件范围。

## 4.5 第三层：模型选择与思考预算

不同阶段需要不同的推理深度。Auto-Claude 根据 `phase` 参数动态选择模型和思考预算：

```python
# apps/backend/core/phase_config.py（概念化展示）

PHASE_CONFIGS = {
    "spec_critic": {
        "model": "claude-opus-4-6",       # 最强推理
        "thinking_budget": 31999,          # 最大思考空间
        "extended_thinking": True,
    },
    "planner": {
        "model": "claude-sonnet-4-6",     # 平衡性能
        "thinking_budget": 10000,
        "extended_thinking": True,
    },
    "coder": {
        "model": "claude-sonnet-4-6",     # 最佳编码模型
        "thinking_budget": 5000,
        "extended_thinking": False,        # 编码不需要深度推理
    },
    "qa_reviewer": {
        "model": "claude-sonnet-4-6",
        "thinking_budget": 8000,
        "extended_thinking": True,         # 评审需要仔细推理
    },
}
```

思考预算（Thinking Budget）是 Claude 的扩展思考功能：在生成最终回复前，模型有一段"内部推理"空间，对用户不可见。预算越大，推理越深，成本也越高。

对于 Spec Critic 这样需要批判性评估的角色，31999 个 token 的思考空间允许它进行真正深入的分析；对于 Coder 这样以执行为主的角色，关闭扩展思考反而更高效。

## 4.6 第四层：工具权限与文件系统许可

这是 `create_client()` 最复杂的部分，也是最关键的安全层。

### 工具许可列表

每个智能体只能使用其角色所需的工具子集：

```python
# 不同阶段的工具许可（简化）

PHASE_TOOL_ALLOWLISTS = {
    "planner": [
        "Read", "Write", "Glob", "Grep",
        "WebSearch", "WebFetch",
        # Planner 不需要 Bash——它只规划，不执行
    ],
    "coder": [
        "Read", "Write", "Edit", "Glob", "Grep",
        "Bash",           # Coder 需要运行命令
        "WebSearch",
        "TodoWrite",      # 进度跟踪
    ],
    "qa_reviewer": [
        "Read", "Glob", "Grep",
        "Bash",           # 需要运行测试
        "WebFetch",       # 可能需要查阅文档
        # QA 不能写入文件——只读评估
    ],
    "qa_fixer": [
        "Read", "Write", "Edit", "Glob", "Grep",
        "Bash",
    ],
}
```

注意 `qa_reviewer` 没有写入权限。这不是偶然的：QA 审查者的职责是**评估**，不是修改。如果它能写文件，就可能在评审过程中无意间改变被测试的代码，破坏评估结果的可信度。

### 文件系统许可边界

Bash 工具允许智能体运行任意 shell 命令，但文件系统访问通过许可列表限制：

```python
def _build_filesystem_permissions(
    worktree_path: str,
    project_path: str,
    spec_path: Optional[str],
) -> list[dict]:
    """
    构建文件系统访问许可列表。
    """
    allowed_paths = [
        worktree_path,           # 主工作目录
        "/tmp",                  # 临时文件
    ]

    # spec 目录：智能体可以读取需求规格
    if spec_path and os.path.exists(spec_path):
        allowed_paths.append(spec_path)

    # 项目根目录只读：智能体可以查阅项目配置，但不能修改
    if project_path != worktree_path:
        allowed_paths.append(f"{project_path}:read-only")

    return [{"type": "filesystem", "paths": allowed_paths}]
```

这个设计实现了**最小权限原则（Principle of Least Privilege）**：

- 智能体可以完全控制自己的 worktree
- 可以读取规格文档（需要理解任务）
- 可以读取项目配置（需要了解上下文）
- 不能修改其他 worktree（避免并行智能体互相干扰）
- 不能修改主分支（主分支安全）

### Worktree 许可扩展

并行 Coder 智能体有时需要读取其他 worktree 的输出（例如查看共享接口定义）。`create_client()` 会检测是否有其他活跃的 worktree，并动态扩展许可：

```python
def _expand_worktree_permissions(
    base_permissions: list,
    is_parallel: bool,
    project_path: str,
) -> list:
    """
    并行模式下，允许读取其他 worktree（只读）。
    """
    if not is_parallel:
        return base_permissions

    # 查找所有活跃的 worktree
    worktrees = _list_active_worktrees(project_path)
    for wt in worktrees:
        # 以只读方式添加其他 worktree
        base_permissions.append({
            "path": wt.path,
            "access": "read-only"
        })

    return base_permissions
```

`★ Insight ─────────────────────────────────────`

**动态权限 vs 静态权限**：安全系统的常见错误是"一刀切"的静态权限。Auto-Claude 的设计展示了动态权限的价值：权限根据运行时上下文（是否并行、是否有 worktree）动态计算。这增加了复杂度，但实现了更精确的安全边界——权限刚好够用，不多也不少。

`─────────────────────────────────────────────────`

## 4.7 第五层：MCP 服务器注册

MCP（Model Context Protocol）是 Anthropic 设计的标准协议，允许外部"工具服务器"向 Claude 暴露工具。Auto-Claude 使用多个 MCP 服务器扩展智能体能力：

```python
# MCP 服务器配置（简化）

def _build_mcp_servers(
    phase: str,
    project_path: str,
    settings: dict,
) -> list[MCPServerConfig]:
    """
    根据阶段和项目配置，注册合适的 MCP 服务器。
    """
    servers = []

    # Context7：文档和库参考（所有阶段都可用）
    servers.append(MCPServerConfig(
        name="context7",
        command="npx",
        args=["-y", "@upstash/context7-mcp"],
    ))

    # Graphiti：记忆知识图谱（仅在启用时）
    if settings.get("graphiti_enabled"):
        servers.append(MCPServerConfig(
            name="graphiti",
            command="python",
            args=["-m", "graphiti_mcp"],
            env={"NEO4J_URI": settings["neo4j_uri"]},
        ))

    # Electron MCP：GUI 测试（仅 QA 阶段）
    if phase in ("qa_reviewer", "qa_fixer"):
        if settings.get("electron_mcp_enabled"):
            servers.append(MCPServerConfig(
                name="electron",
                command="node",
                args=["electron-mcp-server.js"],
            ))

    # Puppeteer：Web 测试（仅 QA 阶段）
    if phase in ("qa_reviewer", "qa_fixer"):
        servers.append(MCPServerConfig(
            name="puppeteer",
            command="npx",
            args=["-y", "@anthropic-ai/mcp-server-puppeteer"],
        ))

    # Linear：项目管理集成（仅在配置时）
    if settings.get("linear_enabled"):
        servers.append(MCPServerConfig(
            name="linear",
            command="npx",
            args=["-y", "linear-mcp-server"],
            env={"LINEAR_API_KEY": settings["linear_api_key"]},
        ))

    return servers
```

注意 Electron MCP 和 Puppeteer 只在 QA 阶段注册。这不仅是性能优化（减少不必要的进程），也是安全考虑：Coder 没有理由操控 GUI，这个能力只属于 QA 角色。

### MCP 协议的工作原理

```
Claude SDK
    │
    │  MCP 协议（stdio / HTTP）
    ▼
MCP 服务器进程
    │
    │  工具调用
    ▼
外部系统（Context7、Neo4j、Electron、浏览器...）
```

每个 MCP 服务器是一个独立进程，通过标准输入输出与 SDK 通信。SDK 自动处理进程启动、协议握手、工具发现，并将 MCP 工具与内置工具统一暴露给 Claude。

## 4.8 神来之笔：Monkey-Patching SDK

Auto-Claude 做了一件非常规的事：**修改了 SDK 的内部解析器**。

```python
# apps/backend/core/client.py

def _patch_sdk_message_parser():
    """
    补丁：处理 SDK 未知消息类型，避免解析崩溃。

    Claude 有时会返回实验性的消息类型，旧版 SDK 会抛出
    ValueError: Unknown message type。这个补丁让解析器
    优雅降级，而不是崩溃。
    """
    from claude_agent_sdk import _internal

    original_parse = _internal.MessageParser.parse

    def patched_parse(self, data: dict) -> Any:
        try:
            return original_parse(self, data)
        except ValueError as e:
            if "Unknown message type" in str(e):
                # 返回通用消息对象，而不是崩溃
                logger.warning(f"SDK parse warning: {e}. Skipping unknown message.")
                return None
            raise  # 其他 ValueError 正常抛出

    _internal.MessageParser.parse = patched_parse
```

为什么需要这样做？

Claude API 是活跃演进的产品，有时会返回实验性的消息类型。如果你的 SDK 版本比 API 更旧，SDK 就不认识这些新消息类型，并抛出 `ValueError`——导致整个智能体会话崩溃。

这个补丁实现了**优雅降级**：遇到未知消息类型时，记录警告并跳过，而不是崩溃。对于长达数小时的智能体任务，一个未知消息类型不应该让一切前功尽弃。

`★ Insight ─────────────────────────────────────`

**Monkey-patching 的工程权衡**：这是一把双刃剑。优点：在不等待 SDK 更新的情况下快速修复生产问题；缺点：如果 SDK 内部重构了 `_internal` 模块的结构，补丁会静默失效或崩溃。好的 monkey-patch 应该有明确的注释说明原因、被替换的行为、以及何时可以移除。

`─────────────────────────────────────────────────`

## 4.9 Windows 的特殊处理

跨平台 AI 应用需要处理平台特有的限制。Windows 有一个鲜为人知的问题：**命令行长度限制**。

```python
# apps/backend/core/client.py

WINDOWS_CMD_MAX_LENGTH = 32768  # Windows CreateProcess 限制

def _sanitize_for_windows(options: ClaudeAgentOptions) -> ClaudeAgentOptions:
    """
    Windows 专用：确保命令行参数不超过系统限制。

    MCP 服务器配置、系统提示、工具描述都通过命令行传递。
    超过 32,768 字符会导致进程启动失败。
    """
    if not sys.platform.startswith("win"):
        return options  # 非 Windows 不需要处理

    # 计算总命令行长度
    total_length = _estimate_cmdline_length(options)

    if total_length > WINDOWS_CMD_MAX_LENGTH:
        logger.warning(
            f"Command line length {total_length} exceeds Windows limit "
            f"{WINDOWS_CMD_MAX_LENGTH}. Truncating system prompt."
        )
        # 策略：优先保留工具配置，截断系统提示末尾
        options = _truncate_system_prompt(options, WINDOWS_CMD_MAX_LENGTH - 1000)

    return options
```

这个问题在 Unix 系统上根本不存在（Linux 的命令行长度限制通常是 2MB），所以很容易被遗忘。但 Windows 用户如果使用了大量 MCP 服务器或复杂提示，就会遇到神秘的"进程无法启动"错误。

Auto-Claude 的解决方案是：测量、检测、优雅降级——截断系统提示而不是崩溃，并记录清晰的警告。

## 4.10 认证：OAuth 与 API Key 的统一抽象

Auto-Claude 支持两种认证模式：

1. **Claude Code OAuth**：使用 Claude Code 订阅的 OAuth token（无需 API Key）
2. **API Profile**：直接使用 Anthropic API Key，支持自定义 endpoint

```python
# apps/backend/core/auth.py（概念化）

class AuthResolver:
    """
    统一认证接口：调用者不需要知道是 OAuth 还是 API Key。
    """

    def resolve(self, profile_name: Optional[str] = None) -> AuthCredentials:
        if profile_name:
            return self._resolve_profile(profile_name)
        else:
            return self._resolve_oauth()

    def _resolve_oauth(self) -> AuthCredentials:
        """从 OS 密钥链读取 OAuth token，必要时自动刷新。"""
        token = keychain.get("claude-code-oauth-token")
        if self._is_expired(token):
            token = self._refresh_token(token)
        return AuthCredentials(type="oauth", token=token)

    def _resolve_profile(self, name: str) -> AuthCredentials:
        """从加密存储读取 API profile。"""
        profile = encrypted_store.get(f"profile:{name}")
        return AuthCredentials(
            type="api_key",
            api_key=profile.api_key,
            base_url=profile.endpoint,  # 支持 z.ai 等兼容端点
        )
```

### 多账户轮换

当主账户触发速率限制时，Auto-Claude 自动切换到备用账户：

```python
# 多账户管理（简化）

class ProfileScorer:
    """根据使用量和可用性为账户评分，选择最佳账户。"""

    def select_best_profile(self) -> ProfileCredentials:
        profiles = self._load_all_profiles()
        scored = [(self._score(p), p) for p in profiles]
        scored.sort(reverse=True)  # 分数高 = 使用量低 = 优先选择

        best_score, best_profile = scored[0]
        if best_score < MINIMUM_VIABLE_SCORE:
            raise RateLimitError("All profiles exhausted. Please wait.")

        return best_profile

    def _score(self, profile: ProfileCredentials) -> float:
        usage_penalty = profile.recent_usage_tokens / profile.daily_limit
        availability_bonus = 1.0 if not profile.is_rate_limited else 0.0
        return availability_bonus - usage_penalty
```

这个机制让 Auto-Claude 在企业环境中具有更高的吞吐量：多个开发者共享账户池，系统自动负载均衡。

## 4.11 完整的 create_client() 调用流程

将所有层次组合在一起，一次 `create_client()` 调用的完整流程如下：

```
create_client(phase="coder", task_id="001", worktree_path="/worktrees/001")
       │
       ├─1─ _patch_sdk_message_parser()          ← 只执行一次（单例）
       │
       ├─2─ _get_cached_project_data(project)    ← 线程安全缓存
       │         └─ 返回 project settings
       │
       ├─3─ _resolve_working_directory()          ← worktree > project > cwd
       │         └─ "/worktrees/001"
       │
       ├─4─ _resolve_auth()                       ← OAuth 或 API Key
       │         └─ AuthCredentials
       │
       ├─5─ phase_config.get("coder")             ← 模型 + 思考预算
       │         └─ {model: sonnet-4-6, budget: 5000}
       │
       ├─6─ _build_tool_allowlist("coder")        ← 工具许可
       │         └─ [Read, Write, Edit, Bash, ...]
       │
       ├─7─ _build_filesystem_permissions()       ← 文件系统边界
       │         └─ [worktree(rw), project(ro), /tmp(rw)]
       │
       ├─8─ _build_mcp_servers("coder", ...)      ← MCP 注册
       │         └─ [context7, graphiti?]
       │
       ├─9─ _sanitize_for_windows()               ← Windows 适配
       │
       └─10─ ClaudeSDKClient(options)             ← 构建客户端
                 └─ 返回完全配置的客户端
```

调用者拿到的 `ClaudeSDKClient` 是一个完全配置好的、安全边界明确的、平台适配过的 AI 客户端。调用者只需关心：用这个客户端做什么。

## 4.12 使用客户端：上下文管理器模式

`ClaudeSDKClient` 设计为上下文管理器，强制正确的资源管理：

```python
# 正确用法
async def run_coder_agent(task: Task, worktree_path: str):
    client = create_client(
        phase="coder",
        task_id=task.id,
        worktree_path=worktree_path,
    )

    async with client as session:
        result = await session.run(
            prompt=build_coder_prompt(task),
            system=load_prompt("coder.md"),
        )

    # 退出 async with 时，SDK 自动：
    # - 关闭 MCP 服务器进程
    # - 释放网络连接
    # - 清理临时资源
    return result
```

`async with client` 保证了即使智能体崩溃，MCP 服务器进程也能被正确终止。如果使用 `client.run()` 而不是 `async with`，MCP 进程可能成为僵尸进程，在并行场景下尤其危险。

## 4.13 事件系统：从后端到前端的实时通信

智能体运行时，前端需要实时显示进度。Auto-Claude 使用基于 stdout 的**结构化事件协议**：

```python
# apps/backend/core/phase_event.py

import json
import sys

class ExecutionPhase(Enum):
    PLANNING = "planning"
    CODING = "coding"
    QA_REVIEW = "qa_review"
    QA_FIXING = "qa_fixing"
    COMPLETE = "complete"
    FAILED = "failed"
    RATE_LIMIT_PAUSED = "rate_limit_paused"
    AUTH_FAILURE_PAUSED = "auth_failure_paused"

def emit_phase(
    phase: ExecutionPhase,
    task_id: str,
    metadata: Optional[dict] = None,
):
    """
    向 stdout 写入结构化阶段事件。
    Electron 主进程监听此输出并更新 UI。
    """
    event = {
        "type": "__EXEC_PHASE__",
        "phase": phase.value,
        "task_id": task_id,
        "timestamp": time.time(),
        "metadata": metadata or {},
    }
    # 使用特殊前缀确保可解析
    print(f"__EXEC_PHASE__:{json.dumps(event)}", flush=True)
```

前端（Electron 主进程）监听子进程的 stdout，识别 `__EXEC_PHASE__:` 前缀并解析 JSON：

```typescript
// apps/frontend/src/main/agent/agent-process.ts（概念化）

const agentProcess = spawn("python", ["run.py", "--spec", specId]);

agentProcess.stdout.on("data", (data: Buffer) => {
    const lines = data.toString().split("\n");
    for (const line of lines) {
        if (line.startsWith("__EXEC_PHASE__:")) {
            const event = JSON.parse(line.slice("__EXEC_PHASE__:".length));
            handlePhaseEvent(event);  // 更新 Kanban 状态
        } else {
            // 普通日志输出显示在终端
            terminalStore.appendOutput(line);
        }
    }
});
```

这个设计将**进程通信**和**日志输出**混合在同一个 stdout 流中，但通过特殊前缀区分。好处是不需要额外的 IPC 机制；代价是前端必须过滤普通日志。

`★ Insight ─────────────────────────────────────`

**stdout 作为消息总线**：这是 Unix 哲学的体现——进程间通信最简单的方式就是管道和标准流。与 WebSocket、消息队列相比，stdout 事件协议零依赖、零配置，在开发阶段也很容易调试（直接运行 Python 脚本即可看到所有事件）。代价是结构不够严格，需要约定前缀格式。

`─────────────────────────────────────────────────`

## 4.14 错误分类与恢复策略

当智能体调用失败时，`create_client()` 构建的客户端会分类错误并触发相应的恢复策略：

```python
# 错误分类体系

class AgentError(Exception):
    pass

class RateLimitError(AgentError):
    """触发：暂停 + 切换账户 + 重试"""
    recovery = "switch_account_and_retry"

class AuthFailureError(AgentError):
    """触发：暂停 + 通知用户 + 等待干预"""
    recovery = "pause_and_notify"

class ContextWindowError(AgentError):
    """触发：压缩上下文 + 重试"""
    recovery = "compact_context_and_retry"

class ToolError(AgentError):
    """触发：记录错误 + 继续（让智能体自我纠正）"""
    recovery = "log_and_continue"

class FatalError(AgentError):
    """触发：标记任务失败 + 人工干预"""
    recovery = "escalate_to_human"
```

每种错误对应不同的 `ExecutionPhase` 发射事件，前端据此更新 UI 状态（例如显示"等待速率限制解除"的计时器）。

## 4.15 小结

`create_client()` 是一个微型操作系统的内核：

| 问题 | 解决方案 |
|------|---------|
| 多线程重复加载配置 | 双重检查锁定缓存 |
| 智能体权限过大 | 工具 + 文件系统许可列表 |
| 角色能力不一致 | 按阶段动态注册 MCP 服务器 |
| SDK 版本兼容性 | Monkey-patch 优雅降级 |
| Windows 命令行限制 | 长度检测 + 自动截断 |
| 认证方式多样 | 统一 AuthResolver 抽象 |
| 前端实时更新 | stdout 结构化事件协议 |
| API 错误处理 | 分类错误 + 差异化恢复 |

理解了 `create_client()`，就理解了 Auto-Claude 的安全哲学：**不信任任何单个组件，让架构强制执行正确行为**。智能体不需要"遵守规则"，因为架构根本不给它违规的机会。

---

## Lab 4.1：构建你自己的智能体客户端工厂

在这个实验中，你将实现一个简化版的 `create_client()` 工厂，深入理解其每个设计决策的必要性。

### 背景

你正在构建一个双智能体系统：一个 **Researcher**（只读，负责调研）和一个 **Writer**（可写，负责生成内容）。你需要确保 Researcher 不能意外写入文件，而 Writer 的写入范围被限制在指定目录内。

### Part A：工具许可系统

```python
# lab_04_client_factory.py

from dataclasses import dataclass, field
from enum import Enum
from typing import Optional
import os

class AgentRole(Enum):
    RESEARCHER = "researcher"
    WRITER = "writer"
    CRITIC = "critic"

@dataclass
class ToolPermissions:
    allowed_tools: list[str]
    allowed_write_paths: list[str]  # 空列表 = 禁止写入
    allowed_read_paths: list[str]

@dataclass
class AgentConfig:
    role: AgentRole
    model: str
    permissions: ToolPermissions
    thinking_budget: int
    working_dir: str

# TODO Part A: 实现工具许可配置
# 为每个角色定义合适的工具集和路径许可
# 要求：
# - RESEARCHER：只能读文件，不能写
# - WRITER：能读写，但只能写入 output_dir
# - CRITIC：只能读，可以向 reviews/ 子目录写评审意见

ROLE_PERMISSIONS: dict[AgentRole, callable] = {
    # TODO: 填写每个角色的许可配置
    # 提示：使用工厂函数，接受 working_dir 参数
    # 示例：
    # AgentRole.RESEARCHER: lambda working_dir: ToolPermissions(
    #     allowed_tools=[...],
    #     allowed_write_paths=[],
    #     allowed_read_paths=[working_dir, "/tmp"],
    # ),
}


def create_agent_config(
    role: AgentRole,
    working_dir: str,
    output_dir: Optional[str] = None,
) -> AgentConfig:
    """
    工厂函数：根据角色创建完整配置。
    """
    if not os.path.exists(working_dir):
        raise ValueError(f"Working directory does not exist: {working_dir}")

    # TODO Part A: 完成工厂函数
    # 1. 从 ROLE_PERMISSIONS 获取该角色的许可
    # 2. 如果提供了 output_dir，动态添加到 WRITER 的写入许可中
    # 3. 根据角色选择合适的模型（researcher 和 critic 用 haiku，writer 用 sonnet）
    # 4. 根据角色设置思考预算（critic 需要最多的思考空间）
    pass
```

### Part B：线程安全缓存

```python
import threading
import time

class ConfigCache:
    """
    线程安全的配置缓存，使用双重检查锁定。
    """

    def __init__(self):
        self._cache: dict[str, AgentConfig] = {}
        self._lock = threading.Lock()
        self._load_count = 0  # 追踪实际加载次数（用于验证）

    def get_or_load(self, cache_key: str, loader: callable) -> AgentConfig:
        """
        TODO Part B: 实现双重检查锁定

        要求：
        1. 先在无锁状态下检查缓存（快速路径）
        2. 如果缓存未命中，获取锁
        3. 在持有锁时再次检查（防止竞争条件）
        4. 如果确实需要加载，调用 loader() 并存入缓存
        5. 递增 self._load_count

        注意：loader() 是一个耗时操作（模拟文件读取），
        在并发场景下应该只被调用一次。
        """
        pass

    def get_load_count(self) -> int:
        return self._load_count


def test_concurrent_caching():
    """验证：10 个并发线程只触发 1 次实际加载。"""
    cache = ConfigCache()

    def slow_loader():
        """模拟耗时的配置加载"""
        time.sleep(0.1)  # 模拟 I/O
        return {"model": "claude-sonnet-4-6", "budget": 5000}

    results = []
    errors = []

    def worker():
        try:
            config = cache.get_or_load("test-project", slow_loader)
            results.append(config)
        except Exception as e:
            errors.append(e)

    threads = [threading.Thread(target=worker) for _ in range(10)]
    for t in threads:
        t.start()
    for t in threads:
        t.join()

    assert len(errors) == 0, f"Errors: {errors}"
    assert len(results) == 10, "All threads should get a result"
    assert cache.get_load_count() == 1, (
        f"Expected 1 load, got {cache.get_load_count()}. "
        "Race condition detected!"
    )
    print(f"✓ 10 concurrent threads, {cache.get_load_count()} actual load. "
          f"Cache is thread-safe.")
```

### Part C：整合与验证

```python
def verify_permission_isolation():
    """
    验证：不同角色的权限确实被隔离。
    """
    import tempfile

    with tempfile.TemporaryDirectory() as tmp_dir:
        output_dir = os.path.join(tmp_dir, "output")
        os.makedirs(output_dir)

        researcher_config = create_agent_config(
            role=AgentRole.RESEARCHER,
            working_dir=tmp_dir,
        )

        writer_config = create_agent_config(
            role=AgentRole.WRITER,
            working_dir=tmp_dir,
            output_dir=output_dir,
        )

        critic_config = create_agent_config(
            role=AgentRole.CRITIC,
            working_dir=tmp_dir,
        )

        # 验证 Researcher 没有写入权限
        assert len(researcher_config.permissions.allowed_write_paths) == 0, \
            "Researcher should have NO write permissions"

        # 验证 Writer 的写入范围被限制
        assert output_dir in writer_config.permissions.allowed_write_paths, \
            "Writer should be able to write to output_dir"
        assert tmp_dir not in writer_config.permissions.allowed_write_paths or \
               output_dir in writer_config.permissions.allowed_write_paths, \
            "Writer's write scope should be limited"

        # 验证 Critic 使用最高思考预算
        assert critic_config.thinking_budget >= writer_config.thinking_budget, \
            "Critic should have more thinking budget than Writer"

        print("✓ Permission isolation verified")
        print(f"  Researcher write paths: {researcher_config.permissions.allowed_write_paths}")
        print(f"  Writer write paths: {writer_config.permissions.allowed_write_paths}")
        print(f"  Critic thinking budget: {critic_config.thinking_budget}")


if __name__ == "__main__":
    test_concurrent_caching()
    verify_permission_isolation()
    print("\n✓ All tests passed!")
```

### 验证标准

| 检查点 | 预期结果 |
|--------|---------|
| 并发缓存测试 | 10 线程只触发 1 次 loader() |
| Researcher 写权限 | `allowed_write_paths == []` |
| Writer 写范围 | 只包含 `output_dir`，不包含整个工作目录 |
| Critic 思考预算 | 大于等于 Writer 的预算 |
| 无效工作目录 | 抛出 `ValueError` |

### 进阶挑战

1. **MCP 注册**：扩展 `AgentConfig` 支持 MCP 服务器列表。Researcher 注册 `context7`（文档查询），Writer 不注册任何 MCP，Critic 注册一个模拟的代码审查 MCP。

2. **动态权限扩展**：实现 `expand_permissions_for_parallel(config, other_configs)` 函数——在并行场景下，让所有智能体能以只读方式读取彼此的工作目录。

3. **事件系统**：在 `create_agent_config()` 中添加事件发射，记录每次配置创建。实现一个监听器，统计各角色的配置创建次数，并以表格形式打印报告。

---

**本章要点回顾**

- `create_client()` 是 Auto-Claude 的"内核"，封装了安全、性能、跨平台等所有系统级关切
- 双重检查锁定解决多线程环境下的缓存竞争问题
- 工具许可和文件系统许可共同构成最小权限安全边界
- MCP 服务器按阶段动态注册，避免不必要的权限扩展
- Monkey-patching 是应对 SDK 兼容性问题的务实工具，需谨慎使用
- stdout 结构化事件是进程间实时通信的最简方案
