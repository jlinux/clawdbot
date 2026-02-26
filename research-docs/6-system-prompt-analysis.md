# OpenClaw System Prompt 分析：消息是如何被理解的？

## 核心结论

OpenClaw **没有任何消息解读/意图识别逻辑**。它只通过 system prompt 把工具清单列给 AI 看，由模型自己决定用什么工具。

---

## System Prompt 的实际内容

源码位置：`src/agents/system-prompt.ts` → `buildAgentSystemPrompt()`

构建出的完整 prompt 结构如下：

```
You are a personal assistant running inside OpenClaw.

## Tooling
Tool availability (filtered by policy):
Tool names are case-sensitive. Call tools exactly as listed.
- read: Read file contents
- write: Create or overwrite files
- edit: Make precise edits to files
- exec: Run shell commands (pty available for TTY-required CLIs)
- process: Manage background exec sessions
- web_search: Search the web (Brave API)
- web_fetch: Fetch and extract readable content from a URL
- browser: Control web browser
- canvas: Present/eval/snapshot the Canvas
- nodes: List/describe/notify/camera/screen on paired nodes
- cron: Manage cron jobs and wake events
- message: Send messages and channel actions
- gateway: Restart, apply config, or run updates
- agents_list: List agent ids allowed for sessions_spawn
- sessions_list: List other sessions with filters/last
- sessions_history: Fetch history for another session/sub-agent
- sessions_send: Send a message to another session/sub-agent
- sessions_spawn: Spawn a sub-agent session
- subagents: List, steer, or kill sub-agent runs
- session_status: Show usage/time/model state
- image: Analyze an image with the configured image model

## Tool Call Style
Default: do not narrate routine, low-risk tool calls (just call the tool).
Narrate only when it helps: multi-step work, complex/challenging problems,
sensitive actions (e.g., deletions), or when the user explicitly asks.

## Safety
You have no independent goals: do not pursue self-preservation, replication...
Prioritize safety and human oversight over completion...

## Skills (mandatory)
Before replying: scan <available_skills> <description> entries.
- If exactly one skill clearly applies: read its SKILL.md, then follow it.
- If multiple could apply: choose the most specific one.
- If none clearly apply: do not read any SKILL.md.
<available_skills>
- github (🐙): github operations via `gh` CLI
- nano-pdf (📄): Edit PDFs with nano-pdf CLI
- himalaya (📮): Email operations via Himalaya
... (按 token 预算截断)
</available_skills>

## Memory Recall
Before answering about prior work, decisions, dates, people, preferences,
or todos: run memory_search on MEMORY.md + memory/*.md; then use memory_get
to pull only the needed lines.

## Messaging
- Reply in current session → automatically routes to the source channel
- Cross-session messaging → use sessions_send(sessionKey, message)
- Never use exec/curl for provider messaging; OpenClaw handles all routing

## Workspace
Your working directory is: /home/user/workspace

## Documentation
OpenClaw docs: /path/to/docs
Mirror: https://docs.openclaw.ai

## Authorized Senders
Authorized senders: abc123def456. These senders are allowlisted.

## Current Date & Time
Time zone: Asia/Shanghai

## Reply Tags
[[reply_to_current]] replies to the triggering message.

## Silent Replies
When you have nothing to say, respond with ONLY: __SILENT__

## Heartbeats
If you receive a heartbeat poll, reply exactly: HEARTBEAT_OK

## Runtime
Runtime: agent=main | host=mac | os=darwin (arm64) | node=22.x |
  model=claude-sonnet-4-20250514 | channel=telegram | thinking=off

# Project Context
## SOUL.md
(如果存在，注入 persona 和 tone)

## .openclaw.md
(workspace 级的自定义指令)
```

---

## Prompt 中没有的东西

以下内容在 system prompt 中**完全不存在**：

```
❌ "如果用户说'搜索'，就调用 web_search"
❌ "如果用户发了图片，就调用 image 工具"
❌ "如果用户提到文件，就调用 read 工具"
❌ "分析用户意图，制定执行计划，然后一步步执行"
❌ "Think step by step"
❌ "You are a ReAct agent, use Thought/Action/Observation format"
❌  任何意图分类、槽位填充、路由规则
```

---

## 消息理解的实际机制

当用户说"帮我搜索最新的AI信息"时：

```
┌─────────────────────────────────────────────┐
│               System Prompt                  │
│  "你有 web_search 工具，功能是搜索互联网"     │
└──────────────────┬──────────────────────────┘
                   │
                   ↓ 发送给 AI API
┌─────────────────────────────────────────────┐
│           Claude 模型内部推理                 │
│                                              │
│  用户要"搜索" → 我有 web_search 工具         │
│  描述说"Search the web" → 语义匹配           │
│  → 决定调用 web_search                       │
│  → 构造参数 { query: "latest AI news" }      │
│                                              │
│  这个决策完全在模型内部完成                    │
│  没有任何外部代码参与                         │
└─────────────────────────────────────────────┘
```

**关键**：不是代码在做意图识别，是模型在做自然语言理解。

---

## System Prompt 各部分的作用

| 部分                | 作用                         | 是否影响工具选择                 |
| ------------------- | ---------------------------- | -------------------------------- |
| **Tooling**         | 列出可用工具 + 一行描述      | ✅ 直接决定 AI 能用什么          |
| **Tool Call Style** | 告诉 AI 不要废话，直接调工具 | 间接影响（减少不必要的文字）     |
| **Safety**          | 安全约束                     | 否（限制危险行为）               |
| **Skills**          | 列出可读的技能说明书         | 间接影响（AI 可能先读 Skill）    |
| **Memory Recall**   | 强制 AI 先查记忆库           | 间接影响（改变 AI 的第一步行动） |
| **Messaging**       | 教 AI 怎么发消息             | 否（只影响消息投递方式）         |
| **Workspace**       | 告诉 AI 工作目录在哪         | 否（影响文件操作路径）           |
| **Runtime**         | 告诉 AI 当前环境             | 否（环境信息）                   |
| **Context Files**   | 注入项目上下文（SOUL.md 等） | 间接影响（改变 AI 的 persona）   |

---

## 与其他系统的对比

| 系统                  | 消息理解方式                                    | 复杂度                  |
| --------------------- | ----------------------------------------------- | ----------------------- |
| **传统 chatbot**      | 意图识别 → 槽位填充 → 规则引擎路由              | 高（需要训练 NLU 模型） |
| **Rasa / Dialogflow** | 意图分类器 + 实体提取 + 对话流程图              | 高                      |
| **LangChain Agent**   | ReAct prompt（Thought/Action/Observation 循环） | 中（框架做提示工程）    |
| **OpenClaw**          | 只列工具清单，零意图路由，全靠模型              | **极低**                |

OpenClaw 的设计哲学：

> **框架尽可能笨，模型尽可能聪明。**
> 框架只负责：(1) 列出工具 (2) 执行工具 (3) 把结果放回对话。
> 所有"智能决策"（用什么工具、传什么参数、结果够不够、要不要继续）全部交给模型。

---

## 特殊的 Prompt 机制

虽然没有意图路由，但 prompt 中有几个值得注意的**强制行为**：

### 1. Memory Recall（强制记忆检索）

```
Before answering anything about prior work, decisions, dates, people,
preferences, or todos: run memory_search...
```

→ 当用户问关于过去的事时，AI 被强制先调 `memory_search`，再回答。

### 2. Skills Scan（强制技能扫描）

```
Before replying: scan <available_skills>...
If exactly one skill clearly applies: read its SKILL.md...
```

→ AI 被强制先扫描技能列表，匹配则先读 SKILL.md。

### 3. Silent Reply（静默回复）

```
When you have nothing to say, respond with ONLY: __SILENT__
```

→ 避免 AI 在无话可说时编造回复。

### 4. Heartbeat（心跳检测）

```
If you receive a heartbeat poll, reply exactly: HEARTBEAT_OK
```

→ 系统健康检查机制，AI 识别心跳消息后不执行任何工具。

---

## 对最小可迁移版本的启示

你需要的 system prompt 可以极其简单：

```typescript
const systemPrompt = `你是一个 AI 助手。

你可以使用以下工具：
- exec: 执行 shell 命令
- read: 读取文件内容
- write: 写入文件
- web_search: 搜索互联网

不要叙述常规操作，直接调用工具。`;
```

**这就够了。** 不需要意图解析，不需要 ReAct 框架，不需要 "think step by step"。

如果想增强，可以加：

- 一两句 persona 描述（"你是专注于 XX 领域的助手"）
- 强制行为（"回答前先搜索记忆"）
- 输出格式要求（"用中文回答"）

但核心逻辑——**用户消息 → 选择工具 → 执行 → 返回结果**——这个决策链完全在模型内部完成，不需要任何代码支持。
