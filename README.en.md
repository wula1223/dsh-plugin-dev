# dsh-plugin-dev

> **DeepSeek Harness plugin / hook / skill development standard** — a battle-tested playbook for AI agents and human developers.

English | [中文](README.md)

---

## What is this

A **DSH agent skill** that makes an AI follow the official contract and avoid known desktop-breaking pitfalls when building, modifying, or reviewing **DeepSeek Harness plugins / hooks / skills**.

It is not a general introduction — it is an **operating manual distilled from real incidents**. Every "pitfall" corresponds to an actual boot failure.

## Why you need it

DSH's philosophy is **"Everything is a Plugin"**, and its plugin ecosystem is wide open. That openness comes with sharp edges:

- Plugins declare an **exact-version allowlist** (not a semver range) — one host upgrade and they break
- A third-party plugin putting a core package in `dependencies` **shadows the host** and can make the desktop app unlaunchable
- A plugin whose `inject`ed service has no provider stays `pending` forever and stalls startup
- A single **BOM byte** is enough to crash the whole boot sequence in `JSON.parse`

Without this knowledge, an AI easily ships a plugin that "looks right but breaks on install".

## Install

### Option 1 — clone into the global skills directory (recommended)

```bash
git clone https://github.com/<your-username>/dsh-plugin-dev.git \
  ~/.dsh/skills/dsh-plugin-dev
```

Windows PowerShell:

```powershell
git clone https://github.com/<your-username>/dsh-plugin-dev.git `
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
| [`SKILL.md`](SKILL.md) | **The skill itself**: official contracts + five fatal pitfalls + local verification checklist + release checklist |
| [`references/official-docs-index.md`](references/official-docs-index.md) | **Official docs map**: where every DSH spec document lives and what it covers |

### What SKILL.md covers

1. **Authoritative sources** — where the official docs are, plus local paths
2. **Plugin contract** — the `dsh` field in `package.json`, `dsh.bundle.patch`, `cordis.patch.yml`
3. **The cordis plugin model** — Service / Context / `inject` / `ctx.effect()`
4. **Hooks and events** — the five dispatch modes (`emit` / `waterfall` / `parallel` / `serial` / `bail`) and waterfall semantics
5. **Skill format** — `SKILL.md` frontmatter and how to write a triggerable `description`
6. **⚠️ Five fatal pitfalls** — version allowlists, shadow packages, dangling dependencies, BOM, concurrent profile writes
7. **Local verification checklist** — copy-pasteable PowerShell
8. **Release checklist** — what to verify before publishing

## The five fatal pitfalls

| # | Pitfall | Consequence |
|---|---|---|
| 1 | Version allowlist not bumped with the host | Plugin shows as **error** |
| 2 | Core package in `dependencies` | Shadow package masks the host → **desktop won't launch** |
| 3 | `inject`ed service has no provider | Plugin stays `pending` → **startup fails** |
| 4 | JSON written with a BOM | `JSON.parse` fails → **boot crash** |
| 5 | Editing config while DSH is running | Changes overwritten |

Each one includes the **real incident, the raw crash log, and the fix** in `SKILL.md`.

## Who it's for

- **AI agents** — load this skill before touching DSH plugins or skills
- **Plugin authors** — use it as a pre-release checklist
- **DSH users** — work through section 7 when the desktop app won't start

## Compatibility

| DSH version | Status |
|---|---|
| `0.2.0-rc.2` | ✅ Verified — every pitfall was reproduced and fixed on this version |
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
