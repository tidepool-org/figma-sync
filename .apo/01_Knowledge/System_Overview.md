---
note_type: knowledge
template_version: 1
contract_version: 1
knowledge_id: "KNOW-0001"
category: architecture
title: "System Overview"
status: in_progress
owner: ""
created: '2026-06-30'
updated: '2026-06-30'
reviewed_on: ""
related_notes: []
tags: [apovault, knowledge, architecture]
---

# System Overview

## Purpose

figma-sync is a **Claude Code plugin** that translates Figma screen flows between
**iOS (Apple HIG)** and **Android (Material 3)** in either direction, and (planned) keeps
the two platforms in sync as design specs evolve. Built for Tidepool's design team.
**Source:** `README.md:1-8` (read 2026-06-30), `.claude-plugin/plugin.json:3` (read 2026-06-30).

## High-Level Architecture

- **A Claude Code plugin, not a standalone app or MCP server.** The repo is simultaneously
  a plugin and its own single-plugin marketplace (`source: "./"`). It declares no MCP server
  of its own — it rides on the already-installed **official Figma MCP** (`use_figma`,
  `get_design_context`, `whoami`, …), which authenticates interactively via Figma desktop.
  This keeps all token/secret handling out of the plugin.
  **Sources:** `ARCHITECTURE.md:23-35` (read 2026-06-30), `.claude-plugin/marketplace.json:7-14` (read 2026-06-30).
- **Three artifact types compose the system:** slash **commands** (entry-point verbs) →
  **skills** (the authoritative per-direction conventions playbooks) → optional **agents**
  (offload/parallelize per-screen building). Two data artifacts bind them: the committed
  **registry** (DS constants) and a user-global **mapping** (work done).
  **Sources:** `ARCHITECTURE.md:36-96` (read 2026-06-30), `README.md:31-39` (read 2026-06-30).
- **Derive, don't copy:** a target platform's design is derived only from the source screens
  plus the target platform's design system — never from a pre-existing target section.
  **Sources:** `README.md:42-43` (read 2026-06-30), `skills/ios-to-android/SKILL.md:25-29` (read 2026-06-30).

## Key Components

Top-level layout (from `ls -F`, read 2026-06-30):

- `commands/` — slash-command entry points (`create-android`, `create-ios`, plus planned
  `sync-screens`, `apply-ds-update`). **Source:** `commands/` listing (read 2026-06-30).
- `skills/` — `ios-to-android/SKILL.md` and `android-to-ios/SKILL.md`, the mirrored conventions
  playbooks. **Source:** `skills/` listing (read 2026-06-30).
- `agents/` — `screen-builder.md` (→ Android) and `screen-builder-ios.md` (→ iOS), for
  offloading per-screen construction. **Source:** `agents/` listing (read 2026-06-30).
- `registry/components.json` — versioned design-system constants. **Source:** `registry/components.json:1-3` (read 2026-06-30).
- `.claude-plugin/` — `plugin.json` + `marketplace.json` (install manifest).
  **Source:** `.claude-plugin/plugin.json` (read 2026-06-30).
- `mappings.example.json`, `README.md`, `ARCHITECTURE.md` — schema doc + docs.
  **Source:** root `ls -F` (read 2026-06-30).

## Technology Stack

- **Runtime / container:** Claude Code plugin; installed via `/plugin marketplace add
  tidepool-org/figma-sync` then `/plugin install figma-sync@tidepool`. No build step.
  **Source:** `README.md:15-21` (read 2026-06-30).
- **Design-tool bridge:** official Figma MCP (`mcp__plugin_figma_figma__*` tools), driven
  through `use_figma`. Plugin does not bundle its own MCP. **Source:** `ARCHITECTURE.md:23-34` (read 2026-06-30).
- **Authored content:** Markdown (skills/commands/agents with YAML frontmatter) + JSON
  (registry, mapping schema). No JS/TS/compiled source; no package manager, lock file, or
  test framework present. **Source:** root tree + `apo mine subsystems` / `apo mine theme-sources` (read 2026-06-30).
- **Versioning:** semver in `plugin.json` (`0.2.0`); `registry/components.json` carries an
  independent `version` (`0.1.0`) tracking DS-spec revisions.
  **Sources:** `.claude-plugin/plugin.json:3` (read 2026-06-30), `registry/components.json:3` (read 2026-06-30).

## References

- [[Code_Map]], [[Domain_Model]], [[Integration_Map]], [[Agent_Workflow]], [[Coding_Standards]]
- `ARCHITECTURE.md` — design rationale + roadmap (§7 phasing, §9 decisions captured).

## Verification status

Fully cited from README, ARCHITECTURE, plugin/marketplace manifests, registry, and directory
listing. No `(verify)` items in this file.
