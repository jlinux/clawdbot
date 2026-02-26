# OpenClaw 消息处理流水线研究

## 目标

分析从用户发送消息到收到 AI 回复的完整链路，评估哪些模块可以迁移到其他项目中。

---

## 一、整体架构（5 层）

```
用户消息 (Telegram/Discord/Slack/WhatsApp/Signal/...)
    ↓
┌─────────────────────────────────────────────────┐
│  1. Channel 层 — 接收消息、去重、防抖            │
│     src/telegram/  src/discord/  src/slack/ ...  │
└──────────────────────┬──────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────┐
│  2. Routing 层 — 确定 agentId + sessionKey       │
│     src/routing/resolve-route.ts                 │
└──────────────────────┬──────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────┐
│  3. Dispatch 层 — 构建上下文、调度到 Agent        │
│     src/auto-reply/reply/dispatch-from-config.ts │
│     src/auto-reply/reply/get-reply.ts            │
└──────────────────────┬──────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────┐
│  4. Agent 层 — AI 对话循环 + 工具执行             │
│     src/agents/pi-embedded-runner/run.ts         │
│     src/agents/pi-embedded-runner/run/attempt.ts │
└──────────────────────┬──────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────┐
│  5. 回复投递层 — 格式化并发送到原始渠道           │
│     src/channels/reply-dispatcher.ts             │
│     各渠道的 reply-delivery.ts                   │
└─────────────────────────────────────────────────┘
```

---

## 二、各层详细分析

### 2.1 Channel 层：消息接收

**入口**：每个渠道有独立的 webhook/polling 处理器。

以 Telegram 为例（`src/telegram/webhook.ts`）：

```
HTTP POST /telegram-webhook
  → grammy webhookCallback 反序列化 Telegram Update
  → 去重（update_id 缓存，TTL 5分钟，上限 2000 条）
  → 防抖（文本片段 1.5s 合并，媒体组 500ms 合并，转发消息 80ms）
  → 调用 processMessage()
```

**关键设计**：

- **去重器** (`src/telegram/bot-updates.ts`)：基于 `update_id` 或 `message:{chatId}:{messageId}` 构造 key，LRU 缓存
- **防抖器** (`src/telegram/bot-handlers.ts`)：把快速连续发送的消息合并为一次处理，避免为每条碎片消息都触发一次 AI 调用
- **媒体组合并**：Telegram 把一组图片作为多个 update 发送，防抖器把它们合并

**可迁移价值**：防抖 + 去重模式对任何消息处理系统都适用。

---

### 2.2 Routing 层：路由到正确的 Agent

**核心函数**：`resolveAgentRoute()` (`src/routing/resolve-route.ts`)

```typescript
resolveAgentRoute({
  cfg,
  channel: "telegram",
  accountId: "default",
  peer: { kind: "direct" | "group", id: peerId },
  parentPeer,  // 线程继承
}) → ResolvedAgentRoute { agentId, sessionKey, matchedBy }
```

**路由优先级**（从高到低）：

| 优先级 | 绑定类型              | 说明             |
| ------ | --------------------- | ---------------- |
| 1      | `binding.peer`        | 直接匹配对话方   |
| 2      | `binding.peer.parent` | 线程的父对话     |
| 3      | `binding.guild+roles` | Guild + 成员角色 |
| 4      | `binding.guild`       | 仅 Guild         |
| 5      | `binding.team`        | Team ID          |
| 6      | `binding.account`     | 账号级配置       |
| 7      | `binding.channel`     | 渠道级兜底       |
| 8      | `default`             | 默认 agent       |

**Session Key 格式**：`{mainKey}:{channel}:{accountId}:{peerKind}:{peerId}`
例如：`main:telegram:default:direct:123456789`

**可迁移价值**：多级路由绑定 + session key 设计适合需要多租户 / 多 agent 的系统。

---

### 2.3 Dispatch 层：构建上下文并调度

**两步走**：

#### 步骤 1：构建 MsgContext

`buildTelegramMessageContext()` (`src/telegram/bot-message-context.ts`) 做：

- 调用 `resolveAgentRoute()` 确定路由
- 访问控制检查（allowFrom 白名单、DM 策略、@mention 要求）
- 组装消息体（文本、媒体、贴纸、位置等）
- 提及检测（显式 @bot、回复机器人消息 = 隐式提及）
- 返回统一的 `MsgContext` 对象

**MsgContext 核心字段**（50+ 字段，这里列最关键的）：

```typescript
{
  // 消息内容
  Body: string           // 带上下文/历史的完整消息
  BodyForAgent: string   // 给 agent 的干净文本
  RawBody: string        // 纯文本，无信封

  // 发送者信息
  From: string           // "telegram:123456"
  SessionKey: string     // 会话 key
  SenderName: string

  // 元数据
  Provider: string       // "telegram"
  ChatType: string       // "group" | "direct"
  WasMentioned: boolean

  // 媒体
  MediaPaths: string[]
  MediaTypes: string[]

  // 回复上下文
  ReplyToBody: string
  ReplyToSender: string

  // 群组/线程
  GroupSubject: string
  InboundHistory: Array<{ sender, body, timestamp }>
}
```

#### 步骤 2：调度到 Agent

```
dispatchReplyWithBufferedBlockDispatcher()
  → dispatchReplyFromConfig()
    → getReplyFromConfig()
```

`getReplyFromConfig()` (`src/auto-reply/reply/get-reply.ts`) 做：

1. 从 sessionKey 解析 agentId
2. 确保 agent workspace 存在
3. 初始化 session state（加载历史对话）
4. 选择 model（provider + model name）
5. 媒体理解（图片识别等）
6. 调用 `runPreparedReply()` 执行 agent

**可迁移价值**：MsgContext 统一抽象是关键设计——无论消息来自哪个渠道，到达 agent 时格式一致。

---

### 2.4 Agent 层：AI 对话循环（最核心）

**入口**：`runEmbeddedPiAgent()` (`src/agents/pi-embedded-runner/run.ts`)

#### 2.4.1 准备阶段

```
1. 解析 model + provider（默认 Anthropic Claude）
2. 解析认证 profile（API Key / OAuth / AWS SDK）
3. 检查 context window 容量
4. 加载会话历史（JSONL 文件）
5. 组装 system prompt
6. 注册工具（tools）
7. 订阅流式事件
```

**System Prompt 组成**：

- 基础 system prompt
- workspace 说明（`.openclaw.md`）
- 渠道能力描述（反应、内联按钮等）
- 运行时信息（机器名、OS、Node 版本）
- 工具描述
- 技能 prompt
- 用户时区 + 当前时间

#### 2.4.2 核心循环

```typescript
// 这是实际调用 AI 的关键一行
await activeSession.prompt(effectivePrompt, { images });
```

底层执行流程：

```
activeSession.prompt()
  → agent.streamFn()           // 包装后的流式函数
    → streamSimple()           // @mariozechner/pi-ai
      → Anthropic/OpenAI API   // 实际 HTTP 请求

  ← 流式接收 response
    ├── 纯文本 → onPartialReply 回调
    ├── tool_use → 执行工具 → tool_result → 追加到 session → 继续循环
    └── 无更多 tool_use → 结束
```

**这就是 agentic loop（代理循环）**：

1. 发送 prompt 给 AI
2. AI 返回文本 + 可能的 tool_use 请求
3. 如果有 tool_use → 执行工具 → 把结果放回对话 → 回到步骤 1
4. 如果没有 tool_use → 对话结束

#### 2.4.3 工具系统

**工具创建**：`createOpenClawCodingTools()` (`src/agents/pi-tools.ts`)

内置工具：
| 工具名 | 功能 |
|--------|------|
| `exec` / `bash` | 执行 shell 命令 |
| `read` | 读取文件 |
| `write` | 写入文件 |
| `edit` | 编辑文件指定范围 |
| `sessions_send` | 发送消息到其他 session |
| `message` | 发送消息到当前渠道 |

工具执行流程：

```
AI 返回 tool_use
  → 查找工具注册表
  → 验证参数
  → 执行工具 handler
  → 获取结果
  → 发射 onToolResult 回调（给前端流式显示）
  → 追加 tool_result 到 session
  → 继续下一轮 AI 调用
```

#### 2.4.4 错误处理 & Failover

- **认证失败** → 轮换 auth profile
- **限流** → 尝试下一个 profile
- **上下文溢出** → 自动压缩（compaction），最多 3 次
- **工具结果过大** → 截断
- **超时** → 中止，不重试

#### 2.4.5 会话持久化

- **格式**：JSONL（每行一个 JSON 对象）
- **路径**：`~/.openclaw/sessions/<sessionId>.jsonl`
- **内容**：`message`、`tool_use`、`tool_result`、系统条目
- 每次工具执行后立即追加写入，带文件锁

**可迁移价值**：整个 agent 层是最有价值的迁移目标。核心模式是 prompt → stream → tool_use loop → persist session。

---

### 2.5 回复投递层

Agent 完成后：

```
buildEmbeddedRunPayloads()   // 构建回复 payload
  → ReplyDispatcher          // 异步投递管理器
    → 渠道特定格式化
      ├── 文本分块（各渠道字符限制不同）
      ├── Markdown 转换
      ├── 媒体附件处理
      └── 渠道 API 发送
    → 更新状态反应（✅ 完成）
```

---

## 三、Gateway 通信协议

Gateway 用 WebSocket JSON 帧通信：

```json
// 请求
{ "type": "req", "id": "req-1", "method": "send", "params": { "to": "...", "message": "..." } }

// 响应
{ "type": "res", "id": "req-1", "ok": true, "payload": { "messageId": "..." } }
```

Gateway 方法分发：

```
socket.on("message")
  → parseJSON
  → validateRequestFrame()
  → handleGatewayRequest()
    → authorizeGatewayMethod()     // 权限检查
    → 限流检查
    → 查找 handler（插件优先，否则 core）
    → 执行 handler
```

---

## 四、插件系统

插件通过 `register()` 注册能力：

```typescript
// extensions/msteams/index.ts 示例
export function register(api: PluginAPI) {
  api.channel.register("msteams", outboundAdapter);
  api.gateway.registerMethod("msteams.status", statusHandler);
  api.hooks.on("before_model_resolve", overrideModel);
}
```

**插件能提供的能力**：

- Channel（新的消息渠道）
- Memory backend（向量数据库等）
- Tools / Skills（新工具）
- Hooks（生命周期钩子）
- HTTP Routes（新的 API 端点）
- Gateway Methods（WebSocket 方法）

**加载方式**：jiti（零配置动态 ESM import），支持热加载。

---

## 五、可迁移性分析

### 最适合迁移的模块（按价值排序）

| 模块                                      | 复杂度 | 迁移价值   | 依赖关系                                             |
| ----------------------------------------- | ------ | ---------- | ---------------------------------------------------- |
| **Agent 循环** (prompt → tool_use → loop) | 中     | ⭐⭐⭐⭐⭐ | `@mariozechner/pi-ai`, `@mariozechner/pi-agent-core` |
| **MsgContext 统一抽象**                   | 低     | ⭐⭐⭐⭐   | 无外部依赖                                           |
| **Session 持久化** (JSONL)                | 低     | ⭐⭐⭐⭐   | `@mariozechner/pi-coding-agent` SessionManager       |
| **路由系统** (多级绑定)                   | 中     | ⭐⭐⭐     | 配置系统                                             |
| **防抖 + 去重**                           | 低     | ⭐⭐⭐     | 无外部依赖                                           |
| **Model failover**                        | 中     | ⭐⭐⭐     | 认证系统                                             |
| **工具注册表**                            | 中     | ⭐⭐⭐     | Agent SDK                                            |
| **插件系统** (jiti)                       | 高     | ⭐⭐       | jiti, 配置系统                                       |
| **渠道适配器**                            | 高     | ⭐⭐       | 各渠道 SDK (grammy, discord.js 等)                   |
| **Gateway 协议**                          | 高     | ⭐         | Express, WebSocket                                   |

### 最小可行迁移方案

要在另一个项目中复现 OpenClaw 的核心 AI 对话能力，最少需要：

```
1. MsgContext 统一消息格式      ← 自己定义，50 行左右
2. Session JSONL 读写           ← 用 SessionManager 或自己实现
3. Agent 循环                   ← 核心依赖：@mariozechner/pi-ai
4. 工具注册 + 执行              ← 定义 tool schema + handler
5. 一个消息入口                 ← HTTP webhook / WebSocket / CLI
6. 一个回复出口                 ← 发送回原始渠道
```

### 关键外部依赖

| 包名                                    | 作用                      | 是否可替换                     |
| --------------------------------------- | ------------------------- | ------------------------------ |
| `@mariozechner/pi-ai`                   | 流式调用 AI provider API  | 可用 Anthropic/OpenAI SDK 替换 |
| `@mariozechner/pi-agent-core`           | Agent 循环框架            | 可自己实现 agentic loop        |
| `@mariozechner/pi-coding-agent`         | Session 管理 + 编码 agent | 可自己实现 JSONL session       |
| `grammy` / `discord.js` / `@slack/bolt` | 渠道 SDK                  | 按需选择                       |
| `jiti`                                  | 插件动态加载              | 可用 `import()` 替换           |

---

## 六、关键文件索引

| 文件                                           | 作用                             |
| ---------------------------------------------- | -------------------------------- |
| `src/telegram/webhook.ts`                      | Telegram 消息入口                |
| `src/telegram/bot-handlers.ts`                 | 防抖 + 消息处理                  |
| `src/telegram/bot-message-context.ts`          | 构建 MsgContext                  |
| `src/telegram/bot-message-dispatch.ts`         | 调度到 agent                     |
| `src/routing/resolve-route.ts`                 | 路由解析（多级绑定）             |
| `src/routing/session-key.ts`                   | Session key 构造                 |
| `src/auto-reply/reply/dispatch-from-config.ts` | 从配置调度回复                   |
| `src/auto-reply/reply/get-reply.ts`            | Agent 调用入口                   |
| `src/agents/pi-embedded-runner/run.ts`         | Agent 主编排器                   |
| `src/agents/pi-embedded-runner/run/attempt.ts` | 单次 attempt（session + prompt） |
| `src/agents/pi-embedded-subscribe.ts`          | 流式事件订阅                     |
| `src/agents/pi-tools.ts`                       | 工具注册表                       |
| `src/agents/model-auth.ts`                     | API key 解析 + 认证              |
| `src/channels/reply-dispatcher.ts`             | 回复投递管理器                   |
| `src/gateway/server-methods.ts`                | Gateway 方法分发                 |
| `src/plugins/loader.ts`                        | 插件加载器                       |
