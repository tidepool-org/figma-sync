---
note_type: knowledge
template_version: 1
contract_version: 1
knowledge_id: "KNOW-0005"
category: architecture
title: "Agent Workflow"
status: in_progress
owner: ""
created: '2026-06-30'
updated: '2026-06-30'
reviewed_on: ""
related_notes: []
tags: [apovault, knowledge, architecture]
---

# Agent Workflow

This plugin *is* an agent workflow: its commands, skills, and sub-agents orchestrate Claude
against the Figma MCP. (For repo-wide agent rails / coding rules, see [[Prompt_Standards]].)

## Agent Roles

- **Command agent (main thread)** — a `/figma-sync:create-*` command drives the flow:
  preflight `whoami` → idempotency guard → build per the matching skill → record mapping →
  report. **Source:** `commands/create-android.md:10-32` (read 2026-06-30).
- **`screen-builder` (→ Android)** — sub-agent that builds one or more Android screens via
  `use_figma`, strictly following `figma-sync:ios-to-android`. Given the `ANDROID Section`
  node id + template node ids + a screen list `{ name, iosContentNodeId, order, hasBack,
  title }`, returns `{ name, screenId }[]` + per-screen warnings.
  **Source:** `agents/screen-builder.md:1-27` (read 2026-06-30).
- **`screen-builder-ios` (→ iOS)** — mirror sub-agent following `figma-sync:android-to-ios`;
  takes Android screen node ids, returns built iOS screen ids. Re-fonts to SF Pro Text/Display
  by 20px threshold; height logic against 812. **Source:** `agents/screen-builder-ios.md:1-32` (read 2026-06-30).
- **Purpose of offload:** parallelize heavy per-screen Figma construction so the main thread
  stays clean. **Source:** `agents/screen-builder.md:3` (read 2026-06-30).

## Workflow Patterns

- **Skill-as-playbook:** commands defer to a skill ("follow it as the authoritative playbook;
  do not deviate"); the skill teaches method and names registry keys.
  **Source:** `commands/create-android.md:8` (read 2026-06-30).
- **Screen 1 first, then batch:** build screen 1 and get user sign-off before building the
  rest. **Source:** `commands/create-android.md:24` (read 2026-06-30).
- **Atomic small steps:** `use_figma` is atomic (failed scripts make no changes); work in
  ≤10-op batches and screenshot between. **Source:** `skills/ios-to-android/SKILL.md:207-208` (read 2026-06-30).
- **Fan-out (planned):** `apply-ds-update` fans out one `screen-builder` agent per mapped
  file to re-apply a DS delta. **Source:** `commands/apply-ds-update.md:17-18` (read 2026-06-30).
- **Mirror source chrome:** terminal/menu screens with no nav/CTA get none on the target —
  don't invent UI the source lacks. **Source:** `agents/screen-builder-ios.md:26-28` (read 2026-06-30).

## Tooling

- Drives the official Figma MCP tools (`use_figma`, `get_design_context`, `whoami`,
  `search_design_system`). See [[Integration_Map]]. **Source:** `ARCHITECTURE.md:25-26` (read 2026-06-30).
- Reads `registry/components.json` for keys/tokens/layout at runtime.
  **Source:** `commands/create-android.md:13` (read 2026-06-30).

## References

- [[Code_Map]], [[Integration_Map]], [[Prompt_Standards]]

## Verification status

Fully cited from command, skill, and agent files. No `(verify)` items.
