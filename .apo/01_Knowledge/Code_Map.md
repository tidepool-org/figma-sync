---
note_type: knowledge
template_version: 1
contract_version: 1
knowledge_id: "KNOW-0002"
category: architecture
title: "Code Map"
status: in_progress
owner: ""
created: '2026-06-30'
updated: '2026-06-30'
reviewed_on: ""
related_notes: []
tags: [apovault, knowledge, architecture]
---

# Code Map

## Subsystems

Small codebase (~13 authored files, no JS/TS source) — treated as a **single subsystem**.
**Source:** `apo mine subsystems` + root tree (read 2026-06-30).

| slug | path | purpose | file count | status |
|---|---|---|---|---|
| figma-sync | `.` (repo root) | The whole plugin: commands + skills + agents + registry | ~13 | active |

## Directory Structure

Annotated top-level (from `ls -F` + `find -maxdepth 2`, read 2026-06-30):

```
figma-sync/
├── .claude-plugin/
│   ├── plugin.json            # name, semver (0.2.0), description, keywords
│   └── marketplace.json       # single-plugin marketplace (name "tidepool", source "./")
├── commands/                  # slash-command entry points (the verbs)
│   ├── create-android.md      # ✅ iOS → Android
│   ├── create-ios.md          # ✅ Android → iOS
│   ├── sync-screens.md        # 🚧 planned
│   └── apply-ds-update.md     # 🚧 planned
├── skills/
│   ├── ios-to-android/SKILL.md  # forward-direction conventions playbook
│   └── android-to-ios/SKILL.md  # reverse-direction conventions playbook
├── agents/
│   ├── screen-builder.md      # per-screen build offload → Android
│   └── screen-builder-ios.md  # per-screen build offload → iOS
├── registry/
│   └── components.json        # versioned DS constants (keys, tokens, layout)
├── mappings.example.json      # schema for the user-global mapping
├── README.md
└── ARCHITECTURE.md
```

**Source:** `ARCHITECTURE.md:36-59` (read 2026-06-30), root `find -maxdepth 2` (read 2026-06-30).

## Key Files

- **Command entry points:** `commands/create-android.md`, `commands/create-ios.md` — resolve
  a Figma URL/file key, preflight the MCP, follow the matching skill, write the mapping, report.
  **Sources:** `commands/create-android.md:1-32`, `commands/create-ios.md:1-33` (read 2026-06-30).
- **Authoritative conventions:** `skills/ios-to-android/SKILL.md` (244 lines),
  `skills/android-to-ios/SKILL.md` (269 lines) — the per-direction playbooks the commands defer to.
  **Source:** `wc -l skills/*/SKILL.md` (read 2026-06-30).
- **DS source of truth:** `registry/components.json` — library + component keys, tokens,
  typography rules, layout constants; forward keys at top, reverse keys under `androidToIos`.
  **Source:** `registry/components.json:1-86` (read 2026-06-30).
- **Mapping schema:** `mappings.example.json` — documents `~/.figma-sync/mappings.json`
  (user-global, not in repo). **Source:** `mappings.example.json:1-20` (read 2026-06-30).
- **Install manifest:** `.claude-plugin/plugin.json` + `marketplace.json`.
  **Source:** `.claude-plugin/plugin.json:1-7` (read 2026-06-30).

## Module Boundaries — figma-sync

- **Commands defer to skills; skills hold the conventions.** A command file is a thin
  orchestration script (preflight → idempotency guard → golden rule → build per skill →
  record mapping → report); it names the skill as the authoritative playbook and says "do not
  deviate." **Source:** `commands/create-android.md:8` (read 2026-06-30).
- **Skills name registry keys; the registry holds values.** Skills teach the *method* and
  reference registry keys by name; literal numbers/hex inline are illustrative only — "when
  prose and registry disagree, the registry wins." **Source:** `skills/ios-to-android/SKILL.md:16-21` (read 2026-06-30).
- **The two skills are mirrors.** `ios-to-android` (forward) and `android-to-ios` (reverse)
  share the registry grid constants + `tokens.brandPrimary`; reverse-only constants live under
  the registry's `androidToIos` block. **Sources:** `ARCHITECTURE.md:93-96` (read 2026-06-30), `registry/components.json:44-85` (read 2026-06-30).
- **Persistent state is user-global, never in-repo.** The mapping lives at
  `~/.figma-sync/mappings.json`; only its schema (`mappings.example.json`) is committed.
  **Source:** `ARCHITECTURE.md:72-83` (read 2026-06-30).

## References

- [[System_Overview]], [[Domain_Model]], [[Agent_Workflow]], [[Coding_Standards]]

## Verification status

Fully cited from ARCHITECTURE, command/skill files, registry, and directory listings. No
`(verify)` items.
