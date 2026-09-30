---
name: dsh-plugin-dev
description: 制作、修改或审查 DeepSeek Harness (DSH) 插件 / 钩子 / Skill 时必读。涵盖官方 cordis bundle 契约、package.json 的 dsh 字段、事件分发模式（emit/waterfall/parallel/serial/bail）、SKILL.md 格式、版本兼容白名单纪律，以及影子包 / 依赖悬空 / BOM / profile 并发写入等实战避坑与本地验证流程。当用户要求开发 DSH 插件、加钩子、写 skill，或排查插件显示「异常 / 未运行 / 启动失败 / 桌面端打不开」时使用。
---

# DSH 插件 / Skill 开发规范

DeepSeek Harness（DSH）的设计哲学是 **「Everything is a Plugin」**。插件、工具、UI、LLM 适配器全部走同一套 cordis 插件模型。

**动手前先读本文，尤其第 4 节（五个必踩的坑）—— 它们会直接搞崩用户的桌面端。**

---

## 0. 权威来源（先读官方，再动手）

| 资源 | 位置 |
|---|---|
| 官方仓库 | `github.com/deepseek-ai/deepseek-harness` |
| 插件入门 | `docs/cordis-primer.zh.md` |
| 插件教程（8 章） | `docs/cordis-tutorial/` |
| 扩展点全集 | `docs/capability-seams.zh.md`（61 KB） |
| 配置项目录 | `docs/config-catalog.zh.md`（210 KB） |
| 工具定义目录 | `docs/tool-catalog.zh.md`（98 KB） |
| 实战配方 | `docs/cookbook/`（`extension-cookbook.zh.md` 最实用） |
| 升级指南 | `docs/upgrade-guide/` |
| 官方 skill 范例 | `.agents/skills/`（16 个） |

本机也有官方实物（可直接读）：

```
D:\DeepSeek-Harness\resources\runtime\office-skills\office-docx\SKILL.md
D:\DeepSeek-Harness\resources\runtime\office-skills\office-pptx\SKILL.md
D:\DeepSeek-Harness\resources\runtime\office-skills\office-xlsx\SKILL.md
```

用户侧关键路径：

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

### 1.1 manifest（`package.json` 的 `dsh` 字段）

```json
{
  "name": "my-dsh-plugin",
  "version": "1.0.0",
  "type": "module",
  "main": "lib/index.js",
  "exports": {
    ".": "./lib/index.js",
    "./client": "./lib/client.js",
    "./package.json": "./package.json"
  },
  "dsh": {
    "bundle": { "patch": "./cordis.patch.yml" },
    "client": {
      "platform": "web",
      "inject": ["@deepseek-ai/dsh-client-ui-slots", "@deepseek-ai/dsh-client-locale"]
    }
  }
}
```

- `dsh.bundle.patch` —— **必需**，指向插件自带的挂载层
- `dsh.client.platform` —— `web` 表示有前端部分
- `dsh.client.inject` —— 前端要注入的宿主模块

### 1.2 插件自带的 `cordis.patch.yml`

```yaml
- insert:
    - id: my-plugin
      name: 'my-dsh-plugin'
```

### 1.3 cordis 插件模型（五个核心概念）

- **插件 = 实现 Service 的对象**：带可选 `inject` 和 `apply(ctx)` 的函数，或 `Service` 子类
- **上下文是服务容器**：服务占稳定 key（`ctx.tools`、`ctx.llm`、`ctx.sessions`…），按 key 查找而非导入实现
- **`inject` 声明服务依赖**：声明后**等依赖就绪才启动**；加载顺序由依赖表达
- **类型化事件通信**：`emit` / `waterfall` / `parallel` / `serial` / `bail`
- **注册是可逆副作用**：用 `ctx.effect()` 或 `ctx.on()` 安装，reload/teardown 时自动撤销

```js
export const name = 'my-plugin'
export const inject = ['tools']        // 等 ctx.tools 就绪
export function apply(ctx) {
  ctx.effect(() => { /* 安装… */ return () => { /* 撤销… */ } })
}
```

---

## 2. 钩子 / 事件

五种分发模式，**每种只能通过对应方法分发**：

| 模式 | 是否 await | 顺序 | 有返回值 | 用途 |
|---|---|---|---|---|
| `emit` | 否 | 注册顺序观察 | 否 | 纯通知 |
| **`waterfall`** | 否 | 注册顺序观察 | **是** | **环绕中间件（拦截/包装）** |
| `parallel` | **是** | 并行 | 否 | 并行扇出 |
| `serial` | **是** | 注册顺序 | 是 | 按序执行 |
| `bail` | 否 | 顺序直到返回 bail 值 | 是 | 短路决策 |

**waterfall 语义**：监听器收到 `(...args, next)`。调用 `next()` 执行下游；下游返回值经 `next()` 返回当前层，可包装后继续外传。**不调用 `next()` 直接返回 = 短路。** 仅需在普通注册之前运行时用 `prepend: true`。

**归属规则**：工具流水线事件 → `ctx.tools`；模型流式输出 → `ctx.llm`；agent 协调 → `ctx.agents`。
**拦截/策略优先用事件；直接能力调用优先用服务方法。**

---

## 3. Skill 格式

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

## 4. ⚠️ 五个必踩的坑（真实事故总结）

### 4.1 版本兼容白名单 —— 插件「异常」的头号原因

DSH 插件的兼容性声明是**精确版本白名单**，**不是语义化范围**：

```json
"peerDependencies": {
  "@deepseek-ai/dsh-skill": "0.1.2-rc.1 || 0.1.7-rc.2 || 0.2.0-rc.1"
}
```

宿主一升级（如 `0.1.7-rc.2` → `0.2.0-rc.2`），白名单没跟上 → 插件**自检失败 → 面板显示「异常」**。

**规则**：宿主升级后，所有插件必须同步升级。排查「异常」时第一步就是比对插件白名单与宿主版本。

### 4.2 核心包绝不能写进 `dependencies` —— 会遮蔽宿主

第三方插件把 `@deepseek-ai/dsh-*` 核心包写进 `dependencies` 并钉死版本，会把该版本装进 profile 的 `node_modules`，**遮蔽 app 自带版本**，导致服务起不来。

**真实事故**：`dsh-harness-zh-cn@0.1.2` 声明了 `"@deepseek-ai/dsh-llm": "0.0.1-rc.1"`，装完后 profile 里出现 `node_modules\@deepseek-ai\dsh-llm@0.0.1-rc.1`，遮蔽宿主 → `llm` 服务无法激活 → 8 个插件排队 → 必需的 `agent-loop` 起不来 → **桌面端彻底打不开**。

**规则**：
- 核心包一律用 `peerDependencies`，让宿主提供
- 装完**必查影子包**（见 5.2）

### 4.3 `inject` 依赖悬空 —— 启动卡死

插件 `inject` 了某个服务，但提供该服务的插件没启用 → 该插件永远 `pending` → **web boot 失败**。

**真实事故**：`@huanlin/dsh-plugin-better-sidebar-plugin-office` 依赖 `betterSidebar` 服务，但 `dsh-better-sidebar` 没启用 → 启动失败。日志：

```
web boot: 1 entry did not activate
@huanlin/...-office: pending (waiting for service: betterSidebar)
```

**规则**：成套的插件必须**成组启用**。启用前先确认它的 `inject` 服务有提供者。

### 4.4 BOM 头 —— 一行字节搞崩启动

用 PowerShell 的 `Set-Content -Encoding UTF8` 写 profile 的 JSON，**Windows PowerShell 5.1 会写入 BOM**（`EF BB BF`）。DSH 用 `JSON.parse` 读清单 → 第一个字符非法：

```
SyntaxError: Unexpected token '', "{ "n"... is not valid JSON
    at readProfileManifest (…/dsh-app-boot/lib/index.js:835:22)
```

**规则**：写 profile JSON 必须**无 BOM**：

```powershell
$utf8NoBom = New-Object System.Text.UTF8Encoding($false)
[System.IO.File]::WriteAllText($path, $json, $utf8NoBom)
# 写完校验
$b = [System.IO.File]::ReadAllBytes($path)
"BOM = $($b[0] -eq 0xEF -and $b[1] -eq 0xBB -and $b[2] -eq 0xBF)   # 必须 False"
```

### 4.5 profile 会被并发改写

DSH 自身/插件管理器会在启动和插件操作时重写 `package.json`、`cordis.yml`、`cordis.patch.yml`。

- **先停 DSH 再改配置**，否则改动会被覆盖
- 改完**核对文件时间戳**确认没被回写
- `cordis.yml` 是自动生成的空模板（约 223 B），**每次启动被重写属正常，不要手改**
- 排查"谁在改"看 `.dsh-market\log.ndjson`

---

## 5. 本地验证清单（每次改完必做）

### 5.1 停 DSH → 改配置 → 校验 JSON

```powershell
$prof = "$env:USERPROFILE\.dsh\profiles\desktop"
Get-Process | Where-Object { $_.ProcessName -match 'DeepSeek|Harness' } | Stop-Process -Force
Start-Sleep -Seconds 5
try { $null = Get-Content "$prof\package.json" -Raw -Encoding UTF8 | ConvertFrom-Json; "JSON OK" }
catch { "!! JSON 坏了: $_" }
```

### 5.2 影子包检查（关键）

```powershell
$prof = "$env:USERPROFILE\.dsh\profiles\desktop"
Get-ChildItem "$prof\node_modules\@deepseek-ai" -Directory -ErrorAction SilentlyContinue |
  ForEach-Object {
    $v = try { (Get-Content (Join-Path $_.FullName 'package.json') -Raw -Encoding UTF8 | ConvertFrom-Json).version } catch { '?' }
    "  $($_.Name) = $v"
  }
# 期望：只有 cosmokit / schemastery 这类工具包
# 出现 dsh-llm / dsh-skill / dsh-tools / dsh-session 等核心包 → 立刻移除！
```

### 5.3 重启并确认没有新崩溃

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

### 5.4 崩溃日志读法

| 现象 | 含义 |
|---|---|
| `Plugins waiting for services (N)` + `xxx (required)` | 某服务没起来（查 4.1 / 4.2） |
| `pending (waiting for service: xxx)` | `inject` 依赖悬空（查 4.3） |
| `Unexpected token ''` | BOM 问题（查 4.4） |
| `duplicate factory registration for "xxx"` | 插件被加载两次，重启通常可清 |

### 5.5 安装/升级插件

```powershell
$prof = "$env:USERPROFILE\.dsh\profiles\desktop"
$rt   = "$env:USERPROFILE\.dsh\dsh-runtimes\dsh-primary-runtime\dependencies"
$node = (Get-ChildItem $rt -Recurse -Filter node.exe | Select-Object -First 1).FullName
$pnpm = (Get-ChildItem $rt -Recurse -Filter pnpm.mjs | Select-Object -First 1).FullName
& $node $pnpm add "包名@版本" --dir $prof     # 升级
& $node $pnpm install --dir $prof             # 按清单安装
```

启用 = 加进 `dsh.profile.bundles` **且** 从 `.dsh-market\state.json` 的 `disabled` 移除（两处必须同时改）。

---

## 6. 发布到 GitHub

### 6.1 仓库结构建议

```
my-dsh-plugin/
├── README.md          # 中文说明：能力、安装、截图、兼容性表
├── README.en.md       # 英文版
├── LICENSE            # MIT / Apache-2.0
├── package.json       # 含 dsh 字段
├── cordis.patch.yml   # 挂载层
├── lib/               # 编译产物（main 指向这里）
├── src/               # 源码（可选）
└── SKILL.md           # 若同时是 skill
```

### 6.2 发布前自检

- [ ] `dsh.bundle.patch` 指向的文件存在且语法正确
- [ ] 核心包全在 `peerDependencies`，`dependencies` 里**没有** `@deepseek-ai/*`
- [ ] `peerDependencies` 白名单覆盖**当前 + 相邻**宿主版本
- [ ] README 写明**支持的 DSH 版本范围**（这是用户最关心的）
- [ ] 本地按第 5 节完整验证过：装 → 查影子包 → 重启 → 无崩溃
- [ ] `dsh.client.inject` 列出的宿主模块确实存在

### 6.3 上架到社区

- GitHub topic 加 **`dsh-plugin`**（市场会同步该 topic）
- 可提交 PR 到 `awesome-dsh-plugin`、`awesome-deepseek-harness`、`dshfind`
- 市场收录要求：`dsh.bundle` manifest + `cordis.patch.yml`

### 6.4 推送前确认凭据

```powershell
# DSH 的 GitForge 策略：accounts 为空时无法自动推送
# 需要用户自行配置 token，或手动 push
```

---

## 7. 排查流程（插件显示异常 / 桌面端打不开）

1. **读最新崩溃日志**（第 5.4 节表格对照）
2. **比对插件白名单与宿主版本**（4.1）
3. **查影子包**（4.2 / 5.2）
4. **查 `inject` 依赖是否悬空**（4.3）
5. **查 BOM**（4.4）
6. **看 `.dsh-market\log.ndjson`** 找 `install-compat` 警告与 toggle 记录
7. 全都不行 → 用 DSH 自带的 **「禁用第三方插件、备份 profile patch 并重启」** 安全模式恢复

---

## 8. 汇报原则

改完必须给出**证据**，不能只说"应该好了"：

- 文件版本号 / 时间戳
- 影子包扫描结果
- 崩溃日志计数 `before -> after`
- 进程存活时长

用户曾因"报告启动成功但实际没起来"而多折腾一轮 —— **验证不到位的代价很高**。
