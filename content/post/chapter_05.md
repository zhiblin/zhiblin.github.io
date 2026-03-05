---
title: 第五章：Git Worktree 隔离与并行智能体协作
date: 2026-01-05 00:00:00
categories: Agents
tags: [AI, Agent, Python, Claude, SDK]
---

# 第五章：Git Worktree 隔离与并行智能体协作

> "并发的核心问题不是速度，而是隔离。"

## 5.1 并行的代价

假设你有三个 AI 智能体同时为同一个项目实现三个功能。如果它们都直接操作同一份代码：

```
智能体 A：修改 utils.py，添加 parse_date() 函数
智能体 B：修改 utils.py，重构 format_string() 函数
智能体 C：修改 utils.py，添加 validate_input() 函数
```

结果会是灾难性的。每个智能体读取文件时看到的是"当前状态"，写入时会覆盖其他智能体的改动。三个智能体的工作可能互相冲突、循环覆盖，最终产出一个混乱的文件。

解决方案不是序列化（让智能体排队）——那会放弃并行的优势。真正的解决方案是**隔离**：让每个智能体在自己的独立副本上工作，最后再智能地合并。

Git 的 worktree 功能天然适合这个场景。

## 5.2 Git Worktree：一个 Repo，多个工作目录

通常，一个 Git 仓库对应一个工作目录。但 `git worktree` 命令允许你**从同一个 Git 仓库检出多个工作目录**，每个工作目录对应不同的分支：

```bash
# 主仓库，在 main 分支
/project/
  src/
  tests/

# 添加 worktree，检出到独立目录
git worktree add .auto-claude/worktrees/tasks/feature-auth auto-claude/feature-auth

# 现在你有两个工作目录：
/project/                              # main 分支
.auto-claude/worktrees/tasks/feature-auth/   # auto-claude/feature-auth 分支
```

两个工作目录共享同一个 `.git` 目录（或通过 `.git` 文件链接到主仓库），但它们有完全独立的文件系统视图。对一个工作目录的修改不会影响另一个，直到你显式合并。

Auto-Claude 将此抽象为清晰的三元一一映射：

```
spec（规格文档）→ worktree（工作目录）→ branch（Git 分支）

spec-001-auth-system  →  .auto-claude/worktrees/tasks/001-auth-system/  →  auto-claude/001-auth-system
spec-002-payment      →  .auto-claude/worktrees/tasks/002-payment/       →  auto-claude/002-payment
spec-003-notifications→  .auto-claude/worktrees/tasks/003-notifications/ →  auto-claude/003-notifications
```

三个智能体可以同时工作，互不干扰。

`★ Insight ─────────────────────────────────────`

**Worktree vs 克隆**：你可能想到"为什么不直接克隆多份仓库？" Worktree 的优势：1）共享 `.git` 对象数据库，节省磁盘空间（尤其对大仓库）；2）一个分支不能在两个 worktree 中同时签出，Git 会强制防止冲突；3）主仓库的 `git log` 和 `git branch` 能看到所有 worktree 的状态，便于管理。

`─────────────────────────────────────────────────`

## 5.3 WorktreeManager：生命周期管理

`apps/backend/core/worktree.py` 中的 `WorktreeManager` 封装了所有 worktree 操作：

```python
class WorktreeManager:
    """
    管理每个 spec 的 Git worktree。

    约定：
    - 路径：.auto-claude/worktrees/tasks/{spec-name}/
    - 分支：auto-claude/{spec-name}
    """

    GIT_PUSH_TIMEOUT = 120   # 2 分钟（网络操作）
    CLI_TIMEOUT = 60          # 1 分钟（gh/glab 命令）
    CLI_QUERY_TIMEOUT = 30    # 30 秒（查询操作）

    def __init__(
        self,
        project_dir: Path,
        base_branch: str | None = None,
        use_local_branch: bool = False,
    ):
        self.project_dir = project_dir
        self.base_branch = base_branch or self._detect_base_branch()
        self.worktrees_dir = project_dir / ".auto-claude" / "worktrees" / "tasks"
        self._merge_lock = asyncio.Lock()  # 合并操作需要互斥
```

注意 `_merge_lock`：虽然多个智能体可以并行**开发**，但最终合并到主分支的操作必须序列化，防止合并竞争。

### 基础分支检测

```python
def _detect_base_branch(self) -> str:
    """
    按优先级检测基础分支。

    Priority:
    1. DEFAULT_BRANCH 环境变量（明确配置优先）
    2. 自动检测 main/master（常规约定）
    3. 回退到当前分支（带警告）
    """
    # 1. 环境变量
    env_branch = os.getenv("DEFAULT_BRANCH")
    if env_branch and self._branch_exists(env_branch):
        return env_branch

    # 2. 约定检测
    for candidate in ["main", "master"]:
        if self._branch_exists(candidate):
            return candidate

    # 3. 回退（可能不是预期行为，记录警告）
    current = self._get_current_branch()
    debug_warning(f"Could not detect main branch, using current: {current}")
    return current
```

这段代码展示了良好的"配置优先，约定次之，回退兜底"策略。环境变量允许 CI 环境明确指定分支；约定检测处理 99% 的情况；回退确保系统不会因为无法检测分支而崩溃。

## 5.4 Worktree 创建：从规格到独立工作空间

```python
async def create_worktree(self, spec_name: str) -> WorktreeInfo:
    """
    为 spec 创建独立的 git worktree。

    步骤：
    1. 确定路径和分支名
    2. 从基础分支创建新分支
    3. 在指定路径添加 worktree
    4. 返回 WorktreeInfo
    """
    # 规范化名称（去除特殊字符）
    safe_name = re.sub(r"[^a-zA-Z0-9_-]", "-", spec_name)
    worktree_path = self.worktrees_dir / safe_name
    branch_name = f"auto-claude/{safe_name}"

    # 如果 worktree 已存在，直接返回
    if worktree_path.exists():
        return self._get_worktree_info(worktree_path, branch_name, spec_name)

    # 确保目录结构存在
    self.worktrees_dir.mkdir(parents=True, exist_ok=True)

    # 创建分支并添加 worktree
    success, error = run_git(
        ["worktree", "add", "-b", branch_name, str(worktree_path), self.base_branch],
        cwd=self.project_dir,
        timeout=30,
    )

    if not success:
        raise WorktreeError(f"Failed to create worktree: {error}")

    logger.info(f"Created worktree: {worktree_path} (branch: {branch_name})")
    return self._get_worktree_info(worktree_path, branch_name, spec_name)
```

`git worktree add -b <branch> <path> <start-point>` 是 Git 的原子操作：同时创建分支和工作目录。如果任何一步失败，整个操作回滚，不会留下半成品状态。

## 5.5 网络重试：指数退避的工程实践

网络操作（git push、PR 创建）在现实环境中经常失败。`worktree.py` 实现了一个通用的重试框架：

```python
def _with_retry(
    operation: Callable[[], tuple[bool, T | None, str]],
    max_retries: int = 3,
    is_retryable: Callable[[str], bool] | None = None,
    on_retry: Callable[[int, str], None] | None = None,
) -> tuple[T | None, str]:
    """
    带重试的操作执行框架。

    operation 返回 (success, result, error) 三元组：
    - success=True：返回 result
    - success=False：检查是否可重试
    """
    last_error = ""

    for attempt in range(1, max_retries + 1):
        try:
            success, result, error = operation()
            if success:
                return result, ""

            last_error = error

            # 只重试可重试的错误
            if is_retryable and attempt < max_retries and is_retryable(error):
                if on_retry:
                    on_retry(attempt, error)
                backoff = 2 ** (attempt - 1)  # 1s, 2s, 4s...
                time.sleep(backoff)
                continue

            break  # 不可重试，立即放弃

        except subprocess.TimeoutExpired:
            last_error = "Operation timed out"
            if attempt < max_retries:
                backoff = 2 ** (attempt - 1)
                time.sleep(backoff)
                continue
            break

    return None, last_error
```

这个函数体现了几个重要设计：

**可重试性判断**：不是所有错误都应该重试。`_is_retryable_network_error()` 识别网络超时、连接拒绝等瞬时错误；`_is_retryable_http_error()` 识别 5xx 服务器错误。但 401 认证失败、404 资源不存在就不应该重试——那是逻辑错误，不是瞬时故障。

**指数退避**：第一次重试等 1 秒，第二次等 2 秒，第三次等 4 秒。这让服务器有时间恢复，也避免在服务过载时雪上加霜。

**关注点分离**：`on_retry` 回调允许调用者记录重试信息，而不污染重试逻辑本身。

```python
# 使用示例
result, error = _with_retry(
    operation=lambda: push_branch(branch_name),
    max_retries=3,
    is_retryable=_is_retryable_network_error,
    on_retry=lambda attempt, err: logger.warning(
        f"Push attempt {attempt} failed: {err}, retrying..."
    ),
)
```

`★ Insight ─────────────────────────────────────`

**将可重试性设计为参数**：`is_retryable` 作为回调传入，而不是硬编码在 `_with_retry` 内部。这使得同一个重试框架可以用于 git 网络错误、HTTP API 错误、数据库连接错误等不同场景，每种场景有自己的可重试性判断逻辑。高内聚，低耦合。

`─────────────────────────────────────────────────`

## 5.6 智能合并：从代码变更到语义理解

当并行智能体完成工作后，最复杂的挑战开始了：**如何将多个智能体的改动合并回主分支？**

传统 Git 合并（`git merge`）是**基于文本的**：它看到的是字符序列的差异，不理解代码语义。如果两个智能体都修改了同一函数（一个添加参数，另一个修改函数体），Git 可能会标记冲突，尽管这两个改动在语义上完全兼容。

Auto-Claude 的 `merge/` 模块实现了**意图感知的语义合并（Intent-Aware Semantic Merge）**。

### 合并系统架构

```
多个 Worktree 的改动
         │
         ▼
  FileEvolutionTracker          ← 追踪每个任务对每个文件的改动历史
         │
         ▼
   SemanticAnalyzer             ← 将文本差异解析为语义变更（添加函数/修改参数/删除类...）
         │
         ▼
   ConflictDetector             ← 检测语义变更之间的兼容性
         │
    ┌────┴────┐
    │         │
    ▼         ▼
AutoMerger  AIResolver          ← 确定性合并 vs AI 辅助合并
    │         │
    └────┬────┘
         │
         ▼
    MergeReport                 ← 合并结果报告
```

整个流程由 `MergeOrchestrator` 统一协调。

## 5.7 语义变更类型

`SemanticAnalyzer` 将文本差异解析为高层次的语义操作：

```python
# merge/types.py（概念化）

class ChangeType(Enum):
    # 结构变更
    FUNCTION_ADDED = "function_added"
    FUNCTION_MODIFIED = "function_modified"
    FUNCTION_DELETED = "function_deleted"
    CLASS_ADDED = "class_added"
    CLASS_MODIFIED = "class_modified"

    # 接口变更
    PARAMETER_ADDED = "parameter_added"
    PARAMETER_REMOVED = "parameter_removed"
    RETURN_TYPE_CHANGED = "return_type_changed"

    # 内容变更
    IMPORT_ADDED = "import_added"
    IMPORT_REMOVED = "import_removed"
    CONSTANT_CHANGED = "constant_changed"
    BODY_MODIFIED = "body_modified"
```

这个分类使得冲突检测可以基于**语义**而不是**文本位置**：

| 变更 A | 变更 B | 判断 |
|--------|--------|------|
| 添加函数 `foo()` | 添加函数 `bar()` | 兼容（可自动合并）|
| 添加导入 `import os` | 添加导入 `import sys` | 兼容（可自动合并）|
| 修改函数 `foo()` 的参数 | 修改函数 `foo()` 的函数体 | 可能兼容（需分析）|
| 删除函数 `foo()` | 调用函数 `foo()` | 冲突（需 AI 或人工）|
| 修改同一常量的值 | 修改同一常量的值（不同值）| 冲突（需 AI 或人工）|

## 5.8 冲突分类与解决策略

`ConflictDetector` 分析语义变更后，`ConflictResolver` 根据冲突类型选择解决策略：

```python
class MergeDecision(Enum):
    AUTO_MERGED = "auto_merged"           # 确定性规则可以解决
    AI_MERGED = "ai_merged"               # AI 辅助解决了歧义
    DIRECT_COPY = "direct_copy"           # 只有一个任务修改了此文件
    NEEDS_HUMAN_REVIEW = "needs_human_review"  # 需要人工介入
    FAILED = "failed"                     # 合并失败
```

**AUTO_MERGED**：两个任务的改动在语义上兼容，可以机械地合并。例如两个任务分别添加了不同的函数——只需将两个函数都写入最终文件。

**DIRECT_COPY**：只有一个任务修改了这个文件，无需合并逻辑，直接从该任务的 worktree 读取文件内容即可。这是最常见的情况（大多数文件只被一个任务修改）。

**AI_MERGED**：两个任务修改了同一区域，但改动可能兼容（例如修改了同一函数的不同方面）。规则无法确定，交由 AI 智能体理解代码意图后生成合并结果。

**NEEDS_HUMAN_REVIEW**：涉及语义删除、接口变更、或 AI 也无法确定的冲突。标记后在 Kanban UI 中向用户高亮显示。

## 5.9 MergeOrchestrator：管道的统一入口

```python
class MergeOrchestrator:
    """
    合并流水线的统一协调者。

    最大化自动化，最小化 AI Token 使用。
    """

    def merge_task(
        self,
        task_id: str,
        worktree_path: Path | None = None,
        target_branch: str = "main",
        progress_callback: MergeProgressCallback | None = None,
    ) -> MergeReport:
```

合并一个任务的完整流程：

```python
# 阶段进度报告（前端实时更新用）
# 0%   → ANALYZING:          开始语义分析
# 25%  → DETECTING_CONFLICTS: 冲突检测
# 50%  → RESOLVING:           逐文件合并
# 75%  → VALIDATING:          结果验证
# 100% → COMPLETE:            完成

def merge_task(self, task_id, worktree_path=None, ...):
    report = MergeReport(started_at=datetime.now())

    try:
        # 1. 加载文件演化数据
        self.evolution_tracker.refresh_from_git(task_id, worktree_path)

        # 2. 获取此任务修改的文件列表
        modifications = self.evolution_tracker.get_task_modifications(task_id)

        # 3. 逐文件合并
        for file_path, snapshot in modifications:
            result = self._merge_file(file_path, [snapshot], target_branch)

            # 特殊处理：DIRECT_COPY 需要从 worktree 读取实际文件
            if result.decision == MergeDecision.DIRECT_COPY:
                content, success = self._read_worktree_file(file_path, worktree_path)
                if success:
                    result.merged_content = content
                else:
                    result.decision = MergeDecision.FAILED

            report.file_results[file_path] = result
            self._update_stats(report.stats, result)

    except Exception as e:
        report.success = False
        report.error = str(e)

    report.completed_at = datetime.now()
    return report
```

注意 `DIRECT_COPY` 的特殊处理：语义分析系统识别出"只有一个任务修改了此文件"，但合并逻辑不去重新生成文件内容，而是**直接从 worktree 读取原始文件**。这避免了任何潜在的内容损失——最可靠的内容来源永远是智能体实际写的那个文件。

## 5.10 多任务并行合并

合并多个并行任务时，`merge_tasks()` 需要处理更复杂的情况：

```python
def merge_tasks(
    self,
    requests: list[TaskMergeRequest],
    target_branch: str = "main",
) -> MergeReport:
    # 1. 按优先级排序（优先级高的任务在冲突时优先）
    requests = sorted(requests, key=lambda r: -r.priority)

    # 2. 为所有任务刷新演化数据
    for request in requests:
        self.evolution_tracker.refresh_from_git(
            request.task_id, request.worktree_path
        )

    # 3. 找出被任意任务修改的所有文件
    task_ids = [r.task_id for r in requests]
    file_tasks = self.evolution_tracker.get_files_modified_by_tasks(task_ids)
    # file_tasks: {"src/auth.py": ["task-001", "task-002"], "src/db.py": ["task-002"], ...}

    # 4. 逐文件合并（考虑多任务冲突）
    for file_path, modifying_tasks in file_tasks.items():
        # 获取所有修改此文件的任务的语义快照
        snapshots = [
            evolution.get_task_snapshot(tid)
            for tid in modifying_tasks
        ]
        result = self._merge_file(file_path, snapshots, target_branch)
        report.file_results[file_path] = result
```

`file_tasks` 是关键数据结构：一个文件路径到任务列表的映射。它揭示了哪些文件有并发修改（需要合并），哪些只被单个任务修改（可以直接复制）。

## 5.11 合并结果统计

`MergeStats` 追踪合并操作的各项指标：

```python
@dataclass
class MergeStats:
    files_processed: int = 0
    files_auto_merged: int = 0        # 规则自动解决
    files_ai_merged: int = 0          # AI 辅助解决
    files_need_review: int = 0        # 需要人工
    files_failed: int = 0             # 失败

    conflicts_detected: int = 0
    conflicts_auto_resolved: int = 0
    conflicts_ai_resolved: int = 0

    ai_calls_made: int = 0            # AI 调用次数（成本指标）
    estimated_tokens_used: int = 0    # Token 用量（成本指标）
    duration_seconds: float = 0.0

    @property
    def auto_merge_rate(self) -> float:
        """无 AI 自动解决率——越高越省钱"""
        if self.conflicts_detected == 0:
            return 1.0
        return self.conflicts_auto_resolved / self.conflicts_detected
```

`auto_merge_rate` 是一个重要的**系统健康指标**：高自动合并率意味着智能体的改动大多是独立的，系统运行顺畅；低自动合并率意味着任务划分不够清晰，智能体频繁修改相同区域。

## 5.12 并行策略的边界：何时不该并行

理解 worktree 隔离机制后，更重要的是理解**什么情况下不应该并行**。Auto-Claude 的 Planner 在生成实现计划时必须分析任务依赖：

```
可以并行的条件：
1. 任务 A 的输出不是任务 B 的输入
2. 任务 A 和 B 不修改相同的核心文件（如共享接口定义）
3. 任务 A 和 B 的语义变更类型大概率兼容

不应并行的情况：
1. 任务 B 依赖任务 A 实现的功能（数据依赖）
2. 两个任务都需要修改相同的核心模块
3. 一个任务需要重构另一个任务正在开发的代码
```

在 `planner.md` 提示词中，Planner 被要求输出每个子任务的 `can_parallelize` 标志和 `depends_on` 列表，正是基于这些考量。

## 5.13 完整的并行执行时间轴

把所有组件放在一起，一个典型的并行任务执行流程如下：

```
用户创建任务："实现用户认证和支付功能"
         │
         ▼
Planner 分析依赖，生成实现计划：
  子任务 1：实现用户认证（can_parallelize: true）
  子任务 2：实现支付 API（can_parallelize: true）
  子任务 3：集成测试（depends_on: [1, 2]）
         │
         ├──────────────────────────────────────┐
         ▼                                      ▼
create_worktree("001-auth")          create_worktree("002-payment")
  → .auto-claude/worktrees/auth/       → .auto-claude/worktrees/payment/
  → 分支: auto-claude/001-auth         → 分支: auto-claude/002-payment
         │                                      │
         ▼                                      ▼
Coder Agent A（在 auth/ 中工作）     Coder Agent B（在 payment/ 中工作）
  - 创建 auth/models.py               - 创建 payment/gateway.py
  - 创建 auth/handlers.py             - 修改 config.py（添加支付配置）
  - 修改 config.py（添加 JWT 配置）   - 创建 tests/test_payment.py
  - 创建 tests/test_auth.py
         │                                      │
         ▼                                      ▼
QA Agent A 验证 auth/               QA Agent B 验证 payment/
         │                                      │
         └──────────────────┬───────────────────┘
                            ▼
                    MergeOrchestrator
                      分析 config.py（两个任务都修改了！）
                        → SemanticAnalyzer：A 添加 JWT 配置，B 添加支付配置
                        → ConflictDetector：两个 IMPORT_ADDED + CONFIG_ADDED，兼容
                        → AutoMerger：合并两段配置
                        → MergeDecision: AUTO_MERGED ✓

                      auth/models.py → DIRECT_COPY（只有 A 修改）
                      payment/gateway.py → DIRECT_COPY（只有 B 修改）
                            │
                            ▼
                    结果合并到 develop 分支
                    用户在 Kanban 中看到完成状态
```

## 5.14 WorktreeInfo：任务状态的快照

`WorktreeInfo` 数据类捕获了 worktree 在某个时刻的完整状态：

```python
@dataclass
class WorktreeInfo:
    path: Path
    branch: str
    spec_name: str
    base_branch: str
    is_active: bool = True
    commit_count: int = 0         # 此 worktree 的提交数
    files_changed: int = 0        # 修改的文件数
    additions: int = 0            # 新增行数
    deletions: int = 0            # 删除行数
    last_commit_date: datetime | None = None
    days_since_last_commit: int | None = None
```

这些数据在 Kanban UI 中显示为任务卡片上的统计信息，让用户直观了解每个并行任务的进展。

---

## Lab 5.1：实现微型 Worktree 管理器

在这个实验中，你将实现一个简化的 worktree 管理器和语义合并系统，深入理解并行隔离的机制。

### 背景

你正在构建一个文档协作系统，多个 AI 写作智能体可以同时编辑不同章节。你需要：
1. 为每个写作任务创建隔离的工作空间
2. 追踪每个任务对文件的修改
3. 将多个任务的修改智能合并

### Part A：工作空间管理

```python
# lab_05_worktree.py

import os
import subprocess
import json
from dataclasses import dataclass, field
from pathlib import Path
from typing import Optional

@dataclass
class TaskWorkspace:
    """代表一个写作任务的独立工作空间。"""
    task_id: str
    path: Path
    branch: str
    base_branch: str
    files_created: list[str] = field(default_factory=list)
    files_modified: list[str] = field(default_factory=list)

class WorkspaceManager:
    """管理写作任务的独立工作空间。"""

    def __init__(self, project_dir: Path):
        self.project_dir = project_dir
        self.workspaces_dir = project_dir / ".workspaces"
        self.workspaces_dir.mkdir(exist_ok=True)

    def create_workspace(self, task_id: str) -> TaskWorkspace:
        """
        TODO Part A: 为任务创建独立工作空间

        要求：
        1. 在 self.workspaces_dir / task_id 创建目录
        2. 将 project_dir 的内容"快照"到工作空间（简化版：复制关键文件）
        3. 记录基础状态（用于后续对比）
        4. 返回 TaskWorkspace

        提示：在真实系统中这是 git worktree add，
        在本实验中用目录复制模拟隔离效果。
        """
        workspace_path = self.workspaces_dir / task_id
        workspace_path.mkdir(exist_ok=True)

        # TODO: 复制项目文件到工作空间
        # TODO: 记录基础状态快照
        # TODO: 返回 TaskWorkspace
        pass

    def get_modifications(self, task_id: str) -> dict[str, str]:
        """
        TODO Part A: 检测工作空间相对于基础状态的修改

        返回 {文件路径: "created" | "modified" | "deleted"}

        提示：对比工作空间文件与基础状态快照
        """
        pass

    def list_workspaces(self) -> list[TaskWorkspace]:
        """返回所有活跃的工作空间。"""
        workspaces = []
        for task_dir in self.workspaces_dir.iterdir():
            if task_dir.is_dir() and not task_dir.name.startswith("_"):
                # TODO: 读取并返回工作空间信息
                pass
        return workspaces
```

### Part B：语义合并

```python
from enum import Enum

class MergeResult(Enum):
    AUTO_MERGED = "auto_merged"
    CONFLICT = "conflict"
    DIRECT_COPY = "direct_copy"

@dataclass
class FileMergeResult:
    file_path: str
    decision: MergeResult
    content: Optional[str] = None
    conflict_info: Optional[str] = None

class SimpleMerger:
    """
    简化的语义合并器。

    合并规则：
    - 只有一个任务修改 → DIRECT_COPY
    - 多个任务修改，无重叠行 → AUTO_MERGED（将所有修改合并）
    - 多个任务修改同一行 → CONFLICT（需要人工）
    """

    def merge_file(
        self,
        file_path: str,
        baseline_content: str,
        task_modifications: dict[str, str],  # {task_id: modified_content}
    ) -> FileMergeResult:
        """
        TODO Part B: 实现文件合并逻辑

        参数：
        - baseline_content: 合并前的原始文件内容
        - task_modifications: 每个任务修改后的文件内容

        要求：
        1. 如果只有一个任务修改 → DIRECT_COPY
        2. 如果多个任务修改：
           a. 计算每个任务相对于 baseline 的行级差异
           b. 检查是否有行级冲突（不同任务修改了同一行）
           c. 无冲突 → 合并所有差异 → AUTO_MERGED
           d. 有冲突 → 返回 CONFLICT + 冲突说明

        提示：使用 difflib 库进行差异计算
        """
        if len(task_modifications) == 0:
            return FileMergeResult(
                file_path=file_path,
                decision=MergeResult.DIRECT_COPY,
                content=baseline_content,
            )

        if len(task_modifications) == 1:
            task_id, content = next(iter(task_modifications.items()))
            return FileMergeResult(
                file_path=file_path,
                decision=MergeResult.DIRECT_COPY,
                content=content,
            )

        # TODO: 实现多任务合并逻辑
        pass

    def _compute_line_changes(
        self,
        baseline: str,
        modified: str,
    ) -> dict[int, str]:
        """
        TODO Part B (辅助函数): 计算修改的行号映射

        返回 {行号: 新内容}，表示相对于 baseline 改变了哪些行。

        提示：使用 difflib.ndiff() 或 unified_diff()
        """
        pass
```

### Part C：整合测试

```python
def test_parallel_writing():
    """
    完整测试：模拟两个写作智能体并行工作，然后合并。
    """
    import tempfile

    with tempfile.TemporaryDirectory() as project_dir:
        project_dir = Path(project_dir)

        # 创建初始文档
        doc_path = project_dir / "document.md"
        doc_path.write_text("""# 文档标题

## 第一节
这是第一节的内容。

## 第二节
这是第二节的内容。
""")

        manager = WorkspaceManager(project_dir)
        merger = SimpleMerger()

        # 模拟两个写作任务
        ws1 = manager.create_workspace("task-writer-a")
        ws2 = manager.create_workspace("task-writer-b")

        # Writer A 修改第一节
        (ws1.path / "document.md").write_text("""# 文档标题

## 第一节
这是第一节的内容，Writer A 添加了更多细节。
Writer A 新增的额外说明。

## 第二节
这是第二节的内容。
""")

        # Writer B 修改第二节
        (ws2.path / "document.md").write_text("""# 文档标题

## 第一节
这是第一节的内容。

## 第二节
这是第二节的内容，Writer B 进行了扩充。
Writer B 添加了实例和说明。
""")

        # 合并
        baseline = doc_path.read_text()
        modifications = {
            "task-writer-a": (ws1.path / "document.md").read_text(),
            "task-writer-b": (ws2.path / "document.md").read_text(),
        }

        result = merger.merge_file("document.md", baseline, modifications)

        print(f"合并决策: {result.decision.value}")
        print(f"合并结果:\n{result.content}")

        # 验证
        assert result.decision == MergeResult.AUTO_MERGED, \
            f"Expected AUTO_MERGED, got {result.decision}"
        assert "Writer A 新增的额外说明" in result.content, \
            "A 的修改应该在合并结果中"
        assert "Writer B 添加了实例和说明" in result.content, \
            "B 的修改应该在合并结果中"

        print("✓ 并行写作合并测试通过！")


def test_conflict_detection():
    """测试：两个任务修改同一行时应报告冲突。"""
    merger = SimpleMerger()

    baseline = "第一行\n第二行\n第三行\n"
    modifications = {
        "task-a": "第一行\n任务A修改了第二行\n第三行\n",
        "task-b": "第一行\n任务B也修改了第二行\n第三行\n",
    }

    result = merger.merge_file("test.txt", baseline, modifications)
    assert result.decision == MergeResult.CONFLICT, \
        "同行修改应报告冲突"
    assert result.conflict_info is not None, \
        "应提供冲突说明"

    print(f"✓ 冲突检测正确: {result.conflict_info}")


if __name__ == "__main__":
    test_parallel_writing()
    test_conflict_detection()
    print("\n✓ All tests passed!")
```

### 验证标准

| 检查点 | 预期结果 |
|--------|---------|
| 单任务修改 | 返回 DIRECT_COPY |
| 两任务修改不同行 | 返回 AUTO_MERGED，内容包含两个任务的修改 |
| 两任务修改同一行 | 返回 CONFLICT，包含冲突说明 |
| 工作空间隔离 | 修改一个工作空间不影响另一个 |

### 进阶挑战

1. **重试机制**：给工作空间创建操作添加重试逻辑——模拟随机失败（用 `random.random() < 0.3` 触发失败），验证指数退避是否有效。

2. **优先级合并**：在 `merge_file()` 中添加 `task_priorities: dict[str, int]` 参数。当两个任务有冲突时，优先级高的任务的修改胜出（而不是报告 CONFLICT）。

3. **合并报告**：实现 `MergeReport` 类，统计自动合并率、冲突数量、处理的文件总数，并格式化打印类似 `MergeStats` 的报告。

---

**本章要点回顾**

- Git worktree 让多个智能体在同一仓库中并行工作，每个有自己的独立文件系统视图
- 1:1:1 映射（spec → worktree → branch）是隔离的基础
- 指数退避重试是处理网络操作瞬时失败的标准模式
- 语义合并比文本合并更智能：理解代码结构，而不是字符序列
- 合并决策层次：AUTO_MERGED → AI_MERGED → DIRECT_COPY → NEEDS_HUMAN_REVIEW
- `auto_merge_rate` 是衡量任务划分质量的关键指标
- 并行合并操作需要互斥锁，防止主分支出现竞争条件
