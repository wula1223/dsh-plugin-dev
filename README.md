# dsh-plugin-dev

> **DeepSeek Harness 插件 / 钩子 / Skill 开发规范** —— 给 AI agent 和人类开发者的一份实战避坑手册。

[English](README.en.md) | 中文

---

## 这是什么

一份 **DSH agent skill**（技能），让 AI 在制作、修改、审查 DeepSeek Harness 的**插件 / 钩子 / Skill** 时，自动遵循官方契约、避开已知会搞崩桌面端的坑。

它不是泛泛的介绍文，而是**从真实事故里总结出来的操作手册** —— 每一条"坑"都对应一次真实的启动失败。

## 为什么需要它

DSH 的设计哲学是 **「Everything is a Plugin」**，插件生态非常开放。但开放也意味着：

- 插件声明的是 **精确版本白名单**（不是语义化范围），宿主一升级插件就"异常"
- 第三方插件把核心包写进 `dependencies` 会**遮蔽宿主**，直接让桌面端打不开
- `inject` 依赖的服务没启用，插件会永远 `pending`，启动卡死
- 一行 **BOM 字节**就能让 `JSON.parse` 崩掉整个启动流程

AI 没有这些知识时，很容易做出"看起来对、装上去崩"的插件。

## 安装

### 方式一：克隆到全局 skills 目录（推荐）

```bash
git clone https://github.com/<你的用户名>/dsh-plugin-dev.git \
  ~/.dsh/skills/dsh-plugin-dev
```

Windows PowerShell：

```powershell
git clone https://github.com/<你的用户名>/dsh-plugin-dev.git `
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
| [`SKILL.md`](SKILL.md) | **技能本体**。官方契约 + 五大致命坑 + 本地验证清单 + 发布 checklist |
| [`references/official-docs-index.md`](references/official-docs-index.md) | **官方文档地图**：DSH 官方仓库里所有规范文档的位置与用途 |

### SKILL.md 覆盖的内容

1. **权威来源** —— 官方文档在哪、本机路径在哪
2. **插件契约** —— `package.json` 的 `dsh` 字段、`dsh.bundle.patch`、`cordis.patch.yml`
3. **cordis 插件模型** —— Service / Context / `inject` / `ctx.effect()`
4. **钩子与事件** —— 五种分发模式（`emit` / `waterfall` / `parallel` / `serial` / `bail`）及 waterfall 语义
5. **Skill 格式** —— `SKILL.md` frontmatter 规范与 `description` 触发条件写法
6. **⚠️ 五大致命坑** —— 版本白名单、影子包、依赖悬空、BOM、profile 并发写入
7. **本地验证清单** —— 可直接复制运行的 PowerShell 脚本
8. **发布 checklist** —— 发到 GitHub / 上架社区前的自检项

## 五大致命坑（速览）

| # | 坑 | 后果 |
|---|---|---|
| 1 | 版本白名单没跟上宿主升级 | 插件显示「异常」 |
| 2 | 核心包写进 `dependencies` | 影子包遮蔽宿主 → **桌面端打不开** |
| 3 | `inject` 的服务没有提供者 | 插件永远 `pending` → 启动失败 |
| 4 | JSON 带 BOM 头 | `JSON.parse` 失败 → 启动崩溃 |
| 5 | 改配置时 DSH 还在运行 | 改动被并发覆盖 |

每一条在 `SKILL.md` 里都有**真实事故记录、崩溃日志原文和修复方法**。

## 适用对象

- **AI agent**：在动手做 DSH 插件/skill 前自动加载本技能
- **插件作者**：当作发布前的自检清单
- **DSH 用户**：桌面端打不开时，按第 7 节排查流程走一遍

## 兼容性

| DSH 版本 | 状态 |
|---|---|
| `0.2.0-rc.2` | ✅ 已验证（本技能的全部坑都在该版本上复现并修复） |
| `0.1.7-rc.2` | ✅ 适用（版本白名单一节即来自该版本升级到 rc.2 的过程） |

> ⚠️ 本技能是**文档型技能**，不含任何可执行插件代码，因此不受版本白名单约束 —— 任何 DSH 版本都能安全加载。

## 贡献

发现新的坑？欢迎提 PR。请附上：

1. 现象与完整的崩溃日志
2. 触发条件（DSH 版本 / 插件版本 / 操作步骤）
3. 根因分析
4. 修复方法

## 许可

[MIT](LICENSE)
