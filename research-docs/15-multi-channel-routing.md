# OpenClaw 多渠道消息路由机制

## 概述

OpenClaw 支持 Telegram、Discord、WhatsApp、Slack、Signal、iMessage 等多个消息渠道。所有渠道通过统一的 **ChannelPlugin** 抽象层接入，消息通过 **8 层绑定路由** 分发到对应的 Agent，Session Key 控制会话隔离级别。

---

## 完整消息路由流程

```
入站消息（任意渠道）
    │
    ▼
渠道预检（Channel Preflight）
  - Auth 检查
  - 速率限制
  - DM 策略（pairing/allowlist/block）
  - 消息验证
    │
    ▼
消息 & Peer 信息规范化
  - 提取发送者 ID
  - 确定 peer 类型（direct/group/channel）
  - 渠道特定格式 → 通用格式
    │
    ▼
resolveAgentRoute() 路由解析
  输入: channel, accountId, peer, guildId, memberRoleIds, parentPeer
    │
    ▼
8 层绑定评估（Binding Hierarchy）
  1. binding.peer           ← 渠道/线程特定
  2. binding.peer.parent    ← 线程父级
  3. binding.guild+roles    ← Discord 角色路由
  4. binding.guild          ← Discord 服务器
  5. binding.team           ← Slack 团队
  6. binding.account        ← 账号特定
  7. binding.channel        ← 通配符 ("*")
  8. default                ← 配置默认 agent
    │
    ▼
构建 Session Key
  agent:<agentId>:<mainKey>:<channel>?:<accountId>?:<peerKind>?:<peerId>?
    │
    ▼
加载 Agent Session → 处理消息 → 生成回复
    │
    ▼
通过 Channel Outbound Adapter 发送回复
```

---

## 1. ChannelPlugin 接口契约

### 关键文件

`src/channels/plugins/types.plugin.ts:49`

### 完整结构

```typescript
type ChannelPlugin<ResolvedAccount, Probe, Audit> = {
  id: ChannelId; // "telegram", "discord", "whatsapp"...
  meta: ChannelMeta; // 标签、文档路径、UI 提示
  capabilities: ChannelCapabilities; // 支持的聊天类型
  defaults?: { queue?: { debounceMs?: number } };

  // 适配器（每个可选，但核心必需）
  config: ChannelConfigAdapter; // 必需：账号解析和列举
  gateway?: ChannelGatewayAdapter; // 生命周期：启动/停止
  outbound?: ChannelOutboundAdapter; // 发送消息
  security?: ChannelSecurityAdapter; // DM 策略
  pairing?: ChannelPairingAdapter; // DM 配对
  groups?: ChannelGroupAdapter; // 群组配置
  mentions?: ChannelMentionAdapter; // 去除 @提及
  auth?: ChannelAuthAdapter; // 登录/登出
  onboarding?: ChannelOnboardingAdapter; // CLI 向导
  configSchema?: ChannelConfigSchema; // UI 设置 schema
  setup?: ChannelSetupAdapter; // 初始设置
  commands?: ChannelCommandAdapter; // 原生斜杠命令
  streaming?: ChannelStreamingAdapter; // 消息合并设置
  threading?: ChannelThreadingAdapter; // 回复线程模式
  messaging?: ChannelMessagingAdapter; // 目标规范化
  agentPrompt?: ChannelAgentPromptAdapter; // Agent 工具提示
  directory?: ChannelDirectoryAdapter; // 对等体/群组列表
  resolver?: ChannelResolverAdapter; // 目标 ID 验证
  actions?: ChannelMessageActionAdapter; // 反应/按钮
  heartbeat?: ChannelHeartbeatAdapter; // 定期检查
  agentTools?: ChannelAgentToolFactory; // 渠道专属 AI 工具
};
```

### 核心适配器

| 适配器                     | 文件                        | 职责                                                                    |
| -------------------------- | --------------------------- | ----------------------------------------------------------------------- |
| **ChannelConfigAdapter**   | `types.adapters.ts:42-70`   | `listAccountIds()`, `resolveAccount()`, `isEnabled()`, `isConfigured()` |
| **ChannelOutboundAdapter** | `types.adapters.ts:97-114`  | `sendText()`, `sendMedia()`, `sendPayload()`, `deliveryMode`            |
| **ChannelGatewayAdapter**  | `types.adapters.ts:157-166` | `startAccount()` + AbortSignal 优雅关闭                                 |
| **ChannelSecurityAdapter** | `types.adapters.ts:195-199` | `resolveDmPolicy()`, `resolveAllowFrom()`                               |

---

## 2. 8 层绑定路由

### 关键文件

`src/routing/resolve-route.ts:370-420`

### 层级评估算法

**第一个匹配的层级获胜**（从最具体到最通用）：

| 层级 | 名称                  | 条件                  | 适用场景          |
| ---- | --------------------- | --------------------- | ----------------- |
| 1    | `binding.peer`        | 有 peer 且 peer 匹配  | 特定频道/用户绑定 |
| 2    | `binding.peer.parent` | 有 parentPeer         | 线程继承父级绑定  |
| 3    | `binding.guild+roles` | 有 guildId + roles    | Discord 角色路由  |
| 4    | `binding.guild`       | 有 guildId            | Discord 服务器级  |
| 5    | `binding.team`        | 有 teamId             | Slack 团队级      |
| 6    | `binding.account`     | accountPattern ≠ "\*" | 账号特定          |
| 7    | `binding.channel`     | accountPattern = "\*" | 渠道通配符        |
| 8    | `default`             | 始终                  | 配置默认 agent    |

### 绑定配置示例

```yaml
# 层级 1：Peer 绑定（最高优先级）
agents:
  list:
    - id: "chan-agent"
      bindings:
        - channel: "discord"
          peer: { kind: "channel", id: "c1" }

    # 层级 3：Discord Guild + Roles
    - id: "admin-bot"
      bindings:
        - channel: "discord"
          guildId: "g1"
          roles: ["mod-role-id"]

    # 层级 6：账号特定
    - id: "biz-agent"
      bindings:
        - channel: "whatsapp"
          accountId: "business"

    # 层级 7：渠道通配符
    - id: "fallback"
      bindings:
        - channel: "telegram"
          accountId: "*"
```

---

## 3. Session Key 层级结构

### 关键文件

`src/routing/session-key.ts:114-161`

### 基本格式

```
agent:<agentId>:<rest-of-key>
```

### dmScope 控制隔离级别

| dmScope                      | 格式                                     | 说明                       |
| ---------------------------- | ---------------------------------------- | -------------------------- |
| `"main"`                     | `agent:main:main`                        | 所有 DM 合并到一个 session |
| `"per-peer"`                 | `agent:main:direct:user123`              | 按发送者隔离（跨渠道）     |
| `"per-channel-peer"`         | `agent:main:telegram:direct:12345`       | 按渠道+发送者隔离          |
| `"per-account-channel-peer"` | `agent:main:telegram:tasks:direct:12345` | 最大隔离                   |

### 非 DM 场景（群组/频道）

群组和频道始终按 peer 隔离，不受 dmScope 影响：

```
agent:main:discord:channel:ch123     ← Discord 频道
agent:main:telegram:group:grp456     ← Telegram 群组
```

### 身份链接（Identity Links）

跨渠道用户统一：

```yaml
identityLinks:
  alice: ["telegram:111111111", "discord:222222222"]
  bob: ["whatsapp:+1234567890", "signal:+1987654321"]
```

当 `dmScope=per-peer` 时：

- Telegram 用户 111111111 的消息 → session key: `agent:main:direct:alice`
- Discord 用户 222222222 的消息 → 同一个 session key: `agent:main:direct:alice`

---

## 4. 渠道注册与发现

### 插件注册

```typescript
// extensions/discord/index.ts
export default {
  id: "discord",
  name: "Discord",
  register(api: OpenClawPluginApi) {
    api.registerChannel({ plugin: discordPlugin });
  },
};
```

### 渠道发现

`src/channels/registry.ts:147-172`

通过 `normalizeAnyChannelId()` 查找注册表：

- 精确匹配 plugin ID
- 匹配 `meta.aliases`

### 渠道优先顺序

```typescript
// src/channels/registry.ts:7-16
export const CHAT_CHANNEL_ORDER = [
  "telegram",
  "whatsapp",
  "discord",
  "irc",
  "googlechat",
  "slack",
  "signal",
  "imessage",
];
```

---

## 5. 消息规范化

### 各渠道规范化

| 渠道         | 文件                    | DM ID 格式              | 群组 ID 格式          |
| ------------ | ----------------------- | ----------------------- | --------------------- |
| **Telegram** | `normalize/telegram.ts` | 整数 `"123456789"`      | 负数 `"-123456789"`   |
| **Discord**  | `normalize/discord.ts`  | Snowflake `"987654321"` | 频道 ID `"123456789"` |
| **WhatsApp** | `normalize/whatsapp.ts` | E164 `"+15551234567"`   | JID `"...@g.us"`      |
| **Slack**    | `normalize/slack.ts`    | `"U123ABCD"`            | `"C123ABCD"`          |
| **Signal**   | `normalize/signal.ts`   | E164 `"+15551234567"`   | UUID                  |

### 共享工具

`src/channels/plugins/normalize/shared.ts`

- `trimMessagingTarget()` — 清理输入
- `looksLikeHandleOrPhoneTarget()` — 识别 @handle、邮箱、电话号码

---

## 6. 出站消息投递

### 投递模式

| 模式        | 说明              | 使用渠道          |
| ----------- | ----------------- | ----------------- |
| `"direct"`  | 适配器直接发送    | Telegram、Discord |
| `"gateway"` | 通过 Gateway 中转 | WhatsApp、Signal  |
| `"hybrid"`  | 两者皆可          | iMessage          |

### 投递流程

```typescript
// src/infra/outbound/deliver.ts:120-127
const adapter = loadChannelOutboundAdapter(channel);
const handler = createChannelHandler(adapter, context);

// 根据消息类型调用
await adapter.sendText(ctx); // 纯文本
await adapter.sendMedia(ctx); // 带媒体
await adapter.sendPayload(ctx); // 结构化（按钮、表单）
```

### 文本分块限制

| 渠道     | 最大长度  | 说明         |
| -------- | --------- | ------------ |
| Discord  | 2000 字符 | 超过自动分块 |
| Telegram | 4096 字符 | —            |
| WhatsApp | 4096 字符 | —            |
| Slack    | 3000 字符 | —            |

---

## 7. Gateway 渠道生命周期管理

### 关键文件

`src/gateway/server-channels.ts:80-200+`

### 管理器

```
createChannelManager()
  ├── startChannels()     — 启动所有已配置渠道
  ├── startChannel(id, accountId)  — 启动特定账号
  │   └── plugin.gateway.startAccount(ctx)
  │       └── AbortSignal 用于优雅关闭
  ├── stopChannel(id, accountId)   — 停止特定账号
  └── restartChannel(id, accountId) — 重启
```

### 渠道状态

每个渠道账号追踪：

- 运行状态（running/stopped/error）
- 重启尝试次数
- AbortSignal
- 运行时元数据

---

## 8. 安全策略

### DM 策略

| 策略          | 行为             |
| ------------- | ---------------- |
| `"open"`      | 接受所有 DM      |
| `"pairing"`   | 需要配对验证     |
| `"allowlist"` | 仅允许白名单用户 |
| `"disabled"`  | 禁止所有 DM      |

### 配对存储

`src/pairing/pairing-store.ts` — 管理跨渠道的 DM 配对白名单。

---

## 9. 关键文件索引

| 文件                                      | 职责               |
| ----------------------------------------- | ------------------ |
| `src/routing/resolve-route.ts:291-443`    | 主路由算法         |
| `src/routing/bindings.ts`                 | 绑定列表管理       |
| `src/routing/session-key.ts:114-161`      | Session Key 构建   |
| `src/channels/plugins/types.plugin.ts:49` | ChannelPlugin 契约 |
| `src/channels/plugins/types.adapters.ts`  | 适配器定义         |
| `src/channels/registry.ts`                | 渠道发现与注册     |
| `src/gateway/server-channels.ts:80-200`   | 渠道生命周期管理   |
| `src/infra/outbound/deliver.ts:120-127`   | 投递处理器创建     |
| `src/channels/plugins/normalize/`         | 各渠道消息规范化   |

---

## 10. 对最小可迁移版本的启示

### 必须实现

```typescript
// 1. 简单的渠道抽象
interface Channel {
  id: string;
  onMessage: (handler: (msg: Message) => void) => void;
  send: (target: string, text: string) => Promise<void>;
}

// 2. 路由到 Agent（简化版）
function routeMessage(channel: string, senderId: string): string {
  return `agent:main:${channel}:direct:${senderId}`;
}

// 3. 注册渠道
const channels: Channel[] = [telegramChannel, discordChannel];
channels.forEach((ch) =>
  ch.onMessage((msg) => {
    const sessionKey = routeMessage(ch.id, msg.senderId);
    processWithAgent(sessionKey, msg);
  }),
);
```

### 可以简化的地方

| OpenClaw 做法             | 最小版本              |
| ------------------------- | --------------------- |
| 8 层绑定路由              | 按渠道+发送者直接映射 |
| 4 种 dmScope 隔离级别     | 固定 per-channel-peer |
| Identity Links 跨渠道统一 | 不需要                |
| 22 个适配器接口           | 只需 onMessage + send |
| DM 策略 + 配对 + 白名单   | 简单 allowlist        |
| Gateway 生命周期管理      | 直接启动              |
| 消息规范化层              | 简单 trim             |
