## 5. ⚠️ 九个必踩的坑（🟡 实测经验，非官方规定）

> **本节全部为 🟡 实测内容** —— 都是在本机真实事故中复现并修复的，附崩溃日志原文。**官方文档并没有把这些写成规则**，引用时请注明这是实践经验，而不是官方规定。
>
> 例外：**5.2 与 5.6 的底层规则本身是 🟢 官方的**（见 `publish.md` 的「依赖划分」与「从 GitHub 安装」两节），这里只是把官方规则与真实事故放在一起讲。
>
> **每条都按「故障模式」写，不绑具体插件** —— 遇到同类现象直接套用判断依据，不要记插件名。

### 5.1 版本兼容白名单 —— 宿主升级后最常见的两类失败

DSH 插件的兼容性声明是**精确版本白名单**，**不是语义化范围**：

```json
"peerDependencies": {
  "@deepseek-ai/dsh-skill": "0.1.2-rc.1 || 0.1.7-rc.2 || 0.2.0-rc.1"
}
```

宿主一升级，白名单没跟上 → 插件**自检失败**。但**失败分两种，处理方式完全相反**，必须先分清：

| 界面/日志表现 | 含义 | 正确动作 |
|---|---|---|
| 面板显示「**异常**」，带上被拒绝的依赖清单 | 白名单**缺宿主版本**，插件自身没问题 | **升级插件** |
| 面板显示「**更新失败**」，且日志有 `update-rollback` | 新版本**自身有 bug** | **不要升**，停用或等作者修复 |

**判断方法**（不要猜，查日志）：

```powershell
$prof = "$env:USERPROFILE\.dsh\profiles\desktop"
# 1. 宿主版本
(Get-Item "D:\DeepSeek-Harness\DeepSeek Harness.exe").VersionInfo.FileVersion
# 2. 插件白名单里有没有这个版本
(Get-Content "$prof\node_modules\<包名>\package.json" -Raw -Encoding UTF8 |
  ConvertFrom-Json).peerDependencies.'@deepseek-ai/dsh-skill'
# 3. 是「异常」还是「更新失败」——看 market 日志
Get-Content "$prof\.dsh-market\log.ndjson" -Encoding UTF8 |
  Select-String 'error' | Select-Object -Last 10
```

判定规则：

- 白名单**缺**当前宿主版本，且 npm 上**有**含该版本的新版 → **升级**，安全
- 日志里有 `update-rollback` / `restoration ... could not be verified` → **新版有 bug**，升级会失败并回滚，别试第二次
- 同一升级**连续失败两次以上** → 属稳定缺陷，不是偶发，**停止重试**

**规则**：宿主升级后逐个核对白名单；但「异常」才升级，「更新失败」要停手 —— 这两者外观都是「这个插件有问题」，实际结论相反。


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


### 5.5 补充：`disabled` / `brokenPlugins` 也是会被程序改写的状态

除了正文提到的三份配置文件，`.dsh-market\state.json` 同样会被市场写回。「改完没生效」要按 5.5 的思路核对**文件时间戳**与**进程是否重启**，而不是只重试一次。

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

### 5.7 客户端工厂重复注册 —— 装了但加载器找不到，或被注册两次

**症状**（浏览器渲染器控制台）：

```
client-modules: duplicate factory registration for "<factory-id>"
(bundle executed twice without invalidate?)
```

主机侧表现为：

```
web boot: 1 entry did not activate
<包名>: import failed (see console for the import error)
```

**故障模式**：某个 `dsh.client` 声明了 `id`，但同一 factory 在**一次启动内被执行了两次**。常见成因：

| 成因 | 判断方法 |
|---|---|
| `dsh.client.inject` **漏声明**它所依赖的对端插件 | 看 `package.json` 的 `dsh.client.inject`，是否与代码里实际用到的服务/模块对不上 |
| 缺少 `immediately` 字段，加载时序不定 | 对比同批插件，多数写了 `"immediately": true` |
| 同一包在 `bundles` 里挂了两遍 | 见下方去重检查 |

**判断步骤**：

```powershell
$prof = "$env:USERPROFILE\.dsh\profiles\desktop"
# 1. bundles 是否真的重复（注意：只统计 bundles 段内的字符串）
$raw = [System.IO.File]::ReadAllText("$prof\package.json",[System.Text.Encoding]::UTF8)
$b = [regex]::Match($raw,'"bundles":\s*\[(.*?)\]','Singleline').Groups[1].Value
[regex]::Matches($b,'"([^"]+)"') | ForEach-Object { $_.Groups[1].Value } |
  Group-Object | Where-Object Count -gt 1
# 2. 这个 factory id 是否只有一个来源
Get-ChildItem $prof\node_modules -Recurse -Include *.js -ErrorAction SilentlyContinue |
  Where-Object { $_.FullName -notmatch 'node_modules\\.*node_modules' } |
  ForEach-Object {
    $t = [System.IO.File]::ReadAllText($_.FullName,[System.Text.Encoding]::UTF8)
    if ($t -match '<factory-id>') { $_.FullName }
  }
```

**规则**：

- 先确认 `bundles` 没重复（**最常见的排查错误就是把 `dependencies` 里的同名条目算成重复**）
- 只有一个来源却仍报重复 → 是该包的 `dsh.client` 声明不完整（`inject` 缺项 / 缺 `immediately`）
- **此类缺陷改配置往往无效**：实测给 `inject` 补上缺失项、加 `immediately: true` 后，重复从 1 次变成 2 次，反而更糟。**唯一可靠的处理是停用/卸载该插件**，并向作者提 issue
- 若成组插件互相依赖（例如 A 提供侧边栏服务、B/C/D 挂在 A 上），**整组一起停**，只停 B 会让 C/D 变成 5.3 的 PENDING


### 5.8 路由重复注册 —— 多见于升级回滚后的半安装状态

**症状**：

```
dsh: warning: 1 entry did not activate
<包名>: Error: webserver: duplicate exact route "/<前缀>/<路径>"
inspect this profile before restarting
```

market 面板同时给出：

```
更新失败: <包名> — 更新失败，且更新前的构建未能验证恢复
```

**故障模式**：**升级进行到一半失败**。旧版本仍驻留内存，新版本已注册同名路由，两边撞车。`.dsh-market\log.ndjson` 里的典型序列：

```
update-start    : <包> -> <包>@<新版本> (profile desktop)
update-rollback : failed update command and restoration of the previous build could not be verified
update          : <包> -> <包>@<新版本> exit=1 err=…
```

**判断步骤**：

```powershell
$prof = "$env:USERPROFILE\.dsh\profiles\desktop"
# 1. 实际装的是哪个版本（以 node_modules 为准，不看面板）
(Get-Content "$prof\node_modules\<包名>\package.json" -Raw -Encoding UTF8 |
  ConvertFrom-Json).version
# 2. 该包是否被装了多份
Get-ChildItem $prof -Recurse -Directory -Filter "<包名>" -Depth 5 -ErrorAction SilentlyContinue
# 3. 升级失败了几次
Get-Content "$prof\.dsh-market\log.ndjson" -Encoding UTF8 |
  Select-String '<包名>.*update' | Select-Object -Last 10
```

**规则**：

- **`node_modules` 的版本才是事实**，面板上的「更新失败」只是历史记录
- 版本已回滚到旧版（与之前一致）→ **系统是好的，不用处理**，重启即可清除提示
- 同一升级**失败两次以上** → 稳定缺陷。多半是新版把单包拆成多个子包、子包依赖的服务没起来（表现为一串 `pending (waiting for service: xxx)`）。**冻结在旧版本，不要重试**
- 排查新版为何失败，看 `err=` 后面的 PENDING 列表 —— 那是在告诉你缺哪个服务


### 5.9 market 的双重拦截 —— 「已安装、在 bundles 里，却加载不了」

**症状**：面板提示「**更新失败后被移除了**」「为避免下次启动卡住，市场把它从 profile 里移除」；但查 `package.json` 发现它**仍在 `bundles` 里**。日志：

```
toggle : <包名>: no loader entry matched
toggle : <包名> -> off: fiber=false
boot   : plugin kept off: <包名>
```

**故障模式**：插件被 **两处独立状态**同时挡住，只解一处不够：

```
.dsh-market\state.json → disabled[]        ← 直接原因（kept off 的来源）
.dsh-market\state.json → brokenPlugins{}   ← 历史坏件记录，可能 spec 已过期
```

`brokenPlugins` 会**记住某个具体版本**是坏的。插件后来更新了，这条记录却不会自动清除 —— 于是新版也被误伤。

**判断步骤**：

```powershell
$prof = "$env:USERPROFILE\.dsh\profiles\desktop"
$s = Get-Content "$prof\.dsh-market\state.json" -Raw -Encoding UTF8 | ConvertFrom-Json
"disabled      : " + ($s.disabled -join ', ')
"brokenPlugins : " + ($s.brokenPlugins | ConvertTo-Json -Compress)
# 该包实际版本，与 brokenPlugins 里记录的 spec 对比
(Get-Content "$prof\node_modules\<包名>\package.json" -Raw -Encoding UTF8 | ConvertFrom-Json).version
```

**规则**：

- `disabled` 里有 → **必须移除**，这是 `kept off` 的直接原因
- `brokenPlugins` 里记录的 `spec` **与实际版本不同** → 记录已过期，可清除
- **改完必须完全退出 DSH 再启动**（托盘退出，不是关窗口）。配置改了但进程没重启，界面仍显示旧警告 —— 别误判为「改了没用」
- 改前先备份 `state.json`；**写回时必须无 BOM**



---


### 5.10 `Config` 写成普通对象 —— 插件激活时 `.validate` 崩溃

**症状**：面板显示「**异常**」，点启用只弹一行，没有堆栈：

```
启用失败: dsh: warning: 1 entry did not activate <id> (<包名>):
TypeError: Cannot read properties of undefined (reading 'validate')
```

**故障模式**：插件把设置项声明成**普通对象**，而宿主按 **Schemastery Schema** 使用它：

```ts
// ✗ 崩溃：普通对象没有 validate
export const Config = { pythonPath: 'python', latexEngine: 'xelatex' }
```

Cordis 挂载插件时会校验 config schema。取到的是普通对象，读 `.validate` 得 `undefined`，
随后调用即抛错，**整个插件激活失败**（工具一个都没注册）。

**为什么容易写错**：官方 `publish.md` 说「依赖来自用户 profile 的 `node_modules`」，
常被理解成「不能 import 任何 `@deepseek-ai/*`」，于是有人为「零依赖」把 Schema 降级成普通对象 —— 反而制造崩溃。

**正确写法**（`@deepseek-ai/schemastery` 是 Cordis 生态基础库，profile 里必然可解析）：

```ts
import Schema from '@deepseek-ai/schemastery'

export const Config: Schema<Config> = Schema.object({
  pythonPath: Schema.string().default('python'),
  latexEngine: Schema.union(['xelatex', 'tectonic'] as const).default('xelatex'),
  buildTimeoutMs: Schema.number().default(300000),
})
```

同时 `package.json` 声明 peer 依赖：

```json
"peerDependencies": { "@deepseek-ai/schemastery": "^3.18.5" }
```

**判断步骤**（区分「代码没写好」与「装的是旧版」）：

```js
const m = await import('file:///<profile>/node_modules/<包名>/lib/index.js')
console.log(typeof m.Config)                        // 应为 'function'（Schema 是函数）
console.log(typeof m.Config?.['~standard']?.validate) // 应为 'function'
```

**规则**：

- `Config` **必须是 `Schema.object(...)`**，不能是普通对象、也不能省略
- Schemastery 的校验入口在 **`Schema['~standard'].validate`**，不是 `Schema.validate`
- **可以**依赖 `@deepseek-ai/schemastery`；**不能**依赖 `dsh-tools` / `dsh-skill` 这类藏在 `app.asar` 里的宿主内部包（错误是 `Cannot find package`，与本条不同）
- 面板只显示「异常」两个字时，**先想办法拿到那句英文报错**——本条的 `reading 'validate'` 与 5.3 的 `pending`、5.10 上方各条的错误文本互不重叠，报错原文是唯一可靠的分诊依据


### 5.11 `ctx.get(name)` 在服务缺失时【抛异常】，不是返回 `undefined`

**症状**：插件在面板标「异常」，但**工具其实已注册成功**（能调用），状态灯却是红的。日志：

```
服务激活失败: service not found: skills
    at registerSkills (.../lib/skills.js)
    at apply (.../lib/index.js)
```

**故障模式**：代码里写了「可选依赖 + 静默降级」，但判断方式错了：

```ts
// ✗ 假设 get 返回 undefined —— 缺服务时直接抛，降级逻辑永远走不到
const skills = ctx.get?.('skills')
if (skills == null) return () => {}
```

Cordis 的 `ctx.get(name)` 在服务未加载时**抛错**，不是返回 `undefined`。
未写进 `inject` 的可选服务一旦缺失，`apply()` 在注册完全部工具后抛错，
Loader 把整个插件标记为失败 —— **但工具已经进去了**，于是出现「能用却显示异常」这种矛盾状态。

**正确写法**：

```ts
export const inject = ['tools']   // 只放【必选】；可选服务不要写进来

export function registerSkills(ctx: Context): () => void {
  let skills: any
  try {
    skills = (ctx as any).get?.('skills')     // 探测要 try/catch
  } catch { return () => {} }
  if (skills == null || typeof skills.register !== 'function') return () => {}

  let disposer: any
  try {
    disposer = skills.register({ /* ... */ })  // 注册也 try/catch
  } catch { return () => {} }
  return () => { if (typeof disposer === 'function') disposer() }
}
```

**规则**：

- **可选依赖不要写进 `inject`** —— 写进去会让插件在缺该服务时根本不加载（比运行时降级更糟，见 5.3）
- 任何 `ctx.get(...)` **一律包 `try/catch`**，不要依赖 `== null` 判空
- 判断「异常」到底有没有影响功能：**看工具是否真的可调用**。工具在、只是状态红 → 本条；工具也不在 → 属 5.3 / 5.10
- 同一 `apply()` 里先注册工具再探测可选服务，会让失败表现得更迷惑（工具成功、插件失败）。**把可选服务的探测前置**，失败更早暴露
