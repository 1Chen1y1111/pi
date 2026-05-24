# 05 会话资产、分支与压缩

本文解释 Pi 为什么把一次开发过程当成“资产”保存，以及 resume、tree、fork、compact 等能力在业务上解决什么问题。

## 1. 会话是什么

会话不是普通聊天记录，而是一份完整工作日志：

```mermaid
flowchart TD
  S["Session 会话"] --> H["用户需求"]
  S --> A["助手回复"]
  S --> T["工具调用"]
  S --> R["工具结果"]
  S --> M["模型切换"]
  S --> L["Thinking 切换"]
  S --> C["压缩摘要"]
  S --> B["分支"]
  S --> X["扩展自定义状态"]

  H --> V["可恢复"]
  A --> V
  T --> V
  R --> V
  C --> V
  B --> V
```

产品语言：会话是“用户和 Pi 一起完成任务的过程资产”。

## 2. 为什么需要会话资产

```mermaid
flowchart LR
  Problem["开发任务特点"] --> P1["耗时长"]
  Problem --> P2["中途会失败"]
  Problem --> P3["需要回滚和尝试"]
  Problem --> P4["需要解释过程"]
  Problem --> P5["需要下次继续"]

  P1 --> S["Session"]
  P2 --> S
  P3 --> S
  P4 --> S
  P5 --> S

  S --> V1["恢复上下文"]
  S --> V2["保留证据"]
  S --> V3["支持分支"]
  S --> V4["导出分享"]
```

如果没有会话，用户每次都要重新解释背景，Agent 的工作也无法审计。

## 3. 会话生命周期

```mermaid
stateDiagram-v2
  [*] --> Created: 新建 session
  Created --> Active: 用户提交任务
  Active --> Saved: 产生助手消息后写入文件
  Saved --> Active: 用户继续输入
  Saved --> Resumed: 用户恢复会话
  Resumed --> Active: 继续工作
  Active --> Branched: 用户从历史节点继续
  Branched --> Active: 新分支推进
  Active --> Compacted: 上下文过长或手动压缩
  Compacted --> Active: 带摘要继续
  Saved --> Forked: fork 成新会话
  Forked --> Active: 新文件独立工作
```

## 4. 会话如何保存

Pi 使用 append-only JSONL。可以理解成“每发生一件事，就往日志末尾加一条记录”。

```mermaid
flowchart TD
  A["Session Header<br/>会话基本信息"] --> B["Entry 1<br/>用户消息"]
  B --> C["Entry 2<br/>助手消息"]
  C --> D["Entry 3<br/>工具结果"]
  D --> E["Entry 4<br/>模型切换"]
  E --> F["Entry 5<br/>后续用户消息"]

  A -. "JSONL 首行" .-> A1["id / cwd / timestamp"]
  B -. "每条记录都有" .-> B1["id / parentId / timestamp"]
```

append-only 的产品好处：

- 历史不被覆盖。
- 方便审计。
- 分支可以自然存在。
- 扩展可以追加自己的状态。

## 5. 树形历史

普通聊天是线性的，但 Pi 的会话是树。

```mermaid
flowchart TD
  A["用户: 修复登录 bug"] --> B["Pi: 读取 auth 文件"]
  B --> C["Pi: 方案 A 修改"]
  C --> D["测试失败"]
  B --> E["回到 B 后尝试方案 B"]
  E --> F["测试通过"]
  B --> G["fork 出独立会话探索方案 C"]

  D -. "保留失败路径" .-> Z["历史资产"]
  F -. "当前有效路径" .-> Z
  G -. "独立探索" .-> Z
```

这让用户可以：

- 回到某个历史节点重新开始。
- 保留失败尝试作为知识。
- 对比不同方案。
- 把某条路径 fork 成新会话。

## 6. Tree 功能的业务含义

```mermaid
flowchart TD
  A["用户打开 /tree"] --> B["系统展示历史树"]
  B --> C{"用户选择动作"}
  C -- "跳到旧节点" --> D["当前 leaf 移动到旧节点"]
  C -- "给节点打标签" --> E["记录 label"]
  C -- "过滤视图" --> F["只看用户消息/无工具/标签节点"]
  C -- "继续对话" --> G["下一条消息形成新分支"]

  D --> H["探索不同路径"]
  E --> I["沉淀关键节点"]
  F --> J["降低历史噪音"]
  G --> K["保留原分支不丢失"]
```

产品机会：

- 标签可以变成“里程碑”。
- 过滤可以帮助用户快速找到关键决策点。
- 分支可视化是高级用户理解 Agent 工作的核心。

## 7. Resume 逻辑

```mermaid
flowchart TD
  A["用户想继续历史"] --> B{"入口"}
  B -- "pi -c" --> C["当前项目最近 session"]
  B -- "pi -r" --> D["打开 session 选择器"]
  B -- "--session id/path" --> E["指定 session"]
  B -- "全局搜索 id" --> F["可能来自其他项目"]

  F --> G{"是否同 cwd"}
  G -- "是" --> H["直接打开"]
  G -- "否" --> I["交互模式询问是否 fork"]
  I -- "同意" --> J["fork 到当前项目"]
  I -- "取消" --> K["退出"]

  C --> L["重建上下文"]
  D --> L
  E --> L
  H --> L
  J --> L
```

为什么跨项目 session 要谨慎：

- 工作目录不同，文件路径可能失效。
- 项目 settings 和资源不同。
- 直接继续可能误操作另一个项目。

## 8. Fork 与 Clone

```mermaid
flowchart LR
  S["原 session"] --> F["fork"]
  S --> C["clone"]

  F --> F1["从某个历史用户消息开始"]
  F --> F2["可修改选中的 prompt"]
  F --> F3["新 session 文件"]

  C --> C1["复制当前活动路径"]
  C --> C2["打开空编辑器继续"]
  C --> C3["新 session 文件"]
```

产品区别：

- Fork 更像“从历史某个需求重新提问”。
- Clone 更像“把当前成功路径复制一份继续干别的”。

## 9. Compact 为什么存在

模型有上下文窗口限制。会话越长，能塞进模型的历史越多，但也越贵、越容易超限。

```mermaid
flowchart TD
  A["会话越来越长"] --> B["上下文 token 增加"]
  B --> C{"是否接近模型上限"}
  C -- "否" --> D["继续正常对话"]
  C -- "是" --> E["触发 compaction"]
  E --> F["把旧历史总结成摘要"]
  F --> G["保留近期关键消息"]
  G --> H["继续任务"]
```

Compact 的产品含义：

- 不是删除历史。
- 是把“模型需要看的旧内容”压缩成摘要。
- 原始 JSONL 仍保留完整记录。

## 10. Compact 后上下文怎么变

```mermaid
flowchart TD
  A["原始历史"] --> A1["消息 1"]
  A --> A2["消息 2"]
  A --> A3["消息 3"]
  A --> A4["消息 4"]
  A --> A5["近期消息 5"]
  A --> A6["近期消息 6"]

  A1 --> C["Compaction Summary<br/>旧历史摘要"]
  A2 --> C
  A3 --> C
  A4 --> C

  C --> N["新上下文"]
  A5 --> N
  A6 --> N
```

用户看到的是会话还在继续；模型看到的是“摘要 + 近期原文”。

## 11. 导出和分享

```mermaid
flowchart TD
  A["Session JSONL"] --> B["Export HTML"]
  B --> C["嵌入主题色"]
  B --> D["渲染消息"]
  B --> E["渲染工具调用"]
  B --> F["渲染 diff / 命令输出"]
  B --> G["浏览器可读页面"]
  G --> H["分享给同事/维护者"]
```

导出适合：

- 复盘 Agent 做了什么。
- 在 issue/PR 中解释过程。
- 分享给团队做案例学习。
- 作为 OSS session 数据集材料。

## 12. 会话资产的产品指标

```mermaid
flowchart LR
  A["会话资产质量"] --> M1["Resume 使用率"]
  A --> M2["Tree 使用率"]
  A --> M3["Fork/Clone 使用率"]
  A --> M4["Compact 成功率"]
  A --> M5["Export/Share 使用率"]
  A --> M6["Session 命名率"]
  A --> M7["标签使用率"]

  M1 --> Q1["历史是否真的有用"]
  M2 --> Q2["用户是否需要分支探索"]
  M3 --> Q3["工作路径是否可复用"]
  M4 --> Q4["长任务是否顺畅"]
  M5 --> Q5["过程是否值得分享"]
```

## 13. 机会点

```mermaid
mindmap
  root((会话资产机会))
    发现
      更强搜索
      自动命名
      项目分组
      标签体系
    理解
      分支图
      关键节点摘要
      工具噪音过滤
    复用
      从历史生成模板
      成功路径复制
      失败案例沉淀
    分享
      HTML 导出优化
      PR/Issue 链接
      团队知识库
    治理
      隐私清理
      大会话归档
      敏感信息提示
```

## 14. 小结

Pi 的会话系统把开发过程变成可恢复、可分支、可压缩、可分享的资产。对产品来说，最值得关注的是：

- 用户如何找到历史。
- 用户如何理解分支。
- 长任务如何不丢上下文。
- 成功经验如何被复用。
- 分享出去的过程是否清楚可信。
