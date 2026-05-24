# 模型与 Provider 架构设计

本文说明 Pi 如何统一不同 LLM Provider，以及 coding-agent 如何在此基础上处理用户模型配置、认证和扩展 Provider。阅读本文不需要先读总览文档。

## 分层

```text
packages/coding-agent ModelRegistry
  |
  +--> 用户配置、认证、OAuth、models.json、扩展 Provider
  |
  v
packages/ai streamSimple / completeSimple
  |
  +--> 统一 Context、Message、Tool、AssistantMessageEventStream
  |
  v
Provider implementation
  |
  +--> OpenAI / Anthropic / Google / Bedrock / Mistral / ...
```

`pi-ai` 不关心用户 settings 和 session；`ModelRegistry` 不实现 Provider wire protocol。

## pi-ai 核心类型

`packages/ai/src/types.ts` 定义统一协议：

- `Model<Api>`: 模型 id、provider、api、baseUrl、reasoning、input、cost、contextWindow、maxTokens、compat。
- `Context`: systemPrompt、messages、tools。
- `Message`: user、assistant、toolResult。
- `Tool`: name、description、TypeBox parameters。
- `AssistantMessageEventStream`: assistant 流式事件。
- `SimpleStreamOptions`: reasoning、thinking budgets、transport、apiKey、headers、timeout、retry 等。

统一事件包括：

- `start`
- `text_start` / `text_delta` / `text_end`
- `thinking_start` / `thinking_delta` / `thinking_end`
- `toolcall_start` / `toolcall_delta` / `toolcall_end`
- `done`
- `error`

## API Provider Registry

`packages/ai/src/api-registry.ts` 维护 `api -> provider` 映射。

Provider 结构：

- `api`
- `stream(model, context, options)`
- `streamSimple(model, context, options)`

调用路径：

```text
streamSimple(model, context, options)
  |
  +--> getApiProvider(model.api)
  |
  +--> provider.streamSimple(model, context, options)
```

如果 `model.api` 没有注册 Provider，会抛出错误。

## Built-in Providers

`packages/ai/src/providers/register-builtins.ts` 注册内置 API Provider：

- `anthropic-messages`
- `openai-completions`
- `mistral-conversations`
- `openai-responses`
- `azure-openai-responses`
- `openai-codex-responses`
- `google-generative-ai`
- `google-vertex`
- `bedrock-converse-stream`

Provider 模块用 lazy wrapper 延迟加载。这样 CLI 启动时不用立即加载所有 SDK。

lazy 加载失败时会生成 assistant error message，并通过 `AssistantMessageEventStream` 结束，遵守 Provider stream contract。

## 模型列表

`packages/ai/src/models.ts` 从 `models.generated.ts` 初始化内置模型 registry。

提供：

- `getModel(provider, modelId)`
- `getProviders()`
- `getModels(provider)`
- `calculateCost(model, usage)`
- `getSupportedThinkingLevels(model)`
- `clampThinkingLevel(model, level)`
- `modelsAreEqual(a, b)`

`models.generated.ts` 是生成文件，不应直接修改；应改生成脚本。

## Coding-agent ModelRegistry

`packages/coding-agent/src/core/model-registry.ts` 是应用层模型注册表。

职责：

- 加载 `pi-ai` 内置模型。
- 加载用户 `models.json`。
- 支持 provider override。
- 支持 model override。
- 合并 custom models。
- 根据 OAuth 凭据修改 models。
- 解析 API key 和 request headers。
- 暴露 available models。
- 允许扩展动态注册 Provider。

它比 `pi-ai/src/models.ts` 更高层，因为它处理用户和项目配置。

## models.json

`models.json` 可以：

- 给内置 provider 改 baseUrl、headers、compat。
- 给内置 model 做 override。
- 添加自定义 provider。
- 添加自定义 model。
- 定义 apiKey 来源。
- 定义 authHeader 行为。

custom provider 如果定义 models，通常需要：

- `baseUrl`
- `apiKey` 或 `oauth`
- `api`

内置 provider 的自定义模型可以继承内置 api 和 baseUrl。

## 认证解析

`ModelRegistry.getApiKeyAndHeaders(model)` 合并认证来源：

1. `AuthStorage` 中 provider 对应 auth。
2. `models.json` provider 级 `apiKey`。
3. provider headers。
4. model headers。
5. model 自带 headers。
6. 如果 `authHeader` 为 true，把 apiKey 写入 `Authorization: Bearer ...`。

API key 可以是：

- 已保存的 key。
- OAuth access token。
- 环境变量引用。
- command-backed config value。
- models.json 中的直接 key。

## Provider Request

`AgentSession` 创建 `Agent` 时会传入自定义 `streamFn`：

```text
streamFn(model, context, options)
  |
  +--> modelRegistry.getApiKeyAndHeaders(model)
  +--> settingsManager.getProviderRetrySettings()
  +--> getAttributionHeaders(...)
  +--> streamSimple(model, context, mergedOptions)
```

扩展可通过：

- `before_provider_request` 修改 payload。
- `after_provider_response` 观察 response status/headers。

## Thinking Level

Pi 暴露统一 thinking level：

- `off`
- `minimal`
- `low`
- `medium`
- `high`
- `xhigh`

`pi-ai` 根据模型能力和 `thinkingLevelMap` 判断支持情况。`clampThinkingLevel` 会把不支持的请求级别降级或升级到可用级别。

不同 Provider 的具体 reasoning 参数由各 Provider adapter 转换。

## 动态 Provider

扩展可以注册 Provider：

- 只覆盖已有 provider 的 baseUrl 或 headers。
- 完整替换 provider models。
- 注册 OAuth Provider。
- 注册自定义 `streamSimple`。

`ModelRegistry.registerProvider` 会：

1. 校验配置。
2. 注册 OAuth provider。
3. 注册 API provider。
4. 保存 request config。
5. 添加或替换 models。
6. 应用 OAuth modifyModels。

注销时会 reload 内置模型和剩余动态 Provider。

## 新增内置 Provider 流程

1. 在 `packages/ai/src/providers` 实现 Provider。
2. 在 `packages/ai/src/providers/register-builtins.ts` 添加 lazy loader 和 `registerApiProvider`。
3. 更新模型生成脚本，不直接修改 `models.generated.ts`。
4. 增加 Provider 专属 compat、OAuth 或 headers 处理。
5. 补充 Provider stream、tool call、thinking、cache、error、abort 测试。

## 关键源码

- AI 类型：`packages/ai/src/types.ts`
- AI stream API：`packages/ai/src/stream.ts`
- API registry：`packages/ai/src/api-registry.ts`
- built-in provider registration：`packages/ai/src/providers/register-builtins.ts`
- model facade：`packages/ai/src/models.ts`
- coding-agent ModelRegistry：`packages/coding-agent/src/core/model-registry.ts`
- AuthStorage：`packages/coding-agent/src/core/auth-storage.ts`
