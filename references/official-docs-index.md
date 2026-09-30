# Official DSH documentation index

Two sources, both official:

1. **Documentation site** — <https://deepseek-harness.github.io/deepseek-harness/> (VitePress; source lives in the repo under `docs/user/`)
2. **Repo docs** — `github.com/deepseek-ai/deepseek-harness`, branch `master`, `docs/`

---

## 0. Skill authoring - read these first

If your task is about **writing or reviewing a skill** (not a plugin), these are the authoritative sources:

| Document | Why it matters |
|---|---|
| **`docs/subsystems/skills.md` / `.zh.md`** | The official skill specification. Name format (`^[a-z0-9]+(?:-[a-z0-9]+)*$`), the six local discovery roots and their rank order, the exact frontmatter keys (`disable-model-invocation`, `user-invocable`), the `description` cap (`catalogDescriptionMaxLength`, default 500), and the `ctx.skills` registry surface. |
| **`packages/preset/agent-preset/skills/`** | The official bundled skills - the best structural templates. Contains `cordis-plugin-development` (a thin SKILL.md plus `references/` and `templates/`), `cordis-composition-reference`, and `editing-cordis-compositions`. |
| **`packages/skill/skill-filesystem/src/index.ts`** | The local provider: how directories and flat `.md` files are discovered and parsed. |
| **`packages/skill/tool-skill/src/index.ts`** | The model-facing `skill` tool: what it returns and how `resourceBase` resolves relative files. |
| **`scripts/verify-skill-invocation-metadata.ts`** | The repo's own validator for skill invocation metadata. |

### The progressive-disclosure pattern

The official `cordis-plugin-development` skill is about 8 KB and delegates everything else:

```
SKILL.md              thin router + a Task -> File table
references/*.md       host-plugin / ui-plugin / mcp-bundle / practices / user-actions / verification
templates/            copy-paste starting points
```

Its SKILL.md states that the table **is the complete list - do not enumerate the directory**. Follow this shape: keep the main file small and route.

> NOTE: In the Desktop build the bundled skill directory lives inside `app.asar`. Only the Host process's own file reads can open it - shell commands (`ls`, `cat`, `cp`), the glob and search tools (they run a native ripgrep process), `node`, and pnpm all fail on it.

---
## 1. Documentation site (read this first)

Base: `https://deepseek-harness.github.io/deepseek-harness/`

> 💡 **Every page has a raw Markdown twin** — append `.md` (e.g. `/develop/basic/publish.md`). Much cleaner than scraping HTML.
> English site: insert `en/` after the base path, e.g. `/deepseek-harness/en/develop/basic/`.

### 开发 / develop

| Page | Path | What it covers |
|---|---|---|
| 第一个插件 | `/develop/basic/` | Minimal `apply(ctx)` plugin; the three plugin forms |
| 开发一个 Tool | `/develop/basic/tool` | Tool definition DSL |
| 插件配置 | `/develop/basic/config` | `Config` type + Schemastery schema; no-hardcoded-knobs principle |
| **打包与安装插件** | `/develop/basic/publish` | Bundle vs profile manifest, 4-layer ordering, the git-install build trap |
| 插件与生命周期 | `/develop/framework/` | **Fiber state machine**, auto-cleanup, dispose semantics, HMR |
| 服务与依赖 | `/develop/framework/service` | Providing capabilities to other plugins |
| 事件系统 | `/develop/framework/events` | The 5 dispatch modes, typed events, naming conventions |
| 能力的三层拆分 | `/develop/practice/` | Three-layer capability split |
| LLM 适配器 | `/develop/practice/llm-adapter` | Implementing a full LLM backend |
| 持久化 Harness 插件 | `/develop/practice/dynamic-cordis` | Dynamic persistence |
| Cordis 教程 | `/develop/cordis-tutorial/` | 7 hands-on chapters, no API key needed |

### 入门 / guide · 参考 / reference

| Section | Path |
|---|---|
| 快速开始 | `/guide/quickstart` |
| 参考（子系统、配置、工具目录） | `/reference/` |
| **Skills 子系统** | `/reference/subsystems/skills.md` |

---

## 2. Repo docs

Raw file pattern:

```
https://raw.githubusercontent.com/deepseek-ai/deepseek-harness/master/docs/<file>
```

Nearly every document has a Chinese twin: replace `.md` with `.zh.md`.

### Plugin development

| Document | Size | What it covers |
|---|---|---|
| `docs/cordis-primer.md` / `.zh.md` | 4 KB | The five core cordis concepts |
| `docs/cordis-tutorial/` | 8 chapters | First plugin → lifecycle → services → events → config → HMR → into the harness |
| `docs/cordis-api/` | — | Generated Cordis core API reference |
| `docs/capability-seams.md` / `.zh.md` | **61 KB** | Every extension point the harness exposes |
| `docs/config-catalog.md` / `.zh.md` | **210 KB** | Authoritative catalog of all configuration entries |
| `docs/architecture.md` / `.zh.md` | 19 KB | Composition, core packages, the loop, seams |

### Cookbook

| Document | Size |
|---|---|
| `docs/cookbook/extension-cookbook.zh.md` | 11 KB |
| `docs/cookbook/adding-a-package.zh.md` | 15 KB |
| `docs/cookbook/adding-a-tool.zh.md` | 14 KB |
| `docs/cookbook/adding-a-settings-card.zh.md` | 4 KB |
| `docs/cookbook/adding-an-llm-adapter.zh.md` | 4 KB |
| `docs/cookbook/adding-a-remote-api.zh.md` | 10 KB |
| `docs/cookbook/adding-a-session-format-version.zh.md` | 19 KB |
| `docs/cookbook/adding-a-vendored-package.zh.md` | 4 KB |

### Reference catalogs

| Document | Size |
|---|---|
| `docs/tool-catalog.md` / `.zh.md` | 98 KB |
| `docs/event-producer-consumer.md` / `.zh.md` | 29 KB |
| `docs/module-graph.md` / `.zh.md` | 123 KB |
| `docs/persistence-catalog.md` / `.zh.md` | 342 KB |
| `docs/persistence-schema.json` | 617 KB |
| `docs/dependency-catalog.json` | 168 KB |
| `docs/glossary.md` / `.zh.md` | 7 KB |

### Process & standards

| Document | Size |
|---|---|
| `docs/AGENTS.md` | 11 KB (documentation standard) |
| `docs/development.md` / `.zh.md` | 19 KB |
| `docs/testing.md` / `.zh.md` | 11 KB |
| `docs/defensive-patterns.md` / `.zh.md` | 4 KB |
| `docs/upgrade-guide/` | version upgrade guides |
| `docs/subsystems/` | one reference page per subsystem |
| `docs/postmortem/` | incident write-ups |

---

## 3. Official skills (design references)

`/.agents/skills/` contains 15 official skills:

```
agent-experience            dsh-archive-agent-notes    dsh-ci-test-reliability
dsh-client-ui-ux            dsh-code-review            dsh-create-upgrade-guide
dsh-doc                     dsh-find-simplifications   dsh-merging-stacked-prs
dsh-pre-push-checks         dsh-prose-standard         dsh-speed-up-perf
dsh-translate-docs          dsh-trim-cot-leakage       record-browser-gif
```

## 4. Built-in product skills (shipped with the desktop app)

```
D:\DeepSeek-Harness\resources\runtime\office-skills\office-docx\SKILL.md
D:\DeepSeek-Harness\resources\runtime\office-skills\office-pptx\SKILL.md
D:\DeepSeek-Harness\resources\runtime\office-skills\office-xlsx\SKILL.md
```

The most reliable on-disk examples of the `SKILL.md` frontmatter contract.
