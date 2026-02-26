# OpenClaw 子 Agent 机制

## 概述

OpenClaw 支持通过 `sessions_spawn` 工具创建子 Agent，实现并行任务执行和分层编排。子 Agent 在独立 session 中运行（`promptMode="minimal"`），通过**推送式完成通知**自动将结果报告给父 Agent，无需轮询。

---

## 架构总览

```
┌─────────────────────────────────────────────────────────┐
│ 主 Agent / 父 Session                                   │
│                                                          │
│  调用: sessions_spawn(task="...", label="...")           │
│                            │                             │
│                            ▼                             │
│  spawnSubagentDirect()                                   │
│  ├─ 验证深度: callerDepth < maxSpawnDepth               │
│  ├─ 验证最大子 Agent 数                                  │
│  ├─ 创建 session: agent:X:subagent:UUID                 │
│  ├─ 设置 spawnDepth = callerDepth + 1                   │
│  ├─ 构建 system prompt (minimal mode)                    │
│  ├─ 通过 gateway.agent() 启动                           │
│  └─ registerSubagentRun() 注册到注册表                   │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│ 子 Agent Session (Depth=1)                               │
│                                                          │
│ System Prompt (Minimal): 仅 Tooling + Safety + Skills   │
│ Extra System Prompt: 角色说明 + 规则 + 输出格式          │
│                                                          │
│  ├─ canSpawn=true (编排者): 可以再创建子 Agent           │
│  └─ canSpawn=false (叶子): 只能执行任务                  │
│                                                          │
│ 执行任务 → 生成最终回复                                  │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│ 完成通知（Push-Based）                                   │
│                                                          │
│ completeSubagentRun()                                    │
│   ↓                                                      │
│ runSubagentAnnounceFlow()                                │
│   ├─ 读取子 Agent 最终输出                               │
│   ├─ 格式化通知（✅/❌/⏱️）                              │
│   └─ 路由到父 Agent（AGENT_LANE_SUBAGENT）              │
│       ├─ 编排者: 作为系统消息（内部上下文）              │
│       └─ 主 Agent: 作为用户可见消息                      │
└─────────────────────────────────────────────────────────┘
```

---

## 1. sessions_spawn 工具

### 关键文件

`src/agents/tools/sessions-spawn-tool.ts`

### 参数

```typescript
{
  task: string;                    // 必需: 任务描述
  label?: string;                  // 显示标签
  agentId?: string;                // 目标 agent（默认当前 agent）
  model?: string;                  // 模型覆盖
  thinking?: string;               // 思考级别
  runTimeoutSeconds?: number;      // 执行超时
  thread?: boolean;                // 绑定到线程
  mode?: "run" | "session";        // 执行模式
  cleanup?: "delete" | "keep";     // 清理策略
}
```

### 生成流程

`spawnSubagentDirect()`（`src/agents/subagent-spawn.ts:162-535`）：

1. 验证深度限制（`callerDepth < maxSpawnDepth`）
2. 验证最大活跃子 Agent 数（`activeChildren < maxChildrenPerAgent`）
3. 检查跨 Agent 生成的白名单
4. 创建唯一子 session key: `agent:${targetAgentId}:subagent:${uuid}`
5. 设置 `spawnDepth = callerDepth + 1`
6. 应用 model/thinking 覆盖
7. 构建 minimal system prompt
8. 通过 `gateway.agent()` 在 `AGENT_LANE_SUBAGENT` 启动
9. `registerSubagentRun()` 注册到注册表

---

## 2. Run vs Session 模式

| 特性         | Run 模式（默认）           | Session 模式              |
| ------------ | -------------------------- | ------------------------- |
| **持续时间** | 单次执行后终止             | 任务完成后保持活跃        |
| **Cleanup**  | 默认 "keep"（可 "delete"） | 始终 "keep"               |
| **线程绑定** | 不需要                     | **必需**（`thread=true`） |
| **适用场景** | 并行分析、临时子任务       | 迭代工作流、用户会话      |
| **后续消息** | 不支持                     | 支持（同线程继续）        |

### Session 模式约束

仅在支持线程绑定的渠道有效（Discord、Slack、Signal 等）。

---

## 3. 深度限制

### 默认配置

```typescript
// src/config/agent-limits.ts:6
export const DEFAULT_SUBAGENT_MAX_SPAWN_DEPTH = 1;
```

### 深度层级

| 深度 | 角色         | 说明                   |
| ---- | ------------ | ---------------------- |
| 0    | 主 Agent     | agent:main:main        |
| 1    | 一级子 Agent | 默认为叶子节点         |
| 2+   | 嵌套子 Agent | 需要 maxSpawnDepth > 1 |

### 执行验证

```typescript
// subagent-spawn.ts:227-235
const callerDepth = getSubagentDepthFromSessionStore(requesterKey, { cfg });
const maxSpawnDepth =
  cfg.agents?.defaults?.subagents?.maxSpawnDepth ?? DEFAULT_SUBAGENT_MAX_SPAWN_DEPTH;
if (callerDepth >= maxSpawnDepth) {
  return {
    status: "forbidden",
    error: `sessions_spawn not allowed at depth ${callerDepth} (max: ${maxSpawnDepth})`,
  };
}
```

### 深度追踪

- 存储在 session entry: `spawnDepth` 字段
- 缺失时通过 `spawnedBy` 祖先链推导
- 也可通过 session key 结构解析

---

## 4. 推送式完成通知

### 流程

1. **运行完成检测**（`subagent-registry.ts:516-555`）：
   - Gateway 发出生命周期事件（phase="end"/"error"/"timeout"）
   - `completeSubagentRun()` 以 outcome（ok/error/timeout）调用

2. **通知发起**（`subagent-registry.ts:310-338`）：

   ```typescript
   function startSubagentAnnounceCleanupFlow(runId, entry) {
     void runSubagentAnnounceFlow({
       childSessionKey: entry.childSessionKey,
       requesterSessionKey: entry.requesterSessionKey,
       waitForCompletion: false,
       cleanup: entry.cleanup,
     });
   }
   ```

3. **通知投递**（`subagent-announce.ts:1094-1300+`）：
   - 等待子 run 稳定（如果仍在嵌入运行）
   - 从 session 历史读取子 Agent 最终输出
   - 格式化完成消息（✅ 成功 / ❌ 错误 / ⏱️ 超时）
   - 路由到请求者：
     - 编排者 → 系统消息（内部上下文）
     - 主 Agent/用户 → 用户可见消息

4. **重试机制**：
   - 最大重试：3 次（`MAX_ANNOUNCE_RETRY_COUNT`）
   - 退避：1s, 2s, 4s, 8s（指数，最大 8s）
   - 强制过期：完成后 5 分钟（`ANNOUNCE_EXPIRY_MS`）

### 通知超时

- 默认：60 秒（`DEFAULT_SUBAGENT_ANNOUNCE_TIMEOUT_MS`）
- 可配置：`agents.defaults.subagents.announceTimeoutMs`

---

## 5. 编排者 vs 叶子模式

### 判定逻辑

```typescript
const canSpawn = childDepth < maxSpawnDepth;
```

### 编排者（Orchestrator）

`canSpawn = true`（depth < maxSpawnDepth）

- **可以**通过 `sessions_spawn` 创建自己的子 Agent
- 通过 `subagents(action=list)` 查看自己的子 Agent
- 接收后代的完成结果
- 结果投递方式：系统指令（内部上下文）
- System Prompt 包含：
  ```
  你可以使用 sessions_spawn 创建子 Agent 进行并行或复杂工作。
  子 Agent 的结果会自动通知你（不是主 Agent）。
  协调工作并在报告前综合结果。
  ```

### 叶子（Leaf）

`canSpawn = false`（depth >= maxSpawnDepth）

- **不能**创建子 Agent
- 通过父级 requester key 查看兄弟节点
- System Prompt 包含：
  ```
  你是叶子工作节点，不能创建子 Agent。专注于分配给你的任务。
  ```

### 可见性

```typescript
// subagents-tool.ts:181-228
function resolveRequesterKey(params) {
  if (!isSubagentSessionKey(callerSessionKey)) {
    // 主 Agent: 看到所有它创建的子 Agent
    return { requesterSessionKey: callerSessionKey };
  }

  if (callerDepth < maxSpawnDepth) {
    // 编排者: 看到自己的子 Agent
    return { requesterSessionKey: callerSessionKey };
  }

  // 叶子: 向上走到父级，看到兄弟节点
  return { requesterSessionKey: spawnedBy };
}
```

---

## 6. subagents 工具：子 Agent 生命周期管理

### 关键文件

`src/agents/tools/subagents-tool.ts`

### 三个动作

#### list — 列出子 Agent

```typescript
subagents((action = "list"), (recentMinutes = 30));
```

- 列出活跃和近期的子 Agent runs
- 显示：索引、标签、模型、运行时间、token 用量、状态
- 编排者看自己的子 Agent
- 叶子看兄弟节点

#### kill — 终止子 Agent

```typescript
subagents((action = "kill"), (target = "1" | "label" | "runId" | "all"));
```

- `target="all"` → 级联 kill 所有子+后代
- 更新 session entry：`abortedLastRun = true`
- 标记 run terminated，原因 "killed"
- 抑制通知（避免过时完成消息）

**级联 Kill 逻辑**（`subagents-tool.ts:276-318`）：

```typescript
async function cascadeKillChildren(params) {
  const childRuns = listSubagentRunsForRequester(parentKey);
  for (const run of childRuns) {
    if (!run.endedAt) {
      await killSubagentRun({ cfg, entry: run });
    }
    // 递归 kill 孙子
    await cascadeKillChildren({ parentChildSessionKey: childKey });
  }
}
```

#### steer — 重定向子 Agent

```typescript
subagents((action = "steer"), (target = "1" | "label" | "runId"), (message = "新指令..."));
```

1. 标记原 run 为抑制通知（steer-restart）
2. 中止当前工作并清空队列
3. 等待稳定（5 秒超时）
4. 在同一 session 启动新 run（包含 steer 消息）
5. 替换旧 run 记录

**限制**：

- 不能 steer 自己
- 每对父子 2 秒速率限制
- 消息最长 4000 字符

---

## 7. promptMode "minimal" 设计

### 关键文件

`src/agents/system-prompt.ts:14-17`

```typescript
export type PromptMode = "full" | "minimal" | "none";
```

### Minimal 模式包含的部分

| 部分                   | 包含 |
| ---------------------- | ---- |
| ✅ Tooling（工具列表） | 是   |
| ✅ Tool Call Style     | 是   |
| ✅ Safety              | 是   |
| ✅ Skills（如果提供）  | 是   |
| ❌ Memory Recall       | 否   |
| ❌ User Identity       | 否   |
| ❌ Time Zone           | 否   |
| ❌ Reply Tags          | 否   |
| ❌ Messaging section   | 否   |
| ❌ Voice (TTS)         | 否   |
| ❌ Documentation       | 否   |
| ❌ Group Chat Context  | 否   |

### 模式选择

```typescript
// pi-embedded-runner/run/attempt.ts:226-230
export function resolvePromptModeForSession(sessionKey?: string): "minimal" | "full" {
  return isSubagentSessionKey(sessionKey) ? "minimal" : "full";
}
```

### 额外 System Prompt

`subagent-announce.ts:976-1066` 注入额外上下文：

```
你是由 [主 Agent|父编排者] 创建的子 Agent，用于执行特定任务。

你的角色:
- 完成这个任务，这是你的全部目的。
- 你的最终消息会自动报告给请求者。

规则:
- 不要发起主动操作
- 完成后你可能被终止
- 信任推送式完成（后代结果自动通知你）
```

---

## 8. 结果收集与使用

### 结果类型

```typescript
export type SubagentRunOutcome = {
  status: "ok" | "error" | "timeout" | "unknown";
  error?: string;
};
```

### 结果聚合

编排者收到的完成消息：

```
[子 Agent 1] ✅ 完成任务并返回结果
[子 Agent 2] ⏱️ 超时
[子 Agent 3] ✅ 完成任务并返回结果
```

### 通知指令逻辑

`subagent-announce.ts:1075-1092`：

```typescript
if (remainingActiveSubagentRuns > 0) {
  // 等待剩余 runs
  ("还有 N 个活跃子 Agent。如果是同一工作流，等待剩余结果。");
}
if (requesterIsSubagent) {
  // 父编排者期望简洁更新
  ("将完成结果转为简洁的内部编排更新。");
}
// 主 Agent → 用户
("将结果转为正常助手语气，发送给用户。");
```

### 注册表持久化

`subagent-registry.ts:88-90, 853-860`

- 持久化到磁盘：`subagentRuns` map
- Gateway 重启后恢复
- 过期归档（可配置 `archiveAfterMinutes`）
- 清理扫描（60 秒间隔）

---

## 9. 关键配置默认值

```typescript
DEFAULT_AGENT_MAX_CONCURRENT = 4;
DEFAULT_SUBAGENT_MAX_CONCURRENT = 8;
DEFAULT_SUBAGENT_MAX_SPAWN_DEPTH = 1; // 默认叶子节点
maxChildrenPerAgent = 5;
runTimeoutSeconds = 0; // 无超时
announceTimeoutMs = 60_000;
archiveAfterMinutes = 60;
thinking = "off"; // 默认思考级别
```

---

## 10. 关键文件索引

| 文件                                      | 职责                              |
| ----------------------------------------- | --------------------------------- |
| `src/agents/tools/sessions-spawn-tool.ts` | sessions_spawn 工具定义           |
| `src/agents/subagent-spawn.ts`            | 子 Agent 创建逻辑                 |
| `src/agents/tools/subagents-tool.ts`      | subagents 工具（list/kill/steer） |
| `src/agents/subagent-announce.ts`         | 完成通知流程                      |
| `src/agents/subagent-registry.ts`         | 运行注册表和持久化                |
| `src/agents/subagent-depth.ts`            | 深度解析                          |
| `src/config/agent-limits.ts`              | 默认限制常量                      |
| `src/agents/system-prompt.ts`             | PromptMode 定义                   |

---

## 11. 对最小可迁移版本的启示

### 必须实现

```typescript
// 1. 简单的子 Agent 创建
async function spawnSubAgent(task: string): Promise<string> {
  const result = await callModel({
    systemPrompt: "你是一个子任务执行者。完成任务后直接输出结果。",
    userPrompt: task,
    tools: [readTool, writeTool, execTool],
  });
  return result;
}

// 2. 并行执行
const results = await Promise.all([
  spawnSubAgent("搜索 AI 新闻"),
  spawnSubAgent("搜索量子计算新闻"),
]);
// 合并结果返回给用户
```

### 可以简化的地方

| OpenClaw 做法                     | 最小版本           |
| --------------------------------- | ------------------ |
| 推送式完成通知 + 重试             | `Promise.all()`    |
| Run/Session 两种模式              | 只需 Run 模式      |
| 深度限制 + 编排者/叶子            | 固定 1 层          |
| subagents 工具（list/kill/steer） | 不需要             |
| promptMode "minimal"              | 简化 system prompt |
| 级联 kill + steer 重定向          | 不需要             |
| 注册表持久化 + 归档               | 不需要（内存中）   |
| 线程绑定 + Session 模式           | 不需要             |
