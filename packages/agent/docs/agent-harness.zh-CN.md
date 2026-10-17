# AgentHarness 生命周期

> 本文为[英文原文](./agent-harness.md)的简体中文译本；代码示例中的标识符和注释保留原文。

`AgentHarness` 是位于底层 agent 循环之上的编排层。它负责会话持久化、运行时配置、资源解析、操作锁，以及面向扩展的状态变更语义。

本文描述当前方向和已实现的行为。部分扩展/会话 facade 的细节仍在规划中，文中会明确标出。

## 最终生命周期目标

Harness 的监听器和 hooks 应当能够捕获 `AgentHarness` 实例，并在文档允许的事件中调用公开的 harness API。这些调用不得破坏正在执行的轮次快照、改变已持久化对话条目的顺序、丢失待写入内容、造成等待结算时死锁，或使 harness 留在错误的阶段。

预期规则如下：

- 忙碌时仍拒绝结构性操作。
- 在文档规定的轮次安全点接受队列操作。
- 运行时配置 setter 更新后续快照，但不修改当前 provider 请求。
- 忙碌期间发起的会话写入应可靠排队，并按确定顺序刷新。
- getter 返回最新 harness 配置，而不是执行中的快照。
- 目前不会向监听器/hooks 传入 facade；如果它们捕获原始 harness，并在运行期间调用 `waitForIdle()` 等等待结算的 API，可能造成死锁。未来的 facade 应改为提供 `runWhenIdle()`。

`AssistantMessageStream` 已经将 provider 传输流（例如 SSE 或 WebSocket 读取）与下游事件消费解耦。因此，harness 可以等待监听器、扩展 hooks、持久化和保存点工作，而不会阻塞 provider 传输读取，也无需重新引入临时事件队列。生命周期代码应优先在 harness 边界显式使用 `await` 排定顺序，而不是让 hook/事件结算在后台即发即弃。

最终的生命周期加固工作应通过覆盖广泛的监听器/hook 重入情形的测试套件，证明这些保证成立。

## 错误处理

当前采用以下分工：

- 底层能力和辅助函数在预期失败可以被封装且不应抛异常时使用 `Result<TValue, TError>`，例如 `ExecutionEnv`、文件系统/ shell 操作、shell 输出捕获、资源加载和压缩辅助函数。
- `Session`、`AgentHarness` 等高层变更/编排 API 会拒绝操作或抛出异常，而不是返回容易被忽略的裸结果。
- 在可行时，公开的 `AgentHarness` 错误会统一为 `AgentHarnessError`；子系统错误会作为 `cause` 保留。

Harness 事件观察的是已提交状态。公开 mutator 会尽可能在提交前验证必需输入并完成持久化，然后等待通知处理完毕。如果 hook 或订阅者在提交后失败，状态变更不会回滚，公开方法会以错误码为 `"hook"` 的 `AgentHarnessError` 拒绝。

## 状态模型

Harness 将状态分为四类。

### Harness 配置

Harness 配置是应用或扩展最近设置的运行时配置：

- model
- thinking level
- 工具
- 当前启用的工具名称
- 资源
- stream options
- system prompt 或 system prompt provider

getter 返回 harness 配置，不会返回某个正在执行的 provider 请求所用的快照。

setter 会立即更新 harness 配置，即使某个轮次正在执行也一样。更改会影响下一轮快照，不会影响当前正在运行的 provider 请求。

`setResources()` 接收具体资源，并且每次调用都会以浅拷贝后的当前资源和先前资源发出 `resources_update`。资源从磁盘或其他来源加载/重新加载由应用负责；应用应通过 `setResources()` 设置新值。

`getResources()` 返回浅拷贝后的当前资源。这是对实时配置的读取，不是最近一次轮次的快照。

### 轮次快照

轮次快照是一次 LLM 轮次所用的具体状态。它由 `createTurnState()` 创建，包含：

- 已持久化的会话消息
- 已解析的资源
- 已解析的 system prompt
- model
- thinking level
- 全部工具
- 当前启用的工具
- stream options
- 派生出的会话 ID

静态选项值会直接使用。每次调用 `createTurnState()` 时，都会调用一次 system prompt provider 回调。该轮次的所有逻辑都使用同一份快照。

创建快照时会浅拷贝资源数组，但不会深拷贝单个 skill 和 prompt-template 对象。

创建快照时也会浅拷贝 stream options。`headers` 和 `metadata` 映射会浅拷贝，但其中的值不会深拷贝。每次 provider 请求都会通过 `getApiKeyAndHeaders()` 重新解析凭据，以便刷新即将过期的 token；但配置的 stream options 和派生出的会话 ID 来自当前轮次快照。

### Session

会话只包含已持久化的条目。读取会话时返回的是已持久化状态，不包含排队中的写入。

`Session.buildContextEntries()` 返回用于构建模型上下文、并考虑了压缩操作的条目序列。`Session.buildContext()` 根据完整的当前活动分支派生运行时状态，再将这些上下文条目投影成 `AgentMessage[]`。默认情况下，自定义条目不会进入模型上下文；应用可以向 `Session` 构造函数或 `buildContext()` 传入 `entryProjectors`，将选定的自定义条目投影为消息。应用也可以传入多个叠加的 `entryTransforms`；它们会在默认压缩变换之后运行，在投影前过滤或重新排列上下文条目。

会话存储实现必须将叶节点变更持久化为 `leaf` 条目。`setLeafId()` 不只是更新内存中的游标；它会追加一个持久化条目，其 `targetId` 是活动树叶节点，若目标是根节点则为 `null`。重新打开存储时，必须从最近一条影响叶节点的持久化条目还原当前叶节点。

### 待处理的会话写入

操作处于活动状态时发起的会话写入，会排入待处理会话写入队列。待处理写入基于会话条目形状，但不包含自动生成的字段（`id`、`parentId`、`timestamp`）。

待处理写入总会被持久化。它们会在保存点、操作结算时以及失败清理时刷新。

公开的 pending-writes/session-facade API 仍在规划中，尚未实现。

## 操作阶段

Harness 有明确的阶段：

```ts
type AgentHarnessPhase = "idle" | "turn" | "compaction" | "branch_summary" | "retry";
```

结构性操作要求 `phase === "idle"`，并且会在第一次 `await` 之前同步设置阶段：

- `prompt`
- `skill`
- `promptFromTemplate`
- `compact`
- `navigateTree`

当 harness 不处于 idle 状态时启动另一个结构性操作，会以错误码为 `"busy"` 的 `AgentHarnessError` 拒绝。

适当情况下，轮次期间允许以下操作：

- `steer`
- `followUp`
- `nextTurn`
- `abort`
- 运行时配置 setter

阶段/结算语义仍属暂定，还需要完整的生命周期检查。

## 轮次执行

`prompt`、`skill` 和 `promptFromTemplate` 遵循相同流程：

1. 确认 harness 处于 idle，并将阶段设为 `"turn"`。
2. 使用 `createTurnState()` 创建轮次快照。
3. 根据该快照派生调用文本。
4. 使用 `executeTurn()` 执行轮次。

`skill` 和 `promptFromTemplate` 会从传给轮次的同一份快照中解析资源，不会单独再解析一次资源。

`steer`、`followUp` 和 `nextTurn` 接收文本及可选图片，并在内部创建 user 消息。`nextTurn` 消息会插入下一次由用户发起的轮次中的新 user 消息之前。

队列模式是实时的，不会按轮次拍快照：

- `getSteeringMode()` / `setSteeringMode()`
- `getFollowUpMode()` / `setFollowUpMode()`

运行期间更改队列模式会影响下一次队列排空。队列只会在安全点排空。

## 保存点

assistant 轮次及其工具结果消息完成后，会出现一个保存点。

在保存点，harness 会：

1. 在 agent 为该轮次发出的消息之后刷新待处理会话写入。
2. 如果底层循环可能继续，则创建新的轮次快照。
3. 在下一次 provider 请求前应用新的上下文、model、thinking level、stream options 和会话 ID 状态。

这样，在某轮次期间对 model、thinking level、工具、资源、stream option 和 system prompt 所作的更改，可以影响同一次运行的下一轮，同时不会修改正在执行中的 provider 请求。保存点不会重新创建循环回调。由于 `AssistantMessageStream` 已经将 provider 传输读取解耦，因此可以直接等待保存点工作和 hook 结算，确保对话记录/会话顺序确定。

底层循环会在 provider 边界将 harness 的 `ThinkingLevel` 转换为 `reasoning`：

- `"off"` -> `undefined`
- 其他 thinking level 原样传递

除了刷新剩余的待处理会话写入并清除操作阶段之外，`agent_end` 不需要刷新状态。`settled` 事件的确切触发时机仍在审查中。

如果在启动 `prompt`、`skill` 或 `promptFromTemplate` 时 system prompt 回调抛出异常，操作会以 `AgentHarnessError` 拒绝，harness 会恢复到 idle。如果 `prepareNextTurn` 创建保存点快照时回调抛出异常，底层 agent 运行会记录一条 assistant 错误消息。

## Hooks 和事件

目标 hook 系统见[hooks.md](./hooks.zh-CN.md)。

摘要：

- `AgentHarness` 发出有类型的 hook 事件，并消费有类型的结果。
- 一个 hooks 实现负责注册、清理、来源信息和结果归并器。
- 观察型 hook 和变更型 hook 共用按事件区分的 `on()` API；handler 是否可返回结果由事件结果类型决定。
- 会产生结果的事件由有类型的归并器表处理；应用专属 hooks 仅需为应用专属且会产生结果的事件添加归并器。
- hook 注册来源信息是注册项上的旁路元数据。资源和工具的来源信息属于应用专属的具体值类型。
- Hook 上下文应是由 facade 组成的普通对象，而不是原始内部对象或运行时才绑定的一组 getter。

事件 payload 描述正在发生的事情。Harness getter 描述供未来快照使用的最新配置。只要可行，hook 和 listener 的结算应按生命周期顺序等待完成；传输背压已由 harness 下层的 `AssistantMessageStream` 处理，因此 harness 无需另建异步事件队列来确保 SSE 或 WebSocket 持续读取。

## 计划中的会话 facade

未来扩展应通过作用域限定于 harness 的 `HarnessSession` facade 交互，而不是直接操作原始 session。这个 facade 应包装内部 session，并强制执行 harness 的待处理写入顺序规则。实现后，hooks 和事件监听器可以收到一个同时公开完整 `AgentHarness` 和 session facade 的上下文，同时避免直接进行顺序不确定的原始 session 写入。

计划中的读取语义：

- 读取委托给已持久化的会话状态。
- 读取不包含排队中的待处理写入。

计划中的写入语义：

- idle：立即持久化。
- busy：加入待处理会话写入队列。

计划中的诊断 API 可能会显式公开待处理写入：

```ts
getPendingWrites(): readonly PendingSessionWrite[]
```

agent 发出的消息会在 `message_end` 时持久化，以保持对话记录顺序。扩展/session 写入会在保存点、这些消息之后刷新。

## 中止

轮次期间允许中止。中止会终止底层运行，并清空 steering/follow-up 队列。

中止不会清除 `nextTurn` 消息。通过 `nextTurn()` 排队的消息会在中止后保留，并在下一次由用户发起的轮次中插入 user 消息之前。

中止不会丢弃待处理会话写入。如果到达保存点，会在该保存点刷新；否则会在 `agent_end` 或操作失败清理时刷新。

中止屏障语义仍需审查。

## 压缩和树导航

压缩和树导航属于结构性会话变更。

只有在 idle 状态下才允许执行；它们不会排队，而是直接操作已持久化的会话状态。下一次 prompt 会创建新的轮次快照。

分支摘要生成属于树导航操作的一部分。

`AgentHarness` 尚未实现自动压缩和重试决策点。

## 测试组织

Harness 测试应按领域聚焦，避免所有内容都堆进一个大型通用测试文件。

当前结构：

- `packages/agent/test/harness/agent-harness.test.ts`：核心生命周期和公开 API 行为。
- `packages/agent/test/harness/agent-harness-stream.test.ts`：stream options 和 provider hook 语义。

未来建议的结构：

- `agent-harness-resources.test.ts`：资源快照/加载语义。
- `agent-harness-tools.test.ts`：工具注册表 getter、启用工具语义和更新事件。
- `agent-harness-lifecycle.test.ts`：阶段/保存点/settled/重入行为。

确定性的 harness/provider 测试应使用 `pi-ai` 的 faux provider（`registerFauxProvider`、`fauxAssistantMessage`）。Faux 响应工厂可以检查 `StreamOptions`、调用 `options.onPayload`，并返回脚本化 assistant 消息，无需调用真实 provider API 或访问网络。

Harness 覆盖率与默认 package 测试分开配置：

```bash
npm run test:harness
npm run coverage:harness
```

`coverage:harness` 运行 `test/harness/**/*.test.ts`，并将 `src/harness/**/*.ts` 以及它直接测试的非 harness 运行时文件（`src/agent.ts` 和 `src/agent-loop.ts`）的覆盖率报告写入 `coverage/harness`。像 `src/types.ts` 这样的纯类型依赖不会纳入，因为它们没有有意义的运行时覆盖率。

## 实现待办

此列表跟踪将 `AgentHarness` 视为可用于迁移之前仍需完成的工作。进行中/计划中的项目按从易到难排序；已完成项目归档在底部。

### 1. 添加明确的工具注册表读取/更新语义

状态：进行中

已完成：

- 添加了 `setTools(tools, activeToolNames?)`。
- 添加了 `setActiveTools(toolNames)`。
- 无效的启用工具名称会以 `AgentHarnessError` 拒绝。
- 通过 `AgentHarness<TSkill, TPromptTemplate, TTool>` 添加了通用应用工具形状。
- 从核心类型导出了 `QueueMode`。
- 添加了 `AgentHarnessOptions.steeringMode` 和 `followUpMode`。
- 添加了实时的 `getSteeringMode()` / `setSteeringMode()` 和 `getFollowUpMode()` / `setFollowUpMode()`。
- 添加了 `getTools()` 和 `getActiveTools()`。
- 添加了 `tools_update` 可观测性事件，包括仅更新启用工具的情况。
- 启用工具的变更会以分支作用域的 `active_tools_change` 条目持久化。
- 工具名称和启用工具名称重复时会拒绝。

剩余：

- 无。

备注：

- 可观测性设计：[observability.md](./observability.zh-CN.md)

### 2. 设计每个 `AgentHarness` 独立的 model 注册表

状态：计划中

已完成：

- 保留了当前 `setModel()` 行为。

剩余：

- 决定应用如何提供 model 注册表。
- 决定 harness 保存具体 `Model` 对象、model 引用，还是两者都保存。
- 根据注册表验证 model 选择。
- 定义活动轮次和保存点期间 model 变更的语义。

### 3. 完整检查 `AgentHarness` 生命周期/状态

状态：进行中

已完成：

- 移除了构造函数中的 `void syncFromTree()`、`syncFromTree()`、`liveOperationId` 和 `shell()`。
- 添加了 `createTurnState()`、`applyTurnState()` 和 `executeTurn()`。
- 用显式的 `phase` 替代布尔型 idle 状态。
- 保存点会刷新上下文、model、thinking level、stream options 和会话快照状态。
- 待处理会话写入使用不含自动生成字段的会话条目形状。
- 待处理会话写入会在保存点、结算时和失败清理时刷新。
- `steer`、`followUp` 和 `nextTurn` 会根据文本及可选图片创建 user 消息。
- `nextTurn` 消息会插入新的 user prompt 之前。
- 结构性的压缩/树操作会在 `finally` 中恢复阶段。
- 公开 harness 错误会将子系统原因规范化为 `AgentHarnessError`。
- 待处理会话写入逐条刷新，失败时不会被丢弃。
- 如果队列更新通知失败，队列排空会回滚。
- `message_end` 持久化会在通知订阅者之前发生。
- `abort()` 会先发出取消信号；即使通知出错，仍会等待 idle。
- idle 时的 model/thinking/tool 更新会在提交内存状态前先验证并持久化。
- `setLeafId()` 会持久化 `leaf` 条目，使树导航在重新打开存储后仍然有效。

剩余：

- 最终确定 phase/idle 语义。
- 审查 `settled` 是否可能过早触发。
- 使 `settled` 回调中的会话写入顺序确定。
- 审查 `agent_end` 附近的 follow-up 行为。
- 实现自动压缩决策点。
- 实现重试处理。
- 对照 coding-agent 验证 `before_agent_start` hook 语义。
- 决定 `before_agent_start` 是否需要更多轮次信息，例如工具/工具摘要。
- 记录或调整忙碌期间运行时配置事件的触发时机。
- 审查 `abort()` 屏障语义。

### 4. 实现通用 hook/event 扩展机制

状态：已在 [hooks.md](./hooks.zh-CN.md) 设计，尚未实现

已完成：

- 移除了 `AgentHarnessContext`。
- Hooks 只接收事件 payload。
- `emitHook(event)` 根据 `event.type` 推导 hook 类型。
- Provider request/payload hooks 具有有序变换语义。

剩余：

- 添加 `HookEvent`、`ResultOf`、带泛型来源元数据的注册选项，以及唯一的 `AgentHarnessHooks` 实现。
- 将结果链式处理从 `AgentHarness` 移至归并函数。
- 对基础 harness 归并器做类型检查，确保每种会产生结果的 `AgentHarnessEvent` 都有归并语义。
- 让 `AgentHarness` 接收并公开具体 hooks 实例，并支持推导应用专属 hooks 的构造函数类型。
- 定义最初提供给 hook context 的 harness/context facade。
- 保留当前 provider hook 行为，包括 stream option patch 的字段删除语义。
- 为归并语义添加一致性测试：变换串联、patch 串联、提前阻止/取消、清理、来源元数据，以及应用专属归并器的类型覆盖。

备注：

- Hook 设计：[hooks.md](./hooks.zh-CN.md)

### 5. 对半持久化 harness/session 恢复做原型验证

状态：计划中

已完成：

- 编写了持久化设计：[durable-harness.md](./durable-harness.zh-CN.md)

剩余：

- 决定 session 是否拥有全部持久化 harness 状态，或是否需要为大型 blob 使用 sidecar。
- 为队列、待处理写入、操作、轮次、provider 请求和工具调用定义持久化条目。
- 定义恢复所需的应用提供依赖：工具、模型、扩展、资源、hooks 和 auth provider。
- 为未完成的 agent 轮次、provider 请求、工具调用、压缩和树导航定义保守的恢复策略。
- 原型验证从 session 条目进行基于归并器的恢复。
- 决定被中断的操作应追加用户可见消息，还是只追加内部操作条目。

备注：

- Provider stream 无法恢复；恢复应从持久化边界重新开始，或将操作标记为中断。
- 除非工具声明具备幂等/可安全重试语义，否则不安全地重试未完成的工具调用。

### 6. 最终生命周期加固测试套件

状态：计划中

已完成：

- 无。

剩余：

- 为相关事件添加覆盖广泛监听器/hook 重入场景的测试。
- 测试从底层生命周期事件和 harness 事件调用运行时配置 setter。
- 测试 model、thinking、resources、tools、active tools 和 stream options 的运行时配置可观测性。
- 测试活动轮次和保存点期间对资源/工具/model/thinking/stream option 的更新。
- 测试来自监听器和 hooks 的会话写入，包括 `settled` 写入。
- 测试从轮次事件、工具事件和 provider hooks 执行队列操作。
- 测试忙碌时拒绝结构性操作。
- 测试从监听器/hooks 中止。
- 测试活动操作期间 getter 的行为。
- 测试 agent 发出的消息与监听器待处理写入之间的确定顺序。
- 测试异步监听器调用并等待 harness API 时不会死锁。
- 测试成功、provider 错误、hook 错误、中止、压缩和树导航之后的阶段清理。

### 7. 后续 coding-agent 迁移计划

状态：计划中

已完成：

- 无。

剩余：

- 将 coding-agent 资源映射到带来源信息的加载器。
- 在 harness 之外保留应用层的资源去重/来源信息。
- 让扩展加载适配未来的 hook/session facade。
- 在核心之外保留 UI/session 行为。
- 将 coding-agent 的 stream/auth/retry/header 行为迁移到 harness stream 配置和 provider hooks。

---

## 已完成的实现待办

### 8. 移除 `AgentHarness` 对 `Agent` 的依赖

状态：完成

已完成：

- `AgentHarness` 直接调用 `runAgentLoop()`。
- Harness 自行管理运行生命周期、中止控制器、队列排空、provider stream 配置、事件归并、会话持久化、待处理写入刷新和保存点快照。
- Harness 测试覆盖 prompt 构建、队列排空、中止行为、保存点刷新、待处理写入顺序、等待监听器结算、工具 hooks 和 provider stream 包装。

剩余：

- 无。

备注：

- 更广泛的监听器/hook 重入覆盖见第 6 项。

### 9. 完成精选的 provider/stream 配置

状态：完成

已完成：

- 添加了精选的 `AgentHarnessOptions.streamOptions`、`getStreamOptions()` 和 `setStreamOptions()`。
- Stream options、headers、metadata 和派生的会话 ID 按轮次拍快照。
- Harness 自有的 stream wrapper 会调用 `streamSimple()`，并保留来自底层循环、由生命周期管理的 `signal` 和 `reasoning`。
- `getApiKeyAndHeaders()` 会在每次 provider 请求时解析凭据。
- 已实现 `before_provider_request`、`before_provider_payload` 和 `after_provider_response` hooks。
- Stream option patch 支持显式删除字段和按顺序串联 hooks。
- `agent-harness-stream.test.ts` 覆盖参数转发、auth 合并、hook patch/删除/串联、payload hooks，以及忙碌状态/保存点快照行为。

剩余：

- 无。

### 10. 完成底层 `Result` 清理

状态：完成

已完成：

- 添加了泛型 `Result<TValue, TError>` 及辅助函数。
- 更新 `ExecutionEnv` 和 `NodeExecutionEnv`，使文件系统/进程操作返回有类型的结果。
- 拆分文件系统和 shell 能力。
- 将 JSONL 会话存储/repo 改为通过文件系统能力访问，不再直接导入 Node 模块。
- 为流式追加用例在 `ExecutionEnv` 添加 `appendFile()`。
- 更新 skill 和 prompt-template 加载器，使其使用 `ExecutionEnv` 的结果。
- 更新 shell 输出捕获，使其返回结果并使用 `ExecutionEnv`；完整输出溢写也通过 `appendFile()` 完成。
- 将 `NodeExecutionEnv` 从浏览器安全的根导出中移除。
- 在通用截断工具中用运行时无关的 UTF-8 处理替换 `Buffer` 用法。
- 将压缩和分支摘要辅助函数改为返回有类型的结果。
- 添加 `readTextLines()`，使 JSONL 元数据加载只读取首行。
- 移除 Node 文件系统方法中对无意义取消操作的空实现。
- 将跨越 session 边界的文件系统错误映射为有类型的 `SessionError`。
- 添加有类型的分支摘要错误，以及能够保留 cause 的公开 harness 错误规范化。
- 对非 `not_found` 文件系统错误，资源加载器会报告结构化诊断信息。
- 扩展 `NodeExecutionEnv` 测试，覆盖文件操作、exec 错误、中止、回调、超时和 shell 输出溢写。

剩余：

- 无。

备注：

- 返回 `Result` 的底层能力/辅助 API 应保持不抛异常。
- Session storage/repo/session API 应抛出有类型的 `SessionError`。
- 公开的结构性 harness 错误应统一为 `AgentHarnessError`。
- Node 专属 API 应隔离在 `src/harness/env/nodejs.ts`、基于 Node 的存储/session 实现或明确的 Node 专属入口中。
- 增加 API 时，检查通用 harness 工具是否使用了 Node 全局变量。
- 检查 package exports，确保浏览器/通用导入不会引入仅限 Node 的模块。
- 随 API 演进持续扩展 `ExecutionEnv` 和 shell 输出契约测试。
