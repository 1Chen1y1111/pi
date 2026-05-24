# 会话存储与树形历史设计

本文说明 Pi 如何把对话、工具结果、模型切换、分支、压缩摘要持久化为 JSONL session。阅读本文不需要先读总览文档。

## 设计目标

Session 系统需要支持：

- 自动保存和恢复。
- 在单个文件内保留分支。
- 从任意历史节点继续。
- fork 出新的 session 文件。
- compact 后仍能重建 LLM context。
- 扩展持久化私有状态。
- 多进程场景下尽量减少破坏性写入。

核心设计是 append-only JSONL 加树形 `parentId`。

## 文件格式

每个 session 文件是 JSONL：

```text
{"type":"session", ...}
{"type":"message", "id":"...", "parentId":null, ...}
{"type":"message", "id":"...", "parentId":"...", ...}
```

首行是 `SessionHeader`：

- `type: "session"`
- `version`
- `id`
- `timestamp`
- `cwd`
- `parentSession`

后续每行是 `SessionEntry`。当前版本是 `CURRENT_SESSION_VERSION = 3`。

## Entry 类型

| 类型 | 作用 | 是否进入 LLM context |
| --- | --- | --- |
| `message` | user、assistant、toolResult、bashExecution 等消息 | 取决于 message role |
| `thinking_level_change` | 记录 thinking level 切换 | 否 |
| `model_change` | 记录模型切换 | 否 |
| `compaction` | 记录 compact 摘要和保留起点 | 是，转成 compactionSummary |
| `branch_summary` | 从分支返回时记录摘要 | 是，转成 branchSummary |
| `custom` | 扩展私有状态 | 否 |
| `custom_message` | 扩展注入上下文消息 | 是 |
| `label` | 用户书签/标记 | 否 |
| `session_info` | session 显示名等元数据 | 否 |

## 树形结构

每个 entry 都有：

- `id`
- `parentId`

线性对话就是每个新 entry 指向前一个 entry。分支时只移动 `leafId`，下一次 append 会成为该节点的新 child。已有 entry 不修改、不删除。

```text
root
  |
  +--> A
       |
       +--> B
       |    |
       |    +--> C
       |
       +--> B2
            |
            +--> C2
```

`/tree` 就是在这个结构里切换当前 leaf。

## SessionManager

`packages/coding-agent/src/core/session-manager.ts` 负责：

- 创建、打开、fork session。
- 迁移旧格式。
- 维护 `byId` 索引。
- 维护当前 `leafId`。
- append 各类 entry。
- 构建树结构。
- 从 leaf 重建 context。
- 列出当前项目或所有项目 sessions。

创建方式：

- `SessionManager.create(cwd, sessionDir?)`
- `SessionManager.open(path, sessionDir?, cwdOverride?)`
- `SessionManager.continueRecent(cwd, sessionDir?)`
- `SessionManager.inMemory(cwd?)`
- `SessionManager.forkFrom(sourcePath, targetCwd, sessionDir?)`

## 持久化策略

Session 是 append-only，但有一个特殊延迟：

- 在出现第一条 assistant message 前，不立即写完整 session 文件。
- 第一条 assistant 到来时，把已有 entries 一次性 flush。

这样可以避免只输入了空 session 或未完成 prompt 时留下噪声文件。

之后每个 entry 追加一行 JSON。

## 上下文重建

`buildSessionContext(entries, leafId, byId)` 做 LLM context 重建：

```text
leafId
  |
  +--> walk parentId to root
  +--> collect path
  +--> resolve latest thinking level
  +--> resolve latest model
  +--> find latest compaction
  +--> emit messages
```

无 compaction 时，按路径顺序输出所有可进入 context 的消息。

有 compaction 时：

1. 先输出 compaction summary。
2. 输出 `firstKeptEntryId` 到 compaction 前的保留消息。
3. 输出 compaction 之后的消息。

`custom_message` 会转成 custom AgentMessage，之后由 `convertToLlm` 转成 user message。

## Branch 与 Fork

`branch(id)`：

- 只移动当前 `leafId`。
- 下一次 append 在该节点下生成新分支。

`branchWithSummary(id, summary)`：

- 移动 `leafId`。
- 追加 `branch_summary` entry。
- 让模型知道被离开的分支摘要。

`createBranchedSession(leafId)`：

- 把 root 到 leaf 的单一路径复制到新 session。
- 保留相关 labels。
- 新 session 的 `parentSession` 指向原文件。

`forkFrom(sourcePath, targetCwd)`：

- 跨项目复制整个 source session 的非 header entries。
- 新 header 使用目标 cwd。
- `parentSession` 指向 sourcePath。

## Compact

Compact 通过 `compaction` entry 表达，不删除历史。

字段：

- `summary`
- `firstKeptEntryId`
- `tokensBefore`
- `details`
- `fromHook`

重建 context 时，旧历史由 summary 替代，近期消息仍按原 entry 保留。

这个设计让 session 文件保持可审计，同时控制进入模型的 token 数。

## 扩展状态

扩展有两种持久化入口：

- `custom`: 保存扩展私有数据，不进 LLM。
- `custom_message`: 保存扩展注入的上下文，可选择是否在 UI 显示。

扩展 reload 时可扫描 entries 重建内部状态。

## 修改格式的规则

新增或修改 session 格式时：

1. 增加 entry 类型或字段。
2. 更新 `SessionEntry` 类型。
3. 更新迁移函数。
4. 更新 `buildSessionContext`。
5. 更新 session-format 文档。
6. 添加回归测试。
7. 保持 append-only 语义，不原地改历史。

## 关键源码

- Session manager：`packages/coding-agent/src/core/session-manager.ts`
- 消息转换：`packages/coding-agent/src/core/messages.ts`
- runtime session replacement：`packages/coding-agent/src/core/agent-session-runtime.ts`
- 交互树 UI：`packages/coding-agent/src/modes/interactive/components/tree-selector.ts`
