# TUI 与本地状态设计

本文说明 Pi 的终端 UI 层、输入/渲染机制，以及配置、认证、本地目录和质量门禁。阅读本文不需要先读总览文档。

## TUI 定位

TUI 是 Terminal User Interface。`packages/tui` 是通用终端 UI 框架，不依赖 coding-agent 业务。

coding-agent 的 `InteractiveMode` 使用它构建交互界面：

```text
terminal
  |
  +--> header
  +--> chat messages
  +--> pending messages
  +--> status/widgets
  +--> editor
  +--> footer
```

TUI 不负责模型调用、session 存储或工具执行。

## 核心抽象

`packages/tui/src/tui.ts` 定义：

- `Component`: `render(width): string[]`，可选 `handleInput(data)`。
- `Focusable`: 可显示硬件光标的组件。
- `Container`: 组合多个 Component。
- `TUI`: 根容器，负责 focus、overlay、输入分发、差量渲染。

`packages/tui/src/terminal.ts` 定义：

- `Terminal`: 对终端能力的抽象。
- `ProcessTerminal`: 基于 `process.stdin/stdout` 的真实终端实现。

## 组件系统

组件必须保证渲染出的每一行不超过传入 width。TUI 会在行超宽时出错，这能尽早发现 UI wrapping 问题。

内置组件包括：

- Text
- TruncatedText
- Markdown
- Input
- Editor
- SelectList
- SettingsList
- Loader
- CancellableLoader
- Image
- Box
- Spacer

coding-agent 在 `packages/coding-agent/src/modes/interactive/components` 中定义业务组件，例如 assistant message、tool execution、footer、model selector、tree selector。

## 差量渲染

`TUI.requestRender()` 会节流渲染，最小间隔 16ms。

渲染时：

1. 调用根组件 render。
2. 计算 overlay 布局。
3. 标准化 ANSI 输出。
4. 与上一帧 `previousLines` 比较。
5. 尽量只更新变化行。
6. 必要时清理缩短后的空行。
7. 更新 cursor 位置。

TUI 支持 synchronized output，减少闪烁。

## 输入处理

`ProcessTerminal.start()` 会：

- 保存 raw mode 状态。
- 开启 raw mode。
- 开启 bracketed paste。
- 监听 resize。
- Windows 下启用 VT input。
- 查询 Kitty keyboard protocol。
- 失败时回退 xterm modifyOtherKeys。

`StdinBuffer` 把批量 stdin 拆成单个 key sequence，并把 paste 内容重新包成 bracketed paste marker。

输入分发顺序：

```text
raw terminal input
  |
  +--> TUI input listeners
  |
  +--> cell size response handling
  |
  +--> global debug key
  |
  +--> focused overlay check
  |
  +--> focusedComponent.handleInput(data)
```

## Overlay 和 Focus

TUI 支持 overlay stack：

- `showOverlay(component, options)`
- `hideOverlay()`
- overlay handle: hide、setHidden、focus、unfocus、isFocused。

Overlay 可以设置：

- width / minWidth / maxHeight。
- anchor。
- row / col。
- margin。
- visible callback。
- nonCapturing。

交互模式中的模型选择器、设置面板、登录选择器等都可使用 overlay 或替换 editor 区域。

## 图片支持

`packages/tui/src/terminal-image.ts` 支持：

- Kitty graphics protocol。
- iTerm2 inline image。
- 图片尺寸识别。
- cell 尺寸查询。
- fallback 文本。

`read` tool 和剪贴板图片会在 coding-agent 层处理图片内容，TUI 只负责展示。

## InteractiveMode 布局

`packages/coding-agent/src/modes/interactive/interactive-mode.ts` 用 TUI 组装界面：

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

- 初始化主题和 keybindings。
- 初始化 autocomplete。
- 渲染 loaded resources。
- 渲染历史 session messages。
- 处理快捷键和 editor submit。
- 订阅 AgentSession events。
- 给扩展提供 interactive UI context。

## 本地目录

默认目录来自 `packages/coding-agent/src/config.ts`：

| 类型 | 默认位置 |
| --- | --- |
| agent dir | `~/.pi/agent` |
| project config dir | `<cwd>/.pi` |
| global settings | `~/.pi/agent/settings.json` |
| project settings | `<cwd>/.pi/settings.json` |
| auth | `~/.pi/agent/auth.json` |
| models | `~/.pi/agent/models.json` |
| sessions | `~/.pi/agent/sessions/<encoded-cwd>/` |
| tools/bin | `~/.pi/agent/bin` |
| prompts | `~/.pi/agent/prompts` |

agent dir 和 session dir 可由环境变量或 settings 覆盖。

## SettingsManager

`packages/coding-agent/src/core/settings-manager.ts` 管理全局和项目 settings。

行为：

- 读取 global settings。
- 读取 project settings。
- 深度合并，project 覆盖 global。
- 迁移旧字段，例如 `queueMode`、`websockets`、旧 skills 格式。
- 写入时使用 `proper-lockfile` 同步锁。
- 记录 load errors，启动时转成 diagnostics。

settings 覆盖范围包括：

- 默认 provider/model/thinking。
- transport。
- steering/follow-up mode。
- compaction 和 retry。
- shell path 和 command prefix。
- packages/extensions/skills/prompts/themes。
- terminal/images。
- enabledModels。
- tree/editor/autocomplete。
- HTTP idle timeout。

## 认证状态

认证主要保存在 `AuthStorage`，模型请求时由 `ModelRegistry` 解析。

来源包括：

- global auth file。
- OAuth credentials。
- API key。
- environment variable。
- models.json provider/model request config。
- 扩展注册的 OAuth Provider。

`InteractiveMode` 的 `/login`、`/logout` 和 provider selector 依赖这些状态。

## 构建与质量门禁

根 `package.json` 中：

- `npm run build`: 按 `tui -> ai -> agent -> coding-agent` 顺序构建。
- `npm run check`: Biome、pinned deps、TS relative import、shrinkwrap、`tsgo --noEmit`、browser smoke。
- `./test.sh`: 非 e2e 测试入口。

TypeScript 基础配置：

- `module: Node16`
- `target: ES2022`
- `strict: true`
- `erasableSyntaxOnly: true`
- 相对 TypeScript import 使用 `.ts`，构建时 `rewriteRelativeImportExtensions` 输出 `.js`。

文档变更不需要运行 `npm run check`。源码变更后按项目规则运行。

## 关键源码

- TUI root：`packages/tui/src/tui.ts`
- terminal adapter：`packages/tui/src/terminal.ts`
- key parsing：`packages/tui/src/keys.ts`
- stdin buffer：`packages/tui/src/stdin-buffer.ts`
- terminal image：`packages/tui/src/terminal-image.ts`
- interactive mode：`packages/coding-agent/src/modes/interactive/interactive-mode.ts`
- theme：`packages/coding-agent/src/modes/interactive/theme/theme.ts`
- settings：`packages/coding-agent/src/core/settings-manager.ts`
- config paths：`packages/coding-agent/src/config.ts`
