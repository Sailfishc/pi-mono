# 第六章：Coding Agent CLI packages/coding-agent

> [← 第五章](05-tui-package.md) | [第七章 →](07-cross-cutting.md)

源码位置：`packages/coding-agent/src/`

---

## 6.1 这一层的角色

`coding-agent` 是整个系统的**集成层**，它不实现核心算法，只做三件事：

1. 把 `ai` + `agent` + `tui` **组合**起来
2. 把对话**持久化**成文件（Session）
3. 提供**扩展机制**让外部修改行为（Extension / Skill / 配置）

源码结构：

```
src/
├── core/
│   ├── session-manager.ts    会话持久化（JSONL）
│   ├── agent-session.ts      Agent + TUI 的绑定胶水
│   ├── event-bus.ts          Extension 间通信的事件总线
│   ├── extensions/           Extension 系统（types.ts 1300+ 行）
│   ├── skills.ts             Skill 加载与调用
│   ├── prompt-templates.ts   提示词模板
│   ├── tools/                内置工具（read/write/edit/bash...）
│   └── system-prompt.ts      系统提示词组装
└── cli/
    ├── args.ts               命令行参数解析
    └── app.ts                主程序入口
```

---

## 6.2 三种运行模式

```
pi [prompt]              Interactive 模式（TUI，默认）
pi -p "..."              Print 模式（非交互，打印后退出）
pi --mode rpc            RPC 模式（stdin/stdout JSON 协议）
```

**Interactive 模式**：完整 TUI 界面，Session 持久化，Extension 全部加载。

**Print 模式**（`-p`）：适合脚本/管道：

```bash
pi -p "检查这段代码有没有安全问题" < suspicious.ts
```

**RPC 模式**：供非 Node.js 宿主（IDE 插件、桌面应用）通过子进程集成：

```
宿主进程 ─stdin─► pi --mode rpc
          ◄stdout─ JSON 事件流
```

三种模式与 Extension 的关系：`ExtensionContext.hasUI` 标识当前是否有 UI。Extension 在调用 `ctx.ui.confirm()` 等方法前应检查此字段，Print 和 RPC 模式下 UI 方法会降级为 no-op 或默认值。

---

## 6.3 Extension 系统（核心）

Extension 是最强大的扩展机制，可以深度修改 Agent 的任何行为。`extensions/types.ts` 有 1300+ 行，完整定义了整个扩展体系。

### 6.3.1 Extension 的三个权限上下文

Extension 在不同阶段拿到不同的上下文对象，代表不同的权限级别：

```
ExtensionFactory(pi: ExtensionAPI)    注册阶段：装配工具、命令、事件处理器
         │
         ▼
ExtensionContext (ctx)                事件处理阶段：只读状态 + 轻量操作
         │
         ▼
ExtensionCommandContext (ctx)         命令处理阶段：可以控制 Session 生命周期
```

`ExtensionCommandContext` 继承 `ExtensionContext`，额外提供 `newSession()`、`fork()`、`navigateTree()`、`switchSession()` 等破坏性操作。这些方法只在用户主动触发命令时才安全——自动事件处理器不应该随便操控 Session。

### 6.3.2 事件系统：有些事件有返回值

这是整个 Extension 系统最精妙的设计。`on()` 的处理器不只是"观察者"——有些事件的返回值会直接影响行为：

```typescript
// 普通观察性事件 - 返回值忽略
api.on("agent_start", (event, ctx) => { /* 只能观察 */ });

// 可取消的事件 - 返回 { cancel: true } 阻止操作
api.on("session_before_switch", async (event, ctx) => {
    if (hasUnsavedWork) return { cancel: true };
});

// 可修改用户输入
api.on("input", (event, ctx) => {
    if (event.text.startsWith("@gpt:")) {
        return { action: "transform", text: event.text.replace("@gpt:", "") };
    }
    // action: "handled" → 扩展已完全处理，不进入 agent
});

// 可阻断工具调用
api.on("tool_call", (event, ctx) => {
    if (isToolCallEventType("bash", event)) {
        if (event.input.command.includes("rm -rf")) {
            return { block: true, reason: "危险命令被拦截" };
        }
    }
});

// 可修改工具结果
api.on("tool_result", (event, ctx) => {
    if (isReadToolResult(event)) {
        return { content: [{ type: "text", text: redact(event.content) }] };
    }
});
```

事件结果速查表：

| 事件 | 返回值 | 效果 |
|------|-------|------|
| `input` | `{ action: "transform", text }` | 重写用户输入 |
| `input` | `{ action: "handled" }` | 扩展已处理，不走 agent |
| `tool_call` | `{ block: true, reason }` | 阻止工具执行 |
| `tool_result` | `{ content, isError }` | 替换工具返回值 |
| `session_before_switch` | `{ cancel: true }` | 阻止 Session 切换 |
| `session_before_compact` | `{ cancel: true }` 或 `{ compaction }` | 阻止或替换压缩结果 |
| `before_agent_start` | `{ systemPrompt }` | 替换本轮系统提示词（多个 Extension 链式叠加） |
| `context` | `{ messages }` | 修改发给 LLM 的消息 |

这是**观察者模式的进化版**——不只能观察，还能拦截和修改。

### 6.3.3 ToolDefinition：扩展工具有 UI 渲染能力

Extension 注册的工具比底层 `AgentTool` 多了两个可选方法：

```typescript
// extensions/types.ts:335
export interface ToolDefinition<TParams, TDetails> {
    name: string;
    label: string;
    description: string;
    parameters: TParams;

    execute(toolCallId, params, signal, onUpdate, ctx): Promise<AgentToolResult<TDetails>>;

    // 额外的 UI 渲染钩子
    renderCall?:   (args: Static<TParams>, theme) => Component;   // 调用时展示什么
    renderResult?: (result, options, theme) => Component;          // 结果怎么渲染
}
```

- `renderCall`：工具被调用时的 TUI 展示（默认：工具名 + 参数文本）
- `renderResult`：工具结果的 TUI 展示（默认：纯文本）

例如 `edit` 工具可以渲染 diff 视图，`bash` 工具渲染带颜色的终端输出。

`execute` 里额外有 `ctx: ExtensionContext` 参数——工具执行时可以操作 UI（弹确认框、显示进度等）。

### 6.3.4 registerProvider：Extension 可以接入全新的 LLM

Extension 甚至可以注册整个新的 LLM Provider（含 OAuth 认证流程）：

```typescript
api.registerProvider("corp-ai", {
    baseUrl: "https://ai.corp.com/v1",
    api: "openai-completions",
    models: [
        { id: "corp-gpt-4", name: "Corp GPT-4", reasoning: false,
          input: ["text"], cost: {...}, contextWindow: 128000, maxTokens: 4096 }
    ],
    oauth: {
        name: "Corp SSO",
        login: async (callbacks) => { /* SSO 登录流程 */ return credentials; },
        refreshToken: async (credentials) => { /* 刷新 token */ return newCredentials; },
        getApiKey: (credentials) => credentials.access_token,
    }
});
```

用户就能 `/login corp-ai`，然后切换到企业内部模型——对 `pi` 核心代码零修改。

### 6.3.5 ExtensionUIContext：扩展可以操作整个 UI

`ctx.ui` 提供的能力远超想象：

```typescript
// 对话框
const choice = await ctx.ui.select("选择环境", ["dev", "staging", "prod"]);
const ok = await ctx.ui.confirm("确认删除？", "此操作不可逆");
const name = await ctx.ui.input("输入项目名");

// 状态栏
ctx.ui.setStatus("git", "main ✓");
ctx.ui.setStatus("sync", undefined);  // 清除

// 编辑器上方/下方插入自定义 Widget
ctx.ui.setWidget("preview", ["预览第一行", "第二行"]);

// 完全替换 Footer / Header 组件
ctx.ui.setFooter((tui, theme, footerData) => new MyFooter(tui, theme, footerData));

// 完全替换输入编辑器（可以做 Vim 模式）
ctx.ui.setEditorComponent((tui, theme, keybindings) => new VimEditor(tui, theme, keybindings));
```

`setEditorComponent` 是最惊人的一个——Extension 可以完全替换输入编辑器，实现 Vim 模式、IDE 风格等，只需继承 `CustomEditor` 并覆盖 `handleInput`。

### 6.3.6 Extension 注册方式与结构

```typescript
// my-extension.ts
import type { ExtensionFactory } from "@mariozechner/pi-coding-agent";

const extension: ExtensionFactory = (pi) => {
    // 注册工具
    pi.registerTool({ name: "my_tool", ... });

    // 注册斜杠命令
    pi.registerCommand("mycommand", {
        description: "我的命令",
        handler: async (args, ctx) => { ... }
    });

    // 注册键盘快捷键
    pi.registerShortcut("ctrl+shift+m", {
        description: "我的快捷键",
        handler: (ctx) => { ctx.ui.notify("触发了！"); }
    });

    // 监听事件
    pi.on("agent_end", (event, ctx) => { ... });
};

export default extension;
```

**加载位置**（优先级从低到高）：

```
~/.pi/agent/extensions/    全局 Extension
.pi/extensions/            项目级 Extension
```

---

## 6.4 Session 设计：JSONL 追加日志 + 树形分支

每个 Session 是一个 `.jsonl` 文件，第一行是 header，后续每行是一条 Entry：

```jsonl
{"type":"session","version":3,"id":"...","timestamp":"...","cwd":"/project"}
{"type":"message","id":"a1","parentId":null,"timestamp":"...","message":{...}}
{"type":"message","id":"a2","parentId":"a1","timestamp":"...","message":{...}}
{"type":"model_change","id":"a3","parentId":"a2","provider":"openai","modelId":"gpt-4o"}
{"type":"message","id":"b1","parentId":"a1","timestamp":"...","message":{...}}  ← 分支！
```

`parentId` 形成一棵树：

```
a1 (用户消息)
├── a2 (AI 回复)
│   └── a3 (切换模型事件)
└── b1 (用户在 a1 处开了新分支)
```

**四个设计优势**：
- **追加写，从不修改**：历史永不丢失，任何操作都是新增行
- **树形分支**：`/fork` 从任意节点开新分支探索不同方向
- **可回滚**：`/tree` 导航到任意历史节点
- **人类可读**：JSONL 可以用 `jq` 等工具直接分析

**Entry 类型**：

| type | 含义 |
|------|------|
| `message` | 一条 AgentMessage（user/assistant/toolResult/自定义） |
| `model_change` | 切换了模型 |
| `thinking_level_change` | 调整了思考级别 |
| `compaction` | 上下文压缩摘要（含 `firstKeptEntryId` 标记裁剪点） |
| `branch_summary` | 分支节点摘要（导航时生成） |
| `custom` | Extension 专用持久化数据（不进 LLM 上下文） |

**Extension 写入 Session 的两种方式**：

```typescript
// 方式一：纯持久化，Session 重载时恢复 Extension 状态，不进 LLM
api.appendEntry("my-ext", { lastSyncTime: Date.now() });

// 方式二：进入对话历史，LLM 能看到，可自定义渲染
api.sendMessage({
    customType: "artifact",
    content: "artifact 内容",
    display: "Artifact 已生成"
});
```

---

## 6.5 Skill 系统

Skill 是**给 LLM 看的操作说明书**，用 Markdown 写：

```markdown
---
name: code-review
description: Use when user asks for code review
---
## Steps
1. 读取目标文件
2. 检查逻辑错误、安全漏洞、性能问题
3. 输出分级列表（Critical / Warning / Info）
```

调用：`/skill:code-review src/auth.ts`

**自动发现位置**：

```
~/.pi/agent/skills/
.pi/skills/
.pi/agent/skills/
.agents/skills/
```

**Skill vs Extension 的核心区别**：

| | Skill | Extension |
|---|---|---|
| **形态** | Markdown 文件 | TypeScript 模块 |
| **作用** | 注入 LLM 的提示词/消息 | 注册工具/命令/钩子，修改系统行为 |
| **能力** | 描述"怎么做" | 添加系统"能做什么" |

---

## 6.6 内置工具

```
packages/coding-agent/src/core/tools/

read.ts    读取文件（支持行号范围，大文件自动截断）
write.ts   写入/覆盖文件
edit.ts    精确编辑（old_string → new_string，类似 str_replace）
bash.ts    执行 shell 命令（timeout + abort + 实时输出）
grep.ts    内容搜索
find.ts    文件查找
ls.ts      目录列表
```

所有工具的共同设计：
- TypeBox schema 定义参数（同时是文档和运行时校验器）
- `onUpdate` 回调支持流式输出（bash 工具实时推送输出）
- 超出大小限制时有截断策略（head / tail / middle）

---

## 6.7 配置系统的优先级

```
命令行参数                    （最高）
    ↑
.pi/settings.json            （项目级）
    ↑
~/.pi/agent/settings.json    （全局）
    ↑
代码默认值                    （最低）
```

**系统提示词控制文件**（项目级 > 全局优先）：

| 文件 | 作用 |
|------|------|
| `AGENTS.md` | 项目上下文，自动追加到系统提示词 |
| `SYSTEM.md` | 完全替换系统提示词 |
| `APPEND_SYSTEM.md` | 追加到系统提示词末尾 |

Extension 还可以通过 `before_agent_start` 事件的 `{ systemPrompt }` 返回值动态替换本轮系统提示词，多个 Extension 的替换会链式叠加。

---

## 6.8 架构小结

`coding-agent` 的三件事：

```
1. 组合        ai + agent + tui → 一个完整的 Coding Agent
2. 持久化      Session = JSONL 追加日志 + 树形分支
3. 扩展        Extension（代码）+ Skill（Markdown）+ 配置（JSON）
```

Extension 系统是这一层最精妙的设计，它的核心思想：

- **三级权限上下文**：注册 → 事件处理 → 命令处理，权限逐级递增
- **有返回值的事件**：观察者模式进化成拦截器模式，Extension 可以取消、转换、替换
- **UI 可替换性**：Footer、Header、输入编辑器都可以被 Extension 替换
- **Provider 可扩展**：通过 `registerProvider` 接入全新 LLM，无需修改核心代码

---

*下一步：[第七章 横切关注点 →](07-cross-cutting.md)*
