# OpenClaw 中 Tool 与 Skill 的关系

## 一句话结论

**`web_search` 是 Tool，不是 Skill。** Tool 和 Skill 是两个完全不同的系统。

---

## 核心区别

```
Tool = 能力（AI 能做什么）
Skill = 说明书（教 AI 怎么做）
```

| 维度                    | Tool                                             | Skill                                        |
| ----------------------- | ------------------------------------------------ | -------------------------------------------- |
| **本质**                | 硬编码的函数                                     | Markdown 文档（SKILL.md）                    |
| **AI 怎么用**           | 直接调用（tool_use 块）                          | 先 read 文件学习，再用 Tool 执行             |
| **执行方式**            | SDK 拦截 → 执行函数 → 返回结果                   | AI 读完说明书后自己决定调哪些 Tool           |
| **定义位置**            | `src/agents/tools/*.ts`                          | `skills/*/SKILL.md` 或 `~/.openclaw/skills/` |
| **在 system prompt 中** | `## Tooling` 区域列出                            | `## Skills (mandatory)` 区域列出             |
| **示例**                | `web_search`, `exec`, `read`, `write`, `browser` | `github`, `nano-pdf`, `himalaya`             |

---

## 具体的执行差异

### Tool 的执行流程（以 web_search 为例）

```
用户: "搜索最新AI信息"
  ↓
AI 直接输出 tool_use 块:
  { name: "web_search", input: { query: "latest AI news" } }
  ↓
SDK 拦截 → 调用 web_search.execute() 函数
  ↓
函数内部: HTTP 请求 Brave Search API
  ↓
返回搜索结果给 AI
```

**关键**：AI 直接调用函数，不需要先"学习"任何东西。

### Skill 的执行流程（以 github skill 为例）

```
用户: "帮我创建一个 PR"
  ↓
AI 看到 system prompt 里的 <available_skills>:
  "github (🐙): github operations via `gh` CLI..."
  ↓
AI 判断: 这个任务和 github skill 相关
  ↓
AI 调用 Tool `read`:
  { name: "read", input: { path: "~/.openclaw/skills/github/SKILL.md" } }
  ↓
AI 读到 SKILL.md 的内容，学到:
  "用 `gh pr create --title '...' --body '...'` 创建 PR"
  ↓
AI 调用 Tool `exec`:
  { name: "exec", input: { command: "gh pr create --title '...' --body '...'" } }
  ↓
返回结果给 AI
```

**关键**：Skill 本身不执行任何代码。它只是一份说明书，教 AI 用哪些 Tool、怎么用。

---

## 在 System Prompt 中长什么样

```markdown
## Tooling

Tool availability (filtered by policy):

- read: Read file contents
- write: Create or overwrite files
- edit: Make precise edits to files
- exec: Execute shell commands
- web_search: Search the web (Brave API) ← Tool
- web_fetch: Fetch and extract content from URL ← Tool
- browser: Control web browser ← Tool
- message: Send messages to channels
  ... (共 ~28 个)

## Skills (mandatory)

Before replying: scan <available_skills> <description> entries.

- If exactly one skill clearly applies: read its SKILL.md, then follow it.
- If multiple could apply: choose the most specific one.
- If none clearly apply: do not read any SKILL.md.

<available_skills>

- github (🐙): github operations via `gh` CLI ← Skill
- nano-pdf (📄): Edit PDFs with nano-pdf CLI ← Skill
- himalaya (📮): Email operations via Himalaya ← Skill
  ... (按 token 预算截断)
  </available_skills>
```

---

## Skill 的加载和过滤机制

Skill 不是无条件加载的，有**门控条件**（frontmatter 里的 `metadata.openclaw.requires`）：

```yaml
# skills/nano-pdf/SKILL.md 的 frontmatter
---
name: nano-pdf
description: Edit PDFs with natural-language
metadata:
  openclaw:
    emoji: "📄"
    requires:
      bins: ["nano-pdf"] # PATH 里必须有这个命令
      env: ["GEMINI_API_KEY"] # 必须设置这个环境变量
    os: ["darwin", "linux"] # 只在这些 OS 上可用
---
```

**加载优先级**（高 → 低）：

```
1. <workspace>/skills/      ← 项目级（最高优先）
2. ~/.openclaw/skills/       ← 用户级
3. extensions/*/skills/      ← 插件提供
4. 配置的 extraDirs          ← 额外目录
5. npm 包内置               ← 兜底
```

**Token 预算控制**：

- 最多注入 150 个 skill 描述
- 总字符数上限 30,000
- 用二分搜索找到在预算内能放下的最大 skill 子集

---

## Skill 也可以被用户直接触发

Skill 除了被 AI 自动识别使用外，还可以作为**斜杠命令**暴露给用户：

```
用户输入: /github pr list
  ↓
SkillCommandSpec 匹配到 github skill
  ↓
如果 command-dispatch: tool → 直接路由到工具
如果没有特殊路由 → 把 skill 内容注入到 agent prompt，让 AI 执行
```

frontmatter 控制：

```yaml
user-invocable: true # 是否暴露为斜杠命令（默认 true）
command-dispatch: tool # 可选：跳过 AI 直接调工具
command-tool: exec # 直接路由到哪个工具
```

---

## 关系图

```
┌─────────────────────────────────────────────────────────┐
│                    System Prompt                        │
│                                                         │
│  ┌─────────────────┐    ┌──────────────────────────┐   │
│  │   ## Tooling     │    │   ## Skills (mandatory)   │   │
│  │                  │    │                            │   │
│  │  web_search      │    │  github (🐙)              │   │
│  │  web_fetch       │    │  nano-pdf (📄)            │   │
│  │  exec            │    │  himalaya (📮)            │   │
│  │  read   ←────────┼────┼── Skill 用 read 工具     │   │
│  │  write           │    │    读取 SKILL.md          │   │
│  │  edit            │    │                            │   │
│  │  browser         │    │  读完后用 exec 等工具      │   │
│  │  message         │    │    执行实际操作            │   │
│  │  ...             │    │                            │   │
│  └─────────────────┘    └──────────────────────────┘   │
│         ↑                          ↑                    │
│    直接可调用                  间接使用（先读后做）        │
└─────────────────────────────────────────────────────────┘
                         │
                    AI 模型 (Claude)
                         │
              ┌──────────┴──────────┐
              ↓                     ↓
        调用 Tool                读取 Skill
      (tool_use 块)            (用 read Tool)
              ↓                     ↓
        SDK 执行函数            学到工作流程
        返回结果               再用 Tool 执行
```

---

## 对最小可迁移版本的启示

| 系统           | 是否需要迁移         | 原因                                     |
| -------------- | -------------------- | ---------------------------------------- |
| **Tool 系统**  | **必须**             | 这是 AI 的"手和脚"，没有工具 AI 只能说话 |
| **Skill 系统** | **可选，但强烈推荐** | 不需要写代码就能教 AI 新能力             |

**Skill 系统的精妙之处**：

- 不需要写 TypeScript 就能扩展 AI 的能力
- 只要写一份 Markdown 说明书，AI 就能学会使用任何 CLI 工具
- 用户可以自己创建 Skill，不需要开发者介入
- 门控机制确保只有环境满足条件的 Skill 才会出现

**最小版本的 Skill 实现**：在 system prompt 里加一段话就行：

```
如果用户的任务涉及以下工具，先读取对应的说明文件再执行：
- GitHub 操作: 读取 skills/github/SKILL.md
- PDF 编辑: 读取 skills/nano-pdf/SKILL.md
```

不需要任何代码框架——Skill 本质就是 system prompt 里的一段文字指引。
