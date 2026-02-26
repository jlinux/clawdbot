# OpenClaw Memory（记忆）系统

## 概述

OpenClaw 的 Memory 系统是一个**长期持久化记忆架构**，基于 **Markdown 文件 + SQLite 向量数据库**。它提供对工作区记忆文件（`MEMORY.md`、`memory/*.md`）和可选会话转录的语义搜索，并在上下文压缩前自动执行记忆刷写（Memory Flush），将重要信息持久化。

---

## 架构总览

```
┌─────────────────────────────────────────────────────┐
│                  AGENT 对话                          │
│  memory_search(query)     memory_get(path)           │
└──────────┬──────────────────────┬───────────────────┘
           │                      │
           └──────────┬───────────┘
                      │
       ┌──────────────▼──────────────┐
       │   MemoryIndexManager        │
       └──────────────┬──────────────┘
                      │
       ┌──────────────┼──────────────┐
       │              │              │
  ┌────▼────┐   ┌────▼────┐   ┌────▼────┐
  │ Vector  │   │  FTS    │   │ Fallback│
  │ Search  │   │ (BM25)  │   │ JS     │
  │sqlite-vec│  │ Search  │   │ Cosine  │
  └────┬────┘   └────┬────┘   └────┬────┘
       │              │              │
       └──────────────┼──────────────┘
                      │
       ┌──────────────▼──────────────┐
       │      混合结果合并            │
       │  (vector 0.7 + keyword 0.3) │
       │  + MMR 多样性重排（可选）    │
       │  + 时间衰减（可选）          │
       └──────────────┬──────────────┘
                      │
       ┌──────────────▼──────────────┐
       │   SQLite 数据库              │
       │  chunks (文本+嵌入)          │
       │  chunks_vec (向量搜索)       │
       │  chunks_fts (BM25 搜索)      │
       │  embedding_cache             │
       └──────────┬──────────────────┐
                  │                  │
       ┌──────────▼──┐    ┌─────────▼──┐
       │ 文件系统     │    │ Embedding  │
       │ MEMORY.md   │    │ Providers  │
       │ memory/*.md │    │ OpenAI     │
       │ sessions/*  │    │ Gemini     │
       │  .jsonl     │    │ Voyage     │
       └─────────────┘    │ Mistral    │
                          │ Local      │
                          └────────────┘
```

---

## 1. 存储架构

### 1.1 文件结构

**位置**：Agent 工作区（默认 `~/.openclaw/workspace/`）

```
workspace/
├── MEMORY.md                    # 长期记忆（精选，常青）
├── memory/
│   ├── 2026-02-26.md           # 每日日志（追加，按日期）
│   └── topic-name.md           # 主题记忆文件（常青）
└── sessions/
    └── *.jsonl                 # 会话转录（可选索引）
```

**核心原则**：**Markdown 文件是唯一真相来源**，SQLite 只是索引。

| 文件类型               | 特点                 | 时间衰减   |
| ---------------------- | -------------------- | ---------- |
| `MEMORY.md`            | 精选事实、偏好、决策 | 不衰减     |
| `memory/YYYY-MM-DD.md` | 每日日志，运行上下文 | 按日期衰减 |
| `memory/topic-name.md` | 常青知识             | 不衰减     |
| `sessions/*.jsonl`     | 会话转录（可选）     | 按时间衰减 |

### 1.2 SQLite 数据库 Schema

**关键文件**：`src/memory/memory-schema.ts`

```sql
-- 文件追踪
CREATE TABLE files (
  path TEXT PRIMARY KEY,        -- 工作区相对路径
  source TEXT NOT NULL,         -- "memory" 或 "sessions"
  hash TEXT NOT NULL,           -- 内容哈希（变更检测）
  mtime INTEGER NOT NULL,
  size INTEGER NOT NULL
);

-- 文本块
CREATE TABLE chunks (
  id TEXT PRIMARY KEY,
  path TEXT NOT NULL,
  source TEXT NOT NULL,
  start_line INTEGER NOT NULL,  -- 1-indexed
  end_line INTEGER NOT NULL,
  hash TEXT NOT NULL,
  model TEXT NOT NULL,          -- 使用的嵌入模型
  text TEXT NOT NULL,           -- 块内容（最大 ~700 字符）
  embedding TEXT NOT NULL,      -- JSON 序列化的 float32[]
  updated_at INTEGER NOT NULL
);

-- BM25 全文搜索
CREATE VIRTUAL TABLE chunks_fts USING fts5(
  text, id UNINDEXED, path UNINDEXED,
  source UNINDEXED, model UNINDEXED
);

-- 向量搜索（sqlite-vec 扩展）
CREATE VIRTUAL TABLE chunks_vec (
  id TEXT PRIMARY KEY,
  embedding BLOB               -- Float32Array 二进制
);

-- 嵌入缓存
CREATE TABLE embedding_cache (
  provider TEXT NOT NULL,
  model TEXT NOT NULL,
  provider_key TEXT NOT NULL,
  hash TEXT NOT NULL,
  embedding TEXT NOT NULL,
  dims INTEGER,
  updated_at INTEGER NOT NULL,
  PRIMARY KEY (provider, model, provider_key, hash)
);
```

---

## 2. 向量嵌入生成与存储

### 2.1 Embedding Providers

**关键文件**：`src/memory/embeddings.ts`

**自动回退链**：

| 优先级 | Provider    | 默认模型                 | API Key 来源      |
| ------ | ----------- | ------------------------ | ----------------- |
| 1      | **OpenAI**  | `text-embedding-3-small` | `OPENAI_API_KEY`  |
| 2      | **Gemini**  | `gemini-embedding-001`   | `GEMINI_API_KEY`  |
| 3      | **Voyage**  | `voyage-4-large`         | `VOYAGE_API_KEY`  |
| 4      | **Mistral** | `mistral-embed`          | `MISTRAL_API_KEY` |
| 5      | **Local**   | `embeddinggemma-300m`    | 无需（本地运行）  |

### 2.2 向量规范化

```typescript
// src/memory/embeddings.ts:15-22
function sanitizeAndNormalizeEmbedding(vec: number[]): number[] {
  const sanitized = vec.map((v) => (Number.isFinite(v) ? v : 0));
  const magnitude = Math.sqrt(sanitized.reduce((sum, v) => sum + v * v, 0));
  if (magnitude < 1e-10) return sanitized;
  return sanitized.map((v) => v / magnitude); // L2 归一化
}
```

### 2.3 批量嵌入

```typescript
// src/memory/manager-embedding-ops.ts
const EMBEDDING_BATCH_MAX_TOKENS = 8000;
const EMBEDDING_INDEX_CONCURRENCY = 4;
const EMBEDDING_RETRY_MAX_ATTEMPTS = 3;
const EMBEDDING_RETRY_BASE_DELAY_MS = 500;
```

支持 **Batch API**（OpenAI/Gemini/Voyage）降低大量索引的成本。

### 2.4 嵌入缓存

按 `(provider, model, provider_key, hash)` 复合键缓存，避免重复嵌入相同内容。

---

## 3. 文本分块

### Markdown 分块算法

**关键文件**：`src/memory/internal.ts:184-265`

```typescript
export function chunkMarkdown(
  content: string,
  chunking: { tokens: number; overlap: number },
): MemoryChunk[];
```

**默认参数**：

- `tokens = 400`（约 1600 字符）
- `overlap = 80`（约 320 字符重叠）

**算法**：

1. 按行分割文件
2. 累积行直到达到 `maxChars` 阈值
3. 创建 chunk：`{startLine, endLine, text, hash}`
4. 保留 `overlapChars` 的行传递到下一个 chunk
5. 所有行保持 1-indexed 行号

### Session 文件处理

`src/memory/session-files.ts:39-131`

- 解析 JSONL 会话转录
- 提取 user/assistant 角色的 `message.content`
- 规范化空白（折叠换行符）
- 创建 `lineMap`：内容行 → JSONL 行号映射

---

## 4. 混合搜索（Vector + BM25）

### 4.1 搜索决策树

```
查询到达
  │
  ├── 无 embedding provider？
  │   └── FTS-only 模式
  │       └── 关键词提取 → 多次 FTS 搜索 → 合并
  │
  └── 有 embedding provider？
      ├── 混合搜索（默认）
      │   ├── 向量搜索 (searchVector)
      │   ├── 关键词搜索 (searchKeyword)
      │   └── 加权合并结果
      │       ├── MMR（可选）
      │       └── 时间衰减（可选）
      │
      └── 仅向量模式
          └── 返回 top-K
```

### 4.2 向量搜索

```sql
-- src/memory/manager-search.ts:20-94
SELECT c.id, c.path, c.start_line, c.end_line, c.text, c.source,
       vec_distance_cosine(v.embedding, ?) AS dist
  FROM chunks_vec v
  JOIN chunks c ON c.id = v.id
 WHERE c.model = ?
 ORDER BY dist ASC
 LIMIT ?

-- score = 1 - cosine_distance
```

**回退**：无 sqlite-vec 时，加载所有 chunks 在 JS 中计算余弦相似度。

### 4.3 关键词（BM25）搜索

```typescript
// src/memory/hybrid.ts:33-49
export function buildFtsQuery(raw: string): string | null {
  const tokens = raw.match(/[\p{L}\p{N}_]+/gu);
  // 构建: "token1" AND "token2" AND "token3"
  return tokens.map((t) => `"${t}"`).join(" AND ");
}
```

```sql
SELECT id, path, source, start_line, end_line, text,
       bm25(chunks_fts) AS rank
  FROM chunks_fts
 WHERE chunks_fts MATCH ?
 ORDER BY rank ASC
 LIMIT ?
```

BM25 分数转换：`score = 1 / (1 + rank)`

### 4.4 混合结果合并

```typescript
// src/memory/hybrid.ts:51-72
// 默认权重：
vectorWeight = 0.7; // 70% 语义相似度
textWeight = 0.3; // 30% 关键词匹配

// 合并公式：
score = vectorWeight * vectorScore + textWeight * textScore;
```

### 4.5 高级排序

**MMR（最大边际相关性）** — `src/memory/mmr.ts`

```typescript
export type MMRConfig = {
  enabled: boolean; // 默认 false
  lambda: number; // 0=最大多样性, 1=最大相关性, 默认 0.7
};
// MMR = λ × relevance - (1-λ) × max_similarity_to_selected
```

**时间衰减** — `src/memory/temporal-decay.ts`

```typescript
export type TemporalDecayConfig = {
  enabled: boolean; // 默认 false
  halfLifeDays: number; // 默认 30
};
// 公式: score *= exp(-λ × ageInDays)，λ = ln(2) / halfLifeDays
```

- 日期文件（`memory/2026-02-15.md`）：从文件名提取日期
- 常青文件（`MEMORY.md`、`memory/topic.md`）：不衰减

---

## 5. memory_search 工具

### 定义

```typescript
// src/agents/tools/memory-tool.ts:40-99
{
  name: "memory_search",
  description: "Semantic search over MEMORY.md + memory/*.md",
  parameters: {
    query: string,
    maxResults?: number,     // 默认 6
    minScore?: number,       // 默认 0.35
  },
  execute: async (toolCallId, params) => {
    const manager = await getMemorySearchManager({ cfg, agentId });
    return await manager.search(query, { maxResults, minScore });
  }
}
```

### 返回结果

```typescript
export type MemorySearchResult = {
  path: string; // 相对路径 "memory/2026-02-15.md"
  startLine: number; // 1-indexed
  endLine: number;
  score: number; // 0-1，越高越好
  snippet: string; // 最多 700 字符
  source: "memory" | "sessions";
  citation?: string; // "path#L10-L15"
};
```

---

## 6. memory_get 工具

```typescript
// src/agents/tools/memory-tool.ts:101-140
{
  name: "memory_get",
  description: "Read specific memory file or line range",
  parameters: {
    path: string,        // 相对路径
    from?: number,       // 起始行（1-indexed）
    lines?: number,      // 读取行数
  },
  execute: async (toolCallId, params) => {
    return await manager.readFile({ relPath: path, from, lines });
  }
}
```

**路径验证**：仅允许 `MEMORY.md`、`memory.md`、`memory/**/*.md` 和配置的 `extraPaths`。

---

## 7. Memory Flush（压缩前记忆刷写）

### 7.1 目的

在 **Context Compaction 之前**执行一次静默 agent 轮次，提醒模型将重要事实写入磁盘。

### 7.2 触发条件

`src/auto-reply/reply/memory-flush.ts:113-144`

```
阈值 = contextWindow - reserveFloor - softThreshold
     = 200K - 20K - 4K = 176K tokens

当 session 总 token 数 >= 阈值 → 触发 Memory Flush
```

### 7.3 Flush Prompt

```typescript
// memory-flush.ts:9-102
DEFAULT_MEMORY_FLUSH_PROMPT = [
  "Pre-compaction memory flush.",
  "Store durable memories now (use memory/YYYY-MM-DD.md).",
  "IMPORTANT: APPEND new content only, do not overwrite.",
  "If nothing to store, reply with [[silent]].",
].join(" ");
```

### 7.4 执行流程

`src/auto-reply/reply/agent-runner-memory.ts:27-172`

1. 检查 flush 是否启用 + 是否已在本轮 compaction 执行过
2. 计算 token 阈值，判断是否需要 flush
3. 构造 flush prompt（含当日日期 `YYYY-MM-DD`）
4. 运行 embedded PI agent（带 memory 工具）
5. Agent 写入 `memory/YYYY-MM-DD.md`（或回复 `[[silent]]`）
6. 更新 session 元数据：`memoryFlushAt`、`memoryFlushCompactionCount`

**用户不可见**：flush 的输出是静默的。

### 7.5 跳过条件

- flush 禁用（`memoryFlush.enabled = false`）
- CLI provider 会话
- 心跳消息
- 只读沙盒（`workspaceAccess: "ro"`）
- 本轮 compaction 已经 flush 过

---

## 8. 同步与索引

### 文件监控

使用 **chokidar** 跨平台文件监控：

- 监控 `MEMORY.md`、`memory/` 及额外路径
- 防抖：默认 1500ms
- 忽略：git、node_modules、venv 等

### 同步流程

```
1. 列出磁盘上的 memory 文件
2. 列出 session 文件（如果启用）
3. 对每个文件：
   - 检查 hash 是否变化（未变则跳过）
   - Markdown 分块
   - 批量嵌入（并发 4）
   - 存入 SQLite（chunks + FTS + vector）
4. 清理已删除文件
5. 标记为 clean
```

### 同步策略配置

```typescript
sync: {
  onSessionStart: true,   // session 初始化时预热
  onSearch: true,          // 搜索前同步
  watch: true,             // 启用文件监控
  watchDebounceMs: 1500,   // 防抖间隔
  intervalMinutes: 0,      // 周期同步（默认关闭）
}
```

---

## 9. 关键常量汇总

### 索引参数

| 常量                        | 值   | 说明           |
| --------------------------- | ---- | -------------- |
| `DEFAULT_CHUNK_TOKENS`      | 400  | 分块大小       |
| `DEFAULT_CHUNK_OVERLAP`     | 80   | 重叠 token     |
| `DEFAULT_WATCH_DEBOUNCE_MS` | 1500 | 文件监控防抖   |
| `DEFAULT_MAX_RESULTS`       | 6    | 默认返回结果数 |
| `DEFAULT_MIN_SCORE`         | 0.35 | 最低相关性分数 |
| `SNIPPET_MAX_CHARS`         | 700  | 片段最大字符   |

### 混合搜索参数

| 常量                               | 值    | 说明                |
| ---------------------------------- | ----- | ------------------- |
| `DEFAULT_VECTOR_WEIGHT`            | 0.7   | 向量搜索权重        |
| `DEFAULT_TEXT_WEIGHT`              | 0.3   | 关键词搜索权重      |
| `DEFAULT_MMR_LAMBDA`               | 0.7   | MMR 相关/多样性平衡 |
| `DEFAULT_TEMPORAL_DECAY_HALF_LIFE` | 30 天 | 时间衰减半衰期      |

### Memory Flush 参数

| 常量                  | 值    | 说明            |
| --------------------- | ----- | --------------- |
| `softThresholdTokens` | 4000  | 软阈值          |
| `reserveTokensFloor`  | 20000 | 预留 token 下限 |

---

## 10. 关键文件索引

| 文件                                          | 职责                                     |
| --------------------------------------------- | ---------------------------------------- |
| `src/memory/manager.ts`                       | MemoryIndexManager（搜索、同步、schema） |
| `src/memory/manager-search.ts`                | 向量搜索 + 关键词搜索实现                |
| `src/memory/manager-embedding-ops.ts`         | 批量嵌入和缓存                           |
| `src/memory/manager-sync-ops.ts`              | 文件监控和同步协调                       |
| `src/memory/memory-schema.ts`                 | SQLite schema 定义                       |
| `src/memory/embeddings.ts`                    | Embedding provider 工厂                  |
| `src/memory/hybrid.ts`                        | 混合搜索合并 + BM25 评分                 |
| `src/memory/mmr.ts`                           | MMR 多样性重排                           |
| `src/memory/temporal-decay.ts`                | 时间衰减评分                             |
| `src/memory/internal.ts`                      | 分块、哈希、嵌入解析                     |
| `src/memory/session-files.ts`                 | Session 转录提取                         |
| `src/agents/tools/memory-tool.ts`             | memory_search + memory_get 工具          |
| `src/agents/memory-search.ts`                 | Memory 搜索配置解析                      |
| `src/auto-reply/reply/agent-runner-memory.ts` | Memory Flush 编排                        |
| `src/auto-reply/reply/memory-flush.ts`        | Flush 触发逻辑 + 默认值                  |

---

## 11. 对最小可迁移版本的启示

### 必须实现

```typescript
// 1. Markdown 文件作为记忆存储
// MEMORY.md + memory/*.md

// 2. 嵌入 + 向量搜索
const embedding = await openai.embeddings.create({
  model: "text-embedding-3-small",
  input: query,
});
// 用 sqlite-vec 或简单的余弦相似度搜索

// 3. 记忆搜索工具
const memorySearchTool = {
  name: "memory_search",
  execute: async (args) => {
    const results = await vectorSearch(args.query);
    return results.map((r) => r.snippet).join("\n");
  },
};
```

### 可以简化的地方

| OpenClaw 做法                     | 最小版本         |
| --------------------------------- | ---------------- |
| 混合搜索（vector 70% + BM25 30%） | 仅向量搜索       |
| MMR + 时间衰减                    | 不需要           |
| 5 个 embedding provider + 回退链  | 只用 OpenAI      |
| Batch API 优化                    | 逐个嵌入         |
| Session 转录索引                  | 不需要           |
| Memory Flush（压缩前刷写）        | 手动保存         |
| chokidar 文件监控                 | 搜索时同步       |
| 嵌入缓存                          | 不需要（小规模） |
