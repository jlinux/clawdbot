# 消息执行路径模拟："帮我搜索最新的AI信息"

## 场景设定

- **用户**：通过 Telegram 私聊发送消息
- **消息内容**："帮我搜索最新的AI信息"
- **Agent**：默认 agent，模型为 Claude Sonnet

---

## 完整执行路径

### 阶段 1：消息接收（~0ms）

```
Telegram 服务器 → HTTP POST /telegram-webhook
```

**文件**：`src/telegram/webhook.ts`

grammy 库反序列化 Telegram Update：

```json
{
  "update_id": 98765432,
  "message": {
    "message_id": 1234,
    "from": { "id": 55512345, "first_name": "Yong" },
    "chat": { "id": 55512345, "type": "private" },
    "text": "帮我搜索最新的AI信息",
    "date": 1740480000
  }
}
```

### 阶段 2：去重 + 防抖（~1500ms 等待窗口）

**文件**：`src/telegram/bot-updates.ts` → `src/telegram/bot-handlers.ts`

```
去重检查: key = "message:55512345:1234" → 不在缓存中 → 通过
防抖器: 等待 1.5 秒看是否有后续消息碎片
  └→ 1.5 秒内无新消息 → flush → 调用 processMessage()
```

### 阶段 3：构建消息上下文（~10ms）

**文件**：`src/telegram/bot-message-context.ts`

```
buildTelegramMessageContext()
  ├─ resolveAgentRoute()
  │   channel: "telegram"
  │   accountId: "default"
  │   peer: { kind: "direct", id: "55512345" }
  │   → 无特殊绑定，命中 default
  │   → agentId: "main"
  │   → sessionKey: "main:telegram:default:direct:55512345"
  │
  ├─ 访问控制: allowFrom 白名单 → 通过
  │
  ├─ 消息解析:
  │   text: "帮我搜索最新的AI信息"
  │   media: 无
  │   提及检测: 私聊 → 自动视为提及
  │
  └─ 构建 MsgContext:
      {
        Body: "帮我搜索最新的AI信息",
        BodyForAgent: "帮我搜索最新的AI信息",
        RawBody: "帮我搜索最新的AI信息",
        From: "telegram:55512345",
        SessionKey: "main:telegram:default:direct:55512345",
        SenderName: "Yong",
        Provider: "telegram",
        ChatType: "direct",
        WasMentioned: true,
        MediaPaths: [],
      }
```

### 阶段 4：调度到 Agent（~5ms）

**文件**：`src/telegram/bot-message-dispatch.ts` → `src/auto-reply/reply/dispatch-from-config.ts`

```
dispatchTelegramMessage()
  ├─ 设置 typing indicator（Telegram 显示"正在输入..."）
  ├─ 设置 status reaction: 💭 (thinking)
  │
  └─ dispatchReplyWithBufferedBlockDispatcher()
      └─ dispatchReplyFromConfig()
          └─ getReplyFromConfig()
              ├─ agentId: "main" (从 sessionKey 解析)
              ├─ workspace: ~/.openclaw/agents/main/
              ├─ 加载 session: ~/.openclaw/sessions/main-telegram-default-direct-55512345.jsonl
              ├─ model: { provider: "anthropic", id: "claude-sonnet-4-20250514" }
              └─ → runPreparedReply()
```

### 阶段 5：Agent 准备（~50ms）

**文件**：`src/agents/pi-embedded-runner/run.ts`

```
runEmbeddedPiAgent()
  ├─ 解析 model: anthropic / claude-sonnet-4-20250514
  ├─ 解析 auth: API key from ~/.openclaw/credentials/
  ├─ context window: 200k tokens → 充足
  │
  ├─ 注册工具（createOpenClawCodingTools）:
  │   ┌─────────────────┬────────────────────────────────────────┐
  │   │ 工具名           │ 说明                                   │
  │   ├─────────────────┼────────────────────────────────────────┤
  │   │ exec            │ 执行 shell 命令                        │
  │   │ read            │ 读取文件                               │
  │   │ write           │ 写入文件                               │
  │   │ edit            │ 编辑文件                               │
  │   │ process         │ 管理后台进程                            │
  │   │ web_search  ★   │ 网络搜索（Brave/Perplexity/Grok等）     │
  │   │ web_fetch   ★   │ 抓取 URL 内容（HTML→markdown）          │
  │   │ browser     ★   │ 浏览器自动化                            │
  │   │ message         │ 发送消息到渠道                          │
  │   │ memory_search   │ 搜索记忆库                              │
  │   │ memory_get      │ 获取记忆内容                            │
  │   │ image           │ 图片分析                               │
  │   │ cron            │ 定时任务                               │
  │   │ sessions_send   │ 发送消息到其他 session                  │
  │   │ sessions_spawn  │ 创建子 agent                           │
  │   │ tts             │ 文本转语音                              │
  │   │ canvas          │ Canvas UI 控制                         │
  │   │ nodes           │ 远程节点控制                            │
  │   │ gateway         │ 网关函数调用                            │
  │   └─────────────────┴────────────────────────────────────────┘
  │   ★ = 本次会用到的工具
  │
  ├─ 组装 system prompt:
  │   "你是 OpenClaw，一个 AI 助手..."
  │   + 工具列表及描述
  │   + "Tool Call Style: 不要叙述常规工具调用..."
  │   + 运行时信息（OS, 时区, 当前时间）
  │   + workspace notes
  │
  └─ 订阅流式事件（onTextDelta, onToolStart, onToolEnd...）
```

### 阶段 6：第一轮 AI 调用（~2000ms）

**文件**：`src/agents/pi-embedded-runner/run/attempt.ts`

```
activeSession.prompt("帮我搜索最新的AI信息")
  │
  └─ → Anthropic Messages API (streaming)

请求体（简化）:
{
  model: "claude-sonnet-4-20250514",
  system: "<system prompt with tool descriptions>",
  messages: [
    ...历史消息...,
    { role: "user", content: "帮我搜索最新的AI信息" }
  ],
  tools: [
    {
      name: "web_search",
      description: "Search the web (Brave API). Returns web search results with titles, URLs, and snippets.",
      input_schema: {
        type: "object",
        properties: {
          query: { type: "string", description: "Search query" },
          count: { type: "number", description: "1-10 results, default 5" },
          freshness: { type: "string", description: "pd=past day, pw=past week..." }
        },
        required: ["query"]
      }
    },
    { name: "web_fetch", ... },
    { name: "browser", ... },
    { name: "exec", ... },
    ...其他工具...
  ],
  max_tokens: 8192,
  stream: true
}
```

AI 的决策过程（在模型内部）：

```
用户要"搜索最新AI信息"
  → 需要实时网络信息（我的训练数据有截止日期）
  → 最合适的工具: web_search（专门用于网络搜索）
  → 不用 web_fetch（那是抓取已知URL的）
  → 不用 browser（那是交互式浏览器，太重了）
  → 不用 exec + curl（有专门的搜索工具就不用 curl）
  → 加 freshness: "pw" 限制为最近一周，确保"最新"
```

AI 返回（streaming）：

```json
{
  "content": [
    {
      "type": "text",
      "text": "我来帮你搜索最新的AI信息。"
    },
    {
      "type": "tool_use",
      "id": "toolu_01ABC",
      "name": "web_search",
      "input": {
        "query": "latest AI news developments 2026",
        "count": 5,
        "freshness": "pw"
      }
    }
  ],
  "stop_reason": "tool_use"
}
```

**事件流**：

```
→ event: message_start
→ event: text_delta  "我来帮你搜索最新的AI信息。"
    └─ onTextDelta 回调 → 暂存（不立即发送给用户，等工具结果）
→ event: tool_use_start  { name: "web_search", id: "toolu_01ABC" }
→ event: tool_use_delta  (参数 JSON 流)
→ event: tool_use_end
→ event: message_end     { stop_reason: "tool_use" }
```

### 阶段 7：工具执行 — web_search（~800ms）

**文件**：`src/agents/tools/web-search.ts`

```
handleToolExecutionStart()
  toolName: "web_search"
  args: { query: "latest AI news developments 2026", count: 5, freshness: "pw" }

web_search.execute()
  ├─ 解析搜索 provider 配置
  │   provider: "brave" (默认)
  │   apiKey: 从 BRAVE_SEARCH_API_KEY 环境变量读取
  │
  ├─ HTTP 请求:
  │   GET https://api.search.brave.com/res/v1/web/search
  │     ?q=latest+AI+news+developments+2026
  │     &count=5
  │     &freshness=pw
  │   Headers: { Accept: application/json, X-Subscription-Token: <key> }
  │
  ├─ 响应处理:
  │   将 Brave Search 结果转为纯文本格式
  │
  └─ 返回 ToolResult:
      {
        content: `
          1. [OpenAI发布GPT-5] https://example.com/gpt5
             OpenAI announced GPT-5 with significant improvements...

          2. [Google DeepMind新突破] https://example.com/deepmind
             DeepMind achieves new milestone in protein folding...

          3. [Anthropic Claude 4.6发布] https://example.com/claude
             Anthropic releases Claude 4.6 with enhanced reasoning...

          4. [AI监管新动态] https://example.com/regulation
             EU finalizes AI Act implementation guidelines...

          5. [开源AI模型竞争加剧] https://example.com/opensource
             Meta releases Llama 4, competing with commercial models...
        `,
        is_error: false
      }

handleToolExecutionEnd()
  ├─ sanitizeToolResult() → 截断到 8000 字符（如果需要）
  ├─ 追加 tool_result 到 session
  └─ 发射 onToolResult 事件
```

**Session JSONL 追加两行**：

```jsonl
{"role":"assistant","content":[{"type":"text","text":"我来帮你搜索最新的AI信息。"},{"type":"tool_use","id":"toolu_01ABC","name":"web_search","input":{"query":"latest AI news developments 2026","count":5,"freshness":"pw"}}],"stop_reason":"tool_use","model":"claude-sonnet-4-20250514","timestamp":1740480003000}
{"role":"tool_result","content":[{"type":"tool_result","tool_use_id":"toolu_01ABC","content":"1. [OpenAI发布GPT-5] https://...  2. [Google DeepMind...]..."}],"timestamp":1740480004000}
```

### 阶段 8：第二轮 AI 调用（~3000ms）

```
stop_reason 是 "tool_use" → 内层 agentic loop 继续
AI 现在看到了搜索结果，需要决定下一步
```

**AI 的决策分支**（此处有两条常见路径）：

#### 路径 A：直接总结（最常见，~80% 概率）

AI 判断搜索结果已经足够回答用户，直接生成总结。

```json
{
  "content": [
    {
      "type": "text",
      "text": "以下是最近一周的 AI 重要动态：\n\n1. **OpenAI 发布 GPT-5** ...\n2. **Google DeepMind 新突破** ...\n3. ..."
    }
  ],
  "stop_reason": "end_turn"
}
```

→ stop_reason = "end_turn" → **agentic loop 结束**

#### 路径 B：深入抓取（~20% 概率）

AI 认为搜索摘要不够详细，决定抓取其中 1-2 个链接获取完整内容：

```json
{
  "content": [
    {
      "type": "text",
      "text": "搜索结果不错，让我获取几篇文章的详细内容。"
    },
    {
      "type": "tool_use",
      "id": "toolu_02DEF",
      "name": "web_fetch",
      "input": {
        "url": "https://example.com/gpt5",
        "extractMode": "markdown"
      }
    }
  ],
  "stop_reason": "tool_use"
}
```

此时进入 **第三轮**：

```
web_fetch.execute()
  ├─ SSRF 保护检查（阻止访问内网地址）
  ├─ HTTP GET https://example.com/gpt5
  ├─ HTML → Readability 提取 → Markdown 转换
  ├─ 截断到 maxChars（如有设置）
  └─ 返回 ToolResult: { content: "# OpenAI GPT-5 发布\n\n..." }
```

然后 AI 在第三轮综合所有信息，生成最终回复：

```json
{
  "content": [
    {
      "type": "text",
      "text": "以下是最近一周的 AI 重要动态：\n\n## 1. OpenAI 发布 GPT-5\n（详细内容...）\n\n## 2. ..."
    }
  ],
  "stop_reason": "end_turn"
}
```

→ stop_reason = "end_turn" → **agentic loop 结束**

### 阶段 9：回复投递（~200ms）

**文件**：`src/agents/pi-embedded-runner/run.ts` → `src/telegram/bot-message-dispatch.ts`

```
buildEmbeddedRunPayloads()
  └─ payloads: [{ text: "以下是最近一周的 AI 重要动态：\n\n1. ..." }]

ReplyDispatcher
  └─ deliverTelegramReply()
      ├─ Markdown 转 Telegram MarkdownV2 格式
      ├─ 检查长度: Telegram 单条消息限制 4096 字符
      │   ├─ 如果 ≤ 4096: 发送一条
      │   └─ 如果 > 4096: 智能分块，按段落/标题边界切分
      ├─ Telegram API: sendMessage()
      │   { chat_id: 55512345, text: "...", parse_mode: "MarkdownV2" }
      └─ 更新 status reaction: 💭 → ✅

Session 持久化:
  └─ 追加 assistant 最终回复到 JSONL
```

### 阶段 10：完成

```
用户在 Telegram 中看到:

  ✅ 以下是最近一周的 AI 重要动态：

  1. **OpenAI 发布 GPT-5**
     OpenAI 本周发布了 GPT-5，在推理能力上有显著提升...

  2. **Google DeepMind 新突破**
     DeepMind 在蛋白质折叠领域取得新里程碑...

  3. **Anthropic Claude 4.6 发布**
     Anthropic 发布 Claude 4.6，增强了推理能力...

  4. **AI 监管新动态**
     欧盟确定了 AI 法案的实施指南...

  5. **开源 AI 模型竞争加剧**
     Meta 发布 Llama 4，与商业模型展开竞争...
```

---

## 时间线总结

```
T+0ms      Telegram webhook 收到 POST
T+10ms     去重通过
T+1510ms   防抖结束，processMessage() 触发
T+1520ms   MsgContext 构建完成
T+1525ms   路由: agentId="main", sessionKey="main:telegram:default:direct:55512345"
T+1575ms   Agent 准备完成（工具注册、system prompt、session 加载）
T+1575ms   ─── 第一轮 AI 调用 ───
T+3575ms   AI 返回: text + tool_use(web_search)
T+3575ms   ─── web_search 执行 ───
T+4375ms   Brave Search 返回 5 条结果
T+4375ms   ─── 第二轮 AI 调用 ───
T+7375ms   AI 返回: 最终文本总结, stop_reason="end_turn"
T+7375ms   ─── 回复投递 ───
T+7575ms   Telegram sendMessage() 完成
T+7575ms   ✅ 用户收到回复
```

**总耗时**：~7.5 秒（其中 AI API 调用 ~5 秒，网络搜索 ~0.8 秒，其余为框架开销）

---

## 工具使用决策树

针对"帮我搜索最新的AI信息"这个请求，AI 是这样决策的：

```
用户意图: 搜索最新信息
  │
  ├─ 需要实时网络数据？ → 是
  │   │
  │   ├─ 有明确 URL 要访问？ → 否
  │   │   │
  │   │   └─ → web_search（搜索引擎查询）
  │   │
  │   ├─ 有明确 URL？ → 是
  │   │   │
  │   │   └─ → web_fetch（抓取网页内容）
  │   │
  │   └─ 需要交互（登录、点击、填表）？ → 是
  │       │
  │       └─ → browser（浏览器自动化）
  │
  └─ 不需要实时数据？
      │
      └─ 直接从训练数据回答
```

**web_search 的搜索 provider 选择**：

```
配置了哪个 provider？ (tools.web.search.provider)
  │
  ├─ brave (默认) → Brave Search API
  │   优点: 无需 AI 调用，直接返回搜索结果，快
  │   缺点: 只有摘要，可能需要二次 web_fetch
  │
  ├─ perplexity → Perplexity Sonar
  │   优点: AI 综合答案 + 引用，一步到位
  │   缺点: 需要额外 API key，多一次 AI 调用
  │
  ├─ grok → xAI Grok
  │   优点: 实时 Twitter/X 数据，内联引用
  │   缺点: 需要 xAI API key
  │
  ├─ gemini → Google Gemini (Search Grounding)
  │   优点: Google 搜索数据
  │   缺点: 需要 Gemini API key
  │
  └─ kimi → Moonshot Kimi
      优点: 中文搜索优化
      缺点: 需要 Kimi API key
```

---

## 与最小可迁移版本的对应关系

| 阶段         | OpenClaw 实现                        | 最小版本需要                                 |
| ------------ | ------------------------------------ | -------------------------------------------- |
| 消息接收     | Telegram webhook + grammy            | 一个 HTTP endpoint 或 CLI 输入               |
| 去重/防抖    | LRU 缓存 + 防抖器                    | 可省略（单用户场景）                         |
| 消息上下文   | MsgContext (50+ 字段)                | 简化为 `{ text, sessionId }`                 |
| 路由         | 7 级绑定                             | 可省略（单 agent）                           |
| Session      | JSONL + SessionManager               | **需要**：JSONL 读写                         |
| 工具注册     | createOpenClawCodingTools (20+ 工具) | **需要**：至少 web_search + exec             |
| AI 调用      | streamSimple (多 provider)           | **需要**：Anthropic SDK                      |
| Agentic loop | SDK 自动循环                         | **需要**：while (stop_reason === "tool_use") |
| 回复投递     | ReplyDispatcher + 渠道格式化         | console.log 或 HTTP 响应                     |

**最小版本只需要实现中间 4 个"需要"的部分。**
