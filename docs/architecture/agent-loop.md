# Agent 循环与会话封装设计

本文说明 Pi 的核心对话执行链路：`AgentSession` 如何封装 coding-agent 业务语义，`Agent` 如何驱动低层事件循环。

## 分层职责

```text
AgentSession (packages/coding-agent)
  |
  +--> 产品语义：资源、系统提示词、扩展、会话持久化、compaction、retry、工具注册
  |
  v
Agent (packages/agent)
  |
  +--> 状态机：messages、tools、streaming 状态、队列、事件订阅
  |
  v
runAgentLoop (packages/agent)
  |
  +--> LLM 调用、工具执行、turn 循环、event emission
```

`AgentSession` 是应用层；`Agent` 和 `runAgentLoop` 是可复用 runtime。

## AgentSession

`packages/coding-agent/src/core/agent-session.ts` 是 coding-agent 的核心业务层。

它持有：

- `Agent`: 来自 `@earendil-works/pi-agent-core`。
- `SessionManager`: JSONL 持久化。
- `SettingsManager`: 动态设置。
- `ModelRegistry`: 模型和认证。
- `ResourceLoader`: skills、prompts、themes、context files、extensions。
- `ExtensionRunner`: 扩展事件和 API。
- tool registry: 内置工具和扩展工具。

主要职责：

- 根据资源和启用工具重建 system prompt。
- 执行 slash command、扩展 command、skill command、prompt template 展开。
- 在 streaming 时把输入放入 steering 或 follow-up 队列。
- 在发起 LLM 前校验模型和认证。
- 把 Agent 事件转发给扩展。
- 把 message_end 事件持久化到 session。
- 自动处理 retry、compaction、branch summary。
- 暴露 prompt、steer、followUp、reload、navigateTree、setActiveTools 等 API。

## Prompt 执行路径

```text
AgentSession.prompt(text, options)
  |
  +--> extension command check
  +--> input hook
  +--> skill command expansion
  +--> prompt template expansion
  +--> if streaming:
  |       +--> queue steer/follow-up
  |       +--> return
  |
  +--> flush pending bash messages
  +--> model/auth preflight
  +--> optional compaction before send
  +--> build user message
  +--> append pending next-turn custom messages
  +--> before_agent_start hook
  +--> Agent.prompt(messages)
  +--> post-run retry or compaction continuation
```

扩展可以在多个点修改行为：

- `input`: 拦截或改写用户输入。
- `before_agent_start`: 注入 custom messages 或修改 system prompt。
- `context`: 转换进入 LLM 前的 AgentMessage[]。
- `before_provider_request`: 修改 Provider payload。
- `after_provider_response`: 观察 HTTP response。
- `tool_call` / `tool_result`: 拦截工具调用和结果。
- `message_end`: 替换最终消息。

## Agent 状态机

`packages/agent/src/agent.ts` 提供 `Agent` 类。

核心状态：

- `systemPrompt`
- `model`
- `thinkingLevel`
- `tools`
- `messages`
- `isStreaming`
- `streamingMessage`
- `pendingToolCalls`
- `errorMessage`

核心 API：

- `prompt(...)`: 新用户输入，启动完整 agent run。
- `continue()`: 从已有上下文继续，常用于 tool result 后续或 retry。
- `steer(message)`: 插入到当前 assistant turn 之后、下一次 LLM 调用之前。
- `followUp(message)`: Agent 本来要结束时再执行。
- `abort()`
- `waitForIdle()`
- `subscribe(listener)`

`Agent` 不知道 session 文件、扩展资源、UI 组件，这些由 `AgentSession` 处理。

## 队列语义

Agent 有两条队列：

- steering: 当前 assistant turn 完成工具执行后，下一次 LLM 调用前插入。
- follow-up: 没有工具调用、没有 steering 后，agent 本来会停止时插入。

每条队列支持两种 drain mode：

- `one-at-a-time`: 每次只取一条。
- `all`: 一次取完。

这些设置来自 `SettingsManager.getSteeringMode()` 和 `getFollowUpMode()`。

## 低层循环

`packages/agent/src/agent-loop.ts` 提供 `runAgentLoop` 和 `runAgentLoopContinue`。

新 prompt 流程：

```text
runAgentLoop(prompts, context, config)
  |
  +--> newMessages = prompts
  +--> currentContext = context + prompts
  +--> emit agent_start
  +--> emit turn_start
  +--> emit user message_start/end
  +--> runLoop(...)
```

`runLoop` 内部是两层循环：

```text
outer loop: follow-up messages
  |
  +--> inner loop: tool calls or steering messages
        |
        +--> emit turn_start when needed
        +--> append pending steering/follow-up messages
        +--> stream assistant response
        +--> execute tool calls
        +--> emit turn_end
        +--> prepareNextTurn
        +--> shouldStopAfterTurn
        +--> poll steering messages
```

退出条件：

- assistant stopReason 是 `error` 或 `aborted`。
- `shouldStopAfterTurn` 返回 true。
- 没有 tool calls、没有 steering、没有 follow-up。

## LLM 边界

在 `streamAssistantResponse` 中，消息转换只发生在 LLM 调用前：

```text
AgentMessage[]
  |
  +--> transformContext(...)
  |
  +--> convertToLlm(...)
  |
  v
Message[] + systemPrompt + tools
  |
  +--> streamSimple(model, context, options)
```

这使应用层可以保留 custom message 类型，而 Provider 只看到标准 `user`、`assistant`、`toolResult`。

## 事件模型

常见事件顺序：

```text
agent_start
turn_start
message_start(user)
message_end(user)
message_start(assistant)
message_update(assistant delta...)
message_end(assistant)
tool_execution_start
tool_execution_update
tool_execution_end
message_start(toolResult)
message_end(toolResult)
turn_end
agent_end
```

`Agent.subscribe()` 的 listener 会按注册顺序 await。`agent_end` 是最后一个 loop event，但 `prompt()` 只有在 `agent_end` listeners 完成后才 settle。

## Tool Execution

工具执行由 Agent Core 负责，但工具定义来自 coding-agent 和扩展。

执行步骤：

1. 找到 tool。
2. `prepareArguments` 兼容模型输出差异。
3. `validateToolArguments` 用 TypeBox 校验。
4. 执行 `beforeToolCall`。
5. 调用 `tool.execute(...)`。
6. 执行 `afterToolCall`。
7. 生成 `toolResult` message。

批量 tool calls 有两种策略：

- `parallel`: 默认。preflight 顺序执行，工具实际并行执行，tool result 仍按 assistant 原始顺序写入。
- `sequential`: 顺序执行。任一工具声明 `executionMode: "sequential"` 时，本批整体顺序执行。

## 错误与重试

低层 `Agent` 在 run 失败时会合成 assistant error message：

- `stopReason: "aborted"`: 当前 abort signal 被触发。
- `stopReason: "error"`: 其他异常。

`AgentSession` 在 `agent_end` 后检查最后 assistant message，结合 settings 决定是否自动 retry。retry 成功会发 `auto_retry_end`，失败也会发对应事件。

## 关键源码

- `AgentSession`: `packages/coding-agent/src/core/agent-session.ts`
- `convertToLlm`: `packages/coding-agent/src/core/messages.ts`
- `Agent`: `packages/agent/src/agent.ts`
- `agent-loop`: `packages/agent/src/agent-loop.ts`
- Agent 类型：`packages/agent/src/types.ts`
