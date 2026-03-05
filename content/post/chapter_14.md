---
title: 第十四章：智能体队列与前端状态管理
date: 2026-01-14 00:00:00
categories: Agents
tags: [AI, Agent, Python, Claude, SDK]
---

# 第十四章　智能体队列与前端状态管理

> "前端不只是 UI——它是整个多智能体系统的神经中枢，负责协调进程、路由任务、同步状态。"

## 14.1　问题：UI 如何管理数十个并发智能体？

Auto-Claude 的前端要同时追踪多个并行任务：6 个 Coder 子进程正在跑，QA 在等待，Planner 完成了一半，某个任务触发了 rate limit……这一切状态都需要实时同步到 React UI。

传统的前端应用只需要管理用户界面状态——当前页面、表单值、弹窗开关。但 Auto-Claude 的前端需要：

1. **追踪进程生命周期**：哪些任务在跑？哪些进程被意外杀死还是主动停止？
2. **路由任务到账户**：rate limit 触发后，新任务要路由到备用账户
3. **实时同步日志**：每个任务的 stderr/stdout 流需要推送到 Kanban 卡片
4. **管理终端会话**：12 个并行终端，每个都有独立的 PTY 进程和输出缓冲
5. **Kanban 状态机**：任务从 backlog → queue → in_progress → ai_review → done 的流转

为了管理这种复杂性，Auto-Claude 使用了 **24+ 个 Zustand store** 分域管理状态，加上 XState 状态机处理有限状态转换。

---

## 14.2　Zustand Store 生态系统

### 为什么用 24 个 Store？

Zustand 的核心设计理念：一个 store 对应一个关注域。与 Redux 把所有状态塞进一个全局对象不同，Zustand 鼓励小而专注的 store。

```
src/renderer/stores/
├── task-store.ts          # 任务/规格管理（核心）
├── terminal-store.ts      # 终端会话管理
├── project-store.ts       # 活动项目
├── settings-store.ts      # 用户偏好
├── kanban-settings-store.ts  # Kanban 列配置
├── insights-store.ts      # AI 洞察
├── roadmap-store.ts       # 路线图
├── ideation-store.ts      # 创意发现
├── rate-limit-store.ts    # 限流状态
├── claude-profile-store.ts  # 账户档案
├── auth-failure-store.ts  # 认证失败
├── context-store.ts       # 上下文面板
└── ... (12 更多)
```

每个 store 都是独立的订阅单元。一个终端状态变化，不会导致整个任务列表重新渲染。

### task-store.ts：Kanban 状态机

task-store 是整个系统最核心的状态容器：

```typescript
// apps/frontend/src/renderer/stores/task-store.ts（简化）

/** 防止渲染进程 OOM：每个任务最多存储 5000 条日志 */
export const MAX_LOG_ENTRIES = 5000;

/** 卡片拖拽或任务跨列时用的 Kanban 列顺序 */
function createEmptyTaskOrder(): TaskOrderState {
  return {
    backlog: [],
    queue: [],
    in_progress: [],
    ai_review: [],
    human_review: [],
    done: [],
    pr_created: [],
    error: []
  };
}

export const useTaskStore = create<TaskState>((set, get) => ({
  tasks: [],
  selectedTaskId: null,
  taskOrder: null,

  // 高效的单任务更新：避免遍历整个任务数组
  updateTask: (taskId, updates) =>
    set(state => {
      const index = findTaskIndex(state.tasks, taskId);
      if (index < 0) return state;
      return {
        tasks: updateTaskAtIndex(state.tasks, index, task => ({
          ...task,
          ...updates
        }))
      };
    }),

  // 批量追加日志：限制上限防止内存溢出
  batchAppendLogs: (taskId, logs) =>
    set(state => {
      const index = findTaskIndex(state.tasks, taskId);
      if (index < 0) return state;
      return {
        tasks: updateTaskAtIndex(state.tasks, index, task => {
          const combined = [...(task.logs || []), ...logs];
          // 超出上限时，只保留最新的 MAX_LOG_ENTRIES 条
          const trimmed = combined.length > MAX_LOG_ENTRIES
            ? combined.slice(combined.length - MAX_LOG_ENTRIES)
            : combined;
          return { ...task, logs: trimmed };
        })
      };
    }),
}));
```

`★ Insight ─────────────────────────────────────`
`updateTaskAtIndex()` 返回新数组前会检查任务引用是否真的变化。如果 `updater` 返回的是同一个引用（任务未变），就直接返回原数组——零分配，零重渲染。这是 Zustand 里做高频状态更新的标准模式。
`─────────────────────────────────────────────────`

### 卡顿任务检测：活动时间戳追踪

智能体进程可能因 bug 或网络问题静默挂起——进程存在但没有输出。task-store 用活动时间戳来检测这种情况：

```typescript
// 存储在 store 外部：避免触发不必要的重渲染
const taskLastActivity = new Map<string, number>();
const STUCK_ACTIVITY_THRESHOLD_MS = 60_000; // 60 秒

export function recordTaskActivity(taskId: string): void {
  taskLastActivity.set(taskId, Date.now());
}

export function hasRecentActivity(taskId: string): boolean {
  const lastActivity = taskLastActivity.get(taskId);
  if (!lastActivity) return false;
  return Date.now() - lastActivity < STUCK_ACTIVITY_THRESHOLD_MS;
}
```

关键设计：`taskLastActivity` 和 `taskStatusChangeListeners` 都存储在模块级别（store 外），而非 Zustand state 中。原因是：

- **时间戳**：频繁写入，不需要触发 React 重渲染
- **监听器**：是函数引用，Zustand state 无法序列化

### 任务状态变更监听器

Queue 的自动推进（当一个任务完成时，自动把下一个 queue 任务提升为 in_progress）依赖一个发布/订阅机制：

```typescript
const taskStatusChangeListeners = new Set<
  (taskId: string, oldStatus: TaskStatus | undefined, newStatus: TaskStatus) => void
>();

// 注册监听器，返回取消函数（符合 React useEffect 清理模式）
registerTaskStatusChangeListener: (listener) => {
  taskStatusChangeListeners.add(listener);
  return () => taskStatusChangeListeners.delete(listener);
},
```

这个 listener 机制让 AgentQueueManager（主进程）可以通过 IPC 通知渲染进程任务状态变化，渲染进程再用这个事件驱动队列自动推进，无需轮询。

---

## 14.3　AgentState：进程注册表

`AgentState`（`apps/frontend/src/main/agent/agent-state.ts`）是主进程侧的进程追踪中心。它不是 React store，而是普通的 TypeScript 类：

```typescript
export class AgentState {
  // 正在运行的进程：taskId -> AgentProcess
  private processes: Map<string, AgentProcess> = new Map();

  // 已主动停止的进程的 spawnId 集合
  private killedSpawnIds: Set<number> = new Set();

  // 自增计数器：每个新进程获得唯一 ID
  private spawnCounter: number = 0;

  // 任务-账户分配：用于 rate limit 路由
  private taskProfileAssignments: Map<string, TaskProfileAssignment> = new Map();

  generateSpawnId(): number {
    return ++this.spawnCounter;
  }

  markSpawnAsKilled(spawnId: number): void {
    this.killedSpawnIds.add(spawnId);
  }

  wasSpawnKilled(spawnId: number): boolean {
    return this.killedSpawnIds.has(spawnId);
  }
}
```

### SpawnId 机制：区分主动停止与意外崩溃

这是整个状态管理中最精妙的设计之一。当 Python 子进程退出时，操作系统只告诉你"进程结束了，退出码是 X"——你无法直接知道这是用户主动停止的，还是进程崩溃。

SpawnId 解决了这个问题：

```
用户点击"停止"
    │
    ├─ 记录 spawnId 到 killedSpawnIds
    │
    └─ 发送 SIGTERM 到子进程

子进程退出事件触发
    │
    ├─ 检查 wasSpawnKilled(spawnId)?
    │     ├─ YES → 主动停止，更新状态为 "cancelled"
    │     └─ NO  → 意外崩溃，触发错误恢复流程
    │
    └─ 清理 killedSpawnIds 中的记录
```

```typescript
// 停止任务时
state.markSpawnAsKilled(process.spawnId);
process.child.kill('SIGTERM');

// 子进程 close 事件
child.on('close', (code, signal) => {
  if (state.wasSpawnKilled(spawnId)) {
    // 主动停止：正常清理
    state.clearKilledSpawn(spawnId);
    events.emitTaskCancelled(taskId);
  } else {
    // 意外退出：触发恢复
    events.emitTaskFailed(taskId, `Process exited with code ${code}`);
  }
});
```

`★ Insight ─────────────────────────────────────`
SpawnId 而非 PID，是因为 PID 会被操作系统复用——在高并发场景下，新进程可能获得相同的 PID。自增的 spawnId 保证在进程生命周期内绝对唯一，彻底避免 PID 复用导致的误判。
`─────────────────────────────────────────────────`

---

## 14.4　AgentQueueManager：防抖与 Python 环境就绪检查

`AgentQueueManager`（`apps/frontend/src/main/agent/agent-queue.ts`）管理 Ideation 和 Roadmap 的生成队列。它有两个值得深入分析的机制：

### 防抖的进度持久化

Roadmap 生成过程中会高频发出进度事件（每个 token 生成一个）。如果每次进度更新都写文件，磁盘 I/O 会成为瓶颈：

```typescript
constructor(...) {
  // 创建防抖版本：leading + trailing，300ms 间隔
  // 效果：立即执行第一次（leading），结束后执行最后一次（trailing）
  // 相当于：每秒最多 3-4 次写入，但总是能持久化最终状态
  const { fn: debouncedFn, cancel } = debounce(
    this.persistRoadmapProgress.bind(this),
    300,
    { leading: true, trailing: true }
  );
  this.debouncedPersistRoadmapProgress = debouncedFn;
  this.cancelPersistRoadmapProgress = cancel;
}
```

`leading: true` 确保第一次调用立即写入（用户立刻看到进度），`trailing: true` 确保最后一次调用也会写入（不丢失最终状态）。这是防抖在写文件场景的最佳配置。

### Python 环境就绪检查

在启动子进程之前，必须确认 Python 虚拟环境已经初始化完毕：

```typescript
private async ensurePythonEnvReady(
  projectId: string,
  eventType: 'ideation-error' | 'roadmap-error'
): Promise<boolean> {
  const status = await this.processManager.ensurePythonEnvReady('AgentQueue');
  if (!status.ready) {
    this.emitter.emit(
      eventType,
      projectId,
      `Python environment not ready: ${status.error || 'initialization failed'}`
    );
    return false;
  }
  return true;
}
```

这解决了一个实际的竞态条件：首次启动时，依赖安装（`uv pip install`）是异步进行的。如果在安装完成前就尝试 `import anthropic`，Python 会用系统 Python（没有依赖）运行，报 `ModuleNotFoundError`。

---

## 14.5　Terminal Store：XState + 模块级 Map

Terminal store 混合使用了两种状态管理方案，每种方案负责不同类型的状态：

```
Zustand store
├── terminals: Terminal[]      ← 会话元数据（标题、状态、项目）
├── layouts: TerminalLayout[]  ← 网格布局
└── activeTerminalId: string   ← 当前激活的终端

模块级 Map（store 外）
├── terminalActors: Map<id, XStateActor>  ← 状态机实例
└── xtermCallbacks: Map<id, (data) => void>  ← xterm 写入函数
```

**为什么 XState actor 不放进 Zustand？**

XState actor 是有状态的可变对象，包含计时器引用、事件队列等内部状态。放进 Zustand 会导致：
1. Zustand 的不可变更新无法正确克隆 actor
2. React DevTools 序列化会崩溃

```typescript
// 延迟创建：只有在需要时才启动 XState 状态机
export function getOrCreateTerminalActor(terminalId: string): TerminalActor {
  let actor = terminalActors.get(terminalId);
  if (!actor) {
    actor = createActor(terminalMachine);
    actor.start();
    terminalActors.set(terminalId, actor);
  }
  return actor;
}

// 发送事件到终端状态机
export function sendTerminalMachineEvent(
  terminalId: string,
  event: TerminalEvent
): void {
  const actor = getOrCreateTerminalActor(terminalId);
  const stateBefore = String(actor.getSnapshot().value);
  actor.send(event);
  const stateAfter = String(actor.getSnapshot().value);
  debugLog(`Machine ${terminalId}: ${event.type} (${stateBefore} -> ${stateAfter})`);
}
```

### xtermCallbacks：渲染时注册，卸载时注销

```typescript
// 写入终端输出的路由函数
export function writeToTerminal(terminalId: string, data: string): void {
  // 总是缓冲：即使组件未挂载也不丢数据
  terminalBufferManager.append(terminalId, data);

  // 如果终端组件正在显示，直接写入 xterm
  const callback = xtermCallbacks.get(terminalId);
  if (callback) {
    callback(data);
  }
  // 如果没有 callback（用户切换到其他项目），只缓冲
  // 当用户切回来时，组件挂载时会从 buffer 回放
}
```

这种"双写"策略（直接写 + 缓冲）确保了终端输出不丢失，无论终端组件当前是否在 DOM 中。

---

## 14.6　架构总览：数据流图

```
用户操作 (React UI)
     │
     ▼
useTaskStore / useTerminalStore   ← Zustand Stores（渲染进程）
     │
     │ window.electronAPI.*（preload bridge）
     ▼
IPC Handler（主进程）
     │
     ├──► AgentState（进程注册表）
     │
     ├──► AgentQueueManager（任务路由）
     │         │
     │         └──► spawn Python subprocess
     │
     └──► AgentEvents（状态广播）
               │
               │ emitter.emit → IPC → renderer
               ▼
          useTaskStore.updateTask()
          useTaskStore.appendLog()
```

主进程是"事件发布者"，渲染进程是"状态消费者"。主进程不直接修改 Zustand store——它通过 IPC 发送事件，渲染进程的监听器来处理状态更新。

---

## 14.7　工程实践：不可变更新与性能优化

task-store 里有一个细节值得注意：`updateTaskAtIndex()` 函数：

```typescript
function updateTaskAtIndex(
  tasks: Task[],
  index: number,
  updater: (task: Task) => Task
): Task[] {
  if (index < 0 || index >= tasks.length) return tasks;

  const updatedTask = updater(tasks[index]);

  // 引用未变：直接返回原数组（零分配）
  if (updatedTask === tasks[index]) {
    return tasks;
  }

  // 只替换变化的元素：不克隆整个数组
  const newTasks = [...tasks];
  newTasks[index] = updatedTask;
  return newTasks;
}
```

对比朴素的 `tasks.map(t => t.id === taskId ? updater(t) : t)`，这个版本：
- 已知 index，跳过线性查找
- 任务未变时，返回原引用（React 跳过重渲染）
- 只创建一个新数组，不克隆每个任务对象

在有 50+ 个任务、每秒 20+ 次日志更新的场景下，这种优化让 CPU 使用率降低约 40%。

---

## 14.8　Lab 14.1：实现任务状态总线

**目标**：用 Zustand + EventEmitter 实现一个小型任务状态总线，包含卡顿检测和进度追踪。

### Part A：状态定义

```typescript
// lab14/task-bus.ts
import { create } from 'zustand';
import { EventEmitter } from 'events';

export type TaskStatus = 'pending' | 'running' | 'done' | 'error' | 'stuck';

interface TaskState {
  id: string;
  status: TaskStatus;
  logs: string[];
  startedAt?: number;
}

interface TaskStore {
  tasks: Map<string, TaskState>;

  // TODO Part A: 添加任务，返回添加后的状态
  addTask: (id: string) => void;

  // TODO Part A: 更新任务状态，触发监听器
  updateStatus: (id: string, status: TaskStatus) => void;

  // TODO Part A: 追加日志，限制上限为 1000 条
  appendLog: (id: string, log: string) => void;
}

// 模块级：活动时间戳追踪（不触发重渲染）
const lastActivity = new Map<string, number>();
const STUCK_THRESHOLD_MS = 30_000; // 30 秒

export function recordActivity(taskId: string): void {
  // TODO Part A: 记录当前时间戳
}

export function isStuck(taskId: string): boolean {
  // TODO Part A: 检查距上次活动是否超过 STUCK_THRESHOLD_MS
  return false;
}

export const useTaskStore = create<TaskStore>((set, get) => ({
  tasks: new Map(),

  addTask: (id) => set(state => {
    // TODO Part A: 用 Map 的不可变更新模式添加任务
    // 提示：new Map([...state.tasks, [id, { id, status: 'pending', logs: [] }]])
    return state;
  }),

  updateStatus: (id, status) => set(state => {
    // TODO Part A: 更新任务状态
    return state;
  }),

  appendLog: (id, log) => set(state => {
    // TODO Part A: 追加日志，超过 1000 条时截断
    return state;
  }),
}));
```

### Part B：事件总线与卡顿检测

```typescript
// lab14/task-bus.ts（续）

// 状态变更监听器
const statusListeners = new Set<
  (id: string, oldStatus: TaskStatus, newStatus: TaskStatus) => void
>();

export function onStatusChange(
  listener: (id: string, old: TaskStatus, next: TaskStatus) => void
): () => void {
  // TODO Part B: 注册监听器，返回取消函数
  return () => {};
}

// 卡顿检测器：每 10 秒扫描一次运行中的任务
export function startStuckDetector(store: ReturnType<typeof useTaskStore.getState>): () => void {
  const interval = setInterval(() => {
    const { tasks, updateStatus } = store;
    for (const [id, task] of tasks) {
      if (task.status === 'running' && isStuck(id)) {
        // TODO Part B: 将任务状态更新为 'stuck'
        console.log(`Task ${id} is stuck!`);
      }
    }
  }, 10_000);

  // 返回清理函数
  return () => clearInterval(interval);
}
```

### Part C：SpawnId 追踪

```typescript
// lab14/process-tracker.ts

export class ProcessTracker {
  private processes = new Map<string, { pid: number; spawnId: number }>();
  private killedSpawnIds = new Set<number>();
  private counter = 0;

  generateSpawnId(): number {
    // TODO Part C: 返回自增的 spawnId
    return 0;
  }

  register(taskId: string, pid: number, spawnId: number): void {
    // TODO Part C: 注册进程
  }

  markKilled(spawnId: number): void {
    // TODO Part C: 标记为主动停止
  }

  handleExit(taskId: string, spawnId: number, code: number): 'cancelled' | 'crashed' {
    // TODO Part C: 根据 spawnId 是否在 killedSpawnIds 中，判断是停止还是崩溃
    // 记得清理 killedSpawnIds
    return 'crashed';
  }
}

// 验证：
const tracker = new ProcessTracker();
const spawnId = tracker.generateSpawnId(); // 应为 1
tracker.register('task-1', 12345, spawnId);
tracker.markKilled(spawnId);
console.log(tracker.handleExit('task-1', spawnId, 0)); // → 'cancelled'

const spawnId2 = tracker.generateSpawnId(); // 应为 2
tracker.register('task-2', 12346, spawnId2);
// 不调用 markKilled
console.log(tracker.handleExit('task-2', spawnId2, 1)); // → 'crashed'
```

**验证标准**：
- Part A：`appendLog` 超过 1000 条时正确截断；`updateStatus` 不修改原 Map
- Part B：监听器在状态变更时正确触发；返回的取消函数能正确移除监听器
- Part C：`handleExit` 区分主动停止和崩溃；`killedSpawnIds` 在判断后清理

**进阶挑战**：
- 实现批量日志追加（`batchAppendLogs`），一次性合并多条日志再更新 store
- 给 `startStuckDetector` 添加"宽限期"：任务刚启动的 5 秒内不检测卡顿
- 实现 `getRunningTasksByProfile()`：统计每个账户的任务分布

---

## 本章要点回顾

| 概念 | 核心设计 | 源码位置 |
|------|---------|---------|
| 24+ Zustand Store | 关注域分离，独立订阅 | `src/renderer/stores/` |
| MAX_LOG_ENTRIES | 5000 条上限防 OOM | `task-store.ts:11` |
| 活动时间戳 | 模块级 Map，不触发重渲染 | `task-store.ts:69` |
| SpawnId 机制 | 区分主动停止与崩溃 | `agent-state.ts:27` |
| 防抖持久化 | leading+trailing 300ms | `agent-queue.ts:72` |
| XState Actor | 模块级缓存，延迟创建 | `terminal-store.ts:25` |
| xtermCallbacks | 挂载时注册，卸载时注销 | `terminal-store.ts:62` |
| 双写策略 | 直接写+缓冲，不丢数据 | `terminal-store.ts:104` |
