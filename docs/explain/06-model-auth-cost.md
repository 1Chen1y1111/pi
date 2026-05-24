# 06 模型、认证、成本与失败处理

本文解释 Pi 如何选择模型、处理认证、控制 thinking level，并把模型请求失败转化为用户能理解的状态。

## 1. 模型在业务里的位置

```mermaid
flowchart TD
  U["用户任务"] --> A["Pi Agent"]
  A --> MR["ModelRegistry<br/>可用模型和认证"]
  MR --> M["具体模型"]
  M --> P["Provider<br/>模型服务来源"]
  P --> API["Provider API"]
  API --> R["流式回复"]
  R --> A
  A --> T["工具调用/最终结果"]
```

模型负责理解和决策，但 Pi 负责：

- 找到可用模型。
- 给请求加认证。
- 把不同 Provider 的协议统一。
- 展示成本、上下文、thinking 等状态。

## 2. Provider、Model、API 的关系

```mermaid
erDiagram
  PROVIDER ||--o{ MODEL : "提供"
  MODEL }o--|| API_TYPE : "使用"
  PROVIDER ||--o{ AUTH_METHOD : "支持"
  MODEL ||--o{ CAPABILITY : "具备"
  MODEL ||--o{ COST_RULE : "计费"

  PROVIDER {
    string id "openai/anthropic/google..."
    string displayName "展示名"
  }
  MODEL {
    string id "模型ID"
    string api "协议类型"
    number contextWindow "上下文窗口"
    number maxTokens "最大输出"
  }
  API_TYPE {
    string name "openai-responses/anthropic-messages..."
  }
  AUTH_METHOD {
    string type "OAuth/API Key/headers"
  }
  CAPABILITY {
    string input "text/image"
    boolean reasoning "是否支持推理"
  }
```

产品解释：

- Provider 是“厂商或服务入口”。
- Model 是“具体可选的大脑”。
- API 是“Pi 和这个模型说话的协议”。
- Auth 是“能否使用这个服务的凭证”。

## 3. 模型来源

```mermaid
flowchart TD
  A["模型列表"] --> B["内置生成模型表"]
  A --> C["用户 models.json"]
  A --> D["Provider override"]
  A --> E["Model override"]
  A --> F["扩展动态注册 Provider"]
  A --> G["OAuth 修改模型信息"]

  B --> H["ModelRegistry 合并"]
  C --> H
  D --> H
  E --> H
  F --> H
  G --> H
  H --> I["可展示/可选择模型"]
```

这意味着产品上模型不是固定列表，而是可由用户和扩展动态改变。

## 4. 认证来源

```mermaid
flowchart TD
  A["模型请求需要认证"] --> B{"认证来源"}
  B -- "AuthStorage" --> C["已保存 API key 或 OAuth"]
  B -- "models.json" --> D["provider/model apiKey"]
  B -- "环境变量" --> E["ANTHROPIC_API_KEY 等"]
  B -- "命令动态取值" --> F["command-backed config"]
  B -- "扩展 OAuth" --> G["自定义 OAuth Provider"]

  C --> H["ModelRegistry 解析"]
  D --> H
  E --> H
  F --> H
  G --> H
  H --> I{"是否拿到凭证"}
  I -- "是" --> J["发送 Provider 请求"]
  I -- "否" --> K["展示登录/API key 引导"]
```

## 5. 用户登录旅程

```mermaid
sequenceDiagram
  actor U as 用户
  participant UI as Pi UI
  participant OAuth as OAuth Provider
  participant Auth as AuthStorage
  participant MR as ModelRegistry

  U->>UI: 输入 /login
  UI->>MR: 查询支持登录的 Provider
  UI-->>U: 展示 Provider 列表
  U->>UI: 选择 Provider
  UI->>OAuth: 发起 OAuth/device flow
  OAuth-->>U: 浏览器或设备码确认
  OAuth-->>UI: 返回凭据
  UI->>Auth: 保存凭据
  Auth-->>MR: 后续可解析 token
  MR-->>UI: 模型变为可用
```

产品关键点：

- 登录成功后，用户应该立即知道哪些模型变可用了。
- 订阅登录和 API key 使用成本可能不同，文案要明确。

## 6. 模型选择流程

```mermaid
flowchart TD
  A["创建 session"] --> B{"是否从旧 session 恢复模型"}
  B -- "能恢复且有认证" --> C["使用旧模型"]
  B -- "不能恢复" --> D["记录 fallback 提示"]
  B -- "新 session" --> E["读取默认 provider/model"]

  D --> F["寻找初始模型"]
  E --> F
  F --> G{"是否配置 enabledModels / --models"}
  G -- "是" --> H["从 scoped models 中选"]
  G -- "否" --> I["用 settings 默认或 provider 默认"]

  C --> J["进入会话"]
  H --> J
  I --> J
  J --> K["用户可 /model 切换"]
```

## 7. Scoped Models 的业务含义

```mermaid
flowchart LR
  A["全部模型很多"] --> B["用户配置 enabledModels 或 --models"]
  B --> C["形成模型范围 scopedModels"]
  C --> D["Ctrl+P 快速循环"]
  C --> E["/scoped-models 管理"]
  D --> F["高频切换少数常用模型"]
```

产品解释：

- 全部模型列表适合搜索。
- scoped models 适合日常快速切换。
- thinking level 也可以跟随 scoped model pattern。

## 8. Thinking Level

```mermaid
flowchart TD
  A["用户选择 thinking level"] --> B{"模型是否支持 reasoning"}
  B -- "不支持" --> C["强制 off"]
  B -- "支持" --> D{"请求级别是否支持"}
  D -- "支持" --> E["使用请求级别"]
  D -- "不支持" --> F["clamp 到最近可用级别"]
  E --> G["发送给 Provider"]
  F --> G
```

产品解释：

- Thinking level 是用户对“模型思考强度”的控制。
- 不是所有模型都支持。
- `xhigh` 只有部分模型支持。
- UI 需要告诉用户实际生效的级别，而不只是用户请求的级别。

## 9. 上下文与成本

```mermaid
flowchart TD
  A["一次模型请求"] --> B["输入 token"]
  A --> C["输出 token"]
  A --> D["cache read"]
  A --> E["cache write"]

  B --> F["成本"]
  C --> F
  D --> F
  E --> F

  A --> G["上下文窗口使用率"]
  G --> H{"是否接近上限"}
  H -- "否" --> I["正常继续"]
  H -- "是" --> J["触发 compact 或提示风险"]
```

产品上，footer 里的 token/cost/context 不是技术噪音，而是用户掌控感来源。

## 10. Provider 请求链路

```mermaid
sequenceDiagram
  participant AS as AgentSession
  participant MR as ModelRegistry
  participant EX as ExtensionRunner
  participant AI as pi-ai
  participant P as Provider API

  AS->>MR: getApiKeyAndHeaders(model)
  MR-->>AS: apiKey + headers
  AS->>EX: before_provider_request
  EX-->>AS: 可修改 payload
  AS->>AI: streamSimple(model, context, options)
  AI->>P: Provider 请求
  P-->>AI: HTTP response + stream
  AI-->>AS: 标准化流式事件
  AS->>EX: after_provider_response
```

## 11. 失败类型

```mermaid
flowchart TD
  A["模型请求失败"] --> B{"失败原因"}
  B -- "无认证" --> C["No API key / 需要 login"]
  B -- "OAuth 过期" --> D["提示重新 /login"]
  B -- "网络/超时" --> E["可重试"]
  B -- "Provider 限流" --> F["根据 retry delay 判断"]
  B -- "模型不存在" --> G["fallback 或提示切换模型"]
  B -- "上下文超限" --> H["compact / overflow recovery"]
  B -- "用户中断" --> I["aborted"]

  C --> UI["用户可理解错误"]
  D --> UI
  E --> UI
  F --> UI
  G --> UI
  H --> UI
  I --> UI
```

## 12. 自动重试

```mermaid
stateDiagram-v2
  [*] --> Requesting: 发送模型请求
  Requesting --> Success: 正常完成
  Requesting --> Error: 请求失败
  Error --> Retryable: 满足重试条件
  Error --> FinalError: 不可重试
  Retryable --> Waiting: 指数退避
  Waiting --> Requesting: 再次 continue
  Retryable --> FinalError: 超过最大次数
  Success --> [*]
  FinalError --> [*]
```

产品重点：

- 自动重试不应该让用户困惑。
- UI 应说明“正在重试第几次”。
- 最终失败要说明重试已经结束。

## 13. 模型产品机会点

```mermaid
mindmap
  root((模型体验机会))
    选择
      推荐模型
      最近使用
      按任务类型推荐
      scoped models
    解释
      Provider 差异
      上下文窗口
      图片能力
      reasoning 能力
    成本
      单轮成本
      session 累计成本
      预算提醒
      cache 命中解释
    认证
      登录状态
      过期提醒
      API key 检查
      多账号切换
    失败
      自动重试
      切模型重试
      compact 建议
```

## 14. 小结

模型系统对产品的影响非常大：

- 决定任务能力上限。
- 决定用户是否能成功启动。
- 决定速度、成本、上下文长度。
- 决定失败恢复体验。

产品上要避免把“模型选择”和“认证配置”做成纯技术设置，它们实际是核心用户旅程的一部分。
