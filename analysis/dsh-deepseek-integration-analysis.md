# dsh 与 DeepSeek 模型对接分析

分析日期：2026-09-26。范围：`packages/llm`（dsh-llm、dsh-llm-deepseek、dsh-llm-retry）与 `packages/core/agent-loop` 的完整调用链路。

## 总体架构：三层解耦

dsh 与 DeepSeek 的对接分为三层：agent 循环（消费者）、LLM 服务编排层、DeepSeek 传输适配器，层间以中立的 `StreamChunk` 流协议解耦。

- **Service Definition**：[packages/llm/llm/src/index.ts](../packages/llm/llm/src/index.ts) 的 `LlmRuntime`——适配器注册表 + 可拦截的 `llm/stream` waterfall。
- **Service Provider**：[packages/llm/llm-deepseek/src/index.ts](../packages/llm/llm-deepseek/src/index.ts) 注册 `deepseek-official` 适配器。
- **Consumer**：[packages/core/agent-loop/src/agent.ts](../packages/core/agent-loop/src/agent.ts) 的 agent 循环，只认中立类型（[packages/llm/llm/src/types.ts](../packages/llm/llm/src/types.ts) 的 `StreamChunk`），完全不感知 DeepSeek 线格式。

## 调用时序图

```mermaid
sequenceDiagram
    autonumber
    participant Loop as Agent 循环<br>(dsh-agent-loop)
    participant RT as LlmRuntime<br>(dsh-llm)
    participant AD as DeepSeekAdapter<br>(dsh-llm-deepseek)
    participant API as DeepSeek API<br>/chat/completions

    Note over Loop,RT: ① 请求构建
    Loop->>Loop: buildRequest()<br>agent/request waterfall 提出 seedConfig
    Loop->>RT: prepareCall(request)
    Note right of RT: 校验模型存在 / 物化 defaultMaxTokens<br>与默认 reasoning effort / deepFreeze
    RT-->>Loop: PreparedCall
    Loop->>Loop: markAgentLoopRequest()<br>记录 request/header、request/context 事件

    Note over Loop,AD: ② 发起流式调用
    Loop->>RT: stream(冻结请求)
    Note right of RT: llm/stream waterfall<br>中间件可拦截
    RT->>AD: adapter.stream()
    AD->>AD: 解析连接快照<br>resolveApiKey(): credentials ?? $DEEPSEEK_API_KEY

    Note over AD,API: ③ 序列化与 HTTP 请求
    AD->>AD: serializeRequest()<br>stream:true + include_usage<br>thinking + reasoning_effort<br>tool-result 拆为 role:'tool'
    AD->>API: POST /chat/completions<br>Bearer / SSE / x-deepseek-harness-* 头
    alt 非 2xx
        API-->>AD: WireError
        AD->>AD: httpErrorCode(): AUTH / RATE_LIMIT /<br>CONTEXT_WINDOW_EXCEEDED / SERVER
        AD-->>Loop: error finish chunk（LlmError）
        Loop->>Loop: agent/request-error waterfall<br>llm-retry 指数退避后重试
    else 2xx
        API-->>AD: SSE delta 流
    end

    Note over AD,Loop: ④ SSE 解析与流翻译
    AD->>AD: parseSse(): [DONE] 哨兵<br>EOF 无 [DONE] 则 STREAM_CLOSED
    AD->>AD: translate() → StreamChunk<br>reasoning/text/tool-call delta 按 index 聚合
    AD->>AD: mapUsage(): inputTokens = prompt_tokens - cacheRead<br>block-end/usage/finish 延迟到 [DONE]
    AD-->>RT: StreamChunk 流
    RT-->>Loop: 逐 chunk 透传
    Loop->>Loop: session.append('assistant/chunk')<br>BlockAssembler 组装

    Note over Loop,API: ⑤ finish 与工具循环
    alt finish: tool-calls
        Loop->>Loop: executeToolCalls()<br>tool-result 回灌下一轮请求
    else finish: stop
        Loop->>Loop: 发 assistant/message 事件（含 usage）<br>循环结束
    else finish: error / aborted
        Loop->>Loop: agent/request-error → 重试或抛 LlmError
    end
```

## 分阶段关键处理

### 1. 插件注册与连接凭据（dsh-llm-deepseek）

- baseURL 三级回退：`config.baseURL ?? $DEEPSEEK_BASE_URL ?? https://api.deepseek.com`（仅信任层可读环境变量）。
- API Key 每请求解析：`resolveApiKey()` 优先走 credentials 服务，否则读 launch 环境变量 `DEEPSEEK_API_KEY`；`assertUsableApiKey` 校验，缺失直接报 `MISSING_CREDENTIAL`（fail loud）。
- 配置快照语义：`options()` thunk 缓存 lastGood，坏配置保留上一代并告警；每次 `stream()` 解析 connection/apiKey/userId 快照，in-flight 流不受配置变更影响。
- 原子换路由：retryPolicy 变化时 `registration.replace([PROVIDER])`，全有或全无。
- 代码：[packages/llm/llm-deepseek/src/index.ts](../packages/llm/llm-deepseek/src/index.ts) 的 `apply()`、`resolveAdapterOptions()`。

### 2. 请求构建与校验（agent-loop → LlmRuntime）

- `buildRequest()`：`agent/request` waterfall 允许中间件改写 seedConfig → `llm.prepareCall()` 校验模型存在、物化 `defaultMaxTokens` 与默认 reasoning effort、deepFreeze 整个请求 → 记录 `request/header` 与 `request/context` 事件 → `markAgentLoopRequest()`。
- 模型可见 ⟺ 可从会话日志重建：请求以 `markAgentLoopRequest` 标记入日志。
- 代码：[packages/core/agent-loop/src/agent.ts](../packages/core/agent-loop/src/agent.ts) 的 `step()`/`buildRequest()`；[packages/llm/llm/src/index.ts](../packages/llm/llm/src/index.ts) 的 `prepareCall()`（config 变更后旧 prepared call → `INVALID_PREPARED_CALL`）。

### 3. 请求序列化（DeepSeek 线格式）

- 恒发 `stream: true` + `stream_options: {include_usage: true}`。
- thinking mode 解析：`thinking: {type: enabled/disabled}` 顶层字段 + `reasoning_effort: high|max`；`session-title` 用途强制 disabled；disabled 部署配非 off effort → 加载期报错。
- `reasoning_content` 仅在带 tool_calls 的 assistant 回传轮发送（省 token）。
- assistant 无文本轮发 `content: ""`，永不发 `null`。
- user 消息里的 tool-result 拆成独立 `role: 'tool'` 消息（带 `tool_call_id`），空输出替换为 `'(no output)'`。
- 图片内容直接拒绝（`UNSUPPORTED_CONTENT`）；可选字段省略不发 null。
- 代码：[packages/llm/llm-deepseek/src/serialize.ts](../packages/llm/llm-deepseek/src/serialize.ts) 的 `serializeRequest()`/`serializeMessages()`。

### 4. HTTP + SSE 传输

- `fetch POST ${baseURL}/chat/completions`，headers：`authorization: Bearer`、`accept: text/event-stream`、`attributionHeaders()` 产出的 user-agent（强制不可抑制）、`x-deepseek-harness-user-id/-session-id/-compact`。
- `AbortSignal.any` 组合外部取消与 idleWatchdog（默认 5 分钟无流活动 → TIMEOUT）。
- 非 2xx 解析 WireError 后 `httpErrorCode()` 映射：401/403→AUTH、429→RATE_LIMIT、400→CONTEXT_WINDOW_EXCEEDED 或 INVALID_REQUEST、≥500→SERVER；透传 `retry-after` → `providerRetryAfterMs`、`x-request-id` → requestId。
- SSE 解析用 `eventsource-parser`，`[DONE]` 哨兵结束；EOF 无 `[DONE]` → `STREAM_CLOSED`。
- 代码：[packages/llm/llm-deepseek/src/adapter.ts](../packages/llm/llm-deepseek/src/adapter.ts) 的 `DeepSeekAdapter.stream()`/`request()`；[packages/llm/llm-deepseek/src/sse.ts](../packages/llm/llm-deepseek/src/sse.ts) 的 `parseSse()`。

### 5. 流翻译为中立协议

有状态翻译器把 DeepSeek delta 流翻译为 `StreamChunk`。

- `reasoning_content` → `reasoning-delta`（首个空串不开块）；text/tool_calls delta 按 `call.index` 聚合拼接。
- block-end / usage / finish 延迟到 `[DONE]` 才发出（`pendingFinish`/`pendingUsage`）。
- token 口径修正：DeepSeek 的 `prompt_tokens` 含缓存命中，而 harness 要求 disjoint 计数，故 `inputTokens = prompt_tokens - cacheRead`（cacheRead 取 `prompt_tokens_details.cached_tokens ?? prompt_cache_hit_tokens`）。
- finish_reason 映射：stop→stop、tool_calls→tool-calls、length→max-tokens、其他→error；stop 但无任何块 → `EMPTY_RESPONSE`；坏 JSON → `MALFORMED_RESPONSE`。
- 代码：[packages/llm/llm-deepseek/src/translate.ts](../packages/llm/llm-deepseek/src/translate.ts) 的 `translate()`/`mapUsage()`/`mapFinishReason()`。

### 6. 组装、日志与工具循环

- 逐 chunk `session.append('assistant/chunk')` 持久化，同时喂给 [packages/llm/llm/src/assembler.ts](../packages/llm/llm/src/assembler.ts) 的 `BlockAssembler`（容忍 delta-only 协议，block-end 后忽略迟到 delta，捕获 usage/finish/replayState）。
- 成功 finish → 发 `assistant/message` 事件（含 usage）；有 tool-call 块 → `executeToolCalls()` 执行后把 tool-result 回灌下一轮请求，无工具调用则循环结束。
- 失败 finish（error/aborted）→ 抛 `LlmError` 或走错误恢复。

### 7. 错误与重试（dsh-llm-retry）

- [packages/llm/llm-retry/src/index.ts](../packages/llm/llm-retry/src/index.ts) 挂 `agent/request-error` waterfall：指数退避 + jitter 延迟后重试，重试前先持久化再等待；retryPolicy 由 provider 侧拥有（适配器注册时传入），插件自身无配置。
- `LlmError` 携带 frozen failure facts（code/httpErrorCode/requestId/providerRetryAfterMs）；`adapterStream()` 把适配器异常归一为 terminal error chunk。

## 类 / 模块速查表

| 类 / 模块 | 职责 | 文件 |
|---|---|---|
| `LlmRuntime` | 适配器注册表、`prepareCall` 校验冻结、`llm/stream` waterfall 编排 | [llm/index.ts](../packages/llm/llm/src/index.ts) |
| `LlmAdapter`（抽象） | provider 接口，`stream()` 为唯一必须方法 | [llm/index.ts](../packages/llm/llm/src/index.ts) |
| `BlockAssembler` | chunk 流 → ContentBlock → Message 组装 | [assembler.ts](../packages/llm/llm/src/assembler.ts) |
| `StreamChunk` 等类型 | 中立流协议词汇表 | [types.ts](../packages/llm/llm/src/types.ts) |
| llm-deepseek `apply()` | 插件注册、凭据/连接解析、原子换路由 | [llm-deepseek/index.ts](../packages/llm/llm-deepseek/src/index.ts) |
| `DeepSeekAdapter` | fetch + SSE 直连、快照解析、idle watchdog、错误映射 | [adapter.ts](../packages/llm/llm-deepseek/src/adapter.ts) |
| `serializeRequest` | OpenAI 兼容线格式序列化（thinking/tool/reasoning 处理） | [serialize.ts](../packages/llm/llm-deepseek/src/serialize.ts) |
| `parseSse` | SSE 解析、`[DONE]` 哨兵、STREAM_CLOSED | [sse.ts](../packages/llm/llm-deepseek/src/sse.ts) |
| `translate` | delta 流 → StreamChunk、usage 缓存扣减、finish 映射 | [translate.ts](../packages/llm/llm-deepseek/src/translate.ts) |
| agent-loop `step`/`buildRequest` | 请求构建、会话日志、工具调用循环 | [agent.ts](../packages/core/agent-loop/src/agent.ts) |
| llm-retry `apply` | `agent/request-error` 瀑布重试、指数退避 | [llm-retry/index.ts](../packages/llm/llm-retry/src/index.ts) |

## 核心设计结论

dsh 与 DeepSeek 的对接是纯传输层适配（transport-only）：所有 DeepSeek 特有逻辑（thinking 字段、reasoning_content 回传规则、缓存 token 口径、SSE 线格式）被封闭在 `llm-deepseek` 包内，经 `StreamChunk` 中立协议向上输出；编排能力（校验、冻结、waterfall 拦截、重试、日志重建）全部由 `dsh-llm` 和 agent-loop 以插件机制承担，更换 provider 只需注册新适配器。
