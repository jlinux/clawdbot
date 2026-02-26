# OpenClaw Tool 错误处理和重试机制

## 概述

OpenClaw 的 Tool 错误处理是一个多层系统：**执行层捕获** → **错误分类** → **反馈给 AI** → **AI 自我纠正**。错误通过 `isError: true` 标志注入对话历史，AI 模型读取错误后自行调整策略。

---

## 整体错误处理流程

```
Agent Loop (runEmbeddedPiAgent)
    │
    ▼
构建 Prompt + Session 历史（含之前的错误）
    │
    ▼
API 调用（Anthropic/OpenAI/etc）
    │
    ├── Tool Calls → Tool 执行循环
    │       │
    │       ▼
    │   Tool 执行开始 (handleToolExecutionStart)
    │       │
    │       ▼
    │   执行 Tool (bash/read/etc)
    │       │
    │       ▼
    │   Tool 执行结束 (handleToolExecutionEnd)
    │       │
    │       ├── 成功 → toolResult 写入历史
    │       │
    │       └── 失败 → isError 检测
    │               │
    │               ├── extractToolErrorMessage (截断到 400 字符)
    │               ├── 错误分类 (overflow/rate_limit/auth/timeout/billing)
    │               └── toolResult{isError:true} 写入历史
    │
    └── API 错误 → 错误处理
            │
            ├── Context Overflow → 触发 Compaction → 重试
            ├── Rate Limit → 轮换 Auth Profile → 重试
            ├── Timeout → 重试
            └── Billing/Auth → 终止
```

---

## 1. Agent Loop 中的 Tool 错误捕获

### 关键文件

`src/agents/pi-embedded-subscribe.handlers.tools.ts`（行 293-380）

### 三个执行事件处理器

| 处理器                      | 职责                                                  |
| --------------------------- | ----------------------------------------------------- |
| `handleToolExecutionStart`  | 跟踪 tool 调用，记录元数据到 `ctx.state.toolMetaById` |
| `handleToolExecutionUpdate` | 处理流式中间结果，发出 `tool_update` 事件             |
| `handleToolExecutionEnd`    | **核心错误检测**，分类并记录错误                      |

### 错误检测双重机制

```typescript
// pi-embedded-subscribe.handlers.tools.ts:304-306
const isError = Boolean(evt.isError); // ① 显式 isError 标志
const result = evt.result;
const isToolError = isError || isToolResultError(result); // ② 隐式结果检测
```

**两种检测路径**：

1. **显式 `isError` 标志**：pi-agent-core 框架在 tool 执行抛异常时设置
2. **隐式结果检测**：`isToolResultError()` 检查 result 结构中的 status 字段

### isToolResultError 函数

```typescript
// pi-embedded-subscribe.tools.ts:243-258
export function isToolResultError(result: unknown): boolean {
  if (!result || typeof result !== "object") return false;
  const details = (result as { details?: unknown }).details;
  if (!details || typeof details !== "object") return false;
  const status = (details as { status?: unknown }).status;
  if (typeof status !== "string") return false;
  const normalized = status.trim().toLowerCase();
  return normalized === "error" || normalized === "timeout";
}
```

### 错误状态追踪

当 `isToolError` 为 true 时（行 315-339），错误上下文存入 `ctx.state.lastToolError`：

```typescript
{
  toolName: string;           // 工具名
  meta?: string;              // 附加元数据
  error: string | undefined;  // 错误消息（最多 400 字符）
  mutatingAction?: boolean;   // 是否为变更操作
  actionFingerprint?: string; // 操作指纹（用于去重）
}
```

---

## 2. 错误消息提取

### extractToolErrorMessage

`src/agents/pi-embedded-subscribe.tools.ts:260-287`

**提取优先级**：

1. `result.details.error` / `result.details.message` / `result.details.reason`
2. 根级 error 字段
3. 尝试 JSON 解析 text 内容
4. 规范化为首行文本

**截断规则**：

- 最大长度：400 字符
- 仅保留第一行（去除换行符）

### 错误状态模式匹配

```typescript
// 行 33-48
// 匹配: "error", "fail", "timeout", "denied", "invalid", "forbidden"
```

---

## 3. AJV 参数验证

### 关键文件

`src/plugins/schema-validator.ts`

### 配置

```typescript
// schema-validator.ts:3-7
const ajv = new Ajv({
  allErrors: true, // 收集所有验证错误
  strict: false, // 宽松模式
  removeAdditional: false, // 保留额外属性
});
```

### 验证函数

```typescript
export function validateJsonSchemaValue(params: {
  schema: Record<string, unknown>;
  cacheKey: string;
  value: unknown;
}): { ok: true } | { ok: false; errors: string[] };
```

**错误格式化**（`formatAjvErrors`，行 16-25）：

- 转换 AJV 错误对象为可读消息
- 格式：`"args.timeout: must be number"`
- JSON pointer 路径转点号表示法

**缓存策略**：

- Schema 编译器按 `cacheKey` 缓存
- 避免重复编译相同 schema
- 检测 schema 变化并重新编译

---

## 4. retryAsync 重试工具

### 关键文件

`src/infra/retry.ts`（行 70-136）

### 简单模式

```typescript
await retryAsync(fn, 3, 300);
// 3 次尝试，初始延迟 300ms
// 指数退避：300ms → 600ms → 1200ms
```

### 高级模式

```typescript
await retryAsync(fn, {
  attempts: 3,
  minDelayMs: 300,
  maxDelayMs: 30_000,
  jitter: 0.1, // ±10% 随机抖动
  shouldRetry: (err, attempt) => isTransient(err),
  retryAfterMs: (err) => parseRetryAfter(err),
  onRetry: (err, attempt) => log.warn(`Retry ${attempt}`),
});
```

### 退避公式

```
delay = min(baseDelay × 2^attempt, maxDelayMs)
finalDelay = delay × (1 + jitter × random(-1, 1))
clamp(finalDelay, minDelayMs, maxDelayMs)
```

### Jitter 防雷暴群

抖动防止多客户端同时重试：

- jitter=0.1 时，300ms 变为 270-330ms 随机值
- 分散重试请求到不同时间点

### 使用场景

| 场景               | 文件                       |
| ------------------ | -------------------------- |
| Compaction 摘要    | `src/agents/compaction.ts` |
| Discord API 调用   | `src/discord/api.ts`       |
| 向量数据库批量操作 | `src/memory/batch-http.ts` |

---

## 5. 超时处理

### Run 级超时

`src/agents/pi-embedded-runner/run/attempt.ts:971-1002`

- `params.timeoutMs` 配置整个 run 的最大执行时间
- 超时触发 `abortRun(true)` + `AbortSignal`

### Tool 级超时

| 工具       | 超时方式                                        | 文件                    |
| ---------- | ----------------------------------------------- | ----------------------- |
| web_search | `withTimeout(undefined, timeoutSeconds * 1000)` | `tools/web-search.ts`   |
| browser    | `timeoutMs` 参数                                | `tools/browser-tool.ts` |
| 模型 API   | `AbortSignal.timeout(5000)`                     | 模型发现                |

### 超时检测模式

```typescript
// pi-embedded-helpers/errors.ts:634-643
// 检测: "timeout", "timed out", "deadline exceeded",
//       "without sending chunks", "stop reason: abort"
```

---

## 6. 错误类型分类与处理策略

### 关键文件

`src/agents/pi-embedded-helpers/errors.ts`（450+ 行）

### 错误类型一览

| 类型                 | 检测模式                                      | 可重试 | 处理策略               |
| -------------------- | --------------------------------------------- | ------ | ---------------------- |
| **Context Overflow** | "context window", "prompt too long", HTTP 413 | 是     | 触发 Compaction → 重试 |
| **Rate Limit**       | "rate limit", "429", "quota exceeded"         | 是     | Auth Profile 轮换      |
| **Billing**          | HTTP 402, "insufficient credits"              | 否     | 终止，提示充值         |
| **Auth**             | "invalid api key", HTTP 401/403               | 是     | Profile Failover       |
| **Timeout**          | "timeout", "timed out", HTTP 502/503/504      | 是     | 退避重试               |
| **Format**           | "tool_use.id", "string should match pattern"  | 是     | 清理 ID 格式后重试     |
| **图片尺寸**         | "image dimensions exceed"                     | 否     | 提示缩放               |
| **Cloudflare**       | HTTP 521-526, 530                             | 是     | 退避重试               |

### Context Overflow 处理

检测模式（`errors.ts:50-131`）包含中英文：

- `"context window"`, `"context length exceeded"`, `"prompt too long"`
- `"上下文过长"`, `"上下文超出"`

**处理流程**（`run.ts:689-851`）：

1. 检测到 overflow → 触发 compaction
2. Compaction 压缩上下文 → 重试相同 prompt
3. 用户消息："Context overflow: prompt too large for the model. Try /reset or use larger-context model."

---

## 7. AI 模型如何接收错误反馈并自我纠正

### 错误反馈注入机制

错误通过 **ToolResult 消息**自动注入对话历史：

```
消息历史:
  [user]: "列出 /nonexistent 的文件"
  [assistant]: tool_use[exec] {"command": "ls /nonexistent"}
  [tool_result]: {isError: true, text: "No such file or directory"}  ← 错误注入
  [assistant]: "目录不存在，让我检查可用目录..."                      ← AI 自我纠正
  [assistant]: tool_use[exec] {"command": "ls /"}
  [tool_result]: {isError: false, text: "bin boot dev home ..."}
```

### 自我纠正机制的三个支柱

1. **错误消息可见性**：
   - `isError: true` 标志告诉模型"这没有成功"
   - 错误文本（最多 400 字符）提供具体原因
   - 模型训练时学会解读这些信号

2. **完整历史上下文**：
   - Session 保持完整对话历史
   - 模型可以看到之前所有失败尝试
   - 可以识别错误模式并调整策略

3. **Assistant 错误格式化**：
   - `formatAssistantErrorText()`（`run.ts:658-669`）将错误格式化为模型可读文本
   - 包含 provider、model、session 上下文信息

### Run 级重试循环

```typescript
// run.ts:520-567
while (runLoopIterations < MAX_RUN_LOOP_ITERATIONS) {
  const attempt = await runEmbeddedAttempt({
    prompt,
    // Session 包含之前的错误
  });

  if (contextOverflowError) {
    // 触发 compaction，重试相同 prompt
  }
  if (failoverError) {
    // 切换 provider，重试相同 prompt
  }
  // 带更新后的 session 继续循环
}
```

---

## 8. 关键文件索引

| 文件                                      | 行       | 职责                                                |
| ----------------------------------------- | -------- | --------------------------------------------------- |
| `pi-embedded-subscribe.handlers.tools.ts` | 293-380  | `handleToolExecutionEnd()` — 错误捕获               |
| `pi-embedded-subscribe.tools.ts`          | 243-287  | `isToolResultError()` + `extractToolErrorMessage()` |
| `infra/retry.ts`                          | 70-136   | `retryAsync()` — 指数退避重试                       |
| `pi-embedded-helpers/errors.ts`           | 450+     | 错误分类 + 用户消息                                 |
| `pi-embedded-runner/run.ts`               | 636-900  | Session 历史管理 + 重试决策                         |
| `pi-embedded-runner/run/attempt.ts`       | 971-1002 | 超时 + AbortSignal                                  |
| `plugins/schema-validator.ts`             | 27-44    | AJV 参数验证                                        |

---

## 9. 对最小可迁移版本的启示

### 必须实现

```typescript
// 1. Tool 执行包裹 try-catch
async function executeTool(tool, args) {
  try {
    const result = await tool.execute(args);
    return { content: result, isError: false };
  } catch (error) {
    return { content: error.message.slice(0, 400), isError: true };
  }
}

// 2. 错误结果注入对话历史
messages.push({
  role: "tool",
  tool_use_id: toolCallId,
  content: result.content,
  is_error: result.isError, // AI API 原生支持
});
// AI 模型会自动读取 is_error 并调整策略

// 3. 简单重试
async function retryWithBackoff(fn, maxAttempts = 3) {
  for (let i = 0; i < maxAttempts; i++) {
    try {
      return await fn();
    } catch (e) {
      if (i === maxAttempts - 1) throw e;
      await sleep(300 * 2 ** i);
    }
  }
}
```

### 可以简化的地方

| OpenClaw 做法                 | 最小版本                 |
| ----------------------------- | ------------------------ |
| 8 种错误分类 + 专门处理策略   | 只分"可重试"和"不可重试" |
| AJV schema 验证 + 缓存        | 不验证（让 AI 自己修正） |
| 400 字符截断 + 多层提取       | 直接 `error.message`     |
| Run 级 + Tool 级超时          | 只用 AbortSignal         |
| Context Overflow → Compaction | 提示用户 /reset          |
| Auth Profile 轮换 + 冷却      | 不需要                   |
