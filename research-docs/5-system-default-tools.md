# OpenClaw 系统默认 Tools 全景

## 注册入口

所有 OpenClaw 自有工具在 `src/agents/openclaw-tools.ts` 的 `createOpenClawTools()` 函数中统一注册。它分两步：

1. 逐个创建内置工具（`create*Tool()`）
2. 加载插件工具（`resolvePluginTools()`）

```typescript
// src/agents/openclaw-tools.ts（简化）
export function createOpenClawTools(options?): AnyAgentTool[] {
  const tools: AnyAgentTool[] = [
    createBrowserTool({ ... }),
    createCanvasTool({ ... }),
    createNodesTool({ ... }),
    createCronTool({ ... }),
    createMessageTool({ ... }),         // 可通过配置禁用
    createTtsTool({ ... }),
    createGatewayTool({ ... }),
    createAgentsListTool({ ... }),
    createSessionsListTool({ ... }),
    createSessionsHistoryTool({ ... }),
    createSessionsSendTool({ ... }),
    createSessionsSpawnTool({ ... }),
    createSubagentsTool({ ... }),
    createSessionStatusTool({ ... }),
    createWebSearchTool({ ... }),       // 可能返回 null（未配置 API key）
    createWebFetchTool({ ... }),        // 可能返回 null
    createImageTool({ ... }),           // 可能返回 null
  ];

  const pluginTools = resolvePluginTools({ ... });  // 插件工具
  return [...tools, ...pluginTools];
}
```

注意：这些是 OpenClaw 额外加的工具。AI SDK（`@mariozechner/pi-coding-agent`）还提供 5 个**底层编码工具**：`read`、`write`、`edit`、`exec`、`process`，在 `src/agents/pi-tools.ts` 中组装。

---

## 全部默认工具一览

### 底层编码工具（来自 AI SDK）

| 工具名    | 来源文件                                     | 功能            | 实际做什么                          |
| --------- | -------------------------------------------- | --------------- | ----------------------------------- |
| `read`    | SDK 内置，OpenClaw 包装于 `pi-tools.read.ts` | 读文件          | `fs.readFile()` + 分页 + 图片处理   |
| `write`   | SDK 内置                                     | 写文件          | `fs.writeFile()`                    |
| `edit`    | SDK 内置                                     | 编辑文件        | 基于 old_text → new_text 的精确替换 |
| `exec`    | `bash-tools.exec.ts`                         | 执行 shell 命令 | `child_process.spawn()`             |
| `process` | `bash-tools.exec-runtime.ts`                 | 管理后台进程    | 进程池管理（poll/kill/list）        |

### OpenClaw 自有工具（按功能分组）

#### 信息获取类

| 工具名              | 来源文件                | 功能             | 核心实现                                           |
| ------------------- | ----------------------- | ---------------- | -------------------------------------------------- |
| **`web_search`**    | `tools/web-search.ts`   | 网络搜索         | HTTP 请求 Brave/Perplexity/Grok/Gemini/Kimi API    |
| **`web_fetch`**     | `tools/web-fetch.ts`    | 抓取网页内容     | HTTP GET → Readability 提取 → Markdown 转换        |
| **`browser`**       | `tools/browser-tool.ts` | 浏览器自动化     | 通过 browser control server 操作 Chrome/内置浏览器 |
| **`image`**         | `tools/image-tool.ts`   | 图片分析         | 调用 Vision 模型（Claude/Minimax 等）描述图片      |
| **`memory_search`** | `tools/memory-tool.ts`  | 语义搜索记忆库   | sqlite-vec 向量搜索 MEMORY.md + memory/\*.md       |
| **`memory_get`**    | `tools/memory-tool.ts`  | 获取记忆文件内容 | 读取指定文件的指定行范围                           |

#### 消息通信类

| 工具名              | 来源文件                      | 功能                 | 核心实现                                      |
| ------------------- | ----------------------------- | -------------------- | --------------------------------------------- |
| **`message`**       | `tools/message-tool.ts`       | 发消息到渠道         | 通过 Gateway 路由到 Telegram/Discord/Slack 等 |
| **`tts`**           | `tools/tts-tool.ts`           | 文本转语音           | edge-tts / 渠道原生 TTS                       |
| **`sessions_send`** | `tools/sessions-send-tool.ts` | 发消息给其他 session | 通过 Gateway 转发到目标 session               |

#### 会话管理类

| 工具名                 | 来源文件                         | 功能                  | 核心实现                            |
| ---------------------- | -------------------------------- | --------------------- | ----------------------------------- |
| **`sessions_list`**    | `tools/sessions-list-tool.ts`    | 列出活跃会话          | 查询 Gateway 的 session 列表        |
| **`sessions_history`** | `tools/sessions-history-tool.ts` | 获取会话历史消息      | 读取 session 的 JSONL 文件          |
| **`sessions_spawn`**   | `tools/sessions-spawn-tool.ts`   | 创建子 agent          | 通过 Gateway 启动新的 agent session |
| **`session_status`**   | `tools/session-status-tool.ts`   | 获取会话详细状态      | 查询 session 的元数据和运行状态     |
| **`subagents`**        | `tools/subagents-tool.ts`        | 管理子 agent 生命周期 | steer/cancel/status 子 agent        |
| **`agents_list`**      | `tools/agents-list-tool.ts`      | 列出所有 agent        | 查询 Gateway 的 agent 注册表        |

#### 自动化与基础设施类

| 工具名        | 来源文件                | 功能              | 核心实现                                           |
| ------------- | ----------------------- | ----------------- | -------------------------------------------------- |
| **`cron`**    | `tools/cron-tool.ts`    | 定时任务管理      | 通过 Gateway 的 cron 子系统 CRUD 定时任务          |
| **`nodes`**   | `tools/nodes-tool.ts`   | 远程节点控制      | 通过 Gateway 操作远程设备（摄像头/屏幕/位置/命令） |
| **`canvas`**  | `tools/canvas-tool.ts`  | Canvas UI 控制    | 通过 Gateway 控制前端 Canvas 组件                  |
| **`gateway`** | `tools/gateway-tool.ts` | 调用 Gateway 函数 | 直接调用 Gateway 的任意方法                        |

---

## 统一的工具定义模式

所有工具都遵循同一个模式。以 `web_search` 为代表拆解：

```typescript
// src/agents/tools/web-search.ts

// ① 用 TypeBox 定义参数 schema（最终转为 JSON Schema 给 AI）
const WebSearchSchema = Type.Object({
  query: Type.String({ description: "Search query string." }),
  count: Type.Optional(Type.Number({ description: "Number of results (1-10).", minimum: 1, maximum: 10 })),
  freshness: Type.Optional(Type.String({ description: "Filter: pd/pw/pm/py or date range." })),
  // ...更多可选参数
});

// ② 工厂函数：根据配置创建工具实例（可能返回 null）
export function createWebSearchTool(options?: {
  config?: OpenClawConfig;
  sandboxed?: boolean;
}): AnyAgentTool | null {
  // 检查是否启用
  if (!resolveSearchEnabled({ ... })) return null;

  // ③ 返回工具对象：name + description + parameters + execute
  return {
    label: "Web Search",
    name: "web_search",
    description: "Search the web using Brave Search API. ...",
    parameters: WebSearchSchema,

    // ④ execute 函数：实际执行逻辑
    execute: async (_toolCallId, args) => {
      const query = readStringParam(args, "query", { required: true });
      const apiKey = resolveSearchApiKey(search);
      if (!apiKey) return jsonResult({ error: "No API key configured" });

      // 调用外部 API
      const result = await searchBrave({ query, apiKey, count, ... });

      // 返回结果（jsonResult 把对象序列化为文本给 AI）
      return jsonResult(result);
    },
  };
}
```

**每个工具的 4 个组成部分**：

```
┌─────────────────────────────────────────────────────────┐
│  ① name + label + description                          │
│     → AI 通过 description 决定何时使用这个工具            │
│                                                         │
│  ② parameters (TypeBox → JSON Schema)                   │
│     → AI 根据 schema 构造合法的参数                       │
│                                                         │
│  ③ execute(toolCallId, args, signal?, onUpdate?)        │
│     → 实际执行逻辑，返回 ToolResult                      │
│                                                         │
│  ④ ToolResult = { content: [{ type: "text", text }] }   │
│     → 纯文本结果，放回对话历史给 AI 看                    │
└─────────────────────────────────────────────────────────┘
```

---

## 工具的通用接口

```typescript
// src/agents/tools/common.ts

type AnyAgentTool = {
  name: string; // 工具名（AI 调用时用）
  label?: string; // 显示名
  description: string; // 工具描述（AI 读这个来决策）
  parameters: TObject; // JSON Schema（TypeBox 生成）
  execute: (
    toolCallId: string, // 本次调用的唯一 ID
    args: Record<string, unknown>, // AI 传入的参数
    signal?: AbortSignal, // 取消信号
    onUpdate?: (update: unknown) => void, // 流式中间结果回调
  ) => Promise<AgentToolResult>;
};

type AgentToolResult = {
  content: Array<
    | { type: "text"; text: string } // 文本结果
    | { type: "image"; mimeType?: string; data: string } // 图片（base64）
  >;
  details?: unknown; // 工具特有的元数据
};
```

---

## 各工具的实现策略分类

工具按"execute 里做什么"可以分为 4 类：

### 类型 A：直接调外部 API

```
web_search → HTTP 请求 Brave/Perplexity/Grok API
web_fetch  → HTTP GET 目标 URL → Readability 提取
image      → 调用 Vision 模型 API
tts        → 调用 edge-tts 或渠道 TTS API
```

**模式**：`execute` 内直接 `fetch()` 外部服务，处理响应，返回文本结果。

### 类型 B：通过 Gateway 中转

```
message       → callGatewayTool("message", "send", { ... })
cron          → callGatewayTool("cron", action, { ... })
nodes         → callGatewayTool("nodes", action, { ... })
canvas        → callGatewayTool("canvas", action, { ... })
gateway       → callGatewayTool(method, { ... })
sessions_*    → callGatewayTool("sessions", action, { ... })
agents_list   → callGatewayTool("agents", "list", { ... })
subagents     → callGatewayTool("subagents", action, { ... })
```

**模式**：`execute` 内调用 `callGatewayTool()` 通过 WebSocket 向 Gateway 发请求，Gateway 路由到对应子系统。

```typescript
// src/agents/tools/gateway.ts
async function callGatewayTool(method: string, params: Record<string, unknown>): Promise<unknown> {
  // WebSocket 请求 → Gateway handleGatewayRequest() → 对应 handler
}
```

### 类型 C：本地文件/进程操作

```
read    → fs.readFile()
write   → fs.writeFile()
edit    → 字符串替换 + fs.writeFile()
exec    → child_process.spawn()
process → 进程池查询/管理
```

**模式**：`execute` 内直接调用 Node.js API。

### 类型 D：本地数据查询

```
memory_search → sqlite-vec 向量搜索
memory_get    → 读取 memory 目录下的特定文件
browser       → 通过 browser control server 操作浏览器
```

**模式**：`execute` 内调用本地服务或数据库。

---

## 条件注册（哪些工具可能不存在）

不是所有工具都一定会注册。有些需要条件满足：

| 工具            | 条件                                    | 缺失原因                     |
| --------------- | --------------------------------------- | ---------------------------- |
| `web_search`    | `resolveSearchEnabled()` → 需要 API key | 无 Brave/Perplexity/Grok key |
| `web_fetch`     | `resolveWebFetchEnabled()`              | 配置禁用                     |
| `image`         | `options.agentDir` 必须存在             | 无 agent 目录                |
| `message`       | `!options.disableMessageTool`           | 配置禁用                     |
| `memory_search` | `resolveMemorySearchConfig()` 非空      | 未配置记忆后端               |
| `memory_get`    | 同上                                    | 同上                         |

返回 `null` 的工具会被过滤掉，不出现在 AI 的工具列表中。

---

## 对最小可迁移版本的启示

如果要复用这套模式，你只需要：

### 必须实现

```typescript
// 1. 定义工具接口
interface Tool {
  name: string;
  description: string;
  parameters: object;  // JSON Schema
  execute: (args: Record<string, unknown>) => Promise<{ content: string }>;
}

// 2. 实现几个工具
const tools = [
  { name: "exec",       execute: (args) => /* spawn shell */ },
  { name: "read",       execute: (args) => /* fs.readFile */ },
  { name: "write",      execute: (args) => /* fs.writeFile */ },
  { name: "web_search", execute: (args) => /* fetch Brave API */ },
];

// 3. 传给 AI API
const response = await anthropic.messages.create({
  tools: tools.map(t => ({
    name: t.name,
    description: t.description,
    input_schema: t.parameters,
  })),
  ...
});
```

### 可以简化的地方

| OpenClaw 的做法                            | 最小版本                  |
| ------------------------------------------ | ------------------------- |
| TypeBox 定义 schema → 自动生成 JSON Schema | 手写 JSON Schema 对象即可 |
| `callGatewayTool()` 通过 WebSocket 中转    | 直接在 execute 里写逻辑   |
| `jsonResult()` 序列化                      | `JSON.stringify()`        |
| 工厂函数 `create*Tool(options)` 条件创建   | 直接创建，不需要条件      |
| 工具策略过滤（sandbox、policy、allowlist） | 不需要                    |

### 最小工具集推荐

| 优先级 | 工具            | 代码量  | 说明                       |
| ------ | --------------- | ------- | -------------------------- |
| **P0** | `exec`          | ~50 行  | 万能工具，能做几乎任何事   |
| **P0** | `read`          | ~20 行  | 读文件                     |
| **P0** | `write`         | ~15 行  | 写文件                     |
| **P1** | `web_search`    | ~80 行  | 需要 Brave API key（免费） |
| **P1** | `web_fetch`     | ~60 行  | HTTP GET + HTML→text       |
| **P2** | `edit`          | ~40 行  | 比 write 更精确的编辑      |
| **P3** | `memory_search` | ~100 行 | 需要向量数据库             |

只实现 P0 的 3 个工具（~85 行代码），AI 就已经能做大部分事情了——因为 `exec` 可以跑 `curl`、`grep`、`git` 等任意命令。
