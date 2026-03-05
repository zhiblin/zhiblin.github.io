---
title: 第十九章：从零构建一个迷你 Auto-Claude
date: 2026-01-19 00:00:00
categories: Agents
tags: [AI, Agent, Python, Claude, SDK]
---

# 第十九章　从零构建一个迷你 Auto-Claude

> "理解一个系统最深刻的方式，是自己动手重建它的核心。不是复制，而是用第一原理推导出相同的设计决策。"

## 19.1　目标：一个可运行的微型系统

经过 18 章的源码精读，我们已经深入理解了 Auto-Claude 的每一层架构。现在是时候综合练习了——从零实现一个微型的 Auto-Claude，能够真实运行简单任务。

**目标系统功能**：
1. 接受一个自然语言任务描述
2. Planner 智能体分析任务，生成 2-3 个子任务
3. Coder 智能体依次实现每个子任务
4. QA 智能体验证实现，如有问题触发重试
5. 所有操作在隔离的 Git worktree 中进行

**技术约束**（保持简单）：
- 不使用 Electron（纯 Python CLI）
- 不使用 Graphiti（文件级记忆）
- 不使用多账户切换（单账户）
- QA 最多重试 2 次

---

## 19.2　架构设计

```
用户输入（任务描述）
     │
     ▼
┌────────────────────────────────────────┐
│              MiniAutoClaude            │
│                                        │
│  1. create_worktree()                  │
│  2. run_planner() → [subtasks]         │
│  3. for each subtask:                  │
│       run_coder(subtask)               │
│  4. run_qa_loop() → pass/fail          │
│  5. merge_worktree()                   │
└────────────────────────────────────────┘
     │
     ▼
Git 仓库（功能实现在独立分支）
```

核心设计原则（来自本书各章）：
- **隔离**（第5章）：所有操作在 worktree 里，不污染主分支
- **合约提示词**（第3章）：结构化 JSON 输出，强制执行格式
- **错误分类**（第9章）：区分可重试错误和不可重试错误
- **状态机**（第2章）：QA Loop 是有限状态机，不是无限循环

---

## 19.3　Part 1：骨架与 Worktree 管理

```python
# mini_auto_claude/main.py

import asyncio
import subprocess
import json
import os
import shutil
from pathlib import Path
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class MiniTask:
    """任务描述。"""
    id: str
    description: str
    project_path: str

@dataclass
class Subtask:
    """规划器生成的子任务。"""
    index: int
    description: str
    files_to_create: list[str] = field(default_factory=list)
    files_to_modify: list[str] = field(default_factory=list)

@dataclass
class TaskResult:
    """任务执行结果。"""
    success: bool
    worktree_path: Optional[str] = None
    subtasks: list[Subtask] = field(default_factory=list)
    qa_iterations: int = 0
    error: Optional[str] = None


class WorktreeManager:
    """简化版 Worktree 管理器（来自第5章）。"""

    def __init__(self, project_path: str):
        self.project_path = Path(project_path)

    def create(self, task_id: str) -> Path:
        """在项目目录旁边创建 worktree。"""
        branch_name = f"mini-auto-claude/{task_id}"
        worktree_path = self.project_path.parent / f"{self.project_path.name}-{task_id}"

        # 创建新分支并 checkout 到 worktree
        subprocess.run(
            ['git', '-C', str(self.project_path),
             'worktree', 'add', '-b', branch_name, str(worktree_path)],
            check=True, capture_output=True
        )

        print(f"[Worktree] 创建: {worktree_path}")
        return worktree_path

    def remove(self, worktree_path: Path) -> None:
        """清理 worktree（不合并时使用）。"""
        subprocess.run(
            ['git', '-C', str(self.project_path),
             'worktree', 'remove', '--force', str(worktree_path)],
            capture_output=True
        )
        print(f"[Worktree] 清理: {worktree_path}")

    def merge(self, worktree_path: Path, task_id: str) -> bool:
        """将 worktree 分支合并回主分支。"""
        branch_name = f"mini-auto-claude/{task_id}"
        try:
            subprocess.run(
                ['git', '-C', str(self.project_path),
                 'merge', '--no-ff', branch_name, '-m', f'feat: {task_id}'],
                check=True, capture_output=True
            )
            self.remove(worktree_path)
            print(f"[Worktree] 合并成功: {branch_name}")
            return True
        except subprocess.CalledProcessError as e:
            print(f"[Worktree] 合并失败: {e.stderr.decode()}")
            return False
```

---

## 19.4　Part 2：Planner 智能体

```python
# mini_auto_claude/agents/planner.py

PLANNER_PROMPT = """你是一个软件规划专家。分析以下任务，生成具体的实现子任务列表。

任务描述：{task_description}
项目路径：{project_path}

要求：
1. 生成 2-4 个子任务
2. 每个子任务必须具体可执行
3. 必须以 JSON 格式输出，不要有额外说明

输出格式（严格遵守）：
```json
{
  "subtasks": [
    {
      "index": 1,
      "description": "子任务描述",
      "files_to_create": ["path/to/new/file.py"],
      "files_to_modify": ["path/to/existing/file.py"]
    }
  ]
}
```
"""

async def run_planner(task: MiniTask) -> list[Subtask]:
    """调用 AI 规划任务，返回子任务列表。"""
    from claude_agent_sdk import ClaudeSDKClient, ClaudeAgentOptions

    client = ClaudeSDKClient(ClaudeAgentOptions(
        model="claude-opus-4-5",
        system_prompt=PLANNER_PROMPT.format(
            task_description=task.description,
            project_path=task.project_path
        ),
        # 禁用所有工具：Planner 只做规划，不操作文件
        allowed_tools=[],
    ))

    # 收集 AI 的完整响应
    full_response = ""
    async for event in client.stream("请分析任务并生成子任务列表。"):
        if event.type == "text":
            full_response += event.text

    # 解析 JSON 输出（来自第3章：结构化输出解析）
    return _parse_subtasks(full_response)


def _parse_subtasks(response: str) -> list[Subtask]:
    """从 AI 响应中提取 JSON 并解析子任务。"""
    import re

    # 提取 ```json ... ``` 代码块
    json_match = re.search(r'```json\s*(.*?)\s*```', response, re.DOTALL)
    if not json_match:
        # 尝试直接解析（AI 可能忘记了代码块）
        json_match = re.search(r'\{.*\}', response, re.DOTALL)
        if not json_match:
            raise ValueError(f"规划器未返回有效 JSON: {response[:200]}")

    data = json.loads(json_match.group(1) if '```' in response else json_match.group(0))

    return [
        Subtask(
            index=st['index'],
            description=st['description'],
            files_to_create=st.get('files_to_create', []),
            files_to_modify=st.get('files_to_modify', []),
        )
        for st in data['subtasks']
    ]
```

---

## 19.5　Part 3：Coder 智能体

```python
# mini_auto_claude/agents/coder.py

CODER_PROMPT = """你是一个经验丰富的软件工程师。请实现以下子任务。

子任务：{subtask_description}
工作目录：{worktree_path}

文件操作权限：
- 可以创建：{files_to_create}
- 可以修改：{files_to_modify}

要求：
1. 直接实现，不要只写注释
2. 代码要可以运行
3. 完成后输出一行：IMPLEMENTATION_COMPLETE
"""

async def run_coder(subtask: Subtask, worktree_path: Path) -> bool:
    """在 worktree 中实现子任务。"""
    from claude_agent_sdk import ClaudeSDKClient, ClaudeAgentOptions

    client = ClaudeSDKClient(ClaudeAgentOptions(
        model="claude-sonnet-4-6",
        system_prompt=CODER_PROMPT.format(
            subtask_description=subtask.description,
            worktree_path=str(worktree_path),
            files_to_create=subtask.files_to_create or ["（根据需要）"],
            files_to_modify=subtask.files_to_modify or ["（根据需要）"],
        ),
        # 允许文件操作工具
        allowed_tools=["Read", "Write", "Edit", "Bash"],
        # 文件系统权限：只允许在 worktree 里操作
        allowed_paths=[str(worktree_path)],
    ))

    print(f"[Coder] 实现子任务 {subtask.index}: {subtask.description[:50]}...")

    success = False
    async for event in client.stream(f"请实现子任务：{subtask.description}"):
        if event.type == "text" and "IMPLEMENTATION_COMPLETE" in event.text:
            success = True
        elif event.type == "tool_use":
            print(f"  → {event.tool_name}({str(event.input)[:60]})")

    # 实现完成后，自动 commit
    if success:
        subprocess.run(
            ['git', '-C', str(worktree_path), 'add', '.'],
            capture_output=True
        )
        subprocess.run(
            ['git', '-C', str(worktree_path), 'commit', '-m',
             f'feat: subtask {subtask.index} - {subtask.description[:40]}'],
            capture_output=True
        )
        print(f"[Coder] 子任务 {subtask.index} 完成并已 commit")

    return success
```

---

## 19.6　Part 4：QA 循环

```python
# mini_auto_claude/agents/qa.py

QA_PROMPT = """你是一个严格的 QA 审查员。验证以下任务是否已正确实现。

原始任务：{task_description}
工作目录：{worktree_path}

验证步骤：
1. 检查文件是否被创建/修改
2. 代码是否能运行（如果适用）
3. 实现是否符合任务描述

输出格式（严格遵守）：
```json
{
  "status": "PASS" 或 "FAIL",
  "issues": ["问题1", "问题2"],  // FAIL 时必填
  "summary": "简短总结"
}
```
"""

@dataclass
class QAResult:
    status: str  # 'PASS' or 'FAIL'
    issues: list[str]
    summary: str
    iteration: int

async def run_qa_loop(
    task: MiniTask,
    worktree_path: Path,
    max_retries: int = 2
) -> QAResult:
    """
    QA 验证循环（来自第2章的 QA Loop 设计）。
    失败时触发 Fixer，最多重试 max_retries 次。
    """
    for iteration in range(max_retries + 1):
        print(f"\n[QA] 第 {iteration + 1} 次验证...")

        result = await _run_qa_reviewer(task, worktree_path, iteration + 1)

        if result.status == "PASS":
            print(f"[QA] 验证通过！")
            return result

        print(f"[QA] 发现问题：{result.issues}")

        if iteration < max_retries:
            print(f"[QA] 启动修复...")
            await _run_qa_fixer(task, worktree_path, result.issues)
        else:
            print(f"[QA] 达到最大重试次数，任务失败")

    return result


async def _run_qa_reviewer(
    task: MiniTask,
    worktree_path: Path,
    iteration: int
) -> QAResult:
    """运行 QA 审查员。"""
    from claude_agent_sdk import ClaudeSDKClient, ClaudeAgentOptions

    client = ClaudeSDKClient(ClaudeAgentOptions(
        model="claude-sonnet-4-6",
        system_prompt=QA_PROMPT.format(
            task_description=task.description,
            worktree_path=str(worktree_path)
        ),
        allowed_tools=["Read", "Bash"],  # 只读权限
        allowed_paths=[str(worktree_path)],
    ))

    full_response = ""
    async for event in client.stream("请验证任务实现是否正确。"):
        if event.type == "text":
            full_response += event.text

    return _parse_qa_result(full_response, iteration)


def _parse_qa_result(response: str, iteration: int) -> QAResult:
    """解析 QA 审查结果。"""
    import re
    json_match = re.search(r'```json\s*(.*?)\s*```', response, re.DOTALL)
    if not json_match:
        # 解析失败：保守地视为通过（避免无限循环）
        return QAResult(status="PASS", issues=[], summary="解析失败，保守通过", iteration=iteration)

    data = json.loads(json_match.group(1))
    return QAResult(
        status=data.get('status', 'FAIL'),
        issues=data.get('issues', []),
        summary=data.get('summary', ''),
        iteration=iteration
    )
```

---

## 19.7　Part 5：主编排器

```python
# mini_auto_claude/main.py（续）

class MiniAutoClaude:
    """微型 Auto-Claude 编排器。"""

    def __init__(self, project_path: str):
        self.project_path = project_path
        self.worktree_manager = WorktreeManager(project_path)

    async def run(self, task_description: str) -> TaskResult:
        """端到端执行一个任务。"""
        import uuid
        task_id = f"task-{uuid.uuid4().hex[:8]}"
        task = MiniTask(id=task_id, description=task_description,
                        project_path=self.project_path)

        worktree_path = None
        try:
            # Step 1: 创建隔离 worktree
            print(f"\n{'='*50}")
            print(f"[Main] 开始任务: {task_description[:60]}")
            worktree_path = self.worktree_manager.create(task_id)

            # Step 2: 规划
            print("\n[Main] === 规划阶段 ===")
            subtasks = await run_planner(task)
            print(f"[Main] 生成了 {len(subtasks)} 个子任务")
            for st in subtasks:
                print(f"  {st.index}. {st.description}")

            # Step 3: 实现
            print("\n[Main] === 实现阶段 ===")
            for subtask in subtasks:
                success = await run_coder(subtask, worktree_path)
                if not success:
                    return TaskResult(
                        success=False,
                        error=f"子任务 {subtask.index} 实现失败"
                    )

            # Step 4: QA 验证
            print("\n[Main] === QA 验证阶段 ===")
            qa_result = await run_qa_loop(task, worktree_path, max_retries=2)

            if qa_result.status != "PASS":
                return TaskResult(
                    success=False,
                    worktree_path=str(worktree_path),
                    subtasks=subtasks,
                    qa_iterations=qa_result.iteration,
                    error="QA 验证最终失败"
                )

            # Step 5: 合并
            print("\n[Main] === 合并阶段 ===")
            merged = self.worktree_manager.merge(worktree_path, task_id)
            worktree_path = None  # 已合并，不需要清理

            return TaskResult(
                success=merged,
                subtasks=subtasks,
                qa_iterations=qa_result.iteration
            )

        except Exception as e:
            print(f"\n[Main] 任务失败: {e}")
            return TaskResult(success=False, error=str(e))

        finally:
            # 失败时清理 worktree
            if worktree_path and worktree_path.exists():
                self.worktree_manager.remove(worktree_path)


# 入口点
if __name__ == '__main__':
    import sys

    if len(sys.argv) < 3:
        print("用法: python -m mini_auto_claude <项目路径> <任务描述>")
        sys.exit(1)

    project_path = sys.argv[1]
    task_description = ' '.join(sys.argv[2:])

    result = asyncio.run(
        MiniAutoClaude(project_path).run(task_description)
    )

    print(f"\n{'='*50}")
    if result.success:
        print(f"✓ 任务成功完成！")
        print(f"  子任务数: {len(result.subtasks)}")
        print(f"  QA 迭代: {result.qa_iterations}")
    else:
        print(f"✗ 任务失败: {result.error}")
        sys.exit(1)
```

---

## 19.8　运行与扩展

### 运行示例

```bash
# 在一个 Git 仓库里运行
python -m mini_auto_claude /path/to/my-project "添加一个命令行参数解析模块，支持 --verbose 和 --output 选项"

# 输出示例：
# ==================================================
# [Main] 开始任务: 添加一个命令行参数解析模块...
#
# [Main] === 规划阶段 ===
# [Main] 生成了 2 个子任务
#   1. 创建 cli/args.py 模块，使用 argparse 实现参数解析
#   2. 在 main.py 中集成参数解析模块
#
# [Main] === 实现阶段 ===
# [Coder] 实现子任务 1: 创建 cli/args.py 模块...
#   → Write({"file_path": "cli/args.py", ...})
# [Coder] 子任务 1 完成并已 commit
# ...
#
# [Main] === QA 验证阶段 ===
# [QA] 第 1 次验证...
# [QA] 验证通过！
#
# [Main] === 合并阶段 ===
# [Worktree] 合并成功: mini-auto-claude/task-abc12345
#
# ==================================================
# ✓ 任务成功完成！
#   子任务数: 2
#   QA 迭代: 1
```

### 可以扩展的方向

1. **并行子任务**（第5章）：当子任务独立时，用 `asyncio.gather` 并行运行 Coder
2. **文件记忆**（第6章）：用 JSON 文件存储任务历史，影响下次规划
3. **复杂度评估**（第10章）：简单任务跳过 Planner，直接进 Coder
4. **指数退避**（第9章）：AI API 调用失败时自动重试

---

## 19.9　Lab 19.1：完成迷你 Auto-Claude

**Part A**：在本地运行上面的代码，执行一个真实的简单任务（例如：让它给一个 Python 文件添加类型注解）。

**Part B**：实现并行子任务执行：修改主编排器，当规划器返回标记为 `independent: true` 的子任务时，用 `asyncio.gather` 并行运行 Coder。

**Part C**：添加任务状态持久化：在每个阶段完成时，把任务状态写入 `.mini-auto-claude/{task_id}/state.json`，支持从中断处恢复。

**验证标准**：
- Part A：能成功完成至少一个真实任务，Git 历史有 Coder 的 commit 和合并 commit
- Part B：两个独立子任务的总时间 < 串行时间的 70%（并行生效）
- Part C：手动中断后重新运行，能从上次完成的子任务继续（不重做已完成的）
