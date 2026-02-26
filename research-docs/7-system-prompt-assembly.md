# system-prompt.ts 组装逻辑与调用链路

## 文件关系

```
调用链（从上到下）:

用户消息到达
  ↓
src/agents/pi-embedded-runner/run/attempt.ts     ← 准备所有原材料
  ↓ 调用
src/agents/pi-embedded-runner/system-prompt.ts   ← 薄包装层
  ↓ 调用
src/agents/system-prompt.ts                      ← 真正的组装工厂
  ↓ 返回
一个完整的 system prompt 字符串
  ↓ 传给
session.agent.setSystemPrompt(prompt)            ← 注入到 AI session
  ↓ 随后
activeSession.prompt(userText)                   ← 连同用户消息一起发给 AI API
```

---

## 一、组装工厂：`buildAgentSystemPrompt()`

**文件**：`src/agents/system-prompt.ts:189`

这是核心函数。它接收 ~30 个参数，按固定顺序拼接出一个多段文本。

### 输入参数（数据来源）

```typescript
buildAgentSystemPrompt({
  // ── 工作环境 ──
  workspaceDir, // agent 的工作目录
  sandboxInfo, // 沙箱配置（如果在 Docker 中运行）

  // ── 工具列表 ──
  toolNames, // 当前可用工具的名字数组 ["read", "write", "exec", "web_search", ...]
  toolSummaries, // 工具名 → 一行描述的 map

  // ── 技能 ──
  skillsPrompt, // 格式化后的 <available_skills> 文本

  // ── 记忆 ──
  memoryCitationsMode, // 记忆引用模式（off / on）

  // ── 身份 ──
  ownerNumbers, // 授权用户标识
  ownerDisplay, // 显示方式（raw / hash）

  // ── 模型相关 ──
  modelAliasLines, // 模型别名（如 "sonnet → claude-sonnet-4-20250514"）
  defaultThinkLevel, // thinking 级别
  reasoningLevel, // 推理级别（off/on/stream）
  reasoningTagHint, // 是否要求 <think>/<final> 标签格式

  // ── 上下文注入 ──
  extraSystemPrompt, // 额外系统提示（群聊上下文/子agent上下文）
  contextFiles, // 项目上下文文件（SOUL.md, .openclaw.md 等）
  workspaceNotes, // workspace 级备注

  // ── 渠道相关 ──
  runtimeInfo, // 运行时信息（host/os/model/channel）
  messageToolHints, // 渠道特定的消息工具提示
  reactionGuidance, // 表情反应指导（Telegram/Signal）

  // ── 杂项 ──
  heartbeatPrompt, // 心跳检测提示词
  docsPath, // 文档路径
  ttsHint, // TTS 语音提示
  userTimezone, // 用户时区
  promptMode, // "full" | "minimal" | "none"
});
```

### 组装顺序（逐段拼接）

函数内部就是一个 `lines` 数组，按顺序 push 各段内容，最后 `join("\n")`：

```
第 1 段: 身份声明
  "You are a personal assistant running inside OpenClaw."

第 2 段: ## Tooling（工具列表）
  遍历 toolOrder 数组，按固定顺序列出每个可用工具
  格式: "- {name}: {一行描述}"
  + 长等待提示
  + 子agent提示

第 3 段: ## Tool Call Style（调用风格）
  "不要叙述常规操作，直接调工具"

第 4 段: ## Safety（安全约束）
  "不要追求自我保存、复制、权力扩张..."

第 5 段: ## OpenClaw CLI Quick Reference
  openclaw gateway status/start/stop/restart

第 6 段: ## Skills (mandatory)（技能列表）
  ← 仅当 skillsPrompt 不为空时
  "扫描 available_skills，匹配则读 SKILL.md"

第 7 段: ## Memory Recall（记忆检索）
  ← 仅当 memory_search 工具可用时
  "回答前先 memory_search"

第 8 段: ## OpenClaw Self-Update
  ← 仅当 gateway 工具可用且非 minimal 模式
  "只在用户明确要求时才更新"

第 9 段: ## Model Aliases（模型别名）
  ← 仅当有别名且非 minimal 模式

第 10 段: ## Workspace（工作目录）
  "Your working directory is: /path/to/workspace"
  + workspace notes

第 11 段: ## Documentation（文档路径）
  ← 仅当非 minimal 模式

第 12 段: ## Sandbox（沙箱信息）
  ← 仅当沙箱启用时
  容器路径、权限、浏览器控制等

第 13 段: ## Authorized Senders（授权用户）
  ← 仅当非 minimal 模式

第 14 段: ## Current Date & Time
  "Time zone: Asia/Shanghai"

第 15 段: ## Workspace Files (injected)
  说明项目上下文文件已加载

第 16 段: ## Reply Tags（回复标签）
  [[reply_to_current]] 等

第 17 段: ## Messaging（消息规则）
  渠道路由、message tool 用法、内联按钮支持

第 18 段: ## Voice (TTS)
  ← 仅当有 TTS 配置时

第 19 段: ## Group Chat Context / Subagent Context
  ← 仅当有 extraSystemPrompt 时

第 20 段: ## Reactions（表情反应指导）
  ← 仅当渠道配置了 reaction

第 21 段: ## Reasoning Format
  ← 仅当 reasoningTagHint 为 true
  要求 <think>...</think> <final>...</final> 格式

第 22 段: # Project Context（项目上下文文件注入）
  遍历 contextFiles，把每个文件内容直接嵌入
  如果有 SOUL.md，加 persona 指令

第 23 段: ## Silent Replies
  "无话可说时回复 __SILENT__"

第 24 段: ## Heartbeats
  "收到心跳就回复 HEARTBEAT_OK"

第 25 段: ## Runtime（运行时信息）
  "Runtime: agent=main | host=mac | os=darwin | model=claude-sonnet-4..."
```

### 条件逻辑（promptMode）

```
promptMode = "none"   → 只返回 "You are a personal assistant running inside OpenClaw."
promptMode = "minimal" → 跳过: Memory, Safety详情, Docs, Messaging, Reactions, Silent, Heartbeat
promptMode = "full"    → 全部包含（默认）
```

- `"full"` 用于主 agent
- `"minimal"` 用于子 agent（sessions_spawn 创建的）
- `"none"` 用于极简场景

---

## 二、薄包装层：`buildEmbeddedSystemPrompt()`

**文件**：`src/agents/pi-embedded-runner/system-prompt.ts:11`

这个函数做两件事：

1. 从传入的 `tools: AgentTool[]` 数组提取 `toolNames` 和 `toolSummaries`
2. 转发给 `buildAgentSystemPrompt()`

```typescript
export function buildEmbeddedSystemPrompt(params) {
  return buildAgentSystemPrompt({
    ...params,
    toolNames: params.tools.map((tool) => tool.name), // 提取工具名
    toolSummaries: buildToolSummaryMap(params.tools), // 提取工具描述
  });
}
```

`buildToolSummaryMap()` 的逻辑：遍历工具数组，取每个工具的 `description` 字段的第一行作为摘要。

---

## 三、调用方：`runEmbeddedAttempt()`

**文件**：`src/agents/pi-embedded-runner/run/attempt.ts:530`

这是 system prompt 被实际调用的地方。在每次 agent attempt 开始时：

```typescript
// attempt.ts 第 530 行
const appendPrompt = buildEmbeddedSystemPrompt({
  workspaceDir: effectiveWorkspace,
  defaultThinkLevel: params.thinkLevel,
  reasoningLevel: params.reasoningLevel ?? "off",
  extraSystemPrompt: params.extraSystemPrompt,
  ownerNumbers: params.ownerNumbers,
  ownerDisplay: ownerDisplay.ownerDisplay,
  reasoningTagHint,
  heartbeatPrompt: isDefaultAgent ? resolveHeartbeatPrompt(...) : undefined,
  skillsPrompt,
  docsPath,
  ttsHint,
  workspaceNotes,
  reactionGuidance,
  promptMode,
  runtimeInfo,            // ← 从 os.type(), os.arch(), process.version 等拼出
  messageToolHints,
  sandboxInfo,
  tools,                  // ← 从 createOpenClawCodingTools() 创建的工具数组
  modelAliasLines: buildModelAliasLines(params.config),
  userTimezone,           // ← 从配置或系统时区解析
  userTime,
  userTimeFormat,
  contextFiles,           // ← 从 workspace 加载的 SOUL.md, .openclaw.md 等
  memoryCitationsMode: params.config?.memory?.citations,
});

// 第 582 行: 创建覆盖函数
const systemPromptOverride = createSystemPromptOverride(appendPrompt);

// 第 583 行: 获取最终 prompt 文本
let systemPromptText = systemPromptOverride();

// ... 后续创建 session 时注入 ...
```

### 各参数的数据源追踪

```
attempt.ts 中的参数        ← 数据来源
─────────────────────────────────────────────────
workspaceDir              ← resolveWorkspaceRoot() 或 params.cwd
tools                     ← createOpenClawCodingTools() + resolvePluginTools()
                             每次 attempt 新创建，包含所有内置 + 插件工具
skillsPrompt              ← buildWorkspaceSkillSnapshot() 扫描 skills/ 目录
                             过滤门控条件，格式化为 <available_skills> 文本
contextFiles              ← resolveBootstrapContextForRun() 扫描 workspace
                             加载 SOUL.md, .openclaw.md, TOOLS.md 等
runtimeInfo               ← 实时构建:
                             host: os.hostname()
                             os: os.type() + os.release()
                             arch: os.arch()
                             node: process.version
                             model: params.provider/params.modelId
                             channel: 从 sessionKey 解析
                             capabilities: 从渠道配置读取
userTimezone              ← params.config?.timezone 或 Intl.DateTimeFormat
reactionGuidance          ← 渠道配置（Telegram/Signal 的 reaction level）
sandboxInfo               ← resolveSandboxRuntimeStatus() 读取沙箱配置
promptMode                ← resolvePromptModeForSession() 根据 sessionKey 判断
                             主 session → "full"，子 agent → "minimal"
ownerNumbers              ← params.ownerNumbers（从配置读取的授权用户列表）
```

---

## 四、System Prompt 注入到 AI Session 的过程

```typescript
// attempt.ts 第 582-583 行
const systemPromptOverride = createSystemPromptOverride(appendPrompt);
let systemPromptText = systemPromptOverride();

// attempt.ts 第 687 行: 创建 agent session
const { session } = await createAgentSession({
  cwd: resolvedWorkspace,
  agentDir,
  authStorage: params.authStorage,
  modelRegistry: params.modelRegistry,
  model: params.model,
  tools: builtInTools,
  customTools: allCustomTools,
  sessionManager,
  settingsManager,
  resourceLoader,
});

// system-prompt.ts 第 91-103 行: 强制覆盖 session 的 system prompt
applySystemPromptOverrideToSession(session, systemPromptOverride);
// 内部做: session.agent.setSystemPrompt(prompt)

// attempt.ts 第 1179 行: 发送用户消息
await activeSession.prompt(effectivePrompt);
// → 此时 AI API 收到的是:
//    system: <buildAgentSystemPrompt 的完整输出>
//    messages: [...历史消息, { role: "user", content: "帮我搜索最新的AI信息" }]
//    tools: [{ name: "web_search", ... }, { name: "exec", ... }, ...]
```

---

## 五、完整调用链路图

```
用户发送 "帮我搜索最新的AI信息"
                │
                ↓
    ┌─ runEmbeddedPiAgent() ─────────────────────────────────┐
    │  src/agents/pi-embedded-runner/run.ts                   │
    │  准备: model, auth profile, context window 检查         │
    └────────────────┬───────────────────────────────────────┘
                     ↓
    ┌─ runEmbeddedAttempt() ─────────────────────────────────┐
    │  src/agents/pi-embedded-runner/run/attempt.ts           │
    │                                                         │
    │  ① 收集原材料:                                          │
    │     tools ← createOpenClawCodingTools()                 │
    │     skillsPrompt ← buildWorkspaceSkillSnapshot()        │
    │     contextFiles ← resolveBootstrapContextForRun()      │
    │     runtimeInfo ← os.hostname(), os.type(), ...         │
    │     userTimezone ← config 或 系统时区                    │
    │     reactionGuidance ← 渠道配置                         │
    │     sandboxInfo ← resolveSandboxRuntimeStatus()         │
    │     promptMode ← resolvePromptModeForSession()          │
    │                                                         │
    │  ② 组装 system prompt:                                  │
    │     buildEmbeddedSystemPrompt({ 所有原材料 })            │
    │       ↓                                                 │
    │     buildAgentSystemPrompt({ toolNames, toolSummaries,  │
    │       skillsPrompt, contextFiles, runtimeInfo, ... })   │
    │       ↓                                                 │
    │     lines = [                                           │
    │       "You are a personal assistant...",                 │
    │       "## Tooling", toolLines.join("\n"),                │
    │       "## Tool Call Style", ...,                         │
    │       "## Safety", ...,                                 │
    │       "## Skills (mandatory)", skillsPrompt,            │
    │       "## Memory Recall", ...,                          │
    │       "## Workspace", workspaceDir,                     │
    │       "# Project Context", ...contextFiles,             │
    │       "## Runtime", runtimeLine,                        │
    │     ]                                                   │
    │     return lines.join("\n")   ← 纯文本字符串             │
    │                                                         │
    │  ③ 注入到 session:                                      │
    │     createAgentSession({ tools, sessionManager, ... })  │
    │     applySystemPromptOverrideToSession(session, prompt) │
    │       → session.agent.setSystemPrompt(prompt)           │
    │                                                         │
    │  ④ 发送用户消息:                                        │
    │     activeSession.prompt("帮我搜索最新的AI信息")         │
    │                                                         │
    └────────────────┬───────────────────────────────────────┘
                     ↓
    ┌─ AI API 请求 ──────────────────────────────────────────┐
    │  POST https://api.anthropic.com/v1/messages             │
    │  {                                                      │
    │    system: "You are a personal assistant...\n\n         │
    │             ## Tooling\n- read: Read file contents\n    │
    │             - web_search: Search the web...\n...",      │
    │    messages: [                                           │
    │      ...历史消息,                                        │
    │      { role: "user", content: "帮我搜索最新的AI信息" }   │
    │    ],                                                   │
    │    tools: [                                              │
    │      { name: "web_search", input_schema: {...} },       │
    │      { name: "exec", input_schema: {...} },             │
    │      ...                                                │
    │    ]                                                     │
    │  }                                                      │
    └─────────────────────────────────────────────────────────┘
```

---

## 六、关键设计观察

### 1. System Prompt 是每次 attempt 重新构建的

不是启动时构建一次。每次 `runEmbeddedAttempt()` 都会重新调用 `buildEmbeddedSystemPrompt()`。这意味着：

- 工具列表可能因 policy 变化而不同
- 技能列表可能因环境变化而不同
- 运行时信息（时间、模型）是实时的

### 2. System Prompt 和 Tools 参数是分离的

发给 AI API 的请求有两个地方描述工具：

- `system` 字段里的 `## Tooling` 段落 → **一行文字描述**（给 AI 做宏观决策参考）
- `tools` 字段里的完整 schema → **JSON Schema**（AI 按此构造参数）

这是两层信息：一层是"概览"（在 prompt 里），一层是"精确定义"（在 tools schema 里）。

### 3. 纯字符串拼接，无模板引擎

整个 system prompt 就是一个 `string[]`，最后 `join("\n")`。没有 Jinja、Handlebars 或任何模板引擎。条件逻辑全靠 if 语句控制哪些段落被 push 进 lines 数组。

### 4. 对最小可迁移版本的意义

你只需要：

```typescript
function buildSystemPrompt(tools: Tool[], workspaceDir: string): string {
  const toolLines = tools.map((t) => `- ${t.name}: ${t.description}`).join("\n");
  return `你是一个 AI 助手。

## 可用工具
${toolLines}

不要叙述常规操作，直接调用工具。

## 工作目录
${workspaceDir}`;
}
```

OpenClaw 的 648 行代码之所以长，是因为它要处理 ~20 种条件（沙箱、多渠道、反应指导、心跳、TTS、记忆、技能、子agent 等）。如果你不需要这些特性，几十行就够了。
