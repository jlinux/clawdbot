# 会话管理和上下文窗口压缩机制

## 核心结论

OpenClaw 的上下文管理是一个**四层防线**体系，从实时轻量修剪到离线重度摘要，层层递进：

```
第 1 层: History Limit      — 按 turn 数裁剪（最粗粒度，进入 agent loop 前）
第 2 层: Tool Result 截断   — 单条工具结果超大时截断（进入 agent loop 前）
第 3 层: Context Pruning     — 实时软修剪/硬清除旧 tool result（agent loop 中，每次 API 调用前）
第 4 层: Compaction          — 用 LLM 做摘要压缩（agent loop 外，上下文溢出时触发）
```

---

## 一、Session 的生命周期

### 1.1 存储格式

Session 以 **JSONL** 文件存储在磁盘上：

```
~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl
```

每行是一个 JSON 条目（entry），类型包括：

| type                        | 说明              |
| --------------------------- | ----------------- |
| `message`                   | 用户/AI/工具消息  |
| `compaction`                | 压缩摘要记录      |
| `thinking_level_change`     | thinking 级别切换 |
| `model_change`              | 模型切换记录      |
| `session_info`              | 会话元信息        |
| `custom` / `custom_message` | 插件自定义条目    |
| `branch_summary`            | 分支摘要          |
| `label`                     | 标签              |

### 1.2 生命周期流程

```
用户消息到达
  ↓
SessionManager.open(sessionFile)           ← 打开 JSONL 文件
  ↓
ensureSessionHeader()                      ← 确保文件有头部
  ↓
session.messages                           ← 加载所有消息到内存
  ↓
sanitizeSessionHistory()                   ← 清洗（修复配对、验证格式）
  ↓
limitHistoryTurns(messages, limit)         ← 按 turn 数裁剪
  ↓
sanitizeToolUseResultPairing(messages)     ← 修复裁剪后的孤儿 tool_result
  ↓
session.agent.replaceMessages(limited)     ← 注入到 agent 内存中
  ↓
session.prompt(userText)                   ← 开始 agent loop
  ↓
  ┌─── agent loop ────────────────┐
  │  tool_use → tool_result 循环  │
  │  每个消息自动追加到 JSONL     │
  │  Context Pruning 在每轮前运行 │
  └───────────────────────────────┘
  ↓
如果 context overflow
  ↓
触发 Compaction → 用 LLM 摘要 → 替换旧消息
  ↓
session.dispose()                          ← 释放资源
```

### 1.3 Session 的分支模型

SessionManager 底层采用**有向无环图（DAG）**模型：

- 每个 entry 有 `id` 和 `parentId`
- `branch(parentId)` 可以从任意节点创建新分支
- `getBranch()` 返回从根到当前叶子节点的线性路径
- Compaction 和 Tool Result 截断都通过 **branch + re-append** 实现（不修改已写入的 JSONL 行）

源码位置：`src/agents/pi-embedded-runner/tool-result-truncation.ts:198-208`

```typescript
// Branch from the parent of the first oversized entry
const branchFromId = firstOversizedEntry.parentId;
if (!branchFromId) {
  sessionManager.resetLeaf();
} else {
  sessionManager.branch(branchFromId); // ← 从某节点分叉
}
// Re-append all entries from that point with truncated tool results
```

这意味着 JSONL 文件是 **append-only** 的，旧数据不会被删除，只是通过分支指针跳过。

---

## 二、第 1 层：History Limit（Turn 数限制）

**目的**：防止长期运行的会话累积过多历史消息。

源码位置：`src/agents/pi-embedded-runner/history.ts:15-36`

```typescript
export function limitHistoryTurns(
  messages: AgentMessage[],
  limit: number | undefined,
): AgentMessage[] {
  if (!limit || limit <= 0 || messages.length === 0) {
    return messages;
  }
  let userCount = 0;
  let lastUserIndex = messages.length;
  for (let i = messages.length - 1; i >= 0; i--) {
    if (messages[i].role === "user") {
      userCount++;
      if (userCount > limit) {
        return messages.slice(lastUserIndex); // ← 只保留最近 N 个 user turn
      }
      lastUserIndex = i;
    }
  }
  return messages;
}
```

**配置层级**（优先级从高到低）：

```
channels[provider].dms[userId].historyLimit     ← 针对特定用户
channels[provider].dmHistoryLimit               ← 针对该渠道所有 DM
channels[provider].historyLimit                 ← 针对群聊/频道
```

**关键点**：这是 **agent loop 之前** 执行的，所以是最早的一道防线。裁剪后会运行 `sanitizeToolUseResultPairing()` 修复因裁剪产生的孤儿 tool_result。

---

## 三、第 2 层：Tool Result 截断

**目的**：防止单条工具结果（比如读取了一个巨大文件）占据过多上下文。

源码位置：`src/agents/pi-embedded-runner/tool-result-truncation.ts`

### 3.1 核心参数

| 参数                            | 值        | 说明                                 |
| ------------------------------- | --------- | ------------------------------------ |
| `MAX_TOOL_RESULT_CONTEXT_SHARE` | 0.3 (30%) | 单条 tool result 最多占上下文的 30%  |
| `HARD_MAX_TOOL_RESULT_CHARS`    | 400,000   | 绝对上限，即使 2M token 模型也不超过 |
| `MIN_KEEP_CHARS`                | 2,000     | 截断后至少保留的字符数               |

### 3.2 计算公式

```typescript
maxChars = Math.min(contextWindowTokens * 0.3 * 4, 400_000);
```

例如 200K token 模型：`Math.min(200000 * 0.3 * 4, 400000) = Math.min(240000, 400000) = 240,000 chars`

### 3.3 两种截断模式

**内存截断**（不修改 session 文件）：

```
truncateOversizedToolResultsInMessages(messages, contextWindowTokens)
→ 返回新数组，原消息不变
→ 用于发送给 LLM 之前的预处理
```

**Session 文件截断**（修改 JSONL）：

```
truncateOversizedToolResultsInSession({ sessionFile, contextWindowTokens })
→ 通过 branch + re-append 修改 JSONL
→ 在 context overflow 时作为第一手段尝试
```

### 3.4 截断策略

```typescript
// 尝试在换行处截断（避免切断一行的中间）
let cutPoint = keepChars;
const lastNewline = text.lastIndexOf("\n", keepChars);
if (lastNewline > keepChars * 0.8) {
  cutPoint = lastNewline;
}
return text.slice(0, cutPoint) + TRUNCATION_SUFFIX;
```

截断后追加提示信息，告诉 AI 内容被截断了，建议使用 offset/limit 读取。

---

## 四、第 3 层：Context Pruning（实时上下文修剪）

**目的**：在 agent loop 运行过程中，对过期的旧 tool result 进行渐进式清理。

源码位置：`src/agents/pi-extensions/context-pruning/`

### 4.1 默认配置

```typescript
DEFAULT_CONTEXT_PRUNING_SETTINGS = {
  mode: "cache-ttl",
  ttlMs: 5 * 60 * 1000, // 5 分钟 TTL
  keepLastAssistants: 3, // 保护最近 3 个 assistant 消息
  softTrimRatio: 0.3, // 上下文 > 30% 时开始软修剪
  hardClearRatio: 0.5, // 上下文 > 50% 时开始硬清除
  minPrunableToolChars: 50_000, // 至少 50K 可修剪字符才执行硬清除
  softTrim: {
    maxChars: 4_000, // tool result > 4K 才修剪
    headChars: 1_500, // 保留开头 1.5K
    tailChars: 1_500, // 保留结尾 1.5K
  },
  hardClear: {
    enabled: true,
    placeholder: "[Old tool result content cleared]",
  },
};
```

### 4.2 两阶段修剪

```
                 上下文占用 / 窗口大小
  ─────────┬──────────┬───────────┬──────────→
           0.3       0.5         1.0

  [不修剪]  │ 软修剪   │  硬清除   │ 溢出 → Compaction
```

**软修剪（Soft Trim）**：保留 tool result 的头尾，中间用 `...` 省略

```
原始: [10000 chars 的文件内容]
      ↓ 软修剪
修剪后: [前 1500 chars]
        ...
        [后 1500 chars]
        [Tool result trimmed: kept first 1500 chars and last 1500 chars of 10000 chars.]
```

**硬清除（Hard Clear）**：完全替换 tool result 内容

```
原始: [整个工具结果]
      ↓ 硬清除
清除后: "[Old tool result content cleared]"
```

### 4.3 保护机制

```
保护规则:
  ✅ 最近 3 个 assistant 消息之后的 tool result 不修剪
  ✅ 包含图片的 tool result 不修剪
  ✅ 第一个 user 消息之前的消息不修剪（保护 SOUL.md 等初始化读取）
  ✅ 硬清除需要累计 ≥ 50K 可修剪字符才触发
  ✅ 可配置 allow/deny 列表指定哪些 tool 可修剪
```

---

## 五、第 4 层：Compaction（LLM 摘要压缩）

**目的**：当前三层防线都不够时，用 LLM 对历史消息进行摘要，用一段摘要文本替换所有旧消息。

### 5.1 触发时机

```
触发方式:
  1. overflow — context overflow 错误触发（自动）
  2. manual  — 手动调用

自动触发流程:
  用户消息 → agent loop → API 返回 context overflow 错误
    ↓
  isLikelyContextOverflowError() 检测
    ↓
  先尝试: sessionLikelyHasOversizedToolResults() → 截断 tool result
    ↓
  如果不够: 触发完整 Compaction
    ↓
  重试 agent loop（最多 maxAttempts 次）
```

### 5.2 Compaction 完整流程

源码位置：`src/agents/pi-embedded-runner/compact.ts:248-744`

```
compactEmbeddedPiSessionDirect(params)
  │
  ├── 1. 获取写锁 (session write lock)                    ← 防止并发修改
  │
  ├── 2. 打开 SessionManager + SettingsManager
  │
  ├── 3. 消息准备
  │     ├── sanitizeSessionHistory()                       ← 清洗消息
  │     ├── validateGeminiTurns() / validateAnthropicTurns() ← 验证格式
  │     ├── limitHistoryTurns()                            ← 按 turn 数裁剪
  │     └── sanitizeToolUseResultPairing()                 ← 修复孤儿 tool_result
  │
  ├── 4. 触发 before_compaction hook（fire-and-forget）
  │
  ├── 5. session.compact(customInstructions)                ← SDK 的 compact 方法
  │     ↓ 触发 extension event
  │     session_before_compact → compaction-safeguard 接管
  │
  ├── 6. 触发 after_compaction hook（fire-and-forget）
  │
  ├── 7. 释放写锁
  │
  └── 8. dispose session
```

### 5.3 Compaction Safeguard（核心摘要逻辑）

源码位置：`src/agents/pi-extensions/compaction-safeguard.ts`

这是通过 SDK 的 extension API 注册的 `session_before_compact` 事件处理器，是**真正执行摘要的地方**。

```
session_before_compact 事件
  │
  ├── 1. 计算历史预算
  │     maxHistoryShare = 0.5（默认，历史最多占 50% 的上下文窗口）
  │     budgetTokens = contextWindowTokens × 0.5
  │
  ├── 2. 如果新内容过大 → 先裁剪老消息
  │     newContentTokens > maxHistoryTokens ?
  │       → pruneHistoryForContextShare() 按 chunk 丢弃老消息
  │       → 对丢弃的消息做摘要保留上下文
  │
  ├── 3. 分阶段摘要
  │     summarizeInStages({
  │       messages,           ← 待摘要的消息
  │       model,              ← 用当前模型做摘要
  │       parts: 2,           ← 默认分 2 段独立摘要再合并
  │       maxChunkTokens,     ← 每段的 token 上限
  │       previousSummary,    ← 上次的摘要作为上下文
  │     })
  │
  ├── 4. 追加附加信息
  │     ├── Tool Failures 列表（最多 8 个失败记录）
  │     ├── File Operations（读取/修改过的文件列表）
  │     └── Workspace Context（AGENTS.md 中的关键规则）
  │
  └── 5. 返回 compaction 结果
        {
          summary: "...",                   ← 摘要文本
          firstKeptEntryId: "...",          ← 保留的第一条消息 ID
          tokensBefore: 150000,             ← 压缩前 token 数
          details: { readFiles, modifiedFiles },
        }
```

### 5.4 分阶段摘要算法

源码位置：`src/agents/compaction.ts:276-337`

```
summarizeInStages(messages, parts=2)
  │
  ├── 消息数少或 token 少 → 直接调 summarizeWithFallback()
  │
  └── 消息数够多 →
        ├── splitMessagesByTokenShare(messages, 2)     ← 按 token 均分为 2 段
        │     ↓
        │   chunk1: [msg1, msg2, msg3, ...]
        │   chunk2: [msg4, msg5, msg6, ...]
        │
        ├── 独立摘要每段
        │   summary1 = summarizeWithFallback(chunk1)
        │   summary2 = summarizeWithFallback(chunk2)
        │
        └── 合并摘要
            mergedSummary = summarizeWithFallback(
              [summary1, summary2],
              instructions: "Merge these partial summaries into a single cohesive summary.
                             Preserve decisions, TODOs, open questions, and any constraints."
            )
```

### 5.5 渐进式降级

```
summarizeWithFallback()
  │
  ├── 尝试 1: 完整摘要 ← 对所有消息调 generateSummary()
  │   成功 → 返回
  │   失败 ↓
  │
  ├── 尝试 2: 部分摘要 ← 跳过超大消息（>50% 上下文），只摘要小消息
  │   成功 → 返回 "摘要 + [Large assistant (~45K tokens) omitted from summary]"
  │   失败 ↓
  │
  └── 尝试 3: 兜底文本
      → "Context contained 12 messages (2 oversized). Summary unavailable due to size limits."
```

每次 `generateSummary()` 调用都有**重试机制**：3 次，指数退避（500ms-5000ms），带 20% 抖动。

### 5.6 Token 预算管理

```
┌──────────────────────────────────────────────────────┐
│              总上下文窗口 (e.g. 200,000 tokens)         │
│                                                        │
│  ┌─────────────────────────────────────────────────┐  │
│  │ System Prompt (动态大小)                          │  │
│  ├─────────────────────────────────────────────────┤  │
│  │ 历史消息 (最多占 50%，即 100K tokens)             │  │
│  │  ├── 前次摘要 (compaction summary)                │  │
│  │  ├── 保留的最近消息                               │  │
│  │  └── (已裁剪的旧消息不在这里)                      │  │
│  ├─────────────────────────────────────────────────┤  │
│  │ 新内容 + AI 回复 (剩余空间)                       │  │
│  ├─────────────────────────────────────────────────┤  │
│  │ 摘要开销 (4,096 tokens 预留)                      │  │
│  └─────────────────────────────────────────────────┘  │
│                                                        │
│  安全系数: × 1.2 (估算误差的 20% 缓冲)                 │
└──────────────────────────────────────────────────────┘
```

**自适应 chunk 比例**：

```typescript
// 如果消息平均大小 > 10% 上下文窗口，减小 chunk 比例
BASE_CHUNK_RATIO = 0.4; // 默认每 chunk 占上下文的 40%
MIN_CHUNK_RATIO = 0.15; // 最小 15%

if (avgRatio > 0.1) {
  reduction = Math.min(avgRatio * 2, BASE_CHUNK_RATIO - MIN_CHUNK_RATIO);
  return Math.max(MIN_CHUNK_RATIO, BASE_CHUNK_RATIO - reduction);
}
```

### 5.7 上下文窗口大小来源

| 优先级 | 来源                                                             | 说明                 |
| ------ | ---------------------------------------------------------------- | -------------------- |
| 1      | model metadata (`model.contextWindow`)                           | SDK 从模型元数据获取 |
| 2      | config override (`models.providers[p].models[id].contextWindow`) | 手动配置覆盖         |
| 3      | `agents.defaults.contextTokens`                                  | 全局默认上限         |
| 4      | `DEFAULT_CONTEXT_TOKENS = 200,000`                               | 硬编码兜底值         |

特殊情况：Anthropic 1M context 模型可通过 `context1m = true` 启用 `1,048,576` tokens。

### 5.8 Context Window Guard

| 阈值     | 值            | 行为                 |
| -------- | ------------- | -------------------- |
| 硬下限   | 16,000 tokens | 低于此值**阻止运行** |
| 警告阈值 | 32,000 tokens | 低于此值发出警告     |

---

## 六、Compaction 后的 Session 文件结构

```jsonl
{"type":"session_info","name":"telegram:dm:alice"}
{"type":"message","id":"e1","role":"user","content":"帮我查一下天气"}
{"type":"message","id":"e2","role":"assistant","content":[...tool_use...]}
{"type":"message","id":"e3","role":"toolResult","content":[...]}
{"type":"message","id":"e4","role":"assistant","content":"今天晴天..."}
{"type":"message","id":"e5","role":"user","content":"帮我搜索AI新闻"}
... (更多消息)
{"type":"compaction","id":"e20","summary":"用户先查了天气(晴天)，然后搜索了AI新闻...","firstKeptEntryId":"e15","tokensBefore":85000,"details":{"readFiles":[],"modifiedFiles":[]}}
{"type":"message","id":"e21","role":"user","content":"继续上次的话题"}
... (新消息继续追加)
```

Compaction 之后：

- `e1` ~ `e14` 的消息仍在文件中，但**不再被加载到内存**
- `e20`（compaction entry）包含这些消息的摘要
- 从 `e15` 开始的消息被保留
- 新消息从 `e21` 继续追加

---

## 七、Session Write Lock

源码位置：`src/agents/session-write-lock.ts`

```
目的: 防止多个操作同时修改同一个 session 文件

锁获取: acquireSessionWriteLock({ sessionFile, maxHoldMs })
锁释放: sessionLock.release()

最大持有时间 = EMBEDDED_COMPACTION_TIMEOUT_MS (300 秒)

使用场景:
  - Compaction 期间
  - Tool Result 截断期间
  - 防止并发 agent 写入冲突
```

---

## 八、完整的上下文管理决策树

```
用户消息到达
  │
  ├── History Limit 有配置？
  │   ├── 有 → limitHistoryTurns(messages, limit) → 裁剪 + 修复配对
  │   └── 无 → 保留全部历史
  │
  ├── 有超大 tool result？
  │   ├── 有 → truncateOversizedToolResultsInMessages() → 内存中截断
  │   └── 无 → 跳过
  │
  ├── 发送给 LLM
  │   │
  │   ├── agent loop 每轮前 → Context Pruning
  │   │   ├── 上下文 < 30% → 不修剪
  │   │   ├── 30% ~ 50% → 软修剪（保留头尾）
  │   │   └── > 50% → 硬清除（替换为占位符）
  │   │
  │   ├── LLM 正常返回 → 继续 loop
  │   │
  │   └── LLM 返回 context overflow 错误
  │       │
  │       ├── 有超大 tool result → truncateOversizedToolResultsInSession() → 重试
  │       │
  │       └── 无或截断后仍溢出 → 触发 Compaction
  │           ├── 获取写锁
  │           ├── 准备消息 + 清洗
  │           ├── 新内容占比 > 50%？ → 先裁剪老 chunk
  │           ├── summarizeInStages() → 分段摘要
  │           ├── 追加 tool failures + file ops + workspace context
  │           ├── 保存 compaction entry 到 JSONL
  │           ├── 释放写锁
  │           └── 重试 agent loop
  │
  └── agent loop 结束 → session.dispose()
```

---

## 九、对最小可迁移版本的启示

### 必须实现的

```typescript
// 1. Token 估算（最简单的实现）
function estimateTokens(text: string): number {
  return Math.ceil(text.length / 4); // chars/4 粗估
}

// 2. 上下文溢出检测
function willOverflow(messages: Message[], systemPrompt: string, maxTokens: number): boolean {
  const total =
    estimateTokens(systemPrompt) +
    messages.reduce((sum, m) => sum + estimateTokens(JSON.stringify(m.content)), 0);
  return total > maxTokens * 0.8; // 留 20% 给回复
}

// 3. 最简单的压缩：截断旧消息
function compactSimple(messages: Message[], maxTokens: number): Message[] {
  while (willOverflow(messages, systemPrompt, maxTokens) && messages.length > 2) {
    messages.shift(); // 删除最老的消息
  }
  return messages;
}
```

### 建议实现的

```typescript
// 4. Tool Result 截断（防止单条消息炸上下文）
function truncateToolResult(result: string, maxChars: number): string {
  if (result.length <= maxChars) return result;
  return result.slice(0, maxChars) + "\n[Truncated]";
}

// 5. LLM 摘要（质量更好的压缩）
async function summarizeHistory(messages: Message[], model: string): Promise<string> {
  const response = await callLLM({
    model,
    messages: [
      { role: "user", content: `Summarize this conversation:\n${formatMessages(messages)}` },
    ],
  });
  return response.text;
}

// 6. 带摘要的压缩
async function compactWithSummary(messages: Message[], maxTokens: number): Promise<Message[]> {
  const [oldMessages, recentMessages] = splitAtMidpoint(messages);
  const summary = await summarizeHistory(oldMessages);
  return [
    { role: "user", content: `[Previous conversation summary: ${summary}]` },
    ...recentMessages,
  ];
}
```

### 可选实现的

| 功能            | OpenClaw 实现       | 简化替代                |
| --------------- | ------------------- | ----------------------- |
| Session 持久化  | JSONL + DAG 分支    | SQLite 或简单 JSON 文件 |
| Context Pruning | 实时软修剪/硬清除   | 发送前统一检查          |
| 分阶段摘要      | 分 chunk 摘要再合并 | 一次性摘要              |
| 渐进式降级      | 3 级 fallback       | try/catch 兜底          |
| Write Lock      | 文件级锁            | 单进程不需要            |
| 自适应 chunk    | 动态调整比例        | 固定比例                |

---

## 关键源码索引

| 文件                                                      | 核心函数                                   | 作用                |
| --------------------------------------------------------- | ------------------------------------------ | ------------------- |
| `src/agents/pi-embedded-runner/history.ts`                | `limitHistoryTurns()`                      | Turn 数限制         |
| `src/agents/pi-embedded-runner/history.ts`                | `getHistoryLimitFromSessionKey()`          | 按渠道/用户解析限制 |
| `src/agents/pi-embedded-runner/tool-result-truncation.ts` | `truncateOversizedToolResultsInSession()`  | Session 文件级截断  |
| `src/agents/pi-embedded-runner/tool-result-truncation.ts` | `truncateOversizedToolResultsInMessages()` | 内存级截断          |
| `src/agents/pi-extensions/context-pruning/pruner.ts`      | `pruneContextMessages()`                   | 实时上下文修剪      |
| `src/agents/pi-extensions/context-pruning/settings.ts`    | `DEFAULT_CONTEXT_PRUNING_SETTINGS`         | 修剪默认配置        |
| `src/agents/compaction.ts`                                | `summarizeInStages()`                      | 分阶段摘要          |
| `src/agents/compaction.ts`                                | `summarizeWithFallback()`                  | 渐进式降级摘要      |
| `src/agents/compaction.ts`                                | `pruneHistoryForContextShare()`            | 历史预算裁剪        |
| `src/agents/compaction.ts`                                | `computeAdaptiveChunkRatio()`              | 自适应 chunk 比例   |
| `src/agents/pi-extensions/compaction-safeguard.ts`        | `session_before_compact` handler           | 核心摘要编排逻辑    |
| `src/agents/pi-embedded-runner/compact.ts`                | `compactEmbeddedPiSessionDirect()`         | Compaction 入口     |
| `src/agents/session-write-lock.ts`                        | `acquireSessionWriteLock()`                | 写锁                |
