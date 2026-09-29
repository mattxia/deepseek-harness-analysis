# DeepSeek Harness 与 DeepSeek 模型对接流程分析（dsh-v0.2.0-rc.2）

> 分析对象：DeepSeek Harness 最新 release `dsh-v0.2.0-rc.2` 中与 DeepSeek 模型的对接流程、特殊处理点、类与关键代码。

## 一、完整调用流程

```
Agent loop                              LLM 契约层                       DeepSeek Provider 层
──────────                              ──────────                       ────────────────────
agent.prepareRequest()
  ├─ session header + reasoning/maxTokens
  └─ llm.prepareCall(config)
        │
        ▼
  LlmRuntime.prepareCall()
    ├─ registration(provider) → 找到已注册 adapter
    ├─ adapter.prepareCall()  ────────► DeepSeekAdapter.prepareCall()
    │                                  ├─ modelInfo() 解析模型能力
    │                                  └─ stream = generate(options, connection)
    ├─ normalizeModelInfo() 校验 reasoning efforts
    └─ resolveCallWithInfo() 物化默认 maxTokens / effort
        │
        ▼
  preparedCall.stream(request)
    └─ streamWithRegistration → waterfall 'llm/stream'
        │
        ▼
  DeepSeekAdapter.generate()
    ├─ AbortController + idleWatchdog(MESSAGES_IDLE)
    └─ request() 循环:
        ├─ prepareImages()        图片预处理（base64 / Files API）
        ├─ resolveAuth()          API Key / 凭证
        ├─ RequestFiles           上传图片到 Files API，失败回退 inline
        ├─ serialize()            内部消息 → DeepSeek wire format
        ├─ prepareRequestExtensions()  DeepSeek API 扩展字段
        ├─ POST {baseURL}/messages  (SSE)
        │    headers: anthropic-version, anthropic-beta, x-deepseek-harness-*
        ├─ parseSse()             解析 SSE 事件流
        └─ translate()            SSE 事件 → StreamChunk
              │
              ▼
  StreamChunk(text-delta / reasoning-delta / tool-call-delta /
              block-end / usage / finish)
        │
        ▼
  agent loop 组装 assistant blocks，若 tool-calls 则执行工具并进入下一轮
```

## 二、对 DeepSeek 模型做特殊处理的地方

### 1. 推理 / Thinking（DeepSeek 特有）

**类/函数**：`modelInfo()`、`serialize()` 中的 effort 处理、`translate.ts` 的 thinking 块解析

**文件**：
- `packages/llm/llm-deepseek/src/model-info.ts` — 声明 4 档 reasoning effort：`off / low / high / max`
- `packages/llm/llm-deepseek/src/serialize.ts` — 请求体映射

**关键代码**（`serialize.ts` L146-L155）：

```ts
const effort = options.purpose === 'session-title' ? 'off'
  : options.reasoningEffort
    ?? (connection.defaults.reasoningEffort
      ?? (connection.defaults.thinking === 'disabled' ? 'off' : 'high'))
// ...
thinking: { type: effort === 'off' ? 'disabled' : 'enabled' },
...effort === 'off' ? {} : { output_config: { effort: effort as 'low' | 'high' | 'max' } },
```

**响应侧**（`translate.ts` L51-L53）：DeepSeek 的 `thinking` 块映射为内部 `reasoning` 块，并保留 `signature` 用于原生 replay：

```ts
case 'thinking':
  content = { type: 'reasoning', text: string(native.thinking) }
  replay = { type: 'reasoning', ...native.signature === undefined ? {} : { signature: string(native.signature) } }
```

`session-title` 等不需要推理的用途强制 `off`；`thinking: 'disabled'` 时仅允许 `off`。

### 2. 原生 thinking replay（signature）

**文件**：`packages/llm/llm-deepseek/src/replay.ts`、serialize.ts、translate.ts

DeepSeek 的 thinking 块携带 `signature`，允许后续请求把历史 thinking 作为 replay 直接传回，避免重新推理。`serialize.ts` L31-L34：

```ts
case 'reasoning': return {
  type: 'thinking', thinking: block.text,
  ...replay?.[index]?.signature === undefined ? {} : { signature: replay[index].signature },
}
```

`translate.ts` L78-L81 累加 `signature_delta`，最终写入 replay state 供下次请求复用。

### 3. 工具变更机制（tool_addition / tool_removal，DeepSeek beta）

**文件**：`packages/llm/llm-deepseek/src/serialize.ts` L90-L100、adapter.ts 的 beta header

DeepSeek 支持会话内动态增删工具，不需要重发整个 tools 列表。内部 `developer` 消息的 `tool-addition` / `tool-removal` 块映射为 DeepSeek 的 `tool_addition` / `tool_removal`（引用工具名），并触发 `anthropic-beta: tool-changes-2025-04-22` 头。

```ts
case 'tool-addition': return [{ type: 'tool_addition', tool: { type: 'tool_reference', name: block.toolName } }]
case 'tool-removal':  return [{ type: 'tool_removal',  tool: { type: 'tool_reference', name: block.toolName } }]
```

系统提示更新模式：`systemPromptUpdate === 'in-history'` 时，system 消息被当作 `role: 'system'` 的 in-history 更新，紧跟在前一个 user turn 之后（而非顶层 `system` 字段）。

### 4. 文件 / 图片上传（Files API + inline 回退）

**文件**：
- `packages/llm/llm-deepseek/src/files-api.ts`
- `packages/llm/llm-deepseek/src/file-store.ts`
- `packages/llm/llm-deepseek/src/request-files.ts`
- `packages/llm/llm-deepseek/src/upload-index.ts`

`RequestFiles` 先尝试把图片通过 DeepSeek **Files API** 上传得到 `file_id`（带过期时间、配额清理），请求体用 `source: { type: 'file', file_id }`；若上传失败（`FileResolutionFailure`），回退到 inline base64，并触发 `onReplayDegrade`。`serialize.ts` L70-L77：

```ts
fileId === undefined
  ? { type: 'image', source: { type: 'base64', media_type, data: base64 } }
  : { type: 'image', source: { type: 'file', file_id } }
```

上传时带 `anthropic-beta: files-2024-11-01`（`MESSAGES_FILES_BETA`）。

### 5. 图片 token 计费与 offload

**文件**：`packages/llm/llm-deepseek/src/image-tokens.ts`、`packages/llm/llm-deepseek/src/request-pricing.ts`

DeepSeek 按图片像素计费。`deepSeekImageTokens()` 计算每张图的 token 数，`imagePricing()` 提供每模型图片定价。请求超出 `maxRequestFilesBytes` / `maxImagesPerRequest` 时按 `imageOffloadByteQuantum` / `imageOffloadCountQuantum` 移除最旧图片（offload），并在错误中返回 `offloadImages` 数让调用方重试。

### 6. 特有请求头与归属

**文件**：`packages/llm/llm-deepseek/src/adapter.ts` L120-L131

```ts
headers: {
  ...attributionHeaders(),                                  // 归属头（契约要求）
  'content-type': 'application/json',
  'accept': 'text/event-stream',
  ...auth.headers,
  'anthropic-version': '2023-06-01',                        // DeepSeek Messages API 版本
  ...betas.length === 0 ? {} : { 'anthropic-beta': betas.join(',') },  // files / tool-changes
  'x-deepseek-harness-user-id': this.dependencies.resolveUserId(),
  ...options.sessionId === undefined ? {} : { 'x-deepseek-harness-session-id': String(options.sessionId) },
  ...options.purpose === 'compaction' ? { 'x-deepseek-harness-compact': '1' : {} },
}
```

`x-deepseek-harness-*` 是 DeepSeek 服务端用于识别 harness 客户端的专有头（用户 ID、会话 ID、是否 compaction 用途）。

### 7. 错误分类（DeepSeek 错误码 → 通用 LlmError code）

**文件**：`packages/llm/llm-deepseek/src/transport.ts`

`providerError()` 把 DeepSeek 的 HTTP status + `error.type` 映射到通用错误码：

| DeepSeek 信号 | 通用 code |
|---|---|
| 401/403 / `authentication_error` / `permission_error` | `AUTH` |
| quota exceeded / 402 | `QUOTA` |
| 429 / `rate_limit_error` | `RATE_LIMIT` |
| context window exceeded | `CONTEXT_WINDOW_EXCEEDED` |
| 400/413 / `invalid_request_error` | `INVALID_REQUEST` |
| ≥500 / `api_error` / `overloaded_error` | `SERVER` |

同时解析 `retry-after` 头为 `providerRetryAfterMs`，提取 `x-deepseek-request-id` 作为 `requestId`。Files API 的配额错误会触发上传重试循环（`files.retry(detail)`）。

### 8. 连接配置（DeepSeek 特有参数）

**文件**：`packages/llm/llm-deepseek/src/config.ts`、`defaults.ts`、`models.ts`

```ts
baseURL, thinking('enabled'|'disabled'),
reasoningEffort('off'|'low'|'high'|'max'),
maxTokens(默认 256000), defaultContextWindow(默认 1,000,000),
models(默认 V41 Flash / V4 Pro 目录),
streamIdleTimeoutMs, maxRequestFilesBytes, maxInlineRequestImageBytes,
maxImagesPerRequest, imageOffload*Quantum, fileExpiresAfterSeconds, ...
```

这些都是 DeepSeek 专属的可调参数，不存在于通用 `LlmAdapter` 契约中。

### 9. API Key 与账户发现

**文件**：
- `packages/llm/llm-deepseek-api-key/src/index.ts` — 从环境变量 / credentials seam 发现 `DEEPSEEK_API_KEY`、`DEEPSEEK_BASE_URL`
- `packages/llm/llm-deepseek-account/src/index.ts` — 账户配额（quota）查询与路由

### 10. DeepSeek API 扩展

**文件**：`packages/llm/deepseek-llm-api-extensions/src/index.ts`

`prepareRequestExtensions()` 在发送前向请求体注入 DeepSeek 专有的 API 扩展字段（如 sessionId、purpose 等），扩展失败时 `onExtensionsOmitted` 记录但不中断请求。

### 11. SSE 解析（DeepSeek Messages 事件协议）

**文件**：`packages/llm/llm-deepseek/src/sse.ts`、`packages/llm/llm-deepseek/src/translate.ts`

DeepSeek 采用 Anthropic Messages 风格的 SSE 事件序列：`message_start → content_block_start → content_block_delta → content_block_stop → message_delta → message_stop`。`parseSse` 用 `EventSourceParserStream` 分帧并校验 `event.type` 与 `frame.event` 一致；`translate` 维护 block 索引、累加 tool-call JSON、校验 tool 调用配对，最终产出 `usage`（含 `cacheReadTokens` / `cacheWriteTokens`）和 `finish`（`stop` / `tool-calls` / `max-tokens`）。

## 三、契约与实现的分离

- **通用契约**（`packages/llm/llm/`）：`LlmRuntime`（adapter 注册表 + waterfall）、抽象 `LlmAdapter`（`prepareCall` / `stream` / `resolveModel` / `listModels`）、`StreamChunk`、`LlmError`。**不包含任何 DeepSeek 专有字段**。
- **DeepSeek 实现**（`packages/llm/llm-deepseek/`）：把内部消息/工具/图片翻译成 DeepSeek Messages wire format，处理 thinking/replay、tool changes、files API、图片计费、错误码映射等 DeepSeek 特有行为。
- **对比**：`llm-pi-ai` 是另一个 adapter，走第三方提供商（pi-ai 0.87.1），通过 SDK 而非直连 Messages API，证明契约是 provider 无关的。

## 四、关键调用链索引

| 步骤 | 位置 |
|---|---|
| Agent 发起调用 | `packages/core/agent-loop/src/agent.ts` L547-L596 `prepareRequest` |
| LLM 准备调用 | `packages/llm/llm/src/index.ts` L936-L983 `LlmRuntime.prepareCall` |
| 流式调度 | `packages/llm/llm/src/index.ts` L1033-L1137 `adapterStream` / `stream` |
| DeepSeek 入口 | `packages/llm/llm-deepseek/src/adapter.ts` L20-L73 `DeepSeekAdapter.generate` |
| 请求组装+发送 | `packages/llm/llm-deepseek/src/adapter.ts` L75-L159 `request` |
| 序列化 | `packages/llm/llm-deepseek/src/serialize.ts` L56-L168 |
| SSE 解析 | `packages/llm/llm-deepseek/src/sse.ts` L13-L28 |
| 事件翻译 | `packages/llm/llm-deepseek/src/translate.ts` L104-L166 |
| 错误分类 | `packages/llm/llm-deepseek/src/transport.ts` L22-L44 |
| 模型能力 | `packages/llm/llm-deepseek/src/model-info.ts` L61-L100 |
