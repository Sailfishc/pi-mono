# Pi-Mono 项目学习指南

> 自顶向下拆解一个生产级 AI Agent 框架的设计

---

## 学习大纲

```
第一章  全局视角 —— 项目是什么？解决什么问题？          ← 当前位置
第二章  分层架构 —— 四个包如何协作？依赖关系如何？
第三章  LLM 抽象层（packages/ai）—— 如何统一 10+ 个 LLM Provider？
第四章  Agent 运行时（packages/agent）—— 如何构建可观测的 Agent 循环？
第五章  终端 UI（packages/tui）—— 如何写一个高性能的 TUI 库？
第六章  Coding Agent CLI（packages/coding-agent）—— 如何把以上三层组合成产品？
第七章  横切关注点 —— 类型系统技巧、流式设计、扩展机制
```

---

## 第一章：全局视角

### 1.1 项目是什么

Pi-Mono 是一个 **AI Coding Agent 平台**，主产品是一个交互式命令行工具（类似 Claude Code / Cursor）。

用一句话概括：**在终端里与 AI 结对编程，Agent 可以读写文件、执行命令、调用工具。**

```
用户 ──► [终端 UI] ──► [Agent 运行时] ──► [LLM 抽象层] ──► 各家 LLM API
                              │
                              ▼
                         [内置工具]
                     read / write / edit
                     bash / grep / find
```

### 1.2 项目解决什么问题

| 问题 | 解法 | 对应包 |
|------|------|--------|
| 每个 LLM 的 API 形状不同 | 统一抽象层 + Provider 注册 | `packages/ai` |
| Agent 循环难以被 UI 观测 | 细粒度事件流 + Pub/Sub | `packages/agent` |
| 终端 UI 闪烁、性能差 | 差量渲染 + 同步输出 | `packages/tui` |
| Coding Agent 功能难以扩展 | Extension + Skill + Plugin 系统 | `packages/coding-agent` |

### 1.3 Monorepo 结构

```
pi-mono/
├── packages/
│   ├── ai/            基础层：LLM 多 Provider 统一 API
│   ├── agent/         运行时：Agent 循环、工具调用、状态管理
│   ├── tui/           UI 层：差量渲染终端 UI 库
│   ├── coding-agent/  产品层：CLI Coding Agent
│   ├── web-ui/        Web 组件：AI 对话界面
│   ├── mom/           Slack Bot：把消息委托给 coding-agent
│   └── pods/          运维工具：管理 vLLM GPU 部署
├── AGENTS.md          开发规范（给人和 AI 的）
└── biome.json         Lint / Format 配置
```

**技术栈**：TypeScript + ES Modules + tsgo 构建 + Biome Lint + Vitest 测试

---

## 第二章：分层架构（待展开）

> 参见：[02-architecture.md](02-architecture.md)

**预告**：
- 四个核心包的依赖图
- 每层的职责边界
- 数据如何在各层之间流动
- 关键接口（`Context`、`AgentMessage`、`AgentEvent`）的形状

---

## 第三章：LLM 抽象层 packages/ai（待展开）

> 参见：[03-ai-package.md](03-ai-package.md)

**预告**：
- `Api = KnownApi | (string & {})` 的开放枚举技巧
- Provider 插件注册中心（`api-registry.ts`）
- `AssistantMessageEventStream`：异步迭代器 + 双接口设计
- `AssistantMessageEvent`：判别联合类型做状态机
- `Model<TApi>` 泛型：如何让 Provider 特有选项类型安全

---

## 第四章：Agent 运行时 packages/agent（待展开）

> 参见：[04-agent-package.md](04-agent-package.md)

**预告**：
- `CustomAgentMessages` 声明合并：开放消息类型而不破坏库
- `convertToLlm`：应用消息与 LLM 消息之间的转换边界
- Agent Loop 状态机：从 prompt 到 agent_end 的完整生命周期
- Steering vs Follow-up：两种运行时干预机制
- `EventStream<T, R>`：通用生产者-消费者流

---

## 第五章：终端 UI packages/tui（待展开）

> 参见：[05-tui-package.md](05-tui-package.md)

**预告**：
- 差量渲染策略：三种情况的处理逻辑
- CSI 2026 同步输出：无闪烁的原理
- `Component` 接口：`render(width): string[]` 的约束
- `Focusable` + `CURSOR_MARKER`：IME 输入法支持
- 东亚字符宽度处理

---

## 第六章：Coding Agent CLI packages/coding-agent（待展开）

> 参见：[06-coding-agent-package.md](06-coding-agent-package.md)

**预告**：
- 三种运行模式：Interactive / Print / RPC
- Session 作为 JSONL 追加日志 + 树形分支
- Extension 系统：注册工具、命令、生命周期钩子
- Skill 系统：Markdown 描述的工作流模板
- 配置优先级：全局 → 项目 → 命令行参数

---

## 第七章：横切关注点（待展开）

> 参见：[07-cross-cutting.md](07-cross-cutting.md)

**预告**：
- TypeScript 开放枚举：`KnownX | (string & {})`
- TypeBox：运行时校验 + 编译时类型一体化
- 声明合并（Declaration Merging）作为扩展机制
- 流式优先设计哲学：为什么所有 I/O 都是 Stream
- sourceId 热插拔：动态注册/注销 Provider

---

*下一步：展开 [第二章 分层架构](02-architecture.md)*
