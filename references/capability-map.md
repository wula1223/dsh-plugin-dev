### 1.7 能力地图：`ctx` 键与服务 ⭐

**"我要加的功能该挂在哪？"** —— 官方给出了权威答案。核心包向 Cordis 树贡献能力，各自占据一个稳定的 `ctx` 键：

| 包 | 职责 | `ctx` 键 |
|---|---|---|
| `core/session` | 仅追加的 `SessionEvent` 日志和内存存储 | `ctx.sessions` |
| `core/system-prompt` | 提示词片段与工具 schema 的组装 | `ctx.systemPrompt` |
| `core/tools` | 作用域化的工具注册表和带把关的执行流水线 | `ctx.tools` |
| `core/agent` | `Agent` 接口、活跃 agent 注册表和 `agent/*` 事件 | `ctx.agents` |
| `core/agent-loop` | 实现该接口的默认驱动器 | `ctx.agentLoop` |
| `core/scope` | 按 agent 划分作用域的注册原语 | 库，无 ctx 键 |
| `llm/llm` | 消息与流式词汇表，以及适配器 seam | `ctx.llm` |
| `webhook/webhook` | 已认证 delivery 的分派和 Workspace Session 创建 | `ctx.webhookRuntime` |

### 新行为的归属位置（官方映射表）

| 目标 | 机制 |
|---|---|
| 添加模型提供方 | 在 `ctx.llm` 上注册其适配器 |
| 添加面向模型的能力 | 在 `ctx.tools` 上注册；其 schema 加入提示词组装 |
| 让某个会话拥有不同的能力集合 | 组装一个 agent preset；其中的服务行需要 `isolate` realm |
| 添加 shell 执行 | 注册 `ctx.shell` 后端；本地后端经 `ctx.subprocess` spawn 进程 |
| 添加持久化终端执行 | 注册 `ctx.terminals` 后端和 `dsh-tool-terminal` |
| 添加用户命令 | 在 `ctx.commands` 上注册；无需模型轮次即可分派 |
| 管理后台任务 | 在 `ctx.jobs` 上注册；`job_*` 工具读取或停止任务 |
| 从外部 webhook 启动 Session | 在 `ctx.webhookRuntime` 上注册可信规则，并挂载提供方适配器 |
| 添加文件系统访问或策略 | 注册 `ctx.fs` 提供方，或监听 `fs/*` 事件 |
| 限制所启动的进程 | 使用 `ctx.sandbox` 后端；消费方在启动进程前包装 argv |
| **拦截请求、工具或轮次** | 使用相应的 `agent/*` 或 `tools/*` 事件；`agent/turn-stopping` 会停止轮次 |
| **添加模型可见上下文** | 调用 `agent.inject()`；它会落到下一次获准的请求中 |
| 添加 UI 或编辑器集成 | 驱动 `ctx.agents` 并从 `session/event` 渲染 |
| 添加 Web Client Chat 节点 | 注册 `ConversationNodeDefinition` + keyed renderer |
| 添加持久会话状态 | 扩展 `SessionEventMap`；从日志渲染和回放 |
| 生成会话标题 | 注册唯一的 `ctx.sessionTitle` 提供方 |
| 管理同会话目标 | 使用 `ctx.goals`；通过 `agent/*` 续跑 |
| 在轮次边界 fork 会话 | `ctx.agents.create({ sessionId, seed, meta: { parentSession, seedLength } })` |
| 在新后端存储会话 | 基于共享的句柄脚手架实现 `SessionPersistence`（`create`/`open`/`stat`/`list`/`export`） |
| 将注册项限定到单个 agent | 使用该 agent 的 `agent.ctx` |

### seam（能力接缝）三角色

一项可替换能力 = **Service Definition**（声明接口）+ **Service Provider**（实现）+ **Consumer**（使用它，通常是面向模型的工具）。一个包可以合并承担多个角色，但**单一角色本身不是 seam** —— 添加一项能力意味着把三者一并设计。

> **替换一个提供方就能改变整个产品。** 文件系统与进程提供方共享同一个执行世界，所以把它们指向远程沙箱，**Bash、PTY 和 LSP 会一并搬过去**，无需提供方专用 fork。subagent 提供方在同一接口之后也千差万别 —— 从新建一个子 agent，到把一个轮次委派给另一个产品。

完整能力图见官方 `docs/capability-seams.zh.md`（61 KB）。

---
