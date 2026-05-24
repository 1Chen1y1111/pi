# 内置工具体系设计

本文说明 Pi 的内置工具如何定义、注册、执行和渲染。阅读本文不需要先读其他架构文档。

## 设计目标

内置工具提供 coding agent 的最小能力面：

- 读取文件和图片。
- 执行 shell 命令。
- 创建、覆盖、局部编辑文件。
- 搜索和列举项目内容。

工具需要同时服务三个对象：

- LLM: 需要 name、description、TypeBox 参数 schema。
- Agent runtime: 需要 execute、abort、stream update、executionMode。
- UI/导出: 需要 call/result 渲染逻辑。

## 工具列表

内置工具在 `packages/coding-agent/src/core/tools`。

| 工具 | 文件 | 作用 |
| --- | --- | --- |
| `read` | `read.ts` | 读取文本和图片，支持 offset/limit 和输出截断 |
| `bash` | `bash.ts` | 本地 shell 执行，支持流式输出、timeout、abort、命令前缀和 shell 配置 |
| `edit` | `edit.ts` | 基于唯一旧文本的局部替换，生成 diff/patch |
| `write` | `write.ts` | 创建或覆盖文件，自动创建父目录 |
| `grep` | `grep.ts` | 文本搜索 |
| `find` | `find.ts` | 文件查找 |
| `ls` | `ls.ts` | 目录列举 |

`tools/index.ts` 统一导出：

- tool definition factory。
- executable tool factory。
- `ToolName`
- `allToolNames`
- `createCodingTools`
- `createReadOnlyTools`
- `createAllTools`

默认 coding tools 是 `read`、`bash`、`edit`、`write`。

## Tool Definition

每个工具通常暴露两层 factory：

```text
createXToolDefinition(cwd, options)
  |
  +--> ToolDefinition
        |
        +--> name / label / description
        +--> promptSnippet / promptGuidelines
        +--> TypeBox parameters
        +--> execute(...)
        +--> render call/result

createXTool(cwd, options)
  |
  +--> AgentTool
```

`ToolDefinition` 是 coding-agent 层的 richer abstraction，后续通过 wrapper 转成 Agent Core 可执行的 `AgentTool`。

## 执行生命周期

工具调用由 Agent Core 执行：

```text
assistant toolCall
  |
  +--> tool_execution_start
  +--> prepareArguments
  +--> validateToolArguments(TypeBox)
  +--> beforeToolCall hook
  +--> tool.execute(toolCallId, args, signal, onUpdate)
  |     |
  |     +--> optional tool_execution_update
  |
  +--> afterToolCall hook
  +--> tool_execution_end
  +--> toolResult message_start/end
```

`beforeToolCall` 和 `afterToolCall` 由 `AgentSession` 安装，用来桥接扩展事件 `tool_call` 和 `tool_result`。

## 文件写入串行化

`edit` 和 `write` 会修改文件。它们通过 `withFileMutationQueue` 对同一绝对路径串行化写操作，避免同一轮并行工具调用同时写同一个文件。

这个设计配合 Agent Core 的默认 parallel tool execution：

- 不同文件可以并行写。
- 同一文件按队列顺序写。
- abort 时不提前释放队列，避免后台 fs 操作完成后覆盖后续写入。

## Read Tool

`read` 支持：

- 文本文件读取。
- 图片读取，支持 jpg、png、gif、webp。
- 图片自动 resize。
- `offset` 和 `limit` 按行读取。
- 最大行数和最大字节数截断。
- 当前模型不支持图片时给出提示并省略图片。

`read` 还会对 Pi 自身 docs、resource file、skill file 做 compact render 分类，交互 UI 可以默认折叠这些高频读取内容。

## Bash Tool

`bash` 支持：

- 通过本地 shell 执行命令。
- stdout/stderr 合流流式输出。
- timeout。
- abort 时杀掉进程树。
- shell path 配置。
- command prefix。
- spawnHook 改写 command/cwd/env。
- 输出截断，并把完整输出写到临时文件。

底层 `createLocalBashOperations` 可被扩展复用，用于 wrap 或 rewrite user bash，同时保留 Pi 标准 shell 行为。

## Edit Tool

`edit` 的输入是：

- `path`
- `edits[]`: 多个 `{ oldText, newText }`

约束：

- `oldText` 必须在原文件中唯一。
- 多个 edit 基于原始文件匹配，不是增量匹配。
- 不允许重叠或嵌套 edit。
- 邻近改动应合并成一个 edit。

它会：

- 读取原文件。
- 处理 BOM 和行尾。
- 计算 diff/patch。
- 写回文件。
- 返回 display diff 和 unified patch。

## Write Tool

`write` 用于新文件或完整重写。

行为：

- 自动创建父目录。
- 覆盖已有文件。
- 支持语法高亮预览。
- 出错时在 result render 中展示错误。

系统提示词会引导模型：局部修改优先使用 `edit`，新建或完整重写才使用 `write`。

## 搜索类工具

`grep`、`find`、`ls` 是只读工具组合的一部分。它们用于减少模型直接用 shell 拼搜索命令的需求，并让结果渲染和截断保持一致。

默认启用工具不包含这三个；read-only tool set 会启用它们。

## 扩展工具

扩展可注册自定义工具。它们进入同一套 tool registry：

- 能被 `AgentSession.setActiveToolsByName` 启停。
- 能被 system prompt 汇总。
- 能被 Agent Core 校验和执行。
- 能参与扩展 `tool_call` 和 `tool_result` hook。
- 可提供自定义 TUI/HTML render。

## 新增工具流程

1. 在 `packages/coding-agent/src/core/tools` 添加 tool definition 和 executable tool factory。
2. 在 `tools/index.ts` 导出类型和 factory。
3. 扩展 `ToolName`、`allToolNames`、`createToolDefinition`、`createTool`。
4. 判断是否加入 `createCodingTools` 或 `createReadOnlyTools`。
5. 补充 promptSnippet、promptGuidelines 和 render 逻辑。
6. 针对 schema、执行、错误和并发写入行为补测试。

## 关键源码

- 工具统一出口：`packages/coding-agent/src/core/tools/index.ts`
- read：`packages/coding-agent/src/core/tools/read.ts`
- bash：`packages/coding-agent/src/core/tools/bash.ts`
- edit：`packages/coding-agent/src/core/tools/edit.ts`
- write：`packages/coding-agent/src/core/tools/write.ts`
- mutation queue：`packages/coding-agent/src/core/tools/file-mutation-queue.ts`
- tool wrapper：`packages/coding-agent/src/core/tools/tool-definition-wrapper.ts`
- Agent tool execution：`packages/agent/src/agent-loop.ts`
