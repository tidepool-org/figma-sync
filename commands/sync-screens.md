---
description: Detect iOS↔Android content drift for a mapped flow, present a reviewable sync plan, and — on explicit approval — propagate the content and screen-level structural deltas in the chosen direction with a backup/rollback safety net
argument-hint: "[flow name or file key] [optional freeform hints]"
---

Detect content drift between the iOS and Android screens of a mapped flow, present a **sync
plan** for review, and — **only after the user explicitly approves** — propagate the approved
content **and screen-level structural** deltas in the chosen direction with a backup/rollback
safety net. The target flow (and any steering hints) come from: $ARGUMENTS

Follow the **`figma-sync:drift-sync`** skill as the authoritative playbook for the detection
method, the content-vs-chrome boundary, the snapshot format, the sync-plan format, and the
backup / apply / conflict-execution / restore procedures. Do not duplicate or deviate from it
here. Phases 1–5 are **read-only** (enumerate, read, compare, report); phases 6–12 **write, and
only after approval**. Work through these phases:

## 1. Preflight (do not skip)
- Confirm the Figma MCP is available and authenticated: call `mcp__plugin_figma_figma__whoami`. If it fails, stop and tell the user to open Figma desktop, enable the MCP server, and sign in. (This plugin rides on the official Figma MCP — it does **not** provide its own.)
- Read the design-system constants from `${CLAUDE_PLUGIN_ROOT}/registry/components.json` (component keys, tokens, layout values). Use those keys — do not re-discover by search unless a key is missing/stale.
- Load `~/.figma-sync/mappings.json` and locate the flow named (or file-keyed) in `$ARGUMENTS`. **If the flow has no section mapping at all** (no `iosSectionNodeId` / `androidSectionNodeId`), **stop** and tell the user to run `create-android` / `create-ios` (or `build-page`) first — detection needs the section nodes. A flow that **is** mapped at section level but has **empty `screenPairs`** (an **adopted pair's first sync**, e.g. from `build-page` adoption) is **not** an error: proceed via the skill's **live-pairing** path (§1.2), which pairs the screens live and proposes them in the plan — no manual seeding.

## 2. Direction
- Resolve the sync direction from the arguments / hints; **default iOS → Android** when unspecified. Detection is symmetric, so the direction only orients the proposed action and the live diff (per the skill's §5).

## 3. Hints
- Capture any freeform hint text trailing the flow id in `$ARGUMENTS` (e.g. "focus on the intro copy", "the Android side is ahead"). Use it to steer detection emphasis and the proposed direction — it never authorizes a mutation.

## 4. Detect (read-only)
- Defer to the **`figma-sync:drift-sync`** skill: enumerate each section's direct children and pair by the stored `screenPairs` node ids (never name-regex) — **or, when `screenPairs` is empty/partial, pair the unpaired screens live by normalized name then order (skill §1.2), proposing those pairings in the plan** — compare **content only** (text, images, structure, CTA label, screen fill — ignore chrome and font family), and run the divergence algorithm (live cross-platform diff plus snapshot attribution when a per-side snapshot exists). Use **read-only** `use_figma` only.

## 5. Present the plan
- Output the **per-screen sync plan** exactly as the skill's §6 specifies: per pair the direction, the detected delta fields, any conflict flag (and which side), structural notes with the proposed **build/remove** action (added/removed or terminal screens), and a proposed action; plus an overall summary. Present conflicts and the proposed structural actions for the designer — never resolve or execute them silently; a removal is destructive and must be called out as such. Invite the user to refine the plan (direction, scope, conflict resolution, which structural actions to approve, and — for an adopted pair's first sync — confirm or repair the proposed live pairings).

## 6. Approval gate (do not skip)
- **Make no mutation until the user explicitly approves in a following turn** (e.g. "apply" / "yes, apply"). An echoed or refined plan is **not** approval. No affirmative → stop here with no edits.
- Only conflicts the designer **resolved during refinement** and structural actions the designer **explicitly approved** are eligible for propagation. A **screen removal** is destructive and needs explicit per-screen approval. Unresolved conflicts and any **unapproved** added/removed screens are **skipped** — carry them into the report, never guess or auto-build/auto-delete.

## 7. Back up (before any mutation)
- Per the skill's §7, take the backup **once**, before the first write, keyed to what the delta changes: content-only deltas back up via the pre-sync snapshot (no canvas node); a screen **removal** (or a structure-changing content delta) duplicates the affected target screen(s) into a **hidden frame** (`visible=false`) and records the backup node id + snapshot; a screen **build** records its built node id as the rollback record. **No backup → do not mutate.**

## 8. Apply the approved deltas
- Per the skill's §8, apply each approved **content** delta in the chosen direction — routing to `figma-sync:ios-to-android` or `figma-sync:android-to-ios` — re-running the **content steps only** (re-clone `Content AL`, re-font, re-copy fill, update CTA label); leave chrome and conventions intact. Overwrite-to-source-content, so re-runs are idempotent.
- Per the skill's §8.1, apply each approved **structural** action: **build** a missing target twin via the `create-*` path (the matching direction skill or the `screen-builder` / `screen-builder-ios` agent), slotting it at the column-aligned x/y; **remove** an orphaned target screen only after its backup exists, then re-close the column gap.
- Work in **atomic ≤10-op batches** and **screenshot between**.

## 9. Verify
- Screenshot each touched screen and compare against the source's content. On a mismatch, **restore from the backup** (skill §7/§10) and report the failure — **never leave a silent partial apply.**

## 10. Update the mapping
- Per the skill's §10, rewrite **both per-side snapshots** for each synced pair (re-baselining drift + the rollback record), **reconcile `screenPairs`** for structural deltas (add a pair with fresh snapshots for each built screen; drop the pair for each removed screen), and set `lastSyncedAt` (today) + `lastSyncDirection`. Write to `~/.figma-sync/mappings.json` — mirror the create-* commands' **Record the mapping** phase; see `${CLAUDE_PLUGIN_ROOT}/mappings.example.json` for the schema.

## 11. Clean up backups
- **keep-last-1 per flow** (skill §7): after the run succeeds, delete any older backup frame for this flow, retaining only the most recent.

## 12. Reconcile related mappings (do not skip)
- Reconciliation is **part of the task**, not an optional extra — run it after this flow's mapping is updated, before you report. Per the skill's §10 step 5, re-open `~/.figma-sync/mappings.json` and check every **other** flow for staleness introduced by the deltas you just applied, especially **structural** ones (a built/removed screen changes a section's screen set).
- **Same-file siblings (genuine sharing):** a flow whose **`fileKey` matches** the one you edited references the *same physical nodes* — update its `screenPairs` / snapshots the same way (§10) if your change touched a section/screen it also maps.
- **Cross-file duplicates (NOT sharing):** a flow with the **same node ids but a different `fileKey`** is an independent Figma file (duplicates preserve node ids). Your change did **not** propagate to it — **never** edit its mapping or assume parity; at most **flag** that it may want its own `sync-screens` run. Gate "shared" on `fileKey`, never on bare node ids.
- If nothing else is affected, say so — a silent skip reads as "nothing to reconcile" when there may have been.

## 13. Report
- Mirror the create-* commands' **Report** phase: a per-screen summary of what propagated (content deltas **and** structural builds/removals) and its verify result, plus any **failures**, any **skipped** conflicts / unapproved structural changes, and the **reconciliation outcome** (siblings updated, duplicates flagged, or nothing affected). Surface everything — never a silent partial apply. Share the affected section link.
