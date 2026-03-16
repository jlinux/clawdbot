# OpenClaw 研究问题记录

> **约束**：每次新的研究问题都必须追加到本文件中，保持编号连续。

---

## Q1. OpenClaw 的消息执行流程是什么？能否迁移到其他项目？

**对应文档**: `1-message-pipeline.md`

我想了解我发送信息给 OpenClaw 以后，它的执行方式，我想看能否把这套机制挪移到其他项目中。

---

## Q2. Agent 循环部分的最小可迁移设计方案是什么？

**对应文档**: `2-agent-loop-minimal-design.md`

帮我深入分析 agent 循环那部分，写一个最小可迁移版本的设计方案。

---

## Q3. 发送"帮我搜索最新的AI信息"时，完整执行路径是什么？

**对应文档**: `3-message-trace-simulation.md`

假设我发送一个消息：帮我搜索最新的AI信息，它是如何执行的，帮我模拟一下路径，包括可能用了哪些 ToolRegistry。

---

## Q4. Agent Loop 中"如果摘要不够详细"的判断是如何做出的？

**对应文档**: 包含在 `2-agent-loop-minimal-design.md` 的讨论中

如何判断"如果摘要不够详细"？

**结论**: 没有程序化逻辑判断，完全由 AI 模型根据 stop_reason 自行决定。

---

## Q5. web_search 和 ToolRegistry、Skills 之间的关系是什么？

**对应文档**: `4-tools-vs-skills.md`

web_search 是对应一个 SKILL 还是什么？ToolRegistry 和 SKILLS 之间的关系是什么？

---

## Q6. 类似 web_search 的系统默认 tools 有哪些？如何定义和实现的？

**对应文档**: `5-system-default-tools.md`

类似 web_search 的系统默认 tools 有哪些？都是如何定义和实现的？

---

## Q7. AI SDK（@mariozechner/pi-\*）是什么？核心用途是什么？

**对应文档**: 包含在对话讨论中（未单独保存）

这里提到的 AI SDK 是什么？它的核心用途是什么？

**结论**: 4 个包 — pi-agent-core（类型）、pi-ai（API 调用）、pi-coding-agent（会话管理）、pi-tui（终端 UI）。

---

## Q8. OpenClaw 有初始 prompt 来解读消息并转化为执行吗？

**对应文档**: `6-system-prompt-analysis.md`

假设我发送一个消息：帮我搜索最新的AI信息，OpenClaw 有初始的 prompt 来解读信息，转化为后续的执行吗？

**结论**: 没有任何意图路由/NLU 逻辑，system prompt 只列工具清单，全靠模型自己决定。

---

## Q9. system-prompt.ts 的组装逻辑和调用链是什么？

**对应文档**: `7-system-prompt-assembly.md`

帮我解读 src/agents/system-prompt.ts，告诉我它组装的逻辑，以及被调用的逻辑。

---

## Q10. Agent Loop 中会多次调用 system prompt 的组装吗？

**对应文档**: `8-system-prompt-factors.md`

在 Agent 层 — AI 对话循环 + 工具执行的时候，会多次调用 system-prompt 的组装吗？

**结论**: 每次 attempt（一条用户消息进来时）组装一次，在整个 tool 执行循环中保持不变。

---

## Q11. 影响 system prompt 组装的因素有哪些？每条消息的 system prompt 一样吗？

**对应文档**: `8-system-prompt-factors.md`

影响 buildEmbeddedSystemPrompt 组装 system prompt 的因素有哪些？每个 message 过来的 system prompt 的因素是一样的还是不一样的？

**结论**: 每条消息都会重新组装，因渠道、发送者、时间、工具策略等因素可能不同。三类因素：几乎不变的（配置）、每条消息可能不同的（渠道/发送者/时间）、每次重新加载的（文件系统扫描）。

---

## Q12. Skills 扫描机制："scan available_skills" 到底发生了什么？

**对应文档**: `9-skills-scan-mechanism.md`

技能（强制）回复前：扫描 \<available_skills\> 的 \<description\> 条目。这个会发生什么，是在 agent loop 中去调用扫描和注入吗？

**结论**: "扫描"不是代码行为。技能列表已被代码预先嵌入到 system prompt 的 XML 中，AI 的"扫描"只是阅读 system prompt 中已有的文本，匹配后通过 read 工具读取 SKILL.md。

---

## Q13. System prompt 对 LLM 返回的要求是什么？Agent loop 如何判断结束？

**对应文档**: `10-agent-loop-stop-mechanism.md`

system prompt 对 LLM 返回的要求是什么以便于 agent loop 可以持续运行，直到判断为结束？

**结论**: System prompt 完全不干预 LLM 的返回格式。循环判断完全依赖 AI API 原生的 tool calling 协议——回复中有 tool_use 就继续，没有就结束。

---

## Q14. Tool Calling 协议有什么特点？其他 API 协议？调用技巧？LLM 支持情况？

**对应文档**: `10-agent-loop-stop-mechanism.md`（附录部分）

当前的 AI API 原生的 tool calling 协议，有什么特点，并且 AI API 还有哪些协议，调用的时候，有什么技巧和要求吗？是不是所有的 LLM 都支持？

---

## Q15. 会话（Session）管理和上下文窗口压缩机制是什么？

**对应文档**: `11-session-and-context-compaction.md`

多轮对话如何维持上下文？token 快满时如何压缩？Session 的生命周期是什么？

---

## Q16. 模拟一个完整的 Session Context 案例（搜索AI新闻）

**对应文档**: `12-session-context-simulation.md`

模拟一个 Session 的 context 案例，从消息到达、Session 加载、agent loop 执行、到 context 累积触发 Compaction 的完整生命周期。

---

## Q17. Model Failover（模型故障切换）机制是什么？

**对应文档**: `13-model-failover.md`

当主模型不可用时，如何切换到备用模型？failover 策略和重试逻辑是怎样的？

---

## Q18. Tool 执行的错误处理和重试机制是什么？

**对应文档**: `14-tool-error-handling.md`

当 tool 执行失败（如 web_search 超时、exec 报错），代码如何处理？错误如何反馈给 AI 让它自我纠正？

---

## Q19. 多渠道消息路由机制是什么？

**对应文档**: `15-multi-channel-routing.md`

消息如何从 Telegram/Discord 等不同渠道路由到同一个 agent？渠道插件的抽象层是怎样的？

---

## Q20. Memory/记忆系统的实现是什么？

**对应文档**: `16-memory-system.md`

长期记忆如何存储、检索、注入到对话中？memory_search 和 memory_get 工具的底层实现？

---

## Q21. Tool 的安全策略和权限控制是什么？

**对应文档**: `17-tool-security-policy.md`

owner vs non-owner 的工具白名单/黑名单机制？senderIsOwner 如何影响工具策略？

---

## Q22. Plugin 系统的加载和生命周期是什么？

**对应文档**: `18-plugin-system.md`

`src/plugins/` 用 jiti 动态加载插件，plugin 如何注册 tools、channels、hooks？SDK 接口是什么？

---

## Q23. 子 Agent 机制是什么？

**对应文档**: `19-sub-agent-mechanism.md`

什么场景会启动子 agent？子 agent 和主 agent 的关系？promptMode "minimal" 的设计？

---

## Q24. 媒体处理管道是什么？

**对应文档**: `20-media-pipeline.md`

用户发图片时如何传给 AI（vision API）？语音消息如何转文字？`src/media/` 的处理流程？

---

## Q25. OpenClaw 如何将不同的 LLM 分配到不同的用途上？

**对应文档**: `21-model-routing-by-purpose.md`

不同场景（主对话、图片分析、语音转文字、嵌入生成、摘要压缩、子 Agent 等）如何选用不同模型？模型选择的配置和运行时逻辑是怎样的？

---

## Q26. 从 git commit 历史梳理 OpenClaw 的功能研发路径和演化

**对应文档**: `22-development-evolution.md`

基于 14,362 条 commit 的完整分析，追踪项目从第一行代码（warelay WhatsApp webhook）到当前多渠道 AI 助手平台的六个演化阶段：原型 → Web 化 → 助手能力 → macOS 伴侣 → 多节点平台 → 开放平台/企业级。

---

## 文档索引

| 编号 | 文档                                   | 主题                                    |
| ---- | -------------------------------------- | --------------------------------------- |
| 0    | `0-research-questions.md`              | 研究问题记录（本文件）                  |
| 1    | `1-message-pipeline.md`                | 消息处理管道                            |
| 2    | `2-agent-loop-minimal-design.md`       | 最小可迁移 Agent Loop 设计              |
| 3    | `3-message-trace-simulation.md`        | 消息执行路径模拟                        |
| 4    | `4-tools-vs-skills.md`                 | Tool 与 Skill 的关系                    |
| 5    | `5-system-default-tools.md`            | 系统默认工具全景                        |
| 6    | `6-system-prompt-analysis.md`          | System Prompt 分析                      |
| 7    | `7-system-prompt-assembly.md`          | System Prompt 组装逻辑                  |
| 8    | `8-system-prompt-factors.md`           | System Prompt 动态因素                  |
| 9    | `9-skills-scan-mechanism.md`           | Skills 扫描机制                         |
| 10   | `10-agent-loop-stop-mechanism.md`      | Agent Loop 停止机制 + Tool Calling 协议 |
| 11   | `11-session-and-context-compaction.md` | 会话管理 + 上下文压缩机制               |
| 12   | `12-session-context-simulation.md`     | Session Context 模拟案例                |
| 13   | `13-model-failover.md`                 | Model Failover 机制                     |
| 14   | `14-tool-error-handling.md`            | Tool 错误处理和重试                     |
| 15   | `15-multi-channel-routing.md`          | 多渠道消息路由                          |
| 16   | `16-memory-system.md`                  | Memory 记忆系统                         |
| 17   | `17-tool-security-policy.md`           | Tool 安全策略和权限                     |
| 18   | `18-plugin-system.md`                  | Plugin 系统                             |
| 19   | `19-sub-agent-mechanism.md`            | 子 Agent 机制                           |
| 20   | `20-media-pipeline.md`                 | 媒体处理管道                            |
| 21   | `21-model-routing-by-purpose.md`       | 不同用途的 LLM 分配                     |
| 22   | `22-development-evolution.md`          | 功能研发路径与演化分析                   |
