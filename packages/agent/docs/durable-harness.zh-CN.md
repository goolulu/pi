# AgentHarness 与 session 持久化设计

> 本文为[英文原文](./durable-harness.md)的简体中文译本；代码示例中的标识符和注释保留原文。

<!-- Synced from jot zmnps2zu. Edit this file in-repo going forward. -->

AgentHarness / session 持久化设计笔记。

## 背景

仅靠 `AgentHarness` 本身实现完全持久化并不现实，因为一些重要依赖是由宿主应用在运行时提供的 JavaScript：

- 工具实现
- model/auth provider
- 扩展和 hook handler
- 资源加载器
- system prompt 回调/修饰器

工具注册表属于运行时依赖。Harness 应持久化可序列化的工具配置，例如启用工具名称，但不应持久化具体工具实现。

实际目标是半持久化 harness：

- session 是可持久化、只追加的状态树。
- harness 将其拥有的状态写入 session 条目。
- 恢复时由宿主应用负责重新创建兼容的、无法持久化的依赖。
- 恢复从持久化边界重新开始，而不是从进行中的 provider stream 接续。

## 持久化状态由 Session 所有

应将 session 视为所有可持久化 agent 状态的载体，而不只是对话记录历史。

现有 session 状态已经包括 harness 状态：

- model 变更
- thinking-level 变更
- 启用工具变更
- 叶节点条目
- 标签
- 压缩和分支摘要
- 自定义消息和自定义条目

这说明应继续使用单一的持久化 session 日志，而不是增加 harness sidecar。大型 blob 仍可能适合放在 sidecar 中，但 session 条目应保留对它们的权威引用。

## 应用恢复时必须提供的内容

应用必须重新创建兼容的运行时依赖：

- model 注册表 / model 对象
- 工具注册表
- 扩展集合、版本和顺序
- 资源加载器
- system prompt provider/hooks
- auth provider
- 应用专属 hooks

在有稳定 ID/版本/哈希时，Harness 可以进行校验，但它本身无法序列化这些依赖。

## 运行时配置与恢复

构造函数选项仍是明确的运行时配置，不会读取 session 状态。在构造函数中暗中异步恢复，会让错误处理变得含糊。

未来应由异步 builder/factory 负责持久化状态恢复：

```ts
const harness = await AgentHarness.builder()
  .env(env)
  .session(session)
  .model(defaultModel)
  .tools(runtimeTools)
  .defaultActiveTools(["read", "edit"])
  .restore({ missingActiveTools: "fail" });
```

`restore()` 应读取活动分支、归并持久化的 harness 配置、为缺失条目应用默认值、根据应用提供的运行时依赖进行验证、构造 harness，并可选择在构造完成后发出 `source: "restore"` 更新事件。

针对启用工具：

- `active_tools_change` 条目是按分支作用域保存的持久化配置。
- 如果分支上没有 `active_tools_change`，则恢复时使用 builder 默认值；若没有提供启用工具名称默认值，则启用所有已注册工具。
- 启用工具名称必须唯一。
- 工具注册表中的名称必须唯一。
- 默认情况下，如果恢复出的启用工具名称缺失，恢复应失败；之后可以显式添加宽松策略，例如丢弃/禁用缺失项。
- 具体工具不会从 session 恢复；宿主应用必须提供兼容的工具。

## Harness 应持久化的内容

最小且有用的持久化条目：

- 按分支作用域保存的启用工具名称
- 已排队的 steer/followUp/nextTurn 消息
- 与某个轮次关联的队列消费记录
- 活动操作期间接受的待处理会话写入
- 待处理写入的应用状态
- 操作开始/完成/中断记录
- 轮次开始/完成记录
- 如恢复诊断有需要，则记录 provider 请求开始/完成
- 如需安全恢复工具，则记录工具调用开始/完成

可能采用的条目：

```ts
type DurableHarnessEntry =
  | QueueEnqueuedEntry
  | QueueConsumedEntry
  | PendingWriteEnqueuedEntry
  | PendingWriteAppliedEntry
  | OperationStartedEntry
  | OperationFinishedEntry
  | OperationInterruptedEntry
  | TurnStartedEntry
  | TurnFinishedEntry
  | ProviderRequestStartedEntry
  | ProviderRequestFinishedEntry
  | ToolCallStartedEntry
  | ToolCallFinishedEntry;
```

每项已接受的变更都必须在公开 API 返回之前持久化。

## 恢复模型

启动时：

1. 宿主应用注册工具/model/扩展/资源/auth/hooks。
2. Harness 打开 session。
3. Harness 将 session 条目归并为：
   - 当前叶节点
   - 对话分支
   - harness 配置，包括启用工具名称
   - 队列
   - 待处理写入
   - 活动操作/轮次/工具状态
4. Harness 校验所需的运行时依赖，包括根据应用提供的工具注册表验证恢复出的启用工具名称。
5. Harness 协调处理未完成的操作状态。

Provider stream 无法恢复。恢复只能从某个持久化边界重试，或将操作标记为中断。

## 恢复策略

默认采用保守策略：

- 未完成的 agent 轮次：标记为中断，保留持久化队列/待处理写入，并返回 idle。
- 未完成的 provider 请求：标记为中断；不自动重试。
- 未完成的工具调用：追加中断/错误工具结果；只有工具声明可安全重试/幂等时才重试。
- 未完成的压缩：如果不存在压缩条目，则重新运行。
- 未完成的分支摘要/树导航：如果安全，则重新运行/补应用缺失的摘要或叶节点条目。

可选策略：

```ts
recovery: "mark_interrupted" | "retry_unfinished"
```

`retry_unfinished` 必须防范非幂等工具调用。

## 关键场景

### 队列

- 在 `queue_enqueued` 之前崩溃：消息尚未被接受。
- 在 `queue_enqueued` 之后崩溃：消息会恢复。
- 队列排空之后、持久化轮次记录之前崩溃：可能导致消息丢失/重复。
- 必须满足的不变量：已消费队列 ID 必须记录在 `turn_started` 或等效条目中，之后才能视为已消费。

### 待处理写入

- 在 `pending_write_enqueued` 之前崩溃：写入尚未被接受。
- 入队后、应用前崩溃：恢复时应用该写入。
- 应用后、写入已应用标记之前崩溃：确定性的目标条目 ID 可让恢复过程检测该条目是否已存在，并将其标记为已应用。

### Agent 循环轮次

- provider 请求之前崩溃：重试，或标记为中断。
- provider 请求期间崩溃：默认标记为中断。
- provider 响应返回后、assistant 消息持久化前崩溃：如果 provider 结果未记入日志，该响应会丢失。
- assistant 消息持久化后崩溃：从持久化消息恢复。

### 工具调用

- 工具调用开始后、结果返回前崩溃：外部副作用可能已经发生。
- 默认恢复策略不应重新运行非幂等工具。
- 工具调用需要稳定 ID 和重试安全元数据，才能自动恢复。

### 压缩

- 摘要生成前崩溃：重新运行准备/摘要生成。
- 摘要生成后、压缩条目写入前崩溃：除非摘要已记入日志，否则重新运行。
- 压缩条目写入后崩溃：操作已完成；如果缺少完成标记，则追加该标记。

### 分支摘要 / 树导航

- 摘要生成前崩溃：重新运行，或标记为中断。
- 摘要条目写入后、叶节点条目写入前崩溃：追加缺少的叶节点条目。
- 叶节点条目写入后崩溃：操作已完成；如果缺少完成标记，则追加该标记。

## 最小可行原型

1. 添加持久化队列条目。
2. 添加带确定性目标 ID 的持久化待处理写入条目。
3. 添加操作开始/完成/中断条目。
4. 开始轮次时记录已消费的队列 ID。
5. 通过归并 session 日志进行恢复。
6. 默认将未完成的 agent 轮次标记为中断。
7. 只有在不存在最终条目时，才重新运行未完成的压缩/树操作。
8. 除非工具元数据声明可安全重试，否则不要重试未完成的工具调用。

## 未决问题

- 剩余 harness 配置中，哪些应该优先迁入 session：resources、stream options、system prompt 引用？
- 是否应为审计/调试按轮次快照保存已解析的 system prompt 文本？
- 恢复时是否要求严格匹配依赖的 ID/版本？
- 应将多少 provider 请求数据写入日志？
- 恢复时应追加用户可见的 assistant 中断消息，还是只追加内部操作条目？
- 恢复时，存储是否应支持截断 JSONL 文件末尾不完整的一行？
