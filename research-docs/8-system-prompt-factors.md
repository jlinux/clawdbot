# 影响 System Prompt 组装的因素分析

## 核心结论

每条消息都会重新组装 system prompt，而且大部分参数确实会因渠道、发送者、时间、工具策略等因素而不同。

---

## 调用频率

`runEmbeddedPiAgent()` → `runEmbeddedAttempt()` 是**每条用户消息进来都会调一次**的。每次调用都会重新执行 `buildEmbeddedSystemPrompt()`。

但在一次调用内的 agent loop（tool_use 循环）中，system prompt **不会重建**（见 `7-system-prompt-assembly.md`）。

---

## 三类因素

### 一、几乎不变的（启动后稳定）

| 参数                  | 来源                     | 为什么不变 |
| --------------------- | ------------------------ | ---------- |
| `workspaceDir`        | agent 的工作目录         | 启动后固定 |
| `docsPath`            | OpenClaw 文档路径        | 安装后固定 |
| `ownerNumbers`        | 配置文件中的授权用户列表 | 除非改配置 |
| `sandboxInfo`         | 沙箱配置                 | 除非改配置 |
| `heartbeatPrompt`     | 心跳提示词配置           | 除非改配置 |
| `ttsHint`             | TTS 语音配置             | 除非改配置 |
| `modelAliasLines`     | 模型别名配置             | 除非改配置 |
| `memoryCitationsMode` | 记忆引用模式             | 除非改配置 |

### 二、每条消息可能不同的（动态因素）

| 参数                           | 来源                                | 为什么会变                                        |
| ------------------------------ | ----------------------------------- | ------------------------------------------------- |
| **`messageChannel`**           | 消息来源渠道                        | 同一用户可能从 Telegram 发一条、从 Discord 发一条 |
| **`runtimeInfo.channel`**      | 同上派生                            | 影响 Runtime 行显示的渠道                         |
| **`runtimeInfo.capabilities`** | 渠道能力（inlineButtons 等）        | 不同渠道能力不同                                  |
| **`reactionGuidance`**         | Telegram/Signal 特定的表情反应指导  | 只在这两个渠道有值                                |
| **`messageToolHints`**         | 渠道特定的消息工具提示              | 不同渠道不同                                      |
| **`channelActions`**           | 渠道支持的操作（react/edit/unsend） | 不同渠道不同                                      |
| **`promptMode`**               | "full" / "minimal" / "none"         | 主 session → full，子 agent → minimal             |
| **`extraSystemPrompt`**        | 群聊上下文 / 子 agent 上下文        | 群聊 vs 私聊 vs 子 agent 场景不同                 |
| **`provider` / `modelId`**     | 当前使用的 AI 模型                  | 可能 failover 到另一个模型                        |
| **`runtimeInfo.model`**        | 同上                                | 显示在 Runtime 行中                               |
| **`reasoningTagHint`**         | 是否要求 `<think>` 标签             | 取决于当前 provider（比如 DeepSeek 需要）         |
| **`thinkLevel`**               | thinking 级别                       | 可能按配置或 failover 变化                        |
| **`userTime`**                 | 当前时间                            | 每条消息的时间戳不同                              |
| **`senderId` / `senderName`**  | 发送者信息                          | 不同用户发来的消息                                |
| **`senderIsOwner`**            | 发送者是否是 owner                  | 影响工具策略                                      |

### 三、每条消息重新加载的（文件系统扫描）

| 参数                 | 来源                                                            | 说明                                                                          |
| -------------------- | --------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| **`tools`**          | `createOpenClawCodingTools()`                                   | 每次 attempt **重新创建**全部工具实例。工具列表受渠道、sandbox、sender 等影响 |
| **`skillsPrompt`**   | `loadWorkspaceSkillEntries()` + `buildWorkspaceSkillSnapshot()` | 每次 attempt **重新扫描** `skills/` 目录，过滤门控条件                        |
| **`contextFiles`**   | `resolveBootstrapContextForRun()`                               | 每次 attempt **重新读取** `SOUL.md`, `.openclaw.md` 等项目上下文文件          |
| **`workspaceNotes`** | 从 bootstrap 文件判断                                           | 取决于 contextFiles 的结果                                                    |

---

## 具体示例

假设同一个 agent 连续收到两条消息：

```
消息 A: 来自 Telegram 群聊，user=alice (owner)
消息 B: 来自 Discord 私聊，user=bob (non-owner)
```

它们的 system prompt **至少**有以下差异：

```diff
 ## Tooling
 - read: Read file contents
 - exec: Execute shell commands
 - web_search: Search the web
+- message: Send messages          ← A 有（owner），B 可能没有（non-owner 策略限制）

-## Runtime
-Runtime: channel=telegram          ← A
+Runtime: channel=discord           ← B

-## Reactions
-Telegram reaction guidance: ...    ← A 有
+（无 reaction 段落）                ← B 没有（Discord 无 reaction guidance）

-## Group Chat Context
-You are in a group chat...         ← A 有（群聊）
+（无此段落）                        ← B 没有（私聊）

-## Current Date & Time
-Time: 2026-02-25 14:30             ← A 的时间
+Time: 2026-02-25 14:31             ← B 的时间
```

---

## 影响 System Prompt 的完整调用链

```
用户消息到达
  ↓
runEmbeddedPiAgent()                    ← src/agents/pi-embedded-runner/run.ts
  ↓
runEmbeddedAttempt(params)              ← src/agents/pi-embedded-runner/run/attempt.ts
  │
  ├── resolveSandboxContext()           → sandboxInfo
  ├── loadWorkspaceSkillEntries()       → skillsPrompt（扫描 skills/ 目录）
  ├── resolveBootstrapContextForRun()   → contextFiles（读 SOUL.md 等）
  ├── createOpenClawCodingTools()       → tools（创建全部工具实例）
  ├── resolveChannelCapabilities()      → runtimeCapabilities
  ├── resolveTelegramReactionLevel()    → reactionGuidance
  ├── resolveChannelMessageToolHints()  → messageToolHints
  ├── buildSystemPromptParams()         → runtimeInfo, userTimezone, userTime
  ├── resolvePromptModeForSession()     → promptMode
  ├── buildModelAliasLines()            → modelAliasLines
  ├── resolveOwnerDisplaySetting()      → ownerDisplay
  │
  ↓ 所有参数就绪
buildEmbeddedSystemPrompt({ ... })      ← 组装一次
  ↓
buildAgentSystemPrompt({ ... })         ← 纯字符串拼接，~25 段
  ↓
session.agent.setSystemPrompt(prompt)   ← 注入到 session
  ↓
session.prompt(userText)                ← 开始 agent loop
  ↓
┌─── agent loop ────────────────────┐
│  使用同一份 system prompt          │
│  直到循环结束                      │
└───────────────────────────────────┘
```

---

## 为什么这样设计？

```
目标：让 AI 在每次回复时都有最准确的上下文
  ↓
代价：每条消息都要重新组装 system prompt（~1-2ms，可忽略）
  ↓
收益：
  ✅ 渠道切换时 AI 自动知道当前在哪个渠道
  ✅ 工具策略变化立即生效（不需要重启）
  ✅ 上下文文件修改后下一条消息就能看到
  ✅ 时间戳永远是最新的
  ✅ 模型 failover 后 Runtime 信息准确
```

---

## 对最小可迁移版本的启示

如果你的项目只有单渠道（比如只有 Web 或只有 Telegram），那大部分动态因素都不存在：

```typescript
// 单渠道场景：system prompt 几乎是静态的
function buildSystemPrompt(tools: Tool[], config: Config): string {
  const toolLines = tools.map((t) => `- ${t.name}: ${t.description}`).join("\n");
  return `你是一个 AI 助手。

## 可用工具
${toolLines}

## 工作目录
${config.workspaceDir}

## 当前时间
${new Date().toISOString()}`;
}
```

只有当你需要**多渠道**（Telegram + Discord + Web）或**多用户权限**（owner vs non-owner 工具策略）时，才需要像 OpenClaw 那样每条消息动态重建。

| 场景                        | 是否需要动态重建                      |
| --------------------------- | ------------------------------------- |
| 单渠道 + 单用户             | ❌ 启动时构建一次就够                 |
| 单渠道 + 多用户（权限不同） | ⚠️ 工具列表可能因权限不同             |
| 多渠道                      | ✅ 渠道能力、reaction、消息提示都不同 |
| 多渠道 + 多用户 + 沙箱      | ✅ 完整动态重建（OpenClaw 的做法）    |
