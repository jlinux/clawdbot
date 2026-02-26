# OpenClaw Model Failover（模型故障切换）机制

## 概述

OpenClaw 的 Model Failover 是一个三层故障切换系统：**尝试级重试** → **Auth Profile 轮换** → **模型回退链**。当主模型不可用时，系统自动切换到备用模型，结合断路器（Circuit Breaker）模式避免反复请求已知故障的服务。

---

## 整体流程图

```
runEmbeddedPiAgent()
    │
    ├── resolveModel() → 主 provider/model
    ├── ensureAuthProfileStore() → 加载 auth profiles
    │
    └── FOR 每个 auth profile:
        │
        ├── isProfileInCooldown()? → YES → SKIP
        │
        └── FOR retry 迭代 (最多 MAX_RUN_RETRY_ITERATIONS):
            │
            └── runEmbeddedAttempt(provider, model, profileId)
                │
                ├── 成功 → markAuthProfileGood() → 返回结果
                │
                └── 失败
                    │
                    ├── resolveFailoverReasonFromError()
                    │   ├── context overflow → 直接 rethrow
                    │   ├── user abort → 直接 rethrow
                    │   └── 其他 → 分类（rate_limit/auth/billing/timeout...）
                    │
                    ├── markAuthProfileFailure(reason)
                    │   └── 计算 cooldown → 设置 cooldownUntil
                    │
                    └── 尝试下一个 profile...
                        │
                        └── 所有 profile 失败 → 进入模型回退链
                            │
                            └── runWithModelFallback()
                                │
                                ├── resolveFallbackCandidates()
                                │   └── [primary, fallback1, fallback2, default]
                                │
                                └── FOR 每个候选模型:
                                    ├── 成功 → 返回
                                    └── 失败 → 下一个候选
                                        └── 全部失败 → throwFallbackFailureSummary()
```

---

## 1. 回退链层级结构

### 关键文件

- `src/agents/model-fallback.ts` — 主编排逻辑
- `src/agents/defaults.ts` — 默认模型
- `src/config/zod-schema.agent-model.ts` — 配置 schema

### 模型链结构

```
主模型 (provider/model)
  ↓ 失败
配置的回退模型 #1
  ↓ 失败
配置的回退模型 #2
  ↓ 失败
默认模型 (anthropic/claude-opus-4-6)
```

### 配置 Schema

```typescript
// src/config/zod-schema.agent-model.ts:3-11
export const AgentModelSchema = z.union([
  z.string(), // 简单字符串："anthropic/claude-opus-4-6"
  z.object({
    primary: z.string().optional(), // 主模型
    fallbacks: z.array(z.string()).optional(), // 回退列表
  }),
]);
```

### 解析逻辑

`resolveFallbackCandidates()`（`model-fallback.ts:188-263`）：

- 规范化主模型为 `provider/model` 格式
- 保持韧性：当当前模型已经是回退列表中的某个时，继续遍历链
- 最终回退到 `DEFAULT_PROVIDER="anthropic"`、`DEFAULT_MODEL="claude-opus-4-6"`

### 候选去重

`createModelCandidateCollector()`（`model-fallback.ts:64-95`）：

- 跟踪已尝试的 provider/model 对
- 去重 + allowlist 强制过滤

---

## 2. 重试逻辑

### 通用重试框架

```typescript
// src/infra/retry.ts:3-8
export type RetryConfig = {
  attempts?: number; // 默认: 3
  minDelayMs?: number; // 默认: 300ms
  maxDelayMs?: number; // 默认: 30,000ms (30s)
  jitter?: number; // 默认: 0
};
```

### 指数退避公式

`src/infra/retry.ts:62-68`：

```
delay = min(baseDelay × 2^(attempt-1), maxDelayMs)
finalDelay = baseDelay + jitter × baseDelay × random()
最终: clamp(delay, minDelayMs, maxDelayMs)
```

### 最大重试迭代数

`src/agents/pi-embedded-runner/run.ts:111-121`：

```typescript
const BASE_RUN_RETRY_ITERATIONS = 24;
const RUN_RETRY_ITERATIONS_PER_PROFILE = 8;
const MIN_RUN_RETRY_ITERATIONS = 32;
const MAX_RUN_RETRY_ITERATIONS = 160;

function resolveMaxRunRetryIterations(profileCandidateCount: number): number {
  const scaled =
    BASE_RUN_RETRY_ITERATIONS +
    Math.max(1, profileCandidateCount) * RUN_RETRY_ITERATIONS_PER_PROFILE;
  return Math.min(MAX_RUN_RETRY_ITERATIONS, Math.max(MIN_RUN_RETRY_ITERATIONS, scaled));
}
```

| 场景          | 计算            | 最大迭代    |
| ------------- | --------------- | ----------- |
| 1 个 profile  | 24 + 1×8 = 32   | 32          |
| 3 个 profile  | 24 + 3×8 = 48   | 48          |
| 20 个 profile | 24 + 20×8 = 184 | 160（封顶） |

---

## 3. Auth Profile 轮换

### 关键文件

- `src/agents/auth-profiles/usage.ts` — cooldown/轮换逻辑
- `src/agents/api-key-rotation.ts` — API key 循环
- `src/agents/auth-profiles/types.ts` — 数据结构

### Profile 存储结构

```typescript
// src/agents/auth-profiles/types.ts:44-67
export type ProfileUsageStats = {
  lastUsed?: number;
  cooldownUntil?: number; // 冷却到期时间戳
  disabledUntil?: number; // 永久禁用到期时间戳
  disabledReason?: AuthProfileFailureReason;
  errorCount?: number;
  failureCounts?: Partial<Record<AuthProfileFailureReason, number>>;
  lastFailureAt?: number;
};

export type AuthProfileStore = {
  version: number;
  profiles: Record<string, AuthProfileCredential>;
  order?: Record<string, string[]>; // 每个 agent 的 profile 顺序覆盖
  lastGood?: Record<string, string>; // 轮换追踪
  usageStats?: Record<string, ProfileUsageStats>;
};
```

### API Key 轮换

`executeWithApiKeyRotation()`（`api-key-rotation.ts:40-72`）：

```typescript
export async function executeWithApiKeyRotation<T>(params) {
  const keys = dedupeApiKeys(params.apiKeys);
  for (let attempt = 0; attempt < keys.length; attempt++) {
    try {
      return await params.execute(keys[attempt]);
    } catch (error) {
      const retryable = params.shouldRetry?.(...) ?? isApiKeyRateLimitError(message);
      if (!retryable || attempt + 1 >= keys.length) break;
      params.onRetry?.(...);
    }
  }
  throw lastError;
}
```

**Key 来源**（`src/agents/live-auth-keys.ts`）：

- `{PROVIDER}_API_KEYS`（逗号/分号分隔）
- `{PROVIDER}_API_KEY`（主 key）
- `{PROVIDER}_API_KEY_1`, `{PROVIDER}_API_KEY_2` 等（前缀编号）
- 去重后按顺序轮换

### Profile 选择顺序

`model-fallback.ts:333-379`：

1. `resolveAuthProfileOrder()` 获取有序 profile 列表
2. `isProfileInCooldown()` 检查 → 冷却中则跳过
3. 主模型冷却中时，探测恢复（30s 节流，2 分钟窗口内）

---

## 4. 断路器（Circuit Breaker）/ 冷却机制

### 普通冷却（rate_limit, timeout, format, model_not_found, unknown）

`src/agents/auth-profiles/usage.ts:277-283`：

```typescript
export function calculateAuthProfileCooldownMs(errorCount: number): number {
  const normalized = Math.max(1, errorCount);
  return Math.min(
    60 * 60 * 1000, // 最大 1 小时
    60 * 1000 * 5 ** Math.min(normalized - 1, 3),
  );
}
```

| 连续错误次数 | 冷却时间        |
| ------------ | --------------- |
| 1            | 60 秒           |
| 2            | 5 分钟          |
| 3            | 25 分钟         |
| 4+           | 60 分钟（封顶） |

### 计费冷却（billing 错误）

`src/agents/auth-profiles/usage.ts:335-346`：

```typescript
function calculateAuthProfileBillingDisableMsWithConfig(params) {
  const exponent = Math.min(normalized - 1, 10);
  const raw = baseMs * 2 ** exponent;
  return Math.min(maxMs, raw);
}
```

默认：`baseMs = 5小时`，`maxMs = 24小时`，指数增长 `5h × 2^(n-1)`。

### 状态转换

`computeNextProfileUsageStats()`（`usage.ts:373-424`）：

- 失败在 failureWindow 内 → errorCount 递增
- 失败在 failureWindow 外 → errorCount 重置为 0
- 活跃窗口不会在重试时延长（防止"卡住"）

### 过期与重置

`clearExpiredCooldowns()`（`usage.ts:174-225`）：

- 惰性清理：下次使用更新时检查
- 所有冷却过期时清除 errorCount 和 failureCounts（半开 → 关闭）

### 冷却期间的探测

`shouldProbePrimaryDuringCooldown()`（`model-fallback.ts:265-299`）：

- **节流**：每个 provider 至少 30 秒间隔
- **窗口**：冷却到期前 2 分钟内可探测
- 如果最近到期时间在窗口内，尝试主模型恢复

---

## 5. 错误分类

### FailoverReason 类型

```typescript
// src/agents/pi-embedded-helpers/types.ts:3-10
export type FailoverReason =
  | "auth" // 401/403, invalid API key
  | "format" // 400, 验证错误
  | "rate_limit" // 429, 配额超限
  | "billing" // 402, 余额不足
  | "timeout" // 408, 502/503/504, socket timeout
  | "model_not_found" // 404, unknown model
  | "unknown";
```

### HTTP 状态码映射

| 状态码            | 分类              |
| ----------------- | ----------------- |
| 401/403           | `auth`            |
| 402               | `billing`         |
| 404 + "not found" | `model_not_found` |
| 408               | `timeout`         |
| 429               | `rate_limit`      |
| 502/503/504       | `timeout`         |
| 400               | `format`          |

### 错误消息模式匹配

| 分类         | 关键词                                                                                  |
| ------------ | --------------------------------------------------------------------------------------- |
| `rate_limit` | "rate limit", "too many requests", "429", "quota exceeded", "resource exhausted", "tpm" |
| `timeout`    | "timeout", "timed out", "deadline exceeded", "abort"                                    |
| `billing`    | "402", "payment required", "insufficient credit", "credit balance"                      |
| `auth`       | "invalid api key", "unauthorized", "forbidden", "expired", "oauth token refresh failed" |
| `format`     | "tool_use.id", "tool_call id", "string should match pattern"                            |

### 不触发 Failover 的错误

| 错误类型                | 处理方式                            |
| ----------------------- | ----------------------------------- |
| Context overflow        | 直接 rethrow → 内部 compaction 处理 |
| User abort (AbortError) | 直接 rethrow                        |
| 最后一个候选的未知错误  | 直接 rethrow                        |

---

## 6. 配置示例

### 模型配置

```yaml
agents:
  defaults:
    model:
      primary: "anthropic/claude-opus-4-6"
      fallbacks:
        - "openai/gpt-4-turbo"
        - "google/gemini-2.0-flash"
    imageModel:
      primary: "anthropic/claude-opus-4-6"
      fallbacks:
        - "openai/gpt-4o"

  # 单个 Agent 覆盖
  list:
    - id: "my-agent"
      model:
        primary: "openai/gpt-4-turbo"
        fallbacks: ["anthropic/claude-opus-4-6"]
```

### Auth & 冷却配置

```yaml
auth:
  profiles:
    "anthropic:default":
      provider: "anthropic"
      mode: "api_key"
    "openai:primary":
      provider: "openai"
      mode: "api_key"

  order:
    "anthropic": ["anthropic:default"]
    "openai": ["openai:primary", "openai:secondary"]

  cooldowns:
    billingBackoffHours: 5 # 计费初始退避
    billingBackoffHoursByProvider:
      "anthropic": 4
    billingMaxHours: 24 # 计费最大退避
    failureWindowHours: 24 # 错误计数重置窗口
```

---

## 7. 关键常量汇总

| 常量                               | 值                  | 文件                    | 用途              |
| ---------------------------------- | ------------------- | ----------------------- | ----------------- |
| `DEFAULT_PROVIDER`                 | `"anthropic"`       | `defaults.ts`           | 默认 provider     |
| `DEFAULT_MODEL`                    | `"claude-opus-4-6"` | `defaults.ts`           | 默认模型          |
| `BASE_RUN_RETRY_ITERATIONS`        | 24                  | `run.ts:111`            | 基础重试上限      |
| `RUN_RETRY_ITERATIONS_PER_PROFILE` | 8                   | `run.ts:112`            | 每 profile 迭代数 |
| `MIN_RUN_RETRY_ITERATIONS`         | 32                  | `run.ts:113`            | 最小总迭代        |
| `MAX_RUN_RETRY_ITERATIONS`         | 160                 | `run.ts:114`            | 最大总迭代        |
| `MIN_PROBE_INTERVAL_MS`            | 30,000              | `model-fallback.ts:266` | 探测节流          |
| `PROBE_MARGIN_MS`                  | 120,000             | `model-fallback.ts:267` | 探测窗口 (2min)   |
| `DEFAULT_RETRY_ATTEMPTS`           | 3                   | `retry.ts:26`           | 重试次数          |
| `DEFAULT_MIN_DELAY_MS`             | 300                 | `retry.ts:27`           | 最小退避          |
| `DEFAULT_MAX_DELAY_MS`             | 30,000              | `retry.ts:28`           | 最大退避 (30s)    |
| `COOLDOWN_MAX_MS`                  | 3,600,000           | `usage.ts:280`          | 最大冷却 (1h)     |
| `BILLING_BACKOFF_DEFAULT_HOURS`    | 5                   | `usage.ts:296`          | 计费默认退避      |
| `BILLING_MAX_DEFAULT_HOURS`        | 24                  | `usage.ts:297`          | 计费最大退避      |
| `FAILURE_WINDOW_DEFAULT_HOURS`     | 24                  | `usage.ts:298`          | 错误窗口          |

---

## 8. 对最小可迁移版本的启示

### 必须实现

```typescript
// 1. 简单的模型回退链
const FALLBACK_MODELS = ["anthropic/claude-opus-4-6", "openai/gpt-4-turbo"];

async function callWithFallback(messages) {
  for (const model of FALLBACK_MODELS) {
    try {
      return await callModel(model, messages);
    } catch (error) {
      if (isContextOverflow(error)) throw error; // 不回退
      console.warn(`${model} failed, trying next...`);
    }
  }
  throw new Error("All models failed");
}

// 2. 简单的指数退避重试
async function retryWithBackoff(fn, maxAttempts = 3) {
  for (let i = 0; i < maxAttempts; i++) {
    try {
      return await fn();
    } catch (e) {
      await sleep(300 * 2 ** i);
    }
  }
}
```

### 可以简化的地方

| OpenClaw 做法            | 最小版本                   |
| ------------------------ | -------------------------- |
| Auth Profile 轮换 + 冷却 | 只用一个 API key           |
| 断路器 + 探测恢复        | 简单超时后重试             |
| 错误分类（7 种 reason）  | 只区分"可重试"vs"不可重试" |
| 计费冷却（指数小时级）   | 不需要                     |
| 动态迭代数（32-160）     | 固定 3 次重试              |
| 多 provider 多 profile   | 2-3 个模型的简单列表       |
