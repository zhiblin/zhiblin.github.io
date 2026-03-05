---
title: 第十二章：后台服务与恢复编排
date: 2026-01-12 00:00:00
categories: Agents
tags: [AI, Agent, Python, Claude, SDK]
---

# 第十二章：后台服务与恢复编排

> "一个真正能'放手让 AI 工作'的系统，需要解决一个根本问题：当 AI 在无人监督的情况下遇到麻烦，它应该怎么办？"

---

## 12.1 从"能自动化"到"能自愈"

Auto-Claude 的核心价值主张是**自主性**——用户启动一个任务后，AI 应该能够持续工作，而不需要人工反复干预。但真正的自主性不只是"能自动执行"，还包括"能自动恢复"。

`apps/backend/services/` 目录中的三个模块构成了这套自愈能力：

- `recovery.py`：**恢复管理器**——追踪每次尝试的历史，分析失败原因，决定恢复动作
- `orchestrator.py`：**服务编排器**——在 QA 测试前自动启动多服务环境（Docker Compose、Monorepo）
- `context.py`：**上下文服务**——跨会话的上下文构建辅助

本章聚焦前两者，因为它们代表了两种最重要的自愈模式：**知识型恢复**（记住什么失败了）和**环境型恢复**（确保测试环境就绪）。

---

## 12.2 `RecoveryManager`：构建自愈记忆

`apps/backend/services/recovery.py` 中的 `RecoveryManager` 是一个有状态的失败管理系统，它的核心数据由两个 JSON 文件持久化：

```
spec/memory/
├── attempt_history.json    # 每个子任务的尝试历史
└── build_commits.json      # 已验证的"好 commit"列表
```

### 12.2.1 失败类型分类

```python
# apps/backend/services/recovery.py（简化）

class FailureType(Enum):
    BROKEN_BUILD      = "broken_build"       # 代码无法编译/运行
    VERIFICATION_FAILED = "verification_failed"  # 验证测试失败
    CIRCULAR_FIX      = "circular_fix"       # 同一修复方案反复尝试
    CONTEXT_EXHAUSTED = "context_exhausted"  # 上下文窗口耗尽
    UNKNOWN           = "unknown"

def classify_failure(self, error: str, subtask_id: str) -> FailureType:
    error_lower = error.lower()

    # 语法错误、导入失败 → 代码损坏
    build_errors = ["syntax error", "compilation error",
                    "module not found", "import error", "unexpected token"]
    if any(be in error_lower for be in build_errors):
        return FailureType.BROKEN_BUILD

    # 断言失败、HTTP 状态码错误 → 验证失败
    verification_errors = ["verification failed", "expected", "assertion",
                           "test failed", "status code"]
    if any(ve in error_lower for ve in verification_errors):
        return FailureType.VERIFICATION_FAILED

    # 检查循环修复（需要历史数据）
    if self.is_circular_fix(subtask_id, error):
        return FailureType.CIRCULAR_FIX

    return FailureType.UNKNOWN
```

不同失败类型对应不同的恢复策略，这是整个系统的核心判断逻辑。

### 12.2.2 循环修复检测

最有趣的失败类型是 `CIRCULAR_FIX`——智能体陷入了"修复-破坏-再修复"的死循环，每次尝试本质上都是相同的思路。

`is_circular_fix()` 使用 **Jaccard 相似度**来检测这种情况：

```python
# apps/backend/services/recovery.py（简化）

def is_circular_fix(self, subtask_id: str, current_approach: str) -> bool:
    """检测是否在重复相同的修复方案。"""
    attempts = self.get_subtask_history(subtask_id)["attempts"]

    if len(attempts) < 2:
        return False

    # 取最近 3 次尝试
    recent = attempts[-3:] if len(attempts) >= 3 else attempts

    # 提取当前方案的关键词（去停用词）
    stop_words = {"with", "using", "the", "a", "and", "or", "trying"}
    current_kw = {w for w in current_approach.lower().split()
                  if w not in stop_words}

    similar_count = 0
    for attempt in recent:
        attempt_kw = {w for w in attempt["approach"].lower().split()
                      if w not in stop_words}

        # Jaccard 相似度 = 交集 / 并集
        overlap = len(current_kw & attempt_kw)
        total = len(current_kw | attempt_kw)
        similarity = overlap / total if total > 0 else 0

        # >30% 关键词重叠 → 视为相似方案
        if similarity > 0.3:
            similar_count += 1

    # 最近 2+ 次都是相似方案 → 循环检测触发
    return similar_count >= 2
```

`★ Insight ─────────────────────────────────────`
Jaccard 相似度（`|A∩B| / |A∪B|`）是文本相似性的轻量级计算方式，无需 NLP 库，且对词序不敏感。**30% 阈值**是一个工程权衡：太低（10%）会错误地将不同方案标记为重复；太高（50%）会让真正的循环逃过检测。系统选择了偏保守的 30%，在宁可误判也不要错过循环的方向上权衡。
`─────────────────────────────────────────────────`

### 12.2.3 时间窗口下的尝试计数

`get_attempt_count()` 不是简单地统计历史中所有尝试，而是只计算**最近 2 小时内**的尝试：

```python
# apps/backend/services/recovery.py（简化）

ATTEMPT_WINDOW_SECONDS = 7200  # 2 小时

def get_attempt_count(self, subtask_id: str) -> int:
    """只统计最近 2 小时内的尝试次数。"""
    attempts = self.get_subtask_history(subtask_id)["attempts"]
    cutoff = datetime.now(timezone.utc) - timedelta(seconds=ATTEMPT_WINDOW_SECONDS)

    recent_count = 0
    for attempt in attempts:
        try:
            attempt_time = datetime.fromisoformat(attempt["timestamp"])
            if attempt_time >= cutoff:
                recent_count += 1
        except (KeyError, ValueError):
            recent_count += 1  # 时间戳缺失时保守计入
    return recent_count
```

这个设计解决了"跨崩溃重启的公平重试"问题。如果系统崩溃后重启，昨天失败的尝试不应该影响今天的重试预算。2 小时窗口确保了这种公平性。

另外，每个子任务最多存储 50 条历史记录（`MAX_ATTEMPT_HISTORY_PER_SUBTASK`），防止长时间运行的项目积累无限多的历史。

### 12.2.4 五种恢复决策矩阵

```python
# apps/backend/services/recovery.py（简化）

def determine_recovery_action(self, failure_type, subtask_id) -> RecoveryAction:
    attempt_count = self.get_attempt_count(subtask_id)

    if failure_type == FailureType.BROKEN_BUILD:
        # 代码损坏 → 立刻回滚到上一个好 commit
        last_good = self.get_last_good_commit()
        if last_good:
            return RecoveryAction("rollback", last_good,
                                  "Build broken, rolling back to working state")
        else:
            return RecoveryAction("escalate", subtask_id,
                                  "Build broken and no good commit to rollback to")

    elif failure_type == FailureType.VERIFICATION_FAILED:
        # 验证失败：前 3 次重试，之后跳过
        if attempt_count < 3:
            return RecoveryAction("retry", subtask_id,
                                  f"Retry with different approach ({attempt_count+1}/3)")
        else:
            return RecoveryAction("skip", subtask_id,
                                  f"Failed after {attempt_count} attempts")

    elif failure_type == FailureType.CIRCULAR_FIX:
        return RecoveryAction("skip", subtask_id,
                              "Circular fix detected - same approach repeated")

    elif failure_type == FailureType.CONTEXT_EXHAUSTED:
        # 上下文耗尽：提交已有进展，下次会话继续
        return RecoveryAction("continue", subtask_id,
                              "Context exhausted, will continue in next session")

    else:  # UNKNOWN
        if attempt_count < 2:
            return RecoveryAction("retry", subtask_id, "Unknown error, retrying")
        else:
            return RecoveryAction("escalate", subtask_id, "Persists after retries")
```

| 失败类型 | 优先动作 | 次要动作 | 关键判断条件 |
|---------|---------|---------|------------|
| BROKEN_BUILD | rollback | escalate | 有无好 commit |
| VERIFICATION_FAILED | retry | skip | 尝试次数 < 3 |
| CIRCULAR_FIX | skip | — | 直接跳过 |
| CONTEXT_EXHAUSTED | continue | — | 直接继续 |
| UNKNOWN | retry | escalate | 尝试次数 < 2 |

### 12.2.5 记录好 Commit 与回滚

```python
def record_good_commit(self, commit_hash: str, subtask_id: str) -> None:
    """子任务成功完成时，记录当前 commit 为安全点。"""
    commits = self._load_build_commits()
    commits["commits"].append({
        "hash": commit_hash,
        "subtask_id": subtask_id,
        "timestamp": datetime.now(timezone.utc).isoformat(),
    })
    commits["last_good_commit"] = commit_hash
    self._save_build_commits(commits)

def rollback_to_commit(self, commit_hash: str) -> bool:
    """执行 git reset --hard 回滚代码。"""
    result = subprocess.run(
        ["git", "reset", "--hard", commit_hash],
        cwd=self.project_dir,
        capture_output=True, text=True, check=True,
    )
    return True
```

`record_good_commit()` 在 `post_session_processing()` 的成功路径上被调用——每当一个子任务成功完成，当前的 commit 就被"存档"为安全回滚点。

---

## 12.3 `ServiceOrchestrator`：让测试环境自动就绪

自动化测试的一个常见痛点是：测试需要一堆服务运行（数据库、API 服务、消息队列），但这些服务需要手动启动。`ServiceOrchestrator` 解决了这个问题。

### 12.3.1 服务发现：三种模式

```python
# apps/backend/services/orchestrator.py（简化）

class ServiceOrchestrator:
    def _discover_services(self) -> None:
        # 优先：查找 docker-compose 文件
        self._compose_file = self._find_compose_file()

        if self._compose_file:
            self._parse_compose_services()
        else:
            # 备用：扫描 Monorepo 子目录
            self._discover_monorepo_services()

    def _find_compose_file(self) -> Path | None:
        """按优先级查找 docker-compose 配置文件。"""
        candidates = [
            "docker-compose.yml", "docker-compose.yaml",
            "compose.yml", "compose.yaml",
            "docker-compose.dev.yml", "docker-compose.dev.yaml",
        ]
        for candidate in candidates:
            if (self.project_dir / candidate).exists():
                return self.project_dir / candidate
        return None

    def _discover_monorepo_services(self) -> None:
        """扫描 Monorepo 结构发现子服务。"""
        service_dirs = ["services", "packages", "apps", "microservices"]
        for service_dir in service_dirs:
            dir_path = self.project_dir / service_dir
            if dir_path.exists():
                for item in dir_path.iterdir():
                    if item.is_dir() and self._is_service_directory(item):
                        self._services.append(ServiceConfig(
                            name=item.name,
                            path=item.relative_to(self.project_dir).as_posix(),
                            type="local",
                        ))

    def _is_service_directory(self, path: Path) -> bool:
        """判断目录是否包含一个服务。"""
        indicators = [
            "package.json", "pyproject.toml", "requirements.txt",
            "Dockerfile", "main.py", "app.py", "index.ts",
        ]
        return any((path / ind).exists() for ind in indicators)
```

### 12.3.2 Docker Compose 版本兼容

Docker Compose 有两个版本：独立工具 `docker-compose`（v1）和 Docker 插件 `docker compose`（v2）。系统优雅地处理了这个兼容性问题：

```python
def _get_docker_compose_cmd(self) -> list[str] | None:
    """获取 docker-compose 命令（兼容 v1 和 v2）。"""
    # 先尝试 v2（Docker 插件格式）
    try:
        proc = subprocess.run(["docker", "compose", "version"],
                              capture_output=True, timeout=5)
        if proc.returncode == 0:
            return ["docker", "compose", "-f", str(self._compose_file)]
    except Exception:
        pass

    # 再尝试 v1（独立工具格式）
    try:
        proc = subprocess.run(["docker-compose", "version"],
                              capture_output=True, timeout=5)
        if proc.returncode == 0:
            return ["docker-compose", "-f", str(self._compose_file)]
    except Exception:
        pass

    return None  # 都没有找到
```

### 12.3.3 健康检查循环

启动服务后，系统不会立即运行测试，而是等待所有服务就绪：

```python
def _wait_for_health(self, timeout: int) -> bool:
    """等待所有服务的端口响应。"""
    start_time = time.time()
    while time.time() - start_time < timeout:
        all_healthy = True
        for service in self._services:
            if service.port:
                if not self._check_port(service.port):
                    all_healthy = False
                    break
        if all_healthy:
            return True
        time.sleep(2)  # 每 2 秒检查一次
    return False

def _check_port(self, port: int) -> bool:
    """用 TCP connect 检查端口是否响应。"""
    import socket
    try:
        with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
            s.settimeout(1)
            return s.connect_ex(("localhost", port)) == 0
    except Exception:
        return False
```

### 12.3.4 上下文管理器 API

最优雅的部分是 `ServiceContext`——用 Python 上下文管理器模式封装了"启动→测试→停止"的整个生命周期：

```python
class ServiceContext:
    """
    用法：
        with ServiceContext(project_dir) as ctx:
            if ctx.success:
                run_tests()
        # 退出 with 块后，服务自动停止
    """
    def __enter__(self) -> "ServiceContext":
        if self.orchestrator.is_multi_service():
            self.result = self.orchestrator.start_services(self.timeout)
        return self

    def __exit__(self, exc_type, exc_val, exc_tb) -> None:
        self.orchestrator.stop_services()  # 无论测试是否成功，都要停止
```

`★ Insight ─────────────────────────────────────`
上下文管理器（`with` 语句）是 Python 中**资源安全管理**的标准模式。即使测试过程中抛出异常，`__exit__` 也能保证服务被停止，避免僵尸进程占用端口。这与 Go 的 `defer`、Java 的 `try-with-resources` 原理相同，但 Python 的实现更为直观。
`─────────────────────────────────────────────────`

---

## 12.4 本地服务的安全启动

对于非 Docker 的本地服务，`_start_local_services()` 展示了一个重要的安全细节：

```python
def _start_local_services(self, timeout: int) -> OrchestrationResult:
    for service in self._services:
        if service.startup_command:
            # 使用 shlex.split() 安全解析命令字符串
            # shell=False 防止 shell 注入攻击
            proc = subprocess.Popen(
                shlex.split(service.startup_command),
                shell=False,   # 关键：不通过 shell 执行
                cwd=...,
                stdout=subprocess.PIPE,
                stderr=subprocess.PIPE,
            )
```

`shlex.split()` 按 shell 语法规则拆分命令字符串（正确处理引号和转义），同时 `shell=False` 阻止 shell 解释器介入。如果 `startup_command` 来自用户输入，这两者的结合是防止命令注入的最小代价方案。

---

## 12.5 整体自愈流程图

```
子任务执行完毕
    │
    ├── 成功 ──→ record_good_commit() ──→ 下一子任务
    │
    └── 失败 ──→ classify_failure()
                    │
                    ├── BROKEN_BUILD ──────────────────→ rollback_to_commit()
                    │                                       ↓
                    │                                   重试同子任务
                    │
                    ├── VERIFICATION_FAILED
                    │   ├── attempts < 3 ─────────────→ retry（不同思路）
                    │   └── attempts >= 3 ────────────→ skip（标记为 stuck）
                    │
                    ├── CIRCULAR_FIX ─────────────────→ skip（直接跳过）
                    │
                    ├── CONTEXT_EXHAUSTED ────────────→ continue（下次会话继续）
                    │
                    └── UNKNOWN
                        ├── attempts < 2 ─────────────→ retry
                        └── attempts >= 2 ────────────→ escalate（人工介入）
```

---

## 本章要点回顾

| 概念 | 实现位置 | 关键点 |
|------|---------|-------|
| 五种失败类型 | `services/recovery.py: FailureType` | BROKEN_BUILD/VERIFICATION_FAILED/CIRCULAR_FIX/CONTEXT_EXHAUSTED/UNKNOWN |
| Jaccard 相似度循环检测 | `is_circular_fix()` | 30% 阈值，最近 3 次中有 2 次相似 |
| 时间窗口尝试计数 | `get_attempt_count()` | 只统计 2 小时内的尝试 |
| 好 Commit 存档 | `record_good_commit()` | 子任务成功完成时存档 |
| 服务自动发现 | `ServiceOrchestrator._discover_services()` | docker-compose → monorepo 子目录 |
| Docker v1/v2 兼容 | `_get_docker_compose_cmd()` | 先试 v2，再试 v1 |
| 上下文管理器封装 | `ServiceContext` | `with` 语句保证服务最终停止 |
| 安全启动 | `shlex.split() + shell=False` | 防止命令注入 |

---

## Lab 12.1：迷你自愈恢复管理器

### Part A：失败分类与 Jaccard 检测（必做）

```python
# lab_12_recovery.py
from __future__ import annotations

import json
import subprocess
from dataclasses import dataclass, field
from datetime import datetime, timedelta, timezone
from enum import Enum
from pathlib import Path


class FailureType(Enum):
    BROKEN_BUILD        = "broken_build"
    VERIFICATION_FAILED = "verification_failed"
    CIRCULAR_FIX        = "circular_fix"
    CONTEXT_EXHAUSTED   = "context_exhausted"
    UNKNOWN             = "unknown"


@dataclass
class RecoveryAction:
    action: str   # "rollback" | "retry" | "skip" | "escalate" | "continue"
    target: str
    reason: str


def compute_jaccard_similarity(text_a: str, text_b: str) -> float:
    """
    计算两段文本的 Jaccard 相似度（词级别）。

    TODO: 实现 Jaccard 相似度（5-8 行）：
    1. 将两段文本分别分词（split()）并转为小写
    2. 过滤停用词：{"with", "using", "the", "a", "an", "and", "or", "trying", "to"}
    3. 转为集合
    4. 计算 |A ∩ B| / |A ∪ B|
    5. 如果并集为空，返回 0.0

    例：
    text_a = "fix async await handling"
    text_b = "fix async await in middleware"
    交集 = {"fix", "async", "await"}，并集 = {"fix", "async", "await", "handling", "in", "middleware"}
    相似度 = 3/6 = 0.5
    """
    # YOUR CODE HERE
    pass


def classify_failure(error: str, subtask_id: str,
                     attempt_history: list[dict]) -> FailureType:
    """
    根据错误信息和历史记录分类失败类型。

    TODO: 实现失败分类逻辑（20-30 行）：

    分类规则（按优先级）：
    1. 包含以下关键词 → BROKEN_BUILD：
       "syntax error", "compilation error", "module not found",
       "import error", "unexpected token", "indentation error"
    2. 包含以下关键词 → VERIFICATION_FAILED：
       "verification failed", "expected", "assertion",
       "test failed", "status code"
    3. 包含以下关键词 → CONTEXT_EXHAUSTED：
       "context exhausted", "token limit", "maximum length"
    4. 如果最近 3 次历史中有 2 次与当前 error 的 Jaccard 相似度 > 0.3 → CIRCULAR_FIX
    5. 否则 → UNKNOWN

    Args:
        error: 错误信息
        subtask_id: 子任务 ID（备用，此处不需要）
        attempt_history: 已有的尝试历史列表
    """
    # YOUR CODE HERE
    pass


def decide_recovery(
    failure_type: FailureType,
    subtask_id: str,
    attempt_count: int,
    last_good_commit: str | None,
) -> RecoveryAction:
    """
    根据失败类型和尝试次数决定恢复动作。

    TODO: 实现恢复决策逻辑（15-25 行）：

    决策矩阵：
    - BROKEN_BUILD：
        有 last_good_commit → rollback（目标=commit hash）
        无 last_good_commit → escalate
    - VERIFICATION_FAILED：
        attempt_count < 3 → retry
        否则 → skip
    - CIRCULAR_FIX：→ skip
    - CONTEXT_EXHAUSTED：→ continue
    - UNKNOWN：
        attempt_count < 2 → retry
        否则 → escalate
    """
    # YOUR CODE HERE
    pass


# === 验证测试 ===
def test_jaccard():
    assert compute_jaccard_similarity("fix async await", "fix async await") == 1.0
    assert compute_jaccard_similarity("fix auth bug", "implement new feature") < 0.2
    sim = compute_jaccard_similarity("fix async await handling", "fix async await middleware")
    assert 0.3 < sim < 0.8, f"Expected medium similarity, got {sim}"
    print("[PASS] Jaccard similarity")

def test_classify():
    history = [
        {"approach": "fix async await"},
        {"approach": "fix async await in handler"},
        {"approach": "fix async await middleware"},
    ]
    # 相似历史 → CIRCULAR_FIX
    ft = classify_failure("async await error", "t1", history)
    assert ft == FailureType.CIRCULAR_FIX, f"Expected CIRCULAR_FIX, got {ft}"

    # 语法错误 → BROKEN_BUILD
    ft = classify_failure("SyntaxError: unexpected token", "t2", [])
    assert ft == FailureType.BROKEN_BUILD

    # 无特征 → UNKNOWN
    ft = classify_failure("something went wrong", "t3", [])
    assert ft == FailureType.UNKNOWN
    print("[PASS] classify_failure")

def test_decide():
    # BROKEN_BUILD + 有好 commit → rollback
    action = decide_recovery(FailureType.BROKEN_BUILD, "t1", 1, "abc123")
    assert action.action == "rollback" and action.target == "abc123"

    # VERIFICATION_FAILED 前 3 次 → retry
    action = decide_recovery(FailureType.VERIFICATION_FAILED, "t1", 2, None)
    assert action.action == "retry"

    # VERIFICATION_FAILED 超过 3 次 → skip
    action = decide_recovery(FailureType.VERIFICATION_FAILED, "t1", 3, None)
    assert action.action == "skip"

    # CIRCULAR_FIX → skip
    action = decide_recovery(FailureType.CIRCULAR_FIX, "t1", 1, None)
    assert action.action == "skip"
    print("[PASS] decide_recovery")

if __name__ == "__main__":
    print("=== Part A: 测试 ===")
    test_jaccard()
    test_classify()
    test_decide()
```

### Part B：带历史记录的恢复管理器（必做）

```python
# 接续 lab_12_recovery.py
import tempfile

ATTEMPT_WINDOW_SECONDS = 7200  # 2 小时
MAX_HISTORY_PER_SUBTASK = 50


class MiniRecoveryManager:
    """
    带历史追踪的恢复管理器。
    """

    def __init__(self, work_dir: Path):
        self.work_dir = work_dir
        self.history_file = work_dir / "attempt_history.json"
        self.commits_file = work_dir / "build_commits.json"
        work_dir.mkdir(parents=True, exist_ok=True)
        self._init_files()

    def _init_files(self):
        if not self.history_file.exists():
            self.history_file.write_text(json.dumps({"subtasks": {}, "stuck": []}))
        if not self.commits_file.exists():
            self.commits_file.write_text(json.dumps({"commits": [], "last_good": None}))

    def _load_history(self) -> dict:
        return json.loads(self.history_file.read_text())

    def _save_history(self, data: dict):
        self.history_file.write_text(json.dumps(data, indent=2))

    def record_attempt(self, subtask_id: str, success: bool,
                       approach: str, error: str | None = None):
        """
        记录一次尝试结果。

        TODO: 实现记录逻辑（15-20 行）：
        1. 加载历史
        2. 若 subtask_id 不存在，初始化空列表
        3. 追加尝试记录：
           { "timestamp": ISO格式UTC时间, "approach": approach,
             "success": success, "error": error }
        4. 若历史超过 MAX_HISTORY_PER_SUBTASK，只保留最新的
        5. 保存历史

        提示：datetime.now(timezone.utc).isoformat()
        """
        # YOUR CODE HERE
        pass

    def record_good_commit(self, commit_hash: str, subtask_id: str):
        """记录一个已验证的好 commit。"""
        data = json.loads(self.commits_file.read_text())
        data["commits"].append({
            "hash": commit_hash,
            "subtask_id": subtask_id,
            "timestamp": datetime.now(timezone.utc).isoformat(),
        })
        data["last_good"] = commit_hash
        self.commits_file.write_text(json.dumps(data, indent=2))

    def get_last_good_commit(self) -> str | None:
        data = json.loads(self.commits_file.read_text())
        return data.get("last_good")

    def get_recent_attempt_count(self, subtask_id: str) -> int:
        """
        统计最近 ATTEMPT_WINDOW_SECONDS 内的尝试次数。

        TODO: 实现时间窗口统计（10-15 行）：
        1. 加载历史，找到 subtask_id 对应的尝试列表
        2. 计算截止时间 = now - ATTEMPT_WINDOW_SECONDS
        3. 统计 timestamp >= 截止时间 的尝试数
        4. 对无效时间戳（解析失败），保守地计入统计
        """
        # YOUR CODE HERE
        pass

    def get_recent_approaches(self, subtask_id: str, n: int = 3) -> list[str]:
        """获取最近 n 次尝试的方案描述。"""
        history = self._load_history()
        attempts = history.get("subtasks", {}).get(subtask_id, [])
        return [a["approach"] for a in attempts[-n:]]

    def analyze_and_decide(self, subtask_id: str,
                           error: str) -> RecoveryAction:
        """一站式分析并决定恢复动作。"""
        approaches = self.get_recent_approaches(subtask_id)
        failure_type = classify_failure(error, subtask_id, [
            {"approach": a} for a in approaches
        ])
        attempt_count = self.get_recent_attempt_count(subtask_id)
        last_good = self.get_last_good_commit()
        return decide_recovery(failure_type, subtask_id, attempt_count, last_good)


def test_manager():
    tmpdir = Path(tempfile.mkdtemp())
    mgr = MiniRecoveryManager(tmpdir)

    # 模拟 3 次相似失败 → CIRCULAR_FIX → skip
    for i in range(3):
        mgr.record_attempt("t1", False, "fix async await handling", "async error")

    action = mgr.analyze_and_decide("t1", "async await error")
    assert action.action in ("skip", "escalate"), f"Expected skip/escalate, got {action.action}"
    print(f"[PASS] Circular fix detected → action={action.action}")

    # 模拟代码损坏 + 有好 commit → rollback
    mgr2 = MiniRecoveryManager(Path(tempfile.mkdtemp()))
    mgr2.record_good_commit("deadbeef", "t2")
    mgr2.record_attempt("t2", False, "implement feature", "SyntaxError")
    action2 = mgr2.analyze_and_decide("t2", "SyntaxError: unexpected token")
    assert action2.action == "rollback"
    assert action2.target == "deadbeef"
    print(f"[PASS] Broken build with good commit → rollback to {action2.target[:8]}")


def test_time_window():
    """测试时间窗口统计。"""
    tmpdir = Path(tempfile.mkdtemp())
    mgr = MiniRecoveryManager(tmpdir)

    # 记录几次尝试
    for _ in range(3):
        mgr.record_attempt("t1", False, "try something", "error")

    count = mgr.get_recent_attempt_count("t1")
    assert count == 3, f"Expected 3 recent attempts, got {count}"
    print(f"[PASS] Time window count = {count}")


if __name__ == "__main__":
    print("=== Part A: 基础测试 ===")
    test_jaccard()
    test_classify()
    test_decide()

    print("\n=== Part B: 管理器测试 ===")
    test_manager()
    test_time_window()
    print("\n全部测试通过！")
```

### 验证标准
- `compute_jaccard_similarity()` 对完全相同的文本返回 1.0，完全不同的接近 0
- 失败分类正确区分 BROKEN_BUILD（语法错误）和 CIRCULAR_FIX（重复方案）
- 时间窗口统计只计算近 2 小时内的尝试
- 代码损坏+有好 commit → rollback 动作

### 进阶挑战
1. **健康检查**：实现 `ServiceHealthChecker`，支持 HTTP `/health` 端点（`requests.get`）和 TCP 端口检查两种模式，带超时和重试
2. **指数退避健康等待**：修改健康检查循环，将固定的 2 秒间隔替换为指数退避（1s → 2s → 4s → 8s → 上限 30s），加速初期检测

---

*下一章，我们转向前端工程，深入 Electron 主进程架构——了解 40+ IPC 处理器如何组织，Python 子进程如何被管理，以及前后端如何通过事件驱动模式协作。*
