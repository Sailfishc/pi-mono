# 第二章：分层架构

> [← 第一章](study-guide.md) | [第三章 →](03-ai-package.md)

---

## 2.1 依赖关系图

```
┌─────────────────────────────────────────────────────┐
│                  coding-agent (产品层)               │
│         CLI / Session / Extension / Skills          │
└───────┬───────────────────────────┬─────────────────┘
        │ depends on                │ depends on
        ▼                           ▼
┌───────────────┐         ┌─────────────────────┐
│  agent (运行时) │         │    tui (终端 UI)     │
│  Agent Loop   │         │  差量渲染 / 组件系统  │
│  Tool Calling │         └─────────────────────┘
│  Event Stream │
└───────┬───────┘
        │ depends on
        ▼
┌───────────────────────────────────────┐
│           ai (LLM 抽象层)              │
│  Provider 注册 / 流式 API / 模型元数据  │
└───────────────────────────────────────┘
```

**规则**：依赖单向向下，上层不被下层感知。`tui` 与 `agent` 互不依赖，都只被 `coding-agent` 组合。

这个规则不靠自律维持，而是由 `package.json` 锁死：

```
ai 的依赖：        Anthropic SDK, OpenAI SDK, Google SDK...（外部 LLM 库）
agent 的依赖：     只有 @mariozechner/pi-ai
tui 的依赖：       chalk, marked...（UI 工具，无 LLM 相关）
coding-agent 的依赖：pi-agent-core + pi-ai + pi-tui（三层全部组合）
```

`tui` 完全没有出现在 `agent` 的依赖里——这是物理隔离，不是约定。

---

## 2.2 每层的职责边界

### ai 层（纯函数，无状态）

**输入**：Model + Context + Options
**输出**：`AssistantMessageEventStream`（异步事件流）

```typescript
// 核心签名
stream(model, context, options?) → AssistantMessageEventStream
streamSimple(model, context, options?) → AssistantMessageEventStream
```

这一层**不维护任何状态**，不知道 "Agent" 是什么，只管把一次 LLM 调用转化为事件流。

---

### agent 层（有状态，无 UI）

**输入**：prompt 文本 / 工具定义 / 配置
**输出**：`EventStream<AgentEvent>`（Agent 生命周期事件）

```typescript
// 核心行为
agent.prompt("帮我重构这个函数")  → 开始 Agent 循环
agent.steer("先别改，先加测试")   → 运行时干预
agent.subscribe(event => ...)     → 监听所有事件
```

这一层**维护对话历史**（messages），**执行工具调用**，**管理 abort**，但**不渲染任何 UI**。

---

### tui 层（纯展示，无业务逻辑）

**输入**：`Component` 树 + 用户键盘输入
**输出**：终端 ANSI escape 序列

```typescript
// 核心行为
tui.render([header, messageList, inputBox])  // 渲染组件树
component.handleInput(data)                  // 处理键盘事件
```

这一层**不知道 LLM 或 Agent**，只负责把 `string[]` 高效地输出到终端。

---

### coding-agent 层（组合 + 业务逻辑）

把 `agent` 事件绑定到 `tui` 组件，处理 Session 持久化、Extension 加载、Skill 调用等。

---

## 2.2.1 用三个问题区分各层

快速判断一层的"个性"，问三个问题：

| | ai 层 | agent 层 | tui 层 | coding-agent 层 |
|--|--|--|--|--|
| **有状态？** | 无 | 有（消息历史、工具列表） | 无 | 有（Session、Extension） |
| **知道 UI？** | 不知道 | 不知道 | 只做 UI | 知道（胶水层） |
| **知道 Provider？** | 知道（路由给各家 API） | 不知道 | 不知道 | 不知道 |

---

## 2.3 数据流：一次完整的用户请求

**关键认知**：数据流和事件流的方向是**反的**——数据向下流，通知向上冒。这是事件驱动架构的标准结构。

```
用户按下 Enter
      │
      ▼
[tui] Editor.handleInput → 触发 onSubmit 回调
      │
      ▼
[coding-agent] agent.prompt(text)
      │
      ▼
[agent] agentLoop 启动
  ├─ 推送 agent_start / turn_start 事件
  ├─ transformContext() → 裁剪消息历史
  ├─ convertToLlm() → 转换为 LLM 格式的 Message[]
  ├─ streamSimple(model, context) ← 调用 ai 层
  │     │
  │     └─ [ai] Provider 路由 → Anthropic/OpenAI/... API
  │           └─ 返回 AssistantMessageEventStream
  │
  ├─ 监听流事件，推送 message_update 事件
  │
  ├─ 遇到 tool_call → execute 工具 → 推送 tool_execution_* 事件
  │     └─ 检查 steering → 可能中断
  │
  └─ 推送 turn_end / agent_end 事件
      │
      ▼
[coding-agent] 监听 AgentEvent
  └─ message_update → 更新 TUI 中的 Markdown 组件
  └─ tool_execution_start → 在 TUI 显示工具调用 spinner
  └─ agent_end → 保存 Session 到 JSONL 文件
```

**数据向下流**（用户输入 → agent → ai → LLM API）
**事件向上冒**（LLM 响应 → ai EventStream → agent AgentEvent → coding-agent → TUI 更新）

Agent 不主动调用 UI，而是发事件，UI 自己订阅。这是 `agent.ts:202` 里 `subscribe` 的设计意图：

```typescript
// packages/agent/src/agent.ts:202
subscribe(fn: (e: AgentEvent) => void): () => void {
    this.listeners.add(fn);
    return () => this.listeners.delete(fn);  // 返回取消订阅函数，支持多个监听者
}
```

---

## 2.4 三个关键接口

理解这三个接口，就理解了各层之间的契约。

### `Context`（ai 层边界）

```typescript
// packages/ai/src/types.ts
interface Context {
  systemPrompt?: string;
  messages: Message[];  // UserMessage | AssistantMessage | ToolResultMessage
  tools?: Tool[];
}
```

这是**发给 LLM 的全部信息**，完全 JSON 可序列化，跨 Provider 通用。

---

### `AgentMessage`（agent 层内部）

```typescript
// packages/agent/src/types.ts
type AgentMessage =
  | Message                                          // LLM 标准消息
  | CustomAgentMessages[keyof CustomAgentMessages];  // 应用自定义消息
```

比 `Message` 更丰富，可包含 UI 专用消息（notification、artifact 等），在进入 `ai` 层前由 `convertToLlm` 过滤。

---

### `AgentEvent`（agent → coding-agent 的通知）

```typescript
type AgentEvent =
  | { type: "agent_start" }
  | { type: "agent_end"; messages: AgentMessage[] }
  | { type: "turn_start" }
  | { type: "turn_end"; message: AgentMessage; toolResults: ToolResultMessage[] }
  | { type: "message_start"; message: AgentMessage }
  | { type: "message_update"; message: AgentMessage; assistantMessageEvent: AssistantMessageEvent }
  | { type: "message_end"; message: AgentMessage }
  | { type: "tool_execution_start"; toolCallId: string; toolName: string; args: any }
  | { type: "tool_execution_update"; toolCallId: string; toolName: string; args: any; partialResult: any }
  | { type: "tool_execution_end"; toolCallId: string; toolName: string; result: any; isError: boolean };
```

这些事件是 UI 的**数据源**，coding-agent 把它们映射到对应的 TUI 组件更新。

`AgentEvent` 设计了三层粒度的事件，UI 可以只订阅自己关心的那一层：

| 事件 | 粒度 | 典型 UI 行为 |
|------|------|------------|
| `agent_start` / `agent_end` | Agent 级 | 禁用/启用输入框 |
| `turn_start` / `turn_end` | Turn 级 | 显示/隐藏轮次分隔线 |
| `message_update` | Token 级 | 流式更新 Markdown 组件 |
| `tool_execution_start` / `end` | 工具级 | 显示/收起工具调用 spinner |

---

## 2.5 架构的核心取舍

| 决策 | 代价 | 收益 |
|------|------|------|
| agent 层不依赖 tui | coding-agent 要写绑定胶水代码 | agent 可以在非 TUI 环境使用（RPC 模式、测试） |
| ai 层无状态 | 调用方（agent）自己管历史 | ai 层可以被单独测试、任意替换 |
| AgentMessage 可扩展 | 需要 convertToLlm 过滤 | 应用层不受限于 LLM 的消息格式 |
| 事件驱动而非回调 | 需要理解事件流 | UI 与 Agent 完全解耦，可随意添加多个监听者 |

### ai 层无状态的深层意义

对比两种设计，理解为什么无状态更好：

```typescript
// ❌ 有状态的设计
const client = new LLMClient();
client.addMessage("你好");
client.send();           // client 内部记着历史
client.addMessage("再说一遍");
client.send();

// ✅ 无状态的设计（pi-ai 的做法）
const messages = [];
messages.push({ role: "user", content: "你好" });
const reply = await stream(model, { messages }).result();
messages.push(reply);
messages.push({ role: "user", content: "再说一遍" });
stream(model, { messages });   // 每次传完整历史
```

`Context` 是普通数据，因此可以：序列化存磁盘、在进程间传递、用于回放测试、在不同 Provider 间切换（换 Provider 时只换 `model`，`context` 不变）。

### streamFn 注入点与可测试性

`agent.ts:132` 里 `streamFn` 默认是 `streamSimple`，但可以在构造时替换：

```typescript
// packages/agent/src/agent.ts:132
this.streamFn = opts.streamFn || streamSimple;
```

这是 agent 层与 ai 层的**唯一连接点**——一个函数。测试时注入假的 `streamFn`，不需要真实 API Key，整个 Agent 循环逻辑都能测试。

---

*下一步：[第三章 LLM 抽象层 packages/ai →](03-ai-package.md)*
