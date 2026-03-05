---
title: 第二章：多智能体系统的三种基本拓扑
date: 2026-01-02 00:00:00
categories: Agents
tags: [AI, Agent, Python, Claude, SDK]
---

# 第二章：多智能体系统的三种基本拓扑

> "一个智能体能做的事有限，多个智能体的协作方式决定了系统的上限。"

---

## 2.1 为什么一个智能体不够

上一章的 Lab 让你亲身感受了单智能体的能力：给一个有问题的文件，让它读取并修复。

但现实中的软件任务远不止于此。考虑一个常见的需求：

> "实现用户权限管理：支持角色（Role）和权限（Permission）的绑定，在 API 层进行鉴权，前端展示权限控制的菜单，并写完整的测试。"

如果你把这个需求完整地交给一个智能体：

- 它需要读取后端代码，理解数据库结构，实现数据模型
- 然后读取 API 路由代码，实现鉴权中间件
- 然后读取前端代码，实现菜单权限控制组件
- 最后写测试，覆盖所有层次

这个过程会持续 30-60 分钟，产生大量的工具调用记录和中间输出。到最后，智能体的上下文窗口里有：

- 十几个文件的完整内容
- 所有中间步骤的执行记录
- 修复过程中产生的大量调试输出

**上下文窗口不够用了。**

更糟糕的是，当一个智能体承担了所有工作，它必须同时扮演"规划者"、"实现者"和"验证者"三个角色，而这三个角色对 LLM 的要求完全不同：

- 规划者需要宏观视野，不能陷入细节
- 实现者需要关注具体代码，不能被宏观目标干扰
- 验证者需要批判性思维，不能对自己刚写的代码有认知偏差

这就是为什么 Auto-Claude 把一个大任务，分解给多个角色清晰的专职智能体。

---

## 2.2 三种基本拓扑

所有多智能体系统，无论多复杂，都是三种基本拓扑的组合。

### 拓扑一：顺序流水线（Sequential Pipeline）

```
Agent A → Agent B → Agent C
```

上一个智能体的输出，是下一个智能体的输入。每个智能体只负责一个明确的步骤。

**Auto-Claude 的 Spec 创建流水线就是这种拓扑：**

```
Discovery Agent      Context Agent     Requirements Agent
     │                    │                   │
     │   project_index    │  context.json     │  requirements.json
     └──────────────────► └─────────────────► └──────────────►
                                                              │
                                               Spec Writer Agent
                                                     │
                                               Self-Critic Agent
                                                     │
                                              Planner Agent
```

每个阶段产出一个文件，下一个阶段读取这个文件作为输入。这是最简单、最易调试的拓扑。

**优点：**
- 每步结果都可检查
- 出错时知道是哪个阶段的问题
- 每个智能体上下文干净，专注于单一任务

**缺点：**
- 严格顺序，无法并行加速
- 任何一步失败，后续全部阻塞

### 拓扑二：并行扇出（Parallel Fan-out）

```
          ┌→ Agent B1 ─┐
Coordinator │→ Agent B2 ─│→ Aggregator
          └→ Agent B3 ─┘
```

协调者把任务分解为若干独立子任务，同时启动多个工作智能体并行执行，最后汇总结果。

**Auto-Claude 的多 Coder 并行就是这种拓扑：**

```
Planner
  │
  │ implementation_plan.json
  │  (含依赖关系分析: parallel_safe: true)
  ▼
  ├──► Coder Agent #1 (Phase-2: 前端组件)  ─────────────────┐
  └──► Coder Agent #2 (Phase-2: 后端 Worker) ────────────────│
                                                              ▼
                                                        QA 验证循环
```

两个 Coder 同时工作，前提是它们处理的是不同的文件集合（`files_to_modify` 不相交）。

**优点：**
- 大幅减少总耗时（取决于最慢的那个智能体）
- 每个工作智能体上下文独立，互不干扰

**缺点：**
- 需要仔细分析依赖关系，避免竞争同一文件
- 并行产生的改动在合并时可能产生冲突（第 13 章专门讲这个）
- 协调复杂度更高

### 拓扑三：迭代循环（Iterative Loop）

```
┌─────────────────────────────────┐
│                                 │
│  Agent A → 验证 ── 通过 ─────► 退出
│     ↑          └── 失败 ──┐
│     └──────────────────────┘
│                                 │
└─────────────────────────────────┘
```

智能体执行后立即验证，验证失败则继续修复，直到通过或达到最大迭代次数。

**Auto-Claude 的 QA 验证循环是最典型的这种拓扑，**也是 AI 系统可靠性的核心机制。

---

## 2.3 解剖 QA 循环：一个生产级迭代拓扑

`qa/loop.py` 中的 `run_qa_validation_loop()` 是本书里最值得深度阅读的代码之一。它用 660 行实现了一个工业级的迭代智能体循环，包含了所有你在生产环境中需要考虑的情况。

让我们逐层解剖它。

### 第一层：基本状态机

最简化的 QA 循环只需要三件事：

```python
while iteration < MAX_QA_ITERATIONS:
    status = run_qa_reviewer()   # 审查
    if status == "approved":
        return True              # 通过，退出
    run_qa_fixer()               # 失败，修复
    iteration += 1

return False  # 达到上限，失败
```

这就是拓扑三的骨架。但真实代码有更多的处理层。

### 第二层：不同 Agent 使用不同的 Client 配置

注意代码中一个关键细节：每次创建 Agent 都是全新的 `create_client()` 调用：

```python
# qa/loop.py - QA 审查者
client = create_client(
    project_dir, spec_dir, qa_model,
    agent_type="qa_reviewer",   # ← 角色特定的工具配置
    **qa_thinking_kwargs,
)

async with client:
    status, response, _ = await run_qa_agent_session(client, ...)

# 使用 async with 确保 Agent 进程被干净地关闭

# QA 修复者 - 与审查者完全隔离
fix_client = create_client(
    project_dir, spec_dir, qa_model,
    agent_type="qa_fixer",      # ← 不同的角色，不同的工具
    **fixer_thinking_kwargs,
)

async with fix_client:
    fix_status, fix_response, _ = await run_qa_fixer_session(fix_client, ...)
```

`qa_reviewer` 和 `qa_fixer` 是两个完全独立的智能体，拥有各自独立的上下文窗口。这个设计有深刻的工程理由：

**认知偏差隔离：** `qa_reviewer` 不知道之前试图如何修复问题，它的判断是纯粹的。如果同一个 Agent 既审查又修复，它会对自己的修复结果有认知偏差，倾向于认为自己的修复是有效的。

**职责清晰：** 审查者的 System Prompt 侧重于找问题，修复者的 System Prompt 侧重于解决具体问题——两者不该混用。

### 第三层：错误分类与自我纠正

QA 循环处理三种状态：

```python
if status == "approved":
    # 通过 → 直接成功退出

elif status == "rejected":
    # 被拒绝 → 让 Fixer 修复，然后重试
    # 这是正常的失败路径

elif status == "error":
    # 意外错误 → 记录错误上下文，供下次迭代参考
    consecutive_errors += 1

    # 自我纠正：把错误信息注入下一次的 Prompt
    last_error_context = {
        "error_type": "missing_implementation_plan_update",
        "error_message": response,
        "expected_action": "You MUST update implementation_plan.json ...",
        "file_path": str(spec_dir / "implementation_plan.json"),
    }
```

注意 `last_error_context`。这是一个在迭代间传递的"短期记忆"：当 QA Agent 上一次没有按要求更新文件时，下一次迭代的 Prompt 里会包含这次的错误细节和明确的纠正指示。

这是比重试更智能的恢复策略：**不是盲目重试，而是告诉 Agent 上次出了什么问题。**

### 第四层：递归问题检测与人工升级

这是整个 QA 循环最精妙的设计：

```python
# 在记录当前迭代之前，先检查历史中是否有相同问题
history = get_iteration_history(spec_dir)
has_recurring, recurring_issues = has_recurring_issues(
    current_issues, history
)

# 注意顺序：先检查，再记录（防止当前问题和自身匹配）
record_iteration(spec_dir, qa_iteration, "rejected", current_issues, ...)

if has_recurring:
    # 同一个问题出现 3 次以上 → 机器搞不定，升级给人类
    await escalate_to_human(spec_dir, recurring_issues, qa_iteration)
    return False
```

如果 QA Agent 发现了一个问题，Fixer Agent 尝试修复，但下一轮 QA 又发现了同样的问题——且这种情况出现了 3 次以上，系统就认为这个问题超出了 AI 的能力范围，自动创建升级文件并通知人工介入。

```
迭代 1: QA 发现 "SQL 注入漏洞" → Fixer 修复
迭代 2: QA 又发现 "SQL 注入漏洞" → Fixer 修复
迭代 3: QA 还是发现 "SQL 注入漏洞" → 停止，升级给人类

文件写入: spec_dir/ESCALATION.md
内容: "此问题反复出现 3 次，AI 无法解决，需要人工介入"
```

**这是 AI 系统设计中的关键工程实践：** 知道自己什么时候解决不了，并且优雅地交回控制权。

一个不知道何时放弃的 AI 系统，会无限循环直到耗尽 API 配额，或在 50 次迭代后沮丧地失败——两者都是糟糕的用户体验。

---

## 2.4 智能体角色边界的设计原则

在 Auto-Claude 里，每个 Agent 类型都有明确的边界。让我们看看它们的完整列表：

```python
# 来自 agents/tools_pkg/models.py（简化）
AGENT_TYPES = [
    "spec_gatherer",      # Spec 创建第一步：交互式需求收集
    "spec_researcher",    # Spec 创建：研究外部集成
    "spec_writer",        # Spec 创建：编写规格文档
    "spec_critic",        # Spec 创建：批评和改进规格
    "planner",            # 规划：分解任务为子任务
    "coder",              # 实现：执行具体子任务
    "qa_reviewer",        # 质量保证：审查实现结果
    "qa_fixer",           # 质量保证：修复审查发现的问题
    "validation_fixer",   # 验证：修复规划阶段的验证错误
    "followup_planner",   # 规划：为已完成的 Spec 添加后续子任务
    "insights",           # 代码洞察：分析代码库生成见解
    "roadmap",            # 路线图：AI 辅助产品规划
]
```

每个角色遵循三个设计原则：

### 原则一：单一职责

每个智能体只做一件事，并把它做好。

`qa_reviewer` 的唯一职责是：检查实现是否符合规格，然后更新 `implementation_plan.json` 的 `qa_signoff` 字段为 `approved` 或 `rejected`。

它不负责修复问题，也不负责更新其他字段。如果它尝试直接修复代码，那是越权行为——修复是 `qa_fixer` 的工作。

### 原则二：通过文件通信，而非内存传参

每个智能体通过读写文件来交换信息，而不是通过函数参数或共享内存。

```
Planner 写：implementation_plan.json
Coder 读：  implementation_plan.json，找到 pending 的子任务
Coder 写：  implementation_plan.json，更新子任务 status 为 completed
QA 读：     implementation_plan.json，了解实现了什么
QA 写：     implementation_plan.json，填入 qa_signoff 字段
```

这种设计的好处是显而易见的：

1. **持久化**：文件在进程重启后仍然存在
2. **可检查**：人类可以随时打开文件查看当前状态
3. **解耦**：Agent 之间不需要直接通信，通过文件约定接口
4. **可恢复**：任何步骤失败，下次从文件中读取状态继续

这个模式有个专业名字：**文件系统作为消息总线**（Filesystem as Message Bus）。

### 原则三：明确的输入/输出契约

每个智能体的"接口"由以下四部分定义：

```
读取：[input_files]    → 这是输入
写入：[output_files]   → 这是输出
验证：[success_check]  → 如何判断成功
超时：[max_turns]      → 允许运行多少轮
```

以 Planner Agent 为例：

```
读取：spec.md, project_index.json, context.json
写入：implementation_plan.json, init.sh, build-progress.txt
验证：implementation_plan.json 存在且包含 phases 数组
超时：1000 轮（一般用不完）
```

当协调层（`coder.py`）调用 Planner 时，它知道如果 `implementation_plan.json` 被成功写入，规划阶段就完成了。这个契约非常清晰，不需要分析 Agent 说了什么。

---

## 2.5 上下文隔离：最被低估的架构决策

前面多次提到"新鲜上下文窗口"，现在我们来深入理解它。

每次调用 `create_client()` 并执行 `async with client: ...`，都会：
1. 启动一个全新的 Claude Code CLI 子进程
2. 这个子进程拥有完全空白的对话历史
3. 只有通过 `system_prompt` 和首次 `process_query()` 注入的内容是已知的

```python
# 每个 Agent 都是独立进程，独立上下文

# 第 1 次 Coder 运行（子任务 1）
async with create_client(..., agent_type="coder") as client:
    await run_agent_session(client, subtask_1)
    # 进程在 async with 块结束时被销毁
    # 所有上下文消失

# 第 2 次 Coder 运行（子任务 2）
async with create_client(..., agent_type="coder") as client:
    await run_agent_session(client, subtask_2)
    # 这个 Agent 对子任务 1 的细节一无所知
    # 它只能从文件系统获取信息
```

这种设计初看起来像是缺陷（为什么要忘掉之前学到的东西？），实际上是精心设计的优点：

**上下文污染问题：** 如果同一个 Agent 连续执行 10 个子任务，它的上下文里会积累 10 个子任务的所有细节——读过的文件、运行过的命令、踩过的坑、写过的代码。这些内容大部分对当前子任务是无关噪声，反而会干扰 LLM 的注意力。

**注入而非积累：** 更好的策略是：只把当前子任务需要知道的信息，精准地注入到新的上下文中。

```python
# prompt_generator.py 中的 generate_subtask_prompt()

def generate_subtask_prompt(subtask, spec_dir, project_dir):
    """为每个新 Agent 会话构建精准的上下文注入"""

    # 1. 当前子任务的目标
    prompt = f"## 你的任务\n{subtask['description']}\n"

    # 2. 只有相关的记忆（不是全部历史）
    memory = load_relevant_memory(spec_dir, subtask)
    if memory:
        prompt += f"## 来自之前会话的记忆\n{memory}\n"

    # 3. 要修改的文件列表（不是整个代码库）
    prompt += f"## 要修改的文件\n{subtask['files_to_modify']}\n"

    # 4. 参考模式（告诉 Agent 去哪里找例子）
    prompt += f"## 参考以下文件的代码模式\n{subtask['patterns_from']}\n"

    return prompt
```

**关键洞察：** 上下文的价值不在于"越多越好"，而在于"越精准越好"。把 100K token 的上下文填满垃圾，不如用 10K token 的精准上下文。

---

## 2.6 Auto-Claude 的混合拓扑

现在我们有了理解全局架构的基础。Auto-Claude 是三种拓扑的混合：

```
用户输入
  │
  ▼
┌────────────────────────────────────────┐
│        顺序流水线（Spec 创建阶段）        │
│                                        │
│  Discovery → Context → Requirements   │
│  → Research → Spec Writing → Critique │
│  → Planner                            │
└──────────────────────┬─────────────────┘
                       │
                       ▼ implementation_plan.json
┌────────────────────────────────────────┐
│       并行扇出（Coding 阶段）           │
│                                        │
│   ┌─ Coder #1 (Phase-1 subtasks) ─┐   │
│   ├─ Coder #2 (Phase-2 subtasks) ─┤   │
│   └─ Coder #N (parallel phases)  ─┘   │
│                                        │
│   依赖图决定哪些 Phase 可以并行        │
└──────────────────────┬─────────────────┘
                       │
                       ▼ 所有子任务完成
┌────────────────────────────────────────┐
│       迭代循环（QA 验证阶段）           │
│                                        │
│   QA Reviewer ──→ 通过 ──────────────►│
│        │                               │
│        └──→ 拒绝 ──→ QA Fixer ──┐     │
│              ↑                   │     │
│              └───────────────────┘     │
│   (最多 50 次迭代，递归问题则人工升级)  │
└────────────────────────────────────────┘
                       │
                       ▼
             用户审查并 Merge
```

这三个阶段是**嵌套的**：

- 整体是顺序流水线：Spec → Code → QA → Merge
- Coding 阶段内部是并行扇出（多个 Coder 同时工作）
- QA 阶段内部是迭代循环（不断审查修复直到通过）

理解了这个全局视图，你就理解了 Auto-Claude 所有代码的组织逻辑。

---

## 2.7 拓扑选择的工程判断

构建自己的多智能体系统时，如何选择拓扑？

**选顺序流水线，当：**
- 每个步骤依赖上一步的输出
- 需要人工介入某个步骤（在步骤间插入人工审查）
- 调试优先级高（一步一步地检查，问题易于定位）

**选并行扇出，当：**
- 子任务之间没有文件依赖
- 时间成本是主要约束
- 你有能力处理合并冲突

**选迭代循环，当：**
- 任务有明确的"通过/失败"判断标准
- 失败后存在可操作的修复路径
- 需要机器能自动收敛到正确结果

**设计时必须回答的三个问题：**
1. 这个阶段的"完成"标准是什么？（文件是否存在？测试是否通过？）
2. 失败后如何恢复？（重试？人工介入？跳过？）
3. 最坏情况下如何退出？（最大迭代次数？超时？）

```
★ Insight ─────────────────────────────────────────────
• 三种拓扑的本质区别：
  - 顺序流水线：A 的输出是 B 的输入，严格依赖
  - 并行扇出：多个 Agent 处理不相交的数据，汇总结果
  - 迭代循环：重复执行直到满足退出条件（通过或上限）

• QA 循环的三个递进层次：
  1. 基本重试（status != approved → 继续循环）
  2. 自我纠正（把错误信息注入下一次的 Prompt）
  3. 人工升级（同一问题出现 3+ 次 → 停止循环）
  这三层构成了生产级 AI 系统的可靠性基础

• "文件系统作为消息总线"是多智能体通信的核心模式：
  Agent 不直接通信，通过读写约定格式的文件交换信息
  好处：持久化、可检查、进程独立、天然解耦

• 上下文注入优于上下文积累：
  精准的 10K token 比充满噪声的 100K token 更有价值
  每个 Agent 会话应该只注入它当前需要的最小上下文
─────────────────────────────────────────────────────────
```

---

## Lab 2.1：从零实现一个 QA 验证循环

**目标：** 实现一个简化版的迭代循环拓扑，包含审查、修复、递归问题检测。

**前置条件：**
- 完成 Lab 1.1
- `pip install anthropic claude-agent-sdk`

---

### 准备：一个有质量问题的代码库

```python
# lab2_workspace/payment.py — 这是被测试的代码，故意埋了几个问题
BAD_PAYMENT_CODE = '''
import json

def process_payment(user_id, amount, card_number):
    """处理支付"""
    # 问题1: 没有金额验证（负数、零值）
    # 问题2: 信用卡号直接打印到日志（PCI 违规）
    # 问题3: 没有事务处理（部分失败无法回滚）
    print(f"Processing payment for {user_id}: ${amount}, card: {card_number}")

    result = {
        "status": "success",
        "user_id": user_id,
        "amount": amount,
    }

    # 问题4: 直接写文件没有错误处理
    with open("payments.log", "a") as f:
        f.write(json.dumps(result) + "\\n")

    return result

def refund_payment(transaction_id, amount):
    """退款"""
    # 问题5: 没有验证 transaction_id 是否存在
    # 问题6: amount 可能超过原始支付金额
    print(f"Refund: {transaction_id}, ${amount}")
    return {"status": "refunded"}
'''
```

---

### 骨架代码

```python
"""
Lab 2.1 - 实现 QA 验证循环
目标：用迭代循环拓扑，让 AI 自动找到并修复代码质量问题
"""
import asyncio
import json
from pathlib import Path
from claude_agent_sdk import ClaudeSDKClient, ClaudeAgentOptions

# --- 配置 ---
WORK_DIR = Path("./lab2_workspace")
WORK_DIR.mkdir(exist_ok=True)
QA_REPORT_FILE = WORK_DIR / "qa_report.json"
MAX_ITERATIONS = 5

# 写入有问题的代码
(WORK_DIR / "payment.py").write_text(BAD_PAYMENT_CODE)  # 上面定义的有问题代码


# ============================================================
# TODO 1: 实现 QA 审查者 Agent
# ============================================================

async def run_qa_reviewer(iteration: int) -> tuple[str, list]:
    """
    运行 QA 审查者，返回 (status, issues)

    status: "approved" | "rejected"
    issues: 发现的问题列表，每个问题是 {"id": "...", "title": "...", "severity": "..."}

    提示：
    - 创建一个 ClaudeAgentOptions，system_prompt 告诉它是代码审查员
    - 要求它把发现的问题以 JSON 格式写入 qa_report.json
    - 格式：{"status": "approved"/"rejected", "issues": [...]}
    - allowed_tools 需要包含 Read 和 Write
    - 在 prompt 里告诉它要检查哪个文件，输出写到哪里
    """

    # TODO: 创建审查者 Agent
    reviewer_prompt = f"""
    请审查 payment.py 中的代码质量问题。

    重点检查：
    1. 输入验证（金额、参数合法性）
    2. 安全问题（敏感信息泄露、日志安全）
    3. 错误处理（文件操作、事务性）
    4. 业务逻辑正确性（退款金额验证等）

    这是第 {iteration} 次审查。

    将结果以以下 JSON 格式写入 qa_report.json：
    {{
        "status": "approved" 或 "rejected",
        "issues": [
            {{"id": "issue-1", "title": "问题标题", "severity": "critical/high/medium"}}
        ]
    }}

    如果没有发现问题，写入 {{"status": "approved", "issues": []}}
    """

    # TODO: 运行 Agent，等待完成
    # 提示：读取 qa_report.json 获取结果

    # 暂时返回 mock 数据，你需要替换为真实的 Agent 调用
    raise NotImplementedError("请实现 QA 审查者 Agent")


# ============================================================
# TODO 2: 实现 QA 修复者 Agent
# ============================================================

async def run_qa_fixer(issues: list, iteration: int) -> bool:
    """
    根据 issues 列表修复 payment.py

    返回 True 表示修复成功，False 表示失败

    提示：
    - 把 issues 格式化进 prompt
    - 告诉 Fixer 直接修改 payment.py
    - allowed_tools 需要包含 Read, Write, Edit
    """

    issues_text = "\n".join(
        f"- [{issue['severity'].upper()}] {issue['title']}"
        for issue in issues
    )

    fixer_prompt = f"""
    请修复 payment.py 中的以下问题（第 {iteration} 次修复）：

    {issues_text}

    要求：
    - 直接修改 payment.py，不要只给建议
    - 保持原有的函数签名和返回格式
    - 修复后代码应该可以正常运行
    """

    # TODO: 实现修复者 Agent
    raise NotImplementedError("请实现 QA 修复者 Agent")


# ============================================================
# TODO 3: 实现递归问题检测
# ============================================================

def detect_recurring_issues(
    current_issues: list,
    history: list[dict],
    threshold: int = 2
) -> list:
    """
    检测在多次迭代中反复出现的问题

    Args:
        current_issues: 本次发现的问题列表
        history: 历史迭代记录 [{"iteration": 1, "issues": [...]}]
        threshold: 出现多少次认为是递归问题

    Returns:
        反复出现的问题列表

    提示：
    - 按 issue["id"] 或 issue["title"] 统计出现次数
    - 出现次数 >= threshold 的就是递归问题
    """

    # TODO: 实现递归检测逻辑
    # 从 history 中提取所有历史 issue 的 title
    # 统计 current_issues 中每个 issue 的历史出现次数
    # 返回超过 threshold 的问题

    raise NotImplementedError("请实现递归问题检测")


# ============================================================
# 主循环（骨架已实现，补全 TODO 部分后即可运行）
# ============================================================

async def run_qa_loop():
    """QA 验证主循环"""
    history = []
    iteration = 0

    print("=" * 60)
    print("QA 验证循环启动")
    print("=" * 60)

    while iteration < MAX_ITERATIONS:
        iteration += 1
        print(f"\n--- 第 {iteration}/{MAX_ITERATIONS} 次迭代 ---")

        # 1. 运行审查者
        print("运行 QA 审查者...")
        status, issues = await run_qa_reviewer(iteration)

        if status == "approved":
            print(f"\n✅ QA 通过！（第 {iteration} 次迭代）")
            return True

        print(f"❌ 发现 {len(issues)} 个问题：")
        for issue in issues:
            print(f"  [{issue['severity'].upper()}] {issue['title']}")

        # 2. 检测递归问题
        recurring = detect_recurring_issues(issues, history)
        if recurring:
            print(f"\n⚠️ 检测到 {len(recurring)} 个递归问题，AI 无法自动解决：")
            for issue in recurring:
                print(f"  - {issue['title']}")
            print("需要人工介入。")
            return False

        # 3. 记录本次迭代
        history.append({"iteration": iteration, "issues": issues})

        if iteration >= MAX_ITERATIONS:
            print("\n⚠️ 达到最大迭代次数，仍有未解决的问题。")
            return False

        # 4. 运行修复者
        print("\n运行 QA 修复者...")
        success = await run_qa_fixer(issues, iteration)
        if not success:
            print("修复失败，终止循环。")
            return False

        print("✅ 修复完成，重新审查...")

    return False


if __name__ == "__main__":
    result = asyncio.run(run_qa_loop())
    print(f"\n最终结果：{'通过' if result else '失败'}")

    # 打印最终的 payment.py
    print("\n=== 最终的 payment.py ===")
    print((WORK_DIR / "payment.py").read_text())
```

---

### 验证标准

- [ ] `payment.py` 的敏感信息（信用卡号）不再直接打印到日志
- [ ] `process_payment` 对负数金额抛出异常
- [ ] 文件写入有 `try/except` 错误处理
- [ ] `refund_payment` 有退款金额与原始金额的校验逻辑
- [ ] 如果手动在文件里重新引入同一个 bug，循环能检测出递归问题并停止

---

### 思考问题

1. **审查者 vs 修复者的 System Prompt 有何不同？** 你用的 Prompt 侧重点是否清晰区分了"发现问题"和"解决问题"？

2. **递归检测的粒度：** 你是用 `issue["id"]` 还是 `issue["title"]` 来做去重？两种方式有什么不同的误判情况？

3. **迭代次数设置：** 你设置的 `MAX_ITERATIONS = 5` 合适吗？设置太小会发生什么？设置太大呢？

4. **观察 Agent 行为：** 在多次迭代中，修复者会不会尝试相同的修复方式？如果会，你会怎么改进？

---

### 进阶挑战

1. **添加自我纠正：** 当审查者产生 `status == "error"` 时（比如 `qa_report.json` 格式不对），把错误信息注入下一次的 Prompt，让它自己纠正

2. **并行审查：** 把审查分成两个并行的子审查者——一个专注安全问题，一个专注代码质量——然后合并结果

3. **历史摘要注入：** 不只是检测递归问题，还要在每次 Fixer 运行时，把上次修复失败的尝试记录注入 Prompt，帮助它换一种方式解决

---

*本章对应的完整代码：[GitHub 仓库 / chapter_02 / lab2/]()*
