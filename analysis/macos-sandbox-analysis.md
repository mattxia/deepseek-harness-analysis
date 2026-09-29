# macOS 沙箱机制实现架构分析（dsh-v0.2.0-rc.2）

> 分析对象：DeepSeek Harness 最新 release `dsh-v0.2.0-rc.2` 的 macOS 沙箱实现。
> 结论：MAC 版本完整支持进程沙箱，机制是系统自带的 `sandbox-exec` CLI + Seatbelt（SBPL profile），enforcement 完整度报告为 `full`。

## 1. 是否支持沙箱

**支持。** macOS 走 `sandbox-exec`（系统内置，Apple 标记 deprecated 但每台 mac 都自带），不依赖原生 addon。

- **不是**原生 addon：`native/` 的 `node-addon-system` 只为 Linux 提供 `landlock-run`；macOS 只有 `system.node`（flock 绑定）。
- **不是** App Sandbox entitlements：是命令行 `sandbox-exec` 直接在用户态拉起，由内核 Seatbelt 强制执行。
- enforcement 完整度：`full`（seatbelt profile 覆盖全部承诺的文件效应）。

## 2. 分层组件架构

沙箱是一个典型的 Cordis 能力缝（capability seam），分五层：**消费者 → 策略服务 → 服务定义 → 本地 Provider → 内核**。

```
┌────────────────────────────────────────────────────────────┐
│ Consumer        SandboxBashExecutor.execute()             │
│                 策略解析 → confine() 取受限 argv → spawn   │
├────────────────────────────────────────────────────────────┤
│ Policy          SandboxPolicyService.resolve()             │
│                 session 覆盖 > 部署默认 → SandboxPolicy    │
├────────────────────────────────────────────────────────────┤
│ Service Def     SandboxProvider.confine()  «abstract»      │
│                 ctx.sandbox 契约：受限 argv 或 fail-closed │
├────────────────────────────────────────────────────────────┤
│ Provider        LocalSandboxProvider.confine()             │
│                 darwin → seatbelt（唯一候选，enforcement=full）
├────────────────────────────────────────────────────────────┤
│ Kernel          sandbox-exec -p "SBPL"                     │
│                 deny file-write* + /dev/null & workspace 例外
└────────────────────────────────────────────────────────────┘
```

其他平台：Linux `bwrap → landlock`，Windows `windows-acl`（partial）。

## 3. 每层组件、职责与代码

### 第 1 层：消费者（Consumer）—— 工具执行器

| 组件 | 角色 | 职责 | 代码 |
|---|---|---|---|
| `SandboxBashExecutor` | Consumer | 继承 `LocalBashExecutor`；`resolve()` 盖章 per-call 策略；`execute()` 调 `ctx.sandbox.confine(['bash','-c',cmd])` 拿受限 argv 后 spawn；`onProcessDone` 用 `denialSignatures`/`runnerFailureRules` 把失败分类成 denial（沙箱正常拦截）或 `SANDBOX_UNAVAILABLE`（runner 自己崩了） | `packages/shell/bash-sandbox/src/index.ts` |
| `SandboxedFileSystem` | Consumer | 文件系统侧的**进程内策略围栏**（非内核边界）：`read-only` 拒绝一切写；`workspace-write` 校验目标 canonical 路径在 `writableRoots` 之内，否则抛 `FS_SANDBOX_DENIED` | `packages/fs/fs-sandbox/src/index.ts` |
| `SandboxPwshExecutor` | Consumer | PowerShell 版（mac 上不启用，win32 才启用） | `packages/shell/pwsh-sandbox/src/index.ts` |

消费者核心调用：先 `ctx.sandboxPolicy.resolve({session})` 拿 `SandboxExecutionPolicy`，再 `ctx.sandbox.confine(argv, policy, signal)` 拿 `ConfinedArgv`，用返回的 argv 替换原始 argv 去 spawn。

```ts
// packages/shell/bash-sandbox/src/index.ts
private confine(command: string, policy: SandboxPolicy, signal?: AbortSignal): Promise<ConfinedArgv> {
  return this.ctx.sandbox.confine(['bash', '-c', command], policy, signal)
}
```

### 第 2 层：策略服务（Policy）—— `ctx.sandboxPolicy`

单一策略归属：部署默认 mode + fallback root + per-session 覆盖解析。

| 组件 | 职责 | 代码 |
|---|---|---|
| `SandboxPolicyService` | `resolve(request)` 仲裁优先级：显式 approved mode > session 最后一条 `sandbox/mode` 事件 > 部署默认（`read-only`）；workspace root 取 session 不可变 cwd，无 session 则取配置 fallback。还向 system prompt 注入当前策略文本 | `packages/sandbox/sandbox-policy/src/index.ts` |

`base` bundle 默认把 mode 设成 `workspace-write`（经 `DSH_PERMISSION_MODE` 覆盖）：

```yaml
# packages/bundle/base/cordis.patch.yml
- id: sandbox
  name: '@deepseek-ai/dsh-sandbox-local'
- id: sandbox-policy
  name: '@deepseek-ai/dsh-sandbox-policy'
  config:
    mode: !!js process.env.DSH_PERMISSION_MODE ?? 'workspace-write'
    workspaceRoot: !!js process.cwd()
- id: bash-sandbox
  name: '@deepseek-ai/dsh-bash-sandbox'
  disabled: !!js process.platform === 'win32'
```

### 第 3 层：服务定义（Service Definition）—— `ctx.sandbox` 抽象缝

跨平台契约，不绑任何后端。

| 组件 | 职责 | 代码 |
|---|---|---|
| `SandboxProvider`（abstract `Service`）| 唯一方法 `confine(argv, policy, signal): Promise<ConfinedArgv>`。契约铁律：**必须返回受限 argv 或 fail-closed，绝不允许静默放行** | `packages/sandbox/sandbox/src/index.ts` |
| 类型集 | `SandboxMode`（`read-only`/`workspace-write`/`danger-full-access`）、`SandboxPolicy`、`ConfinedArgv`、`RunnerFailureRule` | 同文件 |
| `SandboxUnavailableError` | 无可用后端时抛出，携带 `SANDBOX_UNAVAILABLE` 码 | 同文件 |
| `escalation.ts`/`diagnostics.ts`/`roots.ts` | 提权审批（approved 重试更宽 mode）、runner 失败分类、`writableRoots`/`canonicalPath` 共享辅助 | `packages/sandbox/sandbox/src/` |

### 第 4 层：本地 Provider（实现）—— `ctx.sandbox` 的具体后端

**MAC 路径核心在这一层。** `LocalSandboxProvider` 按平台选 runner 链、构造 profile、返回受限 argv。

| 组件 | 职责 | 代码 |
|---|---|---|
| `PLATFORM_CHAINS` | 平台→候选 runner 映射。`darwin: ['seatbelt']`（唯一候选，**不探测**，直接选） | `packages/sandbox/sandbox-local/src/index.ts` |
| `STATIC_ENFORCEMENT` | 唯一候选的 enforcement 断言。`seatbelt: 'full'`（profile 构造即覆盖全部承诺的文件效应） | 同文件 |
| `LocalSandboxProvider.confine()` | 规范化 `workspaceRoot` → `selectRunner` → `runnerArgv` 拼接 → 返回 `{argv, enforcement, denialSignatures, runnerFailureRules}` | 同文件 |
| `runnerArgv('seatbelt')` | `[this.seatbeltExec(), ...seatbeltProfileArgs(policy)]`，`seatbeltExec()` 默认就是系统 `sandbox-exec` | 同文件 |
| `seatbeltProfileArgs()` | **SBPL profile 构造器**（见下） | `packages/sandbox/sandbox-local/src/profiles.ts` |
| `DENIAL_SIGNATURES.seatbelt` | `['operation not permitted']`（EPERM） | `packages/sandbox/sandbox-local/src/index.ts` |
| `RUNNER_FAILURE_RULES.seatbelt` | `[{fatalSignatures: ['sandbox-exec: ']}]` | 同文件 |

**MAC 的 SBPL profile 实际内容**（`read-only` 时）：

```ts
// packages/sandbox/sandbox-local/src/profiles.ts
// 最终拼成: sandbox-exec -p "(version 1) (allow default) (deny file-write*) (allow file-write* (literal \"/dev/null\"))"
const forms = [
  '(version 1)', '(allow default)', '(deny file-write*)',
  `(allow file-write* (literal ${sbplString('/dev/null')}))`,
]
// workspace-write 时再追加: (allow file-write* (subpath "<workspaceRoot>") (subpath "<tmp>"))
```

含义：
- `(allow default)` 放行除文件写以外的一切。
- `(deny file-write*)` 默认拒绝所有写。
- `/dev/null` 是 shell 必需的 sink 故例外。
- `workspace-write` 模式按 `writableRoots` 给 workspace root 与平台 tmp 开 `(subpath …)` 写豁免。
- **网络和进程可见性不在此词表内**（文档明确）。

唯一候选的 Seatbelt 探测函数 `defaultProbeSeatbelt` 存在但产品链不调它（链只有一个候选时跳过探测）：

```ts
// packages/sandbox/sandbox-local/src/index.ts
function defaultProbeSeatbelt(seatbeltExec: string, timeoutMs: number): boolean {
  const probe = spawnSync(seatbeltExec, [...seatbeltProfileArgs({ mode: 'read-only', workspaceRoot: '/' }), '--', 'true'], {...})
  return probe.status === 0  // sandbox_init 拒绝 profile 时非零
}
```

### 第 5 层：内核/外部 —— `sandbox-exec` + Seatbelt

`sandbox-exec` 是 macOS 内置 CLI（Apple 标记 deprecated 但每台 mac 都自带）。它把 SBPL profile 编译成 Seatbelt 策略并 exec 目标命令，内核强制执行写拒绝。**沙箱到此为止，不再有 JS 层。** 若 `sandbox-exec` 消失，探测会 fail-closed（`unusable` → 抛 `SandboxUnavailableError`）。

## 4. 相互调用关系

```
SandboxBashExecutor.execute()
        │
        ├── ctx.sandboxPolicy.resolve({session})  → SandboxExecutionPolicy
        │
        └── ctx.sandbox.confine(['bash','-c',cmd], policy)
                    │  (抽象 SandboxProvider)
                    ▼
              LocalSandboxProvider.confine()
                    │
                    ├── selectRunner() → PLATFORM_CHAINS['darwin'] = 'seatbelt'
                    │
                    ├── runnerArgv('seatbelt')
                    │     └── seatbeltProfileArgs(policy)  → SBPL 字符串
                    │
                    └── 返回 ConfinedArgv
                          argv: ['sandbox-exec', '-p', '<SBPL>', '--', 'bash', '-c', cmd]
                          enforcement: 'full'
                          denialSignatures: ['operation not permitted']
                          runnerFailureRules: [{fatalSignatures: ['sandbox-exec: ']}]
        │
        ▼
   spawn(confined.argv) → 内核 Seatbelt 强制
        │
        └── onProcessDone 分类：denial / runner-failure / 普通失败
```

## 5. 完整调用链（mac 上启动一个受沙箱保护的 bash 命令）

1. 工具层调 `SandboxBashExecutor.execute(spec)`。
2. `resolve(spec)` 盖章 `sandboxPolicy: ctx.sandboxPolicy.resolve()`。
3. 非 `danger-full-access` 时进 `executeArgv` 回调，调 `ctx.sandbox.confine(['bash','-c',cmd], policy)`。
4. `SandboxProvider.confine`（抽象）落到 `LocalSandboxProvider.confine`。
5. `selectRunner` 走 `PLATFORM_CHAINS['darwin']` → `{runner:'seatbelt', enforcement:'full'}`。
6. `runnerArgv('seatbelt')` → `['sandbox-exec', ...seatbeltProfileArgs(policy)]`。
7. 拼成 `argv = ['sandbox-exec', '-p', '<SBPL>', '--', 'bash', '-c', cmd]` 返回。
8. 消费者用此 argv spawn → 内核 Seatbelt 强制；失败用 `operation not permitted` / `sandbox-exec: ` 分类。

## 6. 关键代码路径索引

| 关注点 | 路径 |
|---|---|
| 沙箱抽象契约（confine / 模式 / 错误） | `packages/sandbox/sandbox/src/index.ts` |
| 策略解析（默认 mode / session 覆盖 / root） | `packages/sandbox/sandbox-policy/src/index.ts` |
| 本地 provider：平台链选择 + seatbelt argv 组装 | `packages/sandbox/sandbox-local/src/index.ts` |
| Seatbelt SBPL profile 构造 | `packages/sandbox/sandbox-local/src/profiles.ts` |
| bash 消费者 | `packages/shell/bash-sandbox/src/index.ts` |
| fs 进程内围栏 | `packages/fs/fs-sandbox/src/index.ts` |
| 沙箱装配（bundle 挂载） | `packages/bundle/base/cordis.patch.yml` |
| 子系统文档 | `docs/subsystems/sandbox.md` |
| 决策记录 | `.agents/notes/implemented/feature/2026-07-06-sandbox.md` |

## 7. 桌面端是否启用

`base` bundle 挂载了完整三件套：`dsh-sandbox-local` + `dsh-sandbox-policy` + `dsh-bash-sandbox`。mac 桌面端走本地 shell，故经此 `sandbox-local` 的 seatbelt 路径。

release notes 中"macOS/Linux 从图形入口启动桌面端时缺少登录 shell 环境"这一修复，**与沙箱机制本身无关**——它解决的是桌面 GUI 启动时 shell 未加载用户登录环境（PATH/代理等）。沙箱策略依赖的 `workspaceRoot` 来自 session cwd，受影响的是工具可见的环境变量而非沙箱 profile。沙箱在这之前就是工作的。

## 8. 与上一 release 的关系

`dsh-v0.2.0-rc.2` 的沙箱架构（seatbelt 路径）与上一版 `rc.1` 相比**未变化**——本次 release 的改动集中在桌面端 UI、PowerShell、模型选择器、异步问答模式等，沙箱 capability seam 不在改动文件清单内。
