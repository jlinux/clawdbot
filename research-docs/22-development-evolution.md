# OpenClaw 功能研发路径与演化分析

基于 14,362 条 git commit 的完整梳理，追踪项目从第一行代码到当前状态的演化轨迹。

---

## 一、总体演化阶段概览

| 阶段 | 时间线(推断) | 核心主题 | 版本 |
|------|-------------|---------|------|
| 0. 原型 | 最早期 | Twilio WhatsApp webhook + Claude 自动回复 | 0.1.x |
| 1. Web 化 | 早期 | Baileys WhatsApp Web + 去 Twilio 依赖 | 0.1.x → 1.0.x |
| 2. 助手能力 | 中期 | 多媒体、心跳、分组、Agent CLI | 1.1.x → 1.4.x |
| 3. macOS 伴侣 | 中期 | SwiftUI 菜单栏应用 + Voice Wake | 1.4.x → 2.0.0-beta |
| 4. 多节点平台 | 中后期 | iOS/Android 节点 + Canvas + Gateway 协议 | 2.0.0-beta |
| 5. 开放平台 | 后期 | 多渠道(Discord/Slack/Signal/iMessage) + 插件 + 技能 | 2026.1.x |
| 6. 企业级 | 当前 | 多 Agent 路由、沙箱、安全加固、社区贡献 | 2026.2.x |

---

## 二、阶段 0：最小原型 — "warelay"

**项目起点：一个 Twilio WhatsApp webhook 转发器。**

核心 commits：
```
f6dd362d3 Initial commit
16dfc1a5b Add warelay CLI with Twilio webhook support
42f64279d Log command execution in config-driven replies
b4a995dbe Add claude auto-reply allowlist and verbose hooks
```

这个阶段的产品极其简单：
- 一个 Express 服务器，监听 Twilio webhook
- 收到 WhatsApp 消息后，调用 Claude CLI 生成回复
- 通过 Twilio API 发回 WhatsApp
- 支持 Tailscale Funnel 暴露 webhook
- 配置驱动的自动回复（allowlist 控制谁可以触发）

**关键设计决策：**
- 名字叫 "warelay"（WhatsApp Relay）
- 使用 Tailscale Funnel 而非公网暴露
- 从第一天起就有 allowlist 安全概念

---

## 三、阶段 1：从 Twilio 到 WhatsApp Web

**关键转折：引入 Baileys 库，直接连接 WhatsApp Web，去掉 Twilio 依赖。**

```
3c8a10516 Add WhatsApp Web provider option and docs
73a3463ec Add web provider inbound monitor with auto-replies
289b417c8 Pin to @whiskeysockets/baileys 7.0.0-rc.9
ba3b271c3 Parse Claude JSON output to return text replies
948ff7f03 feat: add image support across web and twilio
```

这是第一个重大架构转变：
- **从 SaaS 依赖（Twilio）变成自托管**——这定义了整个项目的方向
- 添加了 QR 码登录（扫码连接 WhatsApp）
- 支持图片收发
- 添加了音频转录功能
- 实现了 "same-phone mode"（同一部手机收发）

```
d88ede92b feat: same-phone mode with echo detection and configurable marker
21ba0fb8a Add auto-recovery from stuck WhatsApp sessions
7b7c148f4 Verbose mode now prints stdout/stderr of invoked commands
```

**最终在这个阶段末尾，Twilio 被完全移除：**
```
7c7314f67 chore: drop twilio and go web-only
e13dc02e7 chore: remove twilio and expand pi cli detection
```

---

## 四、阶段 2：从 "转发器" 到 "助手"

### 4.1 Agent 系统的诞生

```
f31e89d5a Agents: add pluggable CLIs
05b76281f CLI: add agent command for direct agent runs
```

从单纯的"消息转发+Claude回复"变成了可插拔的 Agent 系统。Agent 不再只是调用 Claude CLI，而是有了独立的会话管理。

### 4.2 心跳系统（Heartbeat）

```
271004bf6 feat: add heartbeat cli and relay trigger
135d930c9 feat: add heartbeat idle override and preserve session freshness
```

心跳系统让助手**主动**发消息，而不只是被动回复。这是从"聊天机器人"到"个人助手"的关键一步——助手可以定时检查、提醒、汇报。

### 4.3 群聊支持

```
6afe6f4ec feat(web): add group chat mention support
cb5f1fa99 feat(web): prime group sessions with member roster
```

支持 WhatsApp 群聊中通过 @mention 触发助手，群聊中需要 mention 才响应（避免噪音）。

### 4.4 工具与 Verbose 模式

```
ae0d35c72 Auto-reply: add /verbose directives and tool result replies
c3792db0e Auto-reply: stream verbose tool results via tau rpc
```

用户可以在 WhatsApp 中通过 `/verbose` 看到 AI 正在调用什么工具——这是透明度的重要一步。

### 4.5 Thinking 指令

```
58520859e Auto-reply: add thinking directives
8ba35a2dc Auto-reply: ack think directives
```

用户可以在聊天中控制 AI 的"思考深度"（thinking levels），这是一个面向高级用户的能力。

---

## 五、阶段 3：macOS 伴侣应用 — 从 CLI 到 GUI

**这是产品形态的重大飞跃。**

### 5.1 菜单栏应用

```
a5164df29 feat: add mac companion app
b557a73c3 feat: richer mac settings panes and template icon
4aa275e13 feat(mac): animate menubar icon
```

一个 SwiftUI 菜单栏应用——龙虾图标（critter）常驻菜单栏：
- 显示连接状态
- 设置面板（多个 tab）
- 启动/停止 Gateway
- 查看会话

### 5.2 Voice Wake — 语音唤醒

```
e528b439b docs: document macOS Voice Wake and on-device processing
4efecfdfa feat: add mic selection to Voice Wake
b5f65e330 chore: gate Voice Wake on macOS 26
```

完全本地运行的语音唤醒系统：
- 支持自定义唤醒词
- 多语言识别
- 通过 SSH 转发到远程 Gateway
- Push-to-Talk 热键

### 5.3 Web Chat 嵌入

```
3c13a265b mac: add web chat bridge and docs
f3950a5a6 feat(macos): serve web chat over localhost to avoid cors
```

在菜单栏应用中嵌入 Web Chat，点击菜单栏图标直接打开聊天面板。

### 5.4 Canvas 画布

```
27a7d9f9d feat(mac): add agent-controlled Canvas panel
296c0a6b7 feat(macos): allow Canvas placement and resizing
cdb5ddb2d feat(macos): add Canvas A2UI renderer
```

Agent 可以控制一个浮动画布面板，展示 A2UI（一种交互式 UI 渲染技术）。

---

## 六、阶段 4：多节点平台架构

### 6.1 Gateway 协议

从 "relay"（转发器）重构为 "gateway"（网关）：

```
b2e7fb01a Gateway: finalize WS control plane
172ce6c79 Gateway: discriminated protocol schema + CLI updates
d5e4fcd17 feat(gateway)!: switch handshake to req:connect (protocol v2)
```

Gateway 成为一个完整的 WebSocket 控制面：
- 类型化的帧协议
- 健康检查
- 会话管理 RPC
- 存在感广播

### 6.2 iOS 和 Android 节点

```
b508f642b iOS: configurable voice wake words
b2378c01e feat(android): add Compose node app (bridge+canvas+chat+camera)
60321352a Android: add Voice Wake (foreground/always)
```

移动端不是简单的"客户端"，而是 **节点（Node）**：
- 通过 Bonjour/mDNS 自动发现 Gateway
- 提供摄像头、屏幕录制等能力给 Agent
- 双向 Canvas 支持
- Talk Mode（语音对话模式）

### 6.3 Bonjour 发现与配对

```
eace21dca feat(discovery): gateway bonjour + node pairing bridge
557ffdbe3 Discovery: wide-area bridge DNS-SD
```

设备自动发现、配对、连接——zero-config 体验。

### 6.4 Cron 调度器

```
f9409cbe4 Cron: add scheduler, wakeups, and run history
5118ba3dd macOS: add Cron settings tab
```

内置 cron 调度，让助手可以定时执行任务。

---

## 七、阶段 5：多渠道开放平台

### 7.1 渠道爆发

按引入顺序：
1. **WhatsApp Web**（Baileys）— 核心，最早
2. **Telegram**（grammY）— `4b5c43f08 Telegram: enable grammY throttler`
3. **macOS Chat**（SwiftUI 内置）
4. **Discord** — `ac659ff5a feat(discord): Discord transport`
5. **Signal** — `596770942 feat: add Signal provider support`
6. **Slack** — `bf3d120f8 Slack: add new slack connection`
7. **iMessage** — `cbac34347 feat: add imessage rpc adapter`

每个渠道都有独立的：群聊策略、mention 检测、消息分块、媒体处理、typing 指示器。

### 7.2 Skills 技能系统

```
d1850aaad feat: add managed skills gating
ff6a918e7 feat(skills): load bundled skills
cc0075e98 feat: add skills settings and gateway skills management
```

从内置 tools 演进到可管理的 Skills：
- Brave Search、OpenAI Whisper、Obsidian、Apple Notes/Reminders
- Bear Notes、GitHub、Notion、Trello、1Password
- Coding Agent、Food Order、Weather、Session Logs
- 可通过 uv/brew 安装
- ClawdHub 技能市场

### 7.3 Browser 自动化

```
208ba02a4 feat(browser): add clawd browser control
fa54950d2 feat(browser): add MCP tool dispatch
a526d3c1f feat(browser): add native action commands
```

完整的浏览器控制：CDP 连接、截图、DOM 检查、Playwright 集成。

### 7.4 Talk Mode 语音对话

```
20d788203 feat: add talk mode across nodes
fb8f72d5a feat(ui): add centered talk orb
27adfb76f feat(talk): pause + drag overlay orb
```

跨平台语音对话模式，集成 ElevenLabs TTS：
- 浮动球 UI
- 实时语音流
- 中断支持

---

## 八、阶段 6：企业级能力

### 8.1 多 Agent 路由

```
730cc7238 feat: multi-agent routing + multi-account providers
b04c838c1 feat!: redesign model config + auth profiles
ce6c7737c feat: add round-robin rotation and cooldown for auth profiles
```

- 不同渠道/话题路由到不同 Agent
- 每个 Agent 有独立的工作空间、模型、工具
- Auth profiles 支持多账号轮换

### 8.2 沙箱隔离

```
3b075dff8 feat: add per-session agent sandbox
d8a417f7f feat: add sandbox browser support
0914517ee feat(sandbox): add workspace access mode
```

Agent 可以在沙箱中运行 bash 命令和浏览器，限制文件系统访问。

### 8.3 安全加固

```
967cef80b fix(security): lock down inbound DMs by default
455fbc6b6 fix(security): prevent cross-channel reply routing in shared sessions
57c9a1818 fix(security): block env depth-overflow approval bypass
```

大量安全修复：
- DM 配对默认锁定
- 跨渠道回复隔离
- 沙箱路径遍历防护
- Webhook 签名验证
- 临时文件边界限制

### 8.4 社区贡献涌入

```
fe1b89467 docs: clarify personal vs private in README (#125)
d88581eb7 fix: add ~/.local/bin to PATH for uv tool binaries (#78)
cbc599a5b fix: use process PATH for bash tool (#202)
```

从 v2026.1.x 开始，大量外部 PR 被合并，GitHub Issue 编号进入五位数（#25000+），说明社区活跃度极高。

### 8.5 品牌演变

```
a27ee2366 🦞 Rebrand to CLAWDIS - add docs, update README
246adaa11 chore: rename project to clawdbot
52d933b3a refactor: replace bot.molt identifiers with ai.openclaw
```

品牌经历了三次变化：
1. **warelay** → 最初的 WhatsApp Relay
2. **Clawdis** → 加入龙虾品牌元素
3. **ClawdBot** → 简化
4. **OpenClaw** → 开源化后的最终名称

---

## 九、技术栈演进

### 运行时
- 最初：Node.js + tsx
- 中期：引入 Bun 支持（`5fcbd6aad Run CLI via tsx (no build required)`）
- 后期：Bun 优先（`b47214388 refactor: replace tsx with bun for TypeScript execution`）
- macOS 嵌入：Bun 编译的 gateway（`bb7f4abd4 feat(gateway): support bun-compiled embedded gateway`）
- 最终：Node 22+ 保持支持，Bun 用于开发

### AI 接入
- 最初：直接调用 Claude CLI
- 中期：通过 Pi（tau RPC）接入
- 后期：嵌入式 Pi agent runtime
- 当前：支持 Anthropic、OpenAI、Google Gemini、MiniMax、OpenRouter、Z.AI、LM Studio 本地模型

### 构建工具
- Biome → Oxlint/Oxfmt（lint/format）
- Vitest（测试）
- wireit（构建编排）
- rolldown（webchat bundling）
- pnpm workspaces

### 原生应用
- macOS：SwiftUI + Sparkle 更新
- iOS：SwiftUI + ClawdisKit 共享库
- Android：Jetpack Compose + Kotlin

---

## 十、关键洞察

### 10.1 开发节奏特征

1. **极高的迭代速度**：14,000+ commits，大量 "fix:" 紧跟 "feat:"，说明是"快速发布+快速修复"的节奏
2. **UI 打磨密集**：大量 `style:` / `fix(mac):` / `ux:` commits 调整间距、颜色、动画——说明开发者对 UX 细节极其在意
3. **安全修复后移**：安全加固集中在后期（阶段 6），说明早期更关注功能，后期随着用户增长转向安全

### 10.2 架构演进模式

1. **从简单到复杂的自然生长**：webhook → relay → gateway → platform
2. **去中心化倾向**：去掉 Twilio → 自托管 → 多节点 → 每个设备都是节点
3. **协议优先**：Gateway 协议 v2 是一个明确的断点，之后所有通信基于类型化 WebSocket 帧
4. **插件化**：从硬编码渠道到插件系统，再到 ClawdHub 技能市场

### 10.3 产品定位演进

| 阶段 | 定位 |
|------|------|
| 原型 | "用 Claude 自动回复 WhatsApp 消息" |
| 中期 | "个人 AI 助手，通过 WhatsApp 交互" |
| 后期 | "自托管多渠道 AI 助手平台" |
| 当前 | "运行在你自己设备上的个人 AI 助手生态系统" |

### 10.4 为什么受欢迎的技术原因

从 commit 历史可以看出：

1. **解决了真实痛点**：在你已有的聊天 App 里用 AI，而不是再开一个 Web 界面
2. **极致的原生体验**：macOS 菜单栏、Voice Wake、Canvas、Talk Mode——不是简单的 Electron 包装
3. **"it just works" 追求**：大量 auto-reconnect、auto-recovery、fallback 逻辑
4. **开发者友好**：TypeScript、插件系统、技能市场、MCP 支持
5. **隐私优先**：本地运行、支持本地模型、沙箱隔离
6. **持续的 UI 打磨**：几百条 commit 只是调整间距/颜色/动画
