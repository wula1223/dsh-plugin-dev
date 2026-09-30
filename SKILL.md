---
name: dsh-plugin-dev
description: 制作、修改或审查 DeepSeek Harness (DSH) 插件 / 钩子 / Skill 时必读。涵盖官方 cordis bundle 契约、组合包与 profile 两种 manifest、四层加载顺序、Fiber 生命周期、事件分发模式（emit/waterfall/parallel/serial/bail）、插件配置 schema、SKILL.md 格式、版本兼容白名单纪律，以及影子包 / 依赖悬空 / BOM / profile 并发写入等实战避坑与本地验证流程。当用户要求开发 DSH 插件、加钩子、写 skill、打包发布插件，或排查插件显示「异常 / 未运行 / 启动失败 / 桌面端打不开」时使用。
---

# DSH 插件 / Skill 开发规范

DeepSeek Harness（DSH）的设计哲学是 **「Everything is a Plugin」**。插件、工具、UI、LLM 适配器全部走同一套 cordis 插件模型。

**动手前先读本文，尤其第 5 节（六个必踩的坑）—— 它们会直接搞崩用户的桌面端。**

---

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

---

## 3. 生命周期：Fiber 状态机 ⭐

每个被加载的插件都拥有一个 **Fiber** 作用域：

```
PENDING → LOADING → ACTIVE
                 ↘ FAILED
ACTIVE → UNLOADING → DISPOSED
```

| 状态 | 含义 |
|---|---|
| **PENDING** | 已声明，但**所需依赖未就绪** ← 这就是面板显示"等待服务"的状态 |
| LOADING | 依赖就绪，正在执行 `apply` |
| ACTIVE | 插件运行中 |
| **FAILED** | **`apply` 抛出异常** |
| UNLOADING | 正在卸载并释放资源 |
| DISPOSED | 已完全卸载 |

**依赖驱动的加载**：声明了 `inject` 的插件等待所有必需服务就绪。**如果依赖的服务消失**（例如提供方被替换），插件会**自动卸载**（ACTIVE → DISPOSED），**待服务恢复后重新加载**。

### 自动清理

通过 `ctx` 做的任何注册，卸载时自动撤销 —— 以下都会被自动追踪：

- `ctx.on(event, handler)` — 事件监听
- `ctx.tools.register(tool)` — 工具注册
- `ctx.llm.registerAdapter(names, adapter)` — LLM 适配器注册
- `ctx.effect(() => cleanup)` — 自定义资源

```ts
export function apply(ctx: Context) {
  ctx.on('some-event', handler)

  ctx.effect(() => {
    const connection = createConnection()
    return () => connection.close()     // 卸载时执行
  })
}
```

⚠️ **处置顺序的坑**：卸载时处置器**按注册顺序的逆序开始调用**，但**多个异步处置器会并发执行，不保证逐个完成**。

> 存在顺序依赖的清理步骤，**必须放进同一个 `ctx.effect()` 返回的处置器中**，由该处置器负责串行等待。

### 嵌套上下文与 dispose

```ts
ctx.plugin(childPlugin)      // 子 Fiber：继承父上下文，独立生命周期，随父卸载

const fiber = ctx.plugin(myPlugin)
await fiber.dispose()        // 手动提前终止
```

`dispose` 保证：① 该插件所有注册被移除 ② 子树递归卸载 ③ Promise 在所有异步清理完成后兑现。

### HMR

加载 `@deepseek-ai/dsh-hmr` 后，改插件源码触发：卸载旧插件（清理所有注册）→ 重新加载新代码 → 执行新 `apply`。因注册自动清理，**热替换不会保留旧实例的注册**。

---

## 4. Skill 格式

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

## 5. ⚠️ 六个必踩的坑（真实事故总结）

### 5.1 版本兼容白名单 —— 插件「异常」的头号原因

DSH 插件的兼容性声明是**精确版本白名单**，**不是语义化范围**：

```json
"peerDependencies": {
  "@deepseek-ai/dsh-skill": "0.1.2-rc.1 || 0.1.7-rc.2 || 0.2.0-rc.1"
}
```

宿主一升级（如 `0.1.7-rc.2` → `0.2.0-rc.2`），白名单没跟上 → 插件**自检失败 → 面板显示「异常」**。

**规则**：宿主升级后所有插件必须同步升级；排查「异常」第一步就是比对白名单与宿主版本。

### 5.2 核心包写进 `dependencies` 会遮蔽宿主

**官方规则**（`docs/user/develop/basic/publish.md`）：

> 与 harness 自身的包一样，需要与宿主**共享实例**的 dsh 包同时声明在 **`peerDependencies` 与 `devDependencies`** 中。
> 需要**独立版本**的第三方依赖和**无状态 dsh 工具包**放在 `dependencies` 中。

**真实事故**：`dsh-harness-zh-cn@0.1.2` 把 `"@deepseek-ai/dsh-llm": "0.0.1-rc.1"` 写进 `dependencies`，装完后 profile 里出现 `node_modules\@deepseek-ai\dsh-llm@0.0.1-rc.1`，**遮蔽宿主** → `llm` 服务无法激活 → 8 个插件排队（PENDING）→ 必需的 `agent-loop` 起不来 → **桌面端彻底打不开**。

**规则**：**有状态/需共享实例的核心包 → `peerDependencies` + `devDependencies`；无状态工具包 → `dependencies`。** 装完**必查影子包**（见 6.2）。

### 5.3 `inject` 依赖悬空 —— 插件永远 PENDING

插件 `inject` 了某个服务，但提供该服务的插件没启用 → 该插件永远停在 **PENDING** → **web boot 失败**。

**真实事故**：`@huanlin/dsh-plugin-better-sidebar-plugin-office` 依赖 `betterSidebar` 服务，但 `dsh-better-sidebar` 没启用：

```
web boot: 1 entry did not activate
@huanlin/...-office: pending (waiting for service: betterSidebar)
```

**规则**：成套插件必须**成组启用**。启用前先确认它的 `inject` 服务有提供者。

### 5.4 BOM 头 —— 一行字节搞崩启动

用 PowerShell 的 `Set-Content -Encoding UTF8` 写 profile JSON，**Windows PowerShell 5.1 会写入 BOM**（`EF BB BF`）。DSH 用 `JSON.parse` 读清单 → 第一个字符非法：

```
SyntaxError: Unexpected token '', "{ "n"... is not valid JSON
    at readProfileManifest (…/dsh-app-boot/lib/index.js:835:22)
```

**规则**：写 profile JSON 必须**无 BOM**：

```powershell
$utf8NoBom = New-Object System.Text.UTF8Encoding($false)
[System.IO.File]::WriteAllText($path, $json, $utf8NoBom)
$b = [System.IO.File]::ReadAllBytes($path)
"BOM = $($b[0] -eq 0xEF -and $b[1] -eq 0xBB -and $b[2] -eq 0xBF)   # 必须 False"
```

### 5.5 profile 会被并发改写

DSH 自身/插件管理器会在启动和插件操作时重写 `package.json`、`cordis.yml`、`cordis.patch.yml`。

- **先停 DSH 再改配置**，否则改动会被覆盖
- 改完**核对文件时间戳**确认没被回写
- `cordis.yml` 是自动生成的空模板（约 223 B），**每次启动被重写属正常，不要手改**
- 排查"谁在改"看 `.dsh-market\log.ndjson`

### 5.6 从 git 安装 = 拉源码，不是构建产物

```sh
dsh plugin --profile demo add github:you/hello-plugin
```

**没有任何环节运行你的 `build` 脚本** —— TypeScript 包到手时没有 `lib/` 输出，**加载会失败**。必须两边各做一件事：

- **作者**：提供 `prepare` 脚本（pnpm 在 git 安装后运行），从源码构建发布入口，且必须**自包含**（不能假设旁边有 monorepo checkout）。专用 tsdown 配置可直接转译 `src/`，不做类型检查。
- **用户**：为构建授权。pnpm ≥10 在显式允许前拒绝运行 git 依赖的 `prepare` 脚本，第一次 `add` 会失败 —— 把 pnpm 打印的包键复制进 profile 的 `pnpm-workspace.yaml`：

  ```yaml
  allowBuilds:
    dsh-hello-plugin: true
  ```

> ⚠️ **把这项授权视为「允许该包代码在安装时于你机器上执行」**，且不在 agent 运行的任何沙箱之内。只对源码可信的包授权，并**锁定 commit**（`github:you/hello-plugin#<sha>`），让后续推送无法悄悄改变实际运行的内容。

**不想让用户授权？** 改为分发构建产物：

- **发布到 npm**（`pnpm publish` 时构建好 `lib/`）→ `dsh plugin add your-package` 装的就是预构建代码
- **交付 tarball**：`pnpm pack` → 用户 `dsh plugin add ./hello-plugin-0.1.0.tgz`

---

## 6. 本地验证清单（每次改完必做）

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

## 7. 发布到 GitHub / 上架

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

## 8. 排查流程（插件异常 / 桌面端打不开）

1. **读最新崩溃日志**（第 6.5 节表格对照）
2. **比对插件白名单与宿主版本**（5.1）
3. **查影子包**（5.2 / 6.2）
4. **查 `inject` 依赖是否悬空**（5.3，对照 Fiber PENDING）
5. **查 BOM**（5.4）
6. **看 `.dsh-market\log.ndjson`** 找 `install-compat` 警告与 toggle 记录
7. 全都不行 → 用 DSH 自带的 **「禁用第三方插件、备份 profile patch 并重启」** 安全模式恢复

---

## 9. 汇报原则

改完必须给出**证据**，不能只说"应该好了"：

- 文件版本号 / 时间戳 / 哈希
- 影子包扫描结果
- 崩溃日志计数 `before -> after`
- 进程存活时长

用户曾因"报告启动成功但实际没起来"而多折腾一轮 —— **验证不到位的代价很高**。
