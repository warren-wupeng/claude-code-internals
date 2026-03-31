# Claude Code 源码深度解读

> **作者：** Kira Chen
> **日期：** 2026-03-31
> **基于：** [warren-wupeng/claude-code](https://github.com/warren-wupeng/claude-code) — Anthropic Claude Code CLI 泄漏源码存档

---

## 第一章：背景与概述

### 1.1 泄漏事件始末

2026 年 3 月 31 日，安全研究员 Chaofan Shou（@Fried_rice）发现 Anthropic 在发布 Claude Code npm 包时，将一个 `.map`（Source Map）文件意外暴露在公网可访问的 R2 存储桶中。Source Map 是 JavaScript 构建工具生成的调试文件，其中包含从编译产物到原始 TypeScript 源代码的完整映射——这意味着任何人都可以通过这个文件还原出 Claude Code 的全部未混淆源码。

该事件迅速在开发者社区扩散。warren-wupeng/claude-code 仓库是对这批源码的公开存档，附带了一个由仓库作者额外添加的 MCP 服务器（`mcp-server/`），使任何 MCP 兼容工具（VS Code Copilot、Claude Desktop 等）都可以将此源码作为知识库进行探索。

### 1.2 代码规模

| 维度 | 数字 |
|------|------|
| 总行数 | ~512,000 行 |
| 源文件数 | ~1,913 个 |
| 主语言 | TypeScript（30.3 MB） |
| 工具实现 | 41 个 |
| 斜杠命令 | ~50 个 |
| UI 组件 | ~140 个 Ink/React 组件 |

这不是一个玩具项目。从代码量和工程复杂度看，这是一个经过数十人多年迭代、投入大量工程资源的生产级系统。

### 1.3 本文范围

本文聚焦五个核心主题：

1. **宏观架构**：系统是如何组织的？
2. **QueryEngine**：LLM 调用的核心循环是什么？
3. **工具系统**：41 个工具如何设计和管理？
4. **多 Agent 协作**：Swarm 如何实现进程间协调？
5. **工程化亮点**：有哪些值得借鉴的技术决策？

---

## 第二章：宏观架构

### 2.1 五层架构模型

Claude Code 的代码组织遵循清晰的分层设计：

```
┌─────────────────────────────────────────────────────┐
│  入口层 (Entrypoints)                                │
│  cli.tsx · init.ts · sdk/ · mcp.ts                  │
├─────────────────────────────────────────────────────┤
│  核心引擎层 (Core Engine)                            │
│  QueryEngine.ts · query.ts · context.ts             │
├─────────────────────────────────────────────────────┤
│  工具与命令层 (Tools & Commands)                     │
│  tools/ (41个) · commands/ (~50个) · Tool.ts        │
├─────────────────────────────────────────────────────┤
│  服务层 (Services)                                   │
│  compact/ · permissions/ · mcp/ · bridge/ · tasks/  │
├─────────────────────────────────────────────────────┤
│  UI 层 (Terminal UI)                                 │
│  components/ (~140个) · ink/ · screens/             │
└─────────────────────────────────────────────────────┘
```

### 2.2 目录结构与职责

| 目录 | 大小 | 核心职责 |
|------|------|---------|
| `src/components/` | 9.6 MB | Ink/React 终端 UI 组件，含消息气泡、权限弹框、进度条等 |
| `src/utils/` | 6.6 MB | 共享工具库：权限、Shell 封装、插件系统、Swarm 支持 |
| `src/tools/` | 2.7 MB | 41 个 Agent 工具实现（Bash、Read、Write、Grep 等）|
| `src/commands/` | 2.5 MB | ~50 个斜杠命令（`/compact`、`/memory`、`/doctor` 等）|
| `src/services/` | 1.8 MB | 外部集成：API 客户端、MCP、OAuth、LSP、Analytics |
| `src/hooks/` | 1.2 MB | React hooks：状态、权限、键绑定 |
| `src/ink/` | 1.0 MB | 定制 Ink 渲染器（双向文字、滚动框、备用屏幕）|
| `src/screens/` | 1.0 MB | 全屏 TUI 视图：REPL、Doctor、Resume、Compact |
| `src/cli/` | 0.5 MB | CLI 引导、传输层（SSE/WebSocket/Hybrid）|
| `src/bridge/` | 0.5 MB | IDE 桥接（VS Code、JetBrains）|
| `src/tasks/` | 0.3 MB | 后台任务管理 |
| `src/entrypoints/` | 0.2 MB | 初始化逻辑、Agent SDK、MCP 入口 |
| `src/skills/` | 0.15 MB | Skill 系统与内置 Skill |
| `src/native-ts/` | 0.13 MB | 原生 TS 实现：文件索引、Yoga 布局、彩色 diff |

### 2.3 主要数据流

一次完整的用户交互，数据流经以下路径：

```
用户键入消息
    │
    ▼
screens/REPL.tsx（UI 捕获输入）
    │
    ▼
QueryEngine.submitMessage()（消息入队）
    │
    ├── context.ts（并行组装 system prompt）
    │     ├── git status / log
    │     ├── CLAUDE.md 文件（所有父目录）
    │     └── MEMORY.md（持久记忆）
    │
    ├── query.ts（调用 Anthropic API，流式）
    │
    └── 工具调用循环
          ├── Tool.checkPermissions()
          ├── permissions.ts（全局权限决策）
          ├── Pre-tool hooks
          ├── Tool.call()（执行工具）
          ├── Post-tool hooks
          └── 追加 tool_result，继续循环
    │
    ▼
SDKMessage 流（yield 给 UI / SDK 消费者）
    │
    ▼
components/（渲染到终端）
```

### 2.4 运行时环境

Claude Code 使用 **Bun** 作为运行时（而非 Node.js），这是一个关键的技术选型：

- Bun 的启动速度比 Node.js 快 3-5 倍，对于 CLI 工具的感知延迟至关重要
- Bun 内置 `bun:bundle` API，使得 feature flag 的**编译时死码消除**成为可能（见第六章）
- Bun 内置测试运行器，与外部测试框架无缝集成

---

## 第三章：核心引擎——QueryEngine

### 3.1 它是什么

`src/QueryEngine.ts` 是整个 Claude Code 的心脏，46,000 行，单文件。每个对话 session 对应一个 `QueryEngine` 实例，它持有：

- `mutableMessages: Message[]` — 完整对话历史
- `readFileState: FileStateCache` — LRU 缓存，记录已读文件内容（避免重复注入）
- `totalUsage: NonNullableUsage` — 累计 token 用量，用于成本计算和上下文压缩触发

对外暴露的核心方法只有一个：

```typescript
async *submitMessage(input: string): AsyncGenerator<SDKMessage>
```

返回值是一个**异步生成器**——不是 Promise，不是回调，而是一个可以被 `for await...of` 消费的流。UI 层、SDK 外部消费者、测试框架都通过这个统一接口接收实时消息。

### 3.2 submitMessage() 主循环拆解

`submitMessage()` 的执行分为六个有序阶段：

**① 预处理**：判断输入是否是 `/slash-command` 或 `!bash-shortcut`，若是则走命令分发路径，绕过 LLM 调用。

**② 上下文组装**：并行调用 `fetchSystemPromptParts()`，同时获取 git 状态、CLAUDE.md 内容、工具 prompt、用户画像。全部用 `Promise.all` 并发，不串行等待。

**③ 记忆注入**：读取 MEMORY.md（若 auto-memory 开启），追加到 system prompt 末尾。

**④ API 调用**：调用 `query()` 向 Anthropic API 发起流式请求，逐 chunk yield 给调用方。

**⑤ 工具调用循环**：收到 `tool_use` block 后，进入循环：权限检查 → 执行 → 追加 `tool_result` → 继续请求，直到模型不再发起工具调用为止。

**⑥ 压缩检查**：每轮结束后检查累计 token 是否超过阈值，若超过触发 `autoCompact()`。

### 3.3 上下文压缩策略

长对话的最大挑战是上下文窗口溢出。Claude Code 实现了四种压缩方案，按场景分别使用：

| 方案 | 文件 | 触发条件 | 机制 |
|------|------|---------|------|
| `autoCompact` | `services/compact/autoCompact.ts` | token 超阈值，自动 | 用 API 对旧轮次生成摘要，替换原始消息 |
| `compact` | `services/compact/compact.ts` | `/compact` 命令，用户主动 | 完整摘要，保留最近 N 轮原文 |
| `microCompact` | `services/compact/microCompact.ts` | 每轮可选 | 轻量逐轮摘要，压缩比低但信息损失小 |
| `snipCompact` | `services/compact/snipCompact.ts` | feature-gated | 在历史中打"Snip"标记，按需恢复原始片段 |

其中 `snipCompact` 最有意思：它不是真正删除历史，而是在对话流中插入一个 `SnipBoundary` 标记，将旧内容移入侧存储。当模型需要某段历史时，可以通过工具调用"恢复"这段内容——类似操作系统的虚拟内存换页，但语义层面的。

### 3.4 CLAUDE.md 记忆系统

Claude Code 的"记忆"不依赖数据库，而是**Markdown 文件即配置**的设计。

`context.ts` 在组装 system prompt 时，会从当前目录向上遍历所有父目录，收集每一级的 `CLAUDE.md` 文件：

```
/home/user/                    ← 全局指令（工具偏好、代码风格）
/home/user/projects/           ← 团队级指令
/home/user/projects/my-app/    ← 项目级指令（通常 git 管理）
```

层级越深，优先级越高，子目录的指令可以覆盖父目录。此外还有两个特殊文件：

- **`~/.claude/MEMORY.md`**：auto-memory 功能写入的持久记忆，上限 200 行 / 25 KB，超出则截断
- **`.claude/settings.json`**：项目级权限和工具配置，不注入 prompt，由权限系统读取

这个设计的精妙之处在于：指令是**人类可读可编辑的**，可以 git 管理，可以 code review，完全透明。

---

## 第四章：工具系统

### 4.1 Tool 接口设计

每个工具都实现 `src/Tool.ts` 中定义的 `Tool<Input, Output>` 接口，核心成员如下：

```typescript
interface Tool<Input, Output> {
  name: string                          // 工具名，模型调用时使用
  description: string                   // 告诉模型这个工具做什么
  inputSchema: ZodSchema<Input>         // 入参校验（Zod）
  call(input, context): Promise<Output> // 实际执行逻辑
  checkPermissions(input, context)      // 工具级权限检查
  isReadOnly(): boolean                 // 只读工具可跳过部分权限流程
  isConcurrencySafe(): boolean          // 是否可与其他工具并发执行
  renderToolUseMessage(input)           // 在 UI 中如何展示"模型在调用这个工具"
  renderToolResultMessage(output)       // 在 UI 中如何展示工具返回结果
}
```

大多数成员有合理默认值，工具作者只需覆盖与默认行为不同的部分。这通过 `buildTool()` 工厂函数实现——它在运行时做 `{...TOOL_DEFAULTS, ...def}` 展开，同时在类型层面用条件类型推导保留字面量类型，避免工具作者的 TypeScript 类型丢失精度。

### 4.2 权限双门控机制

每次工具调用都要通过**两道独立的权限关卡**：

**第一道：工具自身的 `checkPermissions()`**
工具实现者在这里写领域相关的危险判断。例如 BashTool 会检查命令是否包含 `rm -rf`、`curl | bash` 等危险模式；WriteFileTool 会检查路径是否在项目根目录之外。这道门只有工具自己最清楚该怎么把守。

**第二道：全局 `permissions.ts`**
不管工具自身怎么判断，最终还要经过全局策略：

| 模式 | 行为 |
|------|------|
| `bypassPermissions` | 全部自动允许（CI/自动化场景）|
| `plan` | 展示操作计划，一次性整体授权 |
| `auto` | ML 分类器判断风险，低风险自动放行 |
| `default` | 逐操作弹出 UI 提示用户确认 |

两道门相互独立：工具可以比全局策略更严格，但全局策略永远是最终裁决者。此外 Pre-tool hooks（shell 脚本）可以在第二道门之前插入自定义逻辑，实现企业级策略扩展。

### 4.3 41 个内置工具分类

```
文件操作（只读）
  Read · Glob · Grep · LS · NotebookRead

文件操作（写入）
  Write · Edit · NotebookEdit · MultiEdit

代码执行
  Bash · BashBackground · Computer（截图/点击）

Agent 协作
  Agent · TeamCreate · TeamDelete · SendMessage
  TaskCreate · TaskUpdate · TaskList · TaskOutput · TaskStop

MCP 与外部集成
  MCPTool（动态注册，每个 MCP server 暴露的工具）

Web 与搜索
  WebFetch · WebSearch

其他
  TodoWrite · ExitPlanMode · EnterPlanMode
  Skill · CronCreate · CronDelete · CronList
  EnterWorktree · ExitWorktree
```

几个值得关注的设计：

- **MCPTool 是动态的**：不是硬编码的，而是在 MCP server 连接后根据服务端声明动态注册，数量不固定
- **Agent 工具递归调用**：`AgentTool` 会启动一个子 QueryEngine，实现 Agent 嵌套，每层有独立的权限上下文
- **Bash vs BashBackground**：前者阻塞等待结果，后者立即返回 task ID，结果通过 `TaskOutput` 异步获取

---

## 第五章：多 Agent 协作——Swarm

### 5.1 进程隔离设计

Claude Code 的多 Agent 不是在同一个进程内跑多个协程，而是**每个 Agent 是一个独立的 Claude Code 进程**。

这是一个刻意的架构选择：进程隔离意味着每个 Agent 有独立的内存空间、独立的权限上下文、独立的崩溃边界。一个 Agent 挂掉不会拖垮整个团队。

Team Leader 是唯一持有终端 UI 的进程，Worker Agent 以 headless 模式运行（`CLAUDE_CODE_COORDINATOR_MODE=1` 环境变量标识）。用户始终只看到一个界面，所有 Worker 的权限请求都会通过 `leaderPermissionBridge.ts` 上报给 Leader，在 Leader 的 UI 中集中弹出确认。

后端支持三种运行模式：
- **iTerm2 backend**：每个 Agent 开一个新 tab（macOS 专属）
- **tmux backend**：每个 Agent 开一个新 pane
- **InProcess backend**：调试用，伪进程隔离

### 5.2 Mailbox 通信机制

进程间没有共享内存，通信全靠**文件系统 Mailbox**。

团队创建时，会在 `~/.claude/teams/{team-name}/` 下建立配置文件和 mailbox 目录：

```
~/.claude/teams/my-team/
  config.json          ← 团队成员列表（name、agentId、agentType）
  mailbox/
    leader-inbox/      ← Worker → Leader 的消息队列
    worker-a-inbox/    ← Leader → Worker-A 的消息队列
    worker-b-inbox/
~/.claude/tasks/my-team/
  task-001.json        ← 任务状态文件（pending/running/completed）
```

每条消息是一个 JSON 文件，发送方写入，接收方轮询读取后标记已读。`SendMessageTool` 本质就是向目标 Agent 的 inbox 目录写一个 JSON 文件。

这个设计的好处是极度简单、可调试——用 `ls` 就能看到消息队列，用文本编辑器就能插入或修改消息，不依赖任何消息中间件。代价是轮询延迟（约 500ms），对于 Agent 协作场景完全够用。

### 5.3 任务状态机

任务（Task）是 Swarm 的调度单元。`src/Task.ts` 定义了两个核心枚举：

**TaskType**（任务类型）：
- `local_bash` — 本地 shell 命令
- `local_agent` — 本地 Agent 子进程
- `remote_agent` — 远程 Agent（预留）
- `in_process_teammate` — 进程内伪 Agent（调试）
- `local_workflow` — 本地工作流
- `monitor_mcp` — MCP 监控任务
- `dream` — 推测性/实验性任务类型

**TaskStatus**（状态流转）：
```
pending → running → completed
                 ↘ failed
                 ↘ killed
```

`isTerminalTaskStatus()` 是一个类型守卫函数，判断状态是否已终止。任务 ID 采用类型前缀 + 8 位随机字母数字的格式（如 `b3f7a9kx`），在日志和 UI 中可通过前缀快速识别任务类型。每个任务的完整状态持久化在 `~/.claude/tasks/{team-name}/task-{id}.json`，重启后可恢复。

---

## 第六章：工程化亮点

### 6.1 Bun Feature Flag 作为构建系统

Claude Code 用 Bun 的 `bun:bundle` API 实现了**编译时特性门控**，而不是传统的运行时 `if (process.env.FEATURE_X)` 判断：

```typescript
import { feature } from 'bun:bundle'

const coordinatorModule = feature('COORDINATOR_MODE')
  ? require('./coordinatorMode.js')
  : null
```

当 `COORDINATOR_MODE` 为 false 时，`require('./coordinatorMode.js')` 这整个分支在构建阶段就被 dead code elimination 完全删除——不是运行时跳过，是根本不存在于产物中。

目前有 8 个这样的特性门控：

| 特性 | 功能 |
|------|------|
| `COORDINATOR_MODE` | 多 Agent Swarm 协调 |
| `PROACTIVE` | 主动建议模式 |
| `KAIROS` | 时机感知（实验性）|
| `BRIDGE_MODE` | IDE 桥接（VS Code / JetBrains）|
| `DAEMON` | 后台守护进程 |
| `VOICE_MODE` | 语音输入 |
| `AGENT_TRIGGERS` | 自动触发规则 |
| `MONITOR_TOOL` | 监控工具 |

Ant 内部版本额外通过 `process.env.USER_TYPE === 'ant'` 再加一层门控，公开发布版本中对应代码完全不存在。

### 6.2 启动速度优化

CLI 工具的启动延迟是用户体验的第一印象。Claude Code 用三个层次的技巧将感知延迟压到最低。

**① 快速退出路径（Fast Paths）**

`src/entrypoints/cli.tsx` 的 `main()` 函数开头是一组优先级最高的分支判断：

```typescript
if (args.version) { print(VERSION); process.exit(0) }  // 零模块加载
if (args['daemon-worker']) { startDaemonWorker(); return }
if (args['remote-control']) { startBridge(); return }
// ...其余 6 个快速路径
```

`--version` 在加载任何业务模块之前就返回，对用户来说是即时响应。

**② 模块懒加载**

OpenTelemetry（~400 KB）、gRPC（~700 KB）等重型模块全部用 `dynamic import()` 延迟加载，只在真正需要时才触发。主路径代码不等这些模块。

**③ I/O 与模块加载并发**

`src/main.tsx` 的顶层副作用中，MDM 策略读取和 macOS Keychain 凭证预取在模块 `import` 阶段就已经异步启动：

```typescript
startMdmRawRead()        // 模块加载时立即开始
startKeychainPrefetch()  // 与后续 import 并发执行
```

等到 CLI 真正需要这些数据时，I/O 早已在后台完成，延迟被隐藏。

### 6.3 TypeScript 类型系统亮点

**`DeepImmutable<T>` 权限状态不变性**

`ToolPermissionContext`（权限决策的输入）被包裹在自定义的 `DeepImmutable<T>` 工具类型中，递归地将所有嵌套属性变为 `readonly`：

```typescript
type DeepImmutable<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepImmutable<T[K]> : T[K]
}
```

这在编译期阻止了任何工具实现意外修改共享权限状态的可能——不是文档约定，是类型系统强制执行。

**`lazySchema()` 规避循环依赖**

工具的 Zod 输入 schema 有时需要引用其他工具的类型，直接 import 会产生循环依赖。解决方案是延迟构造：

```typescript
const inputSchema = lazySchema(() => z.object({
  filePath: z.string(),
  content: FileContentSchema(),  // 延迟求值
}))
```

`lazySchema()` 包装器将 schema 构造推迟到首次调用时，此时所有模块已加载完毕，循环引用问题自然消失。

**`BuiltTool<D>` 保留字面量类型**

`buildTool()` 的返回类型是 `BuiltTool<D>`，用条件类型在 `D` 和 `TOOL_DEFAULTS` 的展开上保留了每个字段的字面量类型，而不是退化为宽泛的 `boolean` 或 `string`。这使得 `isConcurrencySafe: true` 在类型层面是 `true` 而非 `boolean`，下游代码可以依赖这个精确信息。

---

## 第七章：总结与启示

### 7.1 对 AI Agent 架构设计的启示

读完这份代码，有几个判断值得记下来。

**异步生成器是 Agent 流式输出的正确抽象。** `AsyncGenerator<SDKMessage>` 比回调、比事件发射器、比 Promise 链都干净。调用方用 `for await...of` 消费，可以随时 `break` 中断，背压（backpressure）天然处理。zoey-ai 目前的 WebSocket 透传方案更底层，如果未来重构可以考虑在服务层引入这个模式。

**工具系统的关键是接口统一而非实现统一。** 41 个工具行为各异，但都实现同一个 `Tool<Input, Output>` 接口。QueryEngine 不需要知道 BashTool 怎么执行命令，只需要知道它有 `call()`、`checkPermissions()`、`renderToolUseMessage()`。这是真正的开闭原则——新增工具不修改引擎。

**Markdown 即配置的思路值得借鉴。** CLAUDE.md 系统让"给 AI 的指令"变成了可 git 管理、可 code review 的工件，而不是藏在数据库某个字段里的魔法字符串。对于 zoey-ai 这类长期个人 Agent，用户的偏好和背景也可以考虑以类似方式存储。

### 7.2 权限系统值得借鉴之处

Claude Code 的权限架构是这份代码里工程完成度最高的部分，有两点特别值得学习。

**分级授权比二元授权更实用。** 不是"允许/拒绝"两档，而是四档：全自动（`bypass`）、计划确认（`plan`）、ML 分类（`auto`）、逐步确认（`default`）。用户可以根据信任程度和场景灵活选择。在 CI 环境选 bypass，在生产操作选 default，平时用 auto——同一套代码服务完全不同的安全需求。

**双门控的层次分离是正确的。** 工具自身负责"这个操作是否危险"（领域知识），全局策略负责"危险操作该怎么处理"（策略知识）。两者互不侵犯。如果把策略逻辑写进工具里，每次策略调整都要改 41 个工具；如果把领域逻辑写进全局策略，全局策略会变成一个无法维护的大 switch。

**Pre/Post Hook 是正确的扩展点。** 企业客户有定制权限策略的需求，但不应该 fork 代码。Shell 脚本 hook 是正确答案——标准接口，任何语言实现，不耦合内部实现。这个思路和 git hooks 一脉相承。

### 7.3 最后的判断

这份代码回答了一个重要问题：**一个生产级 AI Agent 系统的复杂度应该在哪里？**

Claude Code 的答案是：复杂度应该在**基础设施层**（权限系统、上下文管理、工具接口、进程协调），而不在**业务逻辑层**（每个工具的具体实现）。BashTool 的实现很简单，因为 QueryEngine 帮它处理了流式输出、错误重试、权限审计；WriteFileTool 的实现也很简单，因为 ToolPermissionContext 帮它处理了状态不变性和并发安全。

这给我们自己构建 Agent 系统一个参照：**不要急着堆功能，先把基础设施做扎实**。一个干净的工具接口、一套可组合的权限策略、一个清晰的上下文压缩机制——这些比"再加 10 个工具"更有长期价值。

最后值得一提的是：这份代码是意外泄漏的，但从工程质量来看，它完全经得起公开审阅。命名清晰、关注点分离、类型严格、注释到位——这不是应付检查的代码，是真正写给人看、写给时间看的代码。

---

*全文完。共 7 章，约 7,000 字。*
*基于 warren-wupeng/claude-code 仓库，截至 2026-03-31。*
