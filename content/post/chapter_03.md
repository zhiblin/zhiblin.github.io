---
title: 第三章：Prompt Engineering 的工程化
date: 2026-01-03 00:00:00
categories: Agents
tags: [AI, Agent, Python, Claude, SDK]
---

# 第三章：Prompt Engineering 的工程化

> "写一个好的 Prompt 是艺术；构建一个可维护的 Prompt 系统是工程。"

---

## 3.1 一个工程师看 Prompt 的方式

大多数开发者对 Prompt Engineering 的理解停留在这个层次：

```
"你是一个有帮助的助手……请帮我……"
```

当你把一个任务一次性扔给 LLM，然后祈祷它给出正确答案时，你是在**使用** Prompt。

而当你在管理十几个有明确职责的 Prompt 文件、每个文件定义清晰的输入输出契约、通过 Python 代码动态组装它们、并为每种错误情况设计恢复路径时，你是在**工程化** Prompt。

这一章讲的是后者。

让我们从一个具体的观察开始。打开 Auto-Claude 的 `apps/backend/prompts/` 目录，你会看到：

```
prompts/
├── spec_gatherer.md      # 需求收集智能体
├── spec_researcher.md    # 研究智能体
├── spec_writer.md        # 规格编写智能体
├── spec_critic.md        # 规格批评智能体（用扩展思维）
├── complexity_assessor.md
├── planner.md            # 规划智能体（637 行）
├── coder.md              # 编码智能体（1148 行）
├── coder_recovery.md     # 恢复编码智能体
├── qa_reviewer.md        # QA 审查智能体
├── qa_fixer.md           # QA 修复智能体
├── followup_planner.md
└── github/
    ├── pr_reviewer.md
    ├── pr_orchestrator.md
    └── ...（10+ 个）
```

这不是一堆随意写的提示词。这是一个**受版本控制的提示词系统**，每个文件对应一个有明确角色的智能体，按照严格的工程规范组织。

---

## 3.2 契约模式：每个 Prompt 都是一份 API 规范

打开 `prompts/spec_writer.md` 的开头：

```markdown
## YOUR ROLE - SPEC WRITER AGENT

You are the **Spec Writer Agent** in the Auto-Build spec creation pipeline.
Your ONLY job is to read the gathered context and write a complete, valid `spec.md` document.

**Key Principle**: Synthesize context into actionable spec. No user interaction needed.

---

## YOUR CONTRACT

**Inputs** (read these files):
- `project_index.json` - Project structure
- `requirements.json` - User requirements
- `context.json` - Relevant files discovered

**Output**: `spec.md` - Complete specification document

You MUST create `spec.md` with ALL required sections (see template below).

**DO NOT** interact with the user. You have all the context you need.
```

注意这个结构：**YOUR CONTRACT**。

这不是一个软性建议，而是一份硬性的 API 契约：

```
输入：[project_index.json, requirements.json, context.json]
输出：[spec.md]
副作用：无（不与用户交互）
```

与软件工程中的函数签名相比：

```python
# 函数签名（代码世界）
def write_spec(
    project_index: dict,
    requirements: dict,
    context: dict
) -> SpecDocument:
    ...

# Prompt 中的契约（AI 世界）
"""
Inputs: project_index.json, requirements.json, context.json
Output: spec.md
"""
```

两者在工程意义上是等价的。区别只是前者由 Python 解释器强制执行，后者由 LLM 的遵循能力"执行"——因此 Prompt 中的契约必须写得比代码注释更明确、更强调。

这种契约模式贯穿了所有 Auto-Claude 的 Prompt 文件。每个智能体都明确知道：
1. 它的唯一职责是什么
2. 它应该读取哪些文件
3. 它必须输出什么文件
4. 什么情况下应该停止

---

## 3.3 强制执行 vs 软性建议

这是 Prompt 工程化中最重要的技术之一，却往往被忽视。

来看 `planner.md` 中反复出现的一种模式：

```markdown
## PHASE 3: CREATE implementation_plan.json

**🚨 CRITICAL: YOU MUST USE THE WRITE TOOL TO CREATE THIS FILE 🚨**

You MUST use the Write tool to save the implementation plan to `implementation_plan.json`.
Do NOT just describe what the file should contain - you must actually call the Write tool
with the complete JSON content.

**Required action:** Call the Write tool with:
- file_path: `implementation_plan.json` (in the spec directory)
- content: The complete JSON plan structure shown below
```

还有：

```markdown
**🚨 END OF PHASE 4 CHECKPOINT 🚨**

Before proceeding to PHASE 5, verify you have:
1. ✅ Created the complete implementation_plan.json structure
2. ✅ Used the Write tool to save it (not just described it)
3. ✅ Added the summary section with parallelism analysis

If you have NOT used the Write tool yet, STOP and do it now!
```

为什么需要这么强调？

因为 LLM 天生有一种倾向：**描述行动而不是执行行动**。

如果你只写"请创建 `implementation_plan.json`"，LLM 很可能会：

```
Assistant: 好的，这是 implementation_plan.json 的内容：

```json
{
  "feature": "用户权限管理",
  "phases": [...]
}
```

你可以把这个内容复制到文件里。
```

它**描述**了文件内容，但没有**实际调用** Write 工具写入文件。这是一个功能性失败——后续的 Coder Agent 读不到这个文件，整个流水线就断了。

**解决方案是使用强制执行语言：**

| 软性建议（会被忽略） | 强制执行（有效） |
|---------------------|-----------------|
| "Please create the file" | "You MUST use the Write tool" |
| "You should document..." | "**DO NOT** describe what to write — actually call the Write tool" |
| "Consider adding..." | "🚨 CRITICAL: If you skip this, STOP and do it now" |
| "It would be helpful to..." | "REQUIRED action: Call Write tool with..." |

这不是写作风格问题，是**工程可靠性**问题。在自动化流水线中，任何一步的"软性失败"（LLM 描述了但没执行）都会导致下游静默失败，极难调试。

---

## 3.4 动态 Prompt 组装：从静态文件到参数化模板

Auto-Claude 最精妙的 Prompt 工程之一，在 `prompts_pkg/prompt_generator.py` 中。

这个文件的模块文档说明了其设计动机：

```python
"""
Prompt Generator
================

Generates minimal, focused prompts for each subtask.
Instead of a 900-line mega-prompt, each subtask gets a tailored ~100-line prompt
with only the context it needs.

This approach:
- Reduces token usage by ~80%
- Keeps the agent focused on ONE task
- Moves bookkeeping to Python orchestration
"""
```

**减少 80% 的 Token 消耗**。这不是小事，直接影响运行成本和 LLM 的注意力质量。

来看它是怎么做到的：

### 3.4.1 问题：静态大 Prompt 的弊端

假设 Coder Agent 有一个 1148 行的 `coder.md` 系统提示（事实上确实如此）。如果每个子任务都把这 1148 行完整发给 LLM，那么：

- 前 15 个子任务都在处理"如何在 Worktree 中工作"的安全警告，而这对第 10 个子任务来说毫无意义
- LLM 的注意力被稀释到所有历史子任务的细节上
- 90% 的上下文是无关噪声

### 3.4.2 解决方案：精准注入

`generate_subtask_prompt()` 函数为每个子任务动态生成一个约 100 行的、精准聚焦的 Prompt：

```python
def generate_subtask_prompt(
    spec_dir: Path,
    project_dir: Path,
    subtask: dict,
    phase: dict,
    attempt_count: int = 0,
    recovery_hints: list[str] | None = None,
) -> str:
    """
    Generate a minimal, focused prompt for implementing a single subtask.
    Returns: A focused prompt string (~100 lines instead of 900)
    """
    sections = []

    # 1. 环境上下文（工作目录、规格文件位置）
    sections.append(generate_environment_context(project_dir, spec_dir))

    # 2. 只有这个子任务相关的信息
    sections.append(f"## 你的任务\n{subtask['description']}")

    # 3. 只需要修改的那几个文件（不是整个代码库）
    if files_to_modify:
        sections.append(f"## 要修改的文件\n{files_to_modify}")

    # 4. 参考哪些文件的模式（告诉 Agent 去哪里学）
    if patterns_from:
        sections.append(f"## 参考以下文件的代码模式\n{patterns_from}")

    # 5. 如果是重试，注入上次失败的提示
    if recovery_hints:
        hints = "\n".join(f"- {h}" for h in recovery_hints)
        sections.append(f"## 上次尝试的问题\n{hints}")

    return "\n\n".join(sections)
```

关键设计决策：**只注入当前子任务需要的最小上下文**。

`files_to_modify` 通常只有 1-3 个文件。LLM 不需要知道整个代码库，它只需要知道：
- 这次要改哪里
- 去哪里找参考模式
- 如何验证自己的改动

### 3.4.3 上下文感知：Worktree 隔离警告的动态注入

`generate_environment_context()` 函数展示了另一种动态注入技术：

```python
def generate_environment_context(project_dir: Path, spec_dir: Path) -> str:
    # 检测是否在隔离的 Worktree 中运行
    is_worktree, parent_project_path = detect_worktree_isolation(project_dir)

    sections = []

    # 只有在 Worktree 模式下，才注入这段 500 字的安全警告
    if is_worktree and parent_project_path:
        sections.append(generate_worktree_isolation_warning(
            project_dir, parent_project_path
        ))

    # 基础的环境信息（总是注入）
    sections.append(f"""## YOUR ENVIRONMENT

**Working Directory:** `{project_dir}`
**Spec Location:** `{relative_spec}/`
{"**Isolation Mode:** WORKTREE" if is_worktree else ""}
...
""")
    return "".join(sections)
```

当 Agent 运行在普通目录时，不会看到 Worktree 安全警告；当 Agent 运行在隔离的 Worktree 时，会得到一段明确的警告，包含：
- 禁止访问的父目录路径
- 正确的相对路径示例
- 错误路径示例

这是把**运行时上下文**注入 Prompt 的典型应用。

---

## 3.5 结构化输出：让 LLM 写出可机器解析的数据

自动化流水线的核心挑战：LLM 的输出本质上是自然语言，但流水线的下游需要结构化数据。

Auto-Claude 用三种技术强制 LLM 输出结构化结果：

### 技术一：提供精确的 JSON Schema

`spec_gatherer.md` 里对输出格式的要求：

```markdown
You MUST create `requirements.json` with this EXACT structure:

```json
{
  "task_description": "Clear description of what to build",
  "workflow_type": "feature|refactor|investigation|migration|simple",
  "services_involved": ["service1", "service2"],
  "user_requirements": ["Requirement 1", "Requirement 2"],
  "acceptance_criteria": ["Criterion 1", "Criterion 2"],
  "constraints": ["Any constraints or limitations"],
  "created_at": "ISO timestamp"
}
```

**DO NOT** proceed without creating this file.
```

不只是说"请输出 JSON"，而是给出：
- 字段名（精确到大小写）
- 字段类型（字符串 vs 字符串数组）
- 枚举值（`"feature|refactor|investigation|migration|simple"`）
- 强制要求（`EXACT structure`，`DO NOT proceed without creating`）

### 技术二：文件写入强制

注意关键指令的措辞差异：

```markdown
# 弱指令（容易产生"仅文本描述"的失败）
Create a JSON file with the following content...

# 强指令（强制调用 Write 工具）
You MUST create `requirements.json`. DO NOT just show the content —
use the Write tool to actually create the file.
```

还有 `spec_critic.md` 中更直接的方式：

```bash
cat > critique_report.json << 'EOF'
{
  "critique_completed": true,
  "issues_found": [...]
}
EOF
```

直接在 Prompt 里写出 bash 命令，让 LLM 执行这个命令来创建文件，而不是通过 Write 工具——同样是"强制执行"思想，但用了 bash heredoc 的方式。

### 技术三：SDK 原生结构化输出

Auto-Claude 的 `create_client()` 支持传入 `output_format` 参数：

```python
# core/client.py
def create_client(
    ...
    output_format: dict | None = None,  # 结构化输出格式
):
    ...
    options_kwargs = {
        ...
    }
    if output_format:
        options_kwargs["output_format"] = output_format
```

使用方式：

```python
from pydantic import BaseModel

class QAResult(BaseModel):
    status: Literal["approved", "rejected"]
    issues: list[dict]
    timestamp: str

# 创建强制输出 Pydantic 模型的 Client
client = create_client(
    ...,
    output_format={
        "type": "json_schema",
        "schema": QAResult.model_json_schema()
    }
)
```

这是 Claude Agent SDK 的原生功能：当你提供 `output_format`，SDK 会在底层强制 LLM 遵循该 Schema 输出，违反 Schema 的响应会被自动重试。这比在 Prompt 里写"请输出 JSON"可靠得多。

---

## 3.6 自我批评循环：让 AI 检查 AI

`spec_critic.md` 是一个独特的角色——它的全部工作是找另一个 AI（Spec Writer）的错误：

```markdown
## YOUR ROLE - SPEC CRITIC AGENT

You are the **Spec Critic Agent** in the Auto-Build spec creation pipeline.
Your ONLY job is to critically review the spec.md document, find issues, and fix them.

**Key Principle**: Use extended thinking (ultrathink). Find problems BEFORE implementation.
```

注意 `ultrathink` 这个关键词——这个 Agent 使用扩展思维（Extended Thinking），给它更多的推理预算来找到深层问题。

它的批评框架分为五个维度：

```
1.1 技术准确性：包名对吗？API 签名正确吗？
    → 用 Context7 工具验证第三方库的实际 API

1.2 完整性：所有需求都覆盖了吗？验收标准可测试吗？

1.3 一致性：同一概念的术语统一吗？文件路径没有冲突吗？

1.4 可行性：依赖包都存在吗？实现顺序合理吗？

1.5 研究对齐：与 research.json 的发现是否一致？
```

**为什么需要独立的 Critic Agent，而不是让 Writer Agent 自我检查？**

这是认知科学中的"作者盲区"问题：写出文档的人，往往看不到自己的错误，因为大脑会自动"补全"它认为应该在那里的内容。

在 AI 系统中，同样的问题存在：如果你要求 Spec Writer "检查你刚写的内容"，它倾向于确认而非质疑。但一个全新启动、没有写过这份 Spec 的 Critic Agent，会以真正批判的眼光审查。

这个模式叫做**对抗性协作**（Adversarial Collaboration）：让两个 AI 扮演对立角色，Writer 试图通过 Critic 的审查，Critic 试图找到 Writer 的漏洞。

---

## 3.7 模板注入：动态的 QA 能力扩展

`qa_reviewer.md` 里有一段不寻常的注释：

```markdown
<!-- PROJECT-SPECIFIC VALIDATION TOOLS WILL BE INJECTED HERE -->
<!-- The following sections are dynamically added based on project type: -->
<!-- - Electron validation (for Electron apps) -->
<!-- - Puppeteer browser automation (for web frontends) -->
<!-- - Database validation (for projects with databases) -->
<!-- - API validation (for projects with API endpoints) -->
```

这是**运行时模板注入**（Runtime Template Injection）。

QA Reviewer 的基础 Prompt 包含通用的验证逻辑（运行测试、检查代码风格）。但对不同类型的项目，额外的验证步骤会动态注入：

- Electron 应用：注入截图验证步骤（使用 `mcp__electron__take_screenshot`）
- Web 前端：注入 Puppeteer 浏览器自动化步骤
- 有数据库的项目：注入数据库迁移验证步骤
- 有 API 的项目：注入 API 端点验证步骤

```python
# 注入逻辑（简化）
def build_qa_prompt(spec_dir, project_dir):
    base_prompt = load_file("prompts/qa_reviewer.md")
    capabilities = detect_project_capabilities(project_dir)

    injections = []
    if capabilities["has_electron"]:
        injections.append(load_file("prompts/mcp_tools/electron_validation.md"))
    if capabilities["has_database"]:
        injections.append(load_file("prompts/mcp_tools/database_validation.md"))
    if capabilities["has_api"]:
        injections.append(load_file("prompts/mcp_tools/api_validation.md"))

    # 把注入内容替换到模板中的注释位置
    injection_text = "\n\n".join(injections)
    return base_prompt.replace(
        "<!-- PROJECT-SPECIFIC VALIDATION TOOLS WILL BE INJECTED HERE -->",
        injection_text
    )
```

这种技术的好处是：
- 基础 Prompt 保持简洁（不把所有情况都堆在一起）
- 每个注入模块可以独立维护和测试
- 不相关的验证步骤不会占用 QA Agent 的上下文窗口

---

## 3.8 阶段门控：用 Prompt 实现状态机

QA Reviewer 的 Prompt 里有一段看似普通的文字，实际上在实现一个**状态门控**：

```markdown
## PHASE 1: VERIFY ALL SUBTASKS COMPLETED

```bash
echo "Completed: $(grep -c '"status": "completed"' implementation_plan.json)"
echo "Pending: $(grep -c '"status": "pending"' implementation_plan.json)"
```

**STOP if subtasks are not all completed.**
You should only run after the Coder Agent marks all subtasks complete.
```

`STOP if subtasks are not all completed`——这在 Prompt 里实现了一个前置条件检查。

在传统软件工程中，这是一个 `assert`：
```python
assert all_subtasks_complete(), "QA should only run after all subtasks are done"
```

在 Prompt 工程中，同样的约束用自然语言实现：
```
STOP if subtasks are not all completed
```

Auto-Claude 把这种技术用得很系统化。Coder Agent 的 Prompt 里有：

```markdown
## STEP 3: FIND YOUR NEXT SUBTASK

**CRITICAL**: Never work on a subtask if its phase's dependencies aren't complete!

Phase 1: Backend     [depends_on: []]           → Can start immediately
Phase 2: Worker      [depends_on: ["phase-1"]]  → Blocked until Phase 1 done
Phase 3: Frontend    [depends_on: ["phase-1"]]  → Blocked until Phase 1 done
```

这在 Prompt 里实现了依赖关系检查，让 LLM 自己判断哪些子任务可以开始执行。

---

## 3.9 Prompt 的版本控制与测试

把 Prompt 当代码处理，意味着它们需要版本控制和测试。

Auto-Claude 对 Prompt 的管理方式：

**版本控制：** 所有 Prompt 文件都在 git 中，`git blame` 可以追溯每次改动的原因：
```
$ git log --oneline apps/backend/prompts/coder.md
60c4890 Improved prompt for Opus 4.6
a3f2d18 fix: Add PATH CONFUSION PREVENTION section
7b91c4d feat: Add pre-implementation checklist generation
```

**隔离测试：** 每个 Prompt 文件只负责一个 Agent 类型，可以单独测试，不需要启动完整的 Auto-Claude 系统。

**Prompt 变更的影响评估：** 修改 `coder.md` 可能影响所有 Coder Agent，这是高风险变更。修改 `spec_critic.md` 只影响 Spec 创建阶段，风险相对较低。

这种影响范围的可预测性，正是把 Prompt 按角色分文件管理的重要原因之一。

---

## 3.10 七条 Prompt 工程实践原则

基于 Auto-Claude 的源码，总结七条可直接应用的工程实践：

| # | 原则 | 错误示例 | 正确示例 |
|---|------|----------|----------|
| 1 | **契约优先** | "请帮我分析项目并输出结果" | "输入：X，输出：Y，不执行其他操作" |
| 2 | **强制执行** | "你可以创建文件来保存结果" | "你必须用 Write 工具创建文件，不要只描述内容" |
| 3 | **精准注入** | 把所有上下文都塞进去 | 只注入当前任务需要的最小上下文 |
| 4 | **Schema 驱动** | "请输出 JSON 格式" | 提供精确的字段名、类型、枚举值 |
| 5 | **阶段门控** | 让 Agent 自己判断何时继续 | 在 Prompt 里写明前置条件检查 |
| 6 | **对抗批评** | Writer 自我检查 | 独立 Critic Agent，全新上下文 |
| 7 | **模板注入** | 一个巨大的通用 Prompt | 基础 Prompt + 运行时注入项目特定内容 |

```
★ Insight ─────────────────────────────────────────────
• "契约模式"是 Prompt 工程化的基础：
  用 YOUR CONTRACT 明确 Inputs/Outputs，
  与软件工程中的函数签名等价

• 强制执行 vs 软性建议的工程意义：
  在自动化流水线中，软性失败（LLM 描述了但没执行）
  会导致下游静默失败，是最难调试的 bug 类型

• 动态 Prompt 组装的关键洞察：
  "900 行 mega-prompt" → "100 行精准 prompt"
  减少 80% Token，但提高了注意力质量
  关键是把 bookkeeping 移到 Python 代码层

• 自我批评的认知科学基础：
  Writer Agent 有"作者盲区"——它会补全它认为应该有的内容
  独立 Critic Agent 打破这种确认偏误
  这是 AI 系统质量保证的核心机制之一
─────────────────────────────────────────────────────────
```

---

## Lab 3.1：设计一个预实现检查清单生成器

**目标：** 实现一个 Prompt 系统，在编写任何代码前强制 LLM 输出风险分析清单，并将清单写入文件。

这个 Lab 练习以下三个工程技术：
1. **契约模式**：明确 Inputs/Outputs
2. **强制执行**：必须用 Write 工具写文件
3. **结构化输出**：输出可解析的 JSON

---

### 背景：为什么需要"预实现检查清单"

Auto-Claude 的 `coder.md` 里有这样一段：

```markdown
## STEP 5.5: GENERATE & REVIEW PRE-IMPLEMENTATION CHECKLIST

**CRITICAL**: Before writing any code, generate a predictive bug prevention checklist.
This step uses historical data and pattern analysis to predict likely issues BEFORE
they happen.
```

在写代码之前，先让 AI 预测可能出现的问题，能显著减少调试时间。这个 Lab 就是实现这个机制的简化版。

---

### 任务描述

你的任务是实现 `ChecklistGenerator`，它：

1. 接收一个子任务描述（字符串）
2. 启动一个 Agent，分析这个子任务可能的风险
3. **强制** Agent 把分析结果写入 `checklist.json` 文件
4. 从文件读取并返回解析后的清单

清单的 JSON 格式：
```json
{
  "subtask": "实现用户登录 API",
  "risk_level": "high",
  "predicted_issues": [
    {
      "category": "security",
      "issue": "密码明文存储",
      "prevention": "使用 bcrypt 哈希，不存储原始密码"
    },
    {
      "category": "validation",
      "issue": "没有限制登录尝试次数",
      "prevention": "添加速率限制，防止暴力破解"
    }
  ],
  "must_verify_before_submit": [
    "密码使用哈希存储",
    "JWT token 有过期时间",
    "SQL 查询使用参数化，无注入风险"
  ]
}
```

---

### 骨架代码

```python
"""
Lab 3.1 - 预实现检查清单生成器

工程目标：
1. 练习契约模式（明确 Inputs/Outputs）
2. 练习强制执行语言（LLM 必须写文件）
3. 练习结构化输出（JSON Schema 驱动）
"""
import asyncio
import json
from pathlib import Path
from claude_agent_sdk import ClaudeSDKClient, ClaudeAgentOptions

WORK_DIR = Path("./lab3_workspace")
WORK_DIR.mkdir(exist_ok=True)
CHECKLIST_FILE = WORK_DIR / "checklist.json"


# ============================================================
# TODO 1: 设计系统提示词（System Prompt）
# 要求：体现"契约模式"和"强制执行"原则
# ============================================================

CHECKLIST_SYSTEM_PROMPT = """
你是一名代码风险分析专家。

## YOUR CONTRACT

**Input**: 一段子任务描述（通过 user message 提供）
**Output**: checklist.json 文件（必须使用 Write 工具创建）

## 关键规则

TODO: 写出体现以下原则的规则：
1. 强制执行：LLM 必须调用 Write 工具，不能只描述内容
2. 精确的输出格式：提供精确的 JSON Schema
3. 阶段门控：分析完成后必须写文件，然后停止

## 输出格式

TODO: 在这里放入精确的 JSON Schema，包含：
- subtask (string): 子任务描述
- risk_level (string): "low" | "medium" | "high" | "critical"
- predicted_issues (array): 预测的问题列表
  - category (string): "security" | "validation" | "error_handling" | "performance"
  - issue (string): 问题描述
  - prevention (string): 预防方法
- must_verify_before_submit (array of strings): 提交前必须验证的事项

## 强制执行

TODO: 写出强制 LLM 调用 Write 工具的指令
记住：不能只展示 JSON 内容，必须实际写入文件
"""


# ============================================================
# TODO 2: 实现 generate_checklist() 函数
# ============================================================

async def generate_checklist(subtask_description: str) -> dict:
    """
    为给定的子任务生成预实现风险清单。

    工作流：
    1. 启动 Agent，让它分析 subtask_description
    2. Agent 必须将分析结果写入 checklist.json
    3. 读取 checklist.json 并返回解析后的字典

    Args:
        subtask_description: 子任务的文字描述

    Returns:
        解析后的清单字典

    Raises:
        FileNotFoundError: 如果 Agent 没有写入文件（表明强制执行失败）
        json.JSONDecodeError: 如果 Agent 写入的不是有效 JSON
    """

    # 在开始之前，清理上次运行的文件
    if CHECKLIST_FILE.exists():
        CHECKLIST_FILE.unlink()

    # TODO: 创建 ClaudeAgentOptions
    # 提示：
    # - model: "claude-haiku-4-5-20251001"（节约成本）
    # - system_prompt: 使用上面定义的 CHECKLIST_SYSTEM_PROMPT
    # - allowed_tools: 至少需要 ["Write"]（思考：还需要别的吗？）
    # - cwd: str(WORK_DIR)
    # - max_turns: 10（这个任务不需要很多轮）
    options = ClaudeAgentOptions(
        model="claude-haiku-4-5-20251001",
        system_prompt=CHECKLIST_SYSTEM_PROMPT,
        allowed_tools=["Write"],  # TODO: 这就够了吗？
        cwd=str(WORK_DIR),
        max_turns=10,
    )

    client = ClaudeSDKClient(options=options)

    # TODO: 构建 user prompt
    # 提示：要包含子任务描述，以及输出文件路径
    user_prompt = f"""
    TODO: 告诉 Agent 要分析什么，以及把结果写到哪个文件
    子任务描述：{subtask_description}
    """

    # 运行 Agent
    print(f"分析子任务：{subtask_description[:50]}...")
    async for message in client.process_query(user_prompt):
        msg_type = type(message).__name__
        # TODO: 打印有意义的进度信息
        # 提示：可以只打印 ToolUseBlock 类型的消息，查看 Agent 调用了什么工具
        pass

    # TODO: 验证 Agent 是否实际写入了文件
    # 如果文件不存在，这是强制执行失败的信号
    if not CHECKLIST_FILE.exists():
        raise FileNotFoundError(
            f"Agent 没有创建 {CHECKLIST_FILE}。\n"
            f"这说明 System Prompt 的强制执行语言不够强硬。\n"
            f"请修改 CHECKLIST_SYSTEM_PROMPT，让 LLM 明确知道必须调用 Write 工具。"
        )

    # TODO: 读取并解析文件
    content = CHECKLIST_FILE.read_text(encoding="utf-8")
    return json.loads(content)


# ============================================================
# TODO 3: 实现 validate_checklist() 验证函数
# ============================================================

def validate_checklist(checklist: dict) -> tuple[bool, list[str]]:
    """
    验证清单格式是否符合预期 Schema。

    Returns:
        (is_valid, error_messages)
    """
    errors = []

    # TODO: 验证以下字段存在且类型正确：
    # - subtask (str)
    # - risk_level (str, 必须是 "low"/"medium"/"high"/"critical" 之一)
    # - predicted_issues (list, 至少包含 1 个元素)
    #   - 每个元素有 category, issue, prevention 字段
    # - must_verify_before_submit (list of str)

    return len(errors) == 0, errors


# ============================================================
# 测试代码（补全上面的 TODO 后运行）
# ============================================================

async def main():
    test_subtasks = [
        "实现用户登录 API：接受 username/password，返回 JWT token",
        "添加文件上传功能：支持上传图片，最大 10MB",
        "实现支付接口：调用 Stripe API 处理信用卡支付",
    ]

    for subtask in test_subtasks:
        print(f"\n{'='*60}")
        print(f"子任务：{subtask}")
        print('='*60)

        try:
            checklist = await generate_checklist(subtask)

            # 验证输出格式
            is_valid, errors = validate_checklist(checklist)

            if is_valid:
                print(f"✅ 风险级别：{checklist['risk_level'].upper()}")
                print(f"预测问题数：{len(checklist['predicted_issues'])}")
                for issue in checklist['predicted_issues']:
                    print(f"  [{issue['category']}] {issue['issue']}")
                    print(f"    → 预防：{issue['prevention']}")
                print(f"\n提交前验证项：")
                for item in checklist['must_verify_before_submit']:
                    print(f"  ☐ {item}")
            else:
                print(f"❌ 格式验证失败：")
                for error in errors:
                    print(f"  - {error}")

        except FileNotFoundError as e:
            print(f"❌ 强制执行失败：\n{e}")
        except json.JSONDecodeError as e:
            print(f"❌ JSON 解析失败：{e}")
            print(f"Agent 写入的内容：{CHECKLIST_FILE.read_text()[:200]}")


if __name__ == "__main__":
    asyncio.run(main())
```

---

### 验证标准

- [ ] 三个测试用例都生成了 `checklist.json` 文件（证明强制执行生效）
- [ ] 所有输出文件都是有效 JSON，符合 Schema
- [ ] "支付接口"子任务的 `risk_level` 是 `"high"` 或 `"critical"`
- [ ] "用户登录 API"的 `predicted_issues` 包含安全相关问题（如 SQL 注入、密码存储）
- [ ] 修改 System Prompt，把"必须"改成"建议"——观察强制执行失败率的变化

---

### 思考问题

1. **强制执行实验：** 把 System Prompt 里的 `"You MUST use the Write tool"` 改成 `"Please create the file"`，重新运行。Agent 还会写文件吗？失败率是多少？

2. **Schema 精确度：** 如果你的 JSON Schema 只写 `"issues": []` 而不指定每个元素的字段，LLM 输出的格式会更随意吗？

3. **Token 效率：** 用 `claude-haiku-4-5-20251001` 和 `claude-sonnet-4-6` 各跑一次，比较输出质量和 Token 消耗。对于这个任务，Haiku 够用吗？

---

### 进阶挑战

1. **历史数据注入：** 把之前任务生成的清单保存起来，在下次分析类似任务时，把相关的历史清单注入 Prompt，让 LLM 参考历史数据生成更准确的预测

2. **SDK 结构化输出：** 用 `create_client()` 的 `output_format` 参数传入 Pydantic Schema，替代 Prompt 中的强制执行语言——比较两种方式的可靠性差异

3. **Critic Agent 模式：** 生成清单后，再启动第二个 Agent，让它批评第一个 Agent 的清单——是否遗漏了重要的风险？这个模式有何代价？

---

*本章对应的完整代码：[GitHub 仓库 / chapter_03 / lab3/]()*
