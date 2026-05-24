# 03 任务执行主循环

本文解释用户输入一句需求后，Pi 如何一步步变成真实操作：模型思考、调用工具、读取结果、继续推进，直到任务完成。

## 1. 一句话任务为什么会变成多步行动

用户通常只说目标，例如“帮我修这个 bug”。Pi 必须自己补齐中间步骤：

```mermaid
flowchart TD
  A["用户目标<br/>帮我修 bug"] --> B["理解项目上下文"]
  B --> C["定位相关文件"]
  C --> D["读取代码"]
  D --> E["推断原因"]
  E --> F["修改代码"]
  F --> G["运行检查或测试"]
  G --> H{"是否通过"}
  H -- "否" --> I["继续分析失败输出"]
  I --> F
  H -- "是" --> J["总结修改和验证结果"]
```

这个循环就是 Agent 的业务核心。

## 2. 主执行链路

```mermaid
sequenceDiagram
  actor U as 用户
  participant AS as AgentSession
  participant A as Agent
  participant L as AgentLoop
  participant M as 模型
  participant T as 工具
  participant S as Session

  U->>AS: 输入任务
  AS->>AS: 展开技能/模板/扩展输入
  AS->>A: prompt(messages)
  A->>L: runAgentLoop
  L->>S: 记录用户消息
  L->>M: 发送上下文和工具列表
  M-->>L: 流式返回文本/工具调用
  alt 模型请求工具
    L->>T: 执行工具
    T-->>L: 返回工具结果
    L->>S: 记录工具结果
    L->>M: 带工具结果继续请求
  else 模型直接完成
    M-->>L: 最终回答
  end
  L->>S: 记录助手消息
  AS-->>U: 展示结果
```

## 3. Agent 的状态变化

```mermaid
stateDiagram-v2
  [*] --> Idle: 等待用户
  Idle --> Preflight: 用户提交任务
  Preflight --> Streaming: 模型开始响应
  Streaming --> ToolRequested: 模型请求工具
  ToolRequested --> ToolRunning: 工具执行
  ToolRunning --> ToolResultReady: 工具返回结果
  ToolResultReady --> Streaming: 继续请求模型
  Streaming --> Completed: 无更多工具调用
  Streaming --> Error: 模型或请求失败
  ToolRunning --> Aborted: 用户中断
  Completed --> Idle: 保存并展示
  Error --> RetryWaiting: 满足自动重试条件
  RetryWaiting --> Streaming: 继续尝试
  Error --> Idle: 不再重试
  Aborted --> Idle: 恢复输入
```

用户在界面看到的是：

- 模型正在输出。
- 工具正在执行。
- 命令输出不断刷新。
- 工具结果可折叠/展开。
- 结束后回到可输入状态。

## 4. 消息如何进入模型

Pi 内部有一些消息只给 UI 或 session 用，不应该直接给模型。进入模型前会做转换。

```mermaid
flowchart TD
  A["AgentMessage[]<br/>包含标准消息和 Pi 自定义消息"] --> B["transformContext<br/>扩展可裁剪/注入上下文"]
  B --> C["convertToLlm<br/>转成模型认识的消息"]
  C --> D["Message[]<br/>user / assistant / toolResult"]
  D --> E["Context<br/>systemPrompt + messages + tools"]
  E --> F["Provider API"]

  A --> A1["bashExecution"]
  A --> A2["custom"]
  A --> A3["branchSummary"]
  A --> A4["compactionSummary"]

  C --> C1["bashExecution -> user 文本"]
  C --> C2["custom -> user 消息"]
  C --> C3["summary -> user 摘要"]
```

产品解释：

- 用户看到的“工作记录”和模型看到的“上下文”不是完全一样。
- Pi 会把工作记录整理成模型能理解的材料。
- 这让系统既能保存完整过程，又能控制模型输入质量。

## 5. 为什么需要系统提示词

系统提示词相当于“给模型的工作规则”。它由多种资源拼出来：

```mermaid
flowchart TD
  A["基础系统提示词"] --> P["最终 systemPrompt"]
  B["项目 AGENTS.md / CLAUDE.md"] --> P
  C["已加载 Skills"] --> P
  D["启用工具说明"] --> P
  E["工具使用准则"] --> P
  F["append system prompt"] --> P
  G["扩展 before_agent_start 修改"] --> P

  P --> M["模型每轮都看到这些规则"]
```

这就是为什么同样一句用户需求，在不同项目里 Pi 的行为会不同。

## 6. 工具调用循环

```mermaid
flowchart TD
  A["模型输出"] --> B{"是否包含 toolCall"}
  B -- "否" --> C["本轮完成"]
  B -- "是" --> D["Agent Core 找到工具"]
  D --> E["校验参数"]
  E --> F{"扩展是否拦截"}
  F -- "阻止" --> G["返回错误工具结果"]
  F -- "允许" --> H["执行工具"]
  H --> I["工具返回内容和 details"]
  I --> J["生成 toolResult message"]
  J --> K["把 toolResult 送回模型"]
  K --> A
```

产品上可以理解为：模型不是直接操作电脑，而是“申请调用工具”，Agent 负责执行和记录。

## 7. 并行工具与顺序工具

```mermaid
flowchart TD
  A["一条助手消息里有多个工具调用"] --> B{"是否需要顺序执行"}
  B -- "全局 sequential" --> S["按顺序执行"]
  B -- "某个工具声明 sequential" --> S
  B -- "默认 parallel" --> P["先顺序 preflight<br/>再并行执行"]

  P --> P1["工具 A 执行"]
  P --> P2["工具 B 执行"]
  P --> P3["工具 C 执行"]

  P1 --> R["按原始工具调用顺序写回结果"]
  P2 --> R
  P3 --> R

  S --> S1["执行 A"]
  S1 --> S2["执行 B"]
  S2 --> S3["执行 C"]
  S3 --> R
```

为什么产品要关心：

- 并行能提升速度。
- 顺序能降低冲突风险。
- 文件写入工具还会对同一文件加队列，避免同时写坏文件。

## 8. 中途纠偏如何插入

```mermaid
sequenceDiagram
  actor U as 用户
  participant A as Agent
  participant M as 模型
  participant T as 工具

  U->>A: 提交任务 A
  A->>M: 请求模型
  M-->>A: 请求工具 read/edit/bash
  A->>T: 执行工具
  U->>A: 输入纠偏消息 steering
  Note over A: 不打断正在执行的工具
  T-->>A: 工具结果返回
  A->>A: 插入 steering 消息
  A->>M: 带工具结果和纠偏继续请求
```

产品解释：

- Steering 不会粗暴打断正在跑的工具。
- 它会在安全边界插入下一轮上下文。
- 用户感知是“我说的话很快会被听到”。

## 9. Follow-up 如何排队

```mermaid
flowchart TD
  A["Agent 正在执行任务 A"] --> B["用户输入 follow-up 任务 B"]
  B --> C["放入 follow-up 队列"]
  A --> D{"任务 A 是否自然结束"}
  D -- "还有工具调用" --> A
  D -- "还有 steering" --> A
  D -- "没有后续工作" --> E["取出 follow-up B"]
  E --> F["作为新用户消息执行"]
```

适合场景：

- “先修这个 bug，修完后再帮我写 changelog。”
- “你先跑完检查，之后总结风险。”

## 10. 失败处理路径

```mermaid
flowchart TD
  A["执行中失败"] --> B{"失败类型"}
  B -- "用户中断" --> C["stopReason: aborted"]
  B -- "模型/网络/认证/工具异常" --> D["stopReason: error"]

  C --> E["展示中断状态<br/>回到可输入"]
  D --> F{"是否满足自动重试"}
  F -- "是" --> G["等待退避时间"]
  G --> H["重新 continue"]
  H --> I{"重试成功"}
  I -- "是" --> J["发送 auto_retry_end success"]
  I -- "否" --> K["到达最大重试后展示最终错误"]
  F -- "否" --> K
```

产品需要把失败信息讲清楚：

- 是认证问题？
- 是模型请求失败？
- 是工具执行失败？
- 是用户主动取消？
- 是否已经自动重试？

## 11. 从产品指标看任务执行

```mermaid
flowchart LR
  A["任务执行体验"] --> M1["首次可用率"]
  A --> M2["任务完成率"]
  A --> M3["平均工具轮数"]
  A --> M4["用户纠偏次数"]
  A --> M5["自动重试成功率"]
  A --> M6["失败可理解率"]
  A --> M7["最终验证率"]

  M1 --> Q1["模型和认证是否顺畅"]
  M2 --> Q2["工具能力是否足够"]
  M3 --> Q3["模型规划是否高效"]
  M4 --> Q4["用户是否能掌控方向"]
  M5 --> Q5["失败恢复是否有效"]
  M6 --> Q6["错误文案是否清楚"]
  M7 --> Q7["是否运行测试/检查"]
```

## 12. 小结

Pi 的任务执行不是“模型直接给答案”，而是：

1. 用户给目标。
2. 系统整理上下文和规则。
3. 模型决定下一步。
4. 工具执行真实动作。
5. 结果回到模型。
6. 多轮循环直到完成或失败。

产品设计要重点关注：用户能否理解当前状态、能否中途影响方向、失败时能否知道下一步。
