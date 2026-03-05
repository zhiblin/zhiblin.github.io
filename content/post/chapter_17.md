---
title: 第十七章：测试策略：pytest 与智能体系统的 Mock 艺术
date: 2026-01-17 00:00:00
categories: Agents
tags: [AI, Agent, Python, Claude, SDK]
---

# 第十七章　测试策略：pytest 与智能体系统的 Mock 艺术

> "测试智能体系统的最大挑战不是写断言，而是隔离——如何让依赖真实 API、真实 Git 操作、真实 AI 输出的代码，在没有网络的 CI 环境里可靠地运行。"

## 17.1　问题：智能体测试的三重困境

测试传统 Web 应用，你可以 mock 数据库、mock HTTP 请求。但 Auto-Claude 的测试面临三个独特挑战：

**困境一：外部 AI SDK 难以 mock**
`claude_agent_sdk` 和 `claude_code_sdk` 不是简单的 HTTP 客户端，它们包含复杂的内部状态机、事件循环和流式响应。直接 mock 容易遗漏细节，导致测试通过但真实场景失败。

**困境二：模块加载顺序敏感**
Python 在 `import` 时就会执行模块级代码。如果某个模块在 `import` 时调用了 `claude_agent_sdk.some_function()`，测试就必须在 `import` 发生之前就准备好 mock，否则真实 SDK 代码（可能需要认证）会在测试启动时运行。

**困境三：测试间模块状态泄漏**
Python 的 `sys.modules` 是全局的。一个测试文件安装的 mock，如果没有正确清理，会"污染"后续测试的导入环境。

Auto-Claude 的 `conftest.py` 对这三个问题都有系统性的解决方案，值得深入学习。

---

## 17.2　Pre-Import Mock：先发制人

`conftest.py` 的第一段代码（在任何 `import` 之前）就处理了困境二：

```python
# tests/conftest.py（简化）

def _create_sdk_mock():
    """创建 SDK 模块的全面 mock。"""
    mock = MagicMock()
    mock.ClaudeAgentOptions = MagicMock
    mock.ClaudeSDKClient = MagicMock
    mock.HookMatcher = MagicMock
    return mock

# 在任何其他 import 之前注入 mock
if 'claude_agent_sdk' not in sys.modules:
    sys.modules['claude_agent_sdk'] = _create_sdk_mock()
    sys.modules['claude_agent_sdk.types'] = MagicMock()

if 'claude_code_sdk' not in sys.modules:
    sys.modules['claude_code_sdk'] = _create_sdk_mock()
    sys.modules['claude_code_sdk.types'] = MagicMock()

# 连 dotenv 也要 mock：防止 cli.utils 在 import 时调用 sys.exit()
if 'dotenv' not in sys.modules:
    sys.modules['dotenv'] = MagicMock()
    sys.modules['dotenv'].load_dotenv = MagicMock()
```

这个模式叫做 **Pre-Import Injection**（预导入注入）：在目标模块被导入之前，把 mock 塞进 `sys.modules`。当目标模块执行 `import claude_agent_sdk` 时，Python 直接从 `sys.modules` 返回已注册的 mock，不会尝试加载真实模块。

`★ Insight ─────────────────────────────────────`
注意检查 `if 'claude_agent_sdk' not in sys.modules`——不是无条件覆盖。如果 SDK 已经安装（开发环境），这里就跳过，让真实 SDK 被使用。这让同一套测试在"有 SDK"和"无 SDK"两种环境都能工作。
`─────────────────────────────────────────────────`

---

## 17.3　模块级 Mock 清理：解决状态泄漏

困境三（测试间模块状态泄漏）的解决方案是一个精心设计的清理系统：

```python
# 可能被某些测试 mock 的模块列表
_POTENTIALLY_MOCKED_MODULES = [
    'claude_code_sdk', 'claude_code_sdk.types',
    'claude_agent_sdk', 'claude_agent_sdk.types',
    'dotenv', 'ui', 'progress', 'task_logger',
    # ...更多模块
]

# 在 import 时保存模块的原始状态
_original_module_state = {}
for _name in _POTENTIALLY_MOCKED_MODULES:
    if _name in sys.modules:
        _original_module_state[_name] = sys.modules[_name]

def _cleanup_mocked_modules():
    """移除 sys.modules 中所有 MagicMock 模块。"""
    for name in _POTENTIALLY_MOCKED_MODULES:
        if name in sys.modules:
            module = sys.modules[name]
            if isinstance(module, MagicMock):
                if name in _original_module_state:
                    # 恢复原始模块（如果原来就有）
                    sys.modules[name] = _original_module_state[name]
                else:
                    # 删除 mock（原来没有这个模块）
                    del sys.modules[name]
```

### 每个测试文件声明自己需要的 mock

解决"哪些 mock 该保留"的方法是让每个测试文件声明依赖：

```python
# 不同测试模块需要不同的 mock 集
module_mocks = {
    'test_qa_loop': {'claude_code_sdk', 'claude_code_sdk.types',
                     'claude_agent_sdk', 'claude_agent_sdk.types'},

    'test_spec_pipeline': {'claude_code_sdk', 'claude_code_sdk.types',
                           'init', 'client', 'review', 'task_logger', 'ui', 'validate_spec'},

    'test_qa_fixer': {'claude_agent_sdk', 'ui', 'progress', 'task_logger',
                      'agents.memory_manager', 'agents.base',
                      'core.error_utils', 'security.tool_input_validator'},
}

def pytest_runtest_setup(item):
    """每个测试运行前清理：只保留当前测试需要的 mock。"""
    module_name = item.module.__name__
    preserved_mocks = module_mocks.get(module_name, set())

    for name in _POTENTIALLY_MOCKED_MODULES:
        if name in preserved_mocks:
            continue  # 当前测试需要这个 mock，保留
        if name in sys.modules and isinstance(sys.modules[name], MagicMock):
            # 不需要的 mock 清理掉
            del sys.modules[name]
```

这种设计的优雅之处：每个测试文件"声明"自己的依赖，`conftest.py` 在每次测试前自动确保环境符合声明。测试间互不干扰，而且这种声明形式就是一份活的依赖文档。

---

## 17.4　Fixture 体系：临时目录与 Git 仓库

Auto-Claude 的功能大量涉及文件系统操作和 Git 操作。`conftest.py` 提供了一系列标准 fixture：

```python
@pytest.fixture
def temp_dir() -> Generator[Path, None, None]:
    """提供隔离的临时目录，测试结束自动清理。"""
    with tempfile.TemporaryDirectory() as tmp:
        yield Path(tmp)

@pytest.fixture
def git_repo(temp_dir: Path) -> Generator[Path, None, None]:
    """创建一个初始化好的 Git 仓库（有初始 commit）。"""
    subprocess.run(
        ['git', 'init', '--initial-branch=main', str(temp_dir)],
        check=True, capture_output=True
    )
    subprocess.run(
        ['git', '-C', str(temp_dir), 'config', 'user.email', 'test@test.com'],
        check=True, capture_output=True
    )
    subprocess.run(
        ['git', '-C', str(temp_dir), 'config', 'user.name', 'Test User'],
        check=True, capture_output=True
    )

    # 创建初始 commit（很多 Git 操作需要至少一个 commit）
    readme = temp_dir / "README.md"
    readme.write_text("# Test Project\n")
    subprocess.run(['git', '-C', str(temp_dir), 'add', '.'], check=True)
    subprocess.run(
        ['git', '-C', str(temp_dir), 'commit', '-m', 'Initial commit'],
        check=True, capture_output=True
    )
    yield temp_dir
```

**为什么需要 `--initial-branch=main`？**
不同版本的 Git 有不同的默认分支名（`master` vs `main`）。显式指定确保在所有 CI 环境中测试行为一致。

**为什么要设置 `user.email` 和 `user.name`？**
`git commit` 需要用户身份。CI 机器上通常没有全局 Git 配置，不设置会导致 commit 失败。

---

## 17.5　测试分层架构

Auto-Claude 的测试按照覆盖范围分为三层：

```
测试金字塔
               ┌─────────┐
               │  E2E 测试  │ ← Playwright (前端) / Electron MCP (AI 自验证)
               └─────────┘
          ┌───────────────────┐
          │    集成测试         │ ← pytest + 真实文件系统 + mock AI
          └───────────────────┘
    ┌─────────────────────────────────┐
    │           单元测试               │ ← pytest + 全部 mock
    └─────────────────────────────────┘
```

### 单元测试：纯逻辑隔离

以安全验证器为例（第7章讲过）：

```python
# tests/test_security.py

def test_rm_validator_blocks_recursive():
    """测试 rm -rf 被正确拦截。"""
    from security.validator import validate_command
    result = validate_command('rm', ['-rf', '/home/user/project'])
    assert result.allowed is False
    assert 'recursive' in result.reason.lower()

def test_rm_validator_allows_single_file():
    """测试删除单文件被允许。"""
    from security.validator import validate_command
    result = validate_command('rm', ['/tmp/test.txt'])
    assert result.allowed is True
```

这类测试不需要 mock：验证器是纯函数，输入→输出，没有外部依赖。

### 集成测试：真实文件系统 + mock AI

```python
# tests/test_worktree.py（概念化）

def test_worktree_create(git_repo):
    """测试在真实 Git 仓库里创建 worktree。"""
    from core.worktree import WorktreeManager

    manager = WorktreeManager(git_repo)
    worktree_path = manager.create_worktree('feature-branch')

    # 验证：worktree 目录真实存在
    assert worktree_path.exists()
    # 验证：分支被创建
    result = subprocess.run(
        ['git', '-C', str(git_repo), 'branch'],
        capture_output=True, text=True
    )
    assert 'feature-branch' in result.stdout

def test_project_analyzer_detects_python(temp_dir):
    """测试 Python 项目被正确识别。"""
    (temp_dir / 'requirements.txt').write_text('flask==2.0')
    (temp_dir / 'main.py').write_text('print("hello")')

    from project.analyzer import ProjectAnalyzer
    analyzer = ProjectAnalyzer(str(temp_dir))
    stack = analyzer.analyze()

    assert 'python' in stack.languages
```

### QA Loop 测试：精密的 Mock 状态机

QA Loop（第2章讲过）是整个系统中测试最复杂的部分，因为它是一个有状态的循环，依赖 AI 输出来决定下一步：

```python
# tests/test_qa_loop.py（简化）

@pytest.fixture
def mock_qa_agent():
    """模拟 QA Agent 的输出序列。"""
    # 第一次：发现问题；第二次：验证通过
    responses = [
        {"status": "FAIL", "issues": ["Missing error handling"]},
        {"status": "PASS", "issues": []},
    ]
    call_count = 0

    async def fake_run(*args, **kwargs):
        nonlocal call_count
        response = responses[min(call_count, len(responses) - 1)]
        call_count += 1
        return response

    with patch('qa.loop.run_qa_reviewer', side_effect=fake_run):
        yield

async def test_qa_loop_retries_on_failure(git_repo, mock_qa_agent):
    """QA Loop 在失败后重试，第二次通过。"""
    from qa.loop import run_qa_validation_loop

    result = await run_qa_validation_loop(
        project_path=str(git_repo),
        spec_id='test-001',
        max_retries=3
    )

    assert result.status == 'PASSED'
    assert result.iterations == 2
```

`★ Insight ─────────────────────────────────────`
QA Loop 的测试使用了"响应序列"模式（response sequence）：mock 对象在第 N 次调用时返回第 N 个预设响应。这让测试能精确模拟"先失败再通过"的场景，而不需要真实的 AI 响应。`unittest.mock` 的 `side_effect` 参数支持传入可迭代对象来实现这个模式。
`─────────────────────────────────────────────────`

---

## 17.6　前端测试：Vitest + React Testing Library

前端使用 Vitest（兼容 Vite 生态的 Jest 替代）测试 React 组件和 Zustand store：

```typescript
// apps/frontend/src/renderer/stores/__tests__/task-store.test.ts（概念化）

import { renderHook, act } from '@testing-library/react';
import { useTaskStore } from '../task-store';

describe('TaskStore', () => {
  beforeEach(() => {
    // 重置 store 状态（Zustand 支持）
    useTaskStore.setState({ tasks: [], selectedTaskId: null });
  });

  it('should enforce MAX_LOG_ENTRIES limit', () => {
    const { result } = renderHook(() => useTaskStore());

    act(() => {
      result.current.addTask({ id: 'task-1', status: 'running', logs: [] });
    });

    // 追加 5100 条日志
    const logs = Array.from({ length: 5100 }, (_, i) => `log-${i}`);
    act(() => {
      result.current.batchAppendLogs('task-1', logs);
    });

    const task = result.current.tasks.find(t => t.id === 'task-1');
    expect(task?.logs?.length).toBe(5000); // MAX_LOG_ENTRIES
  });
});
```

---

## 17.7　Lab 17.1：实现测试基础设施

**目标**：实现一套完整的测试基础设施，包含 Pre-Import Mock、Fixture 和序列 Mock。

### Part A：Pre-Import Mock

```python
# lab17/conftest.py
import sys
from unittest.mock import MagicMock

def _create_ai_sdk_mock():
    """创建 AI SDK 的基本 mock。"""
    mock = MagicMock()
    # TODO Part A: 给 mock 添加常用的 SDK 属性
    # mock.SomeClass = MagicMock
    # mock.some_constant = "value"
    return mock

# TODO Part A: 在 conftest 顶部注入 mock（在其他 import 之前）
# 如果 'fake_ai_sdk' 未在 sys.modules 中，就注入 mock
```

### Part B：Git 仓库 Fixture

```python
# lab17/conftest.py（续）
import subprocess
import tempfile
from pathlib import Path
from typing import Generator
import pytest

@pytest.fixture
def temp_dir() -> Generator[Path, None, None]:
    """临时目录 fixture（自动清理）。"""
    # TODO Part B: 使用 tempfile.TemporaryDirectory 创建临时目录
    # 记得 yield 而不是 return
    pass

@pytest.fixture
def git_repo(temp_dir: Path) -> Generator[Path, None, None]:
    """初始化好的 Git 仓库 fixture。"""
    # TODO Part B:
    # 1. git init --initial-branch=main {temp_dir}
    # 2. git config user.email / user.name
    # 3. 创建 README.md，git add，git commit -m "Initial commit"
    # 4. yield temp_dir
    pass
```

### Part C：响应序列 Mock

```python
# lab17/test_agent_mock.py
from unittest.mock import patch, AsyncMock, MagicMock
import pytest

def make_sequence_mock(responses: list):
    """
    创建一个按顺序返回响应的 mock。
    第 N 次调用返回 responses[N]，超出范围时返回最后一个。
    """
    # TODO Part C: 实现响应序列 mock
    # 提示：用 AsyncMock(side_effect=...) 或用列表的迭代器

    call_count = 0
    async def mock_func(*args, **kwargs):
        nonlocal call_count
        # TODO: 返回对应的响应，处理越界情况
        pass

    return AsyncMock(side_effect=mock_func)

@pytest.mark.asyncio
async def test_retry_on_failure():
    """测试：第一次失败，第二次成功。"""
    responses = [
        {"status": "FAIL", "issues": ["Bug found"]},
        {"status": "PASS", "issues": []},
    ]
    mock_agent = make_sequence_mock(responses)

    # 模拟一个会重试的函数
    async def run_with_retry(agent, max_retries=3):
        for attempt in range(max_retries):
            result = await agent()
            if result["status"] == "PASS":
                return result, attempt + 1
        return None, max_retries

    result, attempts = await run_with_retry(mock_agent)

    # TODO Part C: 验证
    # assert result["status"] == "PASS"
    # assert attempts == 2  # 第二次才成功
    # assert mock_agent.call_count == 2
```

**验证标准**：
- Part A：注入的 mock 在 `import fake_ai_sdk` 时正确返回 MagicMock 对象
- Part B：`git_repo` fixture 创建的仓库能运行 `git log` 并有至少一个 commit
- Part C：`make_sequence_mock` 按顺序返回响应，超出范围时返回最后一个

**进阶挑战**：
- 实现 `module_isolation` fixture：在某个测试函数运行期间临时替换指定模块，测试结束后自动恢复
- 给 Git 仓库 fixture 添加 `add_files` helper：接受字典（文件名→内容），批量创建文件并 commit
- 实现测试参数化：`@pytest.mark.parametrize` 覆盖 3 种不同的 rate limit 错误类型

---

## 本章要点回顾

| 概念 | 核心设计 | 源码位置 |
|------|---------|---------|
| Pre-Import Injection | `if not in sys.modules` 先注入 mock | `tests/conftest.py:36` |
| 模块状态快照 | `_original_module_state` 保存原始引用 | `tests/conftest.py:86` |
| `pytest_runtest_setup` | 每次测试前按声明清理 mock | `tests/conftest.py:110` |
| `module_mocks` 声明表 | 每个测试文件声明需要的 mock 集 | `tests/conftest.py:122` |
| `git_repo` fixture | 真实 Git 仓库，隔离的临时目录 | `tests/conftest.py` |
| 响应序列 Mock | `side_effect` 列表，模拟先失败后成功 | 多个测试文件 |
| `--initial-branch=main` | 跨 Git 版本的分支名一致性 | `tests/conftest.py` |
