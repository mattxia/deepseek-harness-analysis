# Context 管理架构分析（DeepSeek Harness dsh-v0.2.0-rc.2）

> 分析对象：DeepSeek Harness 最新 release `dsh-v0.2.0-rc.2` 的 Context 管理实现。

## 一、分层架构

Context 管理分两条互补路径，统一由 `core/system-prompt` 注册表驱动，最终都落到 durable session log 的 user/system 消息中。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          L4: 投影 & 持久化层 (Projection)                      │
│   SystemPromptProjection (system/message 表面节点)                            │
│   RuntimeContextProjection  (runtime-context user 消息去重)                    │
│   sessionProjections        (timeContext / tmuxContext 状态折叠)               │
└──────────────────────────────────────▲──────────────────────────────────────┘
                                       │ 渲染 & 提交
┌──────────────────────────────────────┴──────────────────────────────────────┐
│                       L3: Agent Loop 消费层 (agent-loop)                      │
│   preStep(): systemPrompt.assemble() → renderPrompt / renderContextSnapshot │
│   → waterfall 'agent/pre-step' (各 context 插件在此注入)                     │
└──────────────────────────────────────▲──────────────────────────────────────┘
                                       │ assemble()
┌──────────────────────────────────────┴──────────────────────────────────────┐
│                     L2: 核心注册表 (core/system-prompt)                       │
│   SystemPrompt service: section() context() tools() variable()              │
│   ScopedLayers (global + per-agent scope)  → PromptAssembly                 │
│   renderPrompt() / renderContextSections()  ({{variable}} 插值)              │
└──────────────────────────────────────▲──────────────────────────────────────┘
                                       │ 注册贡献
┌──────────────────────────────────────┴──────────────────────────────────────┐
│        L1: Context 贡献者层 (context/ + 各能力插件)                            │
│                                                                              │
│  路径A: systemPrompt.section (系统提示节)                                      │
│    · harness:identity  · persona-prefix/suffix  · 工具描述(每工具插件)         │
│    · plan/team policy  · file-reference  · deliverable refs                   │
│                                                                              │
│  路径B: systemPrompt.context (动态运行时快照)                                  │
│    · SANDBOX_POLICY (sandbox-policy)        order 110                         │
│    · APPROVAL_POLICY (user-approval)        order 115                         │
│    · SUBAGENT_DELEGATION (subagent)         order 120                         │
│                                                                              │
│  路径C: agent/pre-step 直接注入 durable user 消息                               │
│    · agent-instructions  (AGENTS.md 增量)                                     │
│    · time-context       (时间/时区, 节流)                                     │
│    · tmux-context       (tmux 位置/布局)                                      │
│    · session-reference  (@session 引用快照)                                   │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 二、单轮请求 Context 组装流程

```
turn start
    │
    ▼
preStep()
    │
    ├─① systemPrompt.assemble({scope, agent, signal})
    │     ├─ 收集 global + scope sections (scoped 覆盖 global)
    │     ├─ 收集 contexts (SANDBOX/APPROVAL/SUBAGENT)
    │     ├─ 收集 tools (所有工具插件 schema)
    │     ├─ 收集 variables (provider/model/cwd/...)
    │     └─ waterfall 'system-prompt/assemble' (可二次改写)
    │
    ├─② renderContextSections(assembly) → sections[]
    │   runtimeContext.project(joinContextSections(sections), sections)
    │     └─ 与上次保留快照比对，不同则产出 user message "Current runtime context..."
    │        (否则 undefined，不重复写入)
    │
    ├─③ waterfall 'agent/pre-step'
    │     ├─ session-reference: 解析 @session → 注入引用快照 user message
    │     ├─ time-context: 超 refreshIntervalMs 则注入时间 user message
    │     ├─ tmux-context: step===1 且 tmux 状态变化则注入位置 user message
    │     ├─ agent-instructions: AGENTS.md 变更 → 注入/更新指令 user message
    │     └─ 各插件可追加 messages
    │
    ▼
step(): prepareRequest → renderPrompt(assembly) → systemPrompt.project()
    │   └─ system/message 表面节点 (首条 or in-history 替换)
    ▼
LLM request: system message + user messages(含 runtime context + 各插件注入) + tools
```

## 三、涉及的类、文件与关键代码

### L2 核心注册表 — `SystemPrompt`

**文件**：`packages/core/system-prompt/src/index.ts`

| 类/接口 | 职责 | 关键代码 |
|---|---|---|
| `SystemPrompt` (extends `Service`) | 注册中心：sections / contexts / tools / variables，按 scope 分层，`assemble()` 产出 `PromptAssembly` | L405-L636 |
| `PromptSection` | 系统提示节：`name/order/text(静态或函数)/interpolate/complete` | L53-L76 |
| `PromptContext` | 动态运行时上下文：`name/order/text`，渲染为 `runtime-context` user 消息 | L79-L86 |
| `PromptLayer` | 单个 scope 层的全部注册项（sections/contexts/toolProviders/variables/suppressors） | L371-L402 |
| `ScopedLayers` | 来自 `dsh-scope`，管理 global + scope 链，scope 覆盖 global | 构造器 L415-L418 |
| `renderPrompt()` | section 文本 `{{variable}}` 严格插值后用空行拼接 | L279-L284 |
| `renderContextSections()` | contexts 插值后产出 `ContextSnapshotSection[]`，供投影层比对 | L318-L322 |

注册 API：`section()` / `context()` / `tools()` / `variable()` / `suppressRuntimeContext()`，均返回 Cordis effect disposer。

关键设计：
- `complete: true` 的 section 会作为唯一系统提示（水印模式），多于一个则失败。
- `toolOrder` 配置要求包含 `<unlisted-tools>` 占位，未知工具名在 assemble 时失败。
- 变量名必须匹配 `[a-z][a-z0-9_]*`，未注册或 undefined 变量在渲染时抛错（fail loud）。
- `assemble()` 走 `waterfall 'system-prompt/assemble'`，监听器可改写 assembly，但不能添加 complete section。

### L3 Agent Loop 消费 — `preStep` 与投影

**文件**：`packages/core/agent-loop/src/agent.ts`

| 位置 | 职责 |
|---|---|
| L272 | `systemPrompt.assemble(assembleContextFor(this, signal))` 触发组装 |
| L274-L275 | `renderContextSections` → `runtimeContext.project(...)` 生成 runtime-context user 消息 |
| L276-L282 | `waterfall 'agent/pre-step'`，各 context 插件在此注入消息 |
| L405-L418 | `renderPrompt(assembly)` → `systemPrompt.project()` → 提交 `system/message` 表面节点 |

变量提供者（`packages/core/agent-loop/src/index.ts` L370-L372）：

```ts
ctx.systemPrompt.variable('provider', context => context.agent?.options.provider)
ctx.systemPrompt.variable('model', context => context.agent?.options.model)
ctx.systemPrompt.variable('cwd', context => context.agent?.session.header.cwd)
```

### L4 投影层 — 去重与持久化

**文件**：`packages/core/agent-loop/src/runtime-context.ts`

| 类 | 职责 | 关键代码 |
|---|---|---|
| `SystemPromptProjection` | 决定系统提示如何落到 `system/message` 表面节点：首条 append，in-history 模式下变化时 append 新节点，否则 replace head | L65-L111 |
| `RuntimeContextProjection` | 跟踪上次保留的 runtime-context 快照，仅在文本变化时产出新 user 消息；监听 session 事件同步保留状态 | L114-L163 |

关键设计：`RuntimeContextProjection.project()` 比对 `retained.text === snapshot`，相同返回 `undefined`（不重复写入 session log）。runtime-context 无内容时写入 `CLEARED` 标记而非删除，明确告知模型早期快照已失效。

## 四、Context 贡献者详细说明

### 路径 B：运行时上下文贡献者（`systemPrompt.context`）

#### 1. Sandbox Policy Context
**文件**：`packages/sandbox/sandbox-policy/src/index.ts` L142

注册 `SANDBOX_POLICY`（order 110），向模型说明当前沙箱模式（`read-only` / `workspace-write` / `danger-full-access`）及可写根目录。

#### 2. User Approval Policy Context
**文件**：`packages/interaction/user-approval/src/index.ts` L163

注册 `APPROVAL_POLICY`（order 115），说明当前审批策略（哪些工具需用户批准、自动审批配置）。

#### 3. Subagent Delegation Context
**文件**：`packages/subagent/subagent/src/child-agent.ts` L206

注册 `SUBAGENT_DELEGATION`（order 120），子 agent 说明自身委派关系与边界。

### 路径 C：pre-step 注入的 durable context 插件

这些插件不通过 `systemPrompt.context`，而是监听 `agent/pre-step` waterfall，直接向决策的 `messages` 追加 user 消息，带专属 `source.kind` 便于持久化归因与去重。

#### 1. Agent Instructions（AGENTS.md）
**文件**：`packages/context/agent-instructions/src/index.ts`

| 机制 | 说明 |
|---|---|
| 基线加载 | `loadBaselineInstructionSet()` 从项目根 + `dshHome` 扫描 `AGENTS.md`/`CLAUDE.md` 等候选文件，合并 included/observed scopes |
| 增量更新 | 监听 `tools/result`，对 `read/write/edit` 工具触及的文件路径触发 `queueProjection()`，重新 reconcile 指令上下文 |
| 版本管理 | `InstructionVersionCache` (WeakMap) 按 session 缓存各 scope 的指令版本，避免重复加载未变更文件 |
| 去重 | `syncInbox()` 比对 `sameContextPayload`，复用或替换 inbox 中的 agent-instructions 消息 |
| 关键事件 | L315-L341 `agent/pre-step` 中 compose 指令并插入到 claimed batch 之后 |
| 触发 | L343-L359 `tools/result` 中收集 touched paths |

`source.kind = 'agent-instructions'`，带 `baseline`/`baselineIdentity`/`changes` 字段支持替换与变更追踪。

#### 2. Time Context
**文件**：`packages/context/time-context/src/index.ts`

| 机制 | 说明 |
|---|---|
| 节流 | `refreshIntervalMs`（默认 10 分钟），距上次注入不足则跳过 |
| 时区 | 从 user 消息中推导浏览器时区（`deriveBrowserTimeZoneContext`），无则用进程时区 |
| 投影状态 | `sessionProjections.register('timeContext')` 记录 `lastMessageTime`/`lastInjectionTime`/`lastTurnInjectionTime` |
| 注入 | L185-L225 `agent/pre-step` prepend 监听，注入含时间戳+时区+距上条消息耗时的 user 消息 |
| `source.kind` | `'time-context'`，`form: 'snapshot'` |

#### 3. Tmux Context
**文件**：`packages/context/tmux-context/src/index.ts`

| 机制 | 说明 |
|---|---|
| 时机 | 仅 `step === 1`（每轮首次请求） |
| 真实性校验 | `queryTmuxLocation()` 通过 `$TMUX_PANE` + `#{pane_tty}` 对比本进程控制终端，区分真正在 tmux 内 vs 仅继承环境变量 |
| 去重 | `sessionProjections.register('tmuxContext')` 记录稳定状态块（不含 turn 前言），状态未变则跳过 |
| 内容 | session/window/pane 名称、索引、active 状态、`window_layout` |
| 容错 | shell 拒绝或查询失败仅 warn，不中断 turn |

#### 4. Session Reference
**文件**：`packages/context/session-reference/src/index.ts`

| 机制 | 说明 |
|---|---|
| 解析 | `parseSessionReferenceText()` 解析用户消息中的 `@session` 规范提及 |
| 快照 | `prepare()` 读取被引用 session 的 surface，按字节预算 `retainReferencedSession()` 截断 |
| 预算 | `referenceBudget()` 按模型 context window × `referenceContextFraction`（默认 0.2）× 4 bytes/token 计算，无模型信息则用 64 KiB 默认 |
| 溢出处理 | `prepareReferenceOmission()` 将截断的完整快照 spill 到 spill store，提示中包含 omittedBytes/omittedMessages |
| 注入 | L135-L142 `agent/pre-step` prepend，在引用消息后紧跟快照 user 消息 |
| `source.kind` | `'session-reference'`，`form: 'recall'` |

#### 5. File Reference（发现缝）
**文件**：`packages/context/file-reference/src/index.ts` + `file-reference-local`

`FileReferenceService` 是抽象服务，提供 `@path` 自动补全候选列表；本地实现 `file-reference-local` 按 cwd 扫描文件。实际文件内容不直接注入 context，而是在提示中告诉模型"@ 前缀是用户显式引用的路径，需用 read 工具读取"（`FILE_REFERENCE_PROMPT`）。

## 五、核心设计原则

1. **注册即效果**：所有 section/context/tool/variable 通过 `ctx.effect()` 注册，disposer 自动清理，支持 HMR。
2. **Scope 分层**：global 注册被 agent-scope 同名注册覆盖（`ScopedLayers`），支持 per-agent 定制。
3. **增量投影**：runtime-context 和 tmux/time 上下文都做"内容未变则不写"，避免 session log 冗余。
4. **持久化归因**：每条注入消息的 `source.kind` 标识生产者，投影状态从 durable events 还原，不依赖进程内缓存（支持 compaction 与进程恢复）。
5. **两条路径分离**：
   - 系统提示（section）→ `system/message` 表面节点，决定模型身份与工具能力；
   - 运行时快照（context + pre-step 注入）→ `user/message`，随每轮更新，旧快照被新快照或 `CLEARED` 标记取代。
6. **预算与截断**：session-reference 按模型上下文窗口计算字节预算，溢出部分 spill 到存储并在提示中声明 omission。

## 六、关键代码路径索引

| 关注点 | 路径 |
|---|---|
| 系统提示注册表 | `packages/core/system-prompt/src/index.ts` |
| agent loop 组装消费 | `packages/core/agent-loop/src/agent.ts` |
| runtime-context 投影 | `packages/core/agent-loop/src/runtime-context.ts` |
| 变量提供者 | `packages/core/agent-loop/src/index.ts` L370-L372 |
| 沙箱策略 context | `packages/sandbox/sandbox-policy/src/index.ts` L142 |
| 审批策略 context | `packages/interaction/user-approval/src/index.ts` L163 |
| 子 agent 委派 context | `packages/subagent/subagent/src/child-agent.ts` L206 |
| AGENTS.md 指令 | `packages/context/agent-instructions/src/index.ts` |
| 时间上下文 | `packages/context/time-context/src/index.ts` |
| tmux 上下文 | `packages/context/tmux-context/src/index.ts` |
| session 引用 | `packages/context/session-reference/src/index.ts` |
| 文件引用发现 | `packages/context/file-reference/src/index.ts` |
