# 第三章：LLM 抽象层 packages/ai

> [← 第二章](02-architecture.md) | [第四章 →](04-agent-package.md)

源码位置：`packages/ai/src/`

---

## 3.1 这一层解决什么问题

每家 LLM 的 API 形状都不同：

- Anthropic：`messages.create()` with `stream: true`
- OpenAI：`chat.completions.create()` with streaming
- Google：`generateContentStream()`
- AWS Bedrock：`converseStream()`

这一层的目标：**调用方只传 `(model, context)` 两个参数，拿到统一格式的事件流**，不感知任何 Provider 细节。

---

## 3.2 层的入口：模块加载即初始化

`stream.ts` 的第一行就揭示了整个 Provider 机制的启动方式：

```typescript
// packages/ai/src/stream.ts:1-2
import "./providers/register-builtins.js";
import "./utils/http-proxy.js";
```

这两行没有导入任何符号，纯粹是为了执行副作用。`register-builtins.ts` 的最后一行是：

```typescript
// packages/ai/src/providers/register-builtins.ts:73
registerBuiltInApiProviders();
```

模块一被加载，9 个内置 Provider 就自动注册完毕。这就是为什么调用 `stream()` 之前不需要任何初始化代码——**模块加载本身就是初始化**。

---

## 3.3 stream.ts 的四个公开函数

`stream.ts` 只有 61 行，但设计极其精炼，提供了四个函数：

| 函数 | 特点 | 适用场景 |
|------|------|---------|
| `stream()` | 传 Provider 原生 options，完整控制 | 需要 Provider 特有功能时 |
| `complete()` | `stream()` + `.result()`，只要最终结果 | 不需要流式过程 |
| `streamSimple()` | 传统一的 `SimpleStreamOptions` | **agent 层唯一用的接口** |
| `completeSimple()` | `streamSimple()` + `.result()` | 简单场景 |

**`stream` vs `streamSimple` 的区别**是这一层最关键的设计决策：

`stream` 允许传 Anthropic 特有的 `AnthropicOptions`（如 `thinkingBudgetTokens`、`effort`），完全控制，但绑定了具体 Provider。

`streamSimple` 接受统一的 `SimpleStreamOptions`，内部由每个 Provider 把通用选项（如 `reasoning: "high"`）翻译成各自的参数格式。agent 层只用这个接口——**agent 不应该知道它在用 Anthropic 还是 OpenAI**。

---

## 3.4 开放枚举技巧：`Api` 类型

```typescript
// packages/ai/src/types.ts:5-16
export type KnownApi =
  | "openai-completions"
  | "anthropic-messages"
  | "google-generative-ai"
  // ... 8 种已知 API

export type Api = KnownApi | (string & {});
```

**为什么不直接写 `string`？**

`KnownApi | string` 在 TypeScript 中会被优化为 `string`，IDE 自动补全消失。

`(string & {})` 与 `string` 语义等价，但 TypeScript **不会把它和联合类型的字面量成员合并**，所以：
- 输入 `"anthropic-m"` → IDE 补全 `"anthropic-messages"` ✓
- 输入自定义字符串 → 合法，不报错 ✓

同样的技巧用于 `Provider = KnownProvider | (string & {})`。

---

## 3.5 Provider 注册中心

```typescript
// packages/ai/src/api-registry.ts:40
const apiProviderRegistry = new Map<string, RegisteredApiProvider>();

export function registerApiProvider<TApi extends Api>(
  provider: ApiProvider<TApi, TOptions>,
  sourceId?: string,   // 用于批量注销
): void {
  apiProviderRegistry.set(provider.api, {
    provider: {
      api: provider.api,
      stream: wrapStream(provider.api, provider.stream),
      streamSimple: wrapStreamSimple(provider.api, provider.streamSimple),
    },
    sourceId,
  });
}
```

**注册即可用**，调用方不需要知道 Provider 的存在：

```typescript
// packages/ai/src/stream.ts:26-33
export function stream(model, context, options?) {
  const provider = resolveApiProvider(model.api);  // 根据 model.api 找 Provider
  return provider.stream(model, context, options);
}
```

每个 Provider 注册时只提供三个字段：`api`（路由键）、`stream`、`streamSimple`。注册中心不关心 Provider 内部实现，只要实现了这两个函数就能接入。

**`sourceId` 的作用（热插拔）**：

```typescript
// 注册一批自定义 Provider（如插件加载时）
registerApiProvider(myProvider, "my-plugin");

// 插件卸载时，一键注销所有与该 sourceId 关联的 Provider
unregisterApiProviders("my-plugin");
```

**`wrapStream` 的类型安全保障**：

```typescript
// api-registry.ts:42-52
function wrapStream(api, stream) {
  return (model, context, options) => {
    if (model.api !== api) {
      throw new Error(`Mismatched api: ${model.api} expected ${api}`);
    }
    return stream(model, context, options);
  };
}
```

注册时包一层检查，防止 Provider 被错误路由（运行时保障，编译期之外的第二道防线）。

---

## 3.6 EventStream：手写异步迭代器

这是整个项目技术含量最高的基础设施类，逐段理解：

### 结构与"延迟 resolve"技巧

```typescript
// packages/ai/src/utils/event-stream.ts:4-18
export class EventStream<T, R = T> implements AsyncIterable<T> {
  private queue: T[] = [];
  private waiting: ((value: IteratorResult<T>) => void)[] = [];
  private done = false;
  private finalResultPromise: Promise<R>;
  private resolveFinalResult!: (result: R) => void;

  constructor(
    private isComplete: (event: T) => boolean,   // 判断"最后一个事件"的条件
    private extractResult: (event: T) => R,       // 从最后一个事件提取最终结果
  ) {
    this.finalResultPromise = new Promise((resolve) => {
      this.resolveFinalResult = resolve;   // 把 resolve 存起来，后续才调用
    });
  }
```

构造时把 Promise 的 `resolve` 存到实例上，等到终止事件到来时才触发。这是**延迟 resolve**技巧——Promise 在构造时创建，但 resolve 时机由外部控制。

### push 方法：生产者调用

```typescript
// event-stream.ts:20-35
push(event: T): void {
  if (this.done) return;                               // 已结束就忽略

  if (this.isComplete(event)) {
    this.done = true;
    this.resolveFinalResult(this.extractResult(event)); // 触发最终结果
  }

  const waiter = this.waiting.shift();
  if (waiter) {
    waiter({ value: event, done: false });  // 有消费者在等，直接交付
  } else {
    this.queue.push(event);                 // 没人等，入队缓存
  }
}
```

两种情况：
- **消费者比生产者慢**：事件入 `queue`，等消费者来取
- **消费者比生产者快**（在 `await` 挂起中）：`waiting` 里有 resolve 函数，直接唤醒

### asyncIterator：消费者调用

```typescript
// event-stream.ts:49-61
async *[Symbol.asyncIterator](): AsyncIterator<T> {
  while (true) {
    if (this.queue.length > 0) {
      yield this.queue.shift()!;         // 有缓存事件，直接取
    } else if (this.done) {
      return;                             // 已结束，退出
    } else {
      // 队列空且未结束：挂起，等生产者 push
      const result = await new Promise<IteratorResult<T>>(
        (resolve) => this.waiting.push(resolve)
      );
      if (result.done) return;
      yield result.value;
    }
  }
}
```

三种状态的处理：有货就取、结束就停、没货就等。逻辑清晰，三种情况互不干扰。

### 两种消费方式（同一个对象）

```typescript
// 方式一：关心过程，逐事件处理
for await (const event of stream) {
  if (event.type === "text_delta") renderText(event.delta);
  if (event.type === "toolcall_end") executeTool(event.toolCall);
}

// 方式二：只要最终结果
const message = await stream.result();
```

生产者（LLM Provider）`push` 事件，消费者（Agent）`await` 迭代——两者速率不同也不会互相阻塞。

### AssistantMessageEventStream：具体化泛型

```typescript
// event-stream.ts:68-82
export class AssistantMessageEventStream
    extends EventStream<AssistantMessageEvent, AssistantMessage>
{
  constructor() {
    super(
      (event) => event.type === "done" || event.type === "error",  // 终止条件
      (event) => {
        if (event.type === "done") return event.message;
        if (event.type === "error") return event.error;
        throw new Error("Unexpected event type");
      },
    );
  }
}
```

把泛型参数具体化：事件类型是 `AssistantMessageEvent`，最终结果类型是 `AssistantMessage`。终止条件是遇到 `done` 或 `error`。

---

## 3.7 AssistantMessageEvent：判别联合类型状态机

```typescript
// packages/ai/src/types.ts:208-220
export type AssistantMessageEvent =
  | { type: "start";          partial: AssistantMessage }
  | { type: "text_start";     contentIndex: number; partial: AssistantMessage }
  | { type: "text_delta";     contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: "text_end";       contentIndex: number; content: string; partial: AssistantMessage }
  | { type: "thinking_start"; contentIndex: number; partial: AssistantMessage }
  | { type: "thinking_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: "thinking_end";   contentIndex: number; content: string; partial: AssistantMessage }
  | { type: "toolcall_start"; contentIndex: number; partial: AssistantMessage }
  | { type: "toolcall_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: "toolcall_end";   contentIndex: number; toolCall: ToolCall; partial: AssistantMessage }
  | { type: "done";  reason: "stop"|"length"|"toolUse"; message: AssistantMessage }
  | { type: "error"; reason: "aborted"|"error";         error: AssistantMessage };
```

**三个设计亮点**：

1. **每个事件都带 `partial`**：即"到目前为止构建好的完整消息"。消费者**不需要自己维护拼接状态**，直接用 `partial` 就能渲染当前进度。

2. **`_start` / `_delta` / `_end` 三段式**：对流式内容（文本、思考、工具调用）提供生命周期回调，消费者可以选择只处理 `_delta` 做增量渲染。

3. **终止事件区分成功和失败**：`done` vs `error`，各自的 `reason` 字段被约束为不同的可选值，不会混淆。

---

## 3.8 Provider 内部结构：以 Anthropic 为例

```typescript
// providers/anthropic.ts:193-198
export const streamAnthropic: StreamFunction<"anthropic-messages", AnthropicOptions> = (
    model: Model<"anthropic-messages">,
    context: Context,
    options?: AnthropicOptions,
): AssistantMessageEventStream => {
    const stream = new AssistantMessageEventStream();
    // ... 在异步任务里调用 Anthropic SDK，把响应翻译成 push() 调用
    return stream;   // 立即返回，不等待异步完成
};
```

所有 Provider 的结构固定：
1. 创建一个 `AssistantMessageEventStream`
2. 启动异步任务调用各家 SDK
3. 把 SDK 的原生事件翻译成统一的 `push(text_delta/toolcall_end/done...)` 调用
4. **立即返回** stream（不等待异步完成）

**立即返回**是关键——调用方可以立刻开始 `for await` 迭代，同时 Provider 在后台异步推事件。

---

## 3.9 彩蛋：Claude Code 隐身模式

```typescript
// providers/anthropic.ts:64-93
// Stealth mode: Mimic Claude Code's tool naming exactly
const claudeCodeVersion = "2.1.2";

const claudeCodeTools = [
    "Read", "Write", "Edit", "Bash", "Grep", "Glob",
    "AskUserQuestion", "EnterPlanMode", "ExitPlanMode",
    "KillShell", "NotebookEdit", "Skill", "Task",
    "TaskOutput", "TodoWrite", "WebFetch", "WebSearch",
];

const ccToolLookup = new Map(claudeCodeTools.map((t) => [t.toLowerCase(), t]));

// 工具名转为 Claude Code 的标准大小写
const toClaudeCodeName = (name: string) =>
    ccToolLookup.get(name.toLowerCase()) ?? name;
```

发往 Anthropic API 的工具名被伪装成 Claude Code 的命名风格（首字母大写）。

**为什么？** Anthropic 对 Claude Code 有特殊的服务器端 Prompt Caching 优化。当工具名与 Claude Code 一致时，可以命中这个缓存，**减少 token 消耗、降低延迟**。代码里还附了溯源链接（`cchistory.mariozechner.at`）用于追踪 Claude Code 工具命名的版本变化。这是一个真实生产环境中的成本优化技巧。

---

## 3.10 架构小结

```
stream(model, context)
      │
      ▼ stream.ts:1 → import register-builtins（模块加载即注册）
      │
      ▼ resolveApiProvider(model.api)   查注册表
      │
      ▼ provider.stream(model, context) 委托给具体实现
      │
      ├─ anthropic.ts  → Anthropic SDK → push(text_delta / toolcall_end / done...)
      ├─ openai-completions.ts → OpenAI SDK → push(...)
      └─ google.ts     → Google SDK → push(...)
      │
      ▼ AssistantMessageEventStream     统一出口
            │
            ├─ for await (event of stream) {...}   流式消费
            └─ await stream.result()               直接拿结果
```

**设计原则总结**：
- 模块加载即初始化，调用方零配置
- 注册表解耦调用方与 Provider 实现
- `stream` vs `streamSimple`：充分控制 vs 跨 Provider 通用
- EventStream 手写异步迭代器，同时支持流式和 Promise 两种消费模式
- 每个事件携带 `partial` 免去消费者维护拼接状态
- Provider 立即返回 stream，异步推事件，不阻塞调用方

---

*下一步：[第四章 Agent 运行时 packages/agent →](04-agent-package.md)*
