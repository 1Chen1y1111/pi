# 02 用户旅程与模式选择

本文用产品经理视角解释用户从启动 Pi 到完成任务的完整旅程，以及系统如何在 interactive、print、JSON、RPC、SDK 之间选择不同产品形态。

## 1. 用户旅程总览

```mermaid
journey
  title Pi 用户完成一次开发任务的旅程
  section 启动
    打开终端: 4: 用户
    进入项目目录: 4: 用户
    输入 pi: 5: 用户
  section 准备
    Pi 加载配置和项目规范: 3: 系统
    Pi 检查模型和认证: 3: 系统
    用户必要时登录或选择模型: 3: 用户
  section 执行
    用户描述任务: 5: 用户
    Pi 读取文件和运行工具: 4: 系统
    用户观察输出并可中途纠偏: 4: 用户
  section 收尾
    Pi 总结修改: 4: 系统
    会话自动保存: 5: 系统
    用户后续 resume 或 fork: 4: 用户
```

## 2. 启动时系统做了什么

```mermaid
flowchart TD
  A["用户输入 pi"] --> B["进入 CLI 入口"]
  B --> C["解析参数"]
  C --> D{"是否是特殊命令"}
  D -- "包管理/配置/版本/导出" --> E["直接执行命令并退出"]
  D -- "普通启动" --> F["运行迁移"]
  F --> G["读取启动 settings"]
  G --> H["选择或创建 session"]
  H --> I{"session cwd 是否有效"}
  I -- "无效" --> J["交互模式提示用户选择<br/>非交互模式报错"]
  I -- "有效" --> K["按最终 cwd 创建 runtime"]
  K --> L["加载资源、模型、认证"]
  L --> M["读取 stdin 和文件参数"]
  M --> N["进入运行模式"]
```

产品解释：

- Pi 不是一启动就问模型。
- 它先确认“在哪个项目工作”“用哪段历史”“有什么规则”“能不能调用模型”。
- 这些准备决定后面任务是否顺畅。

## 3. 模式选择逻辑

```mermaid
flowchart TD
  A["启动参数 + stdin 状态"] --> B{"--mode rpc?"}
  B -- "是" --> RPC["RPC 模式<br/>给外部应用集成"]
  B -- "否" --> C{"--mode json?"}
  C -- "是" --> JSON["JSON 模式<br/>输出事件流"]
  C -- "否" --> D{"-p 或 stdin 非 TTY?"}
  D -- "是" --> PRINT["Print 模式<br/>一次性执行并输出结果"]
  D -- "否" --> INTERACTIVE["Interactive 模式<br/>终端实时协作"]
```

同一个核心能力，会包装成不同产品形态：

| 模式 | 用户是谁 | 典型场景 | 输出 |
| --- | --- | --- | --- |
| Interactive | 终端用户 | 日常开发协作 | TUI 界面 |
| Print | 脚本用户 | 一次性问答或批处理 | 最终文本 |
| JSON | 自动化用户 | 需要机器读取事件 | JSON 行 |
| RPC | 宿主产品 | IDE/桌面应用/平台集成 | 双向 JSONL |
| SDK | 开发者 | 直接嵌入应用 | 程序 API |

## 4. 首次使用旅程

```mermaid
sequenceDiagram
  actor U as 用户
  participant CLI as Pi CLI
  participant S as Settings
  participant A as Auth
  participant M as ModelRegistry
  participant UI as TUI

  U->>CLI: 输入 pi
  CLI->>S: 读取全局/项目设置
  CLI->>A: 检查是否有认证
  CLI->>M: 查询可用模型
  alt 没有可用模型
    M-->>CLI: 无模型可用
    CLI->>UI: 展示登录/API key 引导
    U->>UI: /login 或配置 API key
  else 有可用模型
    M-->>CLI: 返回默认模型
    CLI->>UI: 进入可用状态
  end
  U->>UI: 输入第一个开发任务
```

首次使用的关键产品点：

- 用户最怕“不知道为什么不能用”。
- 所以认证失败、无模型、模型恢复失败都需要清楚提示。
- `/login`、`/model`、`/settings` 是首次体验里的关键入口。

## 5. 日常开发旅程

```mermaid
flowchart TD
  A["打开项目"] --> B["pi"]
  B --> C["看到启动信息<br/>模型、快捷键、加载资源"]
  C --> D["输入任务"]
  D --> E["Pi 开始工作"]
  E --> F{"用户是否满意当前方向"}
  F -- "满意" --> G["等待结果"]
  F -- "需要纠偏" --> H["Enter 发送 steering<br/>本轮工具后插入"]
  F -- "想到后续任务" --> I["Alt+Enter 发送 follow-up<br/>当前任务结束后执行"]
  G --> J["查看总结和 diff/命令输出"]
  H --> E
  I --> K["自动进入后续任务"]
  J --> L["会话保存，可 resume"]
  K --> L
```

这里的产品亮点是：用户不需要等模型完全结束才能表达新想法。

## 6. 消息队列体验

```mermaid
stateDiagram-v2
  [*] --> Idle: 等待输入
  Idle --> Running: 用户提交任务
  Running --> SteeringQueued: Enter 输入纠偏消息
  Running --> FollowUpQueued: Alt+Enter 输入后续消息
  SteeringQueued --> Running: 当前工具批次结束后插入
  FollowUpQueued --> Running: 当前任务自然结束后插入
  Running --> Idle: 没有工具调用和队列消息
  Running --> Aborted: Escape 中断
  Aborted --> Idle: 恢复可输入
```

产品解释：

- Steering 是“现在方向不对，下一步先听我这句”。
- Follow-up 是“你先把当前事做完，之后再做这个”。
- 这是 Pi 相比普通一次性命令的重要体验差异。

## 7. 恢复历史旅程

```mermaid
flowchart TD
  A["用户重新打开项目"] --> B{"想继续上次工作吗"}
  B -- "直接继续最近一次" --> C["pi -c"]
  B -- "浏览历史" --> D["pi -r 或 /resume"]
  B -- "指定某个会话" --> E["pi --session <id/path>"]
  C --> F["SessionManager 打开最近会话"]
  D --> G["Session selector 展示历史"]
  E --> H["按 id/path 定位会话"]
  F --> I["重建上下文"]
  G --> I
  H --> I
  I --> J["用户继续输入新任务"]
```

值得挖掘的体验问题：

- session 列表如何帮助用户找到“那次工作”？
- 是否需要按项目、时间、模型、首条消息、标签筛选？
- 恢复后是否应该更明显地告诉用户当前处在哪个历史分支？

## 8. 分支探索旅程

```mermaid
flowchart TD
  A["当前会话"] --> B["用户打开 /tree"]
  B --> C["看到历史树"]
  C --> D{"用户选择哪个节点"}
  D -- "回到早期用户需求" --> E["从旧节点继续"]
  D -- "选择某个实验分支" --> F["切换到该分支"]
  D -- "fork 成新会话" --> G["复制路径到新 session"]
  E --> H["新输入形成新分支"]
  F --> H
  G --> I["新 session 独立推进"]
  H --> J["旧历史仍保留"]
  I --> J
```

这对产品的意义：

- Pi 不只是线性聊天。
- 它支持“回到过去重新试一条路”。
- 适合探索式开发，比如“方案 A 不行，回到前面试方案 B”。

## 9. 登录和模型选择旅程

```mermaid
flowchart TD
  A["用户需要模型"] --> B{"认证方式"}
  B -- "订阅账号" --> C["/login<br/>OAuth 登录"]
  B -- "API key" --> D["环境变量或 auth 配置"]
  B -- "自定义 Provider" --> E["models.json 或扩展注册"]

  C --> F["保存 OAuth 凭据"]
  D --> G["保存或读取 API key"]
  E --> H["注册模型和认证规则"]

  F --> I["ModelRegistry 得到可用模型"]
  G --> I
  H --> I
  I --> J["/model 选择模型"]
  J --> K["开始任务"]
```

产品上需要解释清楚：

- “登录”不等于所有 Provider 都可用。
- 不同 Provider 有不同认证来源。
- 模型选择应显示 provider、能力、thinking 支持、上下文、成本等关键差异。

## 10. 用户可感知状态

```mermaid
flowchart LR
  State["Pi 当前状态"] --> S1["是否正在工作"]
  State --> S2["当前模型"]
  State --> S3["Thinking level"]
  State --> S4["上下文使用量"]
  State --> S5["Token / Cost"]
  State --> S6["当前 cwd"]
  State --> S7["Session 名称"]
  State --> S8["队列消息"]
  State --> S9["工具调用是否折叠"]

  S1 --> UI1["Loader / working message"]
  S2 --> UI2["Footer / Model selector"]
  S3 --> UI3["Editor border / Footer"]
  S4 --> UI4["Footer context usage"]
  S5 --> UI5["Footer token/cost"]
  S8 --> UI8["Pending messages area"]
```

产品判断标准：用户应该随时知道“它在干什么、为什么卡住、下一步会做什么、我还能不能输入”。

## 11. 模式之间的产品边界

```mermaid
flowchart TB
  Core["同一套 Agent 核心"] --> I["Interactive"]
  Core --> P["Print"]
  Core --> J["JSON"]
  Core --> R["RPC"]
  Core --> SDK["SDK"]

  I --> I1["用户实时控制"]
  I --> I2["TUI 可视化"]
  I --> I3["扩展 UI 完整能力"]

  P --> P1["简单脚本"]
  P --> P2["只关心最终文本"]

  J --> J1["自动化系统"]
  J --> J2["需要事件流"]

  R --> R1["外部产品集成"]
  R --> R2["宿主管 UI"]

  SDK --> S1["开发者直接嵌入"]
  SDK --> S2["自定义 runtime"]
```

这意味着同一个底层能力可以包装成多个产品 SKU。

## 12. 产品机会点

```mermaid
mindmap
  root((用户旅程机会点))
    首次启动
      更清晰的认证引导
      推荐默认模型
      解释 API Key 与 OAuth 区别
    执行中
      展示下一步计划
      工具调用风险提示
      队列消息更可编辑
    恢复历史
      会话命名
      搜索和标签
      分支地图可视化
    失败恢复
      失败原因分类
      一键重试
      切换模型重试
    自动化
      JSON schema 稳定性
      RPC 宿主 UI 示例
      批处理模板
```

## 13. 小结

从用户旅程看，Pi 的关键产品体验不是“问答”，而是：

- 能进入真实项目。
- 能理解已有规则和历史。
- 能持续执行多步任务。
- 用户能中途纠偏。
- 工作过程能保存、恢复、分支和复用。

后续可以围绕“首次成功率、任务完成率、中途可控性、历史复用率、失败恢复率”设计产品指标。
