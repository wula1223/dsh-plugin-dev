# dsh-plugin-dev

> **DeepSeek Harness plugin / hook / skill development standard** - a battle-tested playbook for AI agents and human developers.

English | [中文](README.md)

---

## What is this

A **DSH agent skill** that makes an AI follow the official contract and avoid known desktop-breaking pitfalls when building, modifying, or reviewing **DeepSeek Harness plugins / hooks / skills**.

It is not a general introduction - it is an **operating manual that fuses the official specification with real incidents**. The mechanism sections track the official docs; every "pitfall" corresponds to an actual boot failure.

## Why you need it

DSH's philosophy is **"Everything is a Plugin"**, and its plugin ecosystem is wide open. That openness comes with sharp edges:

- A **bundle and a profile are two different manifests** - confuse them and your package won't install
- Plugins declare an **exact-version allowlist** (not a semver range) - one host upgrade and they break
- A stateful core package in `dependencies` **shadows the host** and can make the desktop app unlaunchable
- A plugin whose `inject`ed service has no provider stays **PENDING** forever and stalls startup
- A single **BOM byte** is enough to crash the whole boot sequence in `JSON.parse`
- **Installing from git pulls source, not build output** - without a `prepare` script the package won't load

Without this knowledge, an AI easily ships a plugin that "looks right but breaks on install".

## Install

### Option 1 - clone into the global skills directory (recommended)

```bash
git clone https://github.com/wula1223/dsh-plugin-dev.git \
  ~/.dsh/skills/dsh-plugin-dev
```

Windows PowerShell:

```powershell
git clone https://github.com/wula1223/dsh-plugin-dev.git `
  "$env:USERPROFILE\.dsh\skills\dsh-plugin-dev"
```

### Option 2 - copy SKILL.md only

```powershell
$dest = "$env:USERPROFILE\.dsh\skills\dsh-plugin-dev"
New-Item -ItemType Directory -Force -Path $dest | Out-Null
Copy-Item .\SKILL.md $dest
```

**No restart is needed.** DSH's local skill provider runs a file watcher (chokidar) that hot-refreshes the skill catalog - the new skill shows up on its own.

> Option 2 copies only the main file, so the `references/` pages are left behind. Use option 1.

## Layout

The main file is a **thin router** (~18 KB); detail loads on demand:

| File | Purpose |
|---|---|
| [`SKILL.md`](SKILL.md) | **The skill itself**: provenance grading, routing table, authoritative sources, plugin contract, skill format, verification checklist, release, triage, reporting standard |
| [`references/capability-map.md`](references/capability-map.md) | The `ctx` key table + the "where new behavior belongs" map + the three seam roles |
| [`references/events.md`](references/events.md) | The five dispatch modes, the three event domains, a Claude Code term crosswalk, the `hooks.json` bridge |
| [`references/lifecycle.md`](references/lifecycle.md) | The Fiber state machine, auto-cleanup, dispose, HMR |
| [`references/pitfalls.md`](references/pitfalls.md) | **Nine pitfalls**, grouped by failure mode, each with the real crash log and a diagnostic script |
| [`references/ecosystem.md`](references/ecosystem.md) | MCP, hooks bridges, memory, output styles, plugin market, and the official package lineage |
| [`references/official-docs-index.md`](references/official-docs-index.md) | The official docs map, the skill specification, and the official skill templates |

> This "thin main file + reference pages" split follows the structure of the **official bundled skill `cordis-plugin-development`** (~8 KB plus `references/` and `templates/`).

## Content provenance

This skill **does not invent rules**. Everything is graded by source:

| Mark | Meaning |
|---|---|
| 🟢 **official** | Sourced from the official docs site, repo, or bundled artifacts - you can find the original |
| 🟡 **empirical** | Verified in real incidents on this machine, with crash logs, but **not written down as a rule by the official docs** |
| 🔵 **community** | Third-party plugins, names verified on npm, but **not official** |

Do not cite 🟡 or 🔵 material as if it were an official rule.

## The nine pitfalls

**Each one is written as a failure mode, not tied to a specific plugin** - match the symptom, then apply the decision rule.

| # | Failure mode | Symptom / correct action |
|---|---|---|
| 5.1 | **Version allowlist falls behind a host upgrade** | Shows as "error" → **upgrade**. Shows "update failed" + `update-rollback` → **do not upgrade**, the new release is broken |
| 5.2 | **Stateful core package in `dependencies`** | Shadow package masks the host → **desktop won't launch** |
| 5.3 | **`inject`ed service has no provider** | Plugin stays **PENDING**; grouped plugins must be enabled together |
| 5.4 | **JSON written with a BOM** | `Unexpected token ''` → `JSON.parse` fails |
| 5.5 | **Profile gets rewritten concurrently** | Edits rolled back; includes `.dsh-market\state.json` |
| 5.6 | **Git install with no build handling** | No `lib/` in the package; needs `prepare` + `allowBuilds` |
| 5.7 | **Duplicate client factory registration** | `duplicate factory registration` → editing config usually **does not help**; disable the plugin |
| 5.8 | **Duplicate route registration** | `duplicate exact route` → half-installed state after a rolled-back upgrade; trust the `node_modules` version |
| 5.9 | **market double-block** | `plugin kept off` → clear **both** `disabled[]` and `brokenPlugins{}` |

**The most common misread:** "error" and "update failed" both look like "this plugin is broken", but the conclusions are opposite - the first is fixed by upgrading, the second gets worse. See the 5.1 decision table in `references/pitfalls.md`.

Each entry includes the **failure mechanism, a diagnostic script, the raw crash log, and the fix**.

## Who it's for

- **AI agents** - load this skill before touching DSH plugins or skills
- **Plugin authors** - use it as a pre-release checklist
- **DSH users** - work through the triage flow when the desktop app won't start

## Official sources this is based on

- Documentation site: <https://deepseek-harness.github.io/deepseek-harness/>
- Official repo: <https://github.com/deepseek-ai/deepseek-harness>
- **Official skill specification**: `docs/subsystems/skills.md`
- **Official bundled skill templates**: `packages/preset/agent-preset/skills/`
- Bundled official skill examples: `resources/runtime/office-skills/`

## Compatibility

> This is a **documentation-only skill** with no executable plugin code, so it is not subject to version allowlists - it loads safely on any DSH version.

The pitfalls come from boot failures reproduced on this machine's DSH desktop build and verified fixed. **The exact version numbers may not correspond one-to-one with the npm releases** - go by the symptom and the crash log, not the version number.

## Contributing

Found a new pitfall? PRs welcome. Please include:

1. The symptom and the full crash log
2. Trigger conditions (DSH version / plugin version / steps)
3. Root-cause analysis
4. The fix

> Please describe the **failure mode** ("duplicate client factory registration"), not just "plugin X crashed" - the former is reusable, the latter expires in two months.

## License

[MIT](LICENSE)
