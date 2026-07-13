---
note_type: knowledge
template_version: 1
contract_version: 1
knowledge_id: "KNOW-0004"
category: architecture
title: "Integration Map"
status: in_progress
owner: ""
created: '2026-06-30'
updated: '2026-06-30'
reviewed_on: ""
related_notes: []
tags: [apovault, knowledge, architecture]
---

# Integration Map

## External Services

- **Official Figma MCP** — the single external dependency. The plugin calls the
  already-installed `mcp__plugin_figma_figma__*` tools (`use_figma`, `get_design_context`,
  `whoami`, `search_design_system`, …); it bundles no MCP of its own. Auth is interactive via
  Figma desktop. Each command runs a `whoami` preflight and stops if it fails.
  **Sources:** `ARCHITECTURE.md:23-34` (read 2026-06-30), `commands/create-android.md:11` (read 2026-06-30).
- **Figma published libraries** (referenced by key in the registry):
  `tidepoolLoopAndroid`, `material3` (forward); `coreIos17` (reverse). `tidepoolLoopIos` and
  `appleHig` are scaffolded empty — discovered + committed on first reverse run.
  **Sources:** `registry/components.json:4-7` (read 2026-06-30), `registry/components.json:46-51` (read 2026-06-30).
- **Referenced Figma files (in skill docs):** Material 3 Design Kit
  (figma.com/community/file/1035203688168086460), Loop iOS customizations (file
  `m8iprZw0FBO1DDZq0QlpUw`). **Source:** `skills/ios-to-android/SKILL.md:241-243` (read 2026-06-30).

## Internal APIs

- **No HTTP/RPC surface.** The plugin's "API" is its slash-command set
  (`/figma-sync:create-android`, `/figma-sync:create-ios`, planned `sync-screens`,
  `apply-ds-update`) plus the two skills and two agents. **Source:** `README.md:23-30` (read 2026-06-30).

## Data Flow

- **Registry (read) + source Figma section (read) → target section (write) → mapping (write).**
  `create-*` reads `registry/components.json` for keys/tokens/layout, derives the target from
  the source section + target DS via `use_figma`, then writes the flow entry to
  `~/.figma-sync/mappings.json`. **Source:** `commands/create-android.md:13,22-29` (read 2026-06-30).
- **Planned consumers of the mapping:** `sync-screens` reads `screenPairs` to know which
  Android node mirrors a changed iOS node; `apply-ds-update` reads the mapping to enumerate the
  target flows within its **one resolved `fileKey`** (single-file scope — other files, incl.
  cross-file duplicates, need their own run) and offloads an agent for that file.
  **Source:** `ARCHITECTURE.md:84-85,111` (read 2026-07-13).

## References

- [[System_Overview]], [[Domain_Model]], [[Agent_Workflow]]

_No project-level Figma references were supplied to `/apo:init` (empty user input), so no
`## Figma` section is recorded here; workstreams carry their own `figma` refs._

## Verification status

Fully cited from ARCHITECTURE, README, command, skill, and registry files. No `(verify)` items.
