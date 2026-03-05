---
title: 第十八章：可观测性：Sentry、日志与调试
date: 2026-01-18 00:00:00
categories: Agents
tags: [AI, Agent, Python, Claude, SDK]
---

# 第十八章　可观测性：Sentry、日志与调试

> "你无法调试你看不见的系统。可观测性不是事后的补丁，而是系统设计的一部分——从一开始就决定'当出错时，我能知道什么'。"

## 18.1　问题：为什么 AI 系统特别需要可观测性？

传统软件出错时，你可以重现问题：输入相同，输出相同。但 AI 智能体系统有一个独特挑战——**非确定性**。同一个提示词，不同时间调用可能得到不同输出。一个失败的任务，你很难"重现"它失败时 AI 做了什么决定。

Auto-Claude 运行着数十个并行智能体，每个都在独立的 worktree 里操作文件系统。当某个任务失败时，你需要知道：

- 智能体执行了哪些步骤？
- 是哪个具体的 AI 调用失败了？
- 错误发生时，进程的环境是什么？
- 用户路径里有没有泄漏（PII 问题）？

这些问题催生了 Auto-Claude 的可观测性架构：Sentry（生产错误追踪）+ 结构化日志 + 调试模式。

---

## 18.2　Sentry 集成：隐私优先的错误追踪

`apps/backend/core/sentry.py` 是后端 Sentry 集成的核心。它的设计原则：**先保护隐私，再收集数据**。

### 用户路径脱敏

错误报告中最常见的 PII 泄漏来源是文件路径——`/Users/johndoe/project/src/file.py` 包含用户名。

```python
def _mask_user_paths(text: str) -> str:
    """
    将用户路径中的用户名替换为 ***，跨平台支持。

    替换规则：
    - macOS: /Users/username/... → /Users/***/...
    - Windows: C:\\Users\\username\\... → C:\\Users\\***\\...
    - Linux: /home/username/... → /home/***/...
    - WSL: /mnt/c/Users/username/... → /mnt/c/Users/***/...
    """
    # macOS
    text = re.sub(r"/Users/[^/]+(?=/|$)", "/Users/***", text)

    # Windows（保留盘符）
    text = re.sub(
        r"[A-Za-z]:\\Users\\[^\\]+(?=\\|$)",
        lambda m: f"{m.group(0)[0]}:\\Users\\***",
        text,
    )

    # Linux
    text = re.sub(r"/home/[^/]+(?=/|$)", "/home/***", text)

    # WSL
    text = re.sub(
        r"/mnt/[a-z]/Users/[^/]+(?=/|$)",
        lambda m: f"{m.group(0)[:6]}/Users/***",
        text,
    )

    return text
```

`★ Insight ─────────────────────────────────────`
注意正则中的 `(?=/|$)` 是**零宽前视断言**：匹配 `/` 或字符串结尾之前的位置，但不消费这些字符。这确保路径组件的边界正确识别，不会误把 `/Users/johnsmith-backup` 替换成 `/Users/***/smith-backup`。
`─────────────────────────────────────────────────`

### 递归对象路径脱敏

错误报告可能包含复杂的嵌套对象（异常的 `extra` 上下文、breadcrumb 数据）。脱敏必须递归进行：

```python
def _mask_object_paths(obj: Any, _depth: int = 0) -> Any:
    """递归脱敏对象中的所有路径字符串。"""
    # 防止无限递归（循环引用或深度嵌套结构）
    if _depth > 50:
        return obj

    if obj is None:
        return obj
    if isinstance(obj, str):
        return _mask_user_paths(obj)
    if isinstance(obj, list):
        return [_mask_object_paths(item, _depth + 1) for item in obj]
    if isinstance(obj, dict):
        return {
            key: _mask_object_paths(value, _depth + 1)
            for key, value in obj.items()
        }

    # 其他类型（int, float, bool 等）直接返回
    return obj

def _before_send(event: dict, hint: dict) -> dict | None:
    """Sentry 发送前的最后处理：脱敏所有路径。"""
    # 递归脱敏整个 event 对象
    return _mask_object_paths(event)
```

`_before_send` 是 Sentry 的钩子函数，在数据发送到 Sentry 服务器之前调用。返回 `None` 会丢弃这条错误报告（可用于过滤不需要上报的错误类型）。

### 版本检测：从 package.json 读取

```python
def _get_version() -> str:
    """读取前端 package.json 中的版本号，保持前后端版本一致。"""
    try:
        backend_dir = Path(__file__).parent.parent
        frontend_dir = backend_dir.parent / "frontend"
        package_json = frontend_dir / "package.json"

        if package_json.exists():
            with open(package_json, encoding="utf-8") as f:
                data = json.load(f)
                return data.get("version", "0.0.0")
    except Exception as e:
        logger.debug(f"Version detection failed: {e}")

    return "0.0.0"
```

Auto-Claude 是单一版本发布的（前端+后端同步）。统一从前端 `package.json` 读取版本，确保 Sentry 里的 Python 错误报告和 TypeScript 错误报告显示相同的版本号，便于关联。

---

## 18.3　结构化日志：debugLog vs console.log

前端使用自定义的 `debugLog` 工具函数，而非直接使用 `console.log`：

```typescript
// apps/frontend/src/shared/utils/debug-logger.ts（概念化）

const isDebug = process.env.DEBUG === 'true';

export function debugLog(message: string, ...args: unknown[]): void {
  if (!isDebug) return;  // 生产环境静默
  console.log(`[DEBUG] ${message}`, ...args);
}

export function debugError(message: string, ...args: unknown[]): void {
  if (!isDebug) return;
  console.error(`[DEBUG ERROR] ${message}`, ...args);
}
```

这解决了 CLAUDE.md 中提到的一个生产问题：**Electron 打包后 `console.log` 输出不可见**。

在 Electron 的 production build 中，渲染进程和主进程的 `console.log` 输出既不显示在用户界面，也不写入日志文件——它们直接消失。只有通过调试工具（如 `npm run dev:debug`）才能看到。

解决方案：
1. 开发调试信息：使用 `debugLog()`，只在 `DEBUG=true` 时输出
2. 生产错误：使用 Sentry 报告，不依赖 console

### 后端日志配置

Python 后端使用标准库 `logging`，通过模块级 logger 标识来源：

```python
# 每个模块都声明自己的 logger
import logging
logger = logging.getLogger(__name__)

# 使用示例（来自 sentry.py）
logger.debug(f"Version detection failed: {e}")
logger.info("Sentry initialized with version %s", version)
logger.warning("SENTRY_DSN not set, Sentry disabled")
logger.error("Failed to initialize Sentry: %s", e)
```

日志级别策略：
- `DEBUG`：仅在开发环境看到，不出现在生产
- `INFO`：重要的生命周期事件（初始化完成、服务启动）
- `WARNING`：异常但可恢复的情况
- `ERROR`：需要关注的错误（发送到 Sentry）

---

## 18.4　调试模式：多层调试能力

Auto-Claude 提供了四个层次的调试能力：

```
调试层次（从轻到重）
    │
    ├── Level 1: DEBUG=true 环境变量
    │      └─ 启用 debugLog 输出
    │
    ├── Level 2: npm run dev:debug
    │      └─ Electron DevTools 可用
    │      └─ 详细的 IPC 日志
    │
    ├── Level 3: npm run dev:mcp
    │      └─ Electron MCP 服务器启动
    │      └─ AI 可以通过工具与 UI 交互
    │
    └── Level 4: ELECTRON_MCP_ENABLED=true
           └─ QA 智能体可以截图、点击 UI 元素
           └─ 端到端的 AI 自验证
```

### Electron MCP 自验证

最高层次的调试能力：让 QA 智能体直接操作 Electron UI 进行验证：

```bash
# 启动调试模式
npm run dev:debug

# 设置 MCP 标志
echo "ELECTRON_MCP_ENABLED=true" >> apps/backend/.env

# 运行 QA（QA 智能体可以控制 UI）
python run.py --spec 001 --qa
```

QA 智能体可用的工具：
- `take_screenshot`：截图当前 UI 状态
- `click_by_text`：点击包含特定文字的元素
- `fill_input`：填写输入框
- `get_page_structure`：获取 DOM 结构
- `send_keyboard_shortcut`：发送键盘快捷键

这是 Auto-Claude 最独特的测试能力：用 AI 来测试 AI 驱动的 UI，无需人工干预。

---

## 18.5　性能监控：采样率策略

Sentry 的 Performance 监控采用 10% 采样率：

```python
# sentry.py
PRODUCTION_TRACE_SAMPLE_RATE = 0.1  # 10% 的请求会被追踪

def init_sentry():
    sentry_sdk.init(
        dsn=os.environ.get('SENTRY_DSN'),
        traces_sample_rate=PRODUCTION_TRACE_SAMPLE_RATE,
        environment=_get_environment(),
        release=_get_version(),
        before_send=_before_send,
    )
```

为什么是 10% 而不是 100%？

- **成本**：Sentry 按事件数量计费，100% 追踪成本巨大
- **性能**：每次追踪都有轻微开销（上下文传播、采样决策）
- **代表性**：10% 在高流量下仍有足够样本量来分析性能趋势

对于错误事件（`before_send`），不受采样率限制——所有错误都会上报。

---

## 18.6　Lab 18.1：实现基础可观测性系统

**目标**：实现一个支持路径脱敏、结构化日志和错误追踪的可观测性基础设施。

### Part A：路径脱敏器

```python
# lab18/path_masker.py
import re

def mask_user_paths(text: str) -> str:
    """
    脱敏文件路径中的用户名。
    支持 macOS、Windows、Linux 和 WSL。
    """
    if not text:
        return text

    # TODO Part A: 实现路径脱敏
    # 提示：使用 re.sub()，注意边界匹配
    # macOS: /Users/username/... → /Users/***/...
    # Linux: /home/username/... → /home/***/...
    # Windows: C:\Users\username\... → C:\Users\***\...

    return text

def mask_object_paths(obj, _depth: int = 0):
    """递归脱敏对象中所有字符串字段的路径。"""
    # TODO Part A:
    # - depth > 50 时返回原始值（防止无限递归）
    # - str → 调用 mask_user_paths
    # - list → 递归处理每个元素
    # - dict → 递归处理每个值
    # - 其他类型 → 直接返回
    return obj

# 验证：
assert mask_user_paths("/Users/johndoe/project") == "/Users/***/project"
assert mask_user_paths("/home/alice/code") == "/home/***/code"
assert mask_user_paths("no path here") == "no path here"

data = {"error": "File not found: /Users/bob/work/main.py", "count": 42}
masked = mask_object_paths(data)
assert "bob" not in masked["error"]
assert masked["count"] == 42  # 非字符串类型不受影响
```

### Part B：条件日志系统

```python
# lab18/logger.py
import os
import logging
from typing import Any

# 读取调试标志
DEBUG = os.environ.get('DEBUG', '').lower() in ('true', '1', 'yes')

def setup_logging(level: str = 'WARNING') -> None:
    """配置日志系统。DEBUG 模式下使用 DEBUG 级别。"""
    # TODO Part B: 配置 logging.basicConfig
    # DEBUG 模式：使用 DEBUG 级别，显示文件名和行号
    # 生产模式：使用 level 参数指定的级别，简单格式
    pass

def get_logger(name: str) -> logging.Logger:
    """获取模块 logger。"""
    return logging.getLogger(name)

# 使用示例（给调用方）：
# logger = get_logger(__name__)
# logger.debug("只在 DEBUG=true 时显示")
# logger.warning("总是显示")
```

### Part C：错误报告收集器

```typescript
// lab18/error-collector.ts

interface ErrorReport {
  message: string;
  stack?: string;
  context: Record<string, unknown>;
  timestamp: number;
  masked: boolean;  // 是否已脱敏
}

export class ErrorCollector {
  private reports: ErrorReport[] = [];
  private maskFn: (text: string) => string;

  constructor(maskFn?: (text: string) => string) {
    // TODO Part C: 保存 maskFn，默认使用 identity 函数
    this.maskFn = maskFn ?? ((text) => text);
  }

  /**
   * 收集错误，自动脱敏路径
   */
  capture(error: Error, context: Record<string, unknown> = {}): void {
    // TODO Part C:
    // 1. 创建 ErrorReport
    // 2. 对 message 和 stack 调用 maskFn
    // 3. 对 context 的每个字符串值调用 maskFn
    // 4. 推入 reports 列表
  }

  /**
   * 获取最近 N 条报告
   */
  getRecentReports(n: number = 10): ErrorReport[] {
    // TODO Part C
    return [];
  }

  /**
   * 清空报告
   */
  clear(): void {
    this.reports = [];
  }
}

// 验证：
const masker = (text: string) => text.replace(/\/Users\/[^/]+/g, '/Users/***');
const collector = new ErrorCollector(masker);

const err = new Error('Failed to read /Users/alice/project/config.json');
collector.capture(err, { path: '/Users/alice/project' });

const reports = collector.getRecentReports(1);
console.log(reports[0].message); // → 'Failed to read /Users/***/project/config.json'
console.log(reports[0].context.path); // → '/Users/***/project'
console.log(reports[0].masked); // → true
```

**验证标准**：
- Part A：`mask_user_paths` 正确处理四种操作系统路径；`mask_object_paths` 递归深度超过 50 时不崩溃
- Part B：`DEBUG=true` 时显示调试日志；`DEBUG=false` 时只显示 WARNING 以上
- Part C：`capture` 对 message、stack、context 全部脱敏；`getRecentReports` 按插入顺序返回

**进阶挑战**：
- 给 `ErrorCollector` 添加采样率参数：`captureRate = 0.1`，随机丢弃 90% 的错误报告
- 实现 `before_send` 过滤：对于某类错误（如 `NetworkError`），直接丢弃不上报
- 给路径脱敏器添加项目路径白名单：允许显示 `/Users/***/projects/myapp` 中的 `myapp` 部分

---

## 本章要点回顾

| 概念 | 核心设计 | 源码位置 |
|------|---------|---------|
| `_mask_user_paths` | 正则替换四种 OS 路径中的用户名 | `core/sentry.py:65` |
| 零宽前视断言 `(?=/\|$)` | 正确识别路径组件边界 | `core/sentry.py:81` |
| `_mask_object_paths` | 深度限 50 的递归脱敏 | `core/sentry.py:103` |
| `_before_send` | Sentry 钩子，脱敏整个 event | `core/sentry.py:135` |
| `_get_version` | 从 frontend/package.json 读版本 | `core/sentry.py:40` |
| `debugLog` | `DEBUG=true` 才输出，生产静默 | `shared/utils/debug-logger.ts` |
| 10% 采样率 | 平衡成本与代表性 | `core/sentry.py:37` |
| Electron MCP 自验证 | AI 操控 UI 做端到端测试 | `CLAUDE.md` |
