# 07 扩展生态与终端体验

本文解释 Pi 如何通过资源和扩展变成一个可定制平台，以及终端 UI 如何承载这些能力。

## 1. 为什么需要扩展生态

不同团队的开发流程差异很大：

```mermaid
flowchart TD
  A["不同团队"] --> B["不同代码规范"]
  A --> C["不同模型 Provider"]
  A --> D["不同发布流程"]
  A --> E["不同命令和工具"]
  A --> F["不同 UI 偏好"]

  B --> X["Pi 扩展生态"]
  C --> X
  D --> X
  E --> X
  F --> X

  X --> G["不 fork Pi 核心也能定制"]
```

产品定位：Pi 不只是一个固定 CLI，而是一个可被团队塑形的 agent harness。

## 2. 资源类型

```mermaid
mindmap
  root((Pi 可加载资源))
    Context Files
      AGENTS.md
      CLAUDE.md
      项目规范
    Skills
      能力说明
      /skill:name
      专项工作流
    Prompt Templates
      快速提示词
      slash command
    Extensions
      工具
      命令
      UI
      Provider
      生命周期 hook
    Themes
      视觉风格
    Packages
      npm
      git
      团队共享
```

## 3. 资源加载链路

```mermaid
flowchart TD
  A["启动或 reload"] --> B["读取 settings"]
  B --> C["解析 packages"]
  C --> D["合并资源路径"]
  D --> E["加载 extensions"]
  E --> F["扩展可发现更多资源"]
  F --> G["加载 skills"]
  F --> H["加载 prompt templates"]
  F --> I["加载 themes"]
  F --> J["加载 context files"]
  G --> K["构建系统提示词和 UI 列表"]
  H --> K
  I --> K
  J --> K
  E --> K
```

## 4. 资源来源优先级

```mermaid
flowchart LR
  A["全局用户资源"] --> M["资源合并"]
  B["项目资源"] --> M
  C["npm/git package"] --> M
  D["CLI 临时路径"] --> M
  E["扩展动态发现"] --> M

  M --> Dedupe["去重和冲突诊断"]
  Dedupe --> UI["启动页展示来源"]
  Dedupe --> Runtime["进入 Agent runtime"]
```

产品上 sourceInfo 很重要：用户需要知道某条规则、某个技能、某个命令来自哪里。

## 5. Extension 能做什么

```mermaid
flowchart TD
  X["Extension"] --> A["注册工具"]
  X --> B["注册 slash command"]
  X --> C["注册 CLI flag"]
  X --> D["注册快捷键"]
  X --> E["注册 autocomplete"]
  X --> F["替换 editor"]
  X --> G["注入 UI widget"]
  X --> H["监听事件"]
  X --> I["改写上下文"]
  X --> J["拦截工具调用"]
  X --> K["注册模型 Provider"]
  X --> L["保存扩展状态"]
```

这使扩展既能改变“Agent 怎么想”，也能改变“用户怎么操作”。

## 6. 生命周期 hook

```mermaid
sequenceDiagram
  participant UI as 用户界面
  participant AS as AgentSession
  participant EX as Extension
  participant M as 模型
  participant T as 工具

  UI->>AS: 用户输入
  AS->>EX: input hook
  EX-->>AS: 处理/改写/放行
  AS->>EX: before_agent_start
  EX-->>AS: 注入消息或系统提示词
  AS->>EX: context hook
  AS->>M: 请求模型
  M-->>AS: 工具调用
  AS->>EX: tool_call hook
  EX-->>AS: 允许/阻止/修改结果
  AS->>T: 执行工具
  T-->>AS: 工具结果
  AS->>EX: tool_result hook
  AS->>EX: message_end hook
  AS-->>UI: 展示最终消息
```

## 7. 扩展 UI 能力

```mermaid
flowchart TD
  UI["ExtensionUIContext"] --> A["select<br/>选择器"]
  UI --> B["confirm<br/>确认弹窗"]
  UI --> C["input<br/>文本输入"]
  UI --> D["notify<br/>通知"]
  UI --> E["setStatus<br/>状态栏"]
  UI --> F["setWidget<br/>编辑器上下部件"]
  UI --> G["setFooter<br/>自定义 footer"]
  UI --> H["setHeader<br/>自定义 header"]
  UI --> I["custom<br/>自定义组件/overlay"]
  UI --> J["setEditorComponent<br/>替换编辑器"]
  UI --> K["theme<br/>主题访问和切换"]
```

这意味着扩展不仅是后台逻辑，也可以提供完整交互。

## 8. 交互式 TUI 布局

```mermaid
flowchart TD
  Root["TUI Root"] --> Header["Header<br/>启动信息/加载资源"]
  Root --> Chat["Chat Container<br/>消息、工具、错误、通知"]
  Root --> Pending["Pending Messages<br/>排队消息"]
  Root --> Status["Status Container<br/>扩展状态"]
  Root --> WidgetA["Widget Above Editor"]
  Root --> Editor["Editor<br/>用户输入"]
  Root --> WidgetB["Widget Below Editor"]
  Root --> Footer["Footer<br/>cwd/session/model/tokens/cost/context"]

  Extension["Extension"] -. "可注入" .-> Header
  Extension -. "可注入" .-> WidgetA
  Extension -. "可注入" .-> WidgetB
  Extension -. "可替换" .-> Footer
  Extension -. "可替换" .-> Editor
```

## 9. 终端界面状态反馈

```mermaid
flowchart LR
  State["系统状态"] --> UI1["Loader<br/>正在工作"]
  State --> UI2["Footer<br/>模型/成本/上下文"]
  State --> UI3["Editor Border<br/>thinking level"]
  State --> UI4["Tool Components<br/>工具结果"]
  State --> UI5["Pending Area<br/>队列消息"]
  State --> UI6["Overlay<br/>选择器和设置"]
  State --> UI7["Notifications<br/>警告/错误/更新"]
```

对于非技术用户，状态反馈决定信任感。

## 10. RPC 模式下的扩展 UI

```mermaid
sequenceDiagram
  participant EX as Extension
  participant RPC as Pi RPC
  participant Host as 宿主应用
  participant User as 用户

  EX->>RPC: 请求 select/input/notify
  RPC-->>Host: extension_ui_request JSON
  Host-->>User: 宿主自己的 UI
  User-->>Host: 用户操作
  Host-->>RPC: extension_ui_response JSON
  RPC-->>EX: 返回结果
```

产品意义：

- Interactive 模式由 Pi 自己画 UI。
- RPC 模式由宿主产品画 UI。
- 扩展可以复用同一套逻辑，但 UI 交给不同外壳承载。

## 11. Packages 的业务价值

```mermaid
flowchart TD
  A["团队最佳实践"] --> B["打包成 Pi Package"]
  B --> C["包含 extensions"]
  B --> D["包含 skills"]
  B --> E["包含 prompt templates"]
  B --> F["包含 themes"]
  C --> G["团队成员安装"]
  D --> G
  E --> G
  F --> G
  G --> H["统一工作流和体验"]
```

Packages 让团队可以共享：

- 代码审查流程。
- 发布流程。
- 内部工具。
- 自定义 Provider。
- 品牌主题。
- 项目规范。

## 12. 冲突和诊断

```mermaid
flowchart TD
  A["加载多个资源"] --> B{"是否有同名冲突"}
  B -- "否" --> C["正常可用"]
  B -- "是" --> D["选择 winner"]
  D --> E["记录 loser skipped"]
  E --> F["启动页/diagnostics 展示"]
  F --> G["用户知道哪个资源生效"]
```

冲突透明对扩展生态很重要，否则用户会疑惑“为什么我的命令没有生效”。

## 13. 主题和视觉体验

```mermaid
flowchart TD
  A["Theme"] --> B["颜色变量"]
  A --> C["Markdown 样式"]
  A --> D["工具状态颜色"]
  A --> E["Editor 边框"]
  A --> F["Export HTML 颜色"]
  B --> UI["Interactive UI"]
  C --> UI
  D --> UI
  E --> UI
  F --> HTML["HTML 导出"]
```

主题不是装饰，它影响：

- 错误是否醒目。
- 工具成功/失败是否清楚。
- 长会话阅读疲劳。
- 导出内容是否适合分享。

## 14. 生态产品机会

```mermaid
mindmap
  root((扩展生态机会))
    发现
      包市场
      推荐技能
      项目自动提示
    安装
      一键安装
      版本管理
      权限说明
    信任
      来源展示
      冲突诊断
      安全边界
    复用
      团队模板
      行业工作流
      内部 Provider
    UI
      扩展面板
      自定义编辑器
      可视化工具结果
    运营
      热门包
      成功案例
      分享会话
```

## 15. 平台化路径

```mermaid
flowchart LR
  A["单一 CLI 工具"] --> B["可配置 CLI"]
  B --> C["可扩展 Agent"]
  C --> D["团队可共享工作流"]
  D --> E["外部应用可嵌入平台"]
  E --> F["Agent 生态"]
```

Pi 的扩展和 RPC/SDK 能力，让它有从工具走向平台的空间。

## 16. 小结

扩展生态和 TUI 是 Pi 的产品杠杆：

- 资源让 Pi 理解团队规则。
- 扩展让 Pi 拥有团队特定能力。
- TUI 让复杂过程可见、可控。
- RPC/SDK 让 Pi 能被其他产品复用。

对产品经理来说，后续最值得关注的是：扩展如何被发现、安装、信任、调试和分享。
