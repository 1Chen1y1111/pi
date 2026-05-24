# 启动与运行模式设计

本文说明 `pi` 从命令行入口到不同运行模式的装配流程。阅读本文不需要先读总览文档。

## 范围

本专题覆盖：

- CLI 入口和参数解析。
- cwd-bound runtime 的创建。
- session 选择与 cwd 切换。
- interactive、print、JSON、RPC、SDK 五类使用方式。

不覆盖 Agent 内部循环和工具执行细节；这些在 [Agent 循环与会话封装](agent-loop.md) 和 [内置工具体系](tools.md) 中说明。

## 入口文件

`packages/coding-agent/src/cli.ts` 是发布后 `pi` 二进制的入口：

1. 设置 `process.title = APP_NAME`。
2. 设置 `PI_CODING_AGENT=true`。
3. 屏蔽 `process.emitWarning`。
4. 在 Provider SDK 发请求前配置全局 HTTP dispatcher。
5. 调用 `main(process.argv.slice(2))`。

真正的应用装配在 `packages/coding-agent/src/main.ts`。

## 启动阶段

`main(args)` 的启动流程：

```text
main(args)
  |
  +--> offline/self-update/package/config command handling
  +--> parseArgs(args)
  +--> resolveAppMode(parsed, stdin.isTTY)
  +--> runMigrations(process.cwd())
  +--> SettingsManager.create(startup cwd)
  +--> createSessionManager(...)
  +--> create runtime factory
  +--> createAgentSessionRuntime(...)
  +--> read piped stdin and @file args
  +--> initTheme(...)
  +--> dispatch mode
```

早期只创建 startup `SettingsManager`，原因是 `--session` 或 `--resume` 可能打开其他项目的 session。最终 settings、resources、models 必须在 session header 中的 cwd 上重建，避免错用当前 shell cwd 的项目配置。

## Session 选择

`createSessionManager` 处理 session 来源：

| 参数 | 行为 |
| --- | --- |
| `--no-session` | 创建 in-memory session，不持久化 |
| `--fork <path|id>` | 从已有 session fork 到当前 cwd |
| `--session <path|id>` | 打开指定 session；如果来自其他项目，交互模式下询问是否 fork |
| `--resume` | TUI 选择历史 session |
| `--continue` | 打开当前项目最近 session，没有则新建 |
| 默认 | 新建 session |

session 路径解析支持显式文件路径和 session id 前缀。先查当前项目，再查全局所有项目。

## Runtime Factory

`CreateAgentSessionRuntimeFactory` 是启动流程里的关键抽象。它闭包捕获 CLI 固定参数，例如扩展路径、skill 路径、prompt template 路径、theme 路径、authStorage 和未知扩展 flag。

每次创建 runtime 时，它执行：

1. `createAgentSessionServices({ cwd, agentDir, ... })`
2. 基于 services 解析 model scope、默认模型、thinking level 和工具 allowlist。
3. 处理 `--api-key`。
4. `createAgentSessionFromServices(...)` 创建 `AgentSession`。
5. 返回 session、services、diagnostics 和 model fallback 信息。

这样 `/new`、`/resume`、`/fork`、`/import` 可以复用同一套创建逻辑。

## AgentSessionRuntime

`packages/coding-agent/src/core/agent-session-runtime.ts` 持有当前 `AgentSession` 和 `AgentSessionServices`。

职责：

- `switchSession`: 切换到已有 session 文件。
- `newSession`: 创建新 session。
- `fork`: 从某个 entry 创建分支 session。
- `navigateTree`: 在当前 JSONL 树中切换 leaf。
- `importSession`: 导入外部 JSONL。
- `dispose`: 触发 session shutdown 并释放资源。

session replacement 统一流程：

```text
emit before event
  |
  +--> emit session_shutdown
  +--> invalidate old extension context
  +--> dispose old AgentSession
  +--> createRuntime(target cwd/session)
  +--> apply new session/services
  +--> rebind host mode
```

`setRebindSession` 由 interactive/print/RPC 模式注册，用于 session 替换后重新订阅事件和安装对应 UI context。

## 运行模式分派

`resolveAppMode` 根据参数和 stdin 决定模式：

| 模式 | 触发条件 | 入口 |
| --- | --- | --- |
| interactive | 默认且 stdin 是 TTY | `InteractiveMode.run()` |
| print | `-p` 或 stdin pipe | `runPrintMode(..., { mode: "text" })` |
| JSON | `--mode json` | `runPrintMode(..., { mode: "json" })` |
| RPC | `--mode rpc` | `runRpcMode(...)` |

如果 interactive 模式收到 piped stdin，会降级为 print 模式。

## Interactive Mode

`packages/coding-agent/src/modes/interactive/interactive-mode.ts` 是终端产品体验层。

启动时：

1. 注册信号处理。
2. 加载 changelog。
3. 确保 `fd` 和 `rg` 可用。
4. 创建 TUI 组件树。
5. 安装 key handlers 和 editor submit handler。
6. 启动 TUI。
7. 绑定当前 session 和扩展 UI context。
8. 渲染历史消息。
9. 启动 theme watcher、git branch watcher、provider count 更新。

运行时，它订阅 `AgentSessionEvent`，把 user/assistant/tool/compaction/custom message 转成 TUI 组件。

## Print 和 JSON Mode

`packages/coding-agent/src/modes/print-mode.ts` 是 single-shot 模式。

text 模式：

- 发送 initialMessage 和后续 messages。
- 等 Agent 结束。
- 输出最后一条 assistant message 的 text blocks。
- error/aborted 时写 stderr 并返回非零 exit code。

json 模式：

- 先输出 session header。
- 每个 `AgentSessionEvent` 输出一行 JSON。
- 适合脚本消费。

两种模式都会绑定扩展，但 UI context 是非交互版本。

## RPC Mode

`packages/coding-agent/src/modes/rpc/rpc-mode.ts` 用 JSONL 协议连接宿主进程。

协议形态：

- stdin: command JSON line，包含 `type` 和可选 `id`。
- stdout: response JSON line、agent event JSON line、extension UI request JSON line。

扩展 UI 在 RPC 中不会直接操作 TUI，而是输出 `extension_ui_request`，由宿主响应 `extension_ui_response`。

## SDK

`packages/coding-agent/src/core/sdk.ts` 暴露 `createAgentSession(options)`。外部应用可以直接嵌入 Pi runtime，自定义：

- cwd、agentDir。
- authStorage、settingsManager、modelRegistry。
- model、thinkingLevel、scopedModels。
- tools、customTools、noTools。
- resourceLoader。
- sessionManager。

SDK 与 CLI 使用同一个 `AgentSession`，所以会话、扩展、工具和模型行为保持一致。

## 关键源码

- CLI 入口：`packages/coding-agent/src/cli.ts`
- 启动装配：`packages/coding-agent/src/main.ts`
- runtime 宿主：`packages/coding-agent/src/core/agent-session-runtime.ts`
- runtime services：`packages/coding-agent/src/core/agent-session-services.ts`
- SDK：`packages/coding-agent/src/core/sdk.ts`
- interactive mode：`packages/coding-agent/src/modes/interactive/interactive-mode.ts`
- print mode：`packages/coding-agent/src/modes/print-mode.ts`
- RPC mode：`packages/coding-agent/src/modes/rpc/rpc-mode.ts`
