# 第五章：终端 UI packages/tui

> [← 第四章](04-agent-package.md) | [第六章 →](06-coding-agent-package.md)

源码位置：`packages/tui/src/`

---

## 5.1 这一层解决什么问题

终端 UI 有几个天然的难题：

1. **闪烁**：全量重绘屏幕会导致可见的闪烁
2. **性能**：流式 LLM 输出时每个 token 都触发渲染
3. **宽度计算**：CJK 字符宽度为 2，普通 ASCII 为 1，`string.length` 不可信
4. **IME（输入法）支持**：中文/日文输入时，候选框位置要对准光标
5. **能力检测**：不同终端支持不同的特性（图片、键盘协议……）

---

## 5.2 差量渲染策略

```
每次渲染时，TUI 比较新旧两次的 string[] 输出：

情况 1：首次渲染
  → 直接输出所有行

情况 2：终端宽度变化 or 有行在视口之外发生变化
  → 清屏，全量重绘

情况 3：普通更新（最常见）
  → 找到第一个变化的行
  → 移动光标到那一行
  → 清除从该行到屏幕底部
  → 输出变化的行
```

**全部包裹在同步输出（CSI 2026）中**：

```
\x1b[?2026h          ← 开始同步模式（终端暂停渲染）
...所有 ANSI 输出...
\x1b[?2026l          ← 结束同步模式（终端一次性渲染）
```

用户看到的是**原子更新**，不会看到中间状态，消除闪烁。

---

## 5.3 Component 接口：简洁的核心约束

```typescript
interface Component {
  render(width: number): string[];   // 唯一必须实现的方法
  handleInput?(data: string): void;  // 可选：处理键盘输入
  invalidate?(): void;               // 可选：标记需要重新渲染
  wantsKeyRelease?: boolean;         // 可选：是否需要 keyRelease 事件
}
```

**关键约束**：`render()` 返回的每个字符串的**可见宽度不能超过 `width`**。

TUI 框架会验证这个约束，违反时报错（而不是静默截断），强迫组件开发者正确处理宽度。

**组件只返回字符串数组**，不直接操作终端，测试非常容易：

```typescript
const text = new Text("Hello World");
const lines = text.render(80);
assert(lines.length === 1);
assert(lines[0] === "Hello World");
```

---

## 5.4 Focusable + CURSOR_MARKER：IME 支持

IME（中文输入法等）需要知道**光标在屏幕上的精确位置**才能弹出候选词窗口。

但 `render()` 只返回字符串，不知道光标位置。解法：

```typescript
// 组件在渲染时，在光标位置插入一个特殊标记
const CURSOR_MARKER = "\x1b_cursor\x1b\\";  // APC 转义序列（终端会忽略它）

// 组件的 render() 实现
render(width: number): string[] {
  const before = this.text.slice(0, this.cursor);
  const after = this.text.slice(this.cursor);
  const marker = this.focused ? CURSOR_MARKER : "";
  return [`${before}${marker}${after}`];
}
```

TUI 框架扫描输出，找到 `CURSOR_MARKER`：
1. 从输出中**删除**标记（终端看不到它）
2. 计算标记所在的行列坐标
3. 用 ANSI escape 把**硬件光标**移动到那个位置

**结果**：IME 候选词窗口准确对准用户正在输入的位置。

---

## 5.5 东亚字符宽度处理

```typescript
import { getEastAsianWidth } from "get-east-asian-width";

function visibleWidth(str: string): number {
  let width = 0;
  for (const char of str) {
    const w = getEastAsianWidth(char.codePointAt(0)!);
    width += w === "wide" || w === "fullwidth" ? 2 : 1;
  }
  return width;
}

// 示例
visibleWidth("Hello世界") // → 9（不是 length 的 11）
// H=1, e=1, l=1, l=1, o=1, 世=2, 界=2
```

所有组件在计算是否超出 `width` 时，都必须用 `visibleWidth` 而不是 `string.length`。

---

## 5.6 内置组件一览

**文本/展示类**：

| 组件 | 功能 |
|------|------|
| `Text` | 多行文本，支持自动换行、内边距 |
| `TruncatedText` | 单行截断，末尾加省略号 |
| `Markdown` | 完整 Markdown 渲染 + 语法高亮 |
| `Image` | 内联图片（Kitty/iTerm2 协议） |
| `Spacer` | 垂直间距 |

**输入类**：

| 组件 | 功能 |
|------|------|
| `Input` | 单行输入框，支持自动补全、单词导航 |
| `Editor` | 多行编辑器，支持自动补全、粘贴、undo/redo |
| `SelectList` | 键盘导航列表 |
| `SettingsList` | 设置面板（支持子菜单） |

**布局/容器类**：

| 组件 | 功能 |
|------|------|
| `Container` | 垂直排列子组件 |
| `Box` | 内边距 + 背景色 |
| `Overlays` | 模态框/弹窗（居中、锚点、绝对定位） |

**其他**：

| 组件 | 功能 |
|------|------|
| `Loader` / `CancellableLoader` | 动画 spinner，支持取消 |
| `ProcessTerminal` | 展示子进程的终端输出 |

---

## 5.7 键盘输入抽象

```typescript
import { Key, matchesKey } from "@mariozechner/pi-tui";

// 常用匹配
if (matchesKey(data, Key.ctrl("c"))) { /* Ctrl+C */ }
if (matchesKey(data, Key.shift("tab"))) { /* Shift+Tab */ }
if (matchesKey(data, Key.alt("left"))) { /* Alt+Left */ }
if (matchesKey(data, "\r")) { /* Enter */ }
```

TUI 会检测终端是否支持 **Kitty 键盘协议**（更精确的按键区分，如区分 Shift+Enter 和 Enter），有则启用，无则降级。

---

## 5.8 架构小结

**设计哲学**：

- `render(width): string[]` 是极简而正确的接口——可测试、可组合、职责单一
- 差量渲染 + CSI 2026 同步输出解决了终端 UI 最大的两个问题（性能 + 闪烁）
- `CURSOR_MARKER` 是一个优雅的间接层，让 IME 支持在不改变组件接口的前提下工作
- 宽度验证强制执行，让 bug 在开发时暴露，而不是在用户的奇怪终端里

---

*下一步：[第六章 Coding Agent CLI packages/coding-agent →](06-coding-agent-package.md)*
