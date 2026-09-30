# Official DSH documentation index

Source repository: **`github.com/deepseek-ai/deepseek-harness`** (branch `master`).

Raw file pattern:

```
https://raw.githubusercontent.com/deepseek-ai/deepseek-harness/master/docs/<file>
```

Nearly every document has a Chinese twin: replace `.md` with `.zh.md`.

---

## Plugin development

| Document | Size | What it covers |
|---|---|---|
| `docs/cordis-primer.md` / `.zh.md` | 4 KB | The five core cordis concepts: plugin = Service, context as container, `inject`, typed events, reversible effects |
| `docs/cordis-tutorial/` | 8 chapters | Hands-on: 01 first plugin → 02 lifecycle & effects → 03 services → 04 events → 05 config → 06 composition & HMR → 07 into the harness |
| `docs/cordis-api/` | — | Generated Cordis core API reference |
| `docs/capability-seams.md` / `.zh.md` | **61 KB** | Every extension point the harness exposes |
| `docs/config-catalog.md` / `.zh.md` | **210 KB** | Authoritative catalog of all configuration entries |
| `docs/architecture.md` / `.zh.md` | 19 KB | Composition, core packages, the loop, seams, extension points |

## Cookbook (how-tos)

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

## Reference catalogs

| Document | Size | What it covers |
|---|---|---|
| `docs/tool-catalog.md` / `.zh.md` | 98 KB | Every tool definition |
| `docs/event-producer-consumer.md` / `.zh.md` | 29 KB | Event producers and consumers |
| `docs/module-graph.md` / `.zh.md` | 123 KB | Module dependency graph |
| `docs/persistence-catalog.md` / `.zh.md` | 342 KB | Persistence schema catalog |
| `docs/persistence-schema.json` | 617 KB | Machine-readable persistence schema |
| `docs/dependency-catalog.json` | 168 KB | Dependency catalog |
| `docs/glossary.md` / `.zh.md` | 7 KB | Terminology |

## Process & standards

| Document | Size | What it covers |
|---|---|---|
| `docs/AGENTS.md` | 11 KB | The documentation standard: doc structure, tier taxonomy, writing rules |
| `docs/development.md` / `.zh.md` | 19 KB | Development workflow |
| `docs/testing.md` / `.zh.md` | 11 KB | Testing standard |
| `docs/defensive-patterns.md` / `.zh.md` | 4 KB | Defensive coding patterns |
| `docs/upgrade-guide/` | — | Version upgrade guides (e.g. `v0.1.7-rc.2`) |
| `docs/subsystems/` | — | One reference page per subsystem, including generated Cordis API |
| `docs/postmortem/` | — | Incident write-ups |
| `docs/i18n/` | — | Translation conventions |
| `docs/rescope.md` | 5 KB | Scope restructuring notes |
| `docs/graph-atlas.md` | 1 KB | Graph atlas |
| `docs/agent-lifecycle.md` / `.zh.md` | 6 KB | Agent lifecycle |
| `docs/api-gateway.md` / `.zh.md` | 20 KB | API gateway |
| `docs/deepseek-llm-api-wire-extensions.md` | 13 KB | LLM API wire extensions |
| `docs/session-format-status.md` | 6 KB | Session format status |
| `docs/tool-execution-pipeline.md` | 4 KB | Tool execution pipeline |
| `docs/ui-radius.md` / `.zh.md` | 10 KB | UI radius / design tokens |
| `docs/web-styling.md` / `.zh.md` | 7 KB | Web styling standard |

## Official skills (design references)

`/.agents/skills/` contains 16 official skills:

```
agent-experience            dsh-archive-agent-notes    dsh-ci-test-reliability
dsh-client-ui-ux            dsh-code-review            dsh-create-upgrade-guide
dsh-doc                     dsh-find-simplifications   dsh-merging-stacked-prs
dsh-pre-push-checks         dsh-prose-standard         dsh-speed-up-perf
dsh-translate-docs          dsh-trim-cot-leakage       record-browser-gif
```

## Built-in product skills (shipped with the desktop app)

```
D:\DeepSeek-Harness\resources\runtime\office-skills\office-docx\SKILL.md
D:\DeepSeek-Harness\resources\runtime\office-skills\office-pptx\SKILL.md
D:\DeepSeek-Harness\resources\runtime\office-skills\office-xlsx\SKILL.md
```

These are the most reliable on-disk examples of the `SKILL.md` frontmatter contract.
