# Agent 循环最小可迁移版本 — 设计方案

## 一、OpenClaw Agent 循环的本质

剥掉所有渠道适配、插件系统、多 profile failover 后，OpenClaw 的 agent 循环本质是这个模式：

```
┌──────────────────────────────────────────┐
│              外层重试循环                  │
│  处理：认证失败、上下文溢出、模型降级       │
│                                          │
│  ┌──────────────────────────────────────┐ │
│  │          内层 agentic loop           │ │
│  │  ┌─────────┐                        │ │
│  │  │ 用户消息 │                        │ │
│  │  └────┬────┘                        │ │
│  │       ↓                             │ │
│  │  ┌─────────────┐                    │ │
│  │  │ 调用 AI API  │←──────────┐       │ │
│  │  └─────┬───────┘           │       │ │
│  │        ↓                   │       │ │
│  │  ┌───────────┐    ┌───────┴─────┐ │ │
│  │  │ 纯文本回复 │    │ tool_use 块  │ │ │
│  │  │ → 结束    │    │ → 执行工具   │ │ │
│  │  └───────────┘    │ → 结果放回   │ │ │
│  │                   │   对话历史   │ │ │
│  │                   └─────────────┘ │ │
│  └──────────────────────────────────────┘ │
└──────────────────────────────────────────┘
```

**最小可迁移版本只需要实现内层 agentic loop**，外层重试是锦上添花。

---

## 二、核心组件拆解（4 个）

### 组件 1：Session（对话历史管理）

**职责**：维护用户和 AI 之间的完整对话历史，持久化到磁盘。

```typescript
// --- 数据结构 ---

type Role = "user" | "assistant" | "tool_result";

type ContentBlock =
  | { type: "text"; text: string }
  | { type: "image"; mimeType: string; data: string }  // base64
  | { type: "tool_use"; id: string; name: string; input: Record<string, unknown> }
  | { type: "tool_result"; tool_use_id: string; content: string; is_error?: boolean };

type Message = {
  role: Role;
  content: string | ContentBlock[];
  timestamp: number;
  // assistant 独有
  model?: string;
  usage?: { input_tokens: number; output_tokens: number };
  stop_reason?: "end_turn" | "tool_use" | "max_tokens";
};

// --- 接口 ---

interface Session {
  id: string;
  messages: Message[];

  /** 追加消息并持久化 */
  append(msg: Message): void;

  /** 获取可发送给 AI API 的消息数组 */
  toApiMessages(): ApiMessage[];

  /** 持久化到磁盘 */
  save(): Promise<void>;

  /** 从磁盘加载 */
  static load(filePath: string): Promise<Session>;
}
```

**持久化格式**：JSONL（每行一个 JSON 对象），OpenClaw 的选择，原因：

- 追加写入 O(1)，不需要重写整个文件
- 崩溃安全：最坏丢最后一行
- 可 `tail -f` 实时观察
- 比 SQLite 更简单，对单用户场景足够

```jsonl
{"role":"user","content":"帮我写个排序函数","timestamp":1708900000000}
{"role":"assistant","content":[{"type":"text","text":"好的，我来写一个快速排序："},{"type":"tool_use","id":"call_1","name":"write_file","input":{"path":"sort.ts","content":"..."}}],"model":"claude-sonnet-4-20250514","stop_reason":"tool_use","timestamp":1708900001000}
{"role":"tool_result","content":[{"type":"tool_result","tool_use_id":"call_1","content":"文件已写入 sort.ts"}],"timestamp":1708900002000}
{"role":"assistant","content":[{"type":"text","text":"排序函数已写入 sort.ts。"}],"model":"claude-sonnet-4-20250514","stop_reason":"end_turn","timestamp":1708900003000}
```

---

### 组件 2：ToolRegistry（工具注册与执行）

**职责**：定义工具的 schema，执行工具，返回结果。

```typescript
// --- 工具定义 ---

interface ToolDefinition<TParams = Record<string, unknown>> {
  name: string;
  description: string;
  /** JSON Schema 格式，描述参数 */
  parameters: Record<string, unknown>;
  /** 执行函数 */
  execute: (params: TParams, signal?: AbortSignal) => Promise<ToolResult>;
}

interface ToolResult {
  content: string; // 文本结果（给 AI 看）
  is_error?: boolean; // 是否执行失败
  media?: Array<{
    // 可选：媒体附件
    type: string;
    data: string;
  }>;
}

// --- 注册表 ---

class ToolRegistry {
  private tools = new Map<string, ToolDefinition>();

  register(tool: ToolDefinition): void {
    this.tools.set(tool.name, tool);
  }

  get(name: string): ToolDefinition | undefined {
    return this.tools.get(name);
  }

  /** 转为 AI API 需要的 tools 参数格式 */
  toApiTools(): ApiToolDefinition[] {
    return [...this.tools.values()].map((t) => ({
      name: t.name,
      description: t.description,
      input_schema: t.parameters,
    }));
  }

  /** 执行一个工具调用 */
  async execute(
    name: string,
    params: Record<string, unknown>,
    signal?: AbortSignal,
  ): Promise<ToolResult> {
    const tool = this.tools.get(name);
    if (!tool) return { content: `Unknown tool: ${name}`, is_error: true };
    try {
      return await tool.execute(params, signal);
    } catch (err) {
      return { content: `Tool error: ${String(err)}`, is_error: true };
    }
  }
}
```

**OpenClaw 提供的工具模式值得参考**：

| 工具    | 参数                              | 返回         | 说明                    |
| ------- | --------------------------------- | ------------ | ----------------------- |
| `exec`  | `{ command, workdir?, timeout? }` | 命令输出文本 | 最核心的工具            |
| `read`  | `{ path, offset?, limit? }`       | 文件内容     | 支持分页读取大文件      |
| `write` | `{ path, content }`               | 确认信息     | 创建/覆盖文件           |
| `edit`  | `{ path, old_text, new_text }`    | diff 确认    | 精确替换，比 write 安全 |

---

### 组件 3：ModelClient（AI API 调用）

**职责**：调用 AI 模型 API，返回流式或非流式响应。

```typescript
// --- 响应数据结构 ---

type StopReason = "end_turn" | "tool_use" | "max_tokens";

interface ModelResponse {
  content: ContentBlock[]; // 文本块 + tool_use 块的混合
  stop_reason: StopReason;
  usage: { input_tokens: number; output_tokens: number };
  model: string;
}

// --- 流式事件 ---

type StreamEvent =
  | { type: "text_delta"; text: string }
  | { type: "tool_use_start"; id: string; name: string }
  | { type: "tool_use_delta"; id: string; partial_json: string }
  | { type: "tool_use_end"; id: string }
  | { type: "message_end"; stop_reason: StopReason; usage: ModelResponse["usage"] };

// --- 客户端接口 ---

interface ModelClient {
  /** 非流式调用（简单场景） */
  complete(params: {
    model: string;
    system: string;
    messages: ApiMessage[];
    tools?: ApiToolDefinition[];
    max_tokens?: number;
  }): Promise<ModelResponse>;

  /** 流式调用（生产场景） */
  stream(params: {
    model: string;
    system: string;
    messages: ApiMessage[];
    tools?: ApiToolDefinition[];
    max_tokens?: number;
  }): AsyncIterable<StreamEvent>;
}
```

**实现选项**：

- 直接用 `@anthropic-ai/sdk`（最简单）
- 或用 `openai` SDK（兼容 OpenAI + 很多兼容 API）
- OpenClaw 用的是 `@mariozechner/pi-ai` 的 `streamSimple()`，它是对多家 provider 的统一封装

---

### 组件 4：AgentLoop（核心循环）

**职责**：把上面三个组件串起来，实现 prompt → tool_use → 执行 → 继续 的循环。

```typescript
interface AgentLoopParams {
  session: Session;
  modelClient: ModelClient;
  toolRegistry: ToolRegistry;
  systemPrompt: string;
  model: string;
  /** 最大工具调用轮数，防止死循环 */
  maxToolRounds?: number; // 默认 30
  /** 流式回调 */
  onTextDelta?: (text: string) => void;
  onToolStart?: (name: string, params: Record<string, unknown>) => void;
  onToolEnd?: (name: string, result: ToolResult) => void;
  /** 超时 */
  signal?: AbortSignal;
}

interface AgentLoopResult {
  /** 最终文本回复 */
  text: string;
  /** 工具执行记录 */
  toolCalls: Array<{
    name: string;
    params: Record<string, unknown>;
    result: ToolResult;
  }>;
  /** token 用量 */
  usage: { input_tokens: number; output_tokens: number };
}
```

---

## 三、核心循环的完整伪代码

```typescript
async function runAgentLoop(params: AgentLoopParams): Promise<AgentLoopResult> {
  const {
    session,
    modelClient,
    toolRegistry,
    systemPrompt,
    model,
    maxToolRounds = 30,
    onTextDelta,
    onToolStart,
    onToolEnd,
    signal,
  } = params;

  const allToolCalls: AgentLoopResult["toolCalls"] = [];
  let totalUsage = { input_tokens: 0, output_tokens: 0 };
  let rounds = 0;

  while (rounds < maxToolRounds) {
    // ① 调用 AI API
    const response = await modelClient.complete({
      model,
      system: systemPrompt,
      messages: session.toApiMessages(),
      tools: toolRegistry.toApiTools(),
      max_tokens: 8192,
    });

    totalUsage.input_tokens += response.usage.input_tokens;
    totalUsage.output_tokens += response.usage.output_tokens;

    // ② 把 assistant 消息追加到 session
    session.append({
      role: "assistant",
      content: response.content,
      timestamp: Date.now(),
      model: response.model,
      usage: response.usage,
      stop_reason: response.stop_reason,
    });

    // ③ 提取文本（给流式回调）
    const textBlocks = response.content.filter((b) => b.type === "text");
    for (const block of textBlocks) {
      onTextDelta?.(block.text);
    }

    // ④ 检查是否有 tool_use
    const toolUseBlocks = response.content.filter((b) => b.type === "tool_use");

    if (toolUseBlocks.length === 0) {
      // 没有工具调用 → 对话结束
      break;
    }

    // ⑤ 执行所有工具调用
    const toolResults: ContentBlock[] = [];

    for (const toolUse of toolUseBlocks) {
      onToolStart?.(toolUse.name, toolUse.input);

      const result = await toolRegistry.execute(toolUse.name, toolUse.input, signal);

      onToolEnd?.(toolUse.name, result);

      allToolCalls.push({
        name: toolUse.name,
        params: toolUse.input,
        result,
      });

      toolResults.push({
        type: "tool_result",
        tool_use_id: toolUse.id,
        content: result.content,
        is_error: result.is_error,
      });
    }

    // ⑥ 把工具结果追加到 session
    session.append({
      role: "tool_result",
      content: toolResults,
      timestamp: Date.now(),
    });

    rounds++;
  }

  // ⑦ 拼接最终文本回复
  const finalText =
    session.messages
      .filter((m) => m.role === "assistant")
      .flatMap((m) =>
        Array.isArray(m.content)
          ? m.content.filter((b) => b.type === "text").map((b) => b.text)
          : [m.content],
      )
      .pop() ?? "";

  await session.save();

  return { text: finalText, toolCalls: allToolCalls, usage: totalUsage };
}
```

---

## 四、流式版本的关键差异

非流式版本足够用于后端/脚本场景。但如果需要实时把文字推送给用户（类似 ChatGPT 打字效果），需要用流式版本：

```typescript
async function runAgentLoopStreaming(params: AgentLoopParams): Promise<AgentLoopResult> {
  // ... 前面一样 ...

  while (rounds < maxToolRounds) {
    // 用 stream 替代 complete
    let currentText = "";
    const toolUseBlocks: ToolUseBlock[] = [];
    let stopReason: StopReason = "end_turn";
    const toolUseBuffers = new Map<string, string>(); // id → partial JSON

    for await (const event of modelClient.stream({ ... })) {
      switch (event.type) {
        case "text_delta":
          currentText += event.text;
          onTextDelta?.(event.text);  // 实时推送给前端
          break;

        case "tool_use_start":
          toolUseBuffers.set(event.id, "");
          break;

        case "tool_use_delta":
          toolUseBuffers.set(event.id,
            (toolUseBuffers.get(event.id) ?? "") + event.partial_json);
          break;

        case "tool_use_end": {
          const json = toolUseBuffers.get(event.id) ?? "{}";
          toolUseBlocks.push({
            type: "tool_use",
            id: event.id,
            name: /* from tool_use_start */,
            input: JSON.parse(json),
          });
          break;
        }

        case "message_end":
          stopReason = event.stop_reason;
          break;
      }
    }

    // 后续逻辑（追加 session、执行工具）和非流式版本一样
    // ...
  }
}
```

---

## 五、与 OpenClaw 的对比：我们简化了什么

| OpenClaw 特性                                            | 最小版本   | 是否需要                     |
| -------------------------------------------------------- | ---------- | ---------------------------- |
| 外层重试循环（auth failover、context overflow recovery） | 不实现     | 后期按需加                   |
| 多 auth profile 轮换                                     | 不实现     | 只用一个 API key             |
| 上下文压缩（compaction）                                 | 不实现     | 超长对话时需要               |
| thinking level 降级                                      | 不实现     | 只影响 extended thinking     |
| 工具结果截断                                             | **实现**   | 防止单次工具输出撑爆 context |
| 流式事件订阅系统                                         | 简化为回调 | 回调足够                     |
| Session JSONL 持久化                                     | **实现**   | 核心功能                     |
| 多 provider 支持                                         | 不实现     | 先支持一家                   |
| 插件工具注册                                             | 不实现     | 硬编码工具即可               |
| 沙箱/权限控制                                            | 不实现     | 按需加                       |
| 消息去重/防抖                                            | 不实现     | 这是 Channel 层的事          |

---

## 六、目录结构建议

```
my-agent/
├── src/
│   ├── session.ts          # Session 类（JSONL 读写 + 消息管理）
│   ├── tool-registry.ts    # ToolRegistry 类（工具注册与执行）
│   ├── model-client.ts     # ModelClient（封装 AI API 调用）
│   ├── agent-loop.ts       # runAgentLoop()（核心循环）
│   ├── tools/              # 具体工具实现
│   │   ├── exec.ts         # shell 执行
│   │   ├── read-file.ts    # 读取文件
│   │   ├── write-file.ts   # 写入文件
│   │   └── edit-file.ts    # 编辑文件
│   ├── system-prompt.ts    # 系统提示词构建
│   └── index.ts            # 入口：把所有组件串起来
├── sessions/               # JSONL 会话文件存储
├── package.json
└── tsconfig.json
```

---

## 七、依赖清单

```json
{
  "dependencies": {
    "@anthropic-ai/sdk": "^0.52.0",
    "typescript": "^5.8.0"
  },
  "devDependencies": {
    "vitest": "^4.0.0"
  }
}
```

**总依赖：1 个运行时包**。对比 OpenClaw 需要 `@mariozechner/pi-ai` + `@mariozechner/pi-agent-core` + `@mariozechner/pi-coding-agent` + 各渠道 SDK。

---

## 八、使用示例

```typescript
import Anthropic from "@anthropic-ai/sdk";
import { Session } from "./session.js";
import { ToolRegistry } from "./tool-registry.js";
import { createAnthropicClient } from "./model-client.js";
import { runAgentLoop } from "./agent-loop.js";
import { execTool } from "./tools/exec.js";
import { readFileTool } from "./tools/read-file.js";
import { writeFileTool } from "./tools/write-file.js";

// 1. 初始化组件
const session = await Session.load("./sessions/demo.jsonl");
const tools = new ToolRegistry();
tools.register(execTool);
tools.register(readFileTool);
tools.register(writeFileTool);

const client = createAnthropicClient(new Anthropic());

// 2. 添加用户消息
session.append({
  role: "user",
  content: "帮我写一个 Node.js HTTP 服务器，监听 3000 端口",
  timestamp: Date.now(),
});

// 3. 运行 agent 循环
const result = await runAgentLoop({
  session,
  modelClient: client,
  toolRegistry: tools,
  systemPrompt: "你是一个编程助手，可以使用工具来帮用户完成任务。",
  model: "claude-sonnet-4-20250514",
  onTextDelta: (text) => process.stdout.write(text),
  onToolStart: (name, params) => console.log(`\n[工具调用] ${name}`),
  onToolEnd: (name, result) => console.log(`[工具结果] ${result.content.slice(0, 100)}`),
});

// 4. 输出结果
console.log(`\n\n完成。共执行 ${result.toolCalls.length} 次工具调用。`);
console.log(`Token 用量: ${result.usage.input_tokens} in / ${result.usage.output_tokens} out`);
```

---

## 九、后续可选增强（按优先级）

| 优先级 | 增强             | 复杂度 | 说明                                |
| ------ | ---------------- | ------ | ----------------------------------- |
| P0     | 工具结果截断     | 低     | 避免单次工具输出超过 context window |
| P1     | AbortSignal 超时 | 低     | 防止工具执行或 API 调用卡死         |
| P1     | 流式版本         | 中     | 实时推送回复文本                    |
| P2     | 上下文压缩       | 中     | 对话过长时总结旧消息                |
| P2     | 多 provider 支持 | 中     | 加 OpenAI、Gemini                   |
| P3     | 认证 failover    | 中     | 多 API key 轮换                     |
| P3     | 工具权限控制     | 中     | 沙箱、审批机制                      |
| P4     | 插件系统         | 高     | 动态加载第三方工具                  |

---

## 十、从 OpenClaw 学到的关键设计决策

1. **JSONL 而不是 JSON/SQLite**：追加写入 + 崩溃安全，对 agent 场景完美匹配
2. **工具结果是纯文本**：不要把结构化数据塞给 AI，给它纯文本最可靠
3. **tool_use 循环由 AI 自己控制结束**：不要自己判断"够了吗"，让 AI 决定（通过 stop_reason）
4. **截断而不是报错**：工具输出太长就截断，比抛错好——AI 能处理不完整信息
5. **Session = 真理之源**：所有消息都先写入 session，API 调用从 session 构建消息列表
6. **回调而不是事件总线**：对于单次 agent 运行，简单回调比 EventEmitter 更清晰
