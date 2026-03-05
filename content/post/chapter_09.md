---
title: 第九章：智能体会话恢复与错误处理
date: 2026-01-09 00:00:00
categories: Agents
tags: [AI, Agent, Python, Claude, SDK]
---

# 第九章：智能体会话恢复与错误处理

> "失败不是终点，而是数据。好的智能体系统不是永不出错，而是出错后能够理解错误、分类错误，并采取正确的恢复行动。"

---

## 9.1 为什么智能体需要特殊的错误处理？

传统软件的错误处理遵循一个简单逻辑：操作失败 → 抛出异常 → 调用者处理。这套逻辑在确定性代码中运行良好，因为相同的输入总是产生相同的输出，失败的原因是可分类的，修复方案也是可预测的。

但智能体系统破坏了这个假设。一个 AI 智能体在执行编码任务时可能遭遇：

- **工具并发限制（400 错误）**：Claude API 对单次请求的并发工具调用数量有限制，当智能体试图同时调用过多工具时触发
- **速率限制（429 错误）**：API 调用配额耗尽，需要等待重置
- **认证失效**：OAuth token 过期或被吊销
- **智能体自身逻辑失败**：任务未标记为完成（`in_progress` 而非 `completed`）
- **部分进展后崩溃**：已写入若干文件、提交了几个 commit，但整体任务未完成

在这些场景中，简单地"抛出异常"是不够的。系统需要回答：**这次失败是可重试的还是需要人工介入？如果可以重试，应该从哪里继续？已有的进展是否值得保留？**

Auto-Claude 用两个模块来回答这些问题：`agents/base.py` 定义了错误分类常量，`agents/session.py` 实现了完整的会话执行与恢复逻辑。

---

## 9.2 错误分类：在响应之前先理解

`apps/backend/agents/base.py` 是一个短小精悍的模块，但其中的常量定义揭示了系统设计者对错误的深思熟虑：

```python
# apps/backend/agents/base.py（简化）

# 子任务级别的重试限制
MAX_SUBTASK_RETRIES = 5      # 超过此次数标记为"卡死"

# 工具并发错误的重试配置（指数退避）
MAX_CONCURRENCY_RETRIES = 5
INITIAL_RETRY_DELAY_SECONDS = 2   # 依次：2s, 4s, 8s, 16s, 32s
MAX_RETRY_DELAY_SECONDS = 32      # 退避上限

# 暂停/恢复机制的文件标志
RATE_LIMIT_PAUSE_FILE = "RATE_LIMIT_PAUSE"  # 后端创建：已触发速率限制
AUTH_FAILURE_PAUSE_FILE = "AUTH_PAUSE"       # 后端创建：认证失败
RESUME_FILE = "RESUME"                       # 前端创建：用户已处理，请继续

# 等待超时
MAX_RATE_LIMIT_WAIT_SECONDS = 7200           # 最多等待 2 小时
RATE_LIMIT_CHECK_INTERVAL_SECONDS = 30       # 每 30 秒检查一次 RESUME 文件
```

`★ Insight ─────────────────────────────────────`
这里使用**文件系统作为进程间通信（IPC）机制**。后端（Python 进程）和前端（Electron 进程）通过在磁盘创建/删除标志文件来协调状态，无需共享内存或消息队列。这种方案牺牲了效率，但获得了极强的可靠性：即使任一进程崩溃重启，状态仍然持久化在磁盘上。
`─────────────────────────────────────────────────`

错误的正确分类是后续所有恢复决策的基础。`run_agent_session()` 在捕获异常后做的第一件事，就是分类：

```python
# apps/backend/agents/session.py（简化）

except Exception as e:
    is_concurrency = is_tool_concurrency_error(e)   # 400 工具并发
    is_rate_limit = is_rate_limit_error(e)           # 429 配额耗尽
    is_auth = is_authentication_error(e)             # 401 认证失败

    if is_concurrency:
        error_type = "tool_concurrency"
    elif is_rate_limit:
        error_type = "rate_limit"
    elif is_auth:
        error_type = "authentication"
    else:
        error_type = "other"
```

分类结果被封装进 `error_info` 字典返回，不同类型在 `post_session_processing()` 中触发不同的恢复路径。

---

## 9.3 敏感信息脱敏：安全的错误消息

在错误处理中有一个常常被忽视的安全细节：错误消息可能包含 API 密钥。当 Anthropic SDK 抛出认证错误时，异常消息可能包含完整的 `sk-ant-api03-...` 密钥字符串。

`sanitize_error_message()` 在错误消息被打印或传输给前端之前对其脱敏：

```python
# apps/backend/agents/base.py（简化）

def sanitize_error_message(error_message: str, max_length: int = 500) -> str:
    """在输出前对错误消息中的敏感信息进行脱敏。"""
    # 模式 1: sk-... 格式的 API 密钥（OpenAI/Anthropic 格式）
    sanitized = re.sub(
        r"\bsk-[a-zA-Z0-9._\-]{20,}\b",
        "[REDACTED_API_KEY]",
        error_message
    )
    # 模式 2: Bearer token
    sanitized = re.sub(
        r"\bBearer\s+[a-zA-Z0-9._\-]{20,}\b",
        "Bearer [REDACTED_TOKEN]",
        sanitized
    )
    # 模式 3: token=/secret= 赋值形式
    sanitized = re.sub(
        r"(token[=:]\s*)[a-zA-Z0-9._\-]{20,}\b",
        r"\1[REDACTED_TOKEN]",
        sanitized,
        flags=re.IGNORECASE,
    )
    return sanitized[:max_length]  # 同时截断超长消息
```

注意代码注释里这行说明：

```python
# 脱敏必须在打印到 stdout 之前发生，因为 stdout 会被前端捕获
sanitized_error = sanitize_error_message(str(e))
```

前端会订阅后端进程的标准输出流，如果 API 密钥出现在输出中，就会被写入日志文件，构成安全隐患。

---

## 9.4 会话执行：`run_agent_session()`

`run_agent_session()` 是智能体执行的核心循环。它的结构值得仔细研究：

```
┌─────────────────────────────────────────┐
│         run_agent_session()             │
│                                         │
│  1. client.query(message)               │  发送提示词
│              ↓                          │
│  2. async for msg in safe_receive():    │  流式接收
│       AssistantMessage                  │
│         ├── TextBlock → 打印 + 记录     │
│         └── ToolUseBlock → 显示工具调用 │
│       UserMessage                       │
│         └── ToolResultBlock             │
│             ├── is_error=True → [BLOCKED]│
│             └── is_error=False → [Done] │
│              ↓                          │
│  3. is_build_complete()                 │  检查是否完成
│     → "complete" / "continue"           │
│                                         │
│  4. except Exception:                   │  错误分类
│     → 分类 + 脱敏 + 返回 error_info     │
└─────────────────────────────────────────┘
```

函数返回一个三元组 `(status, response_text, error_info)`：

| `status` | 含义 |
|---------|------|
| `"complete"` | 构建完成，所有子任务已完成 |
| `"continue"` | 会话正常结束，但构建未完成（需要继续下一轮） |
| `"error"` | 发生了异常 |

这种设计将"会话的技术结果"（成功/失败）和"任务的业务结果"（完成/未完成）分离开来，使调用方能做出更细粒度的决策。

---

## 9.5 会话后处理：`post_session_processing()`

这是整个错误恢复逻辑的核心。每次智能体会话结束后，无论成功还是失败，都会调用此函数。它的名字揭示了一个重要设计原则：**将事后分析放在 Python 代码中，而非依赖智能体自我报告**。

函数中的注释直接点明了这一意图：

```python
# 这里在 Python 中运行（100% 可靠），而非依赖智能体合规性。
```

智能体是不可靠的执行者——它可能忘记更新状态文件，或者因为上下文窗口耗尽而中断。后处理函数通过读取实际文件来验证状态，而不是相信智能体的报告。

### 9.5.1 三分支决策树

函数根据子任务状态走向三条分支：

```
子任务状态
    │
    ├── "completed" ──→ 成功路径
    │                   ├── 更新进度计数
    │                   ├── 记录恢复点（好的 commit）
    │                   ├── 提取会话洞察（LLM 分析）
    │                   ├── 保存到记忆系统（Graphiti / 文件）
    │                   └── 通知 Linear（如果启用）
    │
    ├── "in_progress" → 失败路径（有部分进展）
    │                   ├── 记录失败尝试
    │                   ├── 检测是否是并发错误
    │                   │   ├── 是 → 重置子任务到 pending（供重试）
    │                   │   └── 否 → 调用 check_and_recover() 自动分析
    │                   └── 保存失败会话记忆（供下次参考）
    │
    └── 其他状态 ──→ 完全失败路径
                    ├── 记录失败
                    └── check_and_recover() 决定恢复动作
```

### 9.5.2 四种恢复动作

`check_and_recover()` 是来自 `recovery` 模块的函数，它分析历史失败记录，返回四种可能的恢复动作之一：

```python
# 恢复动作执行逻辑（apps/backend/agents/session.py，简化）

def _execute_recovery_action(recovery_action, recovery_manager, ...):
    if recovery_action.action == "rollback":
        # 代码回滚到上一个"好的"commit
        recovery_manager.rollback_to_commit(recovery_action.target)

    elif recovery_action.action == "retry":
        # 重置子任务状态为 pending，用不同方法重试
        reset_subtask(spec_dir, project_dir, subtask_id)

    elif recovery_action.action in ("skip", "escalate"):
        # 标记为"卡死"，需要人工介入
        recovery_manager.mark_subtask_stuck(subtask_id, recovery_action.reason)
```

| 恢复动作 | 触发条件 | 效果 |
|---------|---------|------|
| `rollback` | 检测到代码状态异常/破坏 | `git reset --hard <good_commit>` |
| `retry` | 失败次数 < `MAX_SUBTASK_RETRIES` | 重置状态，下轮以新方法尝试 |
| `skip` | 失败次数过多，可跳过 | 标记为 stuck，继续其他子任务 |
| `escalate` | 失败次数过多，不可跳过 | 标记为 stuck，等待人工修复 |

---

## 9.6 并发错误的特殊处理

工具并发错误（400 错误）的处理逻辑是独立的，因为它的原因完全不同于业务逻辑失败：

```python
# apps/backend/agents/session.py（简化）

is_concurrency_error = (error_info and error_info.get("type") == "tool_concurrency")

if is_concurrency_error:
    # 直接重置为 pending，不走 check_and_recover() 恢复逻辑
    reset_subtask(spec_dir, project_dir, subtask_id)

    # 同时重置实现计划中的状态
    for phase in plan.get("phases", []):
        for subtask in phase.get("subtasks", []):
            if subtask.get("id") == subtask_id:
                subtask["status"] = "pending"
                subtask["started_at"] = None
                subtask["completed_at"] = None

    # 原子写入（防止文件损坏）
    write_json_atomic(plan_path, plan, indent=2)
```

`★ Insight ─────────────────────────────────────`
并发错误是一种**外部资源约束**，而非智能体的逻辑错误。将它从通用错误恢复流程中独立出来，避免了错误地增加"失败次数计数"（这会影响 `check_and_recover()` 的决策）。系统正确地区分了"能力边界"和"能力缺陷"。
`─────────────────────────────────────────────────`

---

## 9.7 记忆驱动的失败学习

`post_session_processing()` 一个精妙之处在于：**即使会话失败，也提取洞察并保存到记忆系统**。

```python
# 成功路径和失败路径都调用 extract_session_insights()
extracted_insights = await extract_session_insights(
    spec_dir=spec_dir,
    project_dir=project_dir,
    subtask_id=subtask_id,
    session_num=session_num,
    commit_before=commit_before,
    commit_after=commit_after,
    success=False,        # 失败的会话
    recovery_manager=recovery_manager,
)

# 然后保存到记忆系统
await save_session_memory(
    ...
    success=False,
    discoveries=extracted_insights,  # 失败原因也是"发现"
)
```

这体现了一个重要的 AI 系统设计哲学：失败的尝试往往包含比成功更多的信息。知道"这条路走不通"本身就是有价值的知识，可以指导下一次尝试避免重复同样的错误。

---

## 9.8 状态机视角

将整个流程从状态机角度来理解，更清晰：

```
         ┌──────────────────────────────────────────────────┐
         │                 子任务执行状态机                    │
         └──────────────────────────────────────────────────┘

  pending ──→ [run_agent_session] ──→ completed ──→ 整体完成
      ↑                │
      │                ↓
      │         in_progress / error
      │                │
      │          [post_session_processing]
      │                │
      │    ┌──────────────────────────┐
      │    │ error_type?              │
      │    ├── concurrency → reset ───┤
      │    │                         │
      │    ├── others → check_and    │
      │    │   recover()             │
      │    │   ├── rollback ─────────┼──→ pending（代码已回滚）
      └────┤   ├── retry ────────────┘
           │   ├── skip ─────────────────→ stuck（跳过）
           │   └── escalate ─────────────→ stuck（人工介入）
           └─────────────────────────────────────────────────
```

---

## 9.9 工程实践：鲁棒性的代价

Auto-Claude 的错误处理体系给工程师提供了几个值得深思的权衡：

**原子写入 vs 性能**：`write_json_atomic()` 使用临时文件 + 重命名来确保 JSON 文件写入的原子性，代价是额外的 I/O。在高频写入场景（如记录每个工具调用的结果）中，这个代价是可观的。Auto-Claude 只对关键状态文件（实现计划、会话记忆）使用原子写入，而对日志类文件使用普通写入。

**会话粒度 vs 可恢复性**：系统将一个子任务分成多个会话（session），而非一次性运行到底。每个会话结束后做检查点（checkpoint）。这增加了开销，但使得恢复粒度更细——一次失败最多损失一个会话的进展，而非整个子任务。

**自动恢复 vs 人工审查**：`MAX_SUBTASK_RETRIES = 5` 是一个工程决策。太低会导致可解决的问题也被标记为"卡死"；太高会浪费资源在注定失败的任务上。实际值需要根据观测到的失败率进行校准。

---

## 本章要点回顾

| 概念 | 实现位置 | 关键点 |
|------|---------|-------|
| 错误分类 | `agents/session.py` | concurrency / rate_limit / auth / other |
| 敏感信息脱敏 | `agents/base.py: sanitize_error_message()` | 正则匹配 API key，在输出前处理 |
| 文件系统 IPC | `agents/base.py: *_PAUSE_FILE` | 进程间通信，崩溃安全 |
| 会话后处理 | `agents/session.py: post_session_processing()` | Python 验证，不依赖智能体自报告 |
| 四种恢复动作 | `_execute_recovery_action()` | rollback / retry / skip / escalate |
| 并发错误特殊处理 | `post_session_processing()` | 独立路径，不计入失败次数 |
| 失败记忆 | `save_session_memory(success=False)` | 失败知识也是有价值的 |

---

## Lab 9.1：构建智能重试器

本 Lab 实现一个具有错误分类和指数退避的智能重试装饰器，模拟 Auto-Claude 的会话重试逻辑。

### Part A：错误分类器（必做）

```python
# lab_09_retry.py
import re
import time
import asyncio
import random
from dataclasses import dataclass, field
from enum import Enum
from typing import Any, Callable


class ErrorType(Enum):
    CONCURRENCY = "concurrency"   # 可立即重试（有上限）
    RATE_LIMIT = "rate_limit"     # 等待后重试
    AUTH = "auth"                 # 不可重试，需人工处理
    TRANSIENT = "transient"       # 可重试
    PERMANENT = "permanent"       # 不可重试


@dataclass
class ErrorInfo:
    error_type: ErrorType
    original_message: str
    sanitized_message: str
    retryable: bool
    retry_delay: float = 0.0


def sanitize_error_message(message: str) -> str:
    """
    对错误消息进行脱敏处理。

    TODO: 实现以下脱敏规则（5-8 行）：
    1. 替换 sk-xxx 格式的 API 密钥为 [REDACTED_API_KEY]
    2. 替换 Bearer xxx 格式的 token
    3. 替换 token=xxx 或 token:xxx 格式的值
    4. 截断到 500 字符

    提示：使用 re.sub() 和正则表达式
    """
    # YOUR CODE HERE
    pass


def classify_error(error: Exception) -> ErrorType:
    """
    对异常进行分类。

    TODO: 根据以下规则分类（8-12 行）：
    - 消息包含 "400" 且 "concurrent" → CONCURRENCY
    - 消息包含 "429" 或 "rate limit" → RATE_LIMIT
    - 消息包含 "401" 或 "unauthorized" → AUTH
    - ConnectionError, TimeoutError → TRANSIENT
    - 其他 → PERMANENT

    提示：检查 str(error).lower() 中的关键词
    """
    # YOUR CODE HERE
    pass


def build_error_info(error: Exception, retry_delay: float = 0.0) -> ErrorInfo:
    """根据异常构建 ErrorInfo 对象。"""
    error_type = classify_error(error)
    original = str(error)
    sanitized = sanitize_error_message(original)

    retryable = error_type in (
        ErrorType.CONCURRENCY,
        ErrorType.RATE_LIMIT,
        ErrorType.TRANSIENT,
    )

    return ErrorInfo(
        error_type=error_type,
        original_message=original,
        sanitized_message=sanitized,
        retryable=retryable,
        retry_delay=retry_delay,
    )


# === 验证 Part A ===
def test_sanitize():
    msg = "Request failed with key sk-ant-api03-abcdef1234567890xyz Bearer eyJ0eXAiOiJKV1Q"
    result = sanitize_error_message(msg)
    assert "[REDACTED_API_KEY]" in result, "API key should be redacted"
    assert "[REDACTED_TOKEN]" in result, "Bearer token should be redacted"
    assert "sk-ant-api03" not in result, "Raw API key should not appear"
    print("[PASS] sanitize_error_message")

def test_classify():
    cases = [
        (Exception("400 concurrent tool calls exceeded"), ErrorType.CONCURRENCY),
        (Exception("429 rate limit exceeded"), ErrorType.RATE_LIMIT),
        (Exception("401 unauthorized access"), ErrorType.AUTH),
        (ConnectionError("connection refused"), ErrorType.TRANSIENT),
        (ValueError("unknown error"), ErrorType.PERMANENT),
    ]
    for error, expected in cases:
        result = classify_error(error)
        assert result == expected, f"Expected {expected}, got {result} for {error}"
    print("[PASS] classify_error")
```

### Part B：指数退避重试器（必做）

```python
# 接续 lab_09_retry.py

@dataclass
class RetryConfig:
    max_retries: int = 5
    initial_delay: float = 2.0
    max_delay: float = 32.0
    backoff_factor: float = 2.0


class RetrySession:
    """
    带指数退避的会话重试管理器。

    记录每次尝试的结果，提供退避延迟计算。
    """

    def __init__(self, task_id: str, config: RetryConfig = None):
        self.task_id = task_id
        self.config = config or RetryConfig()
        self.attempts: list[dict] = []
        self.consecutive_failures = 0

    def record_attempt(self, success: bool, error_info: ErrorInfo | None = None):
        """记录一次尝试结果。"""
        self.attempts.append({
            "attempt": len(self.attempts) + 1,
            "success": success,
            "error_type": error_info.error_type.value if error_info else None,
        })
        if success:
            self.consecutive_failures = 0
        else:
            self.consecutive_failures += 1

    def get_retry_delay(self) -> float:
        """
        计算下次重试的等待时间（指数退避）。

        TODO: 实现指数退避公式（3-5 行）：
        delay = initial_delay * (backoff_factor ** (consecutive_failures - 1))
        限制在 max_delay 内
        加入 ±20% 的随机抖动（jitter），防止惊群效应

        提示：使用 min(delay, max_delay) 和 random.uniform(0.8, 1.2)
        """
        # YOUR CODE HERE
        pass

    def should_retry(self, error_info: ErrorInfo) -> bool:
        """
        判断是否应该重试。

        TODO: 实现重试判断逻辑（5-8 行）：
        - 如果 error_type 为 AUTH → 不重试
        - 如果 error_type 为 PERMANENT → 不重试
        - 如果 consecutive_failures >= max_retries → 不重试
        - 如果 error_info.retryable → 重试
        """
        # YOUR CODE HERE
        pass

    @property
    def attempt_count(self) -> int:
        return len(self.attempts)


async def run_with_retry(
    task_fn: Callable,
    task_id: str,
    config: RetryConfig = None,
    verbose: bool = True,
) -> tuple[bool, Any, RetrySession]:
    """
    带重试的任务执行器。

    Args:
        task_fn: 异步任务函数，失败时抛出异常
        task_id: 任务标识符
        config: 重试配置
        verbose: 是否打印重试信息

    Returns:
        (success, result, session) 三元组
    """
    session = RetrySession(task_id, config)
    result = None

    while True:
        try:
            result = await task_fn()
            session.record_attempt(success=True)
            if verbose:
                print(f"  [OK] Attempt {session.attempt_count} succeeded")
            return True, result, session

        except Exception as e:
            error_info = build_error_info(e)
            session.record_attempt(success=False, error_info=error_info)

            if verbose:
                print(f"  [FAIL] Attempt {session.attempt_count}: "
                      f"{error_info.error_type.value} - {error_info.sanitized_message[:80]}")

            if not session.should_retry(error_info):
                if verbose:
                    reason = ("max retries exceeded"
                              if session.consecutive_failures >= session.config.max_retries
                              else "non-retryable error")
                    print(f"  [STOP] Not retrying: {reason}")
                return False, None, session

            delay = session.get_retry_delay()
            if verbose:
                print(f"  [WAIT] Retrying in {delay:.1f}s...")
            await asyncio.sleep(delay)


# === 验证 Part B ===
async def test_retry_success():
    """测试：第 3 次尝试成功"""
    call_count = 0

    async def flaky_task():
        nonlocal call_count
        call_count += 1
        if call_count < 3:
            raise ConnectionError("transient network error")
        return "SUCCESS"

    config = RetryConfig(max_retries=5, initial_delay=0.01, max_delay=0.1)
    success, result, session = await run_with_retry(flaky_task, "test-1", config, verbose=False)

    assert success, "Should succeed after retries"
    assert result == "SUCCESS"
    assert session.attempt_count == 3
    print("[PASS] retry_success after 3 attempts")


async def test_retry_exhausted():
    """测试：重试次数耗尽"""
    async def always_fail():
        raise ConnectionError("persistent error")

    config = RetryConfig(max_retries=3, initial_delay=0.01, max_delay=0.05)
    success, _, session = await run_with_retry(always_fail, "test-2", config, verbose=False)

    assert not success, "Should fail after max retries"
    assert session.attempt_count == 4  # 1 initial + 3 retries
    print("[PASS] retry_exhausted after max retries")


async def test_auth_no_retry():
    """测试：AUTH 错误不重试"""
    async def auth_fail():
        raise Exception("401 unauthorized - token invalid")

    config = RetryConfig(max_retries=5, initial_delay=0.01)
    success, _, session = await run_with_retry(auth_fail, "test-3", config, verbose=False)

    assert not success
    assert session.attempt_count == 1, "Should not retry auth errors"
    print("[PASS] auth_error_no_retry")
```

### Part C：恢复动作决策器（进阶）

```python
# 接续 lab_09_retry.py

class RecoveryAction(Enum):
    ROLLBACK = "rollback"
    RETRY = "retry"
    SKIP = "skip"
    ESCALATE = "escalate"


@dataclass
class RecoveryDecision:
    action: RecoveryAction
    reason: str
    target: str | None = None   # rollback 时为目标 commit hash


def decide_recovery(
    session: RetrySession,
    has_partial_commits: bool = False,
    is_skippable: bool = True,
    max_retries: int = 5,
) -> RecoveryDecision:
    """
    根据会话历史决定恢复动作。

    TODO: 实现恢复决策逻辑（10-15 行）：

    决策规则（按优先级）：
    1. 如果有部分 commit 且连续失败 >= 2 次
       → ROLLBACK（代码状态可能已损坏）
    2. 如果连续失败 < max_retries
       → RETRY（还有重试机会）
    3. 如果是可跳过的任务（is_skippable）
       → SKIP（跳过继续其他任务）
    4. 否则
       → ESCALATE（需要人工处理）

    提示：返回 RecoveryDecision 对象，
    reason 字段应包含有助于诊断的信息
    """
    # YOUR CODE HERE
    pass


# 运行所有测试
async def main():
    print("=== Part A: 错误分类测试 ===")
    test_sanitize()
    test_classify()

    print("\n=== Part B: 重试逻辑测试 ===")
    await test_retry_success()
    await test_retry_exhausted()
    await test_auth_no_retry()

    print("\n=== Part C: 恢复决策测试 ===")

    # 测试 1: 多次失败 → RETRY
    session = RetrySession("task-1")
    for _ in range(2):
        session.record_attempt(False, build_error_info(ConnectionError()))
    decision = decide_recovery(session, has_partial_commits=False)
    assert decision.action == RecoveryAction.RETRY, f"Expected RETRY, got {decision.action}"
    print("[PASS] decide_recovery: RETRY after 2 failures")

    # 测试 2: 有部分 commit 且失败 2 次 → ROLLBACK
    session = RetrySession("task-2")
    for _ in range(2):
        session.record_attempt(False, build_error_info(ConnectionError()))
    decision = decide_recovery(session, has_partial_commits=True)
    assert decision.action == RecoveryAction.ROLLBACK
    print("[PASS] decide_recovery: ROLLBACK with partial commits")

    # 测试 3: 超过最大重试 → SKIP
    session = RetrySession("task-3")
    for _ in range(6):
        session.record_attempt(False, build_error_info(ConnectionError()))
    decision = decide_recovery(session, is_skippable=True, max_retries=5)
    assert decision.action == RecoveryAction.SKIP
    print("[PASS] decide_recovery: SKIP after max retries")

    print("\n所有测试通过！")


if __name__ == "__main__":
    asyncio.run(main())
```

### 验证标准

- `sanitize_error_message()` 正确脱敏 API key 和 token
- `classify_error()` 正确分类 5 种错误类型
- `get_retry_delay()` 实现指数退避，每次失败延迟翻倍
- `should_retry()` 在 AUTH/PERMANENT 错误时直接放弃
- `decide_recovery()` 按优先级返回正确的恢复动作

### 进阶挑战

1. 实现一个 `RateLimitWaiter`：在检测到速率限制后，读取响应头中的 `Retry-After` 字段，等待正确的时间
2. 添加"熔断器"（Circuit Breaker）模式：当短时间内失败率超过阈值时，暂停所有请求一段时间，然后用少量探测请求检查服务是否恢复

---

*下一章，我们将深入规格创建流水线——这是 Auto-Claude 最精妙的部分之一：如何用一个多阶段的 AI 管道，将用户的一句模糊描述转化为一份详细的实现规格。*
