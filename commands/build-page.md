---
description: Reconcile a whole Figma page of many flows so every flow has its target-platform section below it — building missing sections, relocating misplaced ones (geometry only), and leaving unmanaged content untouched, behind an approval gate with a reversible layout backup
argument-hint: "[figma url or file key] [ios|android]"
---

Reconcile a Figma **page that holds many flows** so that every managed flow has its
target-platform section **directly below its source**, on the page referenced by: $ARGUMENTS

For each flow the command **builds** a missing counterpart, **relocates** a misplaced one
(geometry only — never rebuilding its content), or leaves an in-place one alone — while
**unmanaged content is never touched**. This is the page-level driver over the same per-flow
translation `create-android` / `create-ios` perform one at a time.

Follow the **`figma-sync:page-layout`** skill as the authoritative playbook for the identity
stamp, the classifier, the managed column band, the declarative packer / make-room reflow, the
three-state reconcile, legacy adoption, the layout backup, and the optional `sync-screens`
hand-off. Do not duplicate or deviate from it here. Phases 1–5 are **read-only** (preflight,
classify, work set, plan); phase 6 is the **approval gate**; phases 7–11 **write, and only after
approval**. Work through these phases:

## 1. Preflight (do not skip)
- Confirm the Figma MCP is available and authenticated: call `mcp__plugin_figma_figma__whoami`. If it fails, stop and tell the user to open Figma desktop, enable the MCP server, and sign in. (This plugin rides on the official Figma MCP — it does **not** provide its own.)
- Resolve the **page** (a figma.com URL or file key) and the **target platform** (`ios` or `android`) from `$ARGUMENTS`. The platform is the side to *ensure*: `android` builds/relocates an Android section below each iOS source; `ios` does the reverse. If either is missing or ambiguous, stop and ask.
- Read the design-system layout constants from `${CLAUDE_PLUGIN_ROOT}/registry/components.json` (`layout.*`). Use those keys — do not re-discover.

## 2. Classify the page (read-only)
- Per the skill's §2, enumerate the page's top-level sections and their **direct children** (never name-regex) and tag each `ios`, `android`, or `non-flow` from structural signals — screen width (`layout.iosScreenWidth` 375 vs `layout.androidScreenWidth` 412), sidebar `Status` fill (iOS black / Android white), and name as a weak tiebreak. Read each section's identity stamp at the same time. Also **flag any managed section whose name doesn't clearly encode its platform** (a bare `Section`) as **unlabeled**, with a proposed `iOS Section` / `ANDROID Section` label (skill §2.4) — for the plan only. **Make no edits.**

## 3. Build the work set + band (read-only)
- Per the skill's §1 + §3, from the classified sections determine the **source** sections (the platform opposite the target) and, for each, its target-platform counterpart via the **identity stamp** (`figmaSyncPairId` + `figmaSyncRole`). Assign each flow a state: **build** (no counterpart), **move** (counterpart exists but not in its canonical slot), or **no-op** (already in slot).
- Flag each **legacy** (unstamped) counterpart as an **adoption candidate** with its *inferred* pairing (name → order / screen-count; skill §6) for confirmation in the plan.
- Compute the **managed column band** (the x-range of the sources + their target slots). `non-flow` sections and any out-of-band section are excluded — they will not be moved or collision-checked.

## 4. Plan the layout (read-only)
- Per the skill's §4, run the **declarative packer** over the canonical flow order to compute each managed section's target `(x, y)` and the **reserved rects** for sections to be built. This is a computation only — **no mutation**.

## 5. Present the plan
- Show: the **classifier table** (each section's platform, stamped vs adoption-candidate, non-flow/skipped, and whether its **name** is platform-labeled or **unlabeled** with a proposed label); the **per-flow reconcile** (build / move / no-op) for the target platform; the **inferred adoption pairings** with ambiguous ones called out for a decision; and a **placement preview** (what moves where, which sections push down to make room). Invite the user to refine scope, confirm/repair pairings, adjust the platform, and **confirm or decline the proposed section-name labels for any unlabeled managed sections** (skill §2.4). Present only — **make no mutation here.**

## 6. Approval gate (do not skip)
- **Make no mutation until the user explicitly approves in a following turn** (e.g. "apply" / "yes, build"). An echoed or refined plan is **not** approval. No affirmative → stop here with no edits.
- Adoption pairings the user did **not** confirm are not eligible — either confirm them or exclude those flows; never adopt or relocate on a guess.

## 7. Layout backup (after approval, before any write)
- Per the skill's §7, snapshot **every managed section's `id → (x, y)`** (and width/height) and record it for the run. This makes the whole page arrangement reversible independently of any per-section content backup. **No layout backup → do not mutate.**

## 8. Reflow & relocate — the packer writes
- Per the skill's §4 + §5, move each **existing** managed section (including confirmed **adopted** ones, whose stamp is written first, skill §6) to its computed target, opening the reserved rects for the builds. Relocation is **geometry only** — never rebuild or clobber a moved section's content (skill §5.1). Work in **atomic ≤10-op batches** and **screenshot between**; the declarative packer makes a partially-applied reflow safe to resume.
- Apply any **confirmed section-name labels** in this phase — a cosmetic **rename only** (the section's name, never its content or geometry); skip any label the user didn't confirm, and never rename a `non-flow` section.

## 9. Fan out the builds — one agent per flow
- Per the skill's §5 (build state), offload each flow that needs a **new** counterpart to the `screen-builder` (→ Android) / `screen-builder-ios` (→ iOS) agent, giving it the source section node id, the ordered source screen node ids, the **reserved rect** for placement, and the **page node id**. The agent builds each screen via `figma-sync:ios-to-android` / `figma-sync:android-to-ios`, **parents the new section to that page** (`targetPage.appendChild(section)` by id — a `use_figma` call resets `currentPage` to the *first* page, so a section created before switching silently lands on the wrong page; parenting by id makes placement deterministic), **writes the identity stamp** on the new section, and **returns the new section's `parent.id`** for the §10 check. Reserved rects keep parallel agents from colliding.

## 10. Verify, stamp & record
- Verify each built or relocated section: **first assert its `parent.id` equals the page node id** — a screenshot renders a node on *any* page, so it cannot catch a section left on the wrong page; check the parent explicitly. Then screenshot to confirm it sits in its canonical slot, unmanaged sections are untouched, and — for **builds** — each app-bar / nav title reads the flow name (not a content H1). On a **wrong-page** section, **re-parent it** (`targetPage.appendChild`); on any other mismatch, **restore from the layout backup** (and, for a failed build, remove the built node) and report the failure.
- On success per flow, ensure the identity stamp is written and update `~/.figma-sync/mappings.json` (`flowId`, section node ids, `screenPairs`); see `${CLAUDE_PLUGIN_ROOT}/mappings.example.json`. **Never leave a silent partial apply** — report precisely which flows were built, moved, left as no-ops, skipped, or failed-and-restored.

## 11. Offer sync & report
- Per the skill's §8, for each **adopted / relocated** pair only (a freshly built counterpart is in-sync by construction), **offer** — do not auto-run — a per-flow `sync-screens` hand-off, behind `sync-screens`' own approval gate, only after that flow's relocate → stamp → mapping write. Note that the first sync on an adopted pair is **live-only** (no baseline; it asks direction).
- Report the per-flow outcome (built / moved / no-op / skipped / failed), the layout backup id + verify result, the flows stamped + mapped, and which adopted pairs were offered a sync.
