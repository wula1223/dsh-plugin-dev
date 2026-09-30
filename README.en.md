# dsh-plugin-dev

> **DeepSeek Harness plugin / hook / skill development standard** — a battle-tested playbook for AI agents and human developers.

English | [中文](README.md)

---

## What is this

A **DSH agent skill** that makes an AI follow the official contract and avoid known desktop-breaking pitfalls when building, modifying, or reviewing **DeepSeek Harness plugins / hooks / skills**.

It is not a general introduction — it is an **operating manual that fuses the official specification with real incidents**. The mechanism sections track the official docs; every "pitfall" corresponds to an actual boot failure.

## Why you need it

DSH's philosophy is **"Everything is a Plugin"**, and its plugin ecosystem is wide open. That openness comes with sharp edges:

- A **bundle and a profile are two different manifests** — confuse them and your package won't install
- Plugins declare an **exact-version allowlist** (not a semver range) — one host upgrade and they break
- A stateful core package in `dependencies` **shadows the host** and can make the desktop app unlaunchable
- A plugin whose `inject`ed service has no provider stays **PENDING** forever and stalls startup
- A single **BOM byte** is enough to crash the whole boot sequence in `JSON.parse`
- **Installing from git pulls source, not build output** — without a `prepare` script the package won't load

Without this knowledge, an AI easily ships a plugin that "looks right but breaks on install".

## Install

### Option 1 — clone into the global skills directory (recommended)

```bash
git clone https://github.com/wula1223/dsh-plugin-dev.git \
  ~/.dsh/skills/dsh-plugin-dev
```

Windows PowerShell:

```powershell
git clone https://github.com/wula1223/dsh-plugin-dev.git `
  "$env:USERPROFILE\.dsh\skills\dsh-plugin-dev"
```

### Option 2 — copy SKILL.md only

```powershell
$dest = "$env:USERPROFILE\.dsh\skills\dsh-plugin-dev"
New-Item -ItemType Directory -Force -Path $dest | Out-Null
Copy-Item .\SKILL.md $dest
```

Restart DSH afterwards and the skill shows up in the available list.

## Contents

| File | Purpose |
|---|---|
| [`SKILL.md`](SKILL.md) | **The skill itself** (529 lines): official contracts + six pitfalls + verification checklist + release standard |
| [`references/official-docs-index.md`](references/official-docs-index.md) | **Official docs map**: the docs site plus the full repo `docs/` index |

### What SKILL.md covers

| # | Section | Highlights |
|---|---|---|
| 0 | **Authoritative sources** | Official docs site (and the `.md` trick), repo `docs/`, on-disk examples |
| 1 | **Plugin contract** | Three plugin forms · **bundle vs profile** · bundle manifest · **four-layer load order** · the five cordis concepts · **`Config` + Schemastery** |
| 2 | **Hooks and events** | Exact semantics of all five dispatch modes · waterfall must call `next()` · TS declaration merging · **`namespace/action` naming** · ⚠️ session events vs Cordis events |
| 3 | **Lifecycle** | The **Fiber state machine** (PENDING/LOADING/ACTIVE/FAILED/…) · auto-cleanup · ⚠️ disposers run reversed but concurrently · dispose · HMR |
| 4 | **Skill format** | `SKILL.md` frontmatter and how to write a triggerable `description` |
| 5 | **⚠️ Six pitfalls** | Each with the real crash log and the fix |
| 6 | **Local verification checklist** | Copy-pasteable PowerShell + `dsh --dump-config` |
| 7 | **Release / publishing** | Repo layout · pre-flight checklist · topics · git push environment traps |
| 8 | **Triage flow** | 7 steps for a desktop app that won't start |
| 9 | **Reporting standard** | Always show evidence, never just "should be fixed" |

## The six fatal pitfalls

| # | Pitfall | Consequence |
|---|---|---|
| 1 | Version allowlist not bumped with the host | Plugin shows as **error** |
| 2 | Stateful core package in `dependencies` | Shadow package masks the host → **desktop won't launch** |
| 3 | `inject`ed service has no provider | Plugin stays **PENDING** → **startup fails** |
| 4 | JSON written with a BOM | `JSON.parse` fails → **boot crash** |
| 5 | Editing config while DSH is running | Changes overwritten |
| 6 | Git install with no build handling | Package arrives without `lib/` → **load fails** |

Each one includes the **real incident, the raw crash log, and the fix** in `SKILL.md`.

## Who it's for

- **AI agents** — load this skill before touching DSH plugins or skills
- **Plugin authors** — use it as a pre-release checklist
- **DSH users** — work through the triage flow when the desktop app won't start

## Official sources this is based on

- Documentation site: <https://deepseek-harness.github.io/deepseek-harness/>
- Official repo: <https://github.com/deepseek-ai/deepseek-harness>
- Bundled official skill examples: `resources/runtime/office-skills/`

## Compatibility

| DSH version | Status |
|---|---|
| `0.2.0-rc.2` | ✅ Verified — all six pitfalls reproduced and fixed on this version |
| `0.1.7-rc.2` | ✅ Applicable — the version-allowlist section comes from this release line |

> ⚠️ This is a **documentation-only skill** with no executable plugin code, so it is not subject to version allowlists — it loads safely on any DSH version.

## Contributing

Found a new pitfall? PRs welcome. Please include:

1. The symptom and the full crash log
2. Trigger conditions (DSH version / plugin version / steps)
3. Root-cause analysis
4. The fix

## License

[MIT](LICENSE)
