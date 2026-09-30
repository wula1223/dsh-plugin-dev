# dsh-plugin-dev

> **DeepSeek Harness 插件 / 钩子 / Skill 开发规范** —— 给 AI agent 和人类开发者的一份实战避坑手册。

[English](README.en.md) | 中文

---

## 这是什么

一份 **DSH agent skill**（技能），让 AI 在制作、修改、审查 DeepSeek Harness 的**插件 / 钩子 / Skill** 时，自动遵循官方契约、避开已知会搞崩桌面端的坑。

它不是泛泛的介绍文，而是**把官方规范 + 真实事故揉在一起的操作手册** —— 机制部分严格对齐官方文档，每一条"坑"都对应一次真实的启动失败。

## 为什么需要它

DSH 的设计哲学是 **「Everything is a Plugin」**，插件生态非常开放。但开放也意味着：

- **组合包（bundle）和 profile 是两种不同的 manifest** —— 分不清就写不出能装的包
- 插件声明的是 **精确版本白名单**（不是语义化范围），宿主一升级插件就"异常"
- 有状态核心包写进 `dependencies` 会**遮蔽宿主**，直接让桌面端打不开
- `inject` 依赖的服务没启用，插件会永远停在 **PENDING**，启动卡死
- 一行 **BOM 字节**就能让 `JSON.parse` 崩掉整个启动流程
- **从 git 安装拉的是源码不是构建产物** —— 没有 `prepare` 脚本就加载失败

AI 没有这些知识时，很容易做出"看起来对、装上去崩"的插件。

## 安装

### 方式一：克隆到全局 skills 目录（推荐）

```bash
git clone https://github.com/wula1223/dsh-plugin-dev.git \
  ~/.dsh/skills/dsh-plugin-dev
```

Windows PowerShell：

```powershell
git clone https://github.com/wula1223/dsh-plugin-dev.git `
  "$env:USERPROFILE\.dsh\skills\dsh-plugin-dev"
```

### 方式二：只复制 SKILL.md

```powershell
$dest = "$env:USERPROFILE\.dsh\skills\dsh-plugin-dev"
New-Item -ItemType Directory -Force -Path $dest | Out-Null
Copy-Item .\SKILL.md $dest
```

**不需要重启 DSH。** 官方的本地 skill 提供方带文件监视（chokidar），会热刷新技能目录 —— 装好后新技能会自己出现在可用列表里。

> 方式二只复制主文件，`references/` 里的参考页不会被带上；建议用方式一。

## 内容结构

主文件是**薄路由**（约 18 KB），详细内容按需加载：

| 文件 | 内容 |
|---|---|
| [`SKILL.md`](SKILL.md) | **技能本体**：内容来源分级 · 路由表 · 权威来源 · 插件契约 · Skill 格式 · 验证清单 · 发布 · 排查 · 汇报原则 |
| [`references/capability-map.md`](references/capability-map.md) | `ctx` 键总表 + 「新行为归属」映射表 + seam 三角色 |
| [`references/events.md`](references/events.md) | 五种分发模式 · 三大事件域 · Claude Code 术语对照 · `hooks.json` 桥 |
| [`references/lifecycle.md`](references/lifecycle.md) | Fiber 状态机 · 自动清理 · dispose · HMR |
| [`references/pitfalls.md`](references/pitfalls.md) | 六个必踩的坑（含真实崩溃日志） |
| [`references/ecosystem.md`](references/ecosystem.md) | MCP / hooks 桥 / 记忆 / 输出样式 / 插件市场 + 官方包谱系 |
| [`references/official-docs-index.md`](references/official-docs-index.md) | 官方文档地图 + skill 规范 + 官方 skill 样板 |

> 这个「薄主文件 + 参考页」的分法，是照着**官方随包 skill `cordis-plugin-development`** 的结构做的（它约 8 KB + `references/` + `templates/`）。

## 内容来源分级

本技能**不发明规则**，全文区分三类来源：

| 标记 | 含义 |
|---|---|
| 🟢 **官方** | 出自官方文档站 / 官方仓库 / 官方随附实物，可找到原文 |
| 🟡 **实测** | 本机真实事故中验证出的**事实**，有崩溃日志佐证，但**官方未明文规定** |
| 🔵 **社区** | 社区插件，名字已在 npm 核实，但**非官方** |

⚠️ **不要把 🟡 / 🔵 当成官方规定引用。**

## 六大致命坑

| # | 坑 | 后果 |
|---|---|---|
| 1 | 版本白名单没跟上宿主升级 | 插件显示「异常」 |
| 2 | 有状态核心包写进 `dependencies` | 影子包遮蔽宿主 → **桌面端打不开** |
| 3 | `inject` 的服务没有提供者 | 插件永远 **PENDING** → 启动失败 |
| 4 | JSON 带 BOM 头 | `JSON.parse` 失败 → 启动崩溃 |
| 5 | 改配置时 DSH 还在运行 | 改动被并发覆盖 |
| 6 | 从 git 安装却没管构建 | 包到手没有 `lib/` → 加载失败 |

每一条在 `references/pitfalls.md` 里都有**真实事故记录、崩溃日志原文和修复方法**。

## 适用对象

- **AI agent**：在动手做 DSH 插件/skill 前自动加载本技能
- **插件作者**：当作发布前的自检清单
- **DSH 用户**：桌面端打不开时，按排查流程走一遍

## 依据的官方来源

- 官方文档站：<https://deepseek-harness.github.io/deepseek-harness/>
- 官方仓库：<https://github.com/deepseek-ai/deepseek-harness>
- **官方 skill 规范**：`docs/subsystems/skills.md`
- **官方随包 skill 样板**：`packages/preset/agent-preset/skills/`
- 本机随附的官方 skill 实物：`resources/runtime/office-skills/`

## 兼容性

> ⚠️ 本技能是**文档型技能**，不含任何可执行插件代码，因此不受版本白名单约束 —— 任何 DSH 版本都能安全加载。

其中的「坑」来自本机 DSH 桌面端上真实复现的启动故障，修复后已验证。**具体版本号与 npm 上的发行版本未必一一对应**，请以现象和崩溃日志为准，而不是版本号。

## 贡献

发现新的坑？欢迎提 PR。请附上：

1. 现象与完整的崩溃日志
2. 触发条件（DSH 版本 / 插件版本 / 操作步骤）
3. 根因分析
4. 修复方法

## 许可

[MIT](LICENSE)