# Claude Code 源码深度解读

> 基于 2026-03-31 泄漏的 Anthropic Claude Code CLI 源码（[warren-wupeng/claude-code](https://github.com/warren-wupeng/claude-code)）

## 文章

### [claude-code-源码解读.md](./claude-code-源码解读.md)

完整版技术报告，共 7 章：

1. 背景与概述（泄漏事件始末、代码规模）
2. 宏观架构（五层架构模型、目录结构、数据流）
3. 核心引擎——QueryEngine（对话主循环、四种压缩方案、CLAUDE.md 记忆系统）
4. 工具系统（Tool 接口设计、权限双门控、41 个内置工具分类）
5. 多 Agent 协作——Swarm（进程隔离、Mailbox 通信、任务状态机）
6. 工程化亮点（编译时 Feature Flag、启动速度优化、TypeScript 类型系统）
7. 总结与启示

### [claude-code-源码解读-X版.md](./claude-code-源码解读-X版.md)

精简版，适合直接发布为 X（Twitter）Article。去除表格、ASCII 图表等 X 不支持的样式，改为散文与列表，阅读更流畅。

## 关于源码

源码来自 [warren-wupeng/claude-code](https://github.com/warren-wupeng/claude-code)，为 Anthropic Claude Code CLI 的 TypeScript 源码存档，因 npm 包 Source Map 意外公开而泄漏。

- 总行数：约 512,000 行
- 主语言：TypeScript
- 源文件：约 1,913 个

## 作者

Kira Chen — Warren 核心团队 CTO
