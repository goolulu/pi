# Models 架构

> 本文为[英文原文](./models.md)的简体中文译本；代码示例中的标识符和注释保留原文。

本文描述下一阶段 `pi-ai` model/provider 重构的目标设计。它说明的是期望形态，而非当前实现，并力求完整到足以让新会话据此开始实现。

目标：

- `Models` 是一个职责单纯的 provider 运行时集合。
- 具体 provider 负责自己的元数据、认证、模型列表和 stream 行为。
- API 实现在 `src/api/` 下，并且可复用、按需加载。
- 具体 provider 工厂位于 `src/providers/` 下。
- 用户可以只导入所需的 provider。
- 导入 provider 时不能立即加载体积较大的 SDK。
- 动态模型列表是一等能力：读取是同步的（返回最近已知列表），获取列表则通过显式异步 `refresh` 完成。
- `models.json` 和扩展通过包装 provider 实现分层，而不是临时直接修改 provider 内部状态。
- 旧的全局 API 仅通过明确且临时的 `/compat` 入口保留。

当前这轮 `pi-ai` 工作不包含：

- 暂不迁移 coding-agent 的 `ModelRegistry`。
- 不把 stream/API 注册表放在 `Models` 内。
- 暂不实现 Web OAuth 流程。
- 图像生成沿用聊天侧的设计（在 `images-models.ts` 中使用 `ImagesModels`/`ImagesProvider`）；旧的全局图像 API（`images.ts`、`images-api-registry.ts`）保留在 compat 中。

## Package 布局

目标源码布局：

```txt
packages/ai/src/
  index.ts                    # core exports only; no built-in provider imports
  models.ts                   # Models runtime, Provider
  images-models.ts            # ImagesModels runtime, ImagesProvider (mirrors models.ts)
  compat.ts                   # temporary old-API compatibility entrypoint
  auth/                       # auth method types, helpers, shared resolveProviderAuth(), login callbacks
  api/                        # API implementations and lazy wrappers
    openai-completions.ts     # real implementation, imports SDKs, exports stream/streamSimple
    openai-completions.lazy.ts
    openai-responses.ts
    openai-responses.lazy.ts
    openai-codex-responses.ts
    openai-codex-responses.lazy.ts
    azure-openai-responses.ts
    azure-openai-responses.lazy.ts
    anthropic-messages.ts
    anthropic-messages.lazy.ts
    google-generative-ai.ts
    google-generative-ai.lazy.ts
    google-vertex.ts
    google-vertex.lazy.ts
    mistral-conversations.ts
    mistral-conversations.lazy.ts
    bedrock-converse-stream.ts
    bedrock-converse-stream.lazy.ts
    openrouter-images.ts      # image-generation API implementation
    openrouter-images.lazy.ts
    lazy.ts                   # lazyStream()/lazyApi() helpers
    (shared helpers: openai-responses-shared, google-shared, transform-messages, ...)
  providers/                  # concrete provider factories and per-provider catalogs
    openai.ts
    openai.models.ts          # generated OpenAI catalog
    openai-codex.ts
    openai-codex.models.ts
    anthropic.ts
    anthropic.models.ts
    google.ts
    google.models.ts
    ...one pair per built-in provider...
    openrouter-images.ts      # image-generation provider factory
    faux.ts                   # test provider factory
    all.ts                    # explicit aggregate: builtinModels(), builtinImagesModels(), getBuiltin*()
  utils/oauth/                # OAuth flow implementations (node), lazy-loaded
```

`src/index.ts` 必须只包含核心内容，不能导入：

- 生成的模型目录
- 内置 provider 工厂
- provider SDK 实现
- 仅限 Node 的 OAuth 模块
- `providers/all`
- `compat`

Provider、API 和 compat 入口都通过明确的子路径导出。

## 公开用法

最简 provider 用法：

```ts
import { createModels } from "@earendil-works/pi-ai";
import { openaiProvider } from "@earendil-works/pi-ai/providers/openai";

const models = createModels();
models.setProvider(openaiProvider());

const model = models.getModel("openai", "gpt-4o-mini");
if (!model) throw new Error("model not found");

const response = await models.complete(model, context);
```

使用多个 provider：

```ts
const models = createModels();
models.setProvider(openaiProvider());
models.setProvider(openrouterProvider());
```

导入所有内置 provider 的元数据（这是明确的重型入口）：

```ts
import { builtinModels } from "@earendil-works/pi-ai/providers/all";

const models = builtinModels();
```

`providers/all` 可以导入所有 provider 元数据/目录，但仍不得立即导入 SDK 实现；provider stream 应使用延迟加载包装器。

## 核心运行时：Models

`Models` 是 provider 集合，负责应用认证信息并提供便捷的 stream 方法。它不包含 stream 注册表，也不包含认证解析策略对象。

```ts
export function createModels(options?: {
  /** App-owned credential storage. Default: in-memory store. */
  credentials?: CredentialStore;
  /** Environment access for auth resolution (env vars, file existence). Default: process.env/node:fs backed; injectable for tests and non-Node hosts. */
  authContext?: AuthContext;
}): MutableModels;

export interface Models {
  getProviders(): readonly Provider[];
  getProvider(id: string): Provider | undefined;

  /** Sync read of last-known models. Best-effort: a provider whose getModels() throws yields no models. */
  getModels(provider?: string): readonly Model<Api>[];
  /** Dynamic lists are honestly Model<Api>; narrow with the hasApi() guard. */
  getModel(provider: string, id: string): Model<Api> | undefined;

  /**
   * Ask dynamic providers to re-fetch their model lists. With a provider id,
   * rejects on that provider's failure; without, refreshes all concurrently
   * best-effort. Static providers are no-ops.
   */
  refresh(provider?: string): Promise<void>;

  /**
   * Resolve request auth for a model. Includes source label for status UI.
   * Resolves undefined when the provider is unknown or unconfigured. Rejects
   * with ModelsError ("oauth" on refresh failure, "auth" on api-key/store
   * failure); status/availability UIs catch rejections and render
   * "needs re-login" instead of treating them as unconfigured.
   */
  getAuth(model: Model<Api>): Promise<AuthResult | undefined>;

  stream<TApi extends Api>(
    model: Model<TApi>,
    context: Context,
    options?: ApiStreamOptions<TApi>,
  ): AssistantMessageEventStream;

  complete<TApi extends Api>(
    model: Model<TApi>,
    context: Context,
    options?: ApiStreamOptions<TApi>,
  ): Promise<AssistantMessage>;

  streamSimple(model: Model<Api>, context: Context, options?: SimpleStreamOptions): AssistantMessageEventStream;
  completeSimple(model: Model<Api>, context: Context, options?: SimpleStreamOptions): Promise<AssistantMessage>;
}

export interface MutableModels extends Models {
  /** Upsert/replace by provider.id. Provider ids are unique. */
  setProvider(provider: Provider): void;
  deleteProvider(id: string): void;
  clearProviders(): void;
}
```

已移除的概念：

```txt
no Models.setStreamFunctions() / getStreamFunctions()
no api-registry as a real dispatch mechanism
no Models.provider(id) builder, no setModel/upsertModel/patchModel lifecycle
no ModelAuthResolver / setAuthResolver — resolution policy is fixed, store is injected
```

如果应用需要不同的认证策略，应包装 provider（包装其认证方法或 `getModels`），或在 stream options 中显式传入请求认证信息。

## Provider

Provider 是具体的运行时单元，负责自己的 id/name/基础元数据、认证方法、模型列表和 stream 行为。

`Provider` 使用泛型表示其模型所用的 API。具体工厂会声明自己提供哪些 API（例如 `openaiProvider(): Provider<"openai-responses" | "openai-completions">`），因此直接使用工厂的调用方能获得有类型的模型列表。`Models` 集合则以 `Provider<Api>` 持有各个 provider。

```ts
export interface Provider<TApi extends Api = Api> {
  readonly id: string;
  readonly name: string;

  readonly baseUrl?: string;
  readonly headers?: Record<string, string>;

  /**
   * Required: at least one of apiKey/oauth. Even ambient-credential providers
   * (env vars, AWS profiles, ADC) and keyless local servers provide apiKey
   * auth whose resolve() reports whether the provider is configured.
   * getAuth() returning undefined = not configured.
   */
  readonly auth: ProviderAuth;

  /** Current known models, sync. Static providers: the catalog. Dynamic providers: as of the last refresh (empty before the first). */
  getModels(): readonly Model<TApi>[];

  /** Dynamic providers only: fetch and update the model list. Concurrent calls share one in-flight fetch. */
  refreshModels?(): Promise<void>;

  stream<T extends TApi>(model: Model<T>, context: Context, options?: ApiStreamOptions<T>): AssistantMessageEventStream;

  streamSimple(model: Model<TApi>, context: Context, options?: SimpleStreamOptions): AssistantMessageEventStream;
}
```

这里没有 `Provider.api` 字段。API 标识由 `model.api` 携带；provider 内部负责分派（见 `createProvider()`）。

保留 `Model.api`：现有元数据和测试会用到它；它有助于诊断；构造 provider 时也会据此选择 API 实现。但 `Models` 不会根据它分派，分派由 provider 完成。

### 有类型的 stream options

完整的 stream options 因 API 而异。`Model<TApi>` 可以根据 API 推导对应的选项类型：

```ts
// types.ts — type-only imports from API impl modules are erased, so this is tree-shake safe
export interface ApiOptionsMap {
  "anthropic-messages": AnthropicOptions;
  "openai-completions": OpenAICompletionsOptions;
  "openai-responses": OpenAIResponsesOptions;
  "openai-codex-responses": OpenAICodexResponsesOptions;
  "azure-openai-responses": AzureOpenAIResponsesOptions;
  "google-generative-ai": GoogleOptions;
  "google-vertex": GoogleVertexOptions;
  "mistral-conversations": MistralOptions;
  "bedrock-converse-stream": BedrockOptions;
}

export type ApiStreamOptions<TApi extends Api> = TApi extends keyof ApiOptionsMap
  ? ApiOptionsMap[TApi]
  : StreamOptions & Record<string, unknown>;
```

自定义 API 字符串会退回到通用形状。

### 缩窄模型类型

运行时模型列表是动态的，因此 `models.getModel()`/`getModels()` 会如实返回 `Model<Api>`。以下三处可以进一步获得更精确的类型：

1. **`hasApi()` 类型守卫**：对动态查找结果在运行时检查并缩窄类型（无需盲目类型断言）：

   ```ts
   export function hasApi<TApi extends Api>(model: Model<Api>, api: TApi): model is Model<TApi>;

   const model = models.getModel("anthropic", "claude-opus-4-7");
   if (model && hasApi(model, "anthropic-messages")) {
      // model: Model<"anthropic-messages">，stream options 具有完整类型
   }
   ```

2. **`getBuiltinModel()`**：同步查询生成的模型目录，并提供有类型的重载：`(provider, id) -> Model<exact-api-literal>`。适用于已知模型的硬编码查找。

3. **`Provider<TApi>` 工厂**：不通过 `Models` 集合、直接使用 provider 时，模型列表会带有类型。

有意不采用以下做法：让 `models.getModel(provider, ...)` 与有类型的 provider/model ID 绑定。因为这样就必须在静态类型层面知道可变运行时集合中安装了哪些 provider。Harness 使用的路径（`streamSimple` + `SimpleStreamOptions`）与具体 API 无关，不受影响。

作为对比，Vercel AI SDK 将实现附着在 model 对象上。这消除了分派时的类型问题，但使 model 无法序列化（无法将其作为普通数据用于 session/RPC/目录）；其 `providerOptions` 包是 `Record<string, JSON>`，只能依赖 `satisfies` 约定进行检查。使用纯数据 model 并由 provider 管理行为，可以在重要位置保留更强的类型约束。

### 名称冲突

`types.ts` 当前导出 `type Provider = KnownProvider | string`（表示 provider ID）。应将这个别名重命名为 `ProviderId` 并修正使用位置。上文的 `Provider` 名称留给接口使用。

## Provider 模型列表

读取是同步的；获取列表则使用明确的异步动词。`Provider.getModels()` 返回当前已知列表：静态 provider 返回完整目录，动态 provider（llama.cpp、OpenRouter 实时列表）返回最近一次刷新的列表。动态 provider 在 `refreshModels()` 中获取列表。

之所以区分两者，是因为同步/异步联合类型（`Promise<T> | T`）会让代码暗中假设操作是同步的，直到遇到第一个异步 provider 才暴露问题；而只提供异步读取，则会迫使所有消费者（UI 列表、扩展的 `find`/`getAll` 接口）为几乎总是静态的数据使用 Promise。同步读取加显式刷新，可以让数据过期状态清晰可见，并让契约保持简单：`getModels()` = 最近已知列表，`refresh()` = 刷新为当前列表。获取到的列表返回时其实就可能过期了，因此明确标出刷新时机才是诚实的做法。

由应用负责刷新时机，例如启动时、重新加载注册表时或打开模型选择器时。对新鲜度要求高的查找分两步进行：`await models.refresh("llamacpp"); models.getModel("llamacpp", id)`。

动态刷新必须是无副作用的发现过程：

```txt
OK: fetch /v1/models, enumerate local catalog, refresh cached remote model list
Not OK: load model, download model, mutate server state, run request probe
```

provider 专属的模型生命周期操作（加载/卸载）应放在应用/provider 管理命令中，而不是放在 `refreshModels()` 里。

## Stream 调用路径

`Models.stream()` 根据 `model.provider` 找到 provider、解析认证信息、将认证信息合并到请求选项中，然后委托给 provider：

```ts
function stream(model, context, options) {
  const provider = this.getProvider(model.provider);
  if (!provider) {
    // produce an error stream, not a throw — see Error behavior
  }

  // async setup happens inside the returned stream (lazyStream pattern)
  const resolution = await this.getAuth(model);
  const requestModel = resolution?.auth.baseUrl ? { ...model, baseUrl: resolution.auth.baseUrl } : model;
  const requestOptions = mergeAuth(options, resolution?.auth); // explicit options win per-field

  return provider.stream(requestModel, context, requestOptions);
}
```

`stream()` 会同步返回 `AssistantMessageEventStream`；异步准备工作（解析认证信息、延迟加载模块）在返回的 stream 内部完成。当前 `register-builtins.ts` 已有转发模式（`createLazyStream`），应将它提取为 `src/api/lazy.ts` 中的 `lazyStream()`。

请求热路径不做 model 规范化：`stream()` 原样使用传入的 model 对象。如果应用需要最新模型元数据，应在开始轮次前刷新 provider 并重新读取（`await models.refresh(p); models.getModel(p, id)`）。

## `src/api` 下的 API 实现

API 实现是一段可复用的 stream 行为，并不等同于 provider。

统一导出契约：每个真实实现模块都只导出以下内容：

```ts
// src/api/anthropic-messages.ts — imports SDKs
export function stream(model, context, options) { ... }
export function streamSimple(model, context, options) { ... }
```

这样模块本身就符合 `ProviderStreams`，因此延迟加载包装器只需一个通用辅助函数，无需为每种 API 单独编写胶水代码。`ProviderStreams` 是不携带具体类型的分派形状（实现模块导出的是具体类型函数，无法赋给泛型方法）；每种 API 的选项类型由各自模块以及 `Provider.stream()` 上的 `ApiStreamOptions` 提供：

```ts
export interface ProviderStreams {
  stream(model: Model<Api>, context: Context, options?: StreamOptions): AssistantMessageEventStream;
  streamSimple(model: Model<Api>, context: Context, options?: SimpleStreamOptions): AssistantMessageEventStream;
}

// src/api/lazy.ts
export function lazyApi(load: () => Promise<ProviderStreams>): ProviderStreams;

// src/api/anthropic-messages.lazy.ts
export const anthropicMessagesApi = (): ProviderStreams => lazyApi(() => import("./anthropic-messages.ts"));
```

导入链：

```txt
provider module -> lazy API wrapper -> dynamic import(real API impl) -> SDK deps
```

备注：

- Bedrock 在自己的延迟加载包装器中保留仅限 Node 的动态导入技巧（`importNodeOnlyProvider`、`.ts`/`.js` 说明符重写）。`setBedrockProviderModule()`（Bun 构建会用到）移入 bedrock 的延迟加载包装器模块。
- 共享辅助模块（`openai-responses-shared.ts`、`google-shared.ts`、`transform-messages.ts`、prompt cache、Copilot headers）移到 `src/api/`，与实现模块放在一起。

## 多个具体 Provider 共享 API 实现

许多具体 provider 会共享同一个 API 实现（例如 OpenRouter、Groq、Cerebras、xAI、ZAI 等都使用 OpenAI completions）。它们通过引用共享延迟加载的 API 对象：

```ts
import { openAICompletionsApi } from "../api/openai-completions.lazy.ts";

export function openrouterProvider(): Provider {
  return createProvider({
    id: "openrouter",
    name: "OpenRouter",
    baseUrl: "https://openrouter.ai/api/v1",
    auth: { apiKey: envApiKeyAuth("OpenRouter API key", ["OPENROUTER_API_KEY"]) },
    models: OPENROUTER_MODELS,
    api: openAICompletionsApi(),
  });
}
```

这保留了 Vercel AI SDK 的一个优点：用户导入具体 provider，共享的协议实现则封装在内部。

## 认证

请求认证信息的输出保持精简：

```ts
export interface ModelAuth {
  apiKey?: string;
  headers?: Record<string, string>;
  baseUrl?: string;
}
```

如果某个值无法表示为 `apiKey`、`headers` 或 `baseUrl`，它就属于 provider 配置，而不是认证信息（Vertex 的 project/location、Bedrock 的 region/profile、Azure 的 apiVersion 都是 provider 工厂选项）。

### Provider 认证

`Provider.auth` 恰好有两个槽位；真实 provider 至多有一种 API key 认证路径和一种 OAuth 认证路径。槽位名称直接表达 UI 中 OAuth 与 API key 的区分，无需 `kind` 判别字段或方法 ID：

```ts
export interface ProviderAuth {
  apiKey?: ApiKeyAuth; // stored key/provider env + ambient env/files/ADC/IAM
  oauth?: OAuthAuth;   // login flow + refresh
}

export interface ApiKeyAuth {
  name: string; // "Anthropic API key"

  /** Interactive setup (prompt for key/provider env). Absent = ambient-only (env, ADC, IAM). */
  login?(callbacks: AuthLoginCallbacks): Promise<ApiKeyCredential>;

  /**
   * Resolve auth from the stored credential and/or ambient sources, merging
   * per field (credential.key ?? env("..."), credential.env?.NAME ?? env("...")).
   * undefined = not configured.
   */
  resolve(input: {
    model: Model<Api>;
    ctx: AuthContext;
    credential?: ApiKeyCredential;
  }): Promise<AuthResult | undefined>;
}

export interface OAuthAuth {
  name: string; // "Anthropic (Claude Pro/Max)"

  login(callbacks: AuthLoginCallbacks): Promise<OAuthCredential>;

  /** Exchange the refresh token. Network call; throws on failure (invalid_grant etc.). Runs under the store lock. */
  refresh(credential: OAuthCredential): Promise<OAuthCredential>;

  /** Side-effect-free derivation of request auth from a valid credential. Covers Copilot-style per-credential baseUrl. Async so lazy wrappers can load the implementation. */
  toAuth(credential: OAuthCredential): Promise<ModelAuth>;
}

export interface AuthResult {
  auth: ModelAuth;
  /** Human-readable label for status UI: "ANTHROPIC_API_KEY", "OAuth", "~/.aws/credentials". */
  source?: string;
}

export interface AuthContext {
  env(name: string): Promise<string | undefined>;
  fileExists(path: string): Promise<boolean>; // supports leading ~
}
```

OAuth 拆分为 `refresh` + `toAuth`，而不是一个 `resolve`，与旧的 `OAuthProviderInterface`（`refreshToken` + `getApiKey`）相符；同时让 `Models` 能自行管理锁定流程，无需通过复杂闭包绕行：refresh 生成凭据，`toAuth` 则根据最终存储的凭据派生请求认证信息。

这里没有 `usesCallbackServer` 标志。使用 `prompt()`/`notify()` 回调后，流程可以在运行时自行表明行为：启动回调服务器的流程会发出 `manual_code` prompt，与服务器并行等待；回调先完成时就中止该 prompt。UI 无需事先静态判断流程类型。

### 凭据

每个 provider 保存一个带类型标签的凭据，与当前 `auth.json` 的结构完全一致（每个 provider ID 对应 `type: "api_key" | "oauth"`）：

```ts
export interface ApiKeyCredential {
  type: "api_key";
  key?: string;
  env?: ProviderEnv; // e.g. Cloudflare account/gateway ids, Azure/Vertex/Bedrock scoped config
}

export interface OAuthCredential extends OAuthCredentials {
  type: "oauth"; // access, refresh, expires from OAuthCredentials
}

export type Credential = ApiKeyCredential | OAuthCredential;
```

`ApiKeyCredential.env` 与 key 一起或单独保存 provider 作用域的环境/配置值。`ApiKeyAuth.resolve()` 会逐字段合并：`credential.key ?? env("CLOUDFLARE_API_KEY")`、`credential.env?.CLOUDFLARE_ACCOUNT_ID ?? env("CLOUDFLARE_ACCOUNT_ID")` 等。凭据判别值有意沿用当前 `auth.json` 中的 `api_key`，避免文件存储层进行有损类型转换。

### 凭据存储

存储由应用注入；`pi-ai` 提供默认的内存存储。以 provider ID 为键，每个 provider 对应一个凭据：

```ts
export interface CredentialStore {
  /** Read the stored credential, possibly expired. Display/status use; request auth comes from Models.getAuth(). */
  read(providerId: string): Promise<Credential | undefined>;

  /**
   * Serialized write — the only write path. fn sees the current credential
   * because correct writes (refresh, login-during-refresh) depend on it;
   * return the new credential, or undefined to leave the entry unchanged.
   * Mutual exclusion per provider id, cross-process too where the backing
   * store supports it (file lock). Resolves with the post-write credential.
   */
  modify(
    providerId: string,
    fn: (current: Credential | undefined) => Promise<Credential | undefined>,
  ): Promise<Credential | undefined>;

  /** Remove (logout). Serialized against modify. */
  delete(providerId: string): Promise<void>;
}
```

这里有意不提供 `set`：未串行化的写入路径容易产生读-改-写竞态（例如刷新期间登录覆盖新凭据、重复刷新 token）。调用方式如下：

```ts
await store.modify(pid, async () => credential);      // login: store this
await store.read(pid);                                // status UI ("logged in via OAuth")
await store.delete(pid);                               // logout
// refresh RMW happens inside Models.getAuth
```

错误语义：条目不存在时，`read` 返回 `undefined`；只有存储失败时方法才会拒绝，`Models` 会将此类拒绝包装为错误码为 `"auth"` 的 `ModelsError`。尽力而为的存储实现也有效，例如提供内存视图并在内部记录持久化错误（当前 `AuthStorage` 的行为）。

### 解析策略（固定）

`Models.getAuth(model)` 是一棵决策树，而不是循环。已存储的凭据决定该 provider 的认证方式；只有没有存储凭据时才会检查环境等外部来源（与 `AuthStorage` 保持一致：刷新失败或凭据类型不匹配时，不会静默回退到环境变量）：

```ts
const stored = await store.read(provider.id);
if (stored) {
  if (stored.type === "oauth" && provider.auth.oauth) {
    const oauth = provider.auth.oauth;
    let credential = stored;
    if (Date.now() >= credential.expires) {                 // optimistic check, lock-free
      const post = await store.modify(provider.id, async (current) => {
        if (current?.type !== "oauth") return undefined;    // logged out meanwhile
        return Date.now() >= current.expires                // authoritative check, under lock
          ? oauth.refresh(current)                          // throws -> ModelsError("oauth")
          : undefined;                                      // another process/request refreshed
      });
      if (post?.type !== "oauth") return undefined;
      credential = post;
    }
    return { auth: await oauth.toAuth(credential), source: "OAuth" };
  }
  if (stored.type === "api_key" && provider.auth.apiKey) {
    return provider.auth.apiKey.resolve({ model, ctx, credential: stored });
  }
  return undefined; // stored credential without matching handler blocks ambient
}
return provider.auth.apiKey?.resolve({ model, ctx, credential: undefined }); // ambient
```

特性：

- 双重检查锁，与当前的 `refreshOAuthTokenWithLock` 相同：有效 token 只需一次 `read`，不需要加锁；过期 token 会加锁，在锁内再次检查，全局只刷新一次，并在释放锁前持久化。
- 显式请求认证信息（stream options 中的 `apiKey`/`headers`）会在 `stream()` 中按字段合并到最外层，并覆盖其他值。
- 刷新失败时以 `ModelsError("oauth")` 拒绝；已存储凭据保持不变（供重试使用）。请求路径会将错误及其真实原因作为 stream error 抛出（提示“运行 `/login`”）；状态/可用性 UI 应捕获拒绝，并显示“需要重新登录”。这是 `getAuth` 文档化的契约。

### 替换 AuthStorage

coding-agent 的最终状态是删除 `AuthStorage`；它的能力由 `CredentialStore` 实现及其组合方式承接。

当前 `getApiKey` 的优先级及新归属：

| 现有 AuthStorage | 新设计 |
|---|---|
| 运行时覆盖（CLI `--api-key`） | `withRuntimeOverrides(store, overrides)` 装饰器：`read` 将覆盖值作为 `ApiKeyCredential` 返回；不持久化 |
| 已存储的 `api_key`（通过 `resolveConfigValue` 处理 `$ENV`/`!command`） | 已存储的 `ApiKeyCredential`；配置值由 coding-agent 的 adapter/decorator 在 `read` 时解析（命令执行策略仍由应用决定） |
| 已存储的 `oauth` + 加锁刷新，失败时返回 `undefined` | 使用上文的 `getAuth` 决策树；失败时带原因拒绝，而不是静默地视为未配置 |
| 环境变量（仅在没有已存储内容时） | `apiKey.resolve` 的外部环境分支 |
| `fallbackResolver`（models.json 自定义 provider） | 删除；自定义 provider 自行提供 `auth.apiKey` |

```txt
FileCredentialStore        ports AuthStorage's lock backend: read = memory snapshot,
                           modify = withLockAsync(re-read, fn, merge-write), delete,
                           internal error recording (drainErrors equivalent)
└─ withConfigValues        $ENV / !command at read
   └─ withRuntimeOverrides --api-key
      └─ createModels({ credentials: store })

login/logout UI            provider.auth.{oauth,apiKey}.login(callbacks) + store.modify/delete
status UI                  store.read(pid) + getAuth try/catch ("needs /login" on rejection)
getOAuthProviders          presence of provider.auth.oauth across registered providers
```

### 登录回调

API key 和 OAuth 登录共用一个接口：

```ts
export interface AuthLoginCallbacks {
  /** Aborts the whole login flow. Per-prompt cancellation uses AuthPrompt.signal. */
  signal?: AbortSignal;

  prompt(prompt: AuthPrompt): Promise<string>;
  notify(event: AuthEvent): void;
}

/** `signal` lets the flow cancel a pending prompt when an out-of-band event resolves the step. */
export type AuthPrompt = { signal?: AbortSignal } & (
  | { type: "text"; message: string; placeholder?: string }
  | { type: "secret"; message: string; placeholder?: string }
  | { type: "select"; message: string; options: readonly { id: string; label: string; description?: string }[] }
  | { type: "manual_code"; message: string; placeholder?: string }
);

export type AuthEvent =
  | { type: "auth_url"; url: string; instructions?: string }
  | { type: "device_code"; userCode: string; verificationUri: string; intervalSeconds?: number; expiresInSeconds?: number }
  | { type: "progress"; message: string };
```

`prompt()` 返回输入/选择的字符串（`select` 返回选项 ID）。流程可以让 `manual_code` prompt 与回调服务器竞争：设置 `AuthPrompt.signal`，并在回调先完成时中止 prompt。

### 接入 OAuth

支持 OAuth 的 provider 总是附带 OAuth 能力。工厂不提供开关：流程会延迟加载，因此在实际运行 `login()`/`refresh()` 之前，声明支持 OAuth 不会产生开销；从未登录的宿主也不会加载 OAuth 实现。

```ts
export function anthropicProvider(): Provider {
  return createProvider({
    id: "anthropic",
    name: "Anthropic",
    baseUrl: "https://api.anthropic.com/v1",
    auth: {
      apiKey: envApiKeyAuth("Anthropic API key", ["ANTHROPIC_API_KEY"]),
      oauth: lazyOAuth({
        name: "Anthropic (Claude Pro/Max)",
        load: () => import("../utils/oauth/anthropic.ts").then((m) => m.anthropicOAuth),
      }),
    },
    models: ANTHROPIC_MODELS,
    api: anthropicMessagesApi(),
  });
}
```

`lazyOAuth()` 包装动态导入的 `OAuthAuth`，使 provider 定义可以声明支持 OAuth，而无需导入具体实现（`toAuth` 正是因此设计为异步）：

```ts
export function lazyOAuth(input: {
  name: string;
  load: () => Promise<OAuthAuth>;
}): OAuthAuth;
```

OAuth 不得把 Node 专属代码（`node:http`、`node:crypto`）带入浏览器 bundle：`lazyOAuth()` 内的动态导入使用与 bedrock 延迟包装器相同的、对 bundler 不透明的变量说明符技巧。浏览器宿主不会触发加载（没有存储的 Node OAuth 凭据，也不会运行登录流程）。未来如果加入 Web OAuth（sitegeist 已证明可行：Web Crypto PKCE、认证标签页、通过 fetch 交换 token、轮询 device code），它只需成为另一种 `OAuthAuth` 实现，无需预留选项值。

`src/utils/oauth/` 中现有的流程（anthropic、openai-codex、github-copilot）会适配到 `OAuthAuth`（用 `login`/`refresh`/`toAuth` 替换 `login`/`refreshToken`/`getApiKey`/`modifyModels`），使用新的回调，并保持面向 Node 且延迟加载。Copilot 的 `modifyModels` baseUrl 改写会变为由 `toAuth` 返回 `ModelAuth.baseUrl`。

## Provider 包装器与 models.json

`models.json` 是 provider 包装层，不会原地修改 provider：

```ts
function withProviderOverrides(base: Provider, overrides: ProviderOverrides): Provider {
  return {
    ...base,
    name: overrides.name ?? base.name,
    baseUrl: overrides.baseUrl ?? base.baseUrl,
    headers: mergeHeaders(base.headers, overrides.headers),

    getModels: () => applyModelOverrides(base.getModels(), overrides.models),
    refreshModels: base.refreshModels?.bind(base),

    stream: base.stream,
    streamSimple: base.streamSimple,
  };
}
```

这种方式也适用于动态 provider，因为 `getModels()` 委托给底层来源，而 `refreshModels()` 会透传调用。

来自 models.json 的请求认证配置（`$ENV`、`!command`、内联 key）仍由应用作为 sidecar 状态管理；应用可以将其作为显式请求认证信息提供，也可以创建自定义 `ApiKeyAuth` 并设置到包装后 provider 的 `auth.apiKey` 上。

## 自定义 Provider：createProvider()

一个辅助函数根据各个部分构建 provider，同时支持单 API 和混合 API provider：

```ts
export function createProvider(input: {
  id: string;
  name?: string;                 // default: id
  baseUrl?: string;
  headers?: Record<string, string>;
  auth: ProviderAuth;            // required, at least one of apiKey/oauth (no "no-auth" providers)
  /** Initial model list (empty for purely dynamic providers). */
  models: readonly Model<Api>[];
  /** Dynamic providers: fetch the current list; createProvider stores it and dedupes in-flight calls. */
  refreshModels?: () => Promise<readonly Model<Api>[]>;
  /** Single implementation, or map keyed by model.api for mixed-API providers. */
  api: ProviderStreams | Record<string, ProviderStreams>;
}): Provider;
```

- 单个 `api`：所有模型都通过它发起 stream。
- `api` 映射：`stream()`/`streamSimple()` 根据 `model.api` 分派；未知 API 会产生 stream error。

必须支持混合 API 的自定义 provider（opencode Go/Zen 风格的 provider 会在同一个 provider ID 下公开由不同 API 支持的模型）。

内置 provider 工厂内部使用 `createProvider()`。models.json 中的自定义 provider 直接映射到它：

```json
{
  "providers": {
    "my-openai-proxy": {
      "api": "openai-completions",
      "baseUrl": "https://proxy.example/v1",
      "models": [ ... ]
    }
  }
}
```

## Compat 入口

在 coding-agent 迁移完成并删除旧接口之前，`@earendil-works/pi-ai/compat` 会保留旧的全局 API。新代码绝不导入它。

保留的旧语义：全局 `stream()` 仍可通过旧 api-registry 按 `model.api` 分派，以支持自定义 provider、被修改的 model，以及覆盖内置 API 实现的测试/扩展。

- `stream/complete/streamSimple/completeSimple(model, ctx, opts)`：与真实内置 provider/model/API 匹配的调用会通过单例 `builtinModels()` 集合路由，因此与新运行时共用 provider 认证/环境变量/baseUrl 行为。未知 provider、被修改过的 model 或被覆盖的 API 注册，则回退到 api-registry 分派并注入 `getEnvApiKey`。
- 内置 API 注册副作用从根 barrel 移入 compat。如果某个 API ID 已注册，则跳过，因为 compat 可能在测试或扩展注册覆盖项之后才加载。`registerApiProvider()`/`unregisterApiProviders()` 继续写入 compat 本地注册表；`resetApiProviders()` 会清空并重新注册内置项。
- 同步的 `getModel/getModels/getProviders` 是 `providers/all` 中 `getBuiltinModel/getBuiltinModels/getBuiltinProviders` 的已弃用别名（它们始终只是读取生成目录；已确认旧 `modelRegistry` 从未被修改）。
- 重新导出每种 API 的延迟 stream 包装器（包括 `setBedrockProviderModule`）、`env-api-keys.ts` 和图像生成注册表/目录；这些内容都不再留在根 barrel 中。
- `export * from "./index.ts"`：compat 是核心入口的严格超集，因此消费者可以直接整体更换文件的导入路径，无需逐个调整符号。

coding-agent（以及过渡期的 agent package）会把这些符号的导入路径从 `@earendil-works/pi-ai` 改为 `@earendil-works/pi-ai/compat`。这只是导入路径变更；在 ModelManager 迁移之前，其余部分保持不动。

扩展兼容期：coding-agent 扩展加载器（jiti aliases + Bun `virtualModules`）会将 `@earendil-works/pi-ai` 根说明符解析到 compat 入口。现有使用旧全局 API（`complete`、`getModel`、`registerApiProvider` 等）的用户扩展，无需修改即可继续运行；直到 ModelManager 迁移时 compat 被移除才会失效，届时 changelog 会提供迁移指南。类型检查会提示正确路径：编辑器将根入口解析为精简的核心类型，因此要通过类型检查的扩展源码必须从 `/compat` 导入旧全局 API；仓库中的示例扩展就是这样做的。

## 内置静态辅助函数

有类型、同步且只查询生成目录的辅助函数与目录放在一起（从 `providers/all` 导出）：

```ts
getBuiltinModel(provider, id)   // sync, typed overloads from generated catalog
getBuiltinModels(provider)      // sync
getBuiltinProviders()           // sync
```

通过 `Models` 实例进行的运行时查找会同步查询 provider 的最近已知列表：`models.getModel(...)`。对新鲜度有要求的调用方应先运行 `await models.refresh(provider)`。

通过更新 `packages/ai/scripts/generate-models.ts`，将生成的目录按 provider 拆分（`providers/<id>.models.ts`）。如果修改生成器对于本阶段来说太大，可以暂缓拆分；`providers/all` 和 provider 工厂可以暂时导入单体的 `models.generated.ts`，并依靠 `sideEffects: false` 进行裁剪。

## Tree-shaking 与延迟导入

规则：

1. 主入口 `@earendil-works/pi-ai` 只包含核心内容。
2. Provider 模块只导入自己的目录、认证辅助函数和延迟加载 API 包装器。
3. 延迟加载 API 包装器会动态导入真实 API 实现。
4. 真实 API 实现会导入 SDK 依赖。
5. OAuth 实现始终通过 `lazyOAuth()` 接入，并藏在对 bundler 不透明的动态导入之后延迟加载；provider 元数据绝不提前导入仅限 Node 的 OAuth 代码。
6. `providers/all` 会导入所有内置 provider 工厂和全部目录，是明确的重型入口。
7. Provider 模块没有副作用；导入 provider 不会进行任何全局注册。
8. `package.json` 的 `sideEffects` 只列出会产生副作用的 compat/图像注册文件；根入口和 provider 模块保持可 tree-shake。
9. 启用代码分割时，provider SDK 会留在延迟加载的代码块中。不启用代码分割时，bundler 会将静态可达的延迟 API 实现合并进单个 bundle；此时 `providers/all` 会拉入所有静态可见的 SDK。Bedrock 是例外：其 AWS SDK 实现在对 bundler 不透明、仅限 Node 的导入之后；打包成独立单文件时需要 `setBedrockProviderModule()`。

Exports 映射示意：

```json
{
  "exports": {
    ".": "./dist/index.js",
    "./compat": "./dist/compat.js",
    "./providers/all": "./dist/providers/all.js",
    "./providers/openai": "./dist/providers/openai.js",
    "./providers/anthropic": "./dist/providers/anthropic.js",
    "./providers/*": "./dist/providers/*.js",
    "./api/*": "./dist/api/*.js"
  }
}
```

浏览器 smoke check（`scripts/check-browser-smoke.mjs`）必须持续通过：打包核心入口（以及任何非 Node 的 provider 入口）不得引入 `node:http`/`node:crypto`。

## 与 AgentHarness 集成

`AgentHarness` 接收一个 `Models` 实例。

- 必须提供 `AgentHarnessOptions.models`。
- Harness 不会把 `Models` 快照写入轮次状态。
- 请求路径会调用 `this.models.streamSimple(model, context, options)`；压缩/分支摘要路径也相同。
- 请求路径绝不会调用异步 `models.getModel()` 做规范化；如果需要刷新模型元数据，应用应在开始轮次前更新所选 model。
- Harness 测试会构造 `createModels()`，并安装 faux provider（来自 `providers/faux` 的 `fauxProvider()` 工厂）。

## coding-agent 下一阶段工作（不属于本阶段）

coding-agent 会分层构建 provider，并按 session 绑定：

```txt
built-in providers (builtinModels)
-> models.json provider wrappers / custom providers (createProvider)
-> extension provider wrappers/additions
```

```ts
sessionModels.clearProviders();
for (const provider of layeredProviders) sessionModels.setProvider(provider);
```

以下内容由 coding-agent 负责：替代 AuthStorage 的 `FileCredentialStore` + 装饰器（见“替换 AuthStorage”）、models.json 认证 sidecar（`$ENV`、`!command`）、命令执行策略、provider 状态标签（来自 `AuthResult.source`）、登录/登出 UI（通过 `prompt()`/`notify()` 调用 `auth.{apiKey,oauth}.login()`）、扩展生命周期，以及 provider 管理斜杠命令。

当前过渡状态：

- `AgentHarness` 已经接收 `Models` 实例，并用它处理轮次 stream、压缩和分支摘要。
- coding-agent 还没有使用 `AgentHarness`；`AgentSession` 仍通过 `streamFn` 驱动底层 `Agent`。
- coding-agent 仍使用旧的 `AuthStorage` + `ModelRegistry`，并通过 `@earendil-works/pi-ai/compat` 导入旧的全局 pi-ai API。
- 扩展加载器仍将 pi-ai 根入口别名到 `/compat`，作为旧扩展的运行时过渡期兼容措施。

## 实现待办

完成后勾选相应条目。请保持此列表最新；它记录了后续会话继续工作时所需的当前状态。

### 阶段 1 — 核心类型/运行时

- [x] 将 `types.ts` 中的 `Provider` 别名重命名为 `ProviderId`，并修正使用位置。
- [x] 在 `types.ts` 中添加 `ApiOptionsMap` 和 `ApiStreamOptions<TApi>`（仅类型导入）。
- [x] 新建 `models.ts`：包含 `Provider<TApi>` 接口、`hasApi()` 守卫、`ModelsError` 及错误码。认证类型放在 `src/auth/types.ts`（`ProviderAuth` = `{ apiKey?, oauth? }`、凭据、`CredentialStore`（`read`/`modify`/`delete`，每个 provider 一个凭据）、`AuthResult`、`AuthContext`、`ModelAuth`、登录回调）；内存存储位于 `src/auth/credential-store.ts`；默认 context 位于 `src/auth/context.ts`（浏览器安全的 node:fs 技巧）；`lazyStream()` 位于 `src/api/lazy.ts`。
- [x] 实现带 provider 映射的 `Models`/`MutableModels`/`createModels({ credentials?, authContext? })`：同步 `getModel(s)`（隔离各 provider 的失败）、显式异步 `refresh(provider?)`、`getAuth`（决策树及双重检查加锁刷新），以及逐字段合并认证信息的 `stream/complete/streamSimple/completeSimple`。测试：`packages/ai/test/models-runtime.test.ts`。
- [x] 保留元数据辅助函数：`calculateCost`、`getSupportedThinkingLevels`、`clampThinkingLevel`、`modelsAreEqual`。

### 阶段 2 — `src/api/`

- [x] 将 stream 实现从 `src/providers/` 移至 `src/api/`，并按 API ID 重命名（例如 `anthropic.ts` -> `api/anthropic-messages.ts`）。
- [x] 统一每个实现模块的导出，使其恰好导出 `stream` 和 `streamSimple`。
- [x] 将共享辅助模块（`openai-responses-shared`、`google-shared`、`transform-messages`、`openai-prompt-cache`、`github-copilot-headers`、`cloudflare`、`simple-options`）移至 `src/api/`。
- [x] 将 `lazyStream()`/`lazyApi()` 提取到 `src/api/lazy.ts`。
- [x] 为每种 API 添加 `*.lazy.ts` 包装器；bedrock 保留仅限 Node 的导入技巧及 `setBedrockProviderModule()`。
- [x] 删除 `providers/register-builtins.ts`。到阶段 5 compat 完成前的过渡期间，内置 api-registry 注册放在 `stream.ts` 中；延迟 API 包装器从根 barrel 导出。

### 阶段 3 — Provider 工厂 + 目录

- [x] 在 `src/auth/helpers.ts` 中添加认证辅助函数：`envApiKeyAuth()`（包含 secret-prompt `login`）和 `lazyOAuth()`。OAuth 流程通过 `utils/oauth/load.ts` 加载（对 bundler 不透明的动态导入）；它引用的 `OAuthAuth` 导出在阶段 4 落地。
- [x] 在 `models.ts` 中实现 `createProvider()`（支持单一和混合 `api` 映射，根据 `model.api` 分派，遇到未知 API 时返回 stream error）。
- [x] 在 `src/providers/` 下为所有内置目录 provider 添加独立工厂；通过 `lazyOAuth()` 接入 OAuth（anthropic、openai-codex、github-copilot）；为 amazon-bedrock（AWS 环境变量/profile）和 google-vertex（key 或 ADC+project+location）提供外部环境 `ApiKeyAuth`。
- [x] 在 `providers/all.ts` 中导出 `builtinProviders()`、`builtinModels()`、`getBuiltinModel/getBuiltinModels/getBuiltinProviders`。
- [x] 为测试添加 faux provider 工厂（`providers/faux.ts` 中的 `fauxProvider()`）；旧的 `registerFauxProvider()` 保留到 compat 移除为止。
- [x] 通过 `scripts/generate-models.ts` 将生成的目录按 provider 拆分（`providers/<id>.models.ts`）；`models.generated.ts` 改为生成的汇总文件。

### 阶段 4 — OAuth 适配

- [x] 将 `utils/oauth/anthropic.ts`、`openai-codex.ts`、`github-copilot.ts` 适配到 `OAuthAuth`（`login`/`refresh`/`toAuth`）和 `prompt()`/`notify()`；`modifyModels` 的 baseUrl 改写变为 `toAuth().baseUrl`。新导出（`anthropicOAuth`、`openaiCodexOAuth`、`githubCopilotOAuth`）与旧 `OAuthProviderInterface` 对象并存，后者保留到阶段 7。
- [x] `OAuthAuth` 不设 `usesCallbackServer`：回调服务器流程会与 `manual_code` prompt 竞争（流程完成后通过 `AuthPrompt.signal` 中止 prompt）。旧接口的标志会保留到 compat 移除。

### 阶段 5 — 打包

- [x] `index.ts` 仅包含核心且无副作用（不含目录、provider 工厂、api-registry、env-api-keys、images、OAuth 或 compat）。有类型的目录查询（`getBuiltin*`）在 `providers/all.ts` 中实现；`models.ts` 不再导入 `models.generated.ts`。
- [x] 添加 `compat.ts`：它是 index 的超集，包含旧 API 分派全局函数、已弃用的 `getModel/getModels/getProviders` 别名、延迟 API 包装器和 `setBedrockProviderModule`、`getEnvApiKey`、图像 API。注册副作用移至此处（若已有注册则跳过）。
- [x] 添加子路径导出映射（`./compat`、`./providers/*`、`./api/*`）；`sideEffects` 数组只列出有副作用的模块（compat、图像注册），不再设为 `false`。
- [x] 浏览器 smoke（入口现在从 `/compat` 导入旧全局 API）和 shrinkwrap 检查均通过。内部旧全局 API 导入已切换到 `/compat`（agent/coding-agent/examples 中共 42 个文件；vitest 配置将 `/compat` 别名到 src；spawn-CLI 测试从 workspace dist 解析，因此重新构建了 `packages/ai` 和 `packages/agent` 的 dist）。

### 阶段 6 — AgentHarness

- [x] 要求提供 `AgentHarnessOptions.models`（harness 上为 `readonly models`）；harness stream 路径使用 `models.streamSimple()`。重新以结构类型定义 `StreamFn`（不依赖 compat 类型）；`Models.streamSimple` 满足该接口。
- [x] 压缩/分支摘要使用 harness 的 `Models` 实例。彻底移除 `getApiKeyAndHeaders`——`Models` 是唯一认证路径；每次请求的 key 解析成为集合中 provider auth 的职责。`compact()`/`generateSummary()`/`generateBranchSummary()` 不再接收显式 `apiKey`/`headers` 参数。
- [x] Harness 测试使用 `createModels()` + `fauxProvider()`，并为每个 faux provider 分配唯一 ID；不再依赖全局 api-registry 状态，也不需跟踪注销操作。

### 阶段 7 — coding-agent 桥接（最小范围）

- [x] 将旧全局 API 导入切换到 `@earendil-works/pi-ai/compat`（随阶段 5 完成；compat 是超集，因此只改了路径）。扩展加载器在运行时过渡期将 pi-ai 根入口解析到 compat。
- [x] 此处原先设想的其他工作都要等 coding-agent 真正通过 `Models` 实例发起 stream；目前 coding-agent 的 `AgentSession` 通过 `streamFn` 驱动底层 `Agent`，而不是使用 harness。因此这些工作移到了阶段 9。

### 阶段 8 — 收尾

- [x] 更新/添加测试并运行受影响的测试套件（每个阶段都添加相应测试；整个过程中 `./test.sh` 均通过）。
- [x] 在 `packages/ai/CHANGELOG.md` 的 `### Breaking Changes` 中添加迁移指南（compat 入口、`Provider` -> `ProviderId`、API 模块迁移），并在 `### Added` 中记录新的 Models/provider/auth API。
- [x] 在 `packages/coding-agent/CHANGELOG.md` 添加面向扩展作者的 `### Changed` 条目：运行时不受影响（加载器将 pi-ai 根入口解析到 compat），类型检查会提示改用 `/compat` 或新 API；之后移除 compat 时再提供迁移指南。
- [x] 在 `packages/agent/CHANGELOG.md` 的 `### Breaking Changes` 中记录必须提供 `AgentHarnessOptions.models`、压缩方法签名变更和结构化 `StreamFn`。
- [x] `npm run check` 通过。

### 阶段 9 — coding-agent 使用 Models + CredentialStore（本次范围）

coding-agent 使用 `FileCredentialStore` + `MutableModels` 集合替换 AuthStorage 和 ModelRegistry 的内部实现。`AgentSession` 本身保留（采用 AgentHarness 属于 pi 2.0 工作）；只替换其 model/auth 基础层。分层方向严格单向：

```txt
FileCredentialStore (auth.json, locked, $ENV/!command resolution) + explicit --api-key overlay
        ↑
MutableModels: builtin factories (wrapped per models.json config) + custom providers (models.json ∪ extensions)
        ↑
ModelRegistry: compatibility facade — sync last-known reads delegate to the collection; registerProvider/login/logout/status for extensions + UI
        ↑
AgentSession / sdk / interactive-mode (stream via models; await only auth/refresh paths)
```

决策：

- `AuthStorage` 类型会被删除，否则它依赖 provider auth，而 provider auth 又依赖它的 store，形成循环依赖。其接口拆分为：`get`/`set`/`remove` -> `CredentialStore`；`getApiKey` -> `Models.getAuth`；`login`/`logout`/`getAuthStatus` -> 基于 `provider.auth.oauth` + store 实现的 ModelRegistry facade 方法。
- `FileCredentialStore` 自包含（路径、锁、解析/写入、chmod、错误缓存），并负责 `auth.json` 语义，包括解析已存储 API key 凭据中的 `$ENV`/`!command`。持久化值保持原始形式；解析后会返回副本供认证使用。
- 运行时 `--api-key` 覆盖是显式的 store overlay（读取覆盖项时会得到临时存储的 api-key 凭据，并屏蔽已存储的 OAuth；与当前优先级一致）。每个已注册 provider 都保证有 `apiKey` 认证槽位，因此覆盖对仅支持 OAuth 的 provider 也有效。
- 为兼容 SDK 和扩展，`ModelRegistry.getAll`/`find`/`getAvailable` 保持同步；它们委托给集合最近已知的同步模型列表，并快速判断认证是否看起来已配置。动态 provider 通过显式异步 `refresh()` 更新，请求认证仍通过 `getApiKeyAndHeaders()`/`Models.getAuth()` 异步完成。扩展也会获得集合本身，作为新的 API。
- models.json 保持完整功能兼容，通过 provider 装饰实现：包装内置工厂，使 `getModels()` 应用 provider 级 `baseUrl`/`compat` 覆盖、`modelOverrides` 和自定义模型合并（支持异步）；provider 的 `apiKey`/`headers`/`authHeader` 配置转换为该 provider 的 `ApiKeyAuth`（优先使用配置，工厂认证作为回退）；解析错误保留 `getError()` 语义。
- 扩展 `ProviderConfig` 保持兼容：支持按 provider 配置 `streamSimple`、将旧式 `oauth` 适配到 `OAuthAuth`（`modifyModels` -> 包装 `getModels()` + `toAuth`），并支持完整替换每个 provider 的模型。对于调用全局 `complete()` 的消费者，旧式 `registerApiProvider` 注册仍只写入 compat 本地注册表；compat 删除后该能力随之移除。
- Copilot：在存在已存储凭据时包装 `getModels()` 并应用 baseUrl（确保扩展可见的模型信息正确），同时让每次请求的 `toAuth().baseUrl` 生效。
- Cloudflare：改用 provider auth（key + 来自凭据 `env` 或外部 `AuthContext.env()` 的 `CLOUDFLARE_ACCOUNT_ID`/`CLOUDFLARE_GATEWAY_ID` -> `ModelAuth.baseUrl`）。内置 compat 调用会经由 `Models` 路由，因此使用相同的 provider auth 路径。

新会话的执行顺序：

1. [x] 先完成 pi-ai 重构：`Provider.getModels()` 同步，另有可选的 `refreshModels()`；`Models.getModels`/`getModel` 同步，`Models.refresh(provider?)` 异步；`createProvider` 接收 `models` 数组和可选的 `refreshModels` 获取函数（会合并进行中的重复请求）。这推翻了阶段 1 关于异步模型列表的决定；理由见“Provider 模型列表”（同步/异步联合类型会引入潜在的同步假设；仅异步会破坏扩展 `find`/`getAll` 等同步消费者接口）。
2. [x] 在 pi-ai 工厂中实现 Cloudflare provider auth：Workers AI 和 AI Gateway 会验证所需的 account/gateway 环境变量/配置，并通过 provider auth 返回解析出的 `baseUrl`、provider 作用域环境值，以及 header 禁用/覆盖元数据。
3. [ ] 在 coding-agent 中添加 `FileCredentialStore`。
   - 将 pi-ai 的 `CredentialStore` 接口实现为独立的 `auth.json` 存储；不要依赖旧的 `AuthStorageBackend` 抽象，但可以移植其锁定/重试语义。
   - 保留现有文件格式。`ApiKeyCredential` 使用 `{ type: "api_key", key?, env? }`，与当前 `auth.json` 一致；不要把 `env` 转成 metadata，也不要改写判别字段。
   - 使用注入的执行/配置环境，开箱即用地解析已存储 API key 的 `key` 和 `env` 值中的 `$ENV`/`!command`。`$ENV` 应从该环境中查找，`!command` 应通过共享 shell 执行路径运行，而不是直接调用 `execSync`。
   - 持久化原始配置值；为认证返回的已解析凭据必须是副本，除非调用方显式存储新值，否则不能改写 `$ENV`/`!command` 字符串。
   - `read(provider)` 返回当前凭据快照，并记录解析/存储错误，以保持状态 UI 行为一致。
   - `modify(provider, fn)` 必须加锁、重新读取、执行 `fn`、合并写入 provider 条目、设置 chmod `0600`，并返回写入后的凭据。
   - `delete(provider)` 必须加锁，且只能删除该 provider 的条目。
   - 添加文件存储和内存存储测试，覆盖锁/RMW 行为、带配置值解析的 `api_key` 读取、OAuth 读取、保留 provider `env`、删除、解析错误和类似并发刷新的修改。
4. [ ] 根据 coding-agent 策略添加运行时覆盖层。
   - `withRuntimeOverrides(store, overrides)` 实现 CLI `--api-key`：读取覆盖项时，对每个被覆盖的 provider 返回临时 `{ type: "api_key", key }`，屏蔽已存储的 OAuth/API 凭据但不持久化。
   - 运行时覆盖必须也适用于支持 OAuth 的 provider；coding-agent 注册的每个 provider 都必须保留或获得 `apiKey` 认证槽位，确保覆盖有效。
   - 测试优先级：运行时覆盖 > 已存储凭据 > models.json 配置认证 > provider 外部环境变量；并验证已存储凭据会阻止回退到外部环境变量。
5. [ ] 为 models.json 构建 provider 装饰辅助函数。
   - 从内置 provider 工厂开始，而不是从生成的模型数组开始。
   - 包装 provider `getModels()`，使每次同步读取都应用 provider 级 `baseUrl`/`headers`/`compat`、逐模型 `modelOverrides` 和自定义模型合并。
   - 保留 `refreshModels()` 透传，使动态 provider 也能与装饰层组合。
   - 将 models.json 中 provider 的 `apiKey`/`headers`/`authHeader` 配置转换为包装后的 `ApiKeyAuth`：优先解析配置值，缺少时回退到基础 provider auth。
   - 带有 `models` 的自定义 provider 使用 `createProvider()`，并搭配适当的延迟 API 包装器或扩展提供的 stream 实现。
   - 解析错误必须保留当前 `ModelRegistry.getError()` 行为：内置 provider 仍可用，同时错误也能被看到。
6. [ ] 为 Copilot 包装 `getModels()` 以应用 baseUrl。
   - GitHub Copilot OAuth 的 `toAuth()` 已会为 stream 请求返回每个凭据对应的 `baseUrl`。
   - 存在 OAuth 凭据时包装 Copilot provider 的 `getModels()`，让扩展/UI 可见的模型元数据也包含已认证账户的 base URL。
   - 保持 Copilot 的 API key/环境 token 行为不变。
   - 添加测试，覆盖登录前、存在 OAuth 凭据后、刷新/baseUrl 变更后和登出后的模型元数据。
7. [ ] 扩展 OAuth 适配器。
   - 将旧的扩展 `OAuthProviderInterface` 配置适配为 pi-ai `OAuthAuth`。
   - `login` 将旧回调/事件映射到 `prompt()`/`notify()`。
   - `refreshToken` 映射到 `refresh`。
   - `getApiKey` 映射到 `toAuth`。
   - 在适用时，`modifyModels` 改为包装 provider 的 `getModels()`，并通过 `toAuth().baseUrl` 设置 base URL。
   - 在阶段 10 之前通过 `/compat` 别名保持现有扩展运行时兼容。
8. [ ] 基于 `MutableModels` 重建 coding-agent 的 `ModelRegistry`。
   - 它拥有一个 `MutableModels` 实例，由经过装饰的内置 provider + models.json 自定义 provider + 扩展 provider 构成。
   - `getAll()`、`find()` 和 `getAvailable()` 仍是同步兼容方法，基于最近已知模型列表和快速的认证配置状态判断。不要破坏扩展使用这些读取接口时依赖的 `modelRegistry` 表面。
   - `refresh()` 是明确的异步新鲜度边界：重建 provider 层，并在需要时调用 `models.refresh()`；新路径不应重置全局 api-registry，compat 过渡期的兼容行为除外。
   - `registerProvider()`/`unregisterProvider()` 会修改 provider 层并重建集合。
   - Facade 的认证操作（`login`、`logout`、provider 状态、可用 OAuth provider）通过 `provider.auth.{apiKey,oauth}` 和 `CredentialStore` 执行；不再保留 `AuthStorage` 类型。
   - 旧式 `registerApiProvider` 写入只供 `/compat` 调用方使用，并在阶段 10 删除。
9. [ ] 重新连接各个消费者。
   - `AgentSession` 的 stream 函数通过 `ModelRegistry`/`Models` 解析，而不是组合 `getApiKeyAndHeaders()` 和 compat 全局 API。
   - SDK 选项将 `authStorage` 替换为 `credentials?: CredentialStore` 或基于 agent-dir 的默认实现；更新 `sdk.md` 和示例。
   - `model-resolver`、`--list-models`、模型选择器、登录/登出/状态 UI 和 provider 归属信息，使用同步的最近已知模型读取；只有显式刷新/认证操作需要等待。
   - CLI 的 `--api-key` 应设置运行时覆盖装饰器，而不是修改 `AuthStorage`。
   - 阶段 10 前保留扩展加载器根入口到 compat 的别名，但要向前提供新的集合/facade API。
10. [ ] 测试迁移并对真实 provider 做验证。
    - 为 `FileCredentialStore`、运行时覆盖层、provider 装饰、扩展 OAuth 适配器、基于 Models 的 ModelRegistry facade 和消费者重连添加单元测试。
    - 为 Cloudflare account/gateway 环境变量、Copilot OAuth baseUrl 包装、运行时 `--api-key` 优先级、`$ENV`/`!command` 解析、已存储凭据阻止回退外部环境变量添加回归测试。
    - 更新现有测试，验证 `ModelRegistry.getAll/find/getAvailable` 同步返回最近已知数据，并通过显式异步刷新更新。
    - 运行定向非 e2e 测试套件，并使用 tmux 对真实 provider 的登录流程进行验证（Anthropic OAuth/API key、OpenAI Codex OAuth、GitHub Copilot OAuth、Cloudflare AI Gateway；若有凭据，也验证 Bedrock）。

### 阶段 10 — 删除 compat（pi 2.0 阶段，单独处理）

- [ ] 将 AgentSession 迁移到 AgentHarness；用 harness 的 `Models` 替代 registry facade。
- [ ] 将所有内部 `/compat` 导入迁移到新 API：包括每个 package 的 src、所有测试和示例扩展（届时示例会展示新 API）。之后仓库内部不得再导入 `/compat`。
- [ ] 删除 `/compat`、`env-api-keys.ts`、扩展加载器中根入口到 compat 的别名、旧的 `pi-ai/oauth` 注册表和 `OAuthProviderInterface`（包括 `usesCallbackServer`），以及 compat 本地的旧 API 注册表。这是会影响扩展作者的破坏性发布，changelog 必须附带迁移指南。

### 延后处理 / 后续工作

- [ ] 将 sitegeist 风格的 Web OAuth 实现作为另一种 `OAuthAuth`。
- [x] 重设计 Images API：`ImagesModels`/`ImagesProvider`/`createImagesProvider` 沿用聊天侧设计（同步读取、显式刷新、生成过程不拒绝）；通过 `auth/resolve.ts` 中独立的 `resolveProviderAuth()` 与聊天侧共享认证解析（该函数也负责 `ModelsError`；两个集合都将自己的 store/context 作为参数传入，不使用 resolver 对象）。在 `providers/all` 中添加 `openrouterImagesProvider()` 工厂以及 `builtinImagesProviders()`/`builtinImagesModels()`；实现移至带延迟包装器的 `api/openrouter-images.ts`。旧的全局图像 API（注册表 + `getImageModel*` + `generateImages`）留在 compat；将 types.ts 中 `ImagesProvider` ID 别名重命名为 `ImagesProviderId`（对应 `Provider` -> `ProviderId`）。

## 错误行为

`undefined` 表示未找到或尚未配置。实际失败会导致 Promise 拒绝，或转化为 stream error。

```ts
export type ModelsErrorCode =
  | "model_source"      // provider model refresh failed
  | "model_validation"  // model object invalid
  | "provider"          // unknown provider, dispatch failure
  | "stream"            // stream setup failure
  | "auth"              // auth resolution failure
  | "oauth";            // oauth login/refresh failure
```

- `Models.stream()` 遇到异步准备失败时会产生 stream error（error 事件 + error 结果）；返回 stream 后不会直接抛异常。
- `Models.getModels()` 是同步的尽力而为读取：某 provider 的 `getModels()` 若抛错，则该 provider 返回空模型列表。`Models.refresh(provider)` 在该 provider 获取失败时拒绝；`Models.refresh()`（所有 provider）则并发尽力刷新。需要明确看到列表获取错误的应用，应单独刷新对应 provider。
- 认证解析和凭据存储失败会明确拒绝（`ModelsError` 错误码 `auth`/`oauth`）；失败后静默回退到另一种认证路径可能导致意外扣费。只要存在已存储凭据，就始终阻止回退到外部环境变量，包括刷新失败之后。
- 状态/可用性 UI 应捕获 `getAuth` 的拒绝并显示“需要重新登录”，不能把拒绝当成“尚未配置”。
