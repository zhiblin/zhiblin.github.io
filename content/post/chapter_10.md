---
title: 第十章：规格创建流水线
date: 2026-01-10 00:00:00
categories: Agents
tags: [AI, Agent, Python, Claude, SDK]
---

# 第十章：规格创建流水线

> "用户描述的是意图，智能体需要的是规格。把一句话变成一份详细的实现计划，本身就是一个需要多阶段智能协作的工程问题。"

---

## 10.1 为什么需要规格创建流水线？

想象一下直接让一个 AI 智能体执行"给我加一个用户登录功能"。智能体面临一大堆未解决的问题：

- 这个项目已经有认证系统了吗？用的是哪种方案？
- 需要 OAuth 还是用户名密码？要不要两步验证？
- 前端登录页已经存在了吗？需要修改哪些文件？
- 错误处理逻辑应该是什么样的？
- 这个功能涉及几个服务？有没有外部依赖？

如果直接把这个模糊的需求交给编码智能体，它要么会做出大量假设（可能全部错误），要么会在中途卡住。

Auto-Claude 用一个多阶段的**规格创建流水线**（Spec Creation Pipeline）来解决这个问题：在任何编码工作开始之前，先把用户的模糊意图转化为一份结构化的规格文档（`spec.md`）和实现计划（`implementation_plan.json`）。

这个流水线的核心设计理念是**动态适应复杂度**：简单任务走简单路径，复杂任务才需要研究、自我批判等重量级阶段。

---

## 10.2 三种复杂度档位

`apps/backend/spec/complexity.py` 定义了任务复杂度的评估体系：

```python
# apps/backend/spec/complexity.py（简化）

class Complexity(Enum):
    SIMPLE   = "simple"    # 1-2 个文件，单一服务，无外部集成
    STANDARD = "standard"  # 3-10 个文件，1-2 个服务，少量集成
    COMPLEX  = "complex"   # 10+ 个文件，多个服务，外部集成
```

不同复杂度对应不同的阶段组合：

```python
def phases_to_run(self) -> list[str]:
    """根据复杂度返回需要执行的阶段列表。"""
    # 如果 AI 给出了推荐阶段，优先使用
    if self.recommended_phases:
        return self.recommended_phases

    if self.complexity == Complexity.SIMPLE:
        return ["discovery", "historical_context", "quick_spec", "validation"]

    elif self.complexity == Complexity.STANDARD:
        phases = ["discovery", "historical_context", "requirements"]
        if self.needs_research:
            phases.append("research")  # 按需启用研究阶段
        phases.extend(["context", "spec_writing", "planning", "validation"])
        return phases

    else:  # COMPLEX
        return [
            "discovery", "historical_context", "requirements",
            "research", "context", "spec_writing",
            "self_critique",   # 只有 COMPLEX 才默认运行自我批判
            "planning", "validation",
        ]
```

这种设计实现了**成本与质量的动态平衡**。一个"修复拼写错误"的任务不需要经过 9 个阶段；但一个"集成 Stripe 支付"的任务则需要研究现有代码、分析集成方案，并通过自我批判循环确保规格的准确性。

---

## 10.3 复杂度评估：AI 与启发式的双轨制

`ComplexityAnalyzer` 实现了基于关键词的启发式评估：

```python
# apps/backend/spec/complexity.py（简化）

class ComplexityAnalyzer:
    # 暗示简单任务的关键词
    SIMPLE_KEYWORDS = ["fix", "typo", "update", "change", "rename", "rename",
                       "style", "color", "text", "button", "margin", "padding"]

    # 暗示复杂任务的关键词
    COMPLEX_KEYWORDS = ["integrate", "api", "sdk", "database", "migrate",
                        "docker", "authentication", "oauth", "microservice",
                        "refactor", "architecture"]

    def _calculate_complexity(self, signals, integrations,
                               infra_changes, estimated_files, estimated_services):
        """根据所有信号计算最终复杂度。"""
        # SIMPLE 的强指标
        if (estimated_files <= 2 and estimated_services == 1
                and len(integrations) == 0 and not infra_changes
                and signals["simple_keywords"] > 0
                and signals["complex_keywords"] == 0):
            return Complexity.SIMPLE, 0.9, "Single service, minimal scope"

        # COMPLEX 的强指标
        if (len(integrations) >= 2 or infra_changes
                or estimated_services >= 3 or estimated_files >= 10
                or signals["complex_keywords"] >= 3):
            return Complexity.COMPLEX, 0.85, f"{len(integrations)} integrations detected"

        # 默认 STANDARD
        return Complexity.STANDARD, 0.75, f"{estimated_files} files estimated"
```

然而，关键词匹配有局限性——"fix authentication bug"这个描述可能触发 `authentication` 关键词而被错误标记为 COMPLEX。因此 Auto-Claude 提供了 AI 评估作为首选：

```python
# apps/backend/spec/pipeline/orchestrator.py（简化）

async def _phase_complexity_assessment_with_requirements(self):
    if self.complexity_override:
        # 手动强制指定复杂度
        self.assessment = self._create_override_assessment()
    elif self.use_ai_assessment:
        # AI 评估（默认）
        self.assessment = await self._run_ai_assessment(task_logger)
    else:
        # 启发式评估（兜底）
        self.assessment = self._heuristic_assessment()
```

AI 评估通过运行 `complexity_assessor.md` 提示词，让 AI 读取 `requirements.json` 后给出结构化的评估结果（JSON 格式），包括 `complexity`、`recommended_phases`、`flags.needs_research` 等字段。这是比关键词匹配更可靠的判断，但在 AI 评估失败时自动退回到启发式方法。

`★ Insight ─────────────────────────────────────`
**双轨制**（AI 首选 + 启发式兜底）是 AI 系统工程中的重要模式。AI 方法更准确但有失败风险；规则方法较粗糙但极其可靠。Auto-Claude 总是先尝试 AI，失败时优雅降级（graceful degradation）到规则，而非直接报错中断。
`─────────────────────────────────────────────────`

---

## 10.4 完整流水线：`SpecOrchestrator`

`apps/backend/spec/pipeline/orchestrator.py` 中的 `SpecOrchestrator._run_phases()` 是流水线的总调度器。理解这个函数的结构，就理解了整个规格创建流程：

```
SpecOrchestrator.run()
    │
    ├── Phase 1: DISCOVERY
    │   ├── 扫描项目文件（生成 project_index.json）
    │   └── 统计文件数量、语言分布、目录结构
    │
    ├── Phase 2: REQUIREMENTS GATHERING
    │   ├── 交互式（interactive=True）：AI 与用户对话，澄清需求
    │   └── 非交互式：从 task_description 直接生成 requirements.json
    │       ↓ 生成 requirements.json：
    │         { task_description, workflow_type, services_involved,
    │           user_requirements, acceptance_criteria, constraints }
    │   ↓ 重命名 spec 目录（pending → 000-add-login-feature）
    │
    ├── Phase 3: AI COMPLEXITY ASSESSMENT
    │   └── AI 读取 requirements.json → complexity_assessment.json
    │       → 决定后续执行哪些阶段
    │
    ├── [Phase 4: HISTORICAL CONTEXT]（如果启用 Graphiti）
    │   └── 从知识图谱查询相关历史经验
    │
    ├── [Phase 5: RESEARCH]（STANDARD/COMPLEX）
    │   └── AI 搜索并阅读相关代码文件
    │
    ├── [Phase 6: CONTEXT BUILDING]（STANDARD/COMPLEX）
    │   └── 构建精准的任务上下文
    │
    ├── Phase 7: SPEC WRITING
    │   ├── SIMPLE → quick_spec（1 个阶段：spec + plan 一起生成）
    │   └── STANDARD/COMPLEX → 单独的 spec_writing 阶段
    │       生成 spec.md
    │
    ├── [Phase 8: SELF CRITIQUE]（仅 COMPLEX）
    │   └── AI 批判自己写的 spec，并修改
    │
    ├── Phase 9: PLANNING
    │   └── 生成 implementation_plan.json（阶段 + 子任务 + 依赖）
    │
    ├── Phase 10: VALIDATION
    │   └── 验证规格完整性（SpecValidator）
    │
    └── Human Review Checkpoint
        └── 等待用户批准后才开始编码
```

### 10.4.1 阶段容错与重试

每个阶段都包裹在重试循环中：

```python
# apps/backend/spec/phases/models.py（简化）

MAX_RETRIES = 3  # 每个阶段最多重试 3 次

@dataclass
class PhaseResult:
    phase_name: str
    success: bool
    output_files: list[str]   # 本阶段产出的文件
    errors: list[str]         # 遇到的错误
    retries: int              # 实际重试次数
```

如果一个阶段失败了（如规格文件没有生成），它会重试最多 3 次，然后才宣告整个流水线失败。失败时还会触发 `PLANNING_FAILED` 事件，确保前端 XState 状态机正确转移，避免任务永远"卡在规划中"。

### 10.4.2 阶段摘要与上下文压缩

流水线运行到后期阶段时，早期阶段的完整输出可能已经超出了单次 AI 调用的上下文窗口。`_store_phase_summary()` 在每个阶段完成后生成摘要：

```python
# apps/backend/spec/pipeline/orchestrator.py（简化）

async def _store_phase_summary(self, phase_name: str) -> None:
    """将已完成阶段的输出压缩为摘要，供后续阶段使用。"""
    # 收集本阶段产出的所有文件内容
    phase_output = gather_phase_outputs(self.spec_dir, phase_name)

    # 用 AI 将其压缩为约 500 字的摘要
    summary = await summarize_phase_output(
        phase_name, phase_output,
        model="sonnet",
        target_words=500,
    )

    if summary:
        self._phase_summaries[phase_name] = summary

# 在构建每个 Agent 的 prompt 时注入历史阶段摘要
prior_summaries = format_phase_summaries(self._phase_summaries)
return await runner.run_agent(
    prompt_file,
    additional_context=additional_context,
    prior_phase_summaries=prior_summaries,  # 注入历史阶段信息
    ...
)
```

这是**分级上下文管理**的典范：完整的原始输出保存在文件中（供需要时使用），而传给 AI 的是压缩后的摘要（节省 token）。

---

## 10.5 自我批判循环的工程实现

`apps/backend/spec/critique.py` 实现了一个精妙的结构：

```python
# apps/backend/spec/critique.py（简化）

def generate_critique_prompt(subtask, files_modified, patterns_from) -> str:
    """生成让 AI 自我评估的提示词。"""
    return f"""
## MANDATORY Self-Critique: {subtask_id}

### STEP 1: Code Quality Checklist
- [ ] 遵循参考文件的代码模式
- [ ] 变量命名符合代码库惯例
- [ ] 没有遗留调试语句

### STEP 2: Implementation Completeness
Expected files: {files_to_modify}
Actual modified: {files_modified}
- [ ] 所有预期文件都已修改

### STEP 3: Potential Issues Analysis
[列出问题]

### STEP 4: Improvements Made
[已修复的问题]

### STEP 5: Final Verdict
**PROCEED:** [YES/NO]
**REASON:** [...]
**CONFIDENCE:** [High/Medium/Low]
"""
```

注意提示词的结构设计：它不只是说"请检查你的代码"，而是提供了一个 5 步检查清单，每步都有具体的验证项目。这是**结构化输出的反面用法**——不是让 AI 输出结构化数据，而是用结构强制 AI 进行系统性思考。

解析函数 `parse_critique_response()` 使用正则表达式从响应中提取关键信息：

```python
# apps/backend/spec/critique.py（简化）

def parse_critique_response(response: str) -> CritiqueResult:
    # 提取 PROCEED 判决
    proceed_match = re.search(
        r"\*\*PROCEED:\*\*\s*\[?\s*(YES|NO)",
        response, re.IGNORECASE
    )
    passes = proceed_match.group(1).upper() == "YES"

    # 提取 Step 3 中列出的问题
    issues_section = re.search(
        r"### STEP 3:.*?Potential Issues.*?\n\n(.*?)(?=###|\Z)",
        response, re.DOTALL
    )
    # 解析每行问题，过滤掉占位符文本...

    return CritiqueResult(passes=passes, issues=issues, ...)
```

以及判断是否可以继续的逻辑：

```python
def should_proceed(result: CritiqueResult) -> bool:
    """只有 PROCEED=YES 且没有未解决问题，才允许标记完成。"""
    if not result.passes:
        return False
    if result.issues:   # 找到问题但没修复
        return False
    return True
```

`★ Insight ─────────────────────────────────────`
注意最后一行注释 `"Remember: The next session has no context. Quality issues you miss now will be harder to fix later."`——提示词直接向 AI 解释了为什么自我批判是必要的。这是一种**元认知引导**技术：向 AI 解释"为什么"，而不仅仅是"做什么"，往往能得到更认真的执行。
`─────────────────────────────────────────────────`

---

## 10.6 Greenfield 项目的特殊处理

当项目目录为空（新项目从零开始）时，流水线需要特殊适应：

```python
# apps/backend/spec/phases/spec_phases.py（简化）

def _greenfield_context() -> str:
    """为空目录项目返回额外上下文。"""
    return """
**GREENFIELD PROJECT**: 这是一个空的新项目，没有现有代码。

调整你的方法：
- 不要引用现有文件、模式或代码结构
- 聚焦于需要创建什么，而非修改什么
- 定义初始项目结构、文件和目录
- 指定技术栈、框架和依赖项安装方式
- 对于"参考文件"部分，描述行业最佳实践而非现有代码
"""

async def phase_quick_spec(self) -> PhaseResult:
    is_greenfield = _is_greenfield_project(self.spec_dir)

    context_str = f"""
**Task**: {self.task_description}
**Complexity**: SIMPLE
{_greenfield_context() if is_greenfield else ""}
"""
    success, output = await self.run_agent_fn(
        "spec_quick.md",
        additional_context=context_str,
    )
```

这个设计体现了**上下文感知的提示词构建**：基础提示词不变，根据运行时检测的项目状态动态注入额外指令。

---

## 10.7 人工审查检查点

规格创建完成后，流水线不会自动启动编码。它会暂停在一个**人工审查检查点**：

```python
# apps/backend/spec/pipeline/orchestrator.py（简化）

def _run_review_checkpoint(self, auto_approve: bool) -> bool:
    """等待用户批准规格后再开始编码。"""
    try:
        review_state = run_review_checkpoint(
            spec_dir=self.spec_dir,
            auto_approve=auto_approve,  # 可通过 CLI 参数跳过
        )

        if not review_state.is_approved():
            print_status("Build will not proceed without approval.", "warning")
            return False

    except KeyboardInterrupt:
        print_status("Review interrupted. Run again to continue.", "info")
        return False

    return True
```

这个检查点确保：AI 生成的规格在编码开始前经过人工验证。用户可以在此阶段修改 `spec.md`（比如纠正误解的需求），然后手动批准。这是 AI 系统中至关重要的**人在环路**（Human-in-the-Loop）设计。

---

## 10.8 Spec 目录的原子命名

细心的读者可能注意到，Spec 目录在创建时被命名为 `pending`，在需求收集完成后才被重命名为有意义的名字（如 `000-add-login-feature`）：

```python
# apps/backend/spec/pipeline/orchestrator.py（简化）

# requirements.json 生成后，用任务描述重命名目录
new_spec_dir = rename_spec_dir_from_requirements(self.spec_dir)
if new_spec_dir != self.spec_dir:
    self.spec_dir = new_spec_dir
    # 同步更新所有引用了旧路径的对象
    self.validator = SpecValidator(self.spec_dir)
    phase_executor.spec_dir = self.spec_dir
    phase_executor.spec_validator = self.validator
```

这里使用了 `SpecNumberLock` 确保多个并行流水线不会分配到相同的序号：

```python
# 在 SpecNumberLock 作用域内创建目录，确保序号唯一
with SpecNumberLock(self.project_dir) as lock:
    self.spec_dir = create_spec_dir(self.specs_dir, lock)
    self.spec_dir.mkdir(parents=True, exist_ok=True)
```

这是分布式系统中**锁保护的资源分配**模式的简化版本。

---

## 10.9 各阶段产出文件速查

| 阶段 | 产出文件 | 作用 |
|------|---------|------|
| discovery | `project_index.json` | 项目文件结构和统计 |
| requirements | `requirements.json` | 结构化的任务需求 |
| complexity_assessment | `complexity_assessment.json` | 复杂度等级和阶段配置 |
| historical_context | `graph_hints.json` | 从知识图谱获取的历史提示 |
| research | `research.md` | 对相关代码的深度分析 |
| context | `context.json` | 精选的任务相关文件列表 |
| spec_writing | `spec.md` | 完整规格文档 |
| planning | `implementation_plan.json` | 阶段化子任务执行计划 |

---

## 本章要点回顾

| 概念 | 实现位置 | 关键点 |
|------|---------|-------|
| 三级复杂度 | `spec/complexity.py: Complexity` | SIMPLE / STANDARD / COMPLEX |
| 动态阶段选择 | `ComplexityAssessment.phases_to_run()` | 复杂度决定执行哪些阶段 |
| 双轨评估 | `orchestrator.py: _phase_complexity_assessment` | AI 首选，启发式兜底 |
| 阶段摘要压缩 | `orchestrator.py: _store_phase_summary()` | 防止上下文窗口溢出 |
| 自我批判循环 | `spec/critique.py` | 结构化 5 步检查 + 正则解析 |
| Greenfield 适配 | `spec_phases.py: _greenfield_context()` | 运行时动态注入额外上下文 |
| 人工审查检查点 | `orchestrator.py: _run_review_checkpoint()` | Human-in-the-Loop 设计 |
| 原子序号分配 | `SpecNumberLock` | 防止并行流水线号码冲突 |

---

## Lab 10.1：迷你规格创建流水线

本 Lab 实现一个简化版的规格创建流水线，包含复杂度评估、动态阶段选择和结构验证。

### Part A：复杂度评估器（必做）

```python
# lab_10_spec_pipeline.py
from __future__ import annotations

import json
import re
from dataclasses import dataclass, field
from enum import Enum
from pathlib import Path


class Complexity(Enum):
    SIMPLE   = "simple"
    STANDARD = "standard"
    COMPLEX  = "complex"


@dataclass
class ComplexityAssessment:
    complexity: Complexity
    confidence: float
    reasoning: str
    estimated_files: int = 3
    needs_research: bool = False
    needs_self_critique: bool = False

    def phases_to_run(self) -> list[str]:
        """
        根据复杂度返回要执行的阶段列表。

        TODO: 实现阶段选择逻辑（10-20 行）：

        SIMPLE → ["discovery", "quick_spec", "validation"]

        STANDARD → ["discovery", "requirements", "context", "spec_writing", "planning", "validation"]
          如果 needs_research=True，在 context 之前插入 "research"

        COMPLEX → ["discovery", "requirements", "research", "context",
                   "spec_writing", "self_critique", "planning", "validation"]
          self_critique 必须在 spec_writing 之后、planning 之前

        提示：用列表动态构建，需要某些阶段时使用 list.append() 或 list.insert()
        """
        # YOUR CODE HERE
        pass


SIMPLE_KEYWORDS = [
    "fix", "typo", "update", "change", "rename", "delete",
    "style", "color", "text", "button", "margin"
]

COMPLEX_KEYWORDS = [
    "integrate", "integration", "api", "sdk", "database", "migrate",
    "docker", "authentication", "oauth", "microservice",
    "refactor", "architecture", "websocket", "kafka"
]


def assess_complexity(task_description: str) -> ComplexityAssessment:
    """
    基于关键词启发式评估任务复杂度。

    TODO: 实现复杂度评估逻辑（20-30 行）：

    算法：
    1. 将 task_description 转为小写
    2. 统计 SIMPLE_KEYWORDS 和 COMPLEX_KEYWORDS 的命中数
    3. 检测外部集成（oauth, database, redis, docker, payment 等）
    4. 估算文件数（简单任务 1-2 个，标准 5 个，复杂 15 个）
    5. 根据信号决定复杂度：
       - 有 >= 2 个复杂关键词，或有外部集成 → COMPLEX（置信度 0.85）
       - 只有简单关键词，无复杂词 → SIMPLE（置信度 0.90）
       - 否则 → STANDARD（置信度 0.75）
    6. 如果检测到 database/api/oauth 等集成词，设置 needs_research=True

    返回 ComplexityAssessment 对象
    """
    # YOUR CODE HERE
    pass


# === 验证 Part A ===
def test_complexity():
    cases = [
        ("Fix the typo in the login button text", Complexity.SIMPLE),
        ("Add user authentication with OAuth and database integration", Complexity.COMPLEX),
        ("Create a new settings page with form validation", Complexity.STANDARD),
        ("Integrate Stripe payment gateway with webhook handling and database recording",
         Complexity.COMPLEX),
    ]

    for desc, expected in cases:
        result = assess_complexity(desc)
        status = "[PASS]" if result.complexity == expected else "[FAIL]"
        print(f"{status} '{desc[:50]}...' → {result.complexity.value} "
              f"(expected: {expected.value})")
```

### Part B：流水线编排器（必做）

```python
# 接续 lab_10_spec_pipeline.py
import asyncio
from typing import Callable, Any


@dataclass
class PhaseResult:
    phase_name: str
    success: bool
    output: dict = field(default_factory=dict)
    error: str | None = None


@dataclass
class SpecPipelineResult:
    success: bool
    complexity: Complexity
    phases_executed: list[str]
    outputs: dict[str, dict]  # phase_name → output data
    error: str | None = None

    def get_spec(self) -> str | None:
        """从输出中提取规格内容。"""
        for phase in ["spec_writing", "quick_spec"]:
            if phase in self.outputs:
                return self.outputs[phase].get("spec_content")
        return None


class MiniSpecPipeline:
    """
    迷你规格创建流水线。

    将模拟的阶段函数按照复杂度动态组合执行。
    """

    def __init__(
        self,
        task_description: str,
        phase_handlers: dict[str, Callable],
        verbose: bool = True,
    ):
        self.task_description = task_description
        self.phase_handlers = phase_handlers
        self.verbose = verbose
        self._outputs: dict[str, dict] = {}

    async def _run_phase(self, phase_name: str) -> PhaseResult:
        """运行单个阶段，带错误处理。"""
        if phase_name not in self.phase_handlers:
            return PhaseResult(phase_name, False, error=f"No handler for phase: {phase_name}")

        try:
            handler = self.phase_handlers[phase_name]
            output = await handler(self.task_description, self._outputs)
            self._outputs[phase_name] = output
            return PhaseResult(phase_name, True, output=output)
        except Exception as e:
            return PhaseResult(phase_name, False, error=str(e))

    async def run(self) -> SpecPipelineResult:
        """
        运行完整的规格创建流水线。

        TODO: 实现流水线执行逻辑（15-25 行）：

        步骤：
        1. 调用 assess_complexity() 评估任务复杂度
        2. 获取要执行的阶段列表（assessment.phases_to_run()）
        3. 按顺序执行每个阶段（调用 self._run_phase()）
        4. 如果任意阶段失败，停止并返回失败的 SpecPipelineResult
        5. 如果 verbose=True，打印每个阶段的执行状态
        6. 所有阶段成功后，返回成功的 SpecPipelineResult

        提示：
        - 用 phases_executed 列表追踪已执行的阶段
        - SpecPipelineResult 的 outputs 字段对应 self._outputs
        """
        # YOUR CODE HERE
        pass


# === 模拟阶段处理器（用于测试）===

async def mock_discovery(task: str, prev: dict) -> dict:
    """模拟发现阶段：扫描项目文件。"""
    await asyncio.sleep(0.01)  # 模拟耗时
    return {
        "file_count": 47,
        "languages": ["python", "typescript"],
        "has_tests": True,
    }

async def mock_requirements(task: str, prev: dict) -> dict:
    """模拟需求收集：结构化任务描述。"""
    await asyncio.sleep(0.01)
    return {
        "task_description": task,
        "acceptance_criteria": [
            "功能按预期工作",
            "已有测试通过",
            "代码符合风格规范",
        ],
        "constraints": ["不破坏现有 API"],
    }

async def mock_spec_writing(task: str, prev: dict) -> dict:
    """模拟规格写作：生成 spec.md 内容。"""
    await asyncio.sleep(0.01)
    requirements = prev.get("requirements", {})
    criteria = requirements.get("acceptance_criteria", [])
    spec_content = f"""# 规格：{task}

## 目标
{task}

## 验收标准
{chr(10).join(f"- {c}" for c in criteria)}

## 实现方案
[详细实现描述]
"""
    return {"spec_content": spec_content}

async def mock_quick_spec(task: str, prev: dict) -> dict:
    """简单任务的快速规格（跳过多步流程）。"""
    await asyncio.sleep(0.01)
    return {"spec_content": f"# Quick Spec: {task}\n\n简单修改，预计 1-2 个文件。"}

async def mock_planning(task: str, prev: dict) -> dict:
    """模拟规划：生成实现计划。"""
    await asyncio.sleep(0.01)
    return {
        "subtasks": [
            {"id": "1", "description": "实现核心逻辑", "status": "pending"},
            {"id": "2", "description": "添加测试", "status": "pending"},
        ]
    }

async def mock_validation(task: str, prev: dict) -> dict:
    """模拟验证：检查规格完整性。"""
    has_spec = "spec_writing" in prev or "quick_spec" in prev
    has_plan = "planning" in prev
    return {
        "valid": has_spec and has_plan,
        "checks": {
            "has_spec": has_spec,
            "has_plan": has_plan,
        }
    }

# 其他阶段（仅返回空结果，不影响核心流程）
async def mock_research(task: str, prev: dict) -> dict:
    return {"research_notes": "相关代码已分析"}

async def mock_context(task: str, prev: dict) -> dict:
    return {"relevant_files": ["src/auth.py", "src/models.py"]}

async def mock_self_critique(task: str, prev: dict) -> dict:
    spec = prev.get("spec_writing", {}).get("spec_content", "")
    passes = len(spec) > 50  # 简单验证
    return {"passes": passes, "issues": []}


MOCK_HANDLERS = {
    "discovery": mock_discovery,
    "requirements": mock_requirements,
    "research": mock_research,
    "context": mock_context,
    "spec_writing": mock_spec_writing,
    "quick_spec": mock_quick_spec,
    "self_critique": mock_self_critique,
    "planning": mock_planning,
    "validation": mock_validation,
}


async def test_pipeline():
    """测试流水线端到端。"""
    test_cases = [
        ("Fix the typo in button label", Complexity.SIMPLE),
        ("Add user profile settings page with form validation", Complexity.STANDARD),
        ("Integrate OAuth2 authentication with database user management", Complexity.COMPLEX),
    ]

    for task, expected_complexity in test_cases:
        print(f"\n--- Task: {task[:50]} ---")
        pipeline = MiniSpecPipeline(task, MOCK_HANDLERS, verbose=True)
        result = await pipeline.run()

        assert result.complexity == expected_complexity, \
            f"Expected {expected_complexity.value}, got {result.complexity.value}"
        assert result.success, f"Pipeline should succeed for: {task}"
        assert result.get_spec() is not None, "Should have spec content"
        assert "validation" in result.phases_executed, "Validation should run"
        print(f"  Result: {result.complexity.value}, {len(result.phases_executed)} phases")

    print("\n所有流水线测试通过！")
```

### Part C：规格验证器（进阶）

```python
# 接续 lab_10_spec_pipeline.py

@dataclass
class SpecValidationResult:
    valid: bool
    missing_sections: list[str] = field(default_factory=list)
    warnings: list[str] = field(default_factory=list)
    score: float = 0.0   # 0.0 到 1.0


def validate_spec(spec_content: str) -> SpecValidationResult:
    """
    验证规格文档的完整性。

    TODO: 实现规格验证逻辑（15-20 行）：

    必要章节（缺失则 valid=False）：
    - "# " 开头的标题（文档存在）
    - "目标" 或 "Goal" 或 "Overview"
    - "验收标准" 或 "Acceptance" 或 "Requirements"

    可选章节（缺失则加入 warnings）：
    - "实现" 或 "Implementation"
    - "测试" 或 "Test"
    - "约束" 或 "Constraint"

    评分规则：
    - 必要章节每个占 30%（共 3 个必要章节，满分 90%）
    - 可选章节每个占 5%（共 3 个）
    - score 限制在 0.0 到 1.0 之间

    提示：使用 re.search() 进行大小写不敏感的匹配
    """
    # YOUR CODE HERE
    pass


async def main():
    print("=== Part A: 复杂度评估测试 ===")
    test_complexity()

    print("\n=== Part B: 流水线测试 ===")
    await test_pipeline()

    print("\n=== Part C: 规格验证测试 ===")
    good_spec = """# 用户登录功能规格

## 目标
实现安全的用户登录功能。

## 验收标准
- 用户可以用邮件和密码登录
- 失败 5 次后锁定账户

## 实现方案
使用 JWT token，15 分钟过期。

## 测试要求
单元测试覆盖率 > 80%
"""
    result = validate_spec(good_spec)
    assert result.valid, f"Good spec should be valid: {result.missing_sections}"
    assert result.score >= 0.9, f"Score should be high: {result.score}"
    print(f"[PASS] Good spec: valid={result.valid}, score={result.score:.1%}")

    bad_spec = "# Something\n\nJust some text without structure."
    result = validate_spec(bad_spec)
    assert not result.valid, "Bad spec should fail"
    assert len(result.missing_sections) > 0
    print(f"[PASS] Bad spec: valid={result.valid}, missing={result.missing_sections}")

    print("\n全部测试通过！")


if __name__ == "__main__":
    asyncio.run(main())
```

### 验证标准

- `assess_complexity()` 正确分类至少 3/4 的测试用例
- `phases_to_run()` 为 COMPLEX 任务返回包含 `self_critique` 的阶段列表
- 流水线在任意阶段失败时立即停止并返回错误
- `validate_spec()` 能识别缺少必要章节的不完整规格

### 进阶挑战

1. **阶段并行化**：在流水线中识别哪些阶段可以并行执行（如 `research` 和 `historical_context`），用 `asyncio.gather()` 实现并行化，并测量加速效果
2. **渐进增强**：实现一个 `upgrade_spec()` 函数：接受现有的 SIMPLE 规格，将其升级为 STANDARD 或 COMPLEX 规格（补充缺失的章节），模拟用户在审查后请求"把这个规格写得更详细一些"的场景

---

*下一章，我们将探索 Auto-Claude 的项目分析系统——它如何在第一次运行时"认识"一个陌生的代码库，并生成专属于这个项目的安全配置文件。*
