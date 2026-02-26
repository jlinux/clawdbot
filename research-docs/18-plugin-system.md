# OpenClaw Plugin 系统

## 概述

OpenClaw 的插件系统基于 **jiti 动态加载**，支持 TypeScript/JavaScript 插件的运行时发现和加载。插件通过 4 个来源发现，可以提供 **10+ 种能力类型**（工具、钩子、渠道、服务、HTTP 路由等），并通过 **24+ 个生命周期钩子**参与系统的各个环节。

---

## 架构总览

```
┌─ 发现阶段 ────────────────────────────────────────┐
│ 扫描 4 个来源（优先级: config > workspace > global > bundled）
│ ├─ Config:    plugins.load.paths
│ ├─ Workspace: ~/.openclaw/extensions/
│ ├─ Global:    ~/.openclaw/extensions/
│ └─ Bundled:   ./extensions/（随二进制分发）
│                                                    │
│ 结果: PluginCandidate[] + 文件路径和元数据         │
└────────────────────────────────────────────────────┘
                     ↓
┌─ Manifest 加载 ───────────────────────────────────┐
│ 加载 openclaw.plugin.json:                         │
│ ├─ id（必需）                                      │
│ ├─ configSchema（必需）                            │
│ ├─ kind（可选: "memory"）                          │
│ └─ channels, providers, skills（可选）             │
└────────────────────────────────────────────────────┘
                     ↓
┌─ 启用/禁用决策 ───────────────────────────────────┐
│ ├─ 在 allow 列表中？                               │
│ ├─ 在 deny 列表中？                               │
│ ├─ 配置中明确 enabled/disabled？                    │
│ ├─ Memory slot 检查（仅 1 个 memory 插件激活）     │
│ └─ 结果: enabled: boolean                          │
└────────────────────────────────────────────────────┘
                     ↓
┌─ jiti 加载 ───────────────────────────────────────┐
│ 对每个启用的插件:                                   │
│ ├─ 通过 jiti 加载源文件（TypeScript/JavaScript）    │
│ ├─ 验证配置（JSON Schema）                         │
│ ├─ 提取模块导出（函数或对象）                       │
│ ├─ 同步调用 register(api)                          │
│ │   └─ 插件使用 api.registerTool/Hook/Channel/etc │
│ └─ 记录插件元数据到 registry                       │
└────────────────────────────────────────────────────┘
                     ↓
┌─ 运行时使用 ──────────────────────────────────────┐
│ Tools:    resolvePluginTools(context) → AnyAgentTool[]
│ Hooks:    getGlobalHookRunner() → 在事件处触发钩子
│ Channels: 注册到渠道 dock
│ Services: startPluginServices() → start/stop 生命周期
│ HTTP:     注册路由到 gateway
│ CLI:      注册命令到 program
│ Commands: 在 agent 调用前处理
└────────────────────────────────────────────────────┘
```

---

## 1. jiti 动态加载机制

### 关键文件

`src/plugins/loader.ts:418-439`

```typescript
let jitiLoader: ReturnType<typeof createJiti> | null = null;

const getJiti = () => {
  if (jitiLoader) return jitiLoader;

  jitiLoader = createJiti(import.meta.url, {
    interopDefault: true,
    extensions: [".ts", ".tsx", ".mts", ".cts", ".js", ".mjs", ".cjs", ".json"],
    alias: {
      "openclaw/plugin-sdk": resolvePluginSdkAlias(),
      "openclaw/plugin-sdk/account-id": resolvePluginSdkAccountIdAlias(),
    },
  });
  return jitiLoader;
};
```

**特性**：

- **惰性初始化**：仅在插件实际启用时创建 loader（避免测试开销）
- **模块别名**：`openclaw/plugin-sdk` 映射到实际 SDK 位置
- **多格式支持**：`.ts`, `.js`, `.mts`, `.cts`, `.mjs`, `.cjs`
- **ESM/CJS 兼容**：`interopDefault: true` 统一处理

### 模块导出解析

`src/plugins/loader.ts:118-139`

```typescript
function resolvePluginModuleExport(moduleExport: unknown) {
  // 如果是函数 → 视为 register
  if (typeof resolved === "function") {
    return { register: resolved };
  }
  // 如果是对象 → 提取 definition + register/activate
  if (resolved && typeof resolved === "object") {
    const def = resolved as OpenClawPluginDefinition;
    return { definition: def, register: def.register ?? def.activate };
  }
  return {};
}
```

---

## 2. 四个发现来源

### 关键文件

`src/plugins/discovery.ts`

| 优先级    | 来源          | 路径                        | 说明                   |
| --------- | ------------- | --------------------------- | ---------------------- |
| 1（最高） | **Config**    | `plugins.load.paths`        | 明确配置的插件路径     |
| 2         | **Workspace** | `~/.openclaw/extensions/`   | 工作区内的项目本地插件 |
| 3         | **Global**    | `~/.openclaw/extensions/`   | 用户级全局插件         |
| 4（最低） | **Bundled**   | `./extensions/`（随安装包） | 内置插件               |

### PluginCandidate 结构

```typescript
export type PluginCandidate = {
  idHint: string; // 包名或文件名
  source: string; // 实际 .ts/.js 文件路径
  rootDir: string; // 插件根目录
  origin: PluginOrigin; // "bundled"|"global"|"workspace"|"config"
  packageName?: string; // package.json name
  packageVersion?: string;
};
```

---

## 3. 插件能力类型（10+ 种）

| 能力                | 注册方法                                     | 说明                        |
| ------------------- | -------------------------------------------- | --------------------------- |
| **Tools**           | `api.registerTool(tool, opts?)`              | AI 工具（function calling） |
| **Hooks**           | `api.on(hookName, handler, opts?)`           | 生命周期钩子                |
| **Channels**        | `api.registerChannel({plugin})`              | 消息渠道适配器              |
| **Services**        | `api.registerService(service)`               | 后台服务（start/stop）      |
| **HTTP Routes**     | `api.registerHttpRoute({path, handler})`     | 自定义 HTTP 端点            |
| **HTTP Handlers**   | `api.registerHttpHandler(handler)`           | HTTP 请求拦截               |
| **Gateway Methods** | `api.registerGatewayMethod(method, handler)` | WebSocket/REST 方法         |
| **CLI Commands**    | `api.registerCli(registrar, opts?)`          | CLI 子命令                  |
| **Providers**       | `api.registerProvider(provider)`             | LLM/AI 模型 provider        |
| **Commands**        | `api.registerCommand(definition)`            | 直接命令（绕过 agent）      |

---

## 4. Plugin SDK 接口

### 关键文件

`src/plugin-sdk/index.ts`（200+ 导出）

### OpenClawPluginApi

```typescript
export type OpenClawPluginApi = {
  id: string;
  name: string;
  version?: string;
  config: OpenClawConfig;
  pluginConfig?: Record<string, unknown>;
  runtime: PluginRuntime; // 350+ 共享函数
  logger: PluginLogger;

  // 注册方法
  registerTool: (tool, opts?) => void;
  registerHook: (events, handler, opts?) => void;
  registerChannel: (registration) => void;
  registerService: (service) => void;
  registerHttpRoute: (params) => void;
  registerHttpHandler: (handler) => void;
  registerGatewayMethod: (method, handler) => void;
  registerCli: (registrar, opts?) => void;
  registerProvider: (provider) => void;
  registerCommand: (command) => void;

  // 类型化钩子
  on: <K extends PluginHookName>(hookName, handler, opts?) => void;

  // 路径解析
  resolvePath: (input: string) => string;
};
```

### PluginRuntime

350+ 函数，覆盖 30+ 命名空间：

- `config`: loadConfig, writeConfigFile
- `system`: runCommandWithTimeout
- `media`: detectMime, resizeToJpeg
- `tools`: createMemoryGetTool, createMemorySearchTool
- `channel`: text processing, reply dispatch, routing, pairing, media, session
- 各渠道特定: discord, slack, telegram, signal, imessage, whatsapp, line
- `logging`: getChildLogger, shouldLogVerbose
- `state`: resolveStateDir

---

## 5. 生命周期钩子（24+）

### 关键文件

`src/plugins/types.ts:299-755`

### Agent 生命周期

| 钩子                   | 触发时机             | 可修改              |
| ---------------------- | -------------------- | ------------------- |
| `before_model_resolve` | 模型/provider 确定前 | 覆盖 model/provider |
| `before_prompt_build`  | System Prompt 构建前 | 修改 prompt         |
| `before_agent_start`   | Agent 启动前（遗留） | —                   |
| `llm_input`            | LLM 调用前           | 日志                |
| `llm_output`           | LLM 返回后           | 日志 + token 统计   |
| `agent_end`            | Agent 完成后         | 日志                |

### Compaction 钩子

| 钩子                | 触发时机              |
| ------------------- | --------------------- |
| `before_compaction` | 上下文压缩前          |
| `after_compaction`  | 上下文压缩后          |
| `before_reset`      | /new 或 /reset 命令时 |

### 消息钩子

| 钩子               | 触发时机   | 可修改       |
| ------------------ | ---------- | ------------ |
| `message_received` | 收到消息   | —            |
| `message_sending`  | 发送消息前 | 修改消息内容 |
| `message_sent`     | 消息已发送 | —            |

### Tool 钩子

| 钩子                  | 触发时机     | 可修改         |
| --------------------- | ------------ | -------------- |
| `before_tool_call`    | Tool 执行前  | 阻止或修改参数 |
| `after_tool_call`     | Tool 执行后  | 日志           |
| `tool_result_persist` | 结果持久化前 | 过滤/修改      |

### 子 Agent 钩子

| 钩子                       | 触发时机        |
| -------------------------- | --------------- |
| `subagent_spawning`        | 子 Agent 创建中 |
| `subagent_delivery_target` | 投递目标解析    |
| `subagent_spawned`         | 子 Agent 已创建 |
| `subagent_ended`           | 子 Agent 已结束 |

### Gateway 钩子

| 钩子            | 触发时机     |
| --------------- | ------------ |
| `gateway_start` | Gateway 启动 |
| `gateway_stop`  | Gateway 关闭 |

### 其他

| 钩子                   | 触发时机          |
| ---------------------- | ----------------- |
| `before_message_write` | 消息写入 JSONL 前 |
| `session_start`        | Session 创建      |
| `session_end`          | Session 结束      |

### 钩子优先级

钩子按优先级排序执行（数值越大越先执行），支持异步。

---

## 6. 插件注册和初始化流程

### 完整加载管道

`src/plugins/loader.ts:359-700`

```
loadOpenClawPlugins(options)
  ├─ 加载配置
  ├─ 检查缓存
  ├─ 创建 plugin runtime
  ├─ 创建 registry + API 工厂
  │
  ├─ 发现阶段
  │  ├─ discoverOpenClawPlugins()
  │  ├─ loadPluginManifestRegistry()
  │  └─ buildProvenanceIndex()
  │
  ├─ 插件加载循环
  │  对每个候选:
  │  ├─ 检查覆盖
  │  ├─ 解析启用状态
  │  ├─ 验证配置
  │  ├─ jiti 加载模块
  │  ├─ 解析导出
  │  ├─ 创建 API
  │  ├─ 同步调用 register()
  │  └─ 记录到 registry
  │
  └─ 完成
     ├─ 检查 memory slot
     ├─ 缓存 registry
     └─ 初始化 hook runner
```

### 同步注册

```typescript
// loader.ts:654-663
const result = register(api);
if (result && typeof result.then === "function") {
  // 异步 register() 被检测但忽略（发出警告）
  registry.diagnostics.push({
    level: "warn",
    message: "async registration is ignored",
  });
}
```

**注意**：`register()` 必须是同步的。

---

## 7. 插件定义与入口

### 插件定义类型

```typescript
export type OpenClawPluginDefinition = {
  id?: string;
  name?: string;
  description?: string;
  version?: string;
  kind?: PluginKind;
  configSchema?: OpenClawPluginConfigSchema;
  register?: (api: OpenClawPluginApi) => void;
  activate?: (api: OpenClawPluginApi) => void; // register 的别名
};
```

### 标准入口模式

```typescript
// extensions/discord/index.ts
export default {
  id: "discord",
  name: "Discord",
  description: "Discord channel plugin",
  configSchema: emptyPluginConfigSchema(),
  register(api: OpenClawPluginApi) {
    setDiscordRuntime(api.runtime);
    api.registerChannel({ plugin: discordPlugin });
    registerDiscordSubagentHooks(api);
  },
};
```

### Manifest 文件

`openclaw.plugin.json`：

```json
{
  "id": "discord",
  "configSchema": {},
  "kind": "channel",
  "channels": ["discord"]
}
```

### package.json 元数据

```json
{
  "name": "@openclaw/discord",
  "version": "2026.2.1",
  "openclaw": {
    "extensions": ["index.ts"],
    "channel": {
      "id": "discord",
      "label": "Discord"
    }
  }
}
```

---

## 8. 工具注册与解析

### 注册

```typescript
// registry.ts:172-197
api.registerTool(tool, { name: "my_tool", optional: true });
// 或
api.registerTool((ctx) => createTool(ctx), { names: ["tool1", "tool2"] });
```

支持：

- 直接工具对象
- 工厂函数（运行时创建）
- 可选工具（受 allowlist 控制）

### 解析

`src/plugins/tools.ts:45-100`

```typescript
export function resolvePluginTools(params: {
  context: OpenClawPluginToolContext;
  existingToolNames?: Set<string>;
  toolAllowlist?: string[];
}): AnyAgentTool[] {
  const registry = loadOpenClawPlugins({...});

  for (const entry of registry.tools) {
    if (blockedPlugins.has(entry.pluginId)) continue;
    if (!isOptionalToolAllowed({...})) continue;

    const resolved = entry.factory(params.context);
    // 验证 + 冲突检查
    tools.push(...resolved);
  }
  return tools;
}
```

---

## 9. 安全与验证

### 路径安全

`src/plugins/discovery.ts:87-175`

- `checkSourceEscapesRoot()` — 确保插件不逃逸根目录
- `checkPathStatAndPermissions()` — 验证路径状态和所有权
- 阻止 world-writable 路径
- 阻止可疑所有权（非用户、非 root）
- Bundled 插件跳过所有权检查

### 配置验证

`src/plugins/loader.ts:97-116`

使用 JSON Schema 验证插件配置，结果缓存。

### 允许/拒绝列表

```typescript
// config-state.ts:17-21
const BUNDLED_ENABLED_BY_DEFAULT = new Set([
  "device-pair",
  "phone-control",
  "talk-voice",
]);

// 配置:
{
  "plugins": {
    "allow": ["discord", "telegram"],
    "deny": ["dangerous-plugin"]
  }
}
```

---

## 10. 扩展插件目录

```
extensions/
├── discord/          # Discord 渠道
├── memory-lancedb/   # LanceDB 记忆后端
├── feishu/           # 飞书渠道
├── googlechat/       # Google Chat
├── bluebubbles/      # BlueBubbles/iMessage
├── device-pair/      # 设备配对
├── zalo/             # Zalo 渠道
├── diagnostics-otel/ # OpenTelemetry 诊断
└── ...
```

---

## 11. 关键文件索引

| 文件                          | 职责                         |
| ----------------------------- | ---------------------------- |
| `src/plugins/loader.ts`       | 主加载管道（发现→加载→注册） |
| `src/plugins/discovery.ts`    | 4 来源发现 + 路径安全        |
| `src/plugins/registry.ts`     | 注册表 + API 工厂            |
| `src/plugins/types.ts`        | 类型定义（钩子、能力、API）  |
| `src/plugins/hooks.ts`        | 钩子运行器                   |
| `src/plugins/tools.ts`        | 插件工具解析                 |
| `src/plugins/services.ts`     | 后台服务管理                 |
| `src/plugins/config-state.ts` | 配置规范化                   |
| `src/plugins/manifest.ts`     | Manifest 加载                |
| `src/plugin-sdk/index.ts`     | SDK 导出（200+）             |

---

## 12. 对最小可迁移版本的启示

### 必须实现

```typescript
// 1. 插件接口
interface Plugin {
  id: string;
  register: (api: PluginApi) => void;
}

// 2. 简单注册表
interface PluginApi {
  registerTool: (tool: Tool) => void;
}

// 3. 加载
const plugins = [discordPlugin, telegramPlugin];
const tools: Tool[] = [];
plugins.forEach((p) =>
  p.register({
    registerTool: (t) => tools.push(t),
  }),
);
```

### 可以简化的地方

| OpenClaw 做法             | 最小版本           |
| ------------------------- | ------------------ |
| jiti 动态 TypeScript 加载 | 直接 import        |
| 4 来源发现 + Manifest     | 手动注册           |
| 10+ 能力类型              | 只需 Tools         |
| 24+ 生命周期钩子          | 不需要             |
| 路径安全 + 所有权检查     | 不需要（信任插件） |
| JSON Schema 配置验证      | 不需要             |
| 350+ runtime 函数         | 按需提供           |
| allow/deny 列表           | 不需要             |
