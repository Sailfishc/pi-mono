# 第四章：Agent 运行时 packages/agent

> [← 第三章](03-ai-package.md) | [第五章 →](05-tui-package.md)

源码位置：`packages/agent/src/`

---

## 4.1 这一层解决什么问题

AI Agent 不是"发一条消息，收一条回复"。它是：

1. 维护**完整对话历史**
2. 在 LLM 回复中识别**工具调用**，执行工具，把结果喂回 LLM
3. **循环**直到 LLM 决定停止
4. 让外部能**观测**整个过程（UI 更新、日志等）
5. 支持**运行时干预**（用户打断、注入消息）

---

## 4.2 核心文件只有三个

```
packages/agent/src/
├── types.ts        类型定义（AgentMessage、AgentEvent、AgentTool...）
├── agent-loop.ts   Loop 的纯函数实现（无状态）
└── agent.ts        有状态的 Agent 类（包装 agent-loop）
```

**关键关系**：`agent-loop.ts` 是核心算法，`agent.ts` 是它的有状态包装器。先理解 loop，再看 Agent 类。

---

## 4.3 agentLoop 的启动方式：IIFE 模式

```typescript
// agent-loop.ts:28-54
export function agentLoop(
    prompts: AgentMessage[],
    context: AgentContext,
    config: AgentLoopConfig,
    signal?: AbortSignal,
    streamFn?: StreamFn,
): EventStream<AgentEvent, AgentMessage[]> {
    const stream = createAgentStream();

    (async () => {           // ← IIFE：立即执行的异步函数
        // ... 所有工作在这里异步跑
        await runLoop(...);
    })();                    // ← 启动，但不等它完成

    return stream;           // ← 同步返回 stream
}
```

和第三章 Provider 的做法完全一样：**先建好接收管道，立即返回，异步填充**。

调用方拿到 `stream` 就可以开始 `for await`，loop 在后台自己跑，通过 `stream.push()` 把事件交付出来。这是整个框架的统一风格。

`createAgentStream` 也值得看：

```typescript
// agent-loop.ts:94-99
function createAgentStream(): EventStream<AgentEvent, AgentMessage[]> {
    return new EventStream<AgentEvent, AgentMessage[]>(
        (event) => event.type === "agent_end",                              // 终止条件
        (event) => event.type === "agent_end" ? event.messages : [],        // 最终结果
    );
}
```

Agent 流的最终结果是 `AgentMessage[]`——本次 agent 运行新产生的所有消息。

---

## 4.4 双层 Loop 结构

`runLoop` 是整个章节最值得细读的函数，结构是两个嵌套的 `while` 循环：

```typescript
// agent-loop.ts:117-194（简化）
while (true) {                                                    // 外层：follow-up 循环
    while (hasMoreToolCalls || pendingMessages.length > 0) {      // 内层：工具调用循环

        // 1. 注入 pending 消息（steering 消息在此注入）
        // 2. 调用 LLM，拿到 assistant 消息
        // 3. 执行所有工具调用，每个完成后检查 steering
        // 4. turn_end 事件

    }

    // 内层退出 = 没有工具调用且没有 steering
    const followUpMessages = await config.getFollowUpMessages?.() || [];
    if (followUpMessages.length > 0) {
        pendingMessages = followUpMessages;
        continue;    // 带着 follow-up 消息重新进入外层
    }
    break;           // 真正结束
}
```

两个循环负责不同的"继续"条件：

| 循环 | 继续条件 | 终止条件 |
|------|---------|---------|
| 内层 | 有工具调用 or 有 steering 消息 | 两者都没有 |
| 外层 | 内层结束后有 follow-up 消息 | 没有 follow-up |

这个结构让两种干预机制在不破坏循环语义的前提下自然嵌入。

---

## 4.5 streamAssistantResponse：转换管线的实际执行点

```typescript
// agent-loop.ts:204-288（核心段）
async function streamAssistantResponse(context, config, signal, stream, streamFn) {

    // Step 1：transformContext（可选，AgentMessage[] → AgentMessage[]）
    let messages = context.messages;
    if (config.transformContext) {
        messages = await config.transformContext(messages, signal);
    }

    // Step 2：convertToLlm（必须，AgentMessage[] → Message[]）
    const llmMessages = await config.convertToLlm(messages);

    // Step 3：构建 LLM Context（只含 LLM 能理解的内容）
    const llmContext: Context = {
        systemPrompt: context.systemPrompt,
        messages: llmMessages,
        tools: context.tools,
    };

    // Step 4：动态解析 API Key（用于 OAuth 短期 token）
    const resolvedApiKey = config.getApiKey
        ? await config.getApiKey(config.model.provider)
        : config.apiKey;

    // Step 5：调用 ai 层
    const response = await streamFunction(config.model, llmContext, { ...config, apiKey: resolvedApiKey, signal });

    // Step 6：把 ai 层事件翻译成 AgentEvent
    for await (const event of response) {
        switch (event.type) {
            case "start":
                context.messages.push(event.partial);              // 立刻推入历史
                stream.push({ type: "message_start", message: event.partial });
                break;
            case "text_delta": case "toolcall_end": // ... 其他流式事件
                context.messages[context.messages.length - 1] = event.partial;  // 原地更新！
                stream.push({ type: "message_update", assistantMessageEvent: event, message: event.partial });
                break;
            case "done": case "error":
                stream.push({ type: "message_end", message: finalMessage });
                return finalMessage;
        }
    }
}
```

**关键细节：partial 消息的原地更新**

`start` 事件时 partial 消息就被推入 `context.messages`，后续每个 delta 事件都用 `event.partial` 原地替换数组最后一项。`context.messages` 里始终有一份"当前进度"的 assistant 消息，不需要等流结束才入库。

**`getApiKey` 的设计意图**

GitHub Copilot 等 OAuth token 有效期只有几分钟。如果 Agent 在执行工具时 token 过期，下一次 LLM 调用会失败。`getApiKey` 是 per-call 动态解析函数，每次 LLM 调用前都刷新 token，规避过期问题。

---

## 4.6 executeToolCalls：顺序执行 + 逐个检查 steering

```typescript
// agent-loop.ts:305-374（简化）
for (let index = 0; index < toolCalls.length; index++) {
    // 执行单个工具，传入 onUpdate 回调（用于流式输出）
    result = await tool.execute(toolCall.id, validatedArgs, signal, (partialResult) => {
        stream.push({ type: "tool_execution_update", ..., partialResult });
    });

    // ← 每个工具执行完就检查一次 steering
    const steering = await getSteeringMessages?.();
    if (steering.length > 0) {
        steeringMessages = steering;
        // 剩余的工具全部 skip
        for (const skipped of toolCalls.slice(index + 1)) {
            skipToolCall(skipped, stream);
        }
        break;
    }
}
```

**工具是顺序执行的**（`for` 循环，不是 `Promise.all`）。这是有意设计——串行执行让 steering 检查可以在每个工具完成后立即生效，而不是等所有工具都完成。

**`skipToolCall` 的妙处**

LLM 的对话结构必须完整——每个 `toolCall` 都需要对应的 `toolResult`，否则 API 报错。steering 打断后，剩余工具用 `skipToolCall` 补一个 `isError: true`、内容为 `"Skipped due to queued user message."` 的假结果，满足协议要求的同时让用户知道这些工具被跳过了。

---

## 4.7 消息系统的可扩展性：声明合并

### 问题

LLM 只理解三种消息：`user`、`assistant`、`toolResult`。

但应用还需要存储 UI 专用消息：通知、artifact、compaction 摘要……这些消息不能发给 LLM，但要和对话历史一起管理。

### 解法：空接口 + Declaration Merging

```typescript
// packages/agent/src/types.ts:120-129
export interface CustomAgentMessages {
    // 空接口，用于第三方扩展
}

export type AgentMessage = Message | CustomAgentMessages[keyof CustomAgentMessages];
```

**`keyof CustomAgentMessages` 的工作原理**：

初始时 `CustomAgentMessages` 是空接口，`keyof` 结果是 `never`，所以 `AgentMessage = Message | never = Message`。

下游扩展后：

```typescript
declare module "@mariozechner/pi-agent-core" {
    interface CustomAgentMessages {
        notification: { role: "notification"; text: string; timestamp: number };
        artifact:     { role: "artifact"; content: string; mimeType: string };
    }
}
// keyof CustomAgentMessages = "notification" | "artifact"
// AgentMessage = Message | { role: "notification"; ... } | { role: "artifact"; ... }
```

TypeScript 模块扩展在编译期合并，运行时零开销，对库代码完全透明。

### 过滤边界：convertToLlm

```typescript
// packages/agent/src/agent.ts:31
function defaultConvertToLlm(messages: AgentMessage[]): Message[] {
    return messages.filter(m =>
        m.role === "user" || m.role === "assistant" || m.role === "toolResult"
    );
}
```

这是**应用层消息到 LLM 消息的转换边界**，每次 LLM 调用前执行。自定义实现可以做更复杂的事：把 notification 转成 user 消息、压缩历史、注入外部上下文。

---

## 4.8 AgentLoopConfig 继承 SimpleStreamOptions

```typescript
// types.ts:22
export interface AgentLoopConfig extends SimpleStreamOptions {
    model: Model<any>;
    convertToLlm: ...;
    transformContext?: ...;
    getApiKey?: ...;
    getSteeringMessages?: ...;
    getFollowUpMessages?: ...;
}
```

`AgentLoopConfig` 继承了 `SimpleStreamOptions`，所以 `temperature`、`maxTokens`、`reasoning`、`signal` 等 LLM 选项可以直接写在 config 里。这些字段在 `streamAssistantResponse` 里通过 `{ ...config, apiKey, signal }` 展开传给 `streamSimple`，调用方只需要一个配置对象。

---

## 4.9 Agent 类：Loop 的有状态包装

`Agent` 类做了三件事：

**1. 维护全局状态**

```typescript
// agent.ts:97-107
private _state: AgentState = {
    systemPrompt: "",
    model: getModel("google", "gemini-2.5-flash-lite-preview-06-17"),
    thinkingLevel: "off",
    tools: [],
    messages: [],            // ← 全局对话历史，跨多次 prompt 调用持久
    isStreaming: false,
    streamMessage: null,
    pendingToolCalls: new Set<string>(),
};
```

**2. 管理两个消息队列**

```typescript
// agent.ts:252-262
steer(m: AgentMessage) {
    this.steeringQueue.push(m);    // 打断：在下一个工具完成后注入
}

followUp(m: AgentMessage) {
    this.followUpQueue.push(m);    // 追加：在 agent 完全停止后注入
}
```

队列出列支持两种模式：
- `"one-at-a-time"`：每轮只消费一条（避免 LLM 被多条消息淹没，默认）
- `"all"`：一次性消费所有队列消息

**3. 对外暴露 subscribe**

```typescript
// agent.ts:202-205
subscribe(fn: (e: AgentEvent) => void): () => void {
    this.listeners.add(fn);
    return () => this.listeners.delete(fn);  // 返回取消函数
}
```

返回取消函数是 React `useEffect` 风格，让资源管理显式且安全，调用方决定何时取消订阅。

---

## 4.10 Steering vs Follow-up：两种干预机制

```typescript
// types.ts:86-97
getSteeringMessages?: () => Promise<AgentMessage[]>;
getFollowUpMessages?: () => Promise<AgentMessage[]>;
```

| | Steering（转向） | Follow-up（追加） |
|---|---|---|
| **触发时机** | 每个工具执行完后检查 | Agent 完全停止后检查 |
| **行为** | 中断剩余工具，注入消息，继续下一轮 LLM | 追加消息，继续新 Turn |
| **适用场景** | 用户说"等一下，先做这个" | 用户提前排好的下一条指令 |

---

## 4.11 Context 转换管线

```
AgentMessage[]                    （应用层消息，包含自定义类型）
    │
    ▼ transformContext()           （可选：裁剪历史、注入外部上下文）
    │
AgentMessage[]                    （变换后，仍是应用层消息）
    │
    ▼ convertToLlm()               （必须：过滤 + 转换为 LLM 格式）
    │
Message[]                         （LLM 标准消息）
    │
    ▼ streamSimple(model, context) （发给 ai 层）
```

两步分离的原因：
- `transformContext` 在 AgentMessage 层操作，能看到自定义 role，便于智能裁剪
- `convertToLlm` 在边界处操作，只负责格式转换，不含业务逻辑

---

## 4.12 AgentTool：工具的完整接口

```typescript
// types.ts:157-166
export interface AgentTool<TParameters extends TSchema, TDetails = any> extends Tool<TParameters> {
    label: string;  // UI 显示用的人类可读名称
    execute: (
        toolCallId: string,
        params: Static<TParameters>,         // 参数类型由 TypeBox schema 推断，编译期安全
        signal?: AbortSignal,
        onUpdate?: AgentToolUpdateCallback,  // 流式进度回调 → tool_execution_update 事件
    ) => Promise<AgentToolResult<TDetails>>;
}
```

`onUpdate` 允许工具在执行过程中推送中间结果，例如 bash 工具实时推送命令输出：

```typescript
execute: async (id, params, signal, onUpdate) => {
    const proc = spawn("bash", ["-c", params.command]);
    proc.stdout.on("data", data => {
        onUpdate?.({ content: [{ type: "text", text: data.toString() }], details: {} });
    });
    await proc.done;
    return { content: [{ type: "text", text: "完成" }], details: {} };
}
```

---

## 4.13 架构小结

Agent Loop 是一个**有限状态机**：

```
[等待 prompt]
    │ prompt()
    ▼
[流式接收 LLM 回复]  →  message_update 事件（token 级）
    │ LLM 有工具调用
    ▼
[顺序执行工具]  →  tool_execution_* 事件
    │ 每个工具后检查 steering
    │   有 → 跳过剩余工具（skipToolCall 补假结果），注入 steering 消息
    │   无 → 继续下一个工具
    │ 所有工具完成 → 回到 [流式接收 LLM 回复]
    │ LLM 无工具调用
    ▼
[检查 follow-up]
    │ 有 → 注入消息，回到 [流式接收 LLM 回复]
    │ 无 → agent_end
```

三个核心设计保证可靠性：
1. **IIFE + 立即返回**：和 ai 层一样，调用方拿到流就能迭代，异步在后台跑
2. **双层 while**：外层管 follow-up 续跑，内层管工具调用和 steering 打断
3. **skipToolCall 补假结果**：保证 LLM 对话协议的结构完整性

---

*下一步：[第五章 终端 UI packages/tui →](05-tui-package.md)*
