# Claude Code 源码深度解读

*基于 2026-03-31 泄漏的 Anthropic Claude Code CLI 源码*

## 这份代码是怎么出现的

2026 年 3 月 31 日，安全研究员 Chaofan Shou（@Fried_rice）发现 Anthropic 在发布 Claude Code npm 包时，意外将一个 `.map` 文件暴露在公网可访问的 R2 存储桶中。

Source Map 是构建工具生成的调试文件，包含从编译产物到原始 TypeScript 源代码的完整映射——任何人都可以通过它还原出 Claude Code 的全部未混淆源码。

事件迅速扩散，warren-wupeng/claude-code 仓库将这批源码公开存档。

规模有多大？512,000 行 TypeScript，1,913 个源文件，41 个工具实现，约 140 个终端 UI 组件。这不是一个玩具项目，是经过数十人多年迭代的生产级系统。

## 五层架构

Claude Code 的代码组织是清晰的五层设计：

**入口层**：`cli.tsx` · `init.ts` · `sdk/` · `mcp.ts`

**核心引擎层**：`QueryEngine.ts` · `query.ts` · `context.ts`

**工具与命令层**：`tools/`（41个）· `commands/`（约50个）· `Tool.ts`

**服务层**：`compact/` · `permissions/` · `mcp/` · `bridge/` · `tasks/`

**UI 层**：`components/`（约140个）· `ink/` · `screens/`

代码量最大的是 UI 组件（9.6 MB），因为终端界面的细节非常多：消息气泡、权限弹框、进度条、滚动区域……每一个都需要专门处理 ANSI 转义符和终端尺寸变化。

运行时选择了 **Bun** 而非 Node.js，原因很直接：CLI 工具的启动延迟是第一印象，Bun 比 Node.js 快 3-5 倍，且内置了 `bun:bundle` API，用于编译时特性门控（后面详说）。

## 核心引擎：QueryEngine

`src/QueryEngine.ts` 是 Claude Code 的心脏，46,000 行，单文件。每个对话 session 对应一个实例，持有完整对话历史、文件内容缓存、累计 token 用量。

对外只暴露一个方法：

```typescript
async *submitMessage(input: string): AsyncGenerator<SDKMessage>
```

返回值是**异步生成器**——不是 Promise，不是回调，而是可以被 `for await...of` 消费的流。UI 层、SDK 消费者、测试框架都通过这个统一接口接收实时消息。

每次调用经历六个阶段：

**① 预处理**：判断是否是 `/slash-command` 或 `!bash-shortcut`，若是则绕过 LLM。

**② 上下文组装**：用 `Promise.all` 并发获取 git 状态、CLAUDE.md 内容、工具 prompt。不串行等待。

**③ 记忆注入**：若 auto-memory 开启，读取 MEMORY.md 追加到 system prompt。

**④ API 调用**：流式请求 Anthropic API，逐 chunk yield。

**⑤ 工具调用循环**：收到 `tool_use` 后循环：权限检查 → 执行 → 追加结果 → 继续，直到模型停止发起工具调用。

**⑥ 压缩检查**：检查累计 token 是否超阈值，触发自动压缩。

### 四种压缩方案

长对话的核心挑战是上下文窗口溢出。Claude Code 实现了四种方案：

- `autoCompact`：token 超阈值自动触发，用 API 对旧轮次生成摘要替换原始消息
- `compact`：用户主动执行 `/compact`，完整摘要保留最近 N 轮原文
- `microCompact`：轻量逐轮摘要，压缩比低但信息损失小
- `snipCompact`：最精妙的方案——不真正删除历史，而是插入 `SnipBoundary` 标记将旧内容移入侧存储，模型需要时通过工具调用"换页"恢复

最后一个像操作系统的虚拟内存，但在语义层面实现。

### CLAUDE.md 记忆系统

Claude Code 的"记忆"不依赖数据库，用的是 **Markdown 文件即配置**。

`context.ts` 组装 system prompt 时，从当前目录向上遍历所有父目录，收集每一级的 `CLAUDE.md`：全局指令（`~/`）→ 团队级指令（`~/projects/`）→ 项目级指令（`~/projects/my-app/`）。层级越深优先级越高。

另有两个特殊文件：`~/.claude/MEMORY.md` 是 auto-memory 写入的持久状态，上限 200 行；`.claude/settings.json` 是权限配置，不注入 prompt。

这个设计的精妙在于：指令是人类可读的，可以 git 管理，可以 code review，完全透明。

## 工具系统

每个工具都实现 `Tool<Input, Output>` 接口：

```typescript
interface Tool<Input, Output> {
  name: string
  description: string
  inputSchema: ZodSchema<Input>
  call(input, context): Promise<Output>
  checkPermissions(input, context)
  isReadOnly(): boolean
  isConcurrencySafe(): boolean
  renderToolUseMessage(input)
  renderToolResultMessage(output)
}
```

工具作者只需覆盖与默认行为不同的部分，`buildTool()` 工厂函数处理剩余默认值，同时在类型层面保留字面量类型精度。

### 权限双门控

每次工具调用通过两道独立的权限关卡：

**第一道** 是工具自身的 `checkPermissions()`，写领域相关的危险判断。BashTool 检查 `rm -rf`、`curl | bash` 等危险模式；WriteFileTool 检查路径是否在项目根目录之外。

**第二道** 是全局 `permissions.ts`，决定策略：全自动放行（CI 场景）、计划确认（一次性整体授权）、ML 分类（低风险自动放行）、逐步确认（默认模式）。

两道门相互独立，工具可以比全局策略更严格，但全局策略是最终裁决者。Pre-tool hooks（shell 脚本）可以在第二道门前插入企业级自定义逻辑。

### 41 个内置工具

- **文件只读**：Read · Glob · Grep · LS · NotebookRead
- **文件写入**：Write · Edit · NotebookEdit · MultiEdit
- **代码执行**：Bash · BashBackground · Computer
- **Agent 协作**：Agent · TeamCreate · TeamDelete · SendMessage · TaskCreate · TaskUpdate · TaskList · TaskOutput · TaskStop
- **MCP 集成**：MCPTool（动态注册，数量不固定）
- **Web**：WebFetch · WebSearch
- **其他**：TodoWrite · Skill · CronCreate/Delete/List · EnterWorktree · ExitWorktree · EnterPlanMode · ExitPlanMode

值得注意：`AgentTool` 会启动一个子 `QueryEngine`，实现 Agent 递归嵌套，每层有独立权限上下文。`BashBackground` 立即返回 task ID，结果通过 `TaskOutput` 异步获取——这是一个有意识的同步/异步分工。

## 多 Agent 协作：Swarm

Claude Code 的多 Agent 不是进程内协程，而是**每个 Agent 一个独立的 Claude Code 进程**。

这是刻意的设计：进程隔离给每个 Agent 独立的内存空间、权限上下文、崩溃边界。一个 Agent 挂掉不影响整个团队。

Team Leader 是唯一持有终端 UI 的进程，Worker 以 headless 模式运行（`CLAUDE_CODE_COORDINATOR_MODE=1`）。所有 Worker 的权限请求通过 `leaderPermissionBridge.ts` 上报给 Leader，集中在一个界面弹出确认。

### 文件系统 Mailbox

进程间通信靠**文件系统 Mailbox**，没有共享内存，没有消息中间件：

```
~/.claude/teams/my-team/
  config.json          # 团队成员列表
  mailbox/
    leader-inbox/      # Worker → Leader
    worker-a-inbox/    # Leader → Worker-A
~/.claude/tasks/my-team/
  task-001.json        # 任务状态文件
```

每条消息是一个 JSON 文件，发送方写入，接收方轮询。`SendMessageTool` 本质就是向目标 Agent 的 inbox 写一个 JSON 文件。

优点：极度简单可调试，用 `ls` 就能看消息队列，用编辑器就能插入消息。代价是约 500ms 轮询延迟，对 Agent 协作场景完全够用。

## 工程化亮点

### 编译时 Feature Flag

```typescript
import { feature } from 'bun:bundle'

const coordinatorModule = feature('COORDINATOR_MODE')
  ? require('./coordinatorMode.js')
  : null
```

不活跃的分支在构建阶段被 dead code elimination 完全删除——不是运行时跳过，是根本不存在于产物中。目前有 8 个门控：`COORDINATOR_MODE`、`PROACTIVE`、`KAIROS`、`BRIDGE_MODE`、`DAEMON`、`VOICE_MODE`、`AGENT_TRIGGERS`、`MONITOR_TOOL`。

Ant 内部版本额外有 `process.env.USER_TYPE === 'ant'` 门控，公开版本中对应代码完全不存在，无法逆向。

### 启动速度优化

三个层次：

**快速退出路径**：`--version` 在加载任何业务模块之前就返回，零模块加载延迟。

**模块懒加载**：OpenTelemetry（~400 KB）、gRPC（~700 KB）全部 `dynamic import()`，只在需要时触发。

**I/O 与模块加载并发**：MDM 策略读取和 macOS Keychain 凭证预取在模块 `import` 阶段就已异步启动，等 CLI 需要时 I/O 早已完成，延迟被完全隐藏。

### TypeScript 类型系统

三个值得记录的模式：

**`DeepImmutable<T>`**：权限状态被递归包裹为 `readonly`，编译期阻止任何工具意外修改共享权限上下文——不是文档约定，是类型系统强制执行。

**`lazySchema()`**：将 Zod schema 构造推迟到首次调用时，规避工具间的循环 import 依赖。

**`BuiltTool<D>`**：`buildTool()` 的返回类型用条件类型保留字面量精度，`isConcurrencySafe: true` 在类型层面是 `true` 而非宽泛的 `boolean`。

## 最后的判断

读完这份代码，一个问题的答案很清晰：**生产级 AI Agent 系统的复杂度应该在哪里？**

答案是：在基础设施层（权限系统、上下文管理、工具接口、进程协调），而不是业务逻辑层（每个工具的具体实现）。BashTool 的代码很简单，因为 QueryEngine 帮它处理了流式输出、错误重试、权限审计。WriteFileTool 很简单，因为 `ToolPermissionContext` 帮它处理了状态不变性和并发安全。

**不要急着堆功能，先把基础设施做扎实。** 一个干净的工具接口、一套可组合的权限策略、一个清晰的上下文压缩机制——这些比"再加 10 个工具"更有长期价值。

最后值得一提：这份代码是意外泄漏的，但从工程质量看，它完全经得起公开审阅。命名清晰、关注点分离、类型严格——这不是应付检查的代码，是真正写给人看、写给时间看的代码。
