# OpenClaw Tool 安全策略和权限控制

## 概述

OpenClaw 的 Tool 安全系统通过**五层防护**控制 AI 可用的工具：**Owner-Only 门控** → **Tool Profile 过滤** → **7 阶段策略管道** → **沙盒策略** → **子 Agent 深度限制**。采用"默认拒绝"原则，每一层只能收紧权限，不能扩大。

---

## 安全流程图

```
[1] senderIsOwner 从配置 + 消息上下文解析
         ↓
[2] 创建所有 tools（core + plugins）
         ↓
[3] applyOwnerOnlyToolPolicy(tools, senderIsOwner)
    - 非 owner: 过滤 + 包裹（双重门控）
    - Owner: 保留 + 包裹（执行守卫仍激活）
         ↓
[4] 解析 6 个策略来源（profile, global, agent, group, sandbox, subagent）
         ↓
[5] 执行 7 阶段策略管道
    - 每阶段: filterToolsByPolicy(tools, policy)
    - 只能收紧，不能扩大
         ↓
[6] 返回最终工具列表给模型
         ↓
[7] 模型调用工具（owner-only 包裹在执行时再次检查）
```

---

## 1. Owner-Only 工具门控

### 关键文件

`src/agents/tool-policy.ts`

### Owner-Only 工具列表

| 工具             | 原因                 |
| ---------------- | -------------------- |
| `whatsapp_login` | 登录敏感操作         |
| `cron`           | 定时任务管理         |
| `gateway`        | Gateway 函数直接调用 |

### 双重门控机制

**非 Owner**：

1. 工具从列表中**过滤掉**（模型看不到）
2. 即使绕过，执行时**包裹守卫**也会拒绝

**Owner**：

1. 工具**保留**在列表中
2. 执行时**包裹守卫**仍然激活（额外安全层）

### senderIsOwner 判定

**关键文件**：`src/auto-reply/command-auth.ts:253-379`

```
消息到达
  → 解析发送者候选（senderId, senderE164, from, 渠道特定格式）
  → 匹配 commands.ownerAllowFrom 或渠道 allowFrom 列表
  → 返回 senderIsOwner = Boolean(matched)
  → 传给 createOpenClawCodingTools({senderIsOwner})
  → 在策略管道之前首先应用
```

---

## 2. Tool Profiles（工具配置文件）

### 关键文件

`src/agents/tool-catalog.ts`

### 四种内置 Profile

| Profile       | 包含的工具                                                                         | 适用场景 |
| ------------- | ---------------------------------------------------------------------------------- | -------- |
| **minimal**   | `session_status`                                                                   | 最低权限 |
| **coding**    | 14 个工具: read, write, exec, cron, memory*\*, sessions*\*, subagents, agents_list | 编码任务 |
| **messaging** | 5 个工具: message, sessions_list, sessions_send, session_status                    | 消息发送 |
| **full**      | 空白允许列表 = 允许全部                                                            | 完全权限 |

### 工具组

Profile 支持工具组快捷方式：

| 组名             | 包含的工具              |
| ---------------- | ----------------------- |
| `group:fs`       | read, write, edit       |
| `group:runtime`  | exec, process           |
| `group:web`      | web_search, web_fetch   |
| `group:openclaw` | message, tts, cron, ... |

---

## 3. 7 阶段策略管道

### 关键文件

`src/agents/tool-policy-pipeline.ts`

### 管道阶段

**顺序执行，每阶段只能收紧**：

| 阶段 | 来源                                  | 说明                  |
| ---- | ------------------------------------- | --------------------- |
| 1    | `tools.profile`                       | 全局 Profile 策略     |
| 2    | `tools.byProvider.<provider>.profile` | Provider 特定 Profile |
| 3    | `tools.allow`                         | 全局允许列表          |
| 4    | `tools.byProvider.<provider>.allow`   | Provider 特定允许列表 |
| 5    | `agents[id].tools.allow`              | Agent 特定允许列表    |
| 6    | `agents[id].tools.byProvider.allow`   | Agent + Provider 特定 |
| 7    | 群组/渠道允许列表                     | 群组特定策略          |

加上：**沙盒策略** + **子 Agent 策略**

### 过滤逻辑

```typescript
function filterToolsByPolicy(tools, policy) {
  // 如果 policy.allow 存在：只保留允许的
  // 如果 policy.deny 存在：移除拒绝的
  // 如果 policy.alsoAllow 存在：额外添加
  // 每阶段结果传给下一阶段
}
```

### 安全保障

- **仅插件工具的白名单自动剥离**：如果 allowlist 只包含插件工具，自动失效
- **核心工具安全**：不能通过插件白名单意外禁用核心工具
- **级联过滤**：后面的阶段只能在前面的结果上收紧

---

## 4. 沙盒工具策略

### 关键文件

`src/agents/sandbox/tool-policy.ts`

### 默认允许/拒绝

| 默认允许          | 默认拒绝     |
| ----------------- | ------------ |
| exec, process     | browser      |
| read, write, edit | canvas       |
| apply_patch       | nodes        |
| image             | cron         |
| sessions\_\*      | gateway      |
| —                 | 所有渠道工具 |

### 特殊规则

- **image 自动注入**：多模态工作流中，除非明确拒绝，否则自动添加 `image`
- **Agent 特定覆盖**：每个 Agent 可有自己的沙盒策略

---

## 5. 子 Agent 深度限制

### 关键文件

`src/agents/pi-tools.policy.ts`

### 始终拒绝的工具（所有深度）

| 工具             | 原因                    |
| ---------------- | ----------------------- |
| `gateway`        | 基础设施安全            |
| `agents_list`    | 信息泄露                |
| `whatsapp_login` | 登录敏感                |
| `session_status` | 信息泄露                |
| `cron`           | 定时任务安全            |
| `memory_search`  | 记忆隔离                |
| `memory_get`     | 记忆隔离                |
| `sessions_send`  | 防止子 Agent 越权发消息 |

### 叶子子 Agent 额外拒绝

当 `depth >= maxSpawnDepth` 时，叶子节点还被拒绝：

| 工具               | 原因             |
| ------------------ | ---------------- |
| `sessions_spawn`   | 防止无限嵌套     |
| `sessions_list`    | 无需管理子 Agent |
| `sessions_history` | 历史隔离         |

### 深度追踪

- 存储在 session entry 的 `spawnDepth` 字段
- 通过 `spawnedBy` 祖先链推导
- 默认最大深度：`DEFAULT_SUBAGENT_MAX_SPAWN_DEPTH = 1`

---

## 6. Gateway HTTP 拒绝列表

通过 HTTP 访问时，以下工具被额外拒绝：

| 工具             |
| ---------------- |
| `sessions_spawn` |
| `sessions_send`  |
| `cron`           |
| `gateway`        |
| `whatsapp_login` |

---

## 7. 配置示例

### 全局配置

```yaml
tools:
  profile: "coding" # 全局 Profile
  byProvider:
    anthropic:
      profile: "coding"
  allow: ["read", "write", "group:web"] # 全局白名单
  alsoAllow: ["custom_tool"] # 追加允许
  deny: ["gateway"] # 全局黑名单
```

### Agent 特定配置

```yaml
agents:
  list:
    - id: "my-agent"
      tools:
        profile: "coding"
        allow: ["memory_search"]
```

### 沙盒配置

```yaml
tools:
  sandbox:
    tools:
      allow: ["exec", "read", "write"]
      deny: ["browser"]
```

### 子 Agent 配置

```yaml
agents:
  defaults:
    subagents:
      tools:
        allow: ["message"]
```

### 群组配置

```yaml
channels:
  groups:
    telegram:
      "123456":
        toolsBySender:
          "id:user123": { allow: ["*"] } # 特定用户全权
          "*": { allow: ["message"] } # 其他用户仅消息
```

---

## 8. 关键文件索引

| 文件                                     | 职责                       |
| ---------------------------------------- | -------------------------- |
| `src/agents/tool-policy.ts`              | Owner-Only 门控 + 策略应用 |
| `src/agents/tool-policy-pipeline.ts`     | 7 阶段策略管道             |
| `src/agents/tool-catalog.ts`             | Profile 定义 + 工具组      |
| `src/agents/sandbox/tool-policy.ts`      | 沙盒策略                   |
| `src/agents/pi-tools.policy.ts`          | 子 Agent 深度限制          |
| `src/auto-reply/command-auth.ts:253-379` | senderIsOwner 判定         |

---

## 9. 对最小可迁移版本的启示

### 必须实现

```typescript
// 1. 简单的 owner 检查
const OWNER_IDS = ["user123", "+15551234567"];

function isOwner(senderId: string): boolean {
  return OWNER_IDS.includes(senderId);
}

// 2. 按 owner 过滤工具
function getToolsForUser(senderId: string): Tool[] {
  const base = [readTool, writeTool, execTool, webSearchTool];
  if (isOwner(senderId)) {
    return [...base, cronTool, gatewayTool];
  }
  return base;
}
```

### 可以简化的地方

| OpenClaw 做法               | 最小版本             |
| --------------------------- | -------------------- |
| 7 阶段策略管道              | 简单 allowlist       |
| 4 种 Profile                | 不需要               |
| 工具组 (group:fs 等)        | 直接列举             |
| 沙盒策略 + 自动注入         | 不需要               |
| 子 Agent 深度限制           | 不需要（无子 Agent） |
| Gateway HTTP 拒绝           | 不需要               |
| 双重门控（列表 + 执行守卫） | 只过滤列表即可       |
