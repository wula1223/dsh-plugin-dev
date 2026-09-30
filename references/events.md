## 2. 钩子 / 事件

### 2.1 五种分发模式

```ts
ctx.on('event-name', handler)      // 监听
ctx.emit('event-name', payload)    // 触发
```

| 模式 | 是否 await | 顺序 | 返回值规则 | 用途 |
|---|---|---|---|---|
| `emit` | 否 | 注册顺序，**同步执行** | **忽略返回值** | 广播通知 |
| **`bail`** | 否 | 注册顺序 | **首个非 `null`/`false`/`undefined` 的值胜出并停止** | 短路决策 |
| `serial` | **是** | 注册顺序，等异步 | 首个非 `null`/`false`/`undefined` 终止后续 | 按序执行 |
| **`waterfall`** | **是** | 注册顺序 | **包装下游返回值，形成处理链** | 拦截/网关 |
| `parallel` | 是 | 并行 | — | 并行扇出 |

**waterfall 精确语义**：

```ts
const output = await ctx.waterfall('my-plugin/transform', input, async () => input)

ctx.on('my-plugin/transform', async (_input, next) => {
  const downstream = await next()      // 必须调用 next()
  return downstream.trim()             // 包装后外传
})
```

> ⚠️ **waterfall 监听器必须调用 `next()`。** 不调用就**短路整个流水线** —— 这是**故意**的设计，用于实现拦截/网关逻辑。

`bail` 精确语义：

```ts
const result = ctx.bail('some-check', input)
ctx.on('some-check', (input) => {
  if (shouldBlock(input)) return 'blocked'   // 返回值 → 停止
  // 返回 null / false / undefined → 继续下一个监听器
})
```

### 2.2 类型安全（TS 声明合并）

```ts
import '@deepseek-ai/cordis'

declare module '@deepseek-ai/cordis' {
  interface Events {
    'my-plugin/ready': (payload: { id: string }) => void
    'my-plugin/check': (input: string) => boolean | undefined
    'my-plugin/transform': (input: string, next: () => Promise<string>) => Promise<string>
  }
}
```

### 2.3 命名与归属 ⭐

- Cordis 事件遵循 **`namespace/action`** 命名：`agent/pre-step`、`agent/request`、`agent/request-error`、`tools/result`、`session/event`
- ⚠️ **`turn/*`、`step/*`、`tool/call`、`tool/result`、`compaction/*` 是「持久化会话事件类型」，不是 Cordis 事件。**
  要观察它们，请监听 **`session/event`** 并检查 `event.type`
- **归属规则**：工具流水线 → `ctx.tools`；模型流式输出 → `ctx.llm`；agent 协调 → `ctx.agents`
- **拦截/策略优先用事件；直接能力调用优先用服务方法**

### 2.4 监听器也是效果

`ctx.on()` 注册的监听器**在插件卸载时自动移除**，无需手动 off。

### 2.5 示例：工具日志插件

```ts
import type { Context } from '@deepseek-ai/cordis'
import '@deepseek-ai/dsh-tools'

export const name = 'tool-logger'

export function apply(ctx: Context) {
  ctx.on('tools/result', (exec, result) => {
    console.log(`[tool] ${exec.name}(${JSON.stringify(exec.arguments)})`)
    const text = result.content.map(b => b.type === 'text' ? b.text : '').join('')
    console.log(`[tool result] ${text.slice(0, 100)}`)
  })
}
```

### 2.6 三大事件域 ⭐

**事件就是扩展点，而选对事件域是大多数改动的第一个决定。**

| 事件域 | 形态 | 何时用 |
|---|---|---|
| **会话事件** | 追加到日志、并通过 `session/event` 广播的**持久事实** | 某个事实必须在重新加载后仍然存在 |
| **Agent 事件**（`agent/*`） | 携带活跃 `Agent`：inbox、步骤、状态、请求、验证、续跑 | 观察或**拦截进行中**的工作 |
| **能力事件**（`fs/*`、`tools/*`、`telemetry/*`） | 向某个 seam 附加策略与适配器 | 无需导入循环即可扩展能力 |

⚠️ **`turn/*`、`step/*`、`system/message`、`user/message`、`assistant/message`、`assistant/attempt`、`tool/*` 是持久会话事件；其余才是实时扩展点。**

**必须调用 `next()` 的 waterfall 事件**：
`agent/pre-step` · `agent/request` · `llm/stream` · `tools/pre-execute` · `tools/execute` · `tools/post-execute`

**serial 事件（没有 `next()`）**：`agent/turn-stopping`

### 2.7 ⚠️ 别把 Claude Code 的术语搬过来

搜到的"Agent 扩展机制"资料大多是 **Claude Code** 语境。对照如下：

| Claude Code 概念 | DSH 里的对应 |
|---|---|
| `UserPromptSubmit` hook | → **`agent/pre-step`**（waterfall，可改写或拒绝已领取消息） |
| PreToolUse / PostToolUse | → `tools/pre-execute` / `tools/post-execute` |
| **hooks.json（整个配置文件）** | ✅ **官方有桥接插件**：`@deepseek-ai/dsh-hooks-claude-code` |
| Output Styles | ⚠️ **DSH 内核无此概念**；社区插件 **`dsh-output-styles`** 提供等价能力 |
| Knowledge | ❌ **无此概念** —— 用 AGENTS.md / skill / `ctx.systemPrompt` 片段 |
| MCP server | ✅ 有，经官方 **`@deepseek-ai/dsh-mcp-client`**（把 MCP 工具注册到 `ctx.tools`） |
| Subagent | ✅ 有，且是 **seam**（见 `docs/subsystems/subagent.md`） |

#### 🔌 hooks 的两个层次 ⭐

| 层次 | 怎么做 | 适用 |
|---|---|---|
| **底层** | 在插件里 `ctx.on('agent/pre-step', …)` 注册 Cordis 事件 | 要写代码、要深度控制 |
| **声明式** | 装官方 `@deepseek-ai/dsh-hooks-claude-code`，用 Claude Code 格式的 `hooks.json` 配置 | **不写插件代码**，直接复用 Claude Code 的 hook 生态 |

官方 hooks 家族（源：`packages/hooks/`）：

| 包 | 作用 |
|---|---|
| `@deepseek-ai/dsh-hooks-claude-code` | 跑 **Claude Code** 的 `hooks.json` / settings hook 配置 |
| `@deepseek-ai/dsh-hooks-codex` | 跑 **Codex** 的 `hooks.json` |
| `@deepseek-ai/dsh-hook-protocol` | 共享 wire protocol：matcher 引擎、stdin/exit-code/stdout 编解码、多 hook 合并、**`hook/*` 会话事件** |

> 💡 **一句话**：钩子的**底层机制始终是 Cordis 事件**（`ctx.on`）；但**不想写代码时，官方提供了 Claude Code / Codex 的 `hooks.json` 兼容桥** —— 它把外部 hook 配置映射到 DSH 的拦截 seam 上。

---
