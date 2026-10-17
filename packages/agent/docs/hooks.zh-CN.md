# AgentHarness hooks 设计

> 本文为[英文原文](./hooks.md)的简体中文译本；代码示例中的标识符和注释保留原文。

<!-- Synced from jot 3utlzkxy. Edit this file in-repo going forward. -->

最终设计。

## 核心模型

事件通过仅用于类型检查的 phantom 字段携带其结果类型：

```ts
declare const HookResult: unique symbol;

interface HookEvent<TType extends string, TResult = void> {
	type: TType;
	readonly [HookResult]?: TResult;
}

type ResultOf<E> = E extends { readonly [HookResult]?: infer R } ? R : void;

type HookHandler<E, Ctx> = (
	event: E,
	ctx: Ctx,
	signal?: AbortSignal,
) => ResultOf<E> | void | Promise<ResultOf<E> | void>;

type HookObserver<E, Ctx> = (
	event: E,
	ctx: Ctx,
	signal?: AbortSignal,
) => void | Promise<void>;
```

示例：

```ts
interface ContextEvent extends HookEvent<"context", { messages?: AgentMessage[] }> {
	type: "context";
	messages: AgentMessage[];
}

interface ToolCallEvent extends HookEvent<"tool_call", { block?: boolean; reason?: string }> {
	type: "tool_call";
	toolName: string;
	input: Record<string, unknown>;
}

interface MessageEndEvent extends HookEvent<"message_end"> {
	type: "message_end";
	message: AgentMessage;
}
```

不需要结果映射表，也不需要规范表。每种事件类型自行定义其结果。

## Hooks 接口

```ts
interface AgentHarnessHooks<E extends HookEvent<string, unknown>, Ctx> {
	context: Ctx;

	setContext(ctx: Ctx): void;

	observe(handler: HookObserver<E, Ctx>): () => void;

	on<TType extends E["type"]>(
		type: TType,
		handler: HookHandler<Extract<E, { type: TType }>, Ctx>,
	): () => void;

	emit<TEvent extends E>(
		event: TEvent,
		signal?: AbortSignal,
	): Promise<ResultOf<TEvent> | undefined>;

	addCleanup(cleanup: () => void | Promise<void>): () => void;

	clear(): Promise<void>;
	dispose(): Promise<void>;
}
```

重要区分：

- `observe()` 以只读方式接收所有事件，返回值会被忽略。
- `on(type, handler)` 会参与对应事件的语义处理。
- `AgentHarness` 只会调用 `emit(event)`。
- `clear()` 会移除 observers/handlers 并运行清理函数。

## 默认实现的内部逻辑

```ts
class DefaultAgentHarnessHooks<E extends HookEvent<string, unknown>, Ctx>
	implements AgentHarnessHooks<E, Ctx> {
	context: Ctx;

	private observers = new Set<HookObserver<E, Ctx>>();
	private handlers = new Map<string, Set<HookHandler<any, Ctx>>>();
	private cleanups = new Set<() => void | Promise<void>>();

	constructor(ctx: Ctx) {
		this.context = ctx;
	}

	setContext(ctx: Ctx): void {
		this.context = ctx;
	}

	observe(handler: HookObserver<E, Ctx>): () => void {
		this.observers.add(handler);
		return () => this.observers.delete(handler);
	}

	on(type, handler): () => void {
		let handlers = this.handlers.get(type);
		if (!handlers) {
			handlers = new Set();
			this.handlers.set(type, handlers);
		}
		handlers.add(handler);
		return () => handlers.delete(handler);
	}

	async emit(event, signal?) {
		for (const observer of this.observers) {
			await observer(event, this.context, signal);
		}

		switch (event.type) {
			case "context":
				return this.emitContext(event, signal);
			case "before_provider_request":
				return this.emitBeforeProviderRequest(event, signal);
			case "before_provider_payload":
				return this.emitBeforeProviderPayload(event, signal);
			case "before_agent_start":
				return this.emitBeforeAgentStart(event, signal);
			case "tool_call":
				return this.emitToolCall(event, signal);
			case "tool_result":
				return this.emitToolResult(event, signal);
			case "session_before_compact":
			case "session_before_tree":
				return this.emitFirstCancelOrLast(event, signal);
			default:
				await this.emitObservationHandlers(event, signal);
				return undefined;
		}
	}
}
```

由于 `Map<string, ...>` 会丢失具体类型信息，因此在实现内部使用类型断言是可以接受的。公开 API 仍保持有类型约束。

## 变更语义

### 观察

```ts
await hooks.emit({ type: "message_end", message }, signal);
```

Observers 会运行。`message_end` handlers 也会运行。除非该事件之后定义了结果类型，否则返回值会被忽略。

### Context 变换

Handlers 按顺序运行。每个 handler 都会看到当前消息。

```ts
let current = event;

for (const handler of handlers("context")) {
	const result = await handler(current, ctx, signal);
	if (result?.messages) {
		current = { ...current, messages: result.messages };
	}
}

return current.messages === event.messages ? undefined : { messages: current.messages };
```

### Provider request / payload

按顺序执行变换。每个 handler 都会看到前一个 handler 的输出。

```ts
let current = event;

for (const handler of handlers("before_provider_payload")) {
	const result = await handler(current, ctx, signal);
	if (result !== undefined) {
		current = { ...current, payload: result.payload };
	}
}

return changed ? { payload: current.payload } : undefined;
```

### Agent 启动前

收集注入的消息，并串联 system prompt。

```ts
let systemPrompt = event.systemPrompt;
const messages = [];

for (const handler of handlers("before_agent_start")) {
	const result = await handler({ ...event, systemPrompt }, ctx, signal);
	if (result?.messages) messages.push(...result.messages);
	if (result?.systemPrompt !== undefined) systemPrompt = result.systemPrompt;
}

return messages.length || systemPrompt !== event.systemPrompt
	? { messages, systemPrompt }
	: undefined;
```

### 工具调用

按顺序执行；一旦被阻止就提前退出。

```ts
for (const handler of handlers("tool_call")) {
	const result = await handler(event, ctx, signal);
	if (result?.block) return result;
}
```

### 工具结果

按顺序累积 patch。每个 handler 都会看到当前已修改的结果。

```ts
let current = event;
let modified = false;

for (const handler of handlers("tool_result")) {
	const result = await handler(current, ctx, signal);
	if (!result) continue;

	current = {
		...current,
		content: result.content ?? current.content,
		details: result.details ?? current.details,
		isError: result.isError ?? current.isError,
	};

	modified = true;
}

return modified
	? { content: current.content, details: current.details, isError: current.isError }
	: undefined;
```

### Session 前置事件

按顺序执行；遇到取消请求就提前退出。

```ts
let last;

for (const handler of handlers(event.type)) {
	const result = await handler(event, ctx, signal);
	if (!result) continue;
	last = result;
	if (result.cancel) return result;
}

return last;
```

## Harness 的用法

Harness 只执行：

```ts
await this.hooks.emit(event, signal);
```

或者：

```ts
const result = await this.hooks.emit({ type: "context", messages }, signal);
return result?.messages ?? messages;
```

Harness 不会保存 handlers、串联 listeners，也不了解扩展策略。

## Context

Context 是普通对象，不会在每次 emit 时重新构建。

```ts
const hooks = new CodingAgentHooks({
	harness: harnessFacade,
	session: sessionFacade,
	ui: noUiFacade,
});
```

之后可以这样更新：

```ts
hooks.setContext({
	...hooks.context,
	ui: tuiFacade,
});
```

对于动态状态，优先使用稳定的 facade/方法，而不是复杂的一层层 getter：

```ts
interface CodingAgentHookContext {
	harness: HarnessFacade;
	session: SessionFacade;
	ui: UiFacade;
	models: ModelFacade;
}
```

每次运行的 `signal` 会作为 handler 的第三个参数传入。

## 后续的扩展加载

扩展加载可以放在 harness 旁边，并负责构造 hooks：

```ts
const hooks = await loadExtensions({
	paths,
	context,
	hooks: new CodingAgentHooks(context),
});
const harness = new AgentHarness({ ..., hooks });
```

Loader 会在 hooks 中注册处理器：

```ts
hooks.on("context", handler);
hooks.on("tool_call", handler);
hooks.addCleanup(cleanup);
```

重新加载时：

```ts
await hooks.clear();
const nextHooks = await loadExtensions(...);
harness.setHooks(nextHooks); // 若支持，则仅限 idle 状态
```

## 需要考虑的缺口

### 1. 必须明确错误策略

现有 coding-agent 会捕获扩展错误、报告错误并继续运行。新的 hooks 需要采用相同策略，可能是：

```ts
errorMode: "continue" | "throw"
onError(error)
```

coding-agent 的默认值应为 `"continue"`。

### 2. 来源元数据很重要

现有 runner 知道哪个扩展产生了某个错误/资源/工具。普通 `on()` 注册会丢失这类信息，除非添加注册元数据或作用域。

可能需要：

```ts
const scope = hooks.createScope({ sourceInfo });
scope.on("context", handler);
scope.addCleanup(...);
```

或者：

```ts
on(type, handler, { sourceInfo })
```

### 3. 某些扩展能力是注册表，而不是 hooks

这些能力不由 `emit()` 管理，应继续作为 `CodingAgentHooks` 或扩展 host 上的注册表：

- 工具
- 命令
- 快捷键
- 标志位
- 消息渲染器
- provider 注册项
- OAuth provider
- 自定义 model provider

这样没问题。它们不属于 `AgentHarness`。

### 4. 现有 coding-agent 事件都能表达

以下事件都不存在阻碍：

- `context`
- `before_provider_request`
- `after_provider_response`
- `before_agent_start`
- `message_end`
- `tool_call`
- `tool_result`
- `input`
- `user_bash`
- `resources_discover`
- `session_before_*`
- `session_*`
- model/thinking 选择事件
- agent/turn/message/tool 生命周期事件

它们可以成为由 `CodingAgentHooks` 处理的额外事件类型。

### 5. 必须保留旧实现的精确语义

迁移 coding-agent 时，必须复制以下特殊情况：

- `input`：串联变换，`handled` 会使处理提前结束。
- `user_bash`：采用第一个有实际意义的结果。
- `message_end`：替换消息必须保留相同的 role。
- `before_agent_start`：`ctx.getSystemPrompt()` 必须反映当前串联后的 prompt。
- `resources_discover`：汇总路径并保留扩展来源。
- `tool_call`：参数修改对后续 handlers 可见。
- `tool_result`：后续 handlers 会看到前面 handlers 所作的 patch。

这个设计允许这些行为，但默认实现/coding hooks 实现必须明确编码这些语义。

### 6. `emit()` 的 switch 可能漏掉自定义变更事件

如果子类添加会产生结果的事件，却忘了覆盖 `emit()`，该事件就会按观察型事件处理。测试应捕获这个问题。如果之后发现容易出错，可以增加受保护的策略注册表，但初期不必做。

### 7. Observer 语义有意保持有限

Observers 只会看到一次最初发出的事件，不会看到每次中间变更。如果需要最终变换后的状态，应另发一个最终事件，或使用针对该事件的 handler。

## 结论

这个设计可以实现新的 coding-agent。它比当前 runner 更简单，能保持 harness 干净；只要 `CodingAgentHooks` 添加能感知来源的作用域、注册表、清理逻辑，并保留旧事件的精确语义，就能维持重要的扩展能力。

--- 评论 ---

主题 hn2xk0tzh：“addCleanup(cleanup”
  [tmluyaub9v] Owner（2026-05-14T12:55:45.500Z）：cleanup 也应能作为可选参数传给 `on`/`observe`。
