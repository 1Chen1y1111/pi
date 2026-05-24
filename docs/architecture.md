# Pi 源码架构设计文档

本文基于当前仓库源码整理，目标是帮助维护者快速理解 `pi` 从 CLI 启动、资源加载、会话运行、模型调用到终端展示的主要架构边界。

本文是母文档。更细的专题设计文档在 `docs/architecture/` 下，基于本文内容衍生，可独立阅读。

## 1. 系统定位

Pi 是一个面向终端的可扩展 coding agent。仓库采用 npm workspaces 组织，核心产物是 `@earendil-works/pi-coding-agent` 提供的 `pi` CLI；底层拆成可复用 Agent 运行时、统一 LLM Provider 适配层和 TUI 库。

主要设计取向：

- CLI 应用层负责用户体验、资源发现、会话管理和扩展系统。
- Agent Core 只关心消息状态、事件流、工具执行和队列语义。
- AI 包负责把统一上下文转成各 Provider 的流式协议。
- TUI 包保持通用，不依赖 coding-agent 业务语义。
- 会话用 append-only JSONL 表达树形历史，支持 resume、branch、fork、compact。

## 2. Monorepo 包边界

| 包 | 位置 | 职责 | 关键源码 |
| --- | --- | --- | --- |
| `@earendil-works/pi-coding-agent` | `packages/coding-agent` | CLI、交互模式、打印/RPC 模式、会话封装、资源/扩展、内置工具、配置和发布资产 | `src/main.ts`, `src/core/agent-session.ts`, `src/core/sdk.ts`, `src/modes/*` |
| `@earendil-works/pi-agent-core` | `packages/agent` | 可复用 agent 状态机、事件循环、工具执行、steering/follow-up 队列 | `src/agent.ts`, `src/agent-loop.ts`, `src/types.ts` |
| `@earendil-works/pi-ai` | `packages/ai` | 统一模型、消息、工具 schema、Provider registry、OAuth、流式事件抽象 | `src/stream.ts`, `src/api-registry.ts`, `src/providers/*`, `src/models.ts` |
| `@earendil-works/pi-tui` | `packages/tui` | 终端组件系统、差量渲染、输入解析、overlay、图片协议 | `src/tui.ts`, `src/terminal.ts`, `src/components/*` |

依赖方向：

```text
packages/coding-agent
  -> packages/agent
  -> packages/ai

packages/coding-agent
  -> packages/tui

packages/agent
  -> packages/ai

packages/ai, packages/tui
  -> no internal package dependency
```

这个方向是重要约束：`pi-ai` 和 `pi-tui` 应保持可独立复用；业务能力集中在 `pi-coding-agent`。

## 3. 衍生专题文档

以下专题文档基于本文拆出，保留足够上下文，可单独阅读：

- [启动与运行模式](architecture/runtime.md): CLI 启动、runtime 创建、interactive/print/RPC/SDK 模式分派。
- [Agent 循环与会话封装](architecture/agent-loop.md): `AgentSession`、`Agent`、事件循环、队列和扩展 hook。
- [内置工具体系](architecture/tools.md): 内置工具、TypeBox schema、执行生命周期和文件写入串行化。
- [会话存储与树形历史](architecture/sessions.md): JSONL session 格式、branch/fork/compact 和上下文重建。
- [资源加载与扩展系统](architecture/resources-extensions.md): settings/package/CLI 资源发现、扩展 API 和生命周期。
- [模型与 Provider 架构](architecture/models-providers.md): `pi-ai` Provider registry、`ModelRegistry`、认证和动态 Provider。
- [TUI 与本地状态](architecture/tui-and-local-state.md): 终端 UI、输入/渲染、配置、认证、本地目录和质量门禁。

## 4. 总体运行图

```text
pi CLI
  |
  v
packages/coding-agent/src/cli.ts
  |
  v
main(args)
  |
  +--> parseArgs / migrations / settings / session selection
  |
  +--> createAgentSessionRuntime(factory)
          |
          +--> createAgentSessionServices(cwd-bound)
          |     |
          |     +--> SettingsManager
          |     +--> AuthStorage
          |     +--> ModelRegistry
          |     +--> DefaultResourceLoader
          |
          +--> createAgentSession(...)
                |
                +--> AgentSession
                      |
                      +--> Agent from pi-agent-core
                      |     |
                      |     +--> runAgentLoop
                      |           |
                      |           +--> streamSimple from pi-ai
                      |           +--> execute tool calls
                      |
                      +--> SessionManager JSONL persistence
                      +--> ExtensionRunner
                      +--> built-in/custom tools

main mode dispatch
  |
  +--> InteractiveMode -> pi-tui
  +--> runPrintMode   -> stdout text/json
  +--> runRpcMode     -> JSONL RPC protocol
```

## 5. 启动流程

入口是 `packages/coding-agent/src/cli.ts`：

1. 设置进程标题和 `PI_CODING_AGENT` 环境变量。
2. 配置全局 HTTP dispatcher。
3. 调用 `main(process.argv.slice(2))`。

`packages/coding-agent/src/main.ts` 是应用装配层，核心步骤：

1. 处理离线模式、自更新遗留清理、包管理和配置子命令。
2. `parseArgs` 解析 CLI 参数，并根据 `--mode`、`--json`、`-p`、stdin 是否为 TTY 决定运行模式。
3. 运行迁移，创建 startup `SettingsManager`，用它解析 sessionDir。
4. 创建或打开 `SessionManager`，包括 `--session`、`--fork`、`--resume`、`--continue`、`--no-session`。
5. 基于最终 session cwd 创建 runtime factory。这里特别重要：session 可能来自其他项目，所以 settings、resources、models 必须在最终 cwd 上重建。
6. `createAgentSessionRuntime` 创建 `AgentSessionRuntime`，内部持有当前 `AgentSession` 和 cwd-bound services。
7. 读取 stdin 和 `@file` 参数，准备初始 prompt 和图片。
8. 初始化主题，报告 diagnostics。
9. 分派到交互、打印、JSON 或 RPC 模式。

## 6. Runtime 与 cwd-bound services

`AgentSessionRuntime` 是会话宿主，主要解决“当前 session 可能切换 cwd”的问题。

职责：

- 保存当前 `AgentSession` 和 `AgentSessionServices`。
- 在 `/new`、`/resume`、`/fork`、`/import` 等操作中先触发扩展生命周期事件，再销毁旧 session。
- 使用同一个 runtime factory 基于目标 cwd 重建 services 和 session。
- 通过 `setRebindSession` 让交互/打印/RPC 模式在 session 切换后重新绑定事件和 UI。

`AgentSessionServices` 是一组按 cwd 绑定的基础设施：

- `SettingsManager`: 合并全局和项目设置。
- `AuthStorage`: 读取 API key/OAuth 凭据。
- `ModelRegistry`: 合并内置模型、`models.json` 和扩展注册 Provider。
- `DefaultResourceLoader`: 加载扩展、技能、prompt template、theme、AGENTS/CLAUDE 上下文文件。

## 7. AgentSession 业务封装

`packages/coding-agent/src/core/agent-session.ts` 是 coding-agent 的核心业务层。它包住 `pi-agent-core` 的 `Agent`，并加入 Pi 的产品语义：

- 将资源加载结果转成系统提示词。
- 管理当前启用工具和工具 prompt snippets。
- 执行 slash command、扩展 command、skill command、prompt template 展开。
- 处理 streaming 时的 steering 和 follow-up 队列。
- 做模型和认证 preflight。
- 触发扩展事件，包括 input、message、tool、provider request/response、session lifecycle。
- 把 `Agent` 事件持久化到 `SessionManager`。
- 管理自动 compaction、branch summary 和自动重试。
- 暴露给 UI/SDK 的 session 操作，例如 prompt、steer、followUp、reload、navigateTree、setActiveTools。

核心 prompt 路径：

```text
AgentSession.prompt(text, options)
  |
  +--> extension command check
  +--> extension input hook
  +--> skill/template expansion
  +--> if streaming: queue steer/follow-up
  +--> model/auth validation
  +--> optional compaction before send
  +--> before_agent_start extension hook
  +--> Agent.prompt(messages)
  +--> post-run retry or compaction continuation
```

## 8. Agent Core 事件循环

`packages/agent/src/agent.ts` 提供 `Agent` 类，管理：

- `AgentState`: system prompt、model、thinking level、tools、messages、streaming 状态。
- `subscribe` 事件监听。
- `prompt` 和 `continue`。
- `steer` 与 `followUp` 队列。
- abort、waitForIdle、reset。

实际循环在 `packages/agent/src/agent-loop.ts`：

```text
runAgentLoop(prompts, context, config)
  |
  +--> emit agent_start
  +--> emit turn_start
  +--> emit user message_start/end
  |
  +--> runLoop
        |
        +--> drain steering messages
        +--> transformContext(AgentMessage[])
        +--> convertToLlm(Message[])
        +--> stream assistant response
        +--> execute tool calls
        +--> emit turn_end
        +--> prepareNextTurn
        +--> shouldStopAfterTurn
        +--> drain follow-up messages
        +--> emit agent_end
```

工具执行支持两种策略：

- `parallel`: 默认。先顺序 preflight，再并行执行可并行工具；tool result 按 assistant 原始 tool call 顺序写回。
- `sequential`: 全部按顺序执行。任一工具声明 `executionMode: "sequential"` 时，本批 tool calls 会整体顺序执行。

工具调用生命周期：

```text
assistant toolCall
  |
  +--> tool_execution_start
  +--> prepareArguments
  +--> validateToolArguments(TypeBox)
  +--> beforeToolCall hook
  +--> tool.execute(...)
  |     +--> optional tool_execution_update
  +--> afterToolCall hook
  +--> tool_execution_end
  +--> toolResult message_start/end
```

## 9. 内置工具体系

内置工具在 `packages/coding-agent/src/core/tools`：

| 工具 | 文件 | 作用 |
| --- | --- | --- |
| `read` | `read.ts` | 读取文本和图片，支持 offset/limit 和输出截断 |
| `bash` | `bash.ts` | 本地 shell 执行，支持流式输出、timeout、abort、命令前缀和 shell 配置 |
| `edit` | `edit.ts` | 基于唯一旧文本的局部替换，生成 diff/patch |
| `write` | `write.ts` | 创建或覆盖文件，自动创建父目录 |
| `grep` | `grep.ts` | 文本搜索 |
| `find` | `find.ts` | 文件查找 |
| `ls` | `ls.ts` | 目录列举 |

`tools/index.ts` 统一导出工具 definition 和 executable tool。`createCodingTools` 默认启用 `read/bash/edit/write`，`createReadOnlyTools` 提供只读组合。

重要约束：

- 文件变更工具通过 `withFileMutationQueue` 串行化同一文件的写入。
- 工具 schema 使用 TypeBox，Agent Core 在执行前统一校验参数。
- 工具 definition 同时包含 LLM schema、执行函数和 UI/HTML 渲染能力。

## 10. 会话存储与树形历史

`SessionManager` 负责 JSONL 会话文件。当前版本是 `CURRENT_SESSION_VERSION = 3`。

会话文件结构：

- 首行 `SessionHeader`，包含 session id、cwd、timestamp、parentSession。
- 后续每行一个 append-only `SessionEntry`。
- 每个 entry 都有 `id` 和 `parentId`，形成树而非线性列表。

主要 entry 类型：

- `message`: user、assistant、toolResult、bashExecution 等消息。
- `thinking_level_change`: thinking level 切换。
- `model_change`: 模型切换。
- `compaction`: 对历史压缩后的摘要。
- `branch_summary`: 从某分支返回时注入的摘要。
- `custom`: 扩展私有状态，不进 LLM context。
- `custom_message`: 扩展注入消息，会进入 LLM context。
- `label`: 用户书签/标记。
- `session_info`: session 显示名等元数据。

上下文重建由 `buildSessionContext` 完成：

```text
leafId
  |
  +--> walk parentId to root
  +--> collect model/thinking changes
  +--> if compaction exists:
  |       emit compaction summary
  |       keep messages from firstKeptEntryId
  |       append messages after compaction
  |
  +--> convert branch/custom entries to AgentMessage where needed
```

这个设计让 `/tree` 可以在同一个文件里切换分支，`/fork` 可以把某条 path 提取成新 session。

## 11. 资源加载与扩展系统

`DefaultResourceLoader` 统一加载用户和项目资源：

- 全局目录：默认 `~/.pi/agent`。
- 项目目录：默认 `<cwd>/.pi`。
- CLI 临时路径：`--extensions`、`--skills`、`--prompt-templates`、`--themes`。
- 包来源：npm/git package，通过 `DefaultPackageManager` 解析。
- 上下文文件：全局和 cwd 向上祖先目录中的 `AGENTS.md`、`CLAUDE.md`。

加载顺序由 settings、包资源和 CLI 资源合并而来，并对 skills/prompts/themes 做去重和 sourceInfo 标注。扩展可通过 `resources_discover` 等机制继续注入资源路径。

扩展系统由 `extensions/types.ts`、`loader.ts`、`runner.ts` 组成。扩展可以：

- 注册 LLM 工具。
- 注册 slash command、CLI flag、快捷键。
- 监听 agent/session/tool/provider 生命周期事件。
- 拦截输入、上下文和 provider payload。
- 注入 UI，例如 selector、confirm、input、widget、footer、header、custom editor。
- 动态注册 Provider 或 OAuth Provider。

扩展 runner 在非交互模式下也存在；print/RPC 模式提供不同的 `ExtensionUIContext`。RPC 会把 UI 请求转成 JSONL `extension_ui_request` 发给宿主。

## 12. 模型与 Provider 架构

`packages/ai` 提供统一 LLM API。核心类型是：

- `Model<Api>`: 模型元数据、provider、api、baseUrl、cost、contextWindow、thinking 支持等。
- `Context`: systemPrompt、messages、tools。
- `AssistantMessageEventStream`: start、text/thinking/toolcall delta、done/error 等流式事件。
- `Tool`: TypeBox 参数 schema。

Provider 选择流程：

```text
AgentSession / Agent
  |
  +--> streamSimple(model, context, options)
        |
        +--> getApiProvider(model.api)
              |
              +--> provider.streamSimple(...)
                    |
                    +--> OpenAI / Anthropic / Google / Bedrock / Mistral / ...
```

`api-registry.ts` 维护 `api -> stream function` 映射。`providers/register-builtins.ts` 在模块加载时注册内置 API Provider，并用 lazy wrapper 延迟加载具体 Provider 模块。

`ModelRegistry` 位于 coding-agent 层，职责比 `pi-ai/src/models.ts` 更上层：

- 从 `pi-ai` 的生成模型表加载内置模型。
- 合并用户 `models.json` 的 provider/model override 和 custom models。
- 读取 AuthStorage、环境变量、OAuth 和 command-backed API key。
- 允许扩展动态 `registerProvider`，包括自定义 `streamSimple` 和 OAuth。
- 为每次请求解析 apiKey 与 headers。

这样分层后，`pi-ai` 保持 Provider 协议抽象；`pi-coding-agent` 决定用户配置和扩展如何影响模型列表。

## 13. 交互、打印、RPC 和 SDK 模式

### Interactive mode

`InteractiveMode` 是终端产品体验层，使用 `pi-tui` 构建界面：

```text
TUI root
  |
  +--> headerContainer
  +--> chatContainer
  +--> pendingMessagesContainer
  +--> statusContainer
  +--> widgetContainerAbove
  +--> editorContainer
  +--> widgetContainerBelow
  +--> footer
```

它负责：

- 初始化主题、keybindings、autocomplete、fd/rg 工具。
- 渲染历史消息、assistant 流、工具调用、compaction、branch summary、自定义消息。
- 处理编辑器提交、快捷键、剪贴板图片、拖拽/文件参数、外部编辑器。
- 提供 `/login`、`/model`、`/settings`、`/resume`、`/tree`、`/fork`、`/compact`、`/export` 等命令 UI。
- 在 session 切换后通过 `rebindCurrentSession` 重新绑定事件和扩展 UI context。

### Print and JSON mode

`runPrintMode` 是 single-shot 模式：

- text 模式只输出最终 assistant 文本。
- json 模式输出 session header 和 `AgentSessionEvent` JSON 行。
- 仍会绑定扩展，但 UI context 退化为非交互能力。

### RPC mode

`runRpcMode` 通过 stdin/stdout JSONL 与宿主进程通信：

- 输入是带 `type` 和可选 `id` 的命令。
- 输出包括 response、agent events、extension UI requests。
- 用于把 Pi 嵌入其他应用，而不直接占用终端 UI。

### SDK

`packages/coding-agent/src/core/sdk.ts` 暴露 `createAgentSession`，外部应用可直接创建 `AgentSession`，并自定义 model、tools、resourceLoader、sessionManager、settingsManager 等。

## 14. TUI 层设计

`packages/tui` 是通用终端 UI 框架。核心抽象：

- `Terminal`: 输入、输出、尺寸、光标、清屏、标题和进度接口。
- `ProcessTerminal`: 基于 process stdin/stdout 的真实终端实现。
- `Component`: `render(width): string[]` 和可选 `handleInput`。
- `Container`: 组合子组件。
- `TUI`: 管理组件树、focus、overlay、输入分发和差量渲染。

渲染策略：

- `requestRender` 做节流，最小间隔 16ms。
- `doRender` 比较当前渲染行与上一帧，尽量只更新变化行。
- 支持 terminal shrink 清理、hardware cursor marker、Kitty/iTerm 图片协议和 synchronized output。

输入策略：

- `ProcessTerminal` 开启 raw mode、bracketed paste。
- 优先尝试 Kitty keyboard protocol，失败时回退 xterm modifyOtherKeys。
- `StdinBuffer` 把批量输入拆成单个按键序列。
- TUI 先跑全局 input listeners，再交给当前 focused component。

## 15. 配置、认证与本地状态

默认目录来自 `packages/coding-agent/src/config.ts`：

- 全局 agent dir: `~/.pi/agent`，可由环境变量覆盖。
- 项目配置 dir: `.pi`。
- settings: 全局 `settings.json` 和项目 `.pi/settings.json`。
- auth: 全局 `auth.json`。
- models: 全局 `models.json`。
- sessions: `~/.pi/agent/sessions/<encoded-cwd>/`，可由 settings 或环境变量覆盖。

`SettingsManager` 深度合并 global/project 设置。写入时通过 `proper-lockfile` 做同步锁，避免多进程并发覆盖。

认证来源包括：

- AuthStorage 中保存的 API key 或 OAuth credentials。
- `models.json` 中 provider 或 model 级 apiKey/headers。
- 环境变量。
- 扩展注册的 OAuth Provider。

## 16. 构建与质量门禁

根 `package.json` 定义：

- `npm run build`: 按 `tui -> ai -> agent -> coding-agent` 顺序构建。
- `npm run check`: Biome、pinned deps、TS relative import、shrinkwrap、`tsgo --noEmit`、browser smoke。
- `./test.sh`: 非 e2e 测试入口。

TypeScript 基础配置：

- `module: Node16`
- `target: ES2022`
- `strict: true`
- `erasableSyntaxOnly: true`
- 相对 TypeScript import 使用 `.ts`，构建时 `rewriteRelativeImportExtensions` 输出 `.js`。

## 17. 常见变更入口

新增内置工具：

1. 在 `packages/coding-agent/src/core/tools` 添加 tool definition 和 tool factory。
2. 在 `tools/index.ts` 导出并加入 `ToolName`、`allToolNames`、`createToolDefinition`、`createTool`。
3. 如果默认启用，更新 `createCodingTools` 或 `createReadOnlyTools`。
4. 补充 AgentSession/system prompt 相关行为和测试。

新增 Provider：

1. 在 `packages/ai/src/providers` 实现 Provider stream 和 streamSimple。
2. 在 `packages/ai/src/providers/register-builtins.ts` 注册 API Provider。
3. 在模型生成脚本里加入模型来源，不直接改 `models.generated.ts`。
4. 在 coding-agent 的 `ModelRegistry` 侧确认 auth、headers、compat、OAuth 是否需要额外处理。

新增交互命令：

1. 在 `core/slash-commands.ts` 声明 command 信息。
2. 在 `InteractiveMode` 中实现对应处理。
3. 若命令也应支持 headless，补齐 print/RPC/SDK 的行为或明确限制。

修改 session 格式：

1. 增加 entry 类型和版本迁移。
2. 更新 `buildSessionContext`。
3. 更新 session-format 文档和回归测试。
4. 保持 append-only 语义，避免原地修改历史。

新增资源类型或包资源能力：

1. 扩展 `DefaultResourceLoader` 的路径解析和 reload 流程。
2. 更新 package manager 的资源发现。
3. 给交互启动页、reload、diagnostics 和 sourceInfo 展示补齐处理。

## 18. 架构风险与维护建议

- `AgentSession` 和 `InteractiveMode` 都承担大量产品逻辑，修改时优先保持边界清晰：业务状态放 `AgentSession`，终端呈现放 `InteractiveMode`，通用事件循环放 `pi-agent-core`。
- Provider 相关逻辑分布在 `pi-ai` 和 `ModelRegistry`，协议适配不要上移到 coding-agent，用户配置和认证解析不要下沉到 pi-ai。
- JSONL session 是兼容性边界。新增字段要向后兼容，修改语义要有迁移和测试。
- 扩展 API 是外部契约。新增 lifecycle hook 可以，修改已有事件形状要谨慎。
- TUI 组件必须保证每行不超过 viewport width，否则底层会报错；复杂 UI 应在组件层处理 wrapping/truncation。
- 多 session/多进程是常见场景。settings、session、文件编辑相关代码要考虑并发和 append-only 约束。

## 19. 关键源码索引

- CLI 入口：`packages/coding-agent/src/cli.ts`
- 启动装配：`packages/coding-agent/src/main.ts`
- SDK/session 创建：`packages/coding-agent/src/core/sdk.ts`
- runtime 切换：`packages/coding-agent/src/core/agent-session-runtime.ts`
- session 业务封装：`packages/coding-agent/src/core/agent-session.ts`
- session 存储：`packages/coding-agent/src/core/session-manager.ts`
- settings：`packages/coding-agent/src/core/settings-manager.ts`
- resources：`packages/coding-agent/src/core/resource-loader.ts`
- extensions：`packages/coding-agent/src/core/extensions/*`
- built-in tools：`packages/coding-agent/src/core/tools/*`
- interactive mode：`packages/coding-agent/src/modes/interactive/interactive-mode.ts`
- print mode：`packages/coding-agent/src/modes/print-mode.ts`
- RPC mode：`packages/coding-agent/src/modes/rpc/rpc-mode.ts`
- Agent class：`packages/agent/src/agent.ts`
- Agent loop：`packages/agent/src/agent-loop.ts`
- AI stream API：`packages/ai/src/stream.ts`
- AI provider registry：`packages/ai/src/api-registry.ts`
- built-in provider registration：`packages/ai/src/providers/register-builtins.ts`
- generated model facade：`packages/ai/src/models.ts`
- TUI root：`packages/tui/src/tui.ts`
- terminal adapter：`packages/tui/src/terminal.ts`
