---
description: Detect iOS↔Android content drift for a mapped flow, present a reviewable sync plan, and — on explicit approval — propagate the content deltas in the chosen direction with a backup/rollback safety net
argument-hint: "[flow name or file key] [optional freeform hints]"
---

Detect content drift between the iOS and Android screens of a mapped flow, present a **sync
plan** for review, and — **only after the user explicitly approves** — propagate the approved
content deltas in the chosen direction with a backup/rollback safety net. The target flow (and
any steering hints) come from: $ARGUMENTS

Follow the **`figma-sync:drift-sync`** skill as the authoritative playbook for the detection
method, the content-vs-chrome boundary, the snapshot format, the sync-plan format, and the
backup / apply / conflict-execution / restore procedures. Do not duplicate or deviate from it
here. Phases 1–5 are **read-only** (enumerate, read, compare, report); phases 6–12 **write, and
only after approval**. Work through these phases:

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

## 5. Present the plan
- Output the **per-screen sync plan** exactly as the skill's §6 specifies: per pair the direction, the detected delta fields, any conflict flag (and which side), structural notes (added/removed or terminal screens), and a proposed action; plus an overall summary. Flag conflicts and added/removed screens for the designer — never resolve them silently. Invite the user to refine the plan (direction, scope, conflict resolution).

## 6. Approval gate (do not skip)
- **Make no mutation until the user explicitly approves in a following turn** (e.g. "apply" / "yes, apply"). An echoed or refined plan is **not** approval. No affirmative → stop here with no edits.
- Only conflicts the designer **resolved during refinement** are eligible for propagation. Unresolved conflicts and any added/removed screens are **skipped** (structural propagation is out of scope) — carry them into the report, never guess.

## 7. Back up (before any mutation)
- Per the skill's §7, take the backup **once**, before the first write, keyed to the delta type: content-only deltas back up via the pre-sync snapshot (no canvas node); structural deltas duplicate the affected target screen(s) into a **hidden frame** (`visible=false`) and record the backup node id + snapshot. **No backup → do not mutate.**

## 8. Apply the approved deltas
- Per the skill's §8, apply each approved clean delta in the chosen direction — routing to `figma-sync:ios-to-android` or `figma-sync:android-to-ios` — re-running the **content steps only** (re-clone `Content AL`, re-font, re-copy fill, update CTA label); leave chrome and conventions intact. Overwrite-to-source-content, so re-runs are idempotent. Work in **atomic ≤10-op batches** and **screenshot between**.

## 9. Verify
- Screenshot each touched screen and compare against the source's content. On a mismatch, **restore from the backup** (skill §7/§10) and report the failure — **never leave a silent partial apply.**

## 10. Update the mapping
- Per the skill's §10, rewrite **both per-side snapshots** for each synced pair (re-baselining drift + the rollback record) and set `lastSyncedAt` (today) + `lastSyncDirection`. Write to `~/.figma-sync/mappings.json` — mirror the create-* commands' **Record the mapping** phase; see `${CLAUDE_PLUGIN_ROOT}/mappings.example.json` for the schema.

## 11. Clean up backups
- **keep-last-1 per flow** (skill §7): after the run succeeds, delete any older backup frame for this flow, retaining only the most recent.

## 12. Report
- Mirror the create-* commands' **Report** phase: a per-screen summary of what propagated and its verify result, plus any **failures** and any **skipped** conflicts / added-removed screens. Surface everything — never a silent partial apply. Share the affected section link.
