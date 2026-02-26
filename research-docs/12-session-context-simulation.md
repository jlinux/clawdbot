# Session Context 模拟案例："帮我搜索AI新闻"

## 场景设定

- **用户**：Alice，通过 Telegram 私聊发送
- **模型**：Claude Sonnet，上下文窗口 200,000 tokens
- **Session**：已有 3 轮历史对话（不是空 session）
- **Session Key**：`main:telegram:default:direct:55512345`
- **Session File**：`~/.openclaw/agents/main/sessions/abc123.jsonl`

---

## 一、Session 文件初始状态

模拟 Alice 之前已经有 3 轮对话。`abc123.jsonl` 当前内容：

```jsonl
{"type":"session_info","id":"s0","name":"telegram:dm:alice"}
{"type":"message","id":"e1","parentId":"s0","role":"user","content":"你好","timestamp":1740470000000}
{"type":"message","id":"e2","parentId":"e1","role":"assistant","content":[{"type":"text","text":"你好 Alice！有什么我可以帮你的？"}],"timestamp":1740470001000}
{"type":"message","id":"e3","parentId":"e2","role":"user","content":"帮我读一下 /tmp/notes.txt","timestamp":1740471000000}
{"type":"message","id":"e4","parentId":"e3","role":"assistant","content":[{"type":"toolCall","toolCallId":"tc1","toolName":"read","arguments":{"path":"/tmp/notes.txt"}}],"timestamp":1740471001000}
{"type":"message","id":"e5","parentId":"e4","role":"toolResult","toolCallId":"tc1","toolName":"read","content":[{"type":"text","text":"这是笔记内容...(假设 2000 字符)"}],"timestamp":1740471002000}
{"type":"message","id":"e6","parentId":"e5","role":"assistant","content":[{"type":"text","text":"文件内容如下：\n\n这是笔记内容..."}],"timestamp":1740471003000}
{"type":"message","id":"e7","parentId":"e6","role":"user","content":"谢谢","timestamp":1740472000000}
{"type":"message","id":"e8","parentId":"e7","role":"assistant","content":[{"type":"text","text":"不客气！随时可以问我。"}],"timestamp":1740472001000}
```

**DAG 视图**（线性链）：

```
s0 → e1 → e2 → e3 → e4 → e5 → e6 → e7 → e8
                      user  asst  tool  asst  user  asst
                            ↑read        ↑回复
```

---

## 二、新消息到达："帮我搜索AI新闻"

### 2.1 消息进入管道

```
Telegram webhook → 去重 → 防抖(1.5s) → processMessage()
  → resolveAgentRoute() → sessionKey = "main:telegram:default:direct:55512345"
  → runEmbeddedPiAgent(params)
  → runEmbeddedAttempt(params)
```

### 2.2 System Prompt 组装

这是 `buildEmbeddedSystemPrompt()` 的输出。简化示意：

```
┌─────────────────────── System Prompt (~4000 tokens) ───────────────────────┐
│                                                                              │
│  你是 OpenClaw，一个 AI 助手。                                               │
│                                                                              │
│  ## Tooling                                                                  │
│  Available tools:                                                            │
│  - read: Read file contents                                                  │
│  - write: Write file contents                                                │
│  - edit: Edit file contents                                                  │
│  - exec: Execute shell commands                                              │
│  - web_search: Search the web using Brave/Perplexity                         │
│  - web_fetch: Fetch and extract web page content                             │
│  - message: Send messages to channels                                        │
│  - tts: Text to speech                                                       │
│  ... (其他工具)                                                               │
│                                                                              │
│  ## Skills (mandatory)                                                       │
│  Before replying: scan <available_skills> <description> entries.             │
│  <available_skills>                                                          │
│    <skill>                                                                   │
│      <name>github</name>                                                     │
│      <description>github operations via `gh` CLI</description>               │
│      <location>~/.openclaw/skills/github/SKILL.md</location>                 │
│    </skill>                                                                  │
│    ...                                                                       │
│  </available_skills>                                                         │
│                                                                              │
│  ## Runtime                                                                  │
│  Runtime: host=macbook channel=telegram model=claude-sonnet agent=main       │
│                                                                              │
│  ## Date & Time                                                              │
│  Current time: 2026-02-26 10:30 CST (Asia/Shanghai)                          │
│                                                                              │
│  ## Workspace                                                                │
│  Working directory: /Users/alice/workspace                                   │
│                                                                              │
│  ## Context Files                                                            │
│  (SOUL.md 的内容, .openclaw.md 的内容)                                        │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 2.3 Session 消息加载与准备

```
SessionManager.open("abc123.jsonl")
  ↓
session.messages = [e1, e2, e3, e4, e5, e6, e7, e8]   ← 8 条消息
  ↓
sanitizeSessionHistory()                                ← 验证 tool_use/tool_result 配对
  ↓
limitHistoryTurns(messages, undefined)                  ← 无 historyLimit 配置，全部保留
  ↓
sanitizeToolUseResultPairing()                          ← 检查无孤儿
  ↓
session.agent.replaceMessages([e1...e8])                ← 注入到 agent 内存
```

---

## 三、发送给 LLM 的完整 Context

### 3.1 API 请求结构

这是实际发给 Claude API 的 HTTP 请求体（简化）：

```json
{
  "model": "claude-sonnet-4-20250514",
  "max_tokens": 8192,
  "system": "你是 OpenClaw，一个 AI 助手。\n\n## Tooling\n...(约 4000 tokens)",
  "tools": [
    {
      "name": "read",
      "description": "Read file contents",
      "input_schema": {
        "type": "object",
        "properties": {
          "path": { "type": "string" },
          "offset": { "type": "number" },
          "limit": { "type": "number" }
        }
      }
    },
    {
      "name": "web_search",
      "description": "Search the web",
      "input_schema": {
        "type": "object",
        "properties": { "query": { "type": "string" }, "count": { "type": "number" } }
      }
    }
  ],
  "messages": [
    { "role": "user", "content": "你好" },
    { "role": "assistant", "content": "你好 Alice！有什么我可以帮你的？" },
    { "role": "user", "content": "帮我读一下 /tmp/notes.txt" },
    {
      "role": "assistant",
      "content": [
        { "type": "tool_use", "id": "tc1", "name": "read", "input": { "path": "/tmp/notes.txt" } }
      ]
    },
    {
      "role": "user",
      "content": [
        { "type": "tool_result", "tool_use_id": "tc1", "content": "这是笔记内容...(2000字符)" }
      ]
    },
    { "role": "assistant", "content": "文件内容如下：\n\n这是笔记内容..." },
    { "role": "user", "content": "谢谢" },
    { "role": "assistant", "content": "不客气！随时可以问我。" },
    { "role": "user", "content": "帮我搜索AI新闻" }
  ]
}
```

### 3.2 Token 预算快照（发送前）

```
┌──────────────────────────────────────────────────────────────────────┐
│                  上下文窗口: 200,000 tokens                           │
│                                                                       │
│  ┌──────────────────────────────────────┐                            │
│  │ System Prompt           ~4,000 tokens │ ██░░░░░░░░░░░░░░░ 2.0%   │
│  ├──────────────────────────────────────┤                            │
│  │ Tools Schema            ~2,000 tokens │ █░░░░░░░░░░░░░░░░ 1.0%   │
│  ├──────────────────────────────────────┤                            │
│  │ 历史消息 (e1-e8)        ~1,500 tokens │ █░░░░░░░░░░░░░░░░ 0.75%  │
│  │   e1 user "你好"              ~10 t   │                           │
│  │   e2 asst "你好Alice..."      ~30 t   │                           │
│  │   e3 user "帮我读..."         ~20 t   │                           │
│  │   e4 asst [tool_use:read]     ~50 t   │                           │
│  │   e5 toolResult (2000 chars) ~500 t   │                           │
│  │   e6 asst "文件内容..."      ~520 t   │                           │
│  │   e7 user "谢谢"              ~10 t   │                           │
│  │   e8 asst "不客气..."         ~20 t   │                           │
│  ├──────────────────────────────────────┤                            │
│  │ 新用户消息               ~20 tokens   │ ░░░░░░░░░░░░░░░░░ 0.01%  │
│  │   "帮我搜索AI新闻"                    │                           │
│  ├──────────────────────────────────────┤                            │
│  │ 已用合计               ~7,520 tokens  │ ██░░░░░░░░░░░░░░░ 3.76%  │
│  ├──────────────────────────────────────┤                            │
│  │ 可用空间             ~192,480 tokens  │ ░░░░░░░░░░░░░░░░░ 96.2%  │
│  └──────────────────────────────────────┘                            │
│                                                                       │
│  Context Pruning 检查:                                                │
│    总估算字符 = 7520 × 4 = 30,080 chars                               │
│    窗口字符 = 200,000 × 4 = 800,000 chars                             │
│    占比 = 30,080 / 800,000 = 3.76%                                    │
│    < softTrimRatio(30%) → ❌ 不触发修剪                                │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 四、Agent Loop 执行

### 第 1 轮：AI 决定搜索

**LLM 内部推理**（不可见）：

```
System prompt 说我有 web_search 工具。
用户要搜索 AI 新闻 → 应该用 web_search。
Skills 扫描：没有匹配 "AI新闻搜索" 的 skill → 不读任何 SKILL.md。
```

**LLM 返回**：

```json
{
  "id": "msg_01abc",
  "type": "message",
  "role": "assistant",
  "content": [
    {
      "type": "tool_use",
      "id": "tc2",
      "name": "web_search",
      "input": { "query": "latest AI news 2026", "count": 5 }
    }
  ],
  "stop_reason": "tool_use"
}
```

**Agent Loop 检查**：

```
message.content 中有 type: "toolCall" → hasMoreToolCalls = true → 继续循环
```

**追加到 Session JSONL**：

```jsonl
{"type":"message","id":"e9","parentId":"e8","role":"user","content":"帮我搜索AI新闻","timestamp":1740473000000}
{"type":"message","id":"e10","parentId":"e9","role":"assistant","content":[{"type":"toolCall","toolCallId":"tc2","toolName":"web_search","arguments":{"query":"latest AI news 2026","count":5}}],"timestamp":1740473001000}
```

### 第 1 轮：Tool 执行

```
executeToolCalls([{name: "web_search", id: "tc2", args: {...}}])
  ↓
WebSearchTool.execute({ query: "latest AI news 2026", count: 5 })
  ↓
HTTP POST → Brave Search API (或 Perplexity/Grok)
  ↓
返回搜索结果（假设 ~8000 字符）
```

**Tool Result**（追加到 JSONL）：

```jsonl
{
  "type": "message",
  "id": "e11",
  "parentId": "e10",
  "role": "toolResult",
  "toolCallId": "tc2",
  "toolName": "web_search",
  "content": [
    {
      "type": "text",
      "text": "Search results for 'latest AI news 2026':\n\n1. **Claude 4.5 Released with 1M Context** - Anthropic releases...\n   URL: https://techcrunch.com/...\n   Snippet: Anthropic today announced Claude 4.5, featuring a 1 million token context window...\n\n2. **OpenAI GPT-5 Enters Beta Testing** - OpenAI has begun...\n   URL: https://www.theverge.com/...\n   Snippet: OpenAI's GPT-5 model is now available to beta testers...\n\n3. **Google DeepMind's Gemini 3 Achieves SOTA on Math** - ...\n   URL: https://blog.google/...\n   Snippet: Gemini 3 sets new state-of-the-art benchmarks...\n\n4. **Meta Releases LLaMA 4 Open Source** - ...\n   URL: https://ai.meta.com/...\n   Snippet: Meta continues open-source AI leadership...\n\n5. **EU AI Act Enforcement Begins** - ...\n   URL: https://reuters.com/...\n   Snippet: The European Union's AI Act enters enforcement phase..."
    }
  ],
  "timestamp": 1740473002000
}
```

### Token 预算快照（第 1 轮后）

```
  已用合计             ~7,520 tokens (之前)
  + e9 user             ~20 tokens
  + e10 asst tool_use   ~60 tokens
  + e11 toolResult     ~2,000 tokens (8000 chars / 4)
  ─────────────────────────
  合计                 ~9,600 tokens (4.8%)

  Context Pruning: 9,600 × 4 / 800,000 = 4.8% < 30% → 不修剪
```

### 第 2 轮：AI 返回最终回复

**发送给 LLM 的 messages**（现在是 12 条）：

```
e1  user    "你好"
e2  asst    "你好 Alice..."
e3  user    "帮我读一下 /tmp/notes.txt"
e4  asst    [tool_use: read]
e5  tool    [tool_result: 笔记内容]
e6  asst    "文件内容如下..."
e7  user    "谢谢"
e8  asst    "不客气..."
e9  user    "帮我搜索AI新闻"           ← 当前请求
e10 asst    [tool_use: web_search]
e11 tool    [tool_result: 5条搜索结果]
                                        ← LLM 从这里继续生成
```

**LLM 返回**：

```json
{
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "以下是最新的 AI 新闻：\n\n1. **Claude 4.5 发布** — Anthropic 发布了支持 100 万 token 上下文窗口的 Claude 4.5...\n\n2. **OpenAI GPT-5 开始 Beta 测试** — GPT-5 已向测试用户开放...\n\n3. **Google DeepMind Gemini 3** — 在数学推理基准上创下新纪录...\n\n4. **Meta 开源 LLaMA 4** — Meta 继续引领开源 AI...\n\n5. **欧盟 AI 法案开始执行** — 欧洲 AI 监管进入实施阶段...\n\n需要我展开任何一条新闻的详细内容吗？"
    }
  ],
  "stop_reason": "end_turn"
}
```

**Agent Loop 检查**：

```
message.content 中没有 type: "toolCall" → hasMoreToolCalls = false → 循环结束
```

**追加到 Session JSONL**：

```jsonl
{
  "type": "message",
  "id": "e12",
  "parentId": "e11",
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "以下是最新的 AI 新闻：\n\n1. **Claude 4.5 发布**...（完整回复）"
    }
  ],
  "timestamp": 1740473003000
}
```

---

## 五、最终 Session 文件状态

```jsonl
{"type":"session_info","id":"s0","name":"telegram:dm:alice"}
{"type":"message","id":"e1","parentId":"s0","role":"user","content":"你好","timestamp":1740470000000}
{"type":"message","id":"e2","parentId":"e1","role":"assistant","content":[{"type":"text","text":"你好 Alice！..."}],"timestamp":1740470001000}
{"type":"message","id":"e3","parentId":"e2","role":"user","content":"帮我读一下 /tmp/notes.txt","timestamp":1740471000000}
{"type":"message","id":"e4","parentId":"e3","role":"assistant","content":[{"type":"toolCall","toolCallId":"tc1","toolName":"read","arguments":{"path":"/tmp/notes.txt"}}],"timestamp":1740471001000}
{"type":"message","id":"e5","parentId":"e4","role":"toolResult","toolCallId":"tc1","toolName":"read","content":[{"type":"text","text":"这是笔记内容..."}],"timestamp":1740471002000}
{"type":"message","id":"e6","parentId":"e5","role":"assistant","content":[{"type":"text","text":"文件内容如下：..."}],"timestamp":1740471003000}
{"type":"message","id":"e7","parentId":"e6","role":"user","content":"谢谢","timestamp":1740472000000}
{"type":"message","id":"e8","parentId":"e7","role":"assistant","content":[{"type":"text","text":"不客气！..."}],"timestamp":1740472001000}
{"type":"message","id":"e9","parentId":"e8","role":"user","content":"帮我搜索AI新闻","timestamp":1740473000000}
{"type":"message","id":"e10","parentId":"e9","role":"assistant","content":[{"type":"toolCall","toolCallId":"tc2","toolName":"web_search","arguments":{"query":"latest AI news 2026","count":5}}],"timestamp":1740473001000}
{"type":"message","id":"e11","parentId":"e10","role":"toolResult","toolCallId":"tc2","toolName":"web_search","content":[{"type":"text","text":"Search results for..."}],"timestamp":1740473002000}
{"type":"message","id":"e12","parentId":"e11","role":"assistant","content":[{"type":"text","text":"以下是最新的 AI 新闻：..."}],"timestamp":1740473003000}
```

**DAG 视图**：

```
s0 → e1 → e2 → e3 → e4 → e5 → e6 → e7 → e8 → e9 → e10 → e11 → e12
     user  asst  user  asst  tool  asst  user  asst  user  asst   tool  asst
                       read↑       ↑回复              search↑            ↑最终回复
     ├──── Turn 1 ────┤├────── Turn 2 ──────────┤├─ Turn 3 ─┤├── Turn 4（本次）──┤
```

---

## 六、如果 Session 累积到触发压缩

假设 Alice 是重度用户，又继续对话了 200 轮，其中包含大量 `read`、`exec`、`web_search` 的 tool result，Session 累积到 **~170,000 tokens**。

### 6.1 Context Pruning 触发（agent loop 中）

```
第 201 轮开始前，Context Pruning 检查：
  总估算字符 = 170,000 × 4 = 680,000 chars
  窗口字符 = 200,000 × 4 = 800,000 chars
  占比 = 680,000 / 800,000 = 85%

  85% > hardClearRatio(50%) → 触发硬清除
  85% > softTrimRatio(30%) → 也触发软修剪

执行顺序：
  1. 找到 cutoff index（最近 3 个 assistant 消息之前的）
  2. 对 cutoff 之前的 toolResult 执行软修剪
     e5 (read result, 2000 chars): 2000 < 4000 → 跳过（太小）
     e50 (exec result, 15000 chars): 15000 > 4000 → 软修剪
       → 保留前 1500 + "..." + 后 1500 chars
     e100 (web_search result, 8000 chars): 8000 > 4000 → 软修剪
  3. 软修剪后重新计算占比
     假设降到 65% → 仍 > 50%
  4. 对旧的 toolResult 执行硬清除
     e50 → "[Old tool result content cleared]"
     e100 → "[Old tool result content cleared]"
     ...继续直到占比 < 50%
```

### 6.2 Context Overflow → Compaction 触发

如果 Pruning 后仍不够，LLM API 返回 context overflow 错误：

```
Agent Loop 捕获 context overflow
  ↓
isLikelyContextOverflowError(error) → true
  ↓
sessionLikelyHasOversizedToolResults(messages, 200000) → 检查
  ↓
如果有超大 tool result:
  truncateOversizedToolResultsInSession()              ← 第一次尝试
  → 重试 agent loop
  ↓
如果仍溢出:
  触发 Compaction
```

### 6.3 Compaction 过程模拟

```
compactEmbeddedPiSessionDirect({
  sessionFile: "abc123.jsonl",
  trigger: "overflow",
  provider: "anthropic",
  model: "claude-sonnet-4-20250514",
})
  │
  ├── 获取 session write lock
  │
  ├── 加载 400 条消息（e1 ~ e400）
  │   估算 token: ~180,000
  │
  ├── SDK session.compact() → 触发 session_before_compact 事件
  │
  ├── compaction-safeguard 接管：
  │   │
  │   ├── 计算预算:
  │   │   contextWindowTokens = 200,000
  │   │   maxHistoryShare = 0.5
  │   │   budgetTokens = 100,000
  │   │   当前历史 = 180,000 > 100,000
  │   │
  │   ├── pruneHistoryForContextShare():
  │   │   splitMessagesByTokenShare(messages, 2)
  │   │     chunk1: e1 ~ e200  (~90,000 tokens) ← 丢弃
  │   │     chunk2: e201 ~ e400 (~90,000 tokens) ← 保留
  │   │   repairToolUseResultPairing(chunk2)       ← 修复切割点的孤儿
  │   │   → droppedMessages = 200, droppedTokens = 90,000
  │   │
  │   ├── 对丢弃的消息做摘要:
  │   │   summarizeInStages(chunk1, parts=2)
  │   │     sub-chunk1a: e1~e100  → summary_a
  │   │     sub-chunk1b: e101~e200 → summary_b
  │   │     merge(summary_a, summary_b) → droppedSummary
  │   │
  │   ├── 对保留的消息做摘要（但这些消息也可能很多）:
  │   │   summarizeInStages(chunk2, parts=2, previousSummary=droppedSummary)
  │   │     → historySummary
  │   │
  │   ├── 追加附加信息:
  │   │   historySummary += "\n\n## Tool Failures\n- exec: command not found..."
  │   │   historySummary += "\n\n<read-files>\n/tmp/notes.txt\n...</read-files>"
  │   │   historySummary += "\n\n<modified-files>\n/tmp/output.txt\n...</modified-files>"
  │   │   historySummary += "\n\n<workspace-critical-rules>...(AGENTS.md 关键规则)</workspace-critical-rules>"
  │   │
  │   └── 返回:
  │       {
  │         compaction: {
  │           summary: "Alice 在之前的对话中：1) 问候后读取了 /tmp/notes.txt..."
  │                    "2) 搜索了 AI 新闻（Claude 4.5, GPT-5, Gemini 3...）"
  │                    "3) 执行了多个 shell 命令处理数据..."
  │                    "## Tool Failures\n- exec: command not found"
  │                    "<read-files>...</read-files>"
  │                    "<modified-files>...</modified-files>",
  │           firstKeptEntryId: "e350",
  │           tokensBefore: 180000,
  │         }
  │       }
  │
  ├── 估算 tokensAfter: ~25,000
  │
  └── 释放 session write lock
```

### 6.4 Compaction 后的 Session JSONL

```jsonl
... (e1 ~ e349 仍在文件中，但不再被加载) ...

{"type":"compaction","id":"cmp1","summary":"Alice 在之前的对话中：\n1) 问候后读取了 /tmp/notes.txt（笔记内容）\n2) 搜索了 AI 新闻（Claude 4.5, GPT-5, Gemini 3, LLaMA 4, EU AI Act）\n3) 执行了多个 shell 命令处理数据文件\n4) 在 /tmp/output.txt 中生成了处理结果\n\n## Tool Failures\n- exec: command not found (git push)\n\n<read-files>\n/tmp/notes.txt\n/tmp/data.csv\n</read-files>\n\n<modified-files>\n/tmp/output.txt\n</modified-files>","firstKeptEntryId":"e350","tokensBefore":180000}

{"type":"message","id":"e350","parentId":"cmp1","role":"user","content":"帮我看看 output.txt 的结果","timestamp":1740490000000}
... (e351 ~ e400 继续) ...
```

### 6.5 Compaction 后发给 LLM 的 messages

```json
{
  "system": "你是 OpenClaw...(system prompt ~4000 tokens)",
  "messages": [
    {
      "role": "user",
      "content": "[Previous conversation summary]\nAlice 在之前的对话中：\n1) 问候后读取了 /tmp/notes.txt...\n2) 搜索了 AI 新闻...\n3) 执行了多个 shell 命令...\n\n<read-files>...</read-files>\n<modified-files>...</modified-files>"
    },
    {"role": "user", "content": "帮我看看 output.txt 的结果"},
    {"role": "assistant", "content": [{"type": "tool_use", ...}]},
    ... (e351 ~ e400, 约 50 条最近消息)
  ]
}
```

**Compaction 后的 Token 预算**：

```
┌──────────────────────────────────────────────────────────────────────┐
│                  上下文窗口: 200,000 tokens                           │
│                                                                       │
│  ┌──────────────────────────────────────┐                            │
│  │ System Prompt           ~4,000 tokens │ ██░░░░░░░░░░░░░░░ 2.0%   │
│  ├──────────────────────────────────────┤                            │
│  │ Compaction Summary       ~800 tokens  │ █░░░░░░░░░░░░░░░░ 0.4%   │
│  ├──────────────────────────────────────┤                            │
│  │ 保留的消息 (e350~e400) ~20,000 tokens │ ██████░░░░░░░░░░░ 10.0%  │
│  ├──────────────────────────────────────┤                            │
│  │ 已用合计              ~24,800 tokens  │ ████░░░░░░░░░░░░░ 12.4%  │
│  ├──────────────────────────────────────┤                            │
│  │ 可用空间            ~175,200 tokens   │ ░░░░░░░░░░░░░░░░░ 87.6%  │
│  └──────────────────────────────────────┘                            │
│                                                                       │
│  压缩效果: 180,000 → 24,800 tokens (节省 86%)                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 七、时序总览

```
时间轴 →

t=0        t=1         t=2         t=3         t=4
│          │           │           │           │
▼          ▼           ▼           ▼           ▼
消息到达   System      发送给      Tool执行    发送给
           Prompt组装   LLM #1     web_search  LLM #2
           + Session               返回结果    最终回复
           加载

┌──────────────────────────────────────────────────────────────┐
│                    Context 变化过程                            │
│                                                                │
│  t=0: 加载历史 e1~e8                        ~1,500 tokens     │
│  t=1: + system prompt + tools schema        ~7,520 tokens     │
│  t=2: + 新消息 "帮我搜索AI新闻"             ~7,540 tokens     │
│       → 发送给 LLM，返回 tool_use                              │
│  t=3: + e10(tool_use) + e11(tool_result)    ~9,600 tokens     │
│       → web_search 执行，结果 ~8000 chars                      │
│  t=4: 完整 context 发给 LLM                 ~9,600 tokens     │
│       → LLM 返回 end_turn，循环结束                            │
│       + e12(最终回复)                        ~10,200 tokens    │
│                                                                │
│  Session JSONL: s0 + e1~e12 (12 条消息)                        │
└──────────────────────────────────────────────────────────────┘
```

---

## 八、关键观察

| 环节                                 | 发现                                                              |
| ------------------------------------ | ----------------------------------------------------------------- |
| **每次 API 调用都发全部历史**        | e1~e8 的历史每次都发，即使与当前任务无关                          |
| **Tool result 是 token 大户**        | web_search 的 8000 字符结果占了 ~2000 tokens，是普通消息的 100 倍 |
| **Agent loop 内 system prompt 不变** | 第 1 轮和第 2 轮用的是同一份 system prompt                        |
| **Context Pruning 在低占用时不触发** | 这个简单案例只用了 5% 上下文，远未触发任何修剪                    |
| **JSONL 是 append-only**             | 消息只追加不修改，即使 compaction 后旧消息仍在文件中              |
| **Compaction 摘要保留关键信息**      | 文件操作列表、工具失败记录、workspace 规则都会附加到摘要中        |
