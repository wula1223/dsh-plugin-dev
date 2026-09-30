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

装好后重启 DSH，技能就会出现在可用列表里。

## 内容

| 文件 | 内容 |
|---|---|
| [`SKILL.md`](SKILL.md) | **技能本体**（662 行）。官方契约 + 能力地图 + 生态地图 + 六大坑 + 验证清单 + 发布规范 |
| [`references/official-docs-index.md`](references/official-docs-index.md) | **官方文档地图**：文档站 + 仓库 docs 的完整索引 |

### SKILL.md 覆盖的内容

| # | 章节 | 要点 |
|---|---|---|
| 0 | **权威来源** | 官方文档站（含每页 `.md` 取法）+ 仓库 docs + 本机实物 |
| 1 | **插件契约** | 三种插件形态 · **组合包 vs profile** · 组合包 manifest · **四层加载顺序** · cordis 五概念 · **`Config` + Schemastery** |
| 1.7 | **能力地图 ⭐** | **`ctx` 键总表** · **「新行为归属」20 条映射表** · seam 三角色 |
| 2 | **钩子 / 事件** | 五种分发模式精确语义 · waterfall 必须调 `next()` · TS 声明合并 · **事件三大域** · ⚠️ 会话事件 vs Cordis 事件 · **Claude Code 术语对照表** |
| 3 | **生命周期** | **Fiber 状态机**（PENDING/LOADING/ACTIVE/FAILED/…）· 自动清理 · ⚠️ 处置器逆序但并发 · dispose · HMR |
| 4 | **Skill 格式** | `SKILL.md` frontmatter 与 `description` 触发条件写法 |
| 5 | **⚠️ 六大坑** | 逐条附真实崩溃日志与修复 |
| 6 | **本地验证清单** | 可直接复制的 PowerShell + `dsh --dump-config` |
| 7 | **发布 / 上架** | 仓库结构 · 自检清单 · Topics · git 推送的环境坑 |
| 8 | **排查流程** | 桌面端打不开时的 7 步定位 |
| 9 | **汇报原则** | 必须给证据，不能只说"应该好了" |
| 10 | **生态地图** | MCP（官方 `dsh-mcp-client`）/ 记忆 / 技能管理 / Git 凭据 / 多代理协作 |

## 六大致命坑

| # | 坑 | 后果 |
|---|---|---|
| 1 | 版本白名单没跟上宿主升级 | 插件显示「异常」 |
| 2 | 有状态核心包写进 `dependencies` | 影子包遮蔽宿主 → **桌面端打不开** |
| 3 | `inject` 的服务没有提供者 | 插件永远 **PENDING** → 启动失败 |
| 4 | JSON 带 BOM 头 | `JSON.parse` 失败 → 启动崩溃 |
| 5 | 改配置时 DSH 还在运行 | 改动被并发覆盖 |
| 6 | 从 git 安装却没管构建 | 包到手没有 `lib/` → 加载失败 |

每一条在 `SKILL.md` 里都有**真实事故记录、崩溃日志原文和修复方法**。

## 适用对象

- **AI agent**：在动手做 DSH 插件/skill 前自动加载本技能
- **插件作者**：当作发布前的自检清单
- **DSH 用户**：桌面端打不开时，按排查流程走一遍

## 依据的官方来源

本技能内容对齐以下官方来源：

- 官方文档站：<https://deepseek-harness.github.io/deepseek-harness/>
- 官方仓库：<https://github.com/deepseek-ai/deepseek-harness>
- 本机随附的官方 skill 实物：`resources/runtime/office-skills/`

## 兼容性

| DSH 版本 | 状态 |
|---|---|
| `0.2.0-rc.2` | ✅ 已验证（全部六个坑都在该版本上复现并修复） |
| `0.1.7-rc.2` | ✅ 适用（版本白名单一节即来自该版本升级过程） |

> ⚠️ 本技能是**文档型技能**，不含任何可执行插件代码，因此不受版本白名单约束 —— 任何 DSH 版本都能安全加载。

## 贡献

发现新的坑？欢迎提 PR。请附上：

1. 现象与完整的崩溃日志
2. 触发条件（DSH 版本 / 插件版本 / 操作步骤）
3. 根因分析
4. 修复方法

## 许可

[MIT](LICENSE)
