# 多 Agent 模式分析（DeepSeek Harness dsh-v0.2.0-rc.2）

> 分析对象：DeepSeek Harness 最新 release `dsh-v0.2.0-rc.2` 的多 Agent 实现，对应代码包组 `packages/subagent/`。

## 一、多 Agent 模式总览

DeepSeek Harness 的多 Agent 能力由 `packages/subagent/` 包组提供，能力缝（capability seam）= `ctx.subagents` Service Definition + 多个 Provider + `tool-subagent`/`tool-subagent-control` Consumer。模式可沿**三个独立维度**组合。

### 维度 1：执行后端（Provider，7 种）

| Provider | 包 | 进程边界 | 父对话继承 | 启动能力 | 用途 |
|---|---|---|---|---|---|
| `spawn` | subagent-spawn-in-process | 同进程 | 否 | 全部 | 最便宜的 fresh child |
| `fork` | subagent-fork-in-process | 同进程 | 是（completed-turn prefix） | 全部 | 继承父对话上下文 |
| `acp` | subagent-acp | 独立进程 | 否 | 无 | 任意 ACP 兼容 agent |
| `codex` | subagent-codex | 独立进程 | 否 | 无 | OpenAI Codex CLI |
| `claude-code` | subagent-claude-code | 独立进程 | 否 | 无 | Anthropic Claude Code SDK |
| `dsh-sdk` | subagent-dsh-sdk | 独立进程 | 可选 | 部分 | 另一个 dsh 实例（远程） |
| `agent-team` | experimental（preset 内组合） | 同进程 | 取决于组合 | — | 多个 preset agent 协作 |

### 维度 2：运行模式（`backgroundMode`）

| 模式 | 入口 API | 返回值 | 生命周期 |
|---|---|---|---|
| **one-shot** | `ctx.subagents.start()` | `SubagentRun`（含 `result` promise + `dispose`） | 单轮，结果回 parent 后释放 |
| **continuable** | `ctx.subagents.startContinuable()` | `{childId, messageId}` | 持久 child，多轮对话，cold-resume |

### 维度 3：通信拓扑

| 拓扑 | 工具 / API | 方向 |
|---|---|---|
| 委派 → 返回 | `subagent` 工具（foreground） | parent → child → result |
| 后台任务 | `subagent` 工具（`run_in_background:true`, one-shot） | parent → child（job） |
| 持续对话 | `subagent` 工具（continuable）+ `send_message` + `interrupt_agent` | 双向、可中断 |
| 父子对话 | `send_message`（child → parent） | child → parent（resident continuable child 才能向 parent 发消息） |

## 二、分层架构图

```
┌──────────────────────────────────────────────────────────────────────────┐
│                  L4: Model-Facing Tools (Consumer)                       │
│   tool-subagent          : subagent 工具（委派 + 后台 + continuable）     │
│   tool-subagent-control  : send_message / interrupt_agent / list_agents   │
└──────────────────────────────────────▲───────────────────────────────────┘
                                       │ ctx.subagents.*
┌──────────────────────────────────────┴───────────────────────────────────┐
│             L3: Service Definition (SubagentRuntime)                     │
│   - registerProvider / getProvider / list                                │
│   - start()            : one-shot 入口（验证 cap → provider.start）      │
│   - startContinuable() : continuable 入口（manager 接管生命周期）          │
│   - sendMessage()      : 相邻 Agent 间消息路由                            │
│   - interrupt()        : 中断 live Activation                            │
│   - listChildren/listDescendants : durable catalog 读取                  │
│   - SubagentContinuationManager : continuable 编排核心                   │
└──────────────────────────────────────▲───────────────────────────────────┘
                                       │ provider.start / prepareContinuable
┌──────────────────────────────────────┴───────────────────────────────────┐
│                  L2: Service Providers (实现)                             │
│   in-process 共享驱动 (subagent-in-process-driver.startInProcessRun):    │
│     - spawn : fresh child, 无 seed                                       │
│     - fork  : seed = parent.completedTurnPrefix()                        │
│   out-of-process:                                                        │
│     - acp         : startAcpRun (subprocess + ACP 协议)                  │
│     - codex       : startCodexRun (app-server --stdio)                   │
│     - claude-code : startClaudeCodeRun (Agent SDK)                       │
│     - dsh-sdk     : 通过 dsh SDK 启动独立 dsh 实例                       │
└──────────────────────────────────────▲───────────────────────────────────┘
                                       │ agents.create / subprocess.spawn
┌──────────────────────────────────────┴───────────────────────────────────┐
│             L1: Runtime 基础 (Agent / Session / Subprocess)               │
│   ctx.agents.create() : in-process child Agent 工厂（事务化创建）          │
│   ctx.subprocess.spawn() : out-of-process 子进程                          │
│   SessionLog + Catalog Projection : durable 父子关系持久化                │
└──────────────────────────────────────────────────────────────────────────┘
```

## 三、流程图

### 3.1 one-shot 委派流程（spawn / fork / acp / codex / claude-code）

```
parent Agent loop
   │ exec.signal
   ▼
tool-subagent.execute(args)
   │
   ├─ resolveDelegationRun() → runInBackground=false (foreground 默认)
   ├─ parentAgentOptionsForDelegation(parent)
   ├─ preflightChildLlmRoute() (若指定 provider/model)
   │
   ▼
ctx.subagents.start(provider, request)
   │
   ├─ expectProvider(name)             找到 Provider
   ├─ assertCapabilities()             fail-loud 能力校验
   ├─ assertSubagentMaxDepth()         深度校验
   ├─ snapshotSubagentDescriptor({mode:'one-shot',...})
   │
   ▼
provider.start(resolvedRequest)        ┌─────────────────────────────────┐
   │                                   │ spawn/fork (in-process):       │
   │                                   │   startInProcessRun(req,seed?) │
   │                                   │   ├─ resolveChildDepth        │
   │                                   │   ├─ captureDelegatedPolicy   │
   │                                   │   ├─ agents.create({          │
   │                                   │   │    sessionId, parentAgent, │
   │                                   │   │    meta, seed?, setup})    │
   │                                   │   ├─ drivePublishedRun(handle)│
   │                                   │   │   ├─ child.followup(prompt)│
   │                                   │   │   ├─ child.whenIdle()     │
   │                                   │   │   └─ readResult()          │
   │                                   │   └─ return SubagentRun         │
   │                                   │                                 │
   │                                   │ acp/codex/claude-code (oop):   │
   │                                   │   startAcpRun/startCodexRun   │
   │                                   │   ├─ subprocess.spawn()        │
   │                                   │   ├─ 跨进程协议握手            │
   │                                   │   ├─ 发 prompt / 接收 SSE      │
   │                                   │   └─ 返回 SubagentRun          │
   ▼                                   └─────────────────────────────────┘
establishCatalogChild(parent.session, child.header, descriptor)
   │  (写父 session 的 subagent/catalog 事件)
   ▼
observeRun(emitLifecycle, provider, parent, run)
   │  (发布 subagent/start 事件)
   ▼
tool-subagent.settleForegroundRun(run)
   ├─ run.result → SubagentResult
   ├─ stopReasonError() 把非 completed 映射为 isError
   └─ run.dispose()                    释放 child handle
   ▼
返回 tool output 给 parent Agent loop
```

### 3.2 continuable 多轮对话流程

```
parent turn 1
   │
   ▼
tool-subagent.execute (runInBackground=true, continuable=true)
   ▼
ctx.subagents.startContinuable({provider, label, request, signal})
   ▼
SubagentContinuationManager.startContinuable()
   ├─ activations.assertAdmitting(parent)              容量校验（maxActiveSubagents）
   ├─ persistence.stat(childId)                         检查持久化是否已存在
   ├─ host.prepareContinuable(provider, {sessionId, parent, signal})
   │    └─ provider.prepareContinuable() 返回 {seed?}   (fork 返回 prefix, spawn 返回 {})
   ├─ activations 创建 Activation（占住 childId + parent 所有权）
   ├─ agents.create({sessionId, parentAgent, meta, seed, inheritedEventCount, setup})
   │    setup 内：appendDelegatedPolicyOverrides / applyChildComposition
   ├─ child.followup(initialPrompt)                     首条 prompt 进 inbox
   ├─ establishCatalogChild()                           durable 父子关系
   ├─ 发布 subagent/start
   └─ 返回 {childId, messageId}                         ★ 不等结果
   ▼
tool 返回 {kind:'continuable', subagentId}
   ▼
parent 继续自己的工作，child 在后台跑首轮流

—————— 后续多轮 ——————

parent 或 child 调用 send_message 工具:
   ▼
ctx.subagents.sendMessage(sender, targetId, content, options)
   ▼
SubagentContinuationManager.sendMessage()
   ├─ 校验 sender 与 target 的相邻关系（直系父子）
   ├─ child 在跑: steer 模式 → 下一步边界注入
   ├─ child 空闲: 启动新 turn
   └─ child 不在内存: cold-resume 从 persistence 加载
   ▼
返回 messageId

—————— 中断 ——————

parent 或 host 调用 interrupt_agent:
   ▼
ctx.subagents.interrupt(targetSessionId, {kind:'user'|'ancestor',...})
   ▼
ActivationRegistry.interrupt()
   └─ child.cancel({kind:'parent'})                    发出取消信号
   (不等待；child 观察到信号后自行收尾)

—————— 终止 ——————

parent Agent disposed:
   ▼
ctx.subagents.drainContinuableDescendants(parents) / drainContinuableChildren(parent, ids)
   ▼
ActivationRegistry 释放整片或选定 child 树
   └─ 每个 child: handle.dispose() → child-first 释放
```

## 四、时序图：continuable 多轮对话

```
parent Agent          SubagentRuntime       ContinuationMgr       Provider         AgentFactory       child Agent
    │                      │                      │                  │                  │                   │
    │──startContinuable──▶│                      │                  │                  │                   │
    │                      │──prepareContinuable▶│                  │                  │                   │
    │                      │                      │──prepareContinuable (provider)────▶│                   │
    │                      │                      │◀─────────────{seed?}─────────────│                   │
    │                      │                      │──agents.create({seed,setup,...})──────────────────▶  │
    │                      │                      │◀─────────────────AgentHandle──────────────────────  │
    │                      │                      │──child.followup(initialPrompt)────────────────────▶│
    │                      │                      │──emit subagent/start                              │
    │◀────{childId,msgId}──│                      │                  │                  │                   │
    │                      │                      │                  │                  │   child 首轮流    │
    │  ... parent 干别的活 ...                                                                                   │
    │                                                                                                          │
    │──send_message(childId,"...")──▶│                  │                  │                  │                   │
    │                      │──sendMessage─────────▶│                  │                  │                   │
    │                      │                      │──校验相邻→followup(msg)──────────────────────────────────▶│
    │◀──────{messageId}────│                      │                  │                  │   child 新 turn    │
    │                                                                                                          │
    │  ... 或 child 主动向 parent 发 ...                                                                       │
    │                  ▶──send_message(parentId,msg)─────────────────────────────────────────────────────▶│
    │◀────── 注入 parent inbox ─────────────────────────────────────────────────────────────────────────│  │
    │                                                                                                          │
    │──interrupt_agent(childId)──▶│                  │                  │                  │                   │
    │                      │──interrupt────────────▶│                  │                  │   child.cancel()  │
    │                      │                      │──────────────────────────────────────────cancel────▶   │
    │◀───── accepted ──────│                      │                  │                  │                   │
```

## 五、关键类、文件与代码

### L3：Service Definition

| 类/文件 | 职责 | 关键代码 |
|---|---|---|
| `SubagentRuntime` (`packages/subagent/subagent/src/index.ts`) | 注册中心 + start/startContinuable/sendMessage/interrupt/list 入口 | `start` L559-L589；`startContinuable` L261-L263；`sendMessage` L279-L286；`interrupt` L328-L330；`registerProvider` L512-L528 |
| `SubagentContinuationManager` (`packages/subagent/subagent/src/continuation.ts`) | continuable 编排：reserve id → prepare → create → deliver → cold resume → drain | `startContinuable` L104+；`sendMessage`/`steerPrompt`/`queuePrompt`；`interrupt` |
| `ContinuableActivationRegistry` (`packages/subagent/subagent/src/continuation-activation.ts`) | 进程内 Activation 图：父子所有权、容量、cold resume、drain | `assertAdmitting`、`holdOwnership`、`drainDescendants` |
| `SubagentProvider` 接口 (`packages/subagent/subagent/src/types.ts` L344) | Provider 契约：`name`/`capabilities`/`inheritsParentContext`/`start`/`prepareContinuable?` | L344-L390 |
| `SubagentCapabilities` (`packages/subagent/subagent/src/types.ts` L130) | `agentOptions`/`outputSchema`/`depthLimit`/`toolFilter`/`persona` 五项 start-time 能力位 | L130-L136 |
| `SubagentRun` (`packages/subagent/subagent/src/types.ts` L308) | one-shot 句柄：`id`/`localAgent?`/`result`/`dispose` | L308-L334 |
| `lifecycle.ts` | `subagent/start` / `subagent/end` 事件发布 + scope 过滤 | `observeRun`、`createActivationObserver` |
| `catalog.ts` | durable 父子目录投影（`subagent/catalog` 事件，`sessionProjections`） | `establishCatalogChild` |
| `depth.ts` | 委派深度递增与上限校验 | `assertSubagentMaxDepth`、`resolveChildDepth` |
| `child-agent.ts` | 子 agent 选项解析、policy override 捕获/应用、child composition | `resolveChildAgentOptions`、`captureDelegatedPolicyOverrides`、`applyChildComposition` |

### L4：Model-Facing Tools

| 文件 | 工具 | 关键代码 |
|---|---|---|
| `packages/subagent/tool-subagent/src/index.ts` | `subagent` | `apply` L313+；`execute` L471-L568；`resolveDelegationRun` L287-L305；`settleForegroundRun` L208-L238 |
| `packages/subagent/tool-subagent-control/src/index.ts` | `send_message`、`interrupt_agent`、`list_agents` | L28-L72 send_message；L74+ interrupt_agent |

关键设计：tool-subagent 通过 `providerWording(inheritsParentContext)` 动态生成 tool description（fork 模式说"builds on this conversation"，spawn 模式说"does not share this conversation"）。

### L2：Service Providers

| Provider | 文件 | 关键代码 |
|---|---|---|
| spawn | `packages/subagent/subagent-spawn-in-process/src/index.ts` | `start` 返回 `startInProcessRun(request, {})`（无 seed）；`prepareContinuable` 返回 `{}` |
| fork | `packages/subagent/subagent-fork-in-process/src/index.ts` | `completedTurnPrefix()` L48-L55 切到 `turn/end`；`start` 传 seed |
| 共享驱动 | `packages/subagent/subagent-in-process-driver/src/index.ts` | `startInProcessRun` L104-L152；`drivePublishedRun` L158-L209；`readResult` L212-L237 |
| acp | `packages/subagent/subagent-acp/src/index.ts` + `run.ts` | `AcpProvider`：capabilities 全 false；`start` 调 `startAcpRun(request, spec)` |
| codex | `packages/subagent/subagent-codex/src/index.ts` + `run.ts` | `CodexProvider`：`NO_START_CAPABILITIES`；`startCodexRun` 启动 `app-server --stdio` |
| claude-code | `packages/subagent/subagent-claude-code/src/index.ts` + `run.ts` | 调用官方 Agent SDK |
| dsh-sdk | `packages/subagent/subagent-dsh-sdk/src/index.ts` | 通过 dsh SDK 远程启动另一个 dsh 实例 |

### L1：基础

| 文件 | 职责 |
|---|---|
| `packages/core/agent/src/index.ts` (`ctx.agents`) | `agents.create({sessionId, parentAgent, meta, seed, agentOptions, setup, signal})` — 事务化创建 child Agent，失败回滚到 quiescence |
| `packages/subprocess/subprocess/src/index.ts` (`ctx.subprocess.spawn`) | out-of-process 子进程 spawn + 凭证擦除 + 终止分层 |
| `packages/session/...` | durable session log + cold resume（continuable child 重启时从此恢复） |

## 六、模式识别速查

| 你想要的模式 | 配置 |
|---|---|
| 简单委派（不继承对话） | provider=`spawn`, backgroundMode=`one-shot` |
| 委派（继承父对话） | provider=`fork`, backgroundMode=`one-shot` |
| 独立 CLI agent（Codex/Claude） | provider=`codex`/`claude-code` |
| 后台任务（job） | provider=*, backgroundMode=`one-shot`, `run_in_background:true` |
| 持续可对话子 agent | provider=*, backgroundMode=`continuable`（provider 须实现 `prepareContinuable`，spawn/fork 都实现） |
| 子→父消息 | child 是 resident continuable + `send_message(parentId)` |
| 中断 child | `interrupt_agent(childId)` 或 `ctx.subagents.interrupt()` |
| 多 child 并行 | parent 一次 assistant message 启动多个 continuable（系统提示鼓励"start independent delegations together in one assistant message"） |
| 深度限制 | `maxDepth` 配置（0 禁用委派，默认 1，`'provider-managed'` 交给外部进程） |

`maxActiveSubagents` 默认 8，控制同一 parent 的 continuable 子数量上限。

## 七、设计原则

1. **能力缝（capability seam）完整**：`SubagentRuntime`（Service Definition）+ `SubagentProvider`（Provider 接口）+ `tool-subagent`/`tool-subagent-control`（Consumer）三角色分离，可独立演化。
2. **fail-loud 能力校验**：`assertCapabilities()` 在 start 前逐项检查 request 所需的 `agentOptions`/`outputSchema`/`depthLimit`/`toolFilter`/`persona`，不支持就拒绝而不是静默忽略。
3. **注册即效果**：provider 通过 `ctx.effect()` 注册，HMR 安全，移除只阻止新 start 不影响已发布的 run。
4. **durable 优先**：所有父子关系、descriptor、catalog 都写入 session log，进程重启可 cold-resume continuable child。
5. **所有权单一**：one-shot run 的所有权在 `start` 返回时一次性转移；continuable child 的整个生命周期由 `ContinuationManager` 独占，provider 只贡献 detached 创建数据。
6. **进程隔离边界**：out-of-process provider（acp/codex/claude-code）声明 `NO_START_CAPABILITIES`，无法继承父对话、无法接受 outputSchema/agentOptions 等，因为这些都依赖同进程的 typed 边界。
