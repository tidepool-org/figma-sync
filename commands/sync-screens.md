---
description: Detect iOS↔Android content drift for a mapped flow and present a reviewable sync plan (read-only — makes no edits)
argument-hint: "[flow name or file key] [optional freeform hints]"
---

Detect content drift between the iOS and Android screens of a mapped flow and present a
**sync plan** for review. The target flow (and any steering hints) come from: $ARGUMENTS

Follow the **`figma-sync:drift-sync`** skill as the authoritative playbook for the detection
method, the content-vs-chrome boundary, the snapshot format, and the sync-plan format. Do not
duplicate or deviate from it here. This command is **read-only**: it enumerates, reads,
compares, and reports — it makes **no Figma edits**. Work through these phases:

## 1. Preflight (do not skip)
- Confirm the Figma MCP is available and authenticated: call `mcp__plugin_figma_figma__whoami`. If it fails, stop and tell the user to open Figma desktop, enable the MCP server, and sign in. (This plugin rides on the official Figma MCP — it does **not** provide its own.)
- Read the design-system constants from `${CLAUDE_PLUGIN_ROOT}/registry/components.json` (component keys, tokens, layout values). Use those keys — do not re-discover by search unless a key is missing/stale.
- Load `~/.figma-sync/mappings.json` and locate the flow named (or file-keyed) in `$ARGUMENTS`. **If the flow is unmapped, stop** and tell the user to run a `create-android` / `create-ios` command first to establish the section nodes and `screenPairs` — drift detection needs the recorded pairs.

## 2. Direction
- Resolve the sync direction from the arguments / hints; **default iOS → Android** when unspecified. Detection is symmetric, so the direction only orients the proposed action and the live diff (per the skill's §5).

## 3. Hints
- Capture any freeform hint text trailing the flow id in `$ARGUMENTS` (e.g. "focus on the intro copy", "the Android side is ahead"). Use it to steer detection emphasis and the proposed direction — it never authorizes a mutation.

## 4. Detect (read-only)
- Defer to the **`figma-sync:drift-sync`** skill: enumerate each section's direct children and pair by the stored `screenPairs` node ids (never name-regex), compare **content only** (text, images, structure, CTA label, screen fill — ignore chrome and font family), and run the divergence algorithm (live cross-platform diff plus snapshot attribution when a per-side snapshot exists). Use **read-only** `use_figma` only.

## 5. Present the plan & stop
- Output the **per-screen sync plan** exactly as the skill's §6 specifies: per pair the direction, the detected delta fields, any conflict flag (and which side), structural notes (added/removed or terminal screens), and a proposed action; plus an overall summary. Flag conflicts and added/removed screens for the designer — never resolve them silently.
- **Make no mutation.** Invite the user to refine the plan (direction, scope, conflict resolution). Note that applying the approved plan — and the snapshot/rollback safety net — is performed by a later, separate step of this workflow; this command stops at the reviewed plan.
