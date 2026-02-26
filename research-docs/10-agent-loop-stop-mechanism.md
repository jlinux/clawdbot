# Agent Loop 停止机制：LLM 返回什么决定循环继续/结束

## 核心结论

System prompt **完全不干预** LLM 的返回格式。Agent loop 的"继续/结束"判断完全依赖 **AI API 原生的 tool calling 协议**——回复中有 `tool_use` 就继续，没有就结束。这是 Claude/GPT 等模型的内置能力，不需要 prompt 工程。

---

## AI API 原生的 Tool Calling 协议

当发送请求给 Claude API 时，请求里有 `tools` 参数：

```json
POST /v1/messages
{
  "system": "You are a personal assistant...",
  "messages": [{ "role": "user", "content": "帮我搜索最新的AI信息" }],
  "tools": [
    { "name": "web_search", "description": "...", "input_schema": {...} },
    { "name": "exec", "description": "...", "input_schema": {...} }
  ]
}
```

Claude 的回复只有两种可能：

```
可能 A — 纯文本回复（不调工具）:
{
  "content": [{ "type": "text", "text": "这是搜索结果..." }],
  "stop_reason": "end_turn"          ← 模型主动结束
}

可能 B — 调用工具:
{
  "content": [
    { "type": "text", "text": "让我搜索一下" },
    { "type": "tool_use", "id": "toolu_xxx",
      "name": "web_search",
      "input": { "query": "latest AI news" } }
  ],
  "stop_reason": "tool_use"          ← 模型要求执行工具
}
```

**这是 AI API 层面的原生行为，不需要任何 prompt 指令。**

---

## Agent Loop 的判断逻辑

源码位置：`node_modules/@mariozechner/pi-agent-core/dist/agent-loop.js:59`

```javascript
async function runLoop(currentContext, newMessages, config, signal, stream, streamFn) {
    let hasMoreToolCalls = true;

    while (hasMoreToolCalls || pendingMessages.length > 0) {
        // ① 发请求给 AI，获取回复
        const message = await streamAssistantResponse(...);

        // ② 如果出错或被中断，直接退出
        if (message.stopReason === "error" || message.stopReason === "aborted") {
            return;  // 退出循环
        }

        // ③ 关键判断：回复中是否包含 toolCall
        const toolCalls = message.content.filter(c => c.type === "toolCall");
        hasMoreToolCalls = toolCalls.length > 0;

        // ④ 如果有 toolCall，执行工具，结果放回消息列表
        if (hasMoreToolCalls) {
            const toolExecution = await executeToolCalls(...);
            // 工具结果加入 context.messages
        }

        // ⑤ 如果没有 toolCall（hasMoreToolCalls = false），
        //    且没有 pendingMessages，while 条件不满足 → 退出循环
    }
}
```

**关键：代码检查的是回复 `content` 中是否包含 `type: "toolCall"` 的块，而不是 `stopReason`。**

---

## stop_reason 的映射

源码位置：`node_modules/@mariozechner/pi-ai/dist/providers/anthropic.js:681`

```javascript
function mapStopReason(reason) {
  switch (reason) {
    case "end_turn":
      return "stop"; // AI 主动结束 → 循环退出
    case "tool_use":
      return "toolUse"; // AI 要调工具 → 循环继续
    case "max_tokens":
      return "length"; // token 用完 → 循环退出
    case "refusal":
      return "error"; // 拒绝回答 → 循环退出
    case "pause_turn":
      return "stop"; // 暂停 → 循环退出
    case "stop_sequence":
      return "stop"; // 停止序列 → 循环退出
    case "sensitive":
      return "error"; // 安全过滤 → 循环退出
  }
}
```

agent loop 只对 `stopReason === "error" || "aborted"` 做特殊处理（提前退出）。其他情况统一看 content 中是否有 toolCall。

在实践中 `stop_reason: "tool_use"` 时必然有 `tool_use` content block，两者是一致的，但代码实现是基于内容检查的。

---

## 退出循环的所有可能路径

```
┌──────────────────────────────────────────────────────────────┐
│                    Agent Loop 退出条件                         │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  路径 1: AI 回复无 toolCall（正常完成）                        │
│    stop_reason: "end_turn" → stopReason: "stop"              │
│    content: [{ type: "text", text: "这是结果..." }]          │
│    → hasMoreToolCalls = false                                │
│    → 无 pendingMessages                                      │
│    → while 条件不满足 → 退出                                  │
│                                                              │
│  路径 2: 错误                                                 │
│    stopReason: "error"                                       │
│    → 提前 return                                             │
│                                                              │
│  路径 3: 被中断                                               │
│    stopReason: "aborted"                                     │
│    → 提前 return                                             │
│                                                              │
│  路径 4: token 用完                                           │
│    stop_reason: "max_tokens" → stopReason: "length"          │
│    → 无 toolCall（token 不够生成完整 tool_use 块）            │
│    → hasMoreToolCalls = false → 退出                         │
│                                                              │
│  继续循环: AI 回复有 toolCall                                  │
│    stop_reason: "tool_use" → stopReason: "toolUse"           │
│    content: [{ type: "toolCall", name: "web_search", ... }]  │
│    → hasMoreToolCalls = true                                 │
│    → 执行工具 → 结果放回 messages → 下一轮 while 循环         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 额外的退出后检查：follow-up messages

内循环退出后，还有一个外循环检查：

```javascript
// 内循环退出后
const followUpMessages = (await config.getFollowUpMessages?.()) || [];
if (followUpMessages.length > 0) {
    // 有排队中的后续消息 → 继续外循环
    pendingMessages = followUpMessages;
    continue;
}
// 没有后续消息 → 真正退出
break;
```

这是为了处理"用户在 AI 思考时又发了新消息"的场景。

---

## System Prompt 中与"继续/结束"间接相关的内容

虽然 system prompt 没有显式要求返回格式，但有几条**间接影响** AI 决定"用工具"还是"直接回答"的指令：

| System Prompt 内容                                                   | 间接影响                                                       |
| -------------------------------------------------------------------- | -------------------------------------------------------------- |
| `"do not narrate routine, low-risk tool calls (just call the tool)"` | 鼓励 AI 直接调工具，减少纯文本中间回复                         |
| `"Narrate only when it helps"`                                       | 减少不必要的 end_turn（纯文本回复）                            |
| Skills 强制："read its SKILL.md, then follow it"                     | 引导 AI 先调 read 工具，增加 tool_use 轮数                     |
| Memory 强制："Before answering... run memory_search"                 | 引导 AI 先调 memory_search 工具                                |
| `__SILENT__` 机制                                                    | 特殊的结束方式——AI 返回纯文本 `__SILENT__`，后续代码识别并丢弃 |
| `HEARTBEAT_OK` 机制                                                  | 特殊的结束方式——AI 返回纯文本 `HEARTBEAT_OK`                   |

这些都是**影响 AI 的决策倾向**（"更倾向于调工具"还是"直接回答"），而不是控制返回格式。

---

## 完整流程图

```
                      ┌──────────────────────────────────┐
                      │        AI API 请求                │
                      │  system: "You are..."            │
                      │  messages: [用户消息, ...]        │
                      │  tools: [web_search, exec, ...]   │
                      └──────────────┬───────────────────┘
                                     │
                                     ↓
                      ┌──────────────────────────────────┐
                      │        Claude 模型内部             │
                      │                                    │
                      │  我有这些工具可以用...              │
                      │  用户想搜索 → 我应该用 web_search  │
                      │                                    │
                      │  决策完全基于:                      │
                      │  1. tools 参数中的 schema          │
                      │  2. system prompt 中的工具描述      │
                      │  3. 模型的预训练能力                │
                      │                                    │
                      │  不需要任何 prompt 指令             │
                      │  告诉它"返回 tool_use 格式"        │
                      └──────────────┬───────────────────┘
                                     │
                          ┌──────────┴──────────┐
                          ↓                     ↓
                   有 tool_use              无 tool_use
                   stop_reason:             stop_reason:
                   "tool_use"               "end_turn"
                          │                     │
                          ↓                     ↓
              ┌───────────────────┐    ┌────────────────┐
              │ SDK 解析为         │    │ SDK 解析为      │
              │ type: "toolCall"   │    │ type: "text"    │
              │ → 执行工具         │    │ → 无 toolCall   │
              │ → 结果放回 messages │    │                  │
              │ → 继续循环 ✅      │    │ → 退出循环 ❌   │
              └───────────────────┘    └────────────────┘
```

---

## 对最小可迁移版本的启示

你不需要在 system prompt 里写任何关于"何时结束"的指令。只需要：

```typescript
// 最小 agent loop
async function agentLoop(userMessage: string, systemPrompt: string, tools: Tool[]) {
  const messages = [{ role: "user", content: userMessage }];

  while (true) {
    const response = await anthropic.messages.create({
      system: systemPrompt,
      messages,
      tools: tools.map((t) => ({
        name: t.name,
        description: t.description,
        input_schema: t.schema,
      })),
    });

    // 把 AI 回复加入消息历史
    messages.push({ role: "assistant", content: response.content });

    // 检查是否有 tool_use
    const toolUses = response.content.filter((b) => b.type === "tool_use");

    if (toolUses.length === 0) {
      // 没有工具调用 → AI 认为任务完成 → 退出循环
      break;
    }

    // 执行工具，把结果放回消息历史
    const toolResults = [];
    for (const toolUse of toolUses) {
      const tool = tools.find((t) => t.name === toolUse.name);
      const result = await tool.execute(toolUse.input);
      toolResults.push({
        type: "tool_result",
        tool_use_id: toolUse.id,
        content: result,
      });
    }
    messages.push({ role: "user", content: toolResults });
  }

  // 返回最终的文本回复
  return response.content
    .filter((b) => b.type === "text")
    .map((b) => b.text)
    .join("");
}
```

**循环的"继续/结束"完全由 AI 模型自己决定——它觉得还需要调工具就返回 tool_use，觉得够了就返回纯文本。不需要任何 prompt 工程。**

---

## 附录：Tool Calling 协议深度解析

### 一、各家 LLM 的 Tool Calling 协议对比

通过 OpenClaw 的 SDK（`@mariozechner/pi-ai`）可以看到，它适配了 **5 种不同的 API 协议**：

| API 协议                    | 提供商                                   | 工具定义格式                                                                       | 工具调用格式                                                            | 工具结果格式                                               |
| --------------------------- | ---------------------------------------- | ---------------------------------------------------------------------------------- | ----------------------------------------------------------------------- | ---------------------------------------------------------- |
| **Anthropic Messages**      | Claude                                   | `tools: [{ name, description, input_schema }]`                                     | `content: [{ type: "tool_use", id, name, input }]`                      | `content: [{ type: "tool_result", tool_use_id, content }]` |
| **OpenAI Chat Completions** | GPT, DeepSeek, Mistral, xAI, Cerebras 等 | `tools: [{ type: "function", function: { name, description, parameters } }]`       | `tool_calls: [{ id, type: "function", function: { name, arguments } }]` | `{ role: "tool", tool_call_id, content }`                  |
| **OpenAI Responses**        | GPT (新 API)                             | `tools: [{ type: "function", name, description, parameters }]`                     | `{ type: "function_call", call_id, name, arguments }`                   | `{ type: "function_call_output", call_id, output }`        |
| **Google Gemini**           | Gemini                                   | `tools: [{ functionDeclarations: [{ name, description, parametersJsonSchema }] }]` | `parts: [{ functionCall: { name, args } }]`                             | `parts: [{ functionResponse: { name, response } }]`        |
| **Amazon Bedrock**          | Claude (via AWS)                         | Anthropic 格式的 AWS 包装                                                          | 同 Anthropic                                                            | 同 Anthropic                                               |

#### 关键差异

**1. 工具定义的 schema 字段名不同**

```
Anthropic:   input_schema: { type: "object", properties: {...} }
OpenAI:      parameters: { type: "object", properties: {...} }
Gemini:      parametersJsonSchema: { type: "object", properties: {...} }
             （旧版用 parameters，不支持完整 JSON Schema）
```

**2. 工具调用在回复中的位置不同**

```
Anthropic:  放在 content 数组里，和 text 块并列
  content: [
    { type: "text", text: "让我搜索一下" },
    { type: "tool_use", id: "toolu_xxx", name: "web_search", input: {...} }
  ]

OpenAI Chat:  放在独立的 tool_calls 字段
  message: {
    content: "让我搜索一下",
    tool_calls: [{ id: "call_xxx", function: { name: "web_search", arguments: "{...}" } }]
  }
  注意: arguments 是 JSON 字符串，不是对象！

OpenAI Responses:  作为独立的 output item
  output: [
    { type: "message", content: [{ type: "output_text", text: "让我搜索一下" }] },
    { type: "function_call", call_id: "fc_xxx", name: "web_search", arguments: "{...}" }
  ]

Gemini:  放在 parts 数组里
  candidates: [{ content: { parts: [
    { text: "让我搜索一下" },
    { functionCall: { name: "web_search", args: {...} } }
  ]}}]
  注意: args 是对象，不是字符串
```

**3. 工具结果的传回方式不同**

```
Anthropic:  放在 user 消息的 content 里
  { role: "user", content: [
    { type: "tool_result", tool_use_id: "toolu_xxx", content: "搜索结果..." }
  ]}

OpenAI Chat:  独立的 tool 角色消息
  { role: "tool", tool_call_id: "call_xxx", content: "搜索结果..." }

OpenAI Responses:  function_call_output 类型
  { type: "function_call_output", call_id: "fc_xxx", output: "搜索结果..." }

Gemini:  functionResponse 在 parts 里
  { role: "user", parts: [
    { functionResponse: { name: "web_search", response: { result: "搜索结果..." } } }
  ]}
```

**4. stop_reason 的表述不同**

```
Anthropic:   "end_turn" / "tool_use" / "max_tokens"
OpenAI:      "stop" / "tool_calls" / "length"（旧版还有 "function_call"）
Gemini:      FinishReason.STOP / FinishReason.MAX_TOKENS（无专门的 tool 停止原因）
```

---

### 二、Tool Calling 协议的共同特点

尽管格式不同，所有支持 tool calling 的 LLM API 都遵循相同的**概念模型**：

```
┌─────────────────────────────────────────────────────────┐
│                 共同的概念模型                              │
│                                                          │
│  1. 工具定义（随请求发送）                                  │
│     { name, description, parameters(JSON Schema) }       │
│                                                          │
│  2. AI 决策（模型内部）                                    │
│     根据用户意图 + 工具描述 → 选择工具 + 构造参数           │
│                                                          │
│  3. 工具调用（AI 回复中）                                  │
│     { tool_id, tool_name, arguments }                    │
│                                                          │
│  4. 工具结果（传回给 AI）                                  │
│     { tool_id, result_content }                          │
│                                                          │
│  5. 循环控制                                              │
│     有 tool call → 继续循环                               │
│     无 tool call → 结束循环                               │
└─────────────────────────────────────────────────────────┘
```

OpenClaw SDK 通过 **provider adapter 模式** 统一了这些差异：

```
OpenClaw 内部统一格式                各 Provider 的 API 格式
┌──────────────────┐
│ type: "toolCall"  │ ←→  Anthropic: type: "tool_use"
│ id: "xxx"         │ ←→  OpenAI:    tool_calls[].id
│ name: "search"    │ ←→  Gemini:    functionCall.name
│ arguments: {...}  │ ←→  各家各自的序列化方式
└──────────────────┘

┌──────────────────┐
│ role: "toolResult"│ ←→  Anthropic: type: "tool_result"
│ toolCallId: "xxx" │ ←→  OpenAI:    role: "tool"
│ content: [...]    │ ←→  Gemini:    functionResponse
└──────────────────┘
```

---

### 三、调用技巧和要求

#### 1. JSON Schema 的兼容性问题

**最大的坑**：不同提供商对 JSON Schema 的支持程度不同。

```
Anthropic (Claude):
  ✅ 支持完整 JSON Schema（type, enum, anyOf, oneOf, allOf, $ref...）
  ✅ 支持嵌套对象、数组
  ✅ 支持 optional 字段

OpenAI (GPT):
  ✅ 支持大部分 JSON Schema
  ⚠️ strict 模式下限制更严（不允许 additionalProperties）
  ⚠️ arguments 是 JSON 字符串，需要 JSON.parse()

Google (Gemini):
  ❌ 不支持 anyOf / oneOf / allOf / const
  ❌ 不支持 patternProperties / additionalProperties
  ❌ 不支持 $ref
  ⚠️ parametersJsonSchema（新）vs parameters（旧）行为不同
  → OpenClaw 专门做了 sanitizeToolsForGoogle() 清洗 schema
```

OpenClaw 的 CLAUDE.md 中明确警告：

```
Tool schema guardrails (google-antigravity):
avoid Type.Union in tool input schemas; no anyOf/oneOf/allOf.
Use stringEnum/optionalStringEnum for string lists.
```

**建议**：如果要跨模型，把 tool schema 限制在最简子集——只用 `type: "object"` + `properties` + `required`，不用高级 JSON Schema 特性。

#### 2. tool_choice 参数（控制 AI 是否必须调工具）

所有主流 API 都支持 `tool_choice` 参数：

```
tool_choice: "auto"     → AI 自己决定（默认，最常用）
tool_choice: "any"      → AI 必须调至少一个工具（强制 tool_use）
tool_choice: "none"     → AI 不能调任何工具（强制纯文本）
tool_choice: { type: "tool", name: "web_search" }  → 强制调指定工具（Anthropic）
tool_choice: { type: "function", function: { name: "web_search" } }  → 同上（OpenAI）
```

OpenClaw **默认用 auto**，不干预 AI 的工具选择。

#### 3. 并行工具调用

AI 可以在一次回复中返回**多个** tool_use：

```json
{
  "content": [
    { "type": "tool_use", "id": "toolu_1", "name": "read", "input": { "path": "a.ts" } },
    { "type": "tool_use", "id": "toolu_2", "name": "read", "input": { "path": "b.ts" } }
  ]
}
```

OpenClaw 的 agent loop 按**顺序**执行（不是并行），并且支持在工具执行之间检查 steering messages（用户中断）：

```javascript
for (let index = 0; index < toolCalls.length; index++) {
    // 执行当前工具
    result = await tool.execute(toolCall.id, validatedArgs, signal, ...);

    // 检查用户是否发了新消息（steering）
    if (getSteeringMessages) {
        const steering = await getSteeringMessages();
        if (steering.length > 0) {
            // 跳过剩余工具，优先处理用户新消息
            for (const skipped of toolCalls.slice(index + 1)) {
                results.push(skipToolCall(skipped, stream));
            }
            break;
        }
    }
}
```

#### 4. 工具结果的 token 消耗

工具结果会被放回消息历史，消耗 context window。OpenClaw 有专门的处理：

```
tool-result-truncation.ts     → 截断过长的工具结果
tool-result-context-guard.ts  → 防止工具结果撑爆 context
context-pruning/              → 智能裁剪旧的消息和工具结果
```

**建议**：工具结果尽量简洁。如果工具返回大量数据（如文件内容），考虑截断或摘要。

#### 5. 参数验证

LLM 生成的工具参数**不一定符合 schema**。OpenClaw 使用 AJV 做运行时验证：

```javascript
// validation.js
export function validateToolArguments(tool, toolCall) {
  const validate = ajv.compile(tool.parameters);
  const args = structuredClone(toolCall.arguments);
  if (validate(args)) {
    return args; // 验证通过，可能有类型转换
  }
  throw new Error(`Validation failed for tool "${toolCall.name}": ...`);
}
```

验证失败时抛出错误，agent loop 把错误信息作为 tool result 返回给 AI，AI 通常会修正参数重试。

---

### 四、哪些 LLM 支持 Tool Calling？

#### 原生支持（API 级别）

| 提供商        | 模型                         | 支持程度                 | 协议                         |
| ------------- | ---------------------------- | ------------------------ | ---------------------------- |
| **Anthropic** | Claude 3/3.5/4 全系列        | ✅ 完整支持              | Anthropic Messages           |
| **OpenAI**    | GPT-4o, GPT-4, GPT-3.5-turbo | ✅ 完整支持              | Chat Completions / Responses |
| **Google**    | Gemini 1.5/2.0 全系列        | ✅ 支持（schema 限制多） | Generative AI / Vertex       |
| **Meta**      | Llama 3.1+ (via 第三方)      | ⚠️ 通过 OpenAI 兼容 API  | Chat Completions             |
| **Mistral**   | Mistral Large/Medium         | ✅ 支持                  | OpenAI 兼容                  |
| **xAI**       | Grok                         | ✅ 支持                  | OpenAI 兼容                  |
| **DeepSeek**  | DeepSeek V2/V3               | ✅ 支持                  | OpenAI 兼容                  |
| **Cerebras**  | Llama 系列 (快速推理)        | ✅ 支持                  | OpenAI 兼容                  |
| **Amazon**    | Claude via Bedrock           | ✅ 支持                  | Bedrock Converse             |
| **Azure**     | GPT via Azure                | ✅ 支持                  | Azure OpenAI                 |

#### 不支持或有限支持

| 模型/提供商            | 情况                                           |
| ---------------------- | ---------------------------------------------- |
| 旧版 GPT-3.5           | 不支持 tool calling，只有旧版 function calling |
| 小型开源模型 (7B 以下) | 大多不支持或效果很差                           |
| 纯文本补全 API         | 无 tool calling 支持                           |
| 一些量化模型           | tool calling 能力退化严重                      |

#### OpenAI 兼容 API 生态

很多提供商实现了 **OpenAI Chat Completions 兼容 API**，这意味着只要你的代码支持 OpenAI 格式，就能接入大量提供商：

```
同一套代码可以对接:
  OpenAI → GPT 系列
  DeepSeek → DeepSeek V3
  Mistral → Mistral Large
  xAI → Grok
  Cerebras → Llama (加速)
  Together AI → 各种开源模型
  Groq → Llama/Mixtral (加速)
  Fireworks → 各种开源模型
  OpenRouter → 路由到任意模型
```

OpenClaw SDK 的 `openai-completions.js` 中有针对不同提供商的兼容性检测：

```javascript
function detectCompat(model) {
  const isZai = provider === "zai" || baseUrl.includes("api.z.ai");
  const isNonStandard =
    provider === "cerebras" ||
    provider === "xai" ||
    provider === "mistral" ||
    baseUrl.includes("deepseek.com") ||
    isZai;
  // 不同提供商的兼容性设置不同
  // 比如 strict mode 支持、streaming 行为等
}
```

---

### 五、对最小可迁移版本的建议

#### 如果只支持 Anthropic（最简单）

直接用 `@anthropic-ai/sdk`，tool calling 开箱即用：

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();
const response = await client.messages.create({
  model: "claude-sonnet-4-20250514",
  system: "你是 AI 助手",
  messages,
  tools: [
    {
      name: "web_search",
      description: "搜索互联网",
      input_schema: {
        type: "object",
        properties: { query: { type: "string" } },
        required: ["query"],
      },
    },
  ],
  // tool_choice: "auto",  // 默认值，可省略
});
```

#### 如果要跨模型（推荐方案）

用 **Vercel AI SDK** 或 **LiteLLM**，它们已经做了协议适配：

```typescript
// Vercel AI SDK 示例
import { generateText, tool } from "ai";
import { anthropic } from "@ai-sdk/anthropic";

const result = await generateText({
  model: anthropic("claude-sonnet-4-20250514"),
  tools: {
    web_search: tool({
      description: "搜索互联网",
      parameters: z.object({ query: z.string() }),
      execute: async ({ query }) => {
        /* ... */
      },
    }),
  },
  prompt: "帮我搜索最新的AI信息",
  maxSteps: 10, // 最多循环 10 轮（内置 agent loop）
});
```

#### 工具 Schema 的最佳实践

```typescript
// ✅ 安全的 schema（所有提供商兼容）
{
  type: "object",
  properties: {
    query: { type: "string", description: "搜索关键词" },
    count: { type: "number", description: "结果数量" },
  },
  required: ["query"]
}

// ❌ 不安全的 schema（Gemini 不支持）
{
  type: "object",
  properties: {
    mode: { anyOf: [{ const: "fast" }, { const: "deep" }] },  // ❌ anyOf + const
    options: { $ref: "#/definitions/SearchOptions" },           // ❌ $ref
  },
  additionalProperties: false,  // ❌ Gemini 可能不支持
}
```
