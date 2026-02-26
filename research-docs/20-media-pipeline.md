# OpenClaw 媒体处理管道（Media Pipeline）

## 概述

OpenClaw 的媒体管道处理用户通过各渠道发送的图片、音频、视频和文档，将其转换为 AI 可处理的形式（文本描述或 base64 图片），并支持 AI 输出音频（TTS）和图片的反向投递。

---

## 整体流程图

```
用户消息（Telegram/Discord/WhatsApp...）
    ↓
[媒体提取]
    ├─ 音频 → STT（OpenAI/Deepgram/Google）→ 文本
    ├─ 图片 → Vision Tool（Claude/OpenAI）→ 描述
    ├─ 视频 → Video Understanding Provider → 描述
    └─ 文档 → PDF 文本提取 + 图片提取
    ↓
[纯文本注入对话上下文]
    ↓
[Agent 处理并生成响应]
    ↓
[Agent 输出解析]
    ├─ 文本响应 → 直接发送
    ├─ MEDIA: /path → 图片/文档附件
    └─ [[tts]] 指令 → TTS 语音转换
    ↓
[渠道投递]
    ├─ Telegram: 语音消息 / 音频文件
    ├─ Discord: 文件附件
    ├─ Web: 媒体服务器 /media/:id
    └─ 其他: 直接附件或 URL
```

---

## 1. 图片处理

### 1.1 图片接收

**来源**：

- 用户通过消息渠道上传（Telegram、Discord、WhatsApp 等）
- 消息中的 URL
- 本地文件路径

**接收流程**（以 Telegram 为例）：

1. `src/telegram/bot-message-context.ts` 从 Telegram API 获取媒体元数据
2. 提取 `file_id`，通过 Telegram Bot API 下载文件
3. 转换为 `MediaAttachment` 结构：`{ path, contentType, stickerMetadata }`
4. 存储到本地 `~/.openclaw/media/inbound/`

### 1.2 图片处理给 Vision AI

**关键文件**：`src/agents/tools/image-tool.ts`

**image 工具工作流**：

```
输入解析 → 路径解析 → 媒体加载 → 图片优化 → Vision 模型选择 → 模型推理 → 文本输出
```

1. **输入解析**：接受单张 `.image` 或多张 `.images`（最多 20 张）
2. **路径解析**：支持多种格式
   - 本地路径（含 `~/` 展开）
   - `file://` URL
   - `data:` URL（base64 编码）
   - HTTP/HTTPS URL
   - 工作区相对路径
3. **媒体加载**：`loadWebMedia()` 下载或读取本地文件，检测 MIME 类型
4. **图片优化**（`src/media/image-ops.ts`）：
   - HEIC → JPEG 转换（macOS 用 sips，其他用 sharp）
   - EXIF 方向校正（读取 tag 0x0112）
   - 自适应尺寸：网格搜索 sides `[2048, 1536, 1280, 1024, 800px]` × qualities `[80, 70, 60, 50, 40%]`
   - PNG 透明度保留：有 alpha 通道的保持 PNG 格式
   - GIF 不压缩不缩放（直接发送或拒绝）
5. **Vision 模型选择**：
   - 优先同 provider（如主模型是 OpenAI 则用 OpenAI Vision）
   - 回退到 Anthropic Claude 或 OpenAI
   - 支持 MiniMax VL（仅单张）、ZAI GLM
6. **模型推理**：base64 发送到 Vision 模型
7. **输出**：模型的图片分析作为文本返回

### 1.3 图片大小限制

```typescript
// src/media/image-ops.ts
MAX_IMAGE_BYTES = 6 * 1024 * 1024; // 6MB 默认
// 可通过 cfg.agents.defaults.mediaMaxMb 自定义
```

**后端选择**：优先 sharp（Node.js）；Bun/macOS 回退到 sips。

---

## 2. 音频处理

### 2.1 语音转文字（STT）

**总入口**：`src/media-understanding/runner.ts`

**支持的 STT Provider**：

| Provider           | 模型                               | 端点                    | 来源文件                                              |
| ------------------ | ---------------------------------- | ----------------------- | ----------------------------------------------------- |
| **OpenAI Whisper** | `gpt-4o-mini-transcribe`           | `/audio/transcriptions` | `src/media-understanding/providers/openai/audio.ts`   |
| **Deepgram**       | `nova-3`                           | `GET /listen`           | `src/media-understanding/providers/deepgram/audio.ts` |
| **Google Cloud**   | —                                  | Speech-to-Text API      | `src/media-understanding/providers/google/audio.ts`   |
| **Groq Whisper**   | —                                  | OpenAI 兼容端点         | `src/media-understanding/providers/groq/`             |
| **其他**           | Mistral, Moonshot, ZAI, Minimax 等 | —                       | 各自 provider 目录                                    |

**转录流程**：

1. 渠道接收音频附件
2. 根据配置选择模型/provider
3. Buffer/文件传给转录 provider
4. 结果：`{ text: "...", model: "..." }`
5. 转录文本作为纯文本注入消息上下文

**支持的音频格式**（`src/media/audio.ts`）：

- MIME: `audio/ogg`, `audio/opus`, `audio/mpeg`, `audio/mp3`, `audio/mp4`, `audio/x-m4a`
- 扩展名: `.oga`, `.ogg`, `.opus`, `.mp3`, `.m4a`

### 2.2 文字转语音（TTS）

**工具**：`src/agents/tools/tts-tool.ts`

**TTS Provider**：

| Provider                   | 默认语音                        | 默认格式         | Telegram 格式     | 来源文件              |
| -------------------------- | ------------------------------- | ---------------- | ----------------- | --------------------- |
| **Edge TTS**（默认，免费） | `en-US-MichelleNeural`          | MP3 24kHz/48kbps | Opus 48kHz/64kbps | `src/tts/tts.ts`      |
| **OpenAI TTS**             | `alloy`（`gpt-4o-mini-tts`）    | MP3              | Opus              | `src/tts/tts-core.ts` |
| **ElevenLabs**             | voice ID `pMsXgVXv3BLzUgSXRplE` | MP3              | Opus              | `src/tts/tts-core.ts` |

**TTS 自动模式**（`src/tts/tts.ts`）：

| 模式      | 行为                                                            |
| --------- | --------------------------------------------------------------- |
| `off`     | 禁用                                                            |
| `always`  | 始终生成音频                                                    |
| `inbound` | 仅当用户发送了音频/语音时                                       |
| `tagged`  | 仅当响应包含 `[[tts]]` 或 `[[tts:text]]...[[/tts:text]]` 指令时 |

**TTS 处理流程**：

1. 长度检查：默认最大 4096 字符
2. 超长文本：启用 summarization 时用 Claude 摘要
3. Markdown 剥离：移除 `###` 等标记
4. 格式选择：Telegram 用语音兼容格式（Opus）
5. 音频标记：`[[audio_as_voice]]` 标记 Telegram 语音消息
6. 临时文件清理

---

## 3. 文档处理

### 3.1 支持的格式

**关键文件**：`src/media/input-files.ts`

| 类型 | MIME                                                   |
| ---- | ------------------------------------------------------ |
| 文本 | `text/plain`, `text/markdown`, `text/html`, `text/csv` |
| 数据 | `application/json`                                     |
| 文档 | `application/pdf`                                      |

### 3.2 PDF 处理

1. **文本提取**：PDFjs 提取最多 4 页文本
2. **图片提取**：若文本 < 200 字符（阈值），提取页面为 PNG 图片
3. **图片预算**：单个 PDF 最多 4,000,000 像素
4. **文本截断**：最多 200,000 字符
5. **输出**：`{ filename, text?, images?: [...] }`

---

## 4. 媒体存储与托管

### 4.1 磁盘存储

**位置**：`~/.openclaw/media/`

**文件命名**：`{sanitized-original}---{uuid}.{ext}`

- 提取用户上传的原始文件名
- 清理不安全字符
- 无原始名时仅用 UUID

**文件权限**：`0o600`（仅用户可读）

**大小限制**：

| 类型 | 限制  |
| ---- | ----- |
| 图片 | 6MB   |
| 音频 | 16MB  |
| 视频 | 16MB  |
| 文档 | 100MB |

### 4.2 HTTP 媒体服务器

**关键文件**：`src/media/server.ts`、`src/media/host.ts`

- **端口**：42873（可配置）
- **端点**：`GET /media/:id`
- **安全**：
  - ID 验证：仅字母数字、点、连字符、下划线，最长 200 字符
  - 仅在媒体根目录内读取
  - TTL：2 分钟后删除/返回 404
- **MIME 检测**：自动检测 Content-Type
- **单次清理**：响应完成 ~50ms 后删除文件

**Tailscale 托管**：`https://{tailscale-hostname}/media/{id}`

### 4.3 自动清理

- TTL：2 分钟（默认，可配置）
- 周期性自动清理
- 投递后单次清理（Telegram 等）

---

## 5. 远程媒体获取

**关键文件**：`src/media/fetch.ts`

1. **SSRF 防护**：`fetchWithSsrFGuard()` + hostname 白名单验证
2. **重定向**：最多跟随 5 次，验证最终 URL
3. **大小限制**：下载过程中强制限制（默认 5MB）
4. **嗅探**：捕获前 16KB 用于 MIME 检测
5. **Content-Disposition**：提取原始文件名
6. **错误处理**：提供详细错误信息（HTTP 状态码 + body 片段）

---

## 6. Agent 输出中的媒体解析

**关键文件**：`src/media/parse.ts`

### MEDIA Token 格式

```
MEDIA: /path/to/file.jpg
MEDIA: "path with spaces.png"
MEDIA: https://example.com/image.jpg
```

### 提取规则

1. 扫描 `MEDIA:` 行（大小写不敏感）
2. 支持包裹路径：反引号、引号、方括号
3. 跳过代码块内的 token
4. 单行支持多个媒体
5. 规范化 `file://` 前缀
6. 验证路径（本地、HTTP/HTTPS、纯文件名）
7. 从可见输出中剥离 MEDIA 行

### 音频语音标记

`[[audio_as_voice]]` — 标记音频为 Telegram 语音消息投递（非文件附件）。

---

## 7. MIME 类型检测

**关键文件**：`src/media/mime.ts`

**检测优先级**：

```
HTTP Content-Type 头 > Magic Bytes 文件签名 > 文件扩展名
```

支持图片、音频、视频、文档、压缩包等。

---

## 8. 关键文件索引

| 文件                                                  | 职责                                        |
| ----------------------------------------------------- | ------------------------------------------- |
| `src/media/parse.ts`                                  | 从 agent 输出提取 MEDIA token               |
| `src/media/store.ts`                                  | 媒体磁盘存储管理                            |
| `src/media/fetch.ts`                                  | URL 远程下载 + SSRF 防护                    |
| `src/media/input-files.ts`                            | 输入文件处理（PDF、文档）                   |
| `src/media/image-ops.ts`                              | 图片缩放、优化、格式转换                    |
| `src/media/audio.ts`                                  | Telegram 兼容音频格式检测                   |
| `src/media/host.ts`                                   | HTTP 媒体托管                               |
| `src/media/server.ts`                                 | Express `/media/:id` 路由                   |
| `src/media/mime.ts`                                   | MIME 类型检测                               |
| `src/web/media.ts`                                    | 从 URL/本地加载媒体 + 图片优化              |
| `src/media-understanding/runner.ts`                   | STT / 图片描述 / 视频描述编排               |
| `src/media-understanding/providers/openai/audio.ts`   | OpenAI Whisper 转录                         |
| `src/media-understanding/providers/deepgram/audio.ts` | Deepgram STT                                |
| `src/tts/tts.ts`                                      | TTS provider 编排（Edge/OpenAI/ElevenLabs） |
| `src/tts/tts-core.ts`                                 | 各 TTS provider 核心实现                    |
| `src/agents/tools/image-tool.ts`                      | Vision 模型图片分析工具                     |
| `src/agents/tools/tts-tool.ts`                        | TTS agent 工具                              |

---

## 9. 约束与限制汇总

| 资源         | 限制      | 备注                  |
| ------------ | --------- | --------------------- |
| 图片大小     | 6MB       | 可通过 config 调整    |
| 音频大小     | 16MB      | STT provider 各有限制 |
| 视频大小     | 16MB      | —                     |
| 文档大小     | 100MB     | PDF 上限 5MB          |
| PDF 页数     | 4         | 可提取页数            |
| PDF 像素预算 | 400 万    | 图片提取预算          |
| TTS 输入文本 | 4096 字符 | 超过可摘要            |
| TTS 音频默认 | 1500 字符 | 触发自动摘要          |
| image 工具   | 20 张     | 单次调用最大图片数    |
| 远程下载超时 | 10 秒     | —                     |
| 媒体 TTL     | 2 分钟    | 创建/投递后自动删除   |

---

## 10. 安全机制

- **SSRF 防护**：远程 URL 经 hostname 白名单验证
- **路径穿越防护**：本地文件检查允许的根目录；拒绝符号链接
- **文件权限**：媒体文件创建为 0o600（仅用户可读）
- **文件名清理**：移除不安全字符
- **沙盒**：image 工具支持沙盒化 FS bridge（agent 工作区隔离）
- **TTL 清理**：自动删除防止磁盘耗尽

---

## 11. 对最小可迁移版本的启示

如果要复用媒体管道模式，核心需求是：

### 必须实现

```typescript
// 1. 媒体加载器
async function loadMedia(source: string): Promise<{ buffer: Buffer; mime: string }> {
  // 支持: 本地路径、HTTP URL、base64 data URL
}

// 2. STT 转录（选一个 provider）
async function transcribe(audio: Buffer): Promise<string> {
  // OpenAI Whisper API 最简单
}

// 3. Vision 描述（选一个 provider）
async function describeImage(image: Buffer): Promise<string> {
  // Claude/OpenAI Vision API
}
```

### 可以简化的地方

| OpenClaw 做法               | 最小版本               |
| --------------------------- | ---------------------- |
| 多 STT provider + 自动选择  | 只用 OpenAI Whisper    |
| 多 TTS provider + 自动模式  | 只用 Edge TTS（免费）  |
| 自适应图片优化（网格搜索）  | 固定缩放到 1024px      |
| SSRF 防护 + hostname 白名单 | 仅处理本地文件         |
| HTTP 媒体服务器 + TTL       | 不需要（直接发文件）   |
| MEDIA token 解析            | 让 AI 返回文件路径即可 |
