---
title: 第二十章：扩展与自定义
date: 2026-01-20 00:00:00
categories: Agents
tags: [AI, Agent, Python, Claude, SDK]
---

# 第二十章　扩展与自定义

> "一个好的系统不是一个封闭的黑盒，而是一个精心设计的扩展点集合，让你能在不破坏核心的情况下添加新能力。"

## 20.1　Auto-Claude 的扩展点全景

经过 19 章的学习，你已经理解了 Auto-Claude 的每一层架构。现在让我们系统梳理所有可扩展的地方：

```
扩展点地图
├── AI 模型层
│   └── 添加新的 Provider（第4章：create_client）
│
├── 安全层
│   └── 添加自定义命令验证器（第7章：VALIDATORS 注册表）
│
├── 记忆层
│   └── 添加新的 Episode 类型（第6章：Graphiti）
│
├── 工具集成层
│   └── 添加 MCP 服务器（第4章：MCP 注册）
│
├── 提示词层
│   └── 修改/替换 Agent 提示词（第3章）
│
└── 项目检测层
    └── 添加新技术栈支持（第11章：StackDetector）
```

---

## 20.2　添加新 Provider（接入不同 AI 服务）

Auto-Claude 通过 `create_client()` 工厂函数屏蔽了不同 Provider 的差异。添加新 Provider 只需修改这里：

```python
# apps/backend/core/client.py（简化）

def create_client(
    model: str,
    provider: str = "anthropic",
    **kwargs
) -> ClaudeSDKClient:
    """
    根据 provider 和 model 创建对应的 AI 客户端。

    支持的 Provider：
    - 'anthropic'：官方 Anthropic API
    - 'oauth'：Claude Code OAuth 订阅
    - 'z.ai'：z.ai 兼容端点（GLM 模型）
    - 'custom'：自定义 Anthropic 兼容端点
    """

    if provider == 'z.ai':
        # z.ai 使用 Anthropic 兼容接口
        base_url = os.environ.get('Z_AI_BASE_URL', 'https://api.z.ai/v1')
        api_key = os.environ.get('Z_AI_API_KEY')
        return ClaudeSDKClient(ClaudeAgentOptions(
            model=model,
            base_url=base_url,
            api_key=api_key,
            **kwargs
        ))

    if provider == 'custom':
        # 任何 Anthropic 兼容端点
        base_url = os.environ.get('CUSTOM_API_BASE_URL')
        api_key = os.environ.get('CUSTOM_API_KEY')
        return ClaudeSDKClient(ClaudeAgentOptions(
            model=model,
            base_url=base_url,
            api_key=api_key,
            **kwargs
        ))

    # 默认 Anthropic
    return ClaudeSDKClient(ClaudeAgentOptions(model=model, **kwargs))
```

**添加新 Provider 的步骤**：

1. 在 `.env` 文件里添加 API 端点和密钥环境变量
2. 在 `create_client()` 添加新的 `if provider == 'your-provider'` 分支
3. 在前端 Settings UI 添加 Provider 选项（`src/renderer/stores/settings-store.ts`）
4. 确保新 Provider 支持 Anthropic 的消息格式（`/v1/messages` 接口）

---

## 20.3　添加自定义安全验证器

第7章介绍了 `VALIDATORS` 注册表。添加新命令的验证器非常直接：

```python
# apps/backend/security/custom_validators.py

from security.validator import CommandValidator, ValidationResult

class DockerValidator(CommandValidator):
    """
    Docker 命令验证器。
    允许 docker build/run/ps 等，但禁止危险操作。
    """

    COMMAND = "docker"

    # 允许的子命令白名单
    ALLOWED_SUBCOMMANDS = {
        'build', 'run', 'ps', 'images', 'logs',
        'exec', 'stop', 'rm', 'rmi', 'pull',
        'compose',  # docker compose
    }

    # 绝对禁止的标志
    DANGEROUS_FLAGS = {
        '--privileged',    # 特权容器，可突破容器隔离
        '--pid=host',      # 共享宿主进程空间
        '--net=host',      # 共享宿主网络
        '-v /:/mnt',       # 挂载根文件系统（简化检测）
    }

    def validate(self, command: str, args: list[str]) -> ValidationResult:
        if not args:
            return ValidationResult(allowed=True)

        subcommand = args[0]

        # 检查子命令白名单
        if subcommand not in self.ALLOWED_SUBCOMMANDS:
            return ValidationResult(
                allowed=False,
                reason=f"docker {subcommand} 不在允许列表中"
            )

        # 检查危险标志
        full_cmd = ' '.join(args)
        for dangerous in self.DANGEROUS_FLAGS:
            if dangerous in full_cmd:
                return ValidationResult(
                    allowed=False,
                    reason=f"禁止使用危险标志: {dangerous}"
                )

        return ValidationResult(allowed=True)


# 注册到 VALIDATORS
from security.validator import VALIDATORS
VALIDATORS['docker'] = DockerValidator()
```

**注册方式**：在你的配置文件里 import 这个模块，或者通过 `SecurityProfile.custom_commands` 注入（第11章）。

---

## 20.4　扩展 Graphiti Episode 类型

第6章介绍了 Graphiti 的 7 种 Episode 类型。如果你有新的知识分类需求，可以扩展：

```python
# apps/backend/integrations/graphiti/queries_pkg/schema.py（现有代码）

class EpisodeType:
    """Episode 类型常量。"""
    CODE_CHANGE = "code_change"
    BUG_FIX = "bug_fix"
    ARCHITECTURE_DECISION = "architecture_decision"
    DEPENDENCY_UPDATE = "dependency_update"
    PERFORMANCE_ISSUE = "performance_issue"
    SECURITY_FINDING = "security_finding"
    TEST_FAILURE = "test_failure"

    # 你可以添加自定义类型
    USER_PREFERENCE = "user_preference"      # 用户工作习惯
    DOMAIN_KNOWLEDGE = "domain_knowledge"    # 业务领域知识
    TEAM_CONVENTION = "team_convention"      # 团队约定
```

```python
# 使用自定义 Episode 类型
from integrations.graphiti.queries_pkg.graphiti import GraphitiMemory
from integrations.graphiti.queries_pkg.schema import EpisodeType

memory = GraphitiMemory()

# 记录用户偏好
await memory.add_episode(
    episode_type=EpisodeType.USER_PREFERENCE,
    content="用户偏好 TypeScript 而非 JavaScript，每次提议新文件时优先选 .ts",
    source="user_feedback",
    group_id="user-prefs-project-123"
)

# 检索时，按 Episode 类型过滤
preferences = await memory.search(
    query="文件类型偏好",
    episode_types=[EpisodeType.USER_PREFERENCE],
    group_id="user-prefs-project-123"
)
```

---

## 20.5　添加 MCP 服务器

MCP（Model Context Protocol）服务器让智能体能使用外部工具。添加新 MCP 服务器只需两步：

### Step 1：实现 MCP 服务器

```python
# tools/my_mcp_server.py（MCP 服务器示例）

from mcp.server import Server
from mcp.types import Tool, TextContent

app = Server("my-custom-tools")

@app.list_tools()
async def list_tools():
    return [
        Tool(
            name="query_database",
            description="查询项目数据库，返回业务数据",
            inputSchema={
                "type": "object",
                "properties": {
                    "query": {"type": "string"},
                    "table": {"type": "string"}
                },
                "required": ["query"]
            }
        )
    ]

@app.call_tool()
async def call_tool(name: str, arguments: dict):
    if name == "query_database":
        # 实际的数据库查询逻辑
        result = my_db.execute(arguments["query"])
        return [TextContent(type="text", text=str(result))]
```

### Step 2：在 create_client() 中注册

```python
# apps/backend/core/client.py（添加 MCP 注册）

def create_client(...):
    mcp_servers = [
        MCPServer(
            command="python",
            args=["tools/my_mcp_server.py"],
            env={"DB_URL": os.environ.get("DB_URL", "")}
        )
    ]

    return ClaudeSDKClient(ClaudeAgentOptions(
        model=model,
        mcp_servers=mcp_servers,
        ...
    ))
```

注册后，智能体就可以在工具调用中使用 `query_database` 工具，就像使用内置的 `Read`/`Write` 一样。

---

## 20.6　自定义提示词工程

所有 Agent 提示词都在 `apps/backend/prompts/` 目录，每个是独立的 Markdown 文件（第3章）。

修改提示词的最佳实践：

### 1. 增量修改，不是全量替换

```markdown
<!-- apps/backend/prompts/coder.md（在原有基础上添加） -->

<!-- 在 PHASE 1 之后插入你的团队约定 -->
## TEAM CONVENTIONS (ALWAYS FOLLOW)
- 所有 Python 函数必须有 type hints
- 新文件必须添加 docstring
- 提交消息格式：{type}: {description}

<!-- 原有的 PHASE 2 继续... -->
```

### 2. 使用模板变量（动态组装）

第3章介绍的 `PromptGenerator` 支持根据项目特征动态生成提示词：

```python
# apps/backend/prompts_pkg/prompt_generator.py

def generate_coder_prompt(context: ProjectContext) -> str:
    base_prompt = load_prompt_template('coder.md')

    # 根据技术栈动态注入规范
    if 'typescript' in context.languages:
        base_prompt += TYPESCRIPT_CONVENTIONS
    if 'react' in context.frameworks:
        base_prompt += REACT_CONVENTIONS

    return base_prompt
```

### 3. A/B 测试提示词效果

```python
# 用环境变量控制使用哪个版本的提示词
PROMPT_VERSION = os.environ.get('PROMPT_VERSION', 'v1')

def load_coder_prompt() -> str:
    prompt_file = f"prompts/coder_{PROMPT_VERSION}.md"
    return Path(prompt_file).read_text()
```

---

## 20.7　添加新技术栈支持

第11章的 `StackDetector` 通过文件证据链检测技术栈。添加新技术栈：

```python
# apps/backend/project/stack_detector.py（扩展现有检测器）

class StackDetector:
    def detect_languages(self) -> set[str]:
        languages = set()

        # 现有检测...

        # 添加 Rust 检测
        if self._has_file('Cargo.toml'):
            languages.add('rust')

        # 添加 Kotlin 检测（Android 项目）
        if self._has_pattern('**/*.kt') and self._has_file('build.gradle.kts'):
            languages.add('kotlin')

        return languages

    def detect_frameworks(self) -> set[str]:
        frameworks = set()

        # 添加 Tauri 框架检测（Rust + Web 桌面应用）
        if 'rust' in self.detect_languages():
            if self._has_file('tauri.conf.json'):
                frameworks.add('tauri')

        return frameworks
```

对应地，在 `command_registry.py` 中添加新技术栈的允许命令：

```python
# apps/backend/project/command_registry.py

LANGUAGE_COMMANDS = {
    # 现有条目...
    'rust': ['cargo', 'rustc', 'rustup', 'clippy'],
    'kotlin': ['kotlin', 'kotlinc', 'gradle', 'gradlew'],
}

FRAMEWORK_COMMANDS = {
    # 现有条目...
    'tauri': ['cargo-tauri', 'tauri'],
}
```

---

## 20.8　综合实践：实现一个 Jira 集成插件

把本章所有扩展点综合起来，实现一个 Jira 集成：

**目标**：让 Auto-Claude 能从 Jira ticket 自动创建任务，完成后更新 ticket 状态。

### 组件清单

| 组件 | 扩展点 | 主要工作 |
|------|--------|---------|
| Jira API 客户端 | MCP 服务器 | 实现 `get_ticket`/`update_ticket` 工具 |
| Ticket 转规格 | 提示词扩展 | 修改 Gatherer 提示词理解 Jira 格式 |
| 完成回调 | 主流程钩子 | 任务完成后调用 Jira API 更新状态 |
| Jira 知识 | Episode 类型 | `JIRA_CONTEXT` 记录 ticket 背景 |

### 实现骨架

```python
# integrations/jira/mcp_server.py

from mcp.server import Server

app = Server("jira-tools")

@app.list_tools()
async def list_tools():
    return [
        Tool(name="get_ticket", description="获取 Jira ticket 详情"),
        Tool(name="update_ticket_status", description="更新 ticket 状态"),
        Tool(name="add_comment", description="在 ticket 添加评论"),
    ]

# integrations/jira/hooks.py

async def on_task_completed(task_id: str, result: TaskResult):
    """任务完成后的回调：更新 Jira ticket。"""
    jira_ticket_id = extract_jira_id(task_id)
    if jira_ticket_id:
        await jira_client.transition_issue(
            jira_ticket_id,
            transition="Done",
            comment=f"Auto-Claude 完成实现：{result.summary}"
        )
```

---

## 20.9　Lab 20.1：实现你的第一个扩展

**Part A**：实现一个 `PnpmValidator`，验证 `pnpm` 命令是否安全（允许 `install`/`run`/`test`，禁止 `publish`）。

**Part B**：实现一个 `LanguageEpisode` 类型：当 StackDetector 检测到新技术栈时，自动记录一个 `LANGUAGE_CHANGE` Episode 到 Graphiti，内容包含检测到的证据文件。

**Part C**：实现一个最简单的 MCP 服务器，提供 `get_env_info` 工具，返回当前 Python 版本、Node.js 版本和操作系统信息，并把它注册到 `create_client()` 中。

**验证标准**：
- Part A：`PnpmValidator` 拦截 `pnpm publish`，允许 `pnpm install`
- Part B：运行 StackDetector 后，Graphiti 中出现 `LANGUAGE_CHANGE` Episode
- Part C：在 Auto-Claude 运行过程中，智能体能调用 `get_env_info` 并获得正确结果

---

## 本章要点回顾

| 扩展点 | 扩展方式 | 难度 |
|--------|---------|------|
| 新 Provider | 修改 `create_client()` + 环境变量 | ★☆☆ |
| 安全验证器 | 继承 `CommandValidator`，注册到 `VALIDATORS` | ★☆☆ |
| Episode 类型 | 添加常量到 `EpisodeType`，调用 `memory.add_episode()` | ★☆☆ |
| MCP 服务器 | 实现 `mcp.server.Server`，注册到 `create_client()` | ★★☆ |
| 提示词修改 | 增量添加章节，使用模板变量动态组装 | ★★☆ |
| 技术栈检测 | 扩展 `StackDetector.detect_languages()`，添加命令注册 | ★★☆ |
| 完整插件 | 组合多个扩展点，实现完整业务流程 | ★★★ |
