# 04 工具协作与结果呈现

本文解释 Pi 的工具体系如何把“模型的想法”变成“对项目的真实操作”。重点不是技术实现，而是用户能感知到哪些能力、风险和结果。

## 1. 工具在业务里的角色

```mermaid
flowchart TD
  M["模型<br/>负责判断下一步做什么"] --> R["请求工具调用"]
  R --> A["Agent<br/>负责校验、执行、记录"]
  A --> T["工具<br/>真实读取/修改/执行"]
  T --> W["工作区<br/>代码、文档、配置、测试"]
  W --> O["工具结果"]
  O --> A
  A --> M
  M --> U["用户看到解释和总结"]
```

模型不是直接改文件。它提出工具请求，Agent 执行工具，再把结果返回给模型。

## 2. 内置工具能力地图

```mermaid
mindmap
  root((内置工具))
    读取理解
      read
        文本文件
        图片
        截断
        offset/limit
      ls
        目录结构
      grep
        文本搜索
      find
        文件查找
    行动修改
      edit
        局部替换
        diff
        patch
      write
        新建文件
        完整覆盖
    验证探索
      bash
        跑测试
        查环境
        执行脚本
        查看命令输出
```

## 3. 工具选择逻辑

```mermaid
flowchart TD
  A["模型判断需要行动"] --> B{"目标是什么"}
  B -- "看文件内容" --> Read["read"]
  B -- "看目录" --> Ls["ls"]
  B -- "找关键词" --> Grep["grep"]
  B -- "找文件" --> Find["find"]
  B -- "局部改代码" --> Edit["edit"]
  B -- "新建/完整重写" --> Write["write"]
  B -- "运行命令/测试" --> Bash["bash"]

  Read --> R["工具结果回到模型"]
  Ls --> R
  Grep --> R
  Find --> R
  Edit --> R
  Write --> R
  Bash --> R
```

产品上可以把工具理解为 Agent 的“手脚”。模型是脑，工具是执行能力。

## 4. Read 工具旅程

```mermaid
flowchart TD
  A["模型请求 read(path)"] --> B["解析路径"]
  B --> C{"文件类型"}
  C -- "文本" --> D["读取文本"]
  C -- "图片" --> E["读取图片并可自动缩放"]
  D --> F["按行数/字节数截断"]
  E --> G{"当前模型支持图片吗"}
  G -- "支持" --> H["图片作为附件进入上下文"]
  G -- "不支持" --> I["给出图片被省略提示"]
  F --> J["返回文本结果"]
  H --> K["返回图片结果"]
  I --> K
  J --> L["UI 展示，可折叠/展开"]
  K --> L
```

用户价值：

- 模型可以主动查项目文件。
- 大文件不会一次性塞爆上下文。
- 图片可参与理解，但会受模型能力影响。

## 5. Bash 工具旅程

```mermaid
sequenceDiagram
  participant M as 模型
  participant A as Agent
  participant B as bash 工具
  participant OS as 本地 Shell
  participant UI as 终端界面

  M->>A: 请求执行 command
  A->>B: 校验参数和 timeout
  B->>OS: spawn shell
  OS-->>B: stdout/stderr 流式输出
  B-->>UI: 持续刷新部分结果
  OS-->>B: 进程结束或超时
  B-->>A: exitCode + 输出 + 截断信息
  A-->>M: toolResult
```

产品可见点：

- 命令不是黑盒，会流式显示。
- 超长输出会截断，但保留完整输出路径。
- 超时或用户中断会杀掉进程树。

## 6. Edit 与 Write 的区别

```mermaid
flowchart LR
  A["要修改文件"] --> B{"修改方式"}
  B -- "只改一小段" --> E["edit<br/>oldText -> newText"]
  B -- "新建文件" --> W["write<br/>完整内容"]
  B -- "整个文件重写" --> W

  E --> E1["要求 oldText 唯一"]
  E --> E2["生成 diff"]
  E --> E3["保留原文件大部分内容"]

  W --> W1["创建父目录"]
  W --> W2["覆盖完整文件"]
  W --> W3["适合新文件或完整替换"]
```

产品解释：

- `edit` 更适合低风险局部修改。
- `write` 更适合明确的新文件或完整重写。
- UI 应强调 diff，让用户知道改了哪里。

## 7. 文件写入防冲突

```mermaid
flowchart TD
  A["同一轮可能有多个工具并行"] --> B["工具 A 写 file.ts"]
  A --> C["工具 B 也写 file.ts"]
  B --> Q["file.ts mutation queue"]
  C --> Q
  Q --> D["先执行 A"]
  D --> E["A 完成后执行 B"]
  E --> F["避免同时写同一个文件"]

  A --> G["工具 C 写 other.ts"]
  G --> H["不同文件可并行"]
```

这是一个重要的业务安全设计：既追求速度，又避免同文件并发写坏。

## 8. 工具结果如何被用户理解

```mermaid
flowchart TD
  A["工具结果"] --> B{"结果类型"}
  B -- "read 文本" --> C["代码高亮 / 截断提示"]
  B -- "bash 输出" --> D["流式输出 / exit code / timeout"]
  B -- "edit" --> E["diff / patch / 首个变更行"]
  B -- "write" --> F["写入内容预览 / 错误提示"]
  B -- "图片" --> G["终端图片或 fallback"]
  B -- "扩展工具" --> H["自定义渲染"]

  C --> UI["TUI 展示"]
  D --> UI
  E --> UI
  F --> UI
  G --> UI
  H --> UI
```

产品上最重要的是“工具结果可解释”。如果用户看不懂工具干了什么，就很难信任 Agent。

## 9. 工具执行状态

```mermaid
stateDiagram-v2
  [*] --> Requested: 模型请求工具
  Requested --> Validating: 校验参数
  Validating --> Blocked: 扩展阻止或参数错误
  Validating --> Running: 开始执行
  Running --> Streaming: 有部分输出
  Streaming --> Running: 继续执行
  Running --> Succeeded: 成功
  Running --> Failed: 失败
  Running --> Aborted: 用户中断
  Blocked --> ResultMessage
  Succeeded --> ResultMessage
  Failed --> ResultMessage
  Aborted --> ResultMessage
  ResultMessage --> [*]: 返回模型
```

## 10. 产品风险和缓解

```mermaid
flowchart TD
  Risk["工具相关风险"] --> R1["读不到文件"]
  Risk --> R2["命令跑太久"]
  Risk --> R3["输出太长"]
  Risk --> R4["改错位置"]
  Risk --> R5["并发写冲突"]
  Risk --> R6["工具结果用户看不懂"]

  R1 --> M1["清晰错误信息"]
  R2 --> M2["timeout / abort"]
  R3 --> M3["截断 + 完整输出路径"]
  R4 --> M4["edit 唯一匹配 + diff"]
  R5 --> M5["文件 mutation queue"]
  R6 --> M6["高亮 / 折叠 / 展开 / 摘要"]
```

## 11. 工具与模型的关系

```mermaid
flowchart LR
  Model["模型能力"] --> Need["决定工具需求"]
  Need --> Tool["工具能力"]
  Tool --> Evidence["产生证据"]
  Evidence --> Model
  Model --> Answer["最终回答"]

  Tool -. "能力不足会导致" .-> Gap["任务完成率下降"]
  Model -. "规划差会导致" .-> Waste["工具轮数变多"]
  Evidence -. "呈现差会导致" .-> Trust["用户信任下降"]
```

产品要同时看三件事：

- 工具够不够。
- 模型会不会用。
- 用户看不看得懂。

## 12. 可挖掘的工具体验方向

```mermaid
mindmap
  root((工具体验机会))
    read
      大文件导航
      图片解释提示
      项目文档折叠
    bash
      测试结果摘要
      长命令风险提示
      超时建议
    edit
      diff 更友好
      变更影响范围
      一键打开文件
    write
      新文件归类
      覆盖风险提示
    搜索
      结果聚类
      相关性排序
    扩展工具
      统一渲染规范
      权限与来源展示
```

## 13. 小结

工具体系是 Pi 从“聊天机器人”变成“开发助手”的关键。产品上要关注：

- 工具调用是否透明。
- 风险是否可理解。
- 修改是否可追溯。
- 结果是否能被模型继续利用。
- 用户是否能看懂工具帮自己做了什么。
