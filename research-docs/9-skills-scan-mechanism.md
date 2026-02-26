# Skills 扫描机制："scan available_skills" 到底发生了什么

## 核心结论

system prompt 里的"scan \<available_skills\>"**不是让代码去扫描，而是告诉 AI "你眼前有一段 XML，请在回复前先看看它"**。技能列表已经被代码预先嵌入了。AI 的"扫描"只是阅读 system prompt 中已有的文本。

---

## 两个阶段

### 阶段 1：代码层面 — 在 `buildEmbeddedSystemPrompt` 时（agent loop 之前）

技能列表**已经被预先扫描并嵌入到 system prompt 中**：

```
attempt.ts
  ↓
loadWorkspaceSkillEntries()          ← 扫描 skills/ 目录（代码做的，fs.readdirSync）
  ↓
filterSkillEntries()                 ← 过滤门控条件（bins/env/os）
  ↓
applySkillsPromptLimits()            ← 限制 token 预算（最多 150 个，30000 字符）
  ↓
formatSkillsForPrompt()              ← 格式化为 XML 文本
  ↓
嵌入到 system prompt 的 ## Skills 段落中
```

#### formatSkillsForPrompt() 的输出

源码位置：`node_modules/@mariozechner/pi-coding-agent/dist/core/skills.js:215`

```typescript
export function formatSkillsForPrompt(skills) {
  const lines = [
    "The following skills provide specialized instructions for specific tasks.",
    "Use the read tool to load a skill's file when the task matches its description.",
    "",
    "<available_skills>",
  ];
  for (const skill of visibleSkills) {
    lines.push("  <skill>");
    lines.push(`    <name>${escapeXml(skill.name)}</name>`);
    lines.push(`    <description>${escapeXml(skill.description)}</description>`);
    lines.push(`    <location>${escapeXml(skill.filePath)}</location>`);
    lines.push("  </skill>");
  }
  lines.push("</available_skills>");
  return lines.join("\n");
}
```

#### 最终嵌入 system prompt 的内容

源码位置：`src/agents/system-prompt.ts:20`

```xml
## Skills (mandatory)
Before replying: scan <available_skills> <description> entries.
- If exactly one skill clearly applies: read its SKILL.md at <location> with `read`, then follow it.
- If multiple could apply: choose the most specific one, then read/follow it.
- If none clearly apply: do not read any SKILL.md.
Constraints: never read more than one skill up front; only read after selecting.

<available_skills>
  <skill>
    <name>github</name>
    <description>github operations via `gh` CLI</description>
    <location>~/.openclaw/skills/github/SKILL.md</location>
  </skill>
  <skill>
    <name>nano-pdf</name>
    <description>Edit PDFs with nano-pdf CLI</description>
    <location>~/.openclaw/skills/nano-pdf/SKILL.md</location>
  </skill>
  ...
</available_skills>
```

**这段文本在 AI 收到第一条消息之前就已经在 system prompt 里了，不需要任何工具调用。**

### 阶段 2：AI 模型层面 — 在 agent loop 中

当用户说"帮我创建一个 PR"时，AI 模型的内部推理过程：

```
┌─────────────────────────────────────────────────────────────┐
│  AI 读到 system prompt 里的指令:                              │
│  "Before replying: scan <available_skills>"                  │
│                                                              │
│  AI 看到 <available_skills> 里的内容:                         │
│    github → "github operations via `gh` CLI"                 │
│    nano-pdf → "Edit PDFs with nano-pdf CLI"                  │
│                                                              │
│  AI 内部判断:                                                 │
│    用户要创建 PR → 匹配 github skill                          │
│    location = ~/.openclaw/skills/github/SKILL.md             │
│                                                              │
│  → 决定先调 read 工具读取 SKILL.md                            │
│                                                              │
│  这一步是 AI 模型的自然语言理解，不是代码逻辑                   │
└─────────────────────────────────────────────────────────────┘
        │
        ↓ agent loop 第 1 轮
┌─────────────────────────────────────────────────────────────┐
│  AI 输出: tool_use { name: "read",                           │
│    args: { path: "~/.openclaw/skills/github/SKILL.md" } }   │
│                                                              │
│  SDK 执行 read 工具 → 返回 SKILL.md 的内容                    │
│  (比如: "用 `gh pr create --title '...'` 创建 PR")           │
└─────────────────────────────────────────────────────────────┘
        │
        ↓ agent loop 第 2 轮
┌─────────────────────────────────────────────────────────────┐
│  AI 学到了 SKILL.md 的内容                                    │
│  → 输出: tool_use { name: "exec",                            │
│    args: { command: "gh pr create --title '...' ..." } }     │
│                                                              │
│  SDK 执行 exec 工具 → 返回结果                                │
└─────────────────────────────────────────────────────────────┘
        │
        ↓ agent loop 第 3 轮
┌─────────────────────────────────────────────────────────────┐
│  AI: "PR 已创建: https://github.com/..."                     │
│  stop_reason: end_turn                                       │
└─────────────────────────────────────────────────────────────┘
```

---

## 关键区分表

| 动作                            | 谁做的                             | 什么时候                                | 怎么做的                                  |
| ------------------------------- | ---------------------------------- | --------------------------------------- | ----------------------------------------- |
| **扫描 skills/ 目录**           | 代码 (`loadWorkspaceSkillEntries`) | system prompt 组装时（agent loop 之前） | `fs.readdirSync()` 遍历目录               |
| **过滤不符合条件的 skill**      | 代码 (`filterSkillEntries`)        | 同上                                    | 检查 bins/env/os 门控条件                 |
| **格式化为 XML**                | 代码 (`formatSkillsForPrompt`)     | 同上                                    | 字符串拼接                                |
| **"扫描 \<available_skills\>"** | AI 模型                            | agent loop 第 1 轮之前（模型内部推理）  | 读 system prompt 里已有的 XML 文本        |
| **判断哪个 skill 匹配**         | AI 模型                            | 同上                                    | 语义理解（用户意图 vs skill description） |
| **读取 SKILL.md**               | AI 模型通过 `read` 工具            | agent loop 第 1 轮                      | tool_use → read(path)                     |
| **按 SKILL.md 执行**            | AI 模型通过各种工具                | agent loop 第 2+ 轮                     | tool_use → exec/write/...                 |

---

## 为什么不把 SKILL.md 的全部内容都塞到 system prompt 里？

Skill 的设计是**按需加载**而不是全量注入：

```
全量注入（❌ 不采用）:
  把所有 SKILL.md 内容嵌入 system prompt
  → 可能有 50+ 个 skill，每个几百行
  → system prompt 膨胀到几万 token
  → 绝大部分 skill 与当前任务无关，浪费 context window

按需加载（✅ 采用）:
  只注入 name + description + location（每个 ~3 行 XML）
  → 50 个 skill 也只占 ~150 行
  → AI 根据 description 判断，匹配后才读取完整 SKILL.md
  → 每次只读 1 个 SKILL.md，节省 context window
```

这也是 system prompt 里写 "never read more than one skill up front" 的原因——防止 AI 贪心地读多个 SKILL.md 浪费 token。

---

## 对最小可迁移版本的启示

如果你想实现类似的 Skill 系统，只需要在 system prompt 里加这段话：

```
如果用户的任务匹配以下技能，先用 read 工具读取对应文件，再按文件指引执行：

<available_skills>
  <skill>
    <name>github</name>
    <description>GitHub 操作（PR、Issue、Release）</description>
    <location>/path/to/skills/github/SKILL.md</location>
  </skill>
</available_skills>
```

不需要任何代码框架——Skill 的核心是 system prompt 里的一段文字指引 + 一个 AI 可读取的 Markdown 文件。
