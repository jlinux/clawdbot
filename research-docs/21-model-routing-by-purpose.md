# OpenClaw 不同用途的 LLM 分配机制

## 概述

OpenClaw 并不是只用一个 LLM 做所有事情。它根据不同用途将不同模型分配到不同的子系统中：**主对话**用大语言模型、**图片分析**用 Vision 模型、**语音转文字**用 Whisper/Deepgram、**嵌入生成**用 Embedding 模型、**TTS**用语音合成模型等。每种用途有独立的配置路径、默认值和回退链。

---

## 模型分配全景图

```
┌─────────────────────────────────────────────────────────────────────┐
│                    OpenClaw 模型分配架构                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌────────────┐ │
│  │ 主对话模型   │  │ Vision 模型  │  │ Embedding   │  │ STT 模型   │ │
│  │             │  │             │  │ 模型         │  │            │ │
│  │ Claude      │  │ Claude/     │  │ text-embed-  │  │ Whisper/   │ │
│  │ Opus 4.6   │  │ GPT-4o/     │  │ ding-3-small │  │ Deepgram/  │ │
│  │             │  │ MiniMax-VL  │  │ /gemini-emb  │  │ Groq       │ │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └─────┬──────┘ │
│         │                │                │                │        │
│  ┌──────┴──────┐  ┌──────┴──────┐  ┌──────┴──────┐  ┌─────┴──────┐ │
│  │ TTS 模型    │  │ 媒体理解    │  │ Compaction  │  │ 子 Agent   │ │
│  │             │  │ 模型        │  │ 模型        │  │ 模型       │ │
│  │ Edge TTS/   │  │ gpt-5-mini/ │  │ (继承主模型) │  │ (可独立    │ │
│  │ OpenAI TTS/ │  │ gemini-3-   │  │             │  │  配置)     │ │
│  │ ElevenLabs  │  │ flash       │  │             │  │            │ │
│  └─────────────┘  └─────────────┘  └─────────────┘  └────────────┘ │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 1. 主对话模型（Primary Conversation Model）

### 用途

处理用户消息、执行工具调用、生成回复——Agent Loop 的核心。

### 配置路径

```yaml
agents:
  defaults:
    model:
      primary: "anthropic/claude-opus-4-6" # 主模型
      fallbacks: # 回退链
        - "openai/gpt-4-turbo"
        - "google/gemini-2.0-flash"

  list:
    - id: "my-agent"
      model:
        primary: "openai/gpt-4-turbo" # Agent 专属覆盖
        fallbacks: ["anthropic/claude-opus-4-6"]
```

### 解析链

**关键函数**：`resolveDefaultModelForAgent()`（`src/agents/model-selection.ts:342-370`）

```
[1] 插件钩子: before_model_resolve（最高优先级，运行时覆盖）
    ↓
[2] Agent 专属: agents.list[agentId].model.primary
    ↓
[3] 全局默认: agents.defaults.model.primary
    ↓
[4] 别名查找: agents.defaults.models.{alias}（如 "opus" → "anthropic/claude-opus-4-6"）
    ↓
[5] 解析 "provider/model" 格式 + 规范化
    ↓
[6] 硬编码默认: anthropic/claude-opus-4-6
```

### "provider/model" 格式

所有模型引用使用 `"provider/model"` 格式：

```
"anthropic/claude-opus-4-6"
"openai/gpt-4-turbo"
"google/gemini-2.0-flash"
"openrouter/meta-llama/llama-3-70b"
```

### Provider 别名规范化

`src/agents/model-selection.ts:39-61`

| 输入                         | 规范化为           |
| ---------------------------- | ------------------ |
| `"z.ai"`, `"z-ai"`           | `"zai"`            |
| `"qwen"`                     | `"qwen-portal"`    |
| `"bedrock"`, `"aws-bedrock"` | `"amazon-bedrock"` |
| `"bytedance"`, `"doubao"`    | `"volcengine"`     |

### Model 别名

| 别名           | 实际模型              |
| -------------- | --------------------- |
| `"opus-4.6"`   | `"claude-opus-4-6"`   |
| `"sonnet-4.6"` | `"claude-sonnet-4-6"` |

### 模型白名单

```yaml
agents:
  defaults:
    models:
      "anthropic/claude-opus-4-6":
        alias: "opus"
        params: { temperature: 0.7 }
      "openai/gpt-4-turbo":
        alias: "gpt4"
```

- 白名单为空 → 允许所有模型
- 白名单非空 → 仅允许列出的模型

### Thinking/推理模式

```yaml
agents:
  defaults:
    thinkingDefault: "low" # "off"|"minimal"|"low"|"medium"|"high"|"xhigh"
```

`resolveThinkingDefault()`（`model-selection.ts:554-571`）：

- 配置有值 → 使用配置
- 模型 `reasoning: true` → 返回 `"low"`
- 否则 → `"off"`

---

## 2. 图片/Vision 模型（Image Model）

### 用途

`image` 工具调用 Vision API 分析用户上传的图片。

### 配置路径

```yaml
agents:
  defaults:
    imageModel:
      primary: "anthropic/claude-opus-4-6"
      fallbacks:
        - "openai/gpt-4o"
```

### 解析逻辑

**关键函数**：`resolveImageModelConfigForTool()`（`src/agents/tools/image-tool.ts:85-177`）

```
[1] 明确配置: agents.defaults.imageModel.primary
    ↓ （未配置时）
[2] 自动配对: 根据主模型的 provider 选择对应 Vision 模型
    ↓
[3] 跨 Provider 回退
```

### 自动配对规则

当 `imageModel` 未明确配置时，根据主对话模型的 provider **自动匹配** Vision 模型：

| 主模型 Provider | 自动选择的 Vision 模型      |
| --------------- | --------------------------- |
| **Anthropic**   | `anthropic/claude-opus-4-6` |
| **OpenAI**      | `openai/gpt-5-mini`         |
| **MiniMax**     | `minimax/MiniMax-VL-01`     |
| **ZAI**         | `zai/glm-4.6v`              |
| **其他**        | 回退到 OpenAI → Anthropic   |

### 回退链

```
Provider 专属 Vision 模型
  ↓ 失败
OpenAI Vision
  ↓ 失败
Anthropic Vision
```

通过 `runWithImageModelFallback()`（`model-fallback.ts:446-503`）执行。

---

## 3. Compaction/摘要模型

### 用途

上下文压缩时，将长对话历史摘要为简短版本。

### 模型选择

**没有独立的 compaction 模型配置**——**直接继承主对话模型**。

**关键函数**：`summarizeInStages()`、`summarizeWithFallback()`（`src/agents/compaction.ts:208-337`）

```typescript
// compaction.ts 接收当前对话模型
async function summarizeWithFallback(params: {
  model: NonNullable<ExtensionContext["model"]>; // ← 继承自主模型
  messages: Message[];
  // ...
});
```

**设计原因**：摘要质量依赖于模型理解上下文的能力，使用与对话相同的模型可以保证最佳摘要质量。

---

## 4. Embedding 模型（向量嵌入）

### 用途

为 Memory 系统的语义搜索生成向量嵌入。

### 配置路径

```yaml
agents:
  defaults:
    memorySearch:
      provider: "openai" # "openai"|"gemini"|"local"|"voyage"|"mistral"|"auto"
      model: "text-embedding-3-small"
      fallback: "gemini" # 回退 provider
```

### 默认模型

| Provider    | 默认模型                            | 来源文件              |
| ----------- | ----------------------------------- | --------------------- |
| **OpenAI**  | `text-embedding-3-small`            | `memory-search.ts:81` |
| **Gemini**  | `gemini-embedding-001`              | `memory-search.ts:82` |
| **Voyage**  | `voyage-4-large`                    | `memory-search.ts:83` |
| **Mistral** | `mistral-embed`                     | `memory-search.ts:84` |
| **Local**   | `embeddinggemma-300m-qat-q8_0-GGUF` | 本地 node-llama-cpp   |

### 解析逻辑

`createEmbeddingProvider()`（`src/memory/embeddings.ts:144-260`）

```
[1] 配置的 provider + model
    ↓ 不可用
[2] fallback provider
    ↓ 不可用
[3] FTS-only 模式（纯关键词搜索，无向量）
```

### 关键特性

- **完全独立于主对话模型**——embedding 模型是专用的小型向量模型
- **本地回退**：可配置 `provider: "local"` 使用本地 GGUF 模型
- **不同维度**：每个 provider 的嵌入维度不同，不可混用

---

## 5. STT（语音转文字）模型

### 用途

将用户发送的语音消息转录为文本。

### 配置路径

```yaml
tools:
  media:
    audio:
      models:
        - provider: "openai"
          model: "gpt-4o-mini-transcribe"
        - provider: "groq"
          model: "whisper-large-v3-turbo"
        - type: "cli"
          command: "whisper"
```

### 默认模型

| Provider     | 默认模型                 | 来源文件         |
| ------------ | ------------------------ | ---------------- |
| **OpenAI**   | `gpt-4o-mini-transcribe` | `defaults.ts:33` |
| **Groq**     | `whisper-large-v3-turbo` | `defaults.ts:31` |
| **Deepgram** | `nova-3`                 | `defaults.ts:34` |
| **Mistral**  | `voxtral-mini-latest`    | `defaults.ts:35` |

### 解析逻辑

`resolveModelEntries()`（`src/media-understanding/resolve.ts:101-139`）

```
[1] 配置的 models[] 有序列表
    ↓ 全部不可用
[2] 活跃 provider 的默认模型
    ↓ 不可用
[3] 本地 CLI 二进制（sherpa-onnx、whisper-cpp、whisper）
```

### 关键特性

- **有序列表**：models[] 按顺序尝试，第一个可用的就用
- **CLI 回退**：支持本地 whisper 二进制作为最后手段
- **API Key 检测**：自动跳过没有 API key 的 provider

---

## 6. TTS（文字转语音）模型

### 用途

将 AI 回复转换为语音消息。

### 配置路径

```yaml
agents:
  defaults:
    tts:
      provider: "edge" # "edge"|"openai"|"elevenlabs"
      summaryModel: "anthropic/claude-opus-4-6" # 超长文本摘要用的模型
```

### 默认模型

| Provider       | 默认模型/语音                                             | 费用 |
| -------------- | --------------------------------------------------------- | ---- |
| **Edge TTS**   | `en-US-MichelleNeural`                                    | 免费 |
| **OpenAI**     | `gpt-4o-mini-tts` / 语音 `alloy`                          | 付费 |
| **ElevenLabs** | `eleven_multilingual_v2` / 语音 ID `pMsXgVXv3BLzUgSXRplE` | 付费 |

### TTS 摘要模型

当文本超过 TTS 最大长度（默认 4096 字符）时，需要先用 LLM 摘要。

`resolveSummaryModelRef()`（`src/tts/tts-core.ts:390-413`）：

- 配置了 `summaryModel` → 使用配置的模型
- 否则 → 回退到主对话模型

---

## 7. 媒体理解模型（Media Understanding）

### 用途

自动分析用户发送的图片/视频内容（非 `image` 工具，而是消息预处理）。

### 配置路径

```yaml
tools:
  media:
    image:
      models:
        - provider: "openai"
          model: "gpt-5-mini"
        - provider: "anthropic"
          model: "claude-opus-4-6"
    video:
      models:
        - provider: "google"
          model: "gemini-3-flash-preview"
```

### 默认模型

| 类型     | Provider  | 默认模型                 |
| -------- | --------- | ------------------------ |
| **图片** | OpenAI    | `gpt-5-mini`             |
| **图片** | Anthropic | `claude-opus-4-6`        |
| **图片** | Google    | `gemini-3-flash-preview` |
| **图片** | MiniMax   | `MiniMax-VL-01`          |
| **图片** | ZAI       | `glm-4.6v`               |
| **视频** | Google    | `gemini-3-flash-preview` |
| **视频** | Moonshot  | 自动检测                 |

### 与 image 工具的区别

| 特性     | Media Understanding | image 工具                   |
| -------- | ------------------- | ---------------------------- |
| 触发方式 | 自动（消息预处理）  | AI 主动调用                  |
| 时机     | Agent Loop 之前     | Agent Loop 中                |
| 输出     | 文本描述注入消息    | 工具结果返回给 AI            |
| 配置     | `tools.media.*`     | `agents.defaults.imageModel` |

---

## 8. 子 Agent 模型

### 用途

子 Agent 可以使用与父 Agent 不同的模型。

### 配置路径

```yaml
agents:
  defaults:
    subagents:
      model: "anthropic/claude-sonnet-4-6" # 子 Agent 默认模型
      thinking: "off"

  list:
    - id: "my-agent"
      subagents:
        model: "openai/gpt-4-turbo" # Agent 专属子 Agent 模型
```

### 解析链

**关键函数**：`resolveSubagentSpawnModelSelection()`（`model-selection.ts:384-402`）

```
[1] 请求参数: sessions_spawn(model="openai/gpt-4-turbo")  ← 最高优先级
    ↓
[2] Agent 专属子 Agent 配置: agents.list[agentId].subagents.model
    ↓
[3] 全局子 Agent 默认: agents.defaults.subagents.model
    ↓
[4] 继承主 Agent 模型: resolveDefaultModelForAgent()
```

### 使用场景

```
主 Agent (Claude Opus 4.6 — 强推理)
  ├── 子 Agent 1: Sonnet 4.6 (快速信息搜索)
  ├── 子 Agent 2: GPT-4 Turbo (特定任务)
  └── 子 Agent 3: 继承 Opus 4.6 (复杂分析)
```

---

## 9. 插件钩子覆盖（Runtime Override）

### before_model_resolve 钩子

插件可以在运行时动态覆盖模型选择：

```typescript
// plugins/types.ts:335-345
api.on("before_model_resolve", (event, ctx) => {
  // 根据消息内容动态选择模型
  if (event.prompt.includes("代码")) {
    return { modelOverride: "claude-opus-4-6", providerOverride: "anthropic" };
  }
  if (event.prompt.includes("翻译")) {
    return { modelOverride: "gpt-4-turbo", providerOverride: "openai" };
  }
});
```

**优先级**：钩子按 priority 排序，第一个返回覆盖的获胜。

---

## 10. Provider 配置

### 内置 Provider

所有 provider 的连接信息通过 `models.providers` 配置：

```yaml
models:
  mode: "merge" # "merge"(默认): 合并内置+自定义; "replace": 仅自定义
  providers:
    anthropic:
      baseUrl: "https://api.anthropic.com"
      apiKey: "sk-..."
      models:
        - id: "claude-opus-4-6"
          name: "Claude Opus 4.6"
          reasoning: true
          input: ["text", "image"]
          contextWindow: 200000
          maxTokens: 8192
          cost: { input: 15, output: 75, cacheRead: 1.5, cacheWrite: 18.75 }
    openai:
      baseUrl: "https://api.openai.com/v1"
      apiKey: "sk-..."
      models: [...]
```

### API Key 来源

每个 provider 的 API key 可来自：

1. 配置文件 `models.providers.<provider>.apiKey`
2. 环境变量 `{PROVIDER}_API_KEY`
3. Auth Profile 系统（多 key 轮换）

---

## 11. 模型分配总结表

| 用途                | 配置路径                          | 默认模型                 | 解析函数                               | 回退策略                         |
| ------------------- | --------------------------------- | ------------------------ | -------------------------------------- | -------------------------------- |
| **主对话**          | `agents.defaults.model`           | `claude-opus-4-6`        | `resolveDefaultModelForAgent()`        | fallbacks 链 → 硬编码            |
| **图片/Vision**     | `agents.defaults.imageModel`      | 自动配对主模型 provider  | `resolveImageModelConfigForTool()`     | Provider 回退 + 跨 Provider      |
| **Compaction 摘要** | 无（继承主模型）                  | 同主对话                 | 直接传入                               | 无                               |
| **Embedding**       | `agents.defaults.memorySearch`    | `text-embedding-3-small` | `resolveMemorySearchConfig()`          | fallback provider → FTS-only     |
| **STT**             | `tools.media.audio.models[]`      | Provider 各异            | `resolveModelEntries()`                | 有序列表 + CLI 二进制            |
| **TTS**             | `agents.defaults.tts`             | Edge TTS（免费）         | 配置驱动                               | Provider 默认                    |
| **TTS 摘要**        | `tts.summaryModel`                | 继承主模型               | `resolveSummaryModelRef()`             | 主模型                           |
| **媒体理解**        | `tools.media.{image\|video}`      | Provider 各异            | `resolveModelEntries()`                | 自动检测 + 回退链                |
| **子 Agent**        | `agents.defaults.subagents.model` | 继承主模型               | `resolveSubagentSpawnModelSelection()` | 请求参数 → Agent → 全局 → 主模型 |
| **Per-Agent**       | `agents.list[id].model`           | 继承全局默认             | `resolveAgentEffectiveModelPrimary()`  | 全局 → 硬编码                    |
| **Thinking**        | `agents.defaults.thinkingDefault` | 模型依赖                 | `resolveThinkingDefault()`             | reasoning=true → "low"           |

---

## 12. 关键设计原则

### 原则 1：用途隔离

每种用途有**独立的配置路径**和**独立的模型解析链**。图片分析不会意外用了主对话模型的回退策略。

### 原则 2：合理的继承

- **Compaction** 和 **TTS 摘要**继承主对话模型（需要相同的理解能力）
- **子 Agent** 默认继承但可覆盖（灵活性）
- **Embedding** 完全独立（不同类型的模型）

### 原则 3：自动配对

图片模型会根据主模型的 provider **自动选择**同 provider 的 Vision 模型，减少配置负担。

### 原则 4：分层回退

每种用途都有多层回退：配置 → 自动检测 → 跨 Provider → 硬编码默认。

### 原则 5：运行时可覆盖

`before_model_resolve` 钩子允许插件在运行时根据消息内容动态切换模型。

---

## 13. 关键文件索引

| 文件                                     | 职责                                      |
| ---------------------------------------- | ----------------------------------------- |
| `src/agents/model-selection.ts`          | 主模型解析、别名、白名单、子 Agent 模型   |
| `src/agents/model-fallback.ts`           | 模型回退链执行                            |
| `src/agents/defaults.ts`                 | 硬编码默认值（anthropic/claude-opus-4-6） |
| `src/agents/agent-scope.ts`              | Per-Agent 模型覆盖解析                    |
| `src/agents/tools/image-tool.ts:85-177`  | Vision 模型自动配对                       |
| `src/agents/compaction.ts`               | 摘要模型（继承主模型）                    |
| `src/agents/memory-search.ts`            | Embedding 模型配置解析                    |
| `src/memory/embeddings.ts`               | Embedding provider 创建和回退             |
| `src/media-understanding/defaults.ts`    | STT/Vision 默认模型                       |
| `src/media-understanding/resolve.ts`     | 媒体理解模型解析                          |
| `src/tts/tts.ts`                         | TTS provider 默认值                       |
| `src/tts/tts-core.ts:390-413`            | TTS 摘要模型解析                          |
| `src/config/types.agent-defaults.ts`     | Agent 默认配置类型                        |
| `src/config/types.models.ts`             | Provider/Model 配置类型                   |
| `src/config/zod-schema.agent-model.ts`   | 模型配置 Schema                           |
| `src/plugins/types.ts:335-345`           | before_model_resolve 钩子                 |
| `src/agents/pi-embedded-runner/model.ts` | 运行时模型实例化                          |

---

## 14. 对最小可迁移版本的启示

### 必须实现

```typescript
// 1. 用途级模型映射
const MODEL_CONFIG = {
  chat: "anthropic/claude-opus-4-6", // 主对话
  embedding: "openai/text-embedding-3-small", // 向量嵌入
  stt: "openai/whisper-1", // 语音转文字
};

// 2. 简单的用途路由
function getModelForPurpose(purpose: keyof typeof MODEL_CONFIG): string {
  return MODEL_CONFIG[purpose];
}

// 3. 调用时按用途选模型
const chatResponse = await callModel(getModelForPurpose("chat"), messages);
const embedding = await getEmbedding(getModelForPurpose("embedding"), text);
const transcript = await transcribe(getModelForPurpose("stt"), audioBuffer);
```

### 可以简化的地方

| OpenClaw 做法                           | 最小版本         |
| --------------------------------------- | ---------------- |
| 每种用途独立配置路径 + 解析链           | 简单字典映射     |
| Vision 自动配对主模型 provider          | 固定 Vision 模型 |
| Provider 别名规范化（10+ 别名）         | 不需要           |
| 模型白名单 + 别名系统                   | 不需要           |
| before_model_resolve 运行时钩子         | 不需要           |
| 多层回退（配置→自动→跨Provider→硬编码） | 单一回退         |
| Auth Profile 轮换（多 API key）         | 单一 API key     |
| CLI 二进制回退（whisper 本地）          | 仅 API           |
