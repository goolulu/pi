<!-- Synced from jot qe0ikdqs. Edit this file in-repo going forward. -->

# Pi 可观测性设计笔记

> 本文为[英文原文](./observability.md)的简体中文译本；代码示例中的标识符和注释保留原文。

## 目标

让 `packages/ai` 和 `packages/agent`/harness 具备可观测性，而不依赖 OpenTelemetry、Sentry 或任何 APM 厂商。

Pi 应发出稳定、结构化的生命周期事件。外部监听器可以将这些事件转换为 OTel spans、Sentry spans、日志、指标或自定义遥测数据。

## 概念模型

Trace 是一棵因果工作树，例如一个用户轮次。

Span 是这棵树中一个带计时的操作。通常用 ID 而不是对象指针表示：

```ts
interface SpanRecord {
  traceId: string;
  spanId: string;
  parentSpanId?: string;
  name: string;
  startTime: number;
  endTime?: number;
  attributes: Record<string, unknown>;
  status: "ok" | "error";
}
```

示例树：

```text
traceId=t1 spanId=s1 parent=-  name=pi.agent.prompt
traceId=t1 spanId=s2 parent=s1 name=pi.agent.turn
traceId=t1 spanId=s3 parent=s2 name=pi.ai.provider.request
traceId=t1 spanId=s4 parent=s2 name=pi.agent.tool_call
traceId=t1 spanId=s5 parent=s4 name=pi.session.append_entry
```

## 异步上下文

JavaScript 虽然只有一个事件循环，但多条异步调用链可能交错执行。并发时，单个全局 `currentContext` 会失效。

对于异步延续，Node 的 `AsyncLocalStorage` 相当于 `ThreadLocal`。它使并发操作可以保有各自独立的当前上下文：

```ts
await Promise.all([
  runWithPiContext({ userId: "alice" }, () => harness.prompt("A")),
  runWithPiContext({ userId: "bob" }, () => harness.prompt("B")),
]);
```

深层代码因此能够读取当前异步调用链对应的上下文。

Pi 必须能运行在 Node、Bun、浏览器、workers 和其他 JS 运行时中，因此 ALS 不能成为核心抽象，而应作为运行时适配器。

## 核心设计

Pi 自身提供一个小型、与运行时无关的可观测性抽象：

```ts
export interface PiObservabilityContext {
  traceId?: string;
  currentSpanId?: string;
  userContext?: Record<string, unknown>;
}

export interface PiObservabilityEvent {
  type: "start" | "end" | "error" | "event";
  name: string;
  traceId: string;
  spanId?: string;
  parentSpanId?: string;
  timestamp: number;
  durationMs?: number;
  context?: Record<string, unknown>;
  payload?: Record<string, unknown>;
  error?: { name: string; message: string };
}

export interface PiObservability {
  getContext(): PiObservabilityContext | undefined;
  runWithContext<T>(context: PiObservabilityContext, fn: () => T): T;
  emit(event: PiObservabilityEvent): void;
  hasSubscribers(): boolean;
}
```

公开 API：

```ts
export function configurePiObservability(observability: PiObservability): void;
export function subscribePiObservability(listener: (event: PiObservabilityEvent) => void): () => void;
export function runWithPiContext<T>(userContext: Record<string, unknown>, fn: () => T): T;
export function traceOperation<T>(name: string, payload: Record<string, unknown>, fn: () => T): T;
```

`traceOperation()` 的流程：

1. 读取当前上下文。
2. 如果缺少 `traceId`，则创建一个。
3. 创建新的 `spanId`。
4. 将当前 span 用作 `parentSpanId`。
5. 发出 `start`。
6. 在子上下文中运行回调。
7. 发出 `end` 或 `error`。
8. 出错时重新抛出异常。

Pseudo-code:

```ts
function traceOperation<T>(name: string, payload: Record<string, unknown>, fn: () => T): T {
  const parent = getContext();
  const traceId = parent?.traceId ?? createId();
  const spanId = createId();
  const parentSpanId = parent?.currentSpanId;

  const child = { ...parent, traceId, currentSpanId: spanId };

  emit({ type: "start", name, traceId, spanId, parentSpanId, timestamp: Date.now(), context: parent?.userContext, payload });

  return runWithContext(child, () => {
    try {
      const result = fn();
      // Promise-aware implementation emits end/error after settlement.
      emit({ type: "end", name, traceId, spanId, parentSpanId, timestamp: Date.now(), context: child.userContext, payload });
      return result;
    } catch (error) {
      emit({ type: "error", name, traceId, spanId, parentSpanId, timestamp: Date.now(), context: child.userContext, payload, error: serializeError(error) });
      throw error;
    }
  });
}
```

## 运行时适配器

核心 package 不应导入仅限 Node 的 API。

可选实现：

- Node 适配器：用 `AsyncLocalStorage` 管理上下文；可以选择通过 `diagnostics_channel` 发布事件。
- 浏览器/workers 回退实现：使用本地订阅者集合，并有限度地/手动传播上下文。
- Bun/Deno 适配器：如果运行时提供异步上下文能力，则使用它。

在 Node 中，可以将 diagnostics channel 用作被动事件总线：

```ts
import { channel } from "diagnostics_channel";
channel("pi.observability").publish(event);
```

订阅者可以据此创建 OTel/Sentry spans，无需 monkey-patch pi。

## Pi 发出什么

Pi 只发出发生了什么，不会直接创建 OTel/Sentry spans。

初期最小事件名称：

```text
pi.agent.prompt
pi.agent.skill
pi.agent.prompt_template
pi.agent.compaction
pi.agent.branch_navigation
pi.agent.session.append_entry
pi.ai.provider.request
```

每种操作会发出：

```text
start
end
error
```

之后可以增加：

```text
pi.agent.turn
pi.agent.tool_call
pi.agent.queue_update
pi.ai.provider.retry
pi.ai.provider.first_token
pi.ai.provider.usage
pi.session.read
pi.session.write
```

## 最少埋点位置

### packages/agent

需要包装：

- `AgentHarness.prompt()`
- `AgentHarness.skill()`
- `AgentHarness.promptFromTemplate()`
- `AgentHarness.compact()`
- `AgentHarness.navigateTree()`
- `Session.appendTypedEntry()` or storage append facade

示例：

```ts
return traceOperation(
  "pi.agent.prompt",
  {
    sessionId: turnState.sessionId,
    provider: turnState.model.provider,
    model: turnState.model.id,
    promptLength: text.length,
    imageCount: options?.images?.length ?? 0,
  },
  () => this.executeTurn(turnState, text, options),
);
```

Session 写入：

```ts
return traceOperation(
  "pi.agent.session.append_entry",
  { entryType: entry.type },
  async () => {
    await this.unwrap(this.storage.appendEntry(entry));
    return entry.id;
  },
);
```

### packages/ai

包装常见的 provider 边界：

- `streamSimple()`
- `completeSimple()`

Example:

```ts
return traceOperation(
  "pi.ai.provider.request",
  {
    api: model.api,
    provider: model.provider,
    model: model.id,
    sessionId: options.sessionId,
    reasoning: options.reasoning,
  },
  () => actualStreamSimple(model, context, options),
);
```

结束/错误 payload 可以包含安全元数据：

- 停止原因
- 状态码
- 重试次数
- 输入/输出/总 token 数
- 总成本
- 是否中止/超时的标志

## 安全与脱敏

默认 payload 必须安全。

默认可安全记录：

- provider
- model
- API 标识符
- session ID
- 条目类型
- 工具名称
- 状态码
- 停止原因
- token 数量
- 成本
- 时长

默认不应记录：

- prompts
- completions
- 工具参数
- 工具结果
- shell 输出
- 文件内容
- provider 请求 payload
- provider 响应正文
- API key
- headers

之后可以通过明确的脱敏 hooks，让内容捕获成为可选择开启的功能。

## Listener 行为

可观测性不得影响 pi 的执行。

订阅者错误应被吞掉或隔离。Harness hooks 属于控制面，可能影响执行；可观测性订阅者是被动的，不能影响执行。

## 用户上下文

用户可以为一个轮次关联任意上下文：

```ts
await runWithPiContext(
  {
    userId: "u123",
    orgId: "acme",
    region: "eu",
  },
  () => harness.prompt("fix this"),
);
```

该异步调用链中发出的每个事件都会包含此上下文：

```ts
{
  type: "start",
  name: "pi.ai.provider.request",
  traceId: "t1",
  spanId: "s3",
  parentSpanId: "s1",
  context: {
    userId: "u123",
    orgId: "acme",
    region: "eu",
  },
  payload: {
    provider: "anthropic",
    model: "claude-sonnet-4",
  },
}
```

OTel 适配器可以将这些内容映射为 span attributes；Sentry 适配器可以映射为 Sentry context/spans；用户也可以自行记录 JSON。

## Package 规划

初期最小 package：

```text
packages/observability
  runtime-agnostic context + traceOperation + subscribe
```

之后：

```text
packages/ai
  emits pi.ai.* events

packages/agent
  emits pi.agent.* / pi.session.* events
```

未来可选增加：

```text
packages/observability-node
  AsyncLocalStorage + diagnostics_channel bridge

packages/otel
  subscribes to pi events and creates OpenTelemetry spans
```

## 核心主张

Pi 定义稳定、安全的事件契约；事件去向由适配器决定。

这样，ai/harness 就能具备可观测性，同时核心 package 不必绑定 OTel、Sentry、仅限 Node 的 API 或 monkey-patching。
