# 资源加载与扩展系统设计

本文说明 Pi 如何加载项目/用户资源，以及扩展如何接入 Agent、UI、Provider 和 session 生命周期。阅读本文不需要先读总览文档。

## 资源类型

Pi 支持的资源包括：

- extensions: TypeScript 扩展模块。
- skills: `SKILL.md` 能力说明。
- prompt templates: slash prompt 模板。
- themes: 交互主题。
- context files: `AGENTS.md`、`CLAUDE.md`。
- packages: npm/git 包中的资源集合。
- system prompt / append system prompt。

资源统一由 `DefaultResourceLoader` 管理。

## 加载来源

资源来自多个层级：

| 来源 | 示例 | 作用 |
| --- | --- | --- |
| 全局 agent dir | `~/.pi/agent` | 用户级默认配置和资源 |
| 项目目录 | `<cwd>/.pi` | 项目级资源 |
| CLI 参数 | `--extensions`, `--skills` | 本次运行临时资源 |
| package | npm/git package | 可分享资源包 |
| 祖先目录 | `AGENTS.md`, `CLAUDE.md` | 项目上下文 |
| 扩展发现 | `resources_discover` | 扩展动态注入资源 |

CLI 路径会先解析成绝对路径，避免 session cwd 切换后被重新解释。

## DefaultResourceLoader

`packages/coding-agent/src/core/resource-loader.ts` 提供 `DefaultResourceLoader`。

职责：

- reload settings。
- 通过 `DefaultPackageManager` 解析 package 资源。
- 合并 package、settings、CLI 资源路径。
- 加载 extensions。
- 应用扩展动态发现的资源。
- 加载 skills、prompts、themes。
- 加载 context files。
- 解析 system prompt 和 append system prompt。
- 记录 diagnostics 和 sourceInfo。

`ResourceLoader` 对外提供：

- `getExtensions()`
- `getSkills()`
- `getPrompts()`
- `getThemes()`
- `getAgentsFiles()`
- `getSystemPrompt()`
- `getAppendSystemPrompt()`
- `extendResources(paths)`
- `reload()`

## Context Files

context file 搜索规则：

1. 先读全局 agent dir 中的 `AGENTS.md` 或 `CLAUDE.md`。
2. 从 cwd 向根目录逐级查找同名文件。
3. 祖先目录文件按从上到下顺序加入。
4. 去重同一路径。

这些内容会进入 system prompt 构建。

## SourceInfo

资源会标注来源：

- user
- project
- temporary path
- cli
- npm package
- git package

交互启动页用 sourceInfo 展示资源来自哪里，并在碰撞时说明 winner 和 skipped loser。

## 扩展能力

扩展是 TypeScript 模块，可以：

- 注册 LLM 工具。
- 注册 slash command。
- 注册 CLI flag。
- 注册 keyboard shortcut。
- 注册 autocomplete provider wrapper。
- 注册自定义 editor。
- 监听 agent 和 session 生命周期。
- 拦截 input、context、tool_call、tool_result。
- 修改 provider request payload。
- 动态注册 Provider 和 OAuth Provider。
- 注入 UI 组件或 RPC UI request。
- 持久化 custom session entries。

扩展类型定义在 `packages/coding-agent/src/core/extensions/types.ts`。

## ExtensionRunner

`packages/coding-agent/src/core/extensions/runner.ts` 是扩展运行时。

它负责：

- 保存已加载 extensions 和 runtime。
- 提供 extension context。
- 分发生命周期事件。
- 汇总 before/after hook 结果。
- 管理 extension command context。
- 管理 UI context。
- 管理错误监听和 stale context invalidation。

非交互模式也会创建 ExtensionRunner，只是 UI context 能力不同。

## 扩展生命周期事件

主要事件包括：

- `session_start`
- `session_shutdown`
- `session_before_switch`
- `session_before_fork`
- `session_before_compact`
- `agent_start`
- `turn_start`
- `message_start`
- `message_update`
- `message_end`
- `tool_execution_start`
- `tool_execution_update`
- `tool_execution_end`
- `turn_end`
- `agent_end`
- `input`
- `context`
- `before_agent_start`
- `before_provider_request`
- `after_provider_response`
- `tool_call`
- `tool_result`
- `resources_discover`

`AgentSession` 把 Agent Core 事件转换成扩展事件，并在必要时允许扩展替换消息或工具结果。

## Extension UI Context

扩展通过 `ExtensionUIContext` 请求 UI 能力。

交互模式支持：

- selector、confirm、input。
- notify。
- raw terminal input listener。
- footer/status。
- working message/indicator。
- widgets。
- custom footer/header。
- overlay 或全屏 custom component。
- editor text 操作。
- custom editor。
- theme 查询和切换。

print 模式提供 no-op 或降级实现。

RPC 模式把 UI 请求输出为 JSONL：

```text
extension_ui_request
  |
  +--> host handles request
  |
  +--> extension_ui_response
```

## 动态 Provider

扩展可注册 Provider 配置。`createAgentSessionServices` 会读取 extension runtime 中的 pending provider registrations，并调用 `ModelRegistry.registerProvider`。

注册内容可以包括：

- provider display name。
- baseUrl 和 headers override。
- models。
- apiKey 规则。
- OAuth provider。
- 自定义 `streamSimple`。

这让扩展可以把新模型后端接入现有 Agent loop。

## Reload

`AgentSession.reload()` 会重新加载资源并重建 runtime：

- settings 重新读取。
- resources 重新发现。
- extensions 重新加载。
- tools 重新注册。
- system prompt 重新构建。
- UI 重新展示 loaded resources 和 diagnostics。

旧 extension context 会被标记 stale，避免扩展在 session replacement 或 reload 后继续操作旧 session。

## 关键源码

- ResourceLoader：`packages/coding-agent/src/core/resource-loader.ts`
- Package manager：`packages/coding-agent/src/core/package-manager.ts`
- Extension types：`packages/coding-agent/src/core/extensions/types.ts`
- Extension loader：`packages/coding-agent/src/core/extensions/loader.ts`
- Extension runner：`packages/coding-agent/src/core/extensions/runner.ts`
- AgentSession extension bridge：`packages/coding-agent/src/core/agent-session.ts`
