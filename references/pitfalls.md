## 5. ⚠️ 六个必踩的坑（🟡 实测经验，非官方规定）

> **本节全部为 🟡 实测内容** —— 都是在本机真实事故中复现并修复的，附崩溃日志原文。**官方文档并没有把这些写成规则**，引用时请注明这是实践经验，而不是官方规定。
>
> 例外：**5.2 与 5.6 的底层规则本身是 🟢 官方的**（见 `publish.md` 的「依赖划分」与「从 GitHub 安装」两节），这里只是把官方规则与真实事故放在一起讲。

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
