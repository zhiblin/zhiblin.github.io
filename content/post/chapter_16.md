---
title: 第十六章：多账户 Profile 系统
date: 2026-01-16 00:00:00
categories: Agents
tags: [AI, Agent, Python, Claude, SDK]
---

# 第十六章　多账户 Profile 系统

> "一个智能体系统在深夜工作时，不能因为某个账户触发了 rate limit 而停下来等人处理——它需要自动切换到备用账户，像接力赛一样继续奔跑。"

## 16.1　问题：为什么需要多账户系统？

Claude API 有两种主要的限制机制：

- **Rate limit（429）**：短期内请求过多，需要等待（分钟级）
- **Usage limit**：按会话或按周的用量阈值（小时级到天级）

对于 Auto-Claude 这样的自动化系统，一个账户的限制会导致整个工作流暂停。解决方案：注册多个 Claude 账户（或配置多个 API key），在一个账户受限时自动切换到另一个。

但实现这个功能需要解决一系列技术问题：
1. **如何存储多个账户的凭据？** 敏感数据不能明文存在磁盘
2. **如何选择最优账户？** 不只是"找一个没 rate limit 的"——用量越低越好，用户有优先级配置
3. **OAuth token 怎么维护？** Token 会过期，需要自动刷新
4. **如何统一管理 OAuth 账户和 API Key 账户？** 两种账户类型，不同的认证机制

---

## 16.2　统一账户模型：UnifiedAccount

`UnifiedAccount` 是整个 Profile 系统的核心抽象，把两种不同类型的账户（OAuth 订阅账户和 API Key 账户）统一成一个接口：

```typescript
// apps/frontend/src/shared/types/unified-account.ts（概念化）

type UnifiedAccount = {
  id: string;          // 'oauth:profile-id' 或 'api:profile-id'
  type: 'oauth' | 'api';
  displayName: string;

  // 可用性状态
  isAuthenticated: boolean;
  isAvailable: boolean;
  isRateLimited: boolean;
  rateLimitType?: 'session' | 'weekly';

  // 用量指标（OAuth 账户专有）
  weeklyPercent?: number;   // 当前周用量百分比
  sessionPercent?: number;  // 当前会话用量百分比
};
```

**为什么需要这层抽象？**

- **OAuth 账户**（Claude Code 订阅）：有复杂的 Token 生命周期管理，有用量百分比概念
- **API 账户**（API key）：无用量限制（`hasUnlimitedUsage = true`），只需验证 key 有效性

统一模型让选择算法不需要区分这两种类型，大大简化了选择逻辑。

---

## 16.3　Profile Scorer：优先级 + 可用性双重过滤

`getBestAvailableUnifiedAccount()` 是多账户切换的核心算法。它的设计理念是**用户意图优先于系统启发式**：

```
选择算法流程
     │
     ▼
转换所有账户为 UnifiedAccount（OAuth + API）
     │
     ▼
按用户配置的优先级排序
     │
     ▼
过滤可用账户（通过所有可用性检查）
     │
     ├─ 有可用账户? → 返回优先级最高的
     │
     └─ 没有? → 降级：返回"最不坏"的选项
```

### 可用性检查三重门

```typescript
// apps/frontend/src/main/claude-profile/profile-scorer.ts（简化）

function scoreUnifiedAccount(
  account: UnifiedAccount,
  priorityIndex: number,
  settings: ClaudeAutoSwitchSettings
): ScoredUnifiedAccount {
  let score = 100;

  // 门 1：认证状态（最关键）
  if (!account.isAuthenticated) {
    score = -1000;  // 未认证，基本不可用
    return { account, score, priorityIndex, isAvailable: false };
  }

  // 门 2：Rate Limit 状态
  if (account.isRateLimited) {
    if (account.rateLimitType === 'weekly') {
      score = -500;  // 周限额更严重（重置慢）
    } else {
      score = -200;  // 会话限额，较快恢复
    }
  }

  // 门 3：用量阈值（默认：Session 95%，Weekly 99%）
  if (account.weeklyPercent !== undefined &&
      account.weeklyPercent >= settings.weeklyThreshold) {
    isOverThreshold = true;
  }
  if (account.sessionPercent !== undefined &&
      account.sessionPercent >= settings.sessionThreshold) {
    isOverThreshold = true;
  }

  // 用量越高，分数越低（即使未达阈值也有惩罚）
  score -= (account.weeklyPercent ?? 0) * 0.3;
  score -= (account.sessionPercent ?? 0) * 0.1;

  const isAvailable = score > 0 && !account.isRateLimited && !isOverThreshold;
  return { account, score, priorityIndex, isAvailable };
}
```

`★ Insight ─────────────────────────────────────`
注意阈值判断使用 `>=`（大于等于），而不是 `>`。例如配置 `sessionThreshold = 95`，当用量正好到 95% 时就立即切换，而不是等到 96%。注释里解释："we want to switch proactively BEFORE hitting hard limits"——主动在硬限额前切换，避免任务被打断。
`─────────────────────────────────────────────────`

### 降级策略：最不坏选项

当所有账户都不可用时，系统不会直接报错，而是返回"最不坏"的那个：

```typescript
// 降级评分：从 100 开始扣分
function calculateFallbackScore(profile, settings): number {
  let score = 100;

  if (!isProfileAuthenticated(profile)) {
    score -= 1000;  // 未认证最差
  }

  const rateLimitStatus = isProfileRateLimited(profile);
  if (rateLimitStatus.limited) {
    if (rateLimitStatus.type === 'weekly') {
      score -= 500;  // 周限额扣 500
    } else {
      score -= 200;  // 会话限额扣 200
    }

    // 越快重置的账户，降级得分越高
    if (rateLimitStatus.resetAt) {
      const hoursUntilReset = (resetAt - now) / 3600000;
      score += Math.max(0, 50 - hoursUntilReset); // 1小时内重置：+49分
    }
  }

  // 用量惩罚
  score -= profile.usage.weeklyUsagePercent * 0.3;
  score -= profile.usage.sessionUsagePercent * 0.1;

  return score;
}
```

这个降级机制的实际意义：系统宁可选择一个"几分钟后就能恢复"的受限账户，也不选择"不知道什么时候能用"的未认证账户。

---

## 16.4　OAuth Token 生命周期管理

Claude 订阅账户使用 OAuth 2.0 认证。OAuth access token 有过期时间（8小时），所以需要主动维护 token 的有效性。

### 双轨刷新策略

```
                     Token 生命周期
    ────────────────────────────────────────────────
    创建        到期前 30 分钟          到期
     │               │                  │
     ▼               ▼                  ▼
    ┌─────────────────────────────────────────────┐
    │              Token 有效期 (8h)               │
    └─────────────────────────────────────────────┘
                     ▲
                     │ 主动刷新（Proactive）
                     │ 30 分钟缓冲区
                     │
                     └── 如果来不及主动刷新：
                         401 错误触发被动刷新（Reactive）
```

```typescript
// apps/frontend/src/main/claude-profile/token-refresh.ts

// 主动刷新阈值：到期前 30 分钟
const PROACTIVE_REFRESH_THRESHOLD_MS = 30 * 60 * 1000;

export function isTokenExpiredOrNearExpiry(
  expiresAt: number | null,
  thresholdMs = PROACTIVE_REFRESH_THRESHOLD_MS
): boolean {
  // 不知道过期时间 → 保守假设已过期（宁可多刷新，不能用过期 token）
  if (expiresAt === null) return true;

  const expiryThreshold = expiresAt - thresholdMs;
  return Date.now() >= expiryThreshold;
}
```

这个"不知道就假设过期"的保守策略很重要：在历史数据缺失的情况下，宁可触发一次不必要的刷新，也不能让一次真正的过期请求失败。

### 刷新实现：指数退避 + 不可重试错误分类

```typescript
export async function refreshOAuthToken(
  refreshToken: string,
  configDir?: string
): Promise<TokenRefreshResult> {
  const MAX_REFRESH_RETRIES = 2;
  const RETRY_DELAY_BASE_MS = 1000;

  for (let attempt = 0; attempt <= MAX_REFRESH_RETRIES; attempt++) {
    if (attempt > 0) {
      // 指数退避：1s, 2s
      const delay = RETRY_DELAY_BASE_MS * 2 ** (attempt - 1);
      await new Promise(resolve => setTimeout(resolve, delay));
    }

    const body = new URLSearchParams({
      grant_type: 'refresh_token',
      refresh_token: refreshToken,
      client_id: CLAUDE_CODE_CLIENT_ID  // 公开客户端 ID
    });

    const response = await fetch(ANTHROPIC_TOKEN_ENDPOINT, {
      method: 'POST',
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
      body: body.toString()
    });

    if (!response.ok) {
      const errorData = await response.json();
      const errorCode = errorData.error || `http_${response.status}`;

      // 不可重试的永久错误：立即返回
      if (errorCode === 'invalid_grant' || errorCode === 'invalid_client') {
        // 清除缓存中的过期凭据，防止无限循环
        clearKeychainCache(configDir);
        return { success: false, error: ..., errorCode };
      }

      // 临时错误：继续重试
      continue;
    }

    // 成功：立即写入 Keychain（旧 token 已被撤销！）
    const data = await response.json();
    // ... 存储新 token
  }
}
```

`★ Insight ─────────────────────────────────────`
代码注释里有一个 CRITICAL 警告："当 token 刷新成功后，旧的 access token 和 refresh token 立即被 Anthropic 撤销。新 token 必须立即写入存储。"这意味着如果写 Keychain 失败，系统会处于一个尴尬状态：旧 token 已失效，新 token 在内存里但没持久化。代码用 `persistenceFailed` 字段标记这种情况，提示用户重新认证。
`─────────────────────────────────────────────────`

---

## 16.5　凭据存储：OS 密钥链

Auto-Claude 使用操作系统的原生密钥链存储敏感凭据，而不是加密文件：

| 平台 | 密钥链 | API |
|------|--------|-----|
| macOS | Keychain Access | Security Framework |
| Windows | Windows Credential Manager | WinCred API |
| Linux | Secret Service (GNOME Keyring / KWallet) | libsecret |

核心思想：密钥链由操作系统保护，不需要应用自己实现加密。即使磁盘被复制，密钥链数据也无法轻易提取（需要操作系统用户身份）。

Token 刷新时，`credential-utils.ts` 提供了缓存机制：

```typescript
// 内存缓存：避免每次都读 Keychain（可能需要用户授权弹窗）
const keychainCache = new Map<string, FullCredentials>();

export async function getFullCredentialsFromKeychain(
  configDir: string
): Promise<FullCredentials | null> {
  // 先查缓存
  const cached = keychainCache.get(configDir);
  if (cached) return cached;

  // 从 OS Keychain 读取
  const credentials = await readFromKeychain(configDir);
  if (credentials) {
    keychainCache.set(configDir, credentials);
  }
  return credentials;
}

// Token 刷新后清除缓存，确保下次读到新 token
export function clearKeychainCache(configDir?: string): void {
  if (configDir) {
    keychainCache.delete(configDir);
  } else {
    keychainCache.clear(); // 错误恢复时清空所有缓存
  }
}
```

---

## 16.6　主动切换检测

除了被动的（任务触发 rate limit 才切换），Profile Scorer 还支持主动切换——在当前账户用量到达阈值前，提前把新任务路由到其他账户：

```typescript
export function shouldProactivelySwitch(
  currentAccount: UnifiedAccount,
  settings: ClaudeAutoSwitchSettings
): boolean {
  // API 账户无限制，不需要切换
  if (currentAccount.type === 'api') return false;

  // 已经 rate limited，应该立即切换（被动切换处理）
  if (currentAccount.isRateLimited) return false;

  // 用量超过阈值：主动切换
  const weeklyClose = (currentAccount.weeklyPercent ?? 0) >= settings.weeklyThreshold;
  const sessionClose = (currentAccount.sessionPercent ?? 0) >= settings.sessionThreshold;

  return weeklyClose || sessionClose;
}
```

主动切换的价值在于：不等到任务失败才换账户，而是在账户还能用时就开始把新任务分配到其他账户，平滑地过渡。

---

## 16.7　架构总览：凭据流图

```
用户操作（UI 配置账户优先级）
     │
     ▼
claude-profile-store.ts（渲染进程 Zustand）
     │ IPC
     ▼
ClaudeProfileManager（主进程单例）
     │
     ├── credential-utils.ts
     │      └── OS Keychain（持久化）
     │
     ├── profile-scorer.ts
     │      └── getBestAvailableUnifiedAccount()
     │
     ├── token-refresh.ts
     │      └── ANTHROPIC_TOKEN_ENDPOINT（OAuth 刷新）
     │
     └── rate-limit-manager.ts
            └── isProfileRateLimited()

任务启动时
     │
     ▼
getBestAvailableUnifiedAccount()
     │ 返回最优账户
     ▼
AgentState.assignProfileToTask()
     │ 记录 taskId → profileId 映射
     ▼
env 变量注入（ANTHROPIC_API_KEY 或 OAuth credentials）
     │
     ▼
Python subprocess（使用对应账户的凭据）
```

---

## 16.8　Lab 16.1：实现账户选择器

**目标**：实现一个简化版的账户选择器，包含优先级排序、可用性过滤和降级策略。

### Part A：账户模型与评分

```typescript
// lab16/account-selector.ts

export interface Account {
  id: string;
  name: string;
  isAuthenticated: boolean;
  isRateLimited: boolean;
  weeklyUsagePercent: number;  // 0-100
  sessionUsagePercent: number; // 0-100
}

export interface SelectionSettings {
  weeklyThreshold: number;    // 默认 99
  sessionThreshold: number;   // 默认 95
}

/**
 * 检查账户是否可用
 * 返回 { available: true } 或 { available: false, reason: string }
 */
export function checkAvailability(
  account: Account,
  settings: SelectionSettings
): { available: boolean; reason?: string } {
  // TODO Part A: 实现三重门检查
  // 1. 未认证 → 不可用
  // 2. Rate limited → 不可用
  // 3. 用量 >= 阈值 → 不可用（注意是 >=，不是 >）
  return { available: false };
}

/**
 * 降级评分（所有账户都不可用时用）
 * 分数越高越好
 */
export function calculateFallbackScore(account: Account): number {
  // TODO Part A:
  // - 从 100 开始
  // - 未认证：-1000
  // - Rate limited：-200（会话级）
  // - 用量越高，扣分越多
  return 0;
}
```

### Part B：优先级排序与账户选择

```typescript
// lab16/account-selector.ts（续）

/**
 * 从账户列表中选择最优账户
 *
 * @param accounts - 所有账户列表
 * @param priorityOrder - 用户配置的优先级顺序（account ID 列表）
 * @param settings - 用量阈值设置
 * @param excludeId - 要排除的账户 ID（通常是当前失败的账户）
 * @returns 最优账户，如果都不可用则返回"最不坏"的
 */
export function selectBestAccount(
  accounts: Account[],
  priorityOrder: string[],
  settings: SelectionSettings,
  excludeId?: string
): Account | null {
  // TODO Part B:
  // 1. 过滤掉 excludeId
  // 2. 按 priorityOrder 排序（不在列表里的排最后）
  // 3. 找第一个通过 checkAvailability 的账户
  // 4. 如果没有：用 calculateFallbackScore 找最高分的作为降级选项
  return null;
}

// 验证：
const accounts: Account[] = [
  { id: 'a', name: 'Account A', isAuthenticated: true, isRateLimited: true,  weeklyUsagePercent: 50, sessionUsagePercent: 30 },
  { id: 'b', name: 'Account B', isAuthenticated: true, isRateLimited: false, weeklyUsagePercent: 20, sessionUsagePercent: 10 },
  { id: 'c', name: 'Account C', isAuthenticated: true, isRateLimited: false, weeklyUsagePercent: 96, sessionUsagePercent: 80 },
];
const settings: SelectionSettings = { weeklyThreshold: 95, sessionThreshold: 90 };
const priority = ['c', 'b', 'a']; // 用户配置 C 优先，但 C 超阈值

const result = selectBestAccount(accounts, priority, settings);
// 应选 B（C 超阈值不可用，A rate limited，B 可用且是候选中优先级最高的可用账户）
console.log(result?.id); // → 'b'
```

### Part C：Token 过期检测

```typescript
// lab16/token-utils.ts

const PROACTIVE_THRESHOLD_MS = 30 * 60 * 1000; // 30 分钟

/**
 * 检查 token 是否已过期或即将过期
 * @param expiresAt - 过期时间戳（ms），null 表示未知
 * @param thresholdMs - 提前刷新的时间窗口（默认 30 分钟）
 */
export function isTokenNearExpiry(
  expiresAt: number | null,
  thresholdMs: number = PROACTIVE_THRESHOLD_MS
): boolean {
  // TODO Part C: 实现检测逻辑
  // - null → 保守假设过期（返回 true）
  // - 距过期 <= thresholdMs → 返回 true
  return true;
}

/**
 * 格式化剩余时间，用于日志
 */
export function formatTimeRemaining(expiresAt: number | null): string {
  // TODO Part C:
  // - null → 'unknown'
  // - 已过期 → 'expired'
  // - >= 1小时 → '2h 30m'
  // - < 1小时 → '45m'
  return 'unknown';
}

// 验证：
const future30min = Date.now() + 30 * 60 * 1000;
const future1h = Date.now() + 60 * 60 * 1000;
const past = Date.now() - 1000;

console.log(isTokenNearExpiry(null));        // → true
console.log(isTokenNearExpiry(future30min)); // → true（刚好在阈值边界）
console.log(isTokenNearExpiry(future1h));    // → false
console.log(isTokenNearExpiry(past));        // → true（已过期）

console.log(formatTimeRemaining(future1h));  // → '1h 0m'（或类似）
console.log(formatTimeRemaining(null));      // → 'unknown'
```

**验证标准**：
- Part A：`checkAvailability` 的三重门顺序正确；阈值使用 `>=`
- Part B：`selectBestAccount` 在有可用账户时返回优先级最高的；全不可用时返回降级选项
- Part C：`null` 时返回 `true`（保守策略）；边界值处理正确

**进阶挑战**：
- 实现 `proactiveSwitchNeeded(currentAccount, settings)`：在用量 >= 阈值时提前标记需要切换
- 给 `selectBestAccount` 添加权重参数：某些账户可以设置倍增权重（比如企业账户）
- 实现简单的内存缓存层：`getCachedCredentials(accountId)` + `invalidateCache(accountId)`

---

## 本章要点回顾

| 概念 | 核心设计 | 源码位置 |
|------|---------|---------|
| UnifiedAccount | OAuth + API 账户统一抽象 | `shared/types/unified-account.ts` |
| 三重门过滤 | 认证 → Rate Limit → 阈值 | `profile-scorer.ts:43` |
| `>=` 阈值判断 | 主动在硬限额前切换 | `profile-scorer.ts:66` |
| 降级评分 | 最不坏选项，不直接报错 | `profile-scorer.ts:90` |
| 30分钟主动刷新 | Token 生命周期的缓冲区 | `token-refresh.ts:44` |
| null → 假设过期 | 保守策略，宁多刷新不失败 | `token-refresh.ts:116` |
| invalid_grant 不重试 | 永久错误分类，避免无限循环 | `token-refresh.ts:230` |
| clearKeychainCache | Token 刷新后清除旧缓存 | `credential-utils.ts` |
| OS Keychain 存储 | 系统级安全，无需自加密 | `credential-utils.ts` |
