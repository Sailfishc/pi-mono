# 第七章：横切关注点

> [← 第六章](06-coding-agent-package.md) | [← 返回大纲](study-guide.md)

本章总结整个项目中反复出现的设计技巧，这些技巧是项目质量的基石。

---

## 7.1 TypeScript 开放枚举

**模式**：`KnownX | (string & {})`

**出现位置**：`packages/ai/src/types.ts`（Api、Provider 类型）

```typescript
// ❌ 问题写法：IDE 补全消失，类型变成 string
type Api = "openai-completions" | "anthropic-messages" | string;

// ✅ 正确写法：既有补全，又允许扩展
type Api = "openai-completions" | "anthropic-messages" | (string & {});
```

**原理**：TypeScript 看到 `字面量联合 | string` 时会优化为 `string`。但 `string & {}` 是交叉类型，TypeScript 不做这个优化，字面量成员保留，补全照常工作。

**适用场景**：任何需要"已知值集合 + 开放扩展"的类型。

---

## 7.2 声明合并（Declaration Merging）作为扩展机制

**模式**：空接口 + `keyof` 类型计算

**出现位置**：`packages/agent/src/types.ts`（CustomAgentMessages）

```typescript
// 库中定义空接口
export interface CustomAgentMessages {}

// 库的核心类型引用它
export type AgentMessage = Message | CustomAgentMessages[keyof CustomAgentMessages];

// 下游应用扩展它（无需修改库）
declare module "@mariozechner/pi-agent-core" {
  interface CustomAgentMessages {
    notification: { role: "notification"; text: string };
  }
}

// 结果：AgentMessage 自动包含 notification，类型安全
```

**优势**：
- 库的核心逻辑不变，扩展点明确
- TypeScript 类型系统全程保障安全
- 第三方库（如 Extension）可以各自扩展，互不干扰

---

## 7.3 TypeBox：运行时校验 + 编译期类型一体化

**出现位置**：所有工具定义（`AgentTool`、内置工具）

```typescript
import { Type, Static } from "@sinclair/typebox";

// 一份 schema，两种用途
const params = Type.Object({
  path: Type.String({ description: "文件路径" }),
  encoding: Type.Union([Type.Literal("utf-8"), Type.Literal("binary")])
});

// 编译期：自动推断参数类型
type Params = Static<typeof params>;
// → { path: string; encoding: "utf-8" | "binary" }

// 运行时：校验 LLM 传来的参数
validateToolArguments(params, llmProvidedArgs);
// → 抛出带描述的错误，而不是静默失败
```

**为什么不用 zod？**

TypeBox 生成的是标准 JSON Schema，可以直接发给 LLM（作为工具的 `parameters` 字段），不需要额外转换。

---

## 7.4 流式优先（Stream-First）设计哲学

整个项目的 I/O 设计遵循一个原则：**所有可能耗时的操作都返回流，而不是 Promise**。

| 操作 | 返回类型 | 原因 |
|------|---------|------|
| LLM 调用 | `AssistantMessageEventStream` | token 逐个到达，需要流式渲染 |
| Agent 循环 | `EventStream<AgentEvent>` | 工具执行可能很慢，需要实时反馈 |
| bash 工具 | `onUpdate` 回调 | 命令输出需要实时显示 |

**流 vs Promise 的选择标准**：
- 需要**中间进度**反馈 → 流
- 只需要**最终结果** → Promise（或用 `stream.result()` 消费流）

两者都支持的好处：下游消费者可以选择自己需要的粒度。

---

## 7.5 sourceId 热插拔

**出现位置**：`packages/ai/src/api-registry.ts`

```typescript
// 注册时带 sourceId
registerApiProvider(myProvider, "my-extension-v1");

// 批量注销（插件卸载时）
unregisterApiProviders("my-extension-v1");
```

这个模式让系统支持**动态加载/卸载组件**而不重启。Extension 系统基于同样的思路——Extension 加载时注册工具和命令，卸载时一键清除。

---

## 7.6 AbortSignal 贯穿全链路

```
用户按 Esc
    │
    ▼
agent.abort()
    │ 创建 AbortController，signal 向下传递
    ▼
agentLoop(... , signal)
    │
    ├─► streamSimple(model, context, { signal })
    │       └─► HTTP 请求携带 signal → 网络请求取消
    │
    └─► tool.execute(id, params, signal)
            └─► 工具内部检查 signal.aborted
```

`AbortSignal` 从最顶层（用户操作）一路传到最底层（HTTP 请求、子进程），保证**取消操作真正生效**，不是假取消。

---

## 7.7 类型条件约束 Provider 特有选项

**出现位置**：`packages/ai/src/types.ts`（Model 的 `compat` 字段）

```typescript
interface Model<TApi extends Api> {
  compat?: TApi extends "openai-completions"
    ? OpenAICompletionsCompat
    : TApi extends "openai-responses"
      ? OpenAIResponsesCompat
      : never;
}
```

`Model<"anthropic-messages">` 的 `compat` 字段类型是 `never`，访问它编译报错。只有 OpenAI 系的 Model 才有这个字段，且类型精确匹配。

这是用 TypeScript 条件类型实现**按 tag 分发类型**的典型用法。

---

## 7.8 设计原则总结

| 原则 | 体现 |
|------|------|
| **开闭原则** | Provider 注册 / Declaration Merging / Extension 系统：对修改关闭，对扩展开放 |
| **单一职责** | ai 层无状态，agent 层无 UI，tui 层无业务逻辑 |
| **依赖倒置** | 上层依赖接口（`StreamFunction`、`Component`），不依赖具体实现 |
| **流式优先** | 所有耗时操作都是流，UI 可以实时响应 |
| **类型即文档** | TypeBox schema 既是参数文档，也是运行时校验 |
| **热插拔** | sourceId 支持动态注册/注销，不需要重启 |

---

## 学习路径建议

如果你要参考这个项目做自己的 AI Agent：

1. **先学 `packages/ai`**：理解 Provider 注册、EventStream、判别联合类型
2. **再学 `packages/agent`**：理解 Agent Loop、声明合并、convertToLlm 边界
3. **选学 `packages/tui`**：如果你要做 TUI，学差量渲染和 CURSOR_MARKER
4. **最后看 `packages/coding-agent`**：学 Extension/Skill 系统和 Session 设计

最小可用组合：`pi-ai` + `pi-agent-core`，加上自己的工具定义和 UI，就能做出一个 Agent。

---

*[← 返回大纲](study-guide.md)*
