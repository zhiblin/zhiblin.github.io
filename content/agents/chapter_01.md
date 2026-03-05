---
title: 第一章：从调用 API 到构建系统
date: 2026-03-05 10:00:00
categories: Agents
tags: [AI, Agent, Python, Claude, SDK]
---

# 第一章：从调用 API 到构建系统

> "工具是你使用的东西；系统是你构建的东西；智能体是代替你工作的东西。"

---

## 1.1 一个失败的下午

2024 年初，很多工程师都经历过这样一个下午。

你打开浏览器，把一段复杂的遗留代码粘贴进 ChatGPT 或 Claude 的对话框，问道："帮我重构这段代码，添加错误处理，然后写测试。"

模型给出了漂亮的回答。代码格式工整，注释详细，逻辑看起来完全正确。你满心期待地把它复制进编辑器——然后花了接下来两个小时修复导入错误、类型不匹配、以及那个模型根本不知道你项目里已经有 `base_handler.py` 的事实。

这不是模型的问题。这是**使用方式**的问题。

你把一个需要系统工程的任务，用了单次对话框的方式来处理。就像你试图用聊天来驾驶一辆汽车：你说"左转"，汽车确实转了，但下一条路怎么走，它不知道；前方有红灯，它不知道；你说完"停车"之后，车钥匙该交给谁，它更不知道。

**本书讲的，就是如何从"对话框用户"升级为"AI 系统工程师"。**

---

## 1.2 两种范式的本质差异

先做一个实验。下面两段代码，都在调用 Claude，但它们是完全不同的东西：

### 范式 A：API 调用

```python
import anthropic

client = anthropic.Anthropic()
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "帮我重构这段代码，添加错误处理"}
    ]
)
print(response.content[0].text)
```

运行时间：1-3 秒。输出：一段文本。你不知道它是否真的可以运行。

### 范式 B：Agent SDK 调用

```python
from claude_agent_sdk import ClaudeSDKClient, ClaudeAgentOptions
from pathlib import Path

client = ClaudeSDKClient(options=ClaudeAgentOptions(
    model="claude-sonnet-4-6",
    system_prompt="你是一名专业的 Python 工程师，工作目录是 /project",
    allowed_tools=["Read", "Write", "Edit", "Bash"],
    cwd="/project",
    max_turns=50,
))

async for message in client.process_query("重构 handler.py，添加错误处理，确保测试通过"):
    print(message)
```

运行时间：2-10 分钟。输出：真实被修改的文件、真实运行通过的测试。

---

这两段代码的**表面差距很小**，但**工程含义天壤之别**：

| 维度 | API 调用 | Agent SDK |
|------|----------|-----------|
| LLM 的角色 | 文本生成器 | 执行主体 |
| 你的代码做什么 | 传入问题，接收回答 | 给 LLM 提供工具和环境 |
| 结果 | 字符串（可能可用） | 文件修改（真实发生） |
| 错误处理 | 你判断输出是否正确 | LLM 自己运行测试并修复 |
| 上下文 | 你手动构建 | LLM 自主读取文件 |
| 迭代 | 每次要你干预 | 自动循环直到任务完成 |

**API 调用解决的是"生成"问题；Agent SDK 解决的是"执行"问题。**

这个区别，是本书所有内容的出发点。

---

## 1.3 Auto-Claude 解决的真正问题

让我们以 Auto-Claude 为例，看看一个工业级 AI 应用面对的真实挑战。

Auto-Claude 是一个"自主多智能体编码框架"。你告诉它"帮我实现用户权限管理功能"，它会：

1. 分析你的代码库，理解你的技术栈
2. 编写一份详细的功能规格说明（Spec）
3. 把任务分解成有依赖关系的子任务列表
4. 启动独立的 Coder Agent 依次完成每个子任务
5. 在 Git Worktree 中隔离所有改动（你的主分支始终安全）
6. 让 QA Agent 验证每个子任务的结果
7. 用 AI 解决代码合并冲突
8. 最后呈现给你一个可以直接 merge 的 PR

整个过程，你只需要描述目标——你可以去睡觉。

这不是科幻。这是 Auto-Claude v2.7.6 今天就能做到的事。

但要构建这样一个系统，需要回答四个工程问题：

```
问题 1：谁来决策？  → 规划智能体（Planner Agent）
问题 2：谁来执行？  → 编码智能体（Coder Agent）
问题 3：谁来验证？  → QA 智能体（QA Agent）
问题 4：谁来记忆？  → 记忆系统（Memory System）
```

每个问题都是一个独立的工程领域。本书每一部分，对应其中一个领域。

---

## 1.4 五步工作流：全貌与细节

在深入每个组件之前，先建立整体认知。Auto-Claude 的核心工作流是：

```
用户描述目标
      │
      ▼
┌─────────────┐
│  Spec 创建  │ ← AI 分析代码库、理解需求、写规格说明
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Planner    │ ← 把 Spec 分解为带依赖的子任务列表
└──────┬──────┘
       │
       ▼
┌─────────────┐     ┌─────────────┐
│  Coder #1   │     │  Coder #2   │ ← 并行执行独立子任务
└──────┬──────┘     └──────┬──────┘
       │                   │
       └────────┬──────────┘
                │
                ▼
┌───────────────────────────┐
│  QA Reviewer → QA Fixer  │ ← 验证 → 修复的迭代循环
└───────────────────────────┘
                │
                ▼
┌─────────────────────────────┐
│  AI Merge → 用户审查 → Merge │
└─────────────────────────────┘
```

让我们打开 Auto-Claude 的入口文件 `run.py`，看看系统启动时发生了什么：

```python
# apps/backend/run.py

# 启动前的三道门：
# 1. Python 版本检查（3.10+ 才支持 match 语法和新型 union types）
if sys.version_info < (3, 10):
    sys.exit("Error: Auto Claude requires Python 3.10 or higher.")

# 2. Windows 编码修复（处理 UTF-8 输出在 Windows 的乱码问题）
if sys.platform == "win32":
    # 双重策略：先尝试 reconfigure，失败则用 TextIOWrapper 包装
    ...

# 3. 平台依赖验证（在导入任何模块前检查，防止 Windows 特有的 pywintypes 导入链崩溃）
from core.dependency_validator import validate_platform_dependencies
validate_platform_dependencies()

# 最后才进入业务逻辑
from cli import main
```

注意这三道"门"。每一道都对应一个**真实踩过的坑**：

- 版本检查：Auto-Claude 使用了 Python 3.10 的 `match` 语句，没有这个检查，用户会在运行时遇到神秘的 `SyntaxError`
- Windows 编码：Electron 通过管道（pipe）读取 Python 子进程的输出，默认编码不是 UTF-8，中文路径会乱码
- 平台依赖：Windows 上某些包的导入链会触发 `pywintypes` 加载，如果它不存在，整个进程崩溃，且错误信息完全看不出是哪里的问题

**这才是 AI 应用工程师要做的事：** 不是调用 API，而是处理所有 API 调用之外的系统性问题。

---

## 1.5 Orchestrator-First：协调者的职责边界

Auto-Claude 的 `CLAUDE.md` 里有一段话，是整个系统的设计哲学核心：

> "You are an orchestrator. Your primary role is to understand what needs to be done, break it into workstreams, and delegate execution to agent teams. This keeps your context window focused on coordination and decision-making rather than filling up with implementation details."

**协调者（Orchestrator）不应该亲自实现**。这条规则背后有深刻的工程原因：

### 上下文窗口是稀缺资源

即使是 Claude Sonnet 4.6 的 200K token 上下文，也会在真实项目中被耗尽：

- 读了 10 个文件：~20K tokens
- 对话历史积累：~30K tokens
- 中间思考过程：~50K tokens
- 剩余可用：不到 100K

当协调者的上下文快满时，它开始"遗忘"之前的细节——就像一个同时记太多事情的人开始犯糊涂。

**解决方案是：让协调者只做决策，把所有实现细节下放给"新鲜上下文"的执行者。**

这就是为什么 Auto-Claude 的每个 Coder Agent 都是独立进程，拥有完全干净的上下文窗口。

### 什么时候委托，什么时候直接做

```
直接做：
  - 单文件修改
  - 简单查找（Grep/Glob）
  - 快速决策

委托给子智能体：
  - 多文件修改
  - 代码库全局分析
  - 可并行的独立任务
  - 会消耗大量上下文的实现工作
```

这个判断标准，也是你在构建自己的 AI 系统时需要做出的第一个架构决策。

---

## 1.6 智能体的四个核心能力

任何有实用价值的智能体，都必须具备这四种能力。缺少其中任何一种，它都只是一个"聊天机器人"。

### 能力一：工具（Tools）

工具让 LLM 从"回答者"变成"执行者"。

```python
# 没有工具的 LLM：只能告诉你应该怎么做
response = "你应该创建一个 users.py 文件，内容如下：..."

# 有了 Write 工具的 LLM：直接把文件写进去
# LLM 内部会调用：
# Write(file_path="users.py", content="...")
# 文件真实被创建
```

Auto-Claude 的 Coder Agent 拥有的工具包括：`Read`、`Write`、`Edit`、`Bash`、`Glob`、`Grep`，以及各种 MCP 工具（第 6 章详述）。

### 能力二：记忆（Memory）

LLM 天生没有记忆——每次对话都是白板。解决方案是把"记忆"外化到文件系统或数据库：

```
# Auto-Claude 的记忆文件结构
.auto-claude/specs/001-my-feature/memory/
├── codebase_map.json      # "auth.py 是处理登录的文件"
├── patterns.md            # "所有 API 端点都用 @require_auth 装饰器"
├── gotchas.md             # "数据库连接必须手动关闭，否则泄漏"
└── session_insights/
    ├── session_001.json   # 第一次执行时发现了什么
    └── session_002.json   # 第二次继续时的补充
```

每次新的 Agent 会话开始时，它的第一步就是读取这些文件，"回忆"之前的上下文。

### 能力三：反馈循环（Feedback Loop）

单次执行的智能体是脆弱的。只有形成"执行 → 验证 → 修复"的循环，才能处理真实世界的不确定性：

```
                    ┌─────────┐
                    │ 执行子任务 │
                    └────┬────┘
                         │
                    ┌────▼────┐
                    │ 运行验证  │  ← "pytest 测试通过了吗？"
                    └────┬────┘
                         │
              ┌──────────▼──────────┐
              │    验证通过了吗？    │
              └──────────┬──────────┘
          通过           │         失败
          ▼              │              ▼
    标记 completed  ─────┘      修复问题，重新验证
```

Auto-Claude 的 QA 阶段，就是这个循环的具体实现。

### 能力四：隔离（Isolation）

**这是最被低估的能力。**

Auto-Claude 的每个构建任务，都在独立的 Git Worktree 中执行。这意味着：

- AI 搞坏了代码？主分支安然无恙
- 两个任务并行开发？互不干扰
- 用户想放弃这次构建？一个命令删除 Worktree，干净无残留

```bash
# 一个正在构建的项目，目录结构可能是这样的：
my-project/
├── src/                           # 主分支代码（干净）
└── .auto-claude/
    └── worktrees/
        └── tasks/
            ├── 001-user-auth/     # 任务 1 的隔离 Worktree
            └── 002-dashboard/     # 任务 2 的隔离 Worktree
```

这四种能力，在接下来的章节中，我们会逐一深入实现。

---

## 1.7 一个真实的执行轨迹

理论讲完了，让我们看一次真实的 Agent 执行过程中发生了什么。

Auto-Claude 使用 `phase_event.py` 在不同执行阶段向前端发送结构化事件：

```python
# apps/backend/core/phase_event.py

class ExecutionPhase(str, Enum):
    PLANNING    = "planning"      # 规划阶段
    CODING      = "coding"        # 编码阶段
    QA_REVIEW   = "qa_review"     # QA 审查
    QA_FIXING   = "qa_fixing"     # QA 修复
    COMPLETE    = "complete"      # 完成
    FAILED      = "failed"        # 失败
    # 智能暂停状态
    RATE_LIMIT_PAUSED   = "rate_limit_paused"   # 被限速，等待恢复
    AUTH_FAILURE_PAUSED = "auth_failure_paused"  # 认证失败，需要人工介入

def emit_phase(phase: ExecutionPhase, message: str = "", **kwargs) -> None:
    """向 stdout 输出结构化阶段事件，供 Electron 前端解析"""
    payload = {"phase": phase.value, "message": message, **kwargs}
    print(f"__EXEC_PHASE__:{json.dumps(payload)}", flush=True)
```

注意 `__EXEC_PHASE__:` 这个前缀。Python 进程通过 stdout 与 Electron 的 Main Process 通信——这是跨语言跨进程通信最简单可靠的方式：**把结构化数据打印到标准输出，让父进程解析**。

当一个 Agent 执行时，这些事件以时间顺序流出：

```
__EXEC_PHASE__:{"phase":"planning","message":"分析代码库结构...","progress":10}
__EXEC_PHASE__:{"phase":"coding","subtask":"subtask-1-1","message":"实现数据模型","progress":25}
__EXEC_PHASE__:{"phase":"coding","subtask":"subtask-1-2","message":"创建 API 端点","progress":50}
__EXEC_PHASE__:{"phase":"qa_review","message":"验证实现质量...","progress":70}
__EXEC_PHASE__:{"phase":"qa_fixing","message":"修复 2 个问题","progress":85}
__EXEC_PHASE__:{"phase":"complete","message":"所有子任务完成","progress":100}
```

同时，`session.py` 中的 `post_session_processing()` 在每个 Agent 会话结束后运行，处理：

- 分析本次会话的 git commit，提取修改了哪些文件
- 把发现的模式和踩到的坑写入记忆文件
- 通知 Linear（如果集成了项目管理工具）
- 如果发生了错误，触发恢复流程

**这就是"AI 应用工程"的日常：** 不是让模型生成代码，而是设计围绕模型的完整数据流、状态机、反馈机制和错误处理。

---

## 1.8 本书的学习路径

```
第一部分（第 1-3 章）
  AI 应用工程的思维转变
  └─ 你现在在这里

第二部分（第 4-8 章）
  Claude Agent SDK 深度实战
  └─ 学会构建生产级 Agent

第三部分（第 9-13 章）
  Python 智能体系统工程
  └─ 学会设计多智能体系统

第四部分（第 14-17 章）
  Electron 桌面应用工程
  └─ 学会构建 AI 原生桌面界面

第五部分（第 18-19 章）
  测试与可观测性
  └─ 学会让 AI 系统变得可靠

第六部分（第 20-21 章）
  综合实战
  └─ 从零构建你自己的 Auto-Claude
```

每一章都有**可运行的 Lab**。Lab 的参考实现都在本书的 GitHub 仓库中，但强烈建议你先自己实现，再对比参考代码——差异之处，正是最有价值的学习点。

---

## 本章小结

```
★ Insight ─────────────────────────────────────────────
• API 调用 vs Agent SDK：前者是问答，后者是委托执行
  区别不在 API，在于你是否给了 LLM 工具和反馈环境

• 智能体的四项核心能力：
  工具（能做事）、记忆（跨会话知道发生了什么）
  反馈循环（能从失败中恢复）、隔离（失败时不影响主系统）

• 协调者的铁律：
  协调者只做决策，把实现细节委托给拥有"新鲜上下文"的执行智能体
  这不是风格问题，是应对上下文窗口限制的工程必需

• run.py 的三道门告诉我们：
  AI 应用的大部分 Bug，不在模型里，在系统层
  版本检查、编码处理、平台差异——这些都是你的工程责任
─────────────────────────────────────────────────────────
```

---

## Lab 1.1：感受两种范式的差异

**目标：** 用实验感受 API 调用与 Agent SDK 的工程差异，建立直觉。

**前置条件：**
- Python 3.10+
- `pip install anthropic claude-agent-sdk`
- 有效的 `ANTHROPIC_API_KEY` 或 `CLAUDE_CODE_OAUTH_TOKEN`

---

### Part A：API 调用版（10 分钟）

在任意目录创建 `lab1_api.py`：

```python
"""
Lab 1.1 Part A - API 调用版
目标：体验 API 调用的能力边界
"""
import anthropic

def analyze_code_via_api(code: str) -> str:
    """用 API 调用分析代码，返回改进建议"""
    client = anthropic.Anthropic()

    response = client.messages.create(
        model="claude-haiku-4-5-20251001",  # 用 Haiku 节约费用
        max_tokens=1024,
        messages=[{
            "role": "user",
            "content": f"分析这段代码，指出问题并给出改进建议：\n\n{code}"
        }]
    )

    return response.content[0].text


# 测试代码
sample_code = """
def get_user(user_id):
    conn = connect_db()
    result = conn.execute(f"SELECT * FROM users WHERE id = {user_id}")
    return result.fetchone()
"""

result = analyze_code_via_api(sample_code)
print("=== API 调用结果 ===")
print(result)
print()
print("注意：模型只是告诉了你应该怎么做，但没有实际修改任何文件")
```

运行并记录：
1. 模型发现了哪些问题？（SQL 注入？连接未关闭？）
2. 它给出的修复建议，你需要做什么才能应用？
3. 如果代码在 `/project/db/users.py` 里，模型能直接去读那个文件吗？

---

### Part B：Agent SDK 版（20 分钟）

**骨架代码**——你需要补全 `TODO` 部分：

```python
"""
Lab 1.1 Part B - Agent SDK 版
目标：让 Agent 实际读取并修复一个真实文件
"""
import asyncio
from pathlib import Path
from claude_agent_sdk import ClaudeSDKClient, ClaudeAgentOptions

# --- 准备工作 ---

# 在当前目录创建一个有问题的 Python 文件
WORK_DIR = Path("./lab1_workspace")
WORK_DIR.mkdir(exist_ok=True)

BAD_CODE = '''\
def get_user(user_id):
    """从数据库获取用户（有 bug）"""
    conn = connect_db()
    # 问题1：SQL 注入漏洞
    result = conn.execute(f"SELECT * FROM users WHERE id = {user_id}")
    # 问题2：连接没有关闭
    return result.fetchone()

def divide(a, b):
    """除法（没有除零保护）"""
    return a / b
'''

target_file = WORK_DIR / "users.py"
target_file.write_text(BAD_CODE)
print(f"已创建有问题的文件：{target_file}")

# --- 核心部分：创建 Agent 并执行修复 ---

async def run_fixer_agent():
    """
    TODO：创建一个 Agent，让它：
    1. 读取 lab1_workspace/users.py
    2. 找出所有安全问题和 bug
    3. 修复它们，直接写回文件
    4. 打印修复总结

    提示：
    - 使用 ClaudeAgentOptions 配置 allowed_tools
    - allowed_tools 至少需要包含：["Read", "Write", "Edit"]
    - cwd 设置为 WORK_DIR 的字符串路径
    - max_turns=20 足够用于这个简单任务
    """

    # TODO 1：创建 ClaudeAgentOptions
    options = ClaudeAgentOptions(
        model="claude-haiku-4-5-20251001",
        system_prompt="""你是一名代码安全专家。
        当你发现代码中的安全问题或 bug 时，你会直接修复它们。
        修复后，简要说明你做了什么改变。""",

        # TODO：填写 allowed_tools 和 cwd
        allowed_tools=["Read", "Write", "Edit"],  # 你认为还需要哪些工具？
        cwd=str(WORK_DIR),
        max_turns=20,
    )

    client = ClaudeSDKClient(options=options)

    # TODO 2：构造 prompt
    # 提示：告诉 Agent 文件在哪里，让它读取、分析并修复
    prompt = """
    请检查并修复 users.py 中的所有问题。
    修复完成后，告诉我你改了什么。
    """

    # TODO 3：运行 Agent 并打印输出
    print("\n=== Agent 开始执行 ===")
    async for message in client.process_query(prompt):
        # 提示：message 对象有不同的类型，打印它的类型和内容
        print(f"[{type(message).__name__}] {message}")

    # 验证：读取修复后的文件
    print("\n=== 修复后的文件内容 ===")
    print(target_file.read_text())


asyncio.run(run_fixer_agent())
```

---

### Part C：对比与思考

完成 Part B 后，对比两个版本的结果，思考以下问题：

1. **Part A 的建议** vs **Part B 的实际修复**——哪个更接近你想要的？

2. **Agent 实际读取了文件吗？** 查看 Part B 的输出，找到 `ToolUseBlock` 类型的消息，你能看到它调用了哪些工具吗？

3. **如果文件有 10 个问题，Part A 能批量修复吗？Part B 呢？**

4. **思考：** 如果你要处理整个项目（50 个文件），两种方式的成本和效果会有什么不同？

---

### 验证标准

- [ ] `lab1_workspace/users.py` 的 SQL 注入漏洞已被修复（使用参数化查询）
- [ ] 数据库连接有了关闭逻辑（`try/finally` 或 `with` 语句）
- [ ] `divide` 函数有了除零保护
- [ ] Agent 的输出中可以找到 `Read` 和 `Write` 工具的调用记录

---

### 进阶挑战

如果基础版做完了，尝试：

1. **添加验证步骤：** 修复后，让 Agent 自己编写一个简单的测试文件，验证修复是否正确
2. **批量处理：** 在 `lab1_workspace` 里创建 3 个有不同问题的文件，让 Agent 一次性修复所有文件
3. **与 Part A 对比成本：** 用 Anthropic 的 Token 计量 API，分别统计两种方式处理同一个问题的 Token 消耗

---

### 参考：Part B 的一种实现

如果你卡住了，下面是一个可以正常运行的参考实现（尝试自己做完再看）：

```python
async def run_fixer_agent():
    options = ClaudeAgentOptions(
        model="claude-haiku-4-5-20251001",
        system_prompt="""你是一名代码安全专家。
        当你发现代码中的安全问题或 bug 时，你会直接修复它们。
        修复后，简要说明你做了什么改变。""",
        allowed_tools=["Read", "Write", "Edit", "Bash"],
        cwd=str(WORK_DIR),
        max_turns=20,
    )

    client = ClaudeSDKClient(options=options)

    prompt = "请读取 users.py，找出所有安全漏洞和 bug，直接修复文件，然后告诉我做了哪些改变。"

    print("\n=== Agent 开始执行 ===")
    async for message in client.process_query(prompt):
        msg_type = type(message).__name__
        if hasattr(message, 'type'):
            if message.type == 'tool_use':
                print(f"  [工具调用] {message.name}({list(message.input.keys())})")
            elif message.type == 'text':
                print(f"  [文本] {message.text[:100]}...")
        else:
            print(f"  [{msg_type}] {str(message)[:100]}")

    print("\n=== 修复后的文件 ===")
    print(target_file.read_text())
```

---

**下一章预告：** 现在你已经感受到了单智能体的能力边界。接下来，我们要思考：当一个任务需要多个智能体协作时，如何设计它们之间的关系？

---

*本章对应的完整代码：[GitHub 仓库 / chapter_01 / lab1/]()*
