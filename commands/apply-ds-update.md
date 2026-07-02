---
description: Roll a committed design-system (registry) change across every mapped Figma file for a platform via a full per-screen re-apply that preserves project overrides, with an approval gate, per-file backup/rollback, and per-flow version stamping
argument-hint: "[ios|android] [summary of what changed]"
---

Roll a **committed** design-system change — reflected into
`${CLAUDE_PLUGIN_ROOT}/registry/components.json` and committed as the auditable record — across
**every mapped Figma file** for the chosen platform. Each file is brought to the current design
system by a **full per-screen re-apply** that **preserves deliberate project overrides**, behind
an explicit approval gate with a per-file backup/rollback safety net and per-flow version
stamping. The platform (and a summary of what changed) come from: $ARGUMENTS

Follow the **`figma-sync:ds-update`** skill as the authoritative playbook for the delta method,
the per-`fileKey` work set (including cross-file duplicates), override detection/preservation,
the backup mechanism, the full re-apply routing, verify/restore, version stamping, and the
reversed reconciliation rule. Do not duplicate or deviate from it here. Phases 1–5 are
**read-only** (preflight, delta, work set, override detect, present); phases 6–11 **write, and
only after approval**. Work through these phases:

## 1. Preflight (do not skip)
- Confirm the Figma MCP is available and authenticated: call `mcp__plugin_figma_figma__whoami`. If it fails, stop and tell the user to open Figma desktop, enable the MCP server, and sign in. (This plugin rides on the official Figma MCP — it does **not** provide its own.)
- Read the current design-system constants from `${CLAUDE_PLUGIN_ROOT}/registry/components.json` — its `version` and the component keys / tokens / layout values as they stand now. This is the state you re-apply to; use those keys, do not re-discover by search unless a key is missing/stale.
- **Confirm the DS change is committed, not just dirty.** The registry commit is the auditable record and the source of the delta — if `registry/components.json` has uncommitted changes, stop and tell the user to commit the DS edit first. Never apply from a dirty working tree.
- Resolve the target platform (`ios` or `android`) from `$ARGUMENTS` and capture the trailing `[summary of what changed]` as the steering hint. If the platform is missing or ambiguous, stop and ask.

## 2. Compute the delta
- Obtain the machine-readable change via `git diff` on `${CLAUDE_PLUGIN_ROOT}/registry/components.json` (the committed before→after). Cross-check it against the human `[summary of what changed]`; **surface any divergence** — a summary naming a change the diff doesn't show, or vice versa — before proceeding. Never apply an unstated change.

## 3. Build the work set
- Load `~/.figma-sync/mappings.json`. Select flows that have a section node for the target platform (`iosSectionNodeId` for `ios`, `androidSectionNodeId` for `android`). **If no flow maps the platform, stop** and tell the user to run a `create-ios` / `create-android` command first to establish the section nodes.
- Per the skill's §2: **group by `fileKey`** so each physical file is processed once, and **include cross-file duplicates** — each is its own file to update (this is the reverse of `sync-screens`, which flags cross-file duplicates rather than editing them).
- **Skip flows whose `dsVersion` already equals the registry `version`** (already at this DS state) — this is what makes a partial fan-out resumable. Override the skip only when the `$ARGUMENTS` hints contain the word **`force`** (re-apply even current flows); note in the report when force was in effect. A flow with no `dsVersion` is never-DS-updated and always in the set.

## 4. Detect overrides (read-only)
- Per the skill's §3, per file surface the **deliberate project overrides** the re-apply must preserve — chrome values the file intentionally sets away from the DS default. Detection is best-effort: **flag ambiguous overrides** (deliberate customization vs. stale drift) for the designer rather than guessing. This pass makes **no edits**; it produces the per-screen override list that feeds the approval gate and the re-apply spec.

## 5. Present the plan
- Show the **DS delta** (from phase 2), the **file work set** split into *applied* vs *skipped-current* (and whether `force` overrode any skip), and the **detected overrides** per file with the ambiguous ones called out for a decision. Invite the user to refine scope, confirm ambiguous overrides, or adjust the platform/force. Present only — **make no mutation here.**

## 6. Approval gate (do not skip)
- **Make no mutation until the user explicitly approves in a following turn** (e.g. "apply" / "yes, apply"). An echoed or refined plan is **not** approval. No affirmative → stop here with no edits.
- Ambiguous overrides the designer did **not** resolve during refinement are not eligible — either confirm them or exclude those files; never auto-clobber or auto-preserve on a guess.

## 7. Back up (after approval, before any write)
- Per the skill's §4, take a per-file backup **before the first write**: a full re-apply is structure-changing, so **duplicate the affected platform section's screens into a hidden frame** (`visible=false`) named for the run, and record the backup node id in `~/.figma-sync/mappings.json`. The content snapshot cannot back up a chrome change; reverting the registry commit cannot restore mutated files. **No backup → do not mutate.**

## 8. Fan out — one agent per `fileKey`
- Per the skill's §5, process **one file per agent**: offload each `fileKey` to the `screen-builder` (→ Android) / `screen-builder-ios` (→ iOS) agent with a **"full re-apply preserving overrides"** spec — the section node id, the ordered screen node ids, and the per-screen override list from phase 4. The agent rebuilds each screen's chrome from the **current** DS (routing per screen to `figma-sync:ios-to-android` / `figma-sync:android-to-ios`) while carrying the overrides forward; an empty override list means a clean rebuild to stock DS. Work in **atomic ≤10-op batches** and **screenshot between**.

## 9. Verify
- Per the skill's §6, screenshot each touched screen and confirm the chrome matches the current DS and each preserved override survived. On a mismatch, **restore from the hidden-duplicate backup** and report the failure. Verify the rebuilt screens' app-bar / nav titles read the flow name, not a content H1. **Never leave a silent partial apply** — report precisely which files applied, which were skipped, and which failed-and-restored.

## 10. Stamp version & clean up backups
- Per the skill's §7, on each file's **successful** re-apply, write `dsVersion` (the registry `version` just applied) + `dsAppliedAt` (today, ISO) for every flow whose file was updated; see `${CLAUDE_PLUGIN_ROOT}/mappings.example.json` for the schema. A file that **failed** (restored) is **not** stamped, so a re-run retries it.
- **keep-last-1 per flow:** after the run succeeds, delete any older backup frame for this flow, retaining only the most recent.

## 11. Report
- Per the skill's §9, output a per-file summary: **applied** (with overrides preserved per file), **skipped-current** (and any `force`), **failed** (mismatched on verify and restored, with reason), the backup node id + verify result per file, and the `dsVersion` / `dsAppliedAt` stamped per updated flow. A DS change does not propagate between files — every file was updated directly, so there is no cross-file reconciliation step. Surface everything — never claim success for a file that failed verify.
