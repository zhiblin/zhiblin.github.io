---
title: 第十五章：终端系统：PTY 与 Claude 集成
date: 2026-01-15 00:00:00
categories: Agents
tags: [AI, Agent, Python, Claude, SDK]
---

# 第十五章　终端系统：PTY 与 Claude 集成

> "一个终端不只是文本框——它是一个完整的进程通信协议，需要处理信号、编码、窗口大小、会话恢复和实时流。"

## 15.1　问题：为什么终端这么复杂？

当你在网页上看到一个"终端模拟器"，背后至少涉及四个层次：

1. **PTY（伪终端）**：操作系统级的双向字节流，模拟真实的串口终端
2. **Shell 进程**：bash/zsh/powershell，通过 PTY 与用户交互
3. **终端模拟器**：解析 ANSI 转义序列，在屏幕上渲染字符和颜色
4. **前端渲染**：把终端模拟器的输出展示在浏览器/Electron 窗口里

Auto-Claude 的终端系统在此之上还要处理两个额外问题：

- **会话持久化**：Electron 重启后，终端内容和 Claude 会话要能恢复
- **Claude SDK 集成**：在普通 shell 终端里直接启动 Claude Code，检测输出中的 rate limit 信号和 OAuth token 更新

这些需求共同催生了一个精心设计的多层架构。

---

## 15.2　PTY Daemon：独立进程，跨重启存活

Auto-Claude 最关键的架构决策之一：**PTY 进程不在 Electron 主进程里管理，而是作为独立的守护进程运行**。

```
Electron 主进程
    │  Unix Socket / Named Pipe
    │  (JSON 消息协议)
    ▼
PTY Daemon（独立 Node.js 进程）
    │
    ├── ManagedPty #1 ──► bash 进程
    ├── ManagedPty #2 ──► zsh 进程
    └── ManagedPty #3 ──► Claude Code 进程
```

为什么要独立进程？

**Electron 主进程死了，终端活着**。当用户更新应用、Electron 崩溃或重启时，PTY Daemon 继续运行。重新打开应用后，主进程通过 socket 重新连接到已存活的 Daemon，读取缓冲区内容，终端会话无缝恢复。

### 通信协议：换行分隔的 JSON

PTY Daemon 与主进程之间使用简单而可靠的协议：

```typescript
// apps/frontend/src/main/terminal/pty-daemon.ts（简化）

interface DaemonMessage {
  type:
    | 'create'    // 创建新 PTY
    | 'write'     // 向 PTY 写入数据（键盘输入）
    | 'resize'    // 调整窗口大小
    | 'kill'      // 终止 PTY
    | 'subscribe' // 订阅 PTY 输出
    | 'get-buffer'// 获取历史缓冲区
    | 'ping';     // 心跳检测
  id?: string;     // PTY 标识符
  data?: unknown;  // 消息体（create 时是 PtyConfig，write 时是字符串）
  requestId?: string; // 用于请求-响应配对
}

// 接收消息的方式：读取换行分隔的 JSON
socket.on('data', (chunk) => {
  buffer += chunk.toString('utf-8');
  const lines = buffer.split('\n');
  buffer = lines.pop() || ''; // 最后一段可能不完整，保留

  for (const line of lines) {
    if (!line.trim()) continue;
    const msg: DaemonMessage = JSON.parse(line);
    this.handleMessage(socket, msg);
  }
});
```

换行分隔的 JSON（NDJSON）是处理流式 JSON 的标准方案：不需要复杂的帧协议，单行解析失败不影响后续消息。

### Socket 路径：跨平台兼容

```typescript
const SOCKET_PATH = isWindows()
  ? `\\\\.\\pipe\\auto-claude-pty-${process.getuid?.() || 'default'}`
  : `/tmp/auto-claude-pty-${process.getuid?.() || 'default'}.sock`;
```

- **macOS/Linux**：Unix Domain Socket，比 TCP 快，只允许同用户连接
- **Windows**：Named Pipe，功能等价，但路径格式完全不同

Unix socket 在启动时会先清理残留文件（防止上次崩溃留下的 stale socket），并设置 `chmod 600` 权限，只有创建用户才能连接。

---

## 15.3　ManagedPty：环形缓冲区与多订阅者

每个 PTY 会话都封装在 `ManagedPty` 结构中：

```typescript
interface ManagedPty {
  id: string;
  process: pty.IPty;    // 实际的 PTY 进程引用
  config: PtyConfig;    // 创建配置（shell、工作目录等）
  buffer: string[];     // 环形输出缓冲区
  bufferSize: number;   // 当前缓冲区总字节数
  clients: Set<net.Socket>; // 订阅此 PTY 的所有连接
  createdAt: number;
  lastDataAt: number;   // 最后收到数据的时间（用于存活检测）
  isDead: boolean;
}

// 最大缓冲区：100KB
const MAX_BUFFER_SIZE = 100_000;
// 最大缓冲块数：1000 块
const RING_BUFFER_MAX_CHUNKS = 1000;
```

当 PTY 有输出时：

```typescript
ptyProcess.onData((data) => {
  managed.lastDataAt = Date.now();

  // 追加到环形缓冲区
  managed.buffer.push(data);
  managed.bufferSize += data.length;

  // 超出上限时，从头部移除旧数据（FIFO 淘汰）
  while (managed.bufferSize > MAX_BUFFER_SIZE && managed.buffer.length > 1) {
    const removed = managed.buffer.shift();
    if (removed) {
      managed.bufferSize -= removed.length;
    }
  }

  // 广播给所有订阅者
  for (const client of managed.clients) {
    this.send(client, { type: 'data', id: managed.id, data });
  }
});
```

`★ Insight ─────────────────────────────────────`
`clients: Set<net.Socket>` 实现了一个轻量级的发布/订阅系统。多个连接（如 Electron 重连、Debug 工具）可以同时订阅同一个 PTY 的输出，互不干扰。这是面向多消费者设计的关键——不用复制数据，直接广播引用。
`─────────────────────────────────────────────────`

### 创建 PTY：安全标识符生成

```typescript
private createPty(config: PtyConfig): string {
  if (this.isShuttingDown) {
    throw new Error('Cannot create PTY: daemon is shutting down');
  }

  // 时间戳 + 随机后缀：避免碰撞，人类可读
  const id = `pty-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;

  const ptyProcess = pty.spawn(config.shell, config.shellArgs, {
    name: 'xterm-256color',  // 声明终端类型，影响颜色和功能支持
    cols: config.cols,
    rows: config.rows,
    cwd: config.cwd,
    env: config.env,
  });

  // ...注册 onData/onExit 处理器
}
```

---

## 15.4　PtyManager：主进程侧的 Shell 管理

PTY Daemon 管理 PTY 进程本身，而 `PtyManager`（`apps/frontend/src/main/terminal/pty-manager.ts`）在 Electron 主进程侧负责更高层的逻辑：Shell 选择、Windows 兼容性和关机保护。

### 关机保护标志

```typescript
// 模块级变量，不使用类状态
let isShuttingDown = false;

export function setShuttingDown(value: boolean): void {
  isShuttingDown = value;
}
```

为什么需要这个？这是一个真实的生产 bug（GitHub issue #1469）：

**场景**：用户关闭应用，Electron 销毁 `BrowserWindow`。PTY 的 `onData` 回调在另一个线程异步触发，试图调用 `webContents.send()`——但 `webContents` 已经销毁，导致 `pty.node` 的 Native 线程函数（`ThreadSafeFunction`）触发 `SIGABRT`，整个应用崩溃。

解决方案：在 `app.on('before-quit')` 时设置 `isShuttingDown = true`，所有 PTY 回调都先检查这个标志：

```typescript
ptyProcess.onData((data) => {
  if (isShuttingDown) return; // 应用正在关闭，静默丢弃
  window.webContents.send(IPC_CHANNELS.TERMINAL_OUTPUT, data);
});
```

### Windows 专项：退出超时差异

```typescript
// Windows 需要更长的等待时间
const PTY_EXIT_TIMEOUT_WINDOWS = 2000;  // 2 秒
const PTY_EXIT_TIMEOUT_UNIX = 500;       // 500 毫秒
```

Windows 进程终止是异步的（不像 Unix 的 `SIGTERM` 同步确认），所以等待时间需要更长。这是跨平台开发中常被忽视的细节。

---

## 15.5　Claude 集成：在终端里识别 AI 信号

`claude-integration-handler.ts` 是终端系统最复杂的部分——它让普通的 shell 终端具备理解 Claude Code 输出的能力。

### 认证终端 ID 模式

当用户在终端里运行 `claude login`，Auto-Claude 需要知道这个终端是为哪个账户 Profile 做认证：

```typescript
// 认证终端的 ID 格式：claude-login-{profileId}-{timestamp}
const AUTH_TERMINAL_ID_PATTERN = /^claude-login-([a-z0-9-]+)-(\d{13,})$/;

function extractProfileIdFromAuthTerminalId(terminalId: string): string | null {
  const match = terminalId.match(AUTH_TERMINAL_ID_PATTERN);
  return match ? match[1] : null;
}
```

示例：
- `claude-login-default-1737298800000` → profile ID `default`
- `claude-login-my-profile-1737298800000` → profile ID `my-profile`

### PII 脱敏：邮件地址掩码

终端输出可能包含用户的邮件地址（认证成功时 Claude 会显示）。写日志时必须脱敏：

```typescript
function maskEmail(email: string): string {
  // 'john.doe@example.com' → 'j***@e***.com'
  const [local, domain] = email.split('@');
  if (!domain) return '***';
  const domainParts = domain.split('.');
  const maskedLocal = local[0] + '***';
  const maskedDomain = domainParts[0][0] + '***.' + domainParts.slice(1).join('.');
  return `${maskedLocal}@${maskedDomain}`;
}
```

这是生产系统中处理 PII 的标准做法：日志中从不出现完整邮件，只保留足够的信息用于调试。

---

## 15.6　会话持久化：7 天 TTL 与版本控制

`session-persistence.ts` 负责在 Electron 重启后恢复终端会话元数据：

```typescript
// 存储位置：Electron userData 目录（跨平台）
const SESSIONS_FILE = path.join(app.getPath('userData'), 'terminal-sessions.json');
const BUFFERS_DIR = path.join(app.getPath('userData'), 'terminal-buffers');

// 会话过期时间：7 天
const MAX_SESSION_AGE_MS = 7 * 24 * 60 * 60 * 1000;

loadSessions(): TerminalSessionState[] {
  const data: TerminalSessionsFile = JSON.parse(
    fs.readFileSync(SESSIONS_FILE, 'utf8')
  );

  // 版本检查：文件格式变更时必须重置
  if (data.version !== 2) {
    console.warn('[SessionPersistence] Incompatible version, starting fresh');
    return [];
  }

  // 过滤过期会话
  const now = Date.now();
  return data.sessions.filter(
    (s) => now - s.lastActiveAt < MAX_SESSION_AGE_MS
  );
}
```

**双层恢复机制**：
1. **PTY Daemon 存活**：主进程重连，直接从环形缓冲区读取最近 100KB 输出，无缝恢复
2. **PTY Daemon 不存活**（系统重启）：从 `terminal-sessions.json` 读取元数据，重新创建 PTY，从 `terminal-buffers/` 目录读取持久化的缓冲区内容

`★ Insight ─────────────────────────────────────`
版本号 `version: 2` 是文件格式演进的关键。当持久化格式需要变更时（比如新增字段、改变结构），版本号让旧格式的文件被干净地丢弃，而不是尝试兼容导致部分数据损坏。这是"宁可清空重来，不要脏数据"的工程哲学。
`─────────────────────────────────────────────────`

---

## 15.7　架构总览：数据流图

```
用户键盘输入
    │
    ▼
xterm.js（渲染进程）
    │ IPC: terminal-input
    ▼
Electron 主进程
    │ Unix Socket / Named Pipe
    ▼
PTY Daemon
    │ write()
    ▼
Shell 进程 (bash/Claude Code)
    │ 输出
    ▼
PTY Daemon
    ├─ 追加到环形缓冲区 (100KB)
    └─ 广播给所有订阅者
          │ Socket
          ▼
    Electron 主进程
          │ IPC: terminal-output
          ▼
    xterm.js 渲染 + Zustand 缓冲
```

Claude Code 集成嵌入在主进程的输出处理层：每行输出都经过 `output-parser.ts` 扫描，识别 rate limit 信号（触发账户切换）和 OAuth token（更新凭据存储）。

---

## 15.8　Lab 15.1：迷你 PTY 服务

**目标**：实现一个简化版的 PTY 管理服务，包含环形缓冲区和多订阅者模式。

### Part A：环形缓冲区

```typescript
// lab15/ring-buffer.ts

export class RingBuffer {
  private chunks: string[] = [];
  private totalSize: number = 0;
  private readonly maxSize: number;

  constructor(maxSizeBytes: number = 100_000) {
    this.maxSize = maxSizeBytes;
  }

  /**
   * 追加数据，超出上限时从头部淘汰旧数据
   */
  append(data: string): void {
    // TODO Part A: 追加数据到 chunks
    // 当 totalSize > maxSize 且 chunks.length > 1 时，从头部移除
    // 注意：至少保留 1 块，即使单块超过 maxSize
  }

  /**
   * 获取所有缓冲内容（拼接）
   */
  getAll(): string {
    // TODO Part A: 拼接所有 chunks 返回
    return '';
  }

  /**
   * 获取当前字节数
   */
  getSize(): number {
    return this.totalSize;
  }

  /**
   * 清空缓冲区
   */
  clear(): void {
    // TODO Part A
  }
}

// 验证：
const buf = new RingBuffer(10); // 10 字节上限
buf.append('Hello');  // 5 字节
buf.append('World');  // 5 字节，总 10 字节（刚好）
buf.append('!!!');    // 超出，应淘汰 'Hello'
console.log(buf.getAll()); // → 'World!!!'（或类似，取决于实现）
console.log(buf.getSize()); // ≤ 10
```

### Part B：多订阅者广播

```typescript
// lab15/pty-session.ts
import { EventEmitter } from 'events';

export class PtySession {
  private id: string;
  private subscribers: Set<(data: string) => void> = new Set();
  private buffer: RingBuffer;
  private emitter: EventEmitter;

  constructor(id: string) {
    this.id = id;
    this.buffer = new RingBuffer();
    this.emitter = new EventEmitter();
  }

  /**
   * 订阅输出，返回取消订阅函数
   * 新订阅者会立即收到历史缓冲区内容
   */
  subscribe(callback: (data: string) => void): () => void {
    // TODO Part B:
    // 1. 立即发送历史缓冲（如果非空）
    // 2. 注册到 subscribers Set
    // 3. 返回取消函数（从 Set 中移除）
    return () => {};
  }

  /**
   * 模拟 PTY 输出：写入缓冲并广播给所有订阅者
   */
  receive(data: string): void {
    // TODO Part B:
    // 1. 追加到 buffer
    // 2. 广播给所有 subscribers
  }
}

// 验证：
const session = new PtySession('test-pty');
session.receive('历史输出\n');

const received: string[] = [];
const cancel = session.subscribe(data => received.push(data));

// 新订阅者应立即收到历史内容
console.log(received[0]); // → '历史输出\n'

session.receive('新输出\n');
console.log(received[1]); // → '新输出\n'

cancel(); // 取消订阅
session.receive('取消后\n');
console.log(received.length); // → 2（取消后不再收到）
```

### Part C：NDJSON 消息协议

```typescript
// lab15/ndjson-parser.ts

export class NdJsonParser {
  private buffer: string = '';

  /**
   * 处理流式数据块，返回解析出的完整消息列表
   * 换行符作为消息分隔符，不完整的消息保留在内部缓冲
   */
  feed(chunk: string): unknown[] {
    // TODO Part C:
    // 1. 追加到 buffer
    // 2. 按 '\n' 分割
    // 3. 最后一段（可能不完整）保留在 buffer
    // 4. 对每个完整行尝试 JSON.parse，失败则跳过
    return [];
  }
}

// 验证：
const parser = new NdJsonParser();

// 分两次收到，第一次不完整
let msgs = parser.feed('{"type":"ping"}\n{"type":"cr');
console.log(msgs.length); // → 1（只有第一条完整）
console.log((msgs[0] as any).type); // → 'ping'

// 第二次完成第二条消息
msgs = parser.feed('eate","id":"pty-1"}\n');
console.log(msgs.length); // → 1
console.log((msgs[0] as any).type); // → 'create'
```

**验证标准**：
- Part A：`RingBuffer` 不超过 maxSize，淘汰策略正确，至少保留 1 块
- Part B：新订阅者立即收到历史内容；取消后不再接收新消息
- Part C：跨块的不完整 JSON 能正确拼接；无效 JSON 被静默跳过

**进阶挑战**：
- 给 `PtySession` 添加 `replay()` 方法：向新订阅者只发送最近 N 字节的历史（不是全部）
- 实现 `PtyManager`：管理多个 `PtySession`，支持 `create`/`kill`/`list` 操作
- 给 `NdJsonParser` 添加最大行长度保护：超过 1MB 的单行直接丢弃

---

## 本章要点回顾

| 概念 | 核心设计 | 源码位置 |
|------|---------|---------|
| PTY Daemon | 独立进程，跨 Electron 重启存活 | `pty-daemon.ts` |
| Unix Socket / Named Pipe | 跨平台 IPC，chmod 600 权限 | `pty-daemon.ts:16` |
| NDJSON 协议 | 换行分隔 JSON，流式解析 | `pty-daemon.ts:145` |
| 环形缓冲区 | 100KB 上限，FIFO 淘汰 | `pty-daemon.ts:22` |
| `clients: Set` | 多订阅者广播，零数据复制 | `pty-daemon.ts:38` |
| `isShuttingDown` | 防 Native 回调访问已销毁资源 | `pty-manager.ts:29` |
| 认证终端 ID | 正则提取 profileId，关联 OAuth | `claude-integration-handler.ts:54` |
| maskEmail | 日志中脱敏 PII | `claude-integration-handler.ts:80` |
| 会话持久化版本号 | format break 时干净重置 | `session-persistence.ts:70` |
| 7 天 TTL | 过期会话自动清理 | `session-persistence.ts:21` |
