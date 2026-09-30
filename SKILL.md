---
name: dsh-plugin-dev
description: Use when building, modifying, reviewing, or debugging a DeepSeek Harness (DSH) plugin, hook, bundle, profile, or agent skill — including packaging and publishing one, and including any DSH plugin that shows as error, stays pending, or makes the desktop app fail to start.
---

# DSH 插件 / Skill 开发规范

DeepSeek Harness（DSH）的设计哲学是 **「Everything is a Plugin」**。插件、工具、UI、LLM 适配器全部走同一套 cordis 插件模型。

**动手前先读本文，尤其第 5 节（六个必踩的坑）—— 它们会直接搞崩用户的桌面端。**

---

## 📋 内容来源分级（先读这一节）

**本技能不发明规则。** 全文区分「官方规定」与「非官方经验」，请勿混用：

| 标记 | 含义 | 可否依赖 |
|---|---|---|
| 🟢 **官方** | 出自官方文档站 / 官方仓库 / 官方随附实物，可找到原文 | ✅ 权威 |
| 🟡 **实测** | 本机真实事故中验证出来的**事实**，有崩溃日志佐证，但**官方未明文规定** | ⚠️ 现实如此，但可能随版本变化 |
| 🔵 **社区** | 社区插件；名字与描述**已在 npm 核实**，但**非官方** | ⚠️ 第三方，自行判断 |

| 位置 | 来源 |
|---|---|
| 本文件 §0 权威来源 | 🟢 官方 |
| 本文件 §1 插件契约 | 🟢 官方 · `docs/user/develop/basic/publish.md`、`docs/reference/` |
| `references/capability-map.md` | 🟢 官方 · `docs/reference/` 的 ctx 键表与归属映射 |
| `references/events.md` | 🟢 官方 · `events.md`；`hooks.json` 桥为 🟢 官方包 |
| `references/lifecycle.md` | 🟢 官方 · `framework/index.md` |
| 本文件 §2 Skill 格式 | 🟢 官方 · `office-skills/SKILL.md` 实物 |
| `references/pitfalls.md` | 🟡 **实测** —— 其中两条背后有 🟢 官方规则支撑，已逐条注明 |
| 本文件 §3 验证清单 | 🟡 实测 + 🟢 官方命令 |
| 本文件 §4 发布 / 上架 | 🟢 官方 · `publish.md`；环境坑为 🟡 实测 |
| 本文件 §5 排查流程 | 🟡 实测 |
| 本文件 §6 汇报原则 | 🟡 实测 |
| `references/ecosystem.md` | 🔵 社区（名字已核实）+ 🟢 官方包谱系 |

> ⚠️ **不要把 🟡 / 🔵 当成官方规定引用。** 尤其 `references/ecosystem.md` 的社区插件名 —— 它们是第三方作品，官方文档里没有它们。

## 📂 详细内容在哪（按需加载）

**本文件是薄路由。** 下面是完整清单 —— **不要枚举 `references/` 目录**，按需读取即可：

| 你要做的事 | 读这个文件 |
|---|---|
| 搞清楚「我要加的功能该挂在哪」（ctx 键 / 新行为归属 / seam 三角色） | `references/capability-map.md` |
| 加钩子、监听事件、理解五种分发模式与三大事件域 | `references/events.md` |
| 排查插件卡在 PENDING、不加载、卸载不干净 | `references/lifecycle.md` |
| 避开会搞崩桌面端的坑（附真实崩溃日志） | `references/pitfalls.md` |
| 选插件：MCP / hooks 桥 / 记忆 / 输出样式 / 插件市场 | `references/ecosystem.md` |
| 找官方文档原文、官方 skill 样板 | `references/official-docs-index.md` |

## 0. 权威来源（先读官方，再动手）

### 🌐 官方文档站（首选，最完整）

**https://deepseek-harness.github.io/deepseek-harness/**

| 页面 | 路径 | 内容 |
|---|---|---|
| 第一个插件 | `/develop/basic/` | `apply(ctx)` 最小插件、三种插件形态 |
| 开发一个 Tool | `/develop/basic/tool` | 工具定义 DSL |
| 插件配置 | `/develop/basic/config` | `Config` 类型 + Schemastery schema |
| **打包与安装插件** | `/develop/basic/publish` | **组合包 manifest、四层加载顺序、git 安装的构建坎** |
| 插件与生命周期 | `/develop/framework/` | **Fiber 状态机**、自动清理、HMR |
| 服务与依赖 | `/develop/framework/service` | 对外提供服务 |
| 事件系统 | `/develop/framework/events` | 五种分发模式、类型安全、命名约定 |
| 能力的三层拆分 | `/develop/practice/` | 实战分层 |
| LLM 适配器 | `/develop/practice/llm-adapter` | 完整 LLM 后端 |
| 持久化插件 | `/develop/practice/dynamic-cordis` | 动态持久化 |
| Cordis 教程 | `/develop/cordis-tutorial/` | 7 章手把手 |
| 参考 | `/reference/` | 子系统、配置、工具目录 |

> 💡 每页可加 `.md` 取原始 Markdown（如 `/develop/basic/publish.md`），比抓 HTML 干净。
> 中文站对应英文站：把 `/deepseek-harness/` 后加 `en/`，如 `/deepseek-harness/en/develop/basic/`。

### 📦 官方仓库

`github.com/deepseek-ai/deepseek-harness` 的 `docs/` 目录（站点源码在 `docs/user/`）：

| 文档 | 大小 | 内容 |
|---|---|---|
| `docs/cordis-primer.zh.md` | 4 KB | Cordis 五个核心概念 |
| `docs/capability-seams.zh.md` | 61 KB | 全部扩展点 |
| `docs/config-catalog.zh.md` | 210 KB | 配置项权威目录 |
| `docs/tool-catalog.zh.md` | 98 KB | 工具定义目录 |
| `docs/cookbook/extension-cookbook.zh.md` | 11 KB | 扩展点实战 |
| `docs/upgrade-guide/` | — | 版本升级指南 |
| `.agents/skills/` | 16 个 | 官方 skill 范例 |

### 💻 本机官方实物（可直接读）

```
D:\DeepSeek-Harness\resources\runtime\office-skills\office-docx\SKILL.md
D:\DeepSeek-Harness\resources\runtime\office-skills\office-pptx\SKILL.md
D:\DeepSeek-Harness\resources\runtime\office-skills\office-xlsx\SKILL.md
```

### 用户侧关键路径

```
C:\Users\hp\.dsh\profiles\<profile>\package.json      ← 依赖 + dsh.profile.bundles（启用清单）
C:\Users\hp\.dsh\profiles\<profile>\cordis.patch.yml  ← 用户的 patch 层（可改）
C:\Users\hp\.dsh\profiles\<profile>\cordis.yml        ← 自动生成，禁止手改
C:\Users\hp\.dsh\profiles\<profile>\.dsh-market\state.json  ← 市场禁用/分组状态
C:\Users\hp\.dsh\profiles\<profile>\.dsh-market\log.ndjson  ← 市场操作日志（排查必看）
C:\Users\hp\.dsh\skills\                              ← 全局 skill 目录
C:\Users\hp\AppData\Roaming\@deepseek-ai\dsh-desktop\logs\  ← 崩溃日志
```

---

## 1. 插件契约

### 1.1 三种插件形态

```ts
// ① 函数形式（最常用）
import type { Context } from '@deepseek-ai/cordis'
export const name = 'my-plugin'
export function apply(ctx: Context) { /* 注册能力 */ }

// ② 对象形式
export default {
  name: 'my-plugin',
  inject: ['tools'],
  apply(ctx: Context) { /* ... */ },
}

// ③ 类形式（需要向其他插件提供服务时用）
import { Service } from '@deepseek-ai/cordis'
export default class MyService extends Service {
  static inject = ['tools']
  constructor(ctx: Context) { super(ctx, 'myService') }   // 构造函数内做同步初始化
}
```

### 1.2 组合包 vs profile —— 两种 manifest

**这是最容易搞混的地方。** 两者都由 `package.json` 描述，但 `dsh` 键下的 manifest 种类不同：

| | **组合包 (bundle)** | **profile** |
|---|---|---|
| manifest 键 | `dsh.bundle` | `dsh.profile` |
| 回答的问题 | **"这个包贡献什么？"** | **"这套配置由哪些组合包按什么顺序组成？"** |
| 内容 | 一个 patch 文件 | 有序的 `bundles` 列表 |
| 谁写 | **插件作者**（你） | 用户（`dsh plugin` 自动维护） |
| 位置 | npm 包 | `$DSH_HOME/profiles/<name>` |

**没有东西同时是两者。**

### 1.3 组合包 manifest（插件作者写这个）

目录结构：

```
hello-plugin/
├── package.json       # 声明 dsh.bundle
├── cordis.patch.yml   # 被 profile 列出时应用的层
└── index.js           # patch 行引用的插件模块
```

`package.json`：

```json
{
  "name": "dsh-hello-plugin",
  "version": "0.1.0",
  "type": "module",
  "main": "index.js",
  "files": ["index.js", "cordis.patch.yml"],
  "dsh": { "bundle": { "patch": "./cordis.patch.yml" } }
}
```

`cordis.patch.yml`：

```yaml
- insert:
    - id: hello
      name: dsh-hello-plugin      # 按【包名】引用，Node 模块解析才能找到
```

要点：
- `patch` 也接受**有序文件列表**：`["./base.patch.yml", "./web.patch.yml"]`，按序作为同一层应用，各文件内相对插件路径**相对于该文件**解析
- **没有 `dsh.bundle` 声明的包仍可安装**，但只作为普通依赖 —— `dsh plugin` 会打印警告且**不激活任何层**（供插件包 import 的库用这种格式）

### 1.4 四层加载顺序 ⭐

生效配置在空根之上按以下顺序逐层组合：

```
1. profile 的 dsh.profile.bundles 所列各组合包 patch（按列表顺序，先 @deepseek-ai/dsh-base）
2. profile 自己的 cordis.patch.yml
3. home 级 $DSH_HOME/cordis.patch.yml（各 profile 共享的机器本地偏好）
4. 每个 --patch <path> overlay（按 argv 顺序）
```

**后应用的层按行胜出。** 两条关键推论（写给组合包作者）：

- 你的 patch 可以**按 `id` 覆盖前面层的行**，但**必须重述该行需要的每一个键** —— 因为
- **patch 会替换目标行的整个 `config` 值，而不是深度合并各键。**

> 所以：**优先给出用户大概率会保留的配置默认值**，其余交给 schema 承担。

内置组合包名（如 `@deepseek-ai/dsh-base`）始终从 dsh 安装目录解析，pnpm 只管理树外包 —— 可以放心依赖它存在。

### 1.5 cordis 插件模型（五个核心概念）

- **插件 = 实现 Service 的对象**：带可选 `inject` 和 `apply(ctx)` 的函数，或 `Service` 子类
- **上下文是服务容器**：服务占稳定 key（`ctx.tools`、`ctx.llm`、`ctx.sessions`…），按 key 查找而非导入实现
- **`inject` 声明服务依赖**：声明后**等依赖就绪才启动**；加载顺序由依赖表达
- **类型化事件通信**：`emit` / `waterfall` / `parallel` / `serial` / `bail`
- **注册是可逆副作用**：用 `ctx.effect()` 或 `ctx.on()` 安装，reload/teardown 时自动撤销

### 1.6 插件配置：`Config` 类型 + Schemastery schema

```ts
import type { Context } from '@deepseek-ai/cordis'
import Schema from '@deepseek-ai/schemastery'

export const name = 'my-plugin'

export interface Config {
  greeting: string
  maxRetries: number
  verbose?: boolean
}

export const Config: Schema<Config> = Schema.object({
  greeting: Schema.string().default('Hello'),
  maxRetries: Schema.number().default(3),
  verbose: Schema.boolean().default(false),
})

export function apply(ctx: Context, config: Config) {
  console.log(config.greeting)   // 用户值或 schema 默认值
}
```

- ⚠️ **不要导出普通对象作为 `Config`** —— 它不满足 Cordis 要求的 Standard Schema 接口
- 校验在**插件加载时**执行；配置不合法则插件加载失败并给出明确错误
- **设计原则：无硬编码可调参数。** 凡是不同部署可能取不同值的参数，都必须定义为配置字段
  检验标准：**能否在 `cordis.yml` 中改变这个值，而不需要修改代码？**
- **配置错误要响亮**：在 schema 中表达自身完备的约束，让无效配置在加载时失败
- 配置变更会触发**热替换**（卸载旧实例 → 加载新实例）

在 `cordis.yml` 中传配置：

```yaml
- insert:
    - id: hello
      name: './src/my-plugin.ts'
      config:
        greeting: 'Hi there'
        maxRetries: 5
```

---


## 2. Skill 格式

`SKILL.md` = YAML frontmatter + Markdown 正文：

```markdown
---
name: my-skill
description: 一句话说明能力 + **何时触发**（"Use when…"）。路由靠这句匹配，必须写清触发条件。
---

# my-skill

正文：可执行的工作流、代码示例、检查步骤、交付方式。
```

要点：
- `description` 必须包含**触发条件**，否则不会被正确路由
- 正文写**可执行的工作流**，不是泛泛介绍
- 全局 skill 放 `C:\Users\hp\.dsh\skills\<name>\SKILL.md`
- 项目级放 `<workspace>\.dsh\skills\`

---


## 3. 本地验证清单（每次改完必做）

### 6.1 停 DSH → 改配置 → 校验 JSON

```powershell
$prof = "$env:USERPROFILE\.dsh\profiles\desktop"
Get-Process | Where-Object { $_.ProcessName -match 'DeepSeek|Harness' } | Stop-Process -Force
Start-Sleep -Seconds 5
try { $null = Get-Content "$prof\package.json" -Raw -Encoding UTF8 | ConvertFrom-Json; "JSON OK" }
catch { "!! JSON 坏了: $_" }
```

### 6.2 影子包检查（关键）

```powershell
$prof = "$env:USERPROFILE\.dsh\profiles\desktop"
Get-ChildItem "$prof\node_modules\@deepseek-ai" -Directory -ErrorAction SilentlyContinue |
  ForEach-Object {
    $v = try { (Get-Content (Join-Path $_.FullName 'package.json') -Raw -Encoding UTF8 | ConvertFrom-Json).version } catch { '?' }
    "  $($_.Name) = $v"
  }
# 期望：只有 cosmokit / schemastery 这类无状态工具包
# 出现 dsh-llm / dsh-skill / dsh-tools / dsh-session 等有状态核心包 → 立刻移除！
```

### 6.3 不启动，先验证配置层

```powershell
dsh --profile demo --dump-config      # 显示的层里应能找到 "# == <你的包名>" 那一节
```

### 6.4 重启并确认没有新崩溃

```powershell
$logdir = "C:\Users\hp\AppData\Roaming\@deepseek-ai\dsh-desktop\logs"
$before = @(Get-ChildItem $logdir -Filter "crash-*.log").Count
Start-Process "D:\DeepSeek-Harness\DeepSeek Harness.exe"
Start-Sleep -Seconds 50
$after = @(Get-ChildItem $logdir -Filter "crash-*.log").Count
"崩溃日志: $before -> $after"
if ($after -gt $before) {
  Get-ChildItem $logdir -Filter "crash-*.log" | Sort-Object LastWriteTime -Descending |
    Select-Object -First 1 | ForEach-Object { Get-Content $_.FullName -Raw -Encoding UTF8 }
}
```

### 6.5 崩溃日志读法

| 现象 | 含义 |
|---|---|
| `Plugins waiting for services (N)` + `xxx (required)` | 某服务没起来（查 5.1 / 5.2） |
| `pending (waiting for service: xxx)` | `inject` 依赖悬空（查 5.3，对应 Fiber PENDING） |
| `Unexpected token ''` | BOM 问题（查 5.4） |
| `apply` 抛异常的堆栈 | Fiber FAILED（查插件自身代码） |
| `duplicate factory registration for "xxx"` | 插件被加载两次，重启通常可清 |

### 6.6 安装 / 升级 / 移除插件

```powershell
# 官方方式（推荐）
dsh plugin --profile demo add ./hello-plugin          # 本地目录
dsh plugin --profile demo add github:you/hello-plugin # 从 git
dsh plugin --profile demo add your-package            # 从 npm
dsh plugin --profile demo remove dsh-hello-plugin     # 同时移除依赖和层

# 桌面端用的是 pnpm 直连方式
$prof = "$env:USERPROFILE\.dsh\profiles\desktop"
$rt   = "$env:USERPROFILE\.dsh\dsh-runtimes\dsh-primary-runtime\dependencies"
$node = (Get-ChildItem $rt -Recurse -Filter node.exe | Select-Object -First 1).FullName
$pnpm = (Get-ChildItem $rt -Recurse -Filter pnpm.mjs | Select-Object -First 1).FullName
& $node $pnpm add "包名@版本" --dir $prof
& $node $pnpm install --dir $prof
```

**手动启用 = 两处同时改**：加进 `dsh.profile.bundles` **且** 从 `.dsh-market\state.json` 的 `disabled` 移除。

---

## 4. 发布到 GitHub / 上架

### 7.1 仓库结构（组合包）

```
dsh-my-plugin/
├── README.md          # 中文说明：能力、安装、兼容性表
├── README.en.md       # 英文版
├── LICENSE            # MIT / Apache-2.0
├── package.json       # 含 dsh.bundle 字段
├── cordis.patch.yml   # 挂载层
├── index.js / lib/    # 入口
└── SKILL.md           # 若同时是 skill
```

### 7.2 发布前自检

- [ ] `dsh.bundle.patch` 指向的文件存在且语法正确
- [ ] `cordis.patch.yml` 里插件行**按包名**引用
- [ ] **有状态核心包**在 `peerDependencies` + `devDependencies`，**不在** `dependencies`
- [ ] `peerDependencies` 白名单覆盖**当前 + 相邻**宿主版本
- [ ] 若走 git 安装：提供自包含的 `prepare` 脚本，并在 README 里说明 `allowBuilds` 授权
- [ ] README 写明**支持的 DSH 版本范围**
- [ ] 本地按第 6 节完整验证过
- [ ] `files` 字段包含所有运行时需要的文件

### 7.3 上架到社区

- GitHub topic 加 **`dsh-plugin`**（市场爬虫靠这个收录）
- 相关 topic：`deepseek-harness`、`dsh`、`cordis`
- 可提 PR 到 `awesome-dsh-plugin` / `awesome-deepseek-harness` / `dshfind`

### 7.4 用 git 推送时的环境坑

- **`GIT_CONFIG_GLOBAL` 可能被改**：DSH 会把它指向 `$DSH_HOME/git-forge/gitconfig`，导致 `~/.gitconfig` 的配置（如代理重写）**不生效**。排查时用 `git config --list --show-origin`。
- **凭据助手可能挂起**：`dsh-git-forge` 助手在账号库为空时会让 `git push` / `ls-remote` **无限等待**。测试时设 `GIT_TERMINAL_PROMPT=0` 让它快速失败。
- **代理重写会劫持推送**：若 `.gitconfig` 里有 `url.<proxy>.insteadOf = https://github.com/`，推送会被改写到只读镜像而失败。用 `https://github.com:443/...`（端口写法不匹配前缀）可绕开。

---

## 5. 排查流程（插件异常 / 桌面端打不开）

1. **读最新崩溃日志**（第 6.5 节表格对照）
2. **比对插件白名单与宿主版本**（5.1）
3. **查影子包**（5.2 / 6.2）
4. **查 `inject` 依赖是否悬空**（5.3，对照 Fiber PENDING）
5. **查 BOM**（5.4）
6. **看 `.dsh-market\log.ndjson`** 找 `install-compat` 警告与 toggle 记录
7. 全都不行 → 用 DSH 自带的 **「禁用第三方插件、备份 profile patch 并重启」** 安全模式恢复

---

## 6. 汇报原则

改完必须给出**证据**，不能只说"应该好了"：

- 文件版本号 / 时间戳 / 哈希
- 影子包扫描结果
- 崩溃日志计数 `before -> after`
- 进程存活时长

用户曾因"报告启动成功但实际没起来"而多折腾一轮 —— **验证不到位的代价很高**。

---