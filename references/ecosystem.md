## 10. 生态地图：装什么插件解决什么问题（🔵 社区，非官方）

> ⚠️ **本节除「官方包谱系」表外，全部是 🔵 第三方社区插件** —— 官方文档里没有它们。
> 名字与描述已在 npm 上核实，但**它们不是官方规定，也不代表官方推荐**。选用请自行判断。
> 装之前务必回看 **§5.1 / §5.2 / §5.3** —— 社区插件正是「显示异常 / 影子包遮蔽 / 依赖悬空」的高发区。

**这一节是给"用户"看的；写插件的人需要的是 §1.7 的 `ctx` 键。**

| 需求 | 方案 |
|---|---|
| **MCP 服务器** | 官方 **`@deepseek-ai/dsh-mcp-client`**（连接 MCP 服务器并把其工具注册到 `ctx.tools`）· 官方 `@deepseek-ai/dsh-mcp-resources`（scoped 资源读写） |
| MCP 管理界面 | `dsh-mcp`（管理 UI + tool search 热注入，工具列表不撑爆上下文）· **`dsh-mcp-manage`**（设置页管理 `cordis.patch.yml` 里的 MCP 服务器，逐个跑真实 initialize 握手验证）· **`dsh-mcp-manager`** · `@xxxyz/dsh-mcp-manager` · `dsh-mcp-market`（MCP 市场）· `dsh-mcp-setting`（设置页改 patch） |
| **Hooks（不写代码）** | 官方 **`@deepseek-ai/dsh-hooks-claude-code`**（跑 Claude Code 的 `hooks.json`）· `@deepseek-ai/dsh-hooks-codex`（Codex 格式）· 底座 `@deepseek-ai/dsh-hook-protocol` |
| **记忆 / 跨会话持久化** | `@openviking/dsh-memory-plugin` · `@furongjun1999/dsh-memory` · **`dsh-memory-search`**（向量化语义检索 + 时间衰减遗忘）· **`dsh-markdown-memory`**（纯 Markdown 为真值源，一个事实一个文件）· `dsh-mnemon` · `dsh-mnemosyne` · `dsh-memory-eternal` · `@a9i5k4/dsh-auto-memory` · OpenViking |
| **知识库接入** | **`dsh-yuque-kb`**（把语雀文档作为外部记忆，对话中自动检索并注入相关片段，支持目录快照检索 / 云端全文搜索 / 在线阅读） |
| **输出样式** | `dsh-output-styles`（Claude Code outputStyles 等价，运行时切换）· `@auggieteo/dsh-output-styles`（设置页管理） |
| **技能管理** | `@michengai/dsh-skills-manager`（本地技能库 + 从 Git 仓库安装） |
| **Git 凭据 / 推送策略** | `dsh-git-forge`（账号库 + 按项目授权 + push 拦截） |
| **搜索** | `dsh-free-search` |
| **交互式 UI** | `@changfenhuang/dsh-genui`（模型在回复里内联渲染图表/表单） |
| **插件市场** | `dshmarket`（可视化市场，一键装社区插件） |
| **多代理协作** | 实验性 **Agent Teams** —— `ctx.agentTeams` 上公开发布、显式启用的协作 seam，在可继续 subagent 之上提供**持久 roster、任务板和 mailbox** |
| **子代理** | subagent seam（`docs/subsystems/subagent.md`） |

### 🟢 官方包谱系（`@deepseek-ai/*`，按 seam 划分）

> 下表全部为 🟢 官方包（`@deepseek-ai` scope，维护者 `imccyu` / `tianyicui-deepseek`）。

写插件时**不要**把下面这些写进 `dependencies` —— 它们由宿主提供（见 §5.2）：

| 能力 | 官方包 | `ctx` 键 |
|---|---|---|
| 工具注册表 | `dsh-tools` | `ctx.tools` |
| 会话存储 | `dsh-session` | `ctx.sessions` |
| 模型适配 | `dsh-llm` | `ctx.llm` |
| Agent / 循环 | `dsh-agent`、`dsh-agent-loop` | `ctx.agents` |
| 子进程 | `dsh-subprocess` + `dsh-subprocess-local`、`dsh-native-command` | `ctx.subprocess` |
| Shell 执行 | `dsh-bash-local`、`dsh-pwsh-local` | `ctx.shell` |
| 文件系统 | `dsh-fs-local`、`dsh-tool-fs` | `ctx.fs` |
| Web 访问 | `dsh-web`（search/fetch 提供方注册表） | `ctx.web` |
| MCP | `dsh-mcp-client`、`dsh-mcp-resources` | 注册到 `ctx.tools` |
| Hooks | `dsh-hooks-claude-code`、`dsh-hooks-codex`、`dsh-hook-protocol` | 拦截 seam |
| 插件自省 | `dsh-tool-cordis`（检视实时运行时、挂载/卸载模型写的插件） | — |
| CLI | `@deepseek-ai/dsh`（`dsh plugin` / `--profile` / `--dump-config`） | — |

### 关于 MCP 与插件的关系

- **MCP** 是连接**外部独立进程**（数据库、浏览器守护进程、公司内部 API）的通用协议
- **插件**是 DSH 的**原生扩展单元**，运行在框架内部，能实现更深度的集成（读会话日志、注入上下文、钩住 UI）
- DSH 的策略是 **「插件优先，MCP 作为适配层」** —— 官方就是用 `dsh-mcp-client` 这个**插件**去接入 MCP 的

### 关于 Subagent / Workflow

- **Subagent** 在**隔离的上下文**中独立运行自己的循环，避免污染主对话上下文
- **Workflow** 用脚本编排多个子代理并行/串行执行，最后汇总为一个结果
- 二者都是 **agent 侧能力**（不是插件作者要实现的机制），由 `subagent` / `workflow` 工具暴露