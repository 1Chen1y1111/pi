# 01 产品全景与业务对象

本文从产品视角解释 Pi 是什么、谁在使用它、系统里有哪些业务对象，以及这些对象如何形成一个完整的开发助手业务闭环。

## 1. 产品一句话

Pi 是一个运行在终端里的 coding agent。用户用自然语言描述开发任务，Pi 会选择模型、读取项目、运行工具、修改文件、执行命令，并把整个过程保存成可恢复的会话。

```mermaid
flowchart TD
  U["用户<br/>开发者/维护者/自动化宿主"] --> I["输入任务<br/>自然语言、文件、图片、stdin"]
  I --> P["Pi Coding Agent<br/>理解任务并组织执行"]
  P --> M["模型 Provider<br/>OpenAI / Anthropic / Google / Bedrock 等"]
  P --> T["工具<br/>read / bash / edit / write / grep / find / ls"]
  P --> S["会话资产<br/>JSONL 历史、分支、压缩摘要、导出"]
  P --> X["扩展生态<br/>Skills / Prompts / Extensions / Themes / Packages"]
  P --> UI["终端交互界面<br/>TUI / Print / JSON / RPC"]
  T --> W["项目工作区<br/>源码、配置、测试、文档"]
  S --> U
  UI --> U
```

## 2. 核心业务对象

```mermaid
erDiagram
  USER ||--o{ SESSION : "创建/恢复"
  USER ||--o{ PROMPT : "提交"
  SESSION ||--o{ MESSAGE : "保存"
  SESSION ||--o{ BRANCH : "形成"
  SESSION ||--o{ COMPACTION : "生成"
  PROMPT ||--o{ MODEL_CALL : "触发"
  MODEL_CALL ||--o{ TOOL_CALL : "请求"
  TOOL_CALL ||--o{ TOOL_RESULT : "返回"
  MODEL_CALL }o--|| MODEL : "使用"
  MODEL }o--|| PROVIDER : "归属"
  PROVIDER ||--o{ AUTH : "需要"
  AGENT ||--o{ RESOURCE : "加载"
  RESOURCE ||--o{ EXTENSION : "扩展能力"

  USER {
    string role "使用者"
  }
  SESSION {
    string id "会话ID"
    string cwd "工作目录"
    string state "当前分支位置"
  }
  MESSAGE {
    string role "用户/助手/工具结果"
    string content "内容"
  }
  MODEL {
    string provider "模型厂商"
    string id "模型ID"
    string capability "文本/图片/推理"
  }
  TOOL_CALL {
    string name "工具名"
    string args "参数"
  }
```

这些对象可以翻译成产品语言：

| 技术对象 | 产品语言 | 用户感知 |
| --- | --- | --- |
| Session | 一次工作记录 | “我上次让它做的事还能继续” |
| Message | 对话和行动日志 | “它刚才读了什么、改了什么、为什么失败” |
| Tool | Agent 的行动能力 | “它能看文件、改文件、跑命令” |
| Model | 思考引擎 | “我现在用哪个大模型帮我干活” |
| Provider | 模型服务来源 | “用 Anthropic、OpenAI 还是别的服务” |
| Resource | 个性化配置资源 | “这个项目有什么规范、技能、模板、主题” |
| Extension | 插件能力 | “这个团队可以给 Pi 加自己的工作流” |

## 3. 用户是谁

```mermaid
flowchart LR
  Dev["开发者"] -->|日常编码| Pi["Pi"]
  Maintainer["维护者"] -->|修 bug / 审 issue / 发布| Pi
  PowerUser["高阶用户"] -->|自定义技能/扩展| Pi
  HostApp["宿主应用"] -->|RPC/SDK 集成| Pi
  Team["团队"] -->|共享包/规范/主题| Pi

  Pi --> Code["项目代码"]
  Pi --> Docs["项目文档"]
  Pi --> Tests["测试与检查"]
  Pi --> Sessions["会话沉淀"]
```

不同用户的核心诉求：

| 用户类型 | 想解决的问题 | Pi 对应能力 |
| --- | --- | --- |
| 普通开发者 | 快速完成一个代码任务 | interactive mode、内置工具、自动会话 |
| 项目维护者 | 处理 issue、回溯修改过程 | session tree、HTML export、share |
| 团队管理员 | 固化团队规范和工作流 | AGENTS.md、skills、prompt templates、packages |
| 高阶用户 | 增加自定义命令和工具 | extensions、custom provider、custom UI |
| 外部应用 | 把 Agent 嵌入产品 | RPC mode、SDK |

## 4. 产品能力地图

```mermaid
mindmap
  root((Pi 产品能力))
    输入
      自然语言
      多轮对话
      文件引用
      图片
      stdin
      初始消息
    执行
      模型推理
      工具调用
      并行工具
      命令执行
      文件修改
      自动重试
    控制
      中断
      Steering 纠偏
      Follow-up 后续
      模型切换
      Thinking 切换
      工具启停
    记忆
      自动保存
      Resume
      Tree
      Fork
      Clone
      Compact
      Export
    个性化
      Settings
      Themes
      Keybindings
      Skills
      Prompt Templates
      Extensions
      Packages
    集成
      Print
      JSON
      RPC
      SDK
      Custom Provider
```

## 5. 业务闭环

Pi 的业务闭环可以看成“输入 -> 行动 -> 反馈 -> 沉淀 -> 再利用”。

```mermaid
flowchart TD
  A["输入需求<br/>用户描述目标"] --> B["理解上下文<br/>读取项目规范、历史、文件"]
  B --> C["制定行动<br/>模型决定要读、搜、改、跑什么"]
  C --> D["执行工具<br/>真实操作工作区"]
  D --> E["观察结果<br/>工具输出进入上下文"]
  E --> F{"目标是否达成"}
  F -- "未达成" --> C
  F -- "达成" --> G["输出说明<br/>总结修改和验证"]
  G --> H["保存会话<br/>形成可恢复资产"]
  H --> I["后续复用<br/>resume / tree / fork / export"]
  I --> B
```

产品上最重要的不是单次回答，而是这个闭环能持续运行。

## 6. 三类产品形态

```mermaid
flowchart TB
  Pi["Pi 核心能力"] --> A["交互式产品<br/>用户在终端里实时协作"]
  Pi --> B["自动化工具<br/>print/json 模式一次性执行"]
  Pi --> C["平台能力<br/>RPC/SDK 被其他产品嵌入"]

  A --> A1["TUI"]
  A --> A2["快捷键"]
  A --> A3["选择器/设置/登录"]

  B --> B1["脚本调用"]
  B --> B2["CI/批处理"]
  B --> B3["JSON 事件流"]

  C --> C1["外部 IDE/桌面应用"]
  C --> C2["团队内部平台"]
  C --> C3["定制 Agent 产品"]
```

## 7. 值得深挖的产品问题

```mermaid
flowchart TD
  Q["值得深挖的问题"] --> Q1["用户什么时候需要中途纠偏"]
  Q --> Q2["会话历史如何变成可复用资产"]
  Q --> Q3["哪些工具结果需要更强可视化"]
  Q --> Q4["模型切换和成本如何让用户有掌控感"]
  Q --> Q5["团队如何共享规范和扩展"]
  Q --> Q6["新用户首次登录/选择模型是否足够清楚"]
  Q --> Q7["失败时用户如何理解原因和下一步"]

  Q1 --> P1["消息队列与 Steering 体验"]
  Q2 --> P2["Tree / Fork / Export / Share"]
  Q3 --> P3["Diff、命令输出、测试结果"]
  Q4 --> P4["Footer、设置、模型选择器"]
  Q5 --> P5["Packages、Skills、Prompt Templates"]
  Q6 --> P6["Onboarding、Auth Guidance"]
  Q7 --> P7["Error Message、Retry、Recovery"]
```

## 8. 产品风险地图

```mermaid
flowchart LR
  Risk["产品风险"] --> R1["模型不可用"]
  Risk --> R2["认证失败"]
  Risk --> R3["工具改错文件"]
  Risk --> R4["上下文太长"]
  Risk --> R5["终端显示复杂"]
  Risk --> R6["扩展冲突"]
  Risk --> R7["用户不知道当前状态"]

  R1 --> M1["模型回退提示 / list models"]
  R2 --> M2["登录引导 / API key 提示"]
  R3 --> M3["diff 可视化 / session 追踪"]
  R4 --> M4["自动 compact / context footer"]
  R5 --> M5["TUI 折叠 / 展开工具结果"]
  R6 --> M6["资源 diagnostics / sourceInfo"]
  R7 --> M7["footer / spinner / pending queue"]
```

## 9. 产品理解小结

Pi 的业务不是单一聊天，而是一个开发任务执行平台：

- 用户输入目标。
- 模型做决策。
- 工具执行真实动作。
- 会话记录全过程。
- 扩展和资源让团队定制工作方式。
- 多种运行模式让它既能当产品，也能当基础设施。

后续文档会分别展开这些业务线。
