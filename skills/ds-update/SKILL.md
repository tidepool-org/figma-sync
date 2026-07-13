---
name: ds-update
description: Roll a committed design-system change (registry edit) across the mapped flows of one specified Figma file for a platform. The authoritative playbook the apply-ds-update command defers to. Covers reading the DS delta from the registry diff, building the single-file work set from the command's resolved fileKey, detecting and preserving deliberate project overrides, the section-level hidden-duplicate backup, a full per-screen re-apply that routes through the direction skills, per-flow version stamping, verify-or-restore, and the single-file scoping rule (other files, including cross-file duplicates, need their own run). Detection is read-only; the apply half writes via the Figma MCP only after the command's approval gate. Use during apply-ds-update.
---

# Design-system update propagation

The authoritative playbook for rolling a **committed registry (design-system) change** into the
mapped flows of **one specified Figma file** for a chosen platform. When an upstream iOS or
Android design system updates, the plugin operator first reflects the change into
`${CLAUDE_PLUGIN_ROOT}/registry/components.json` and **commits** it — that commit is the
auditable record of the DS change. This skill then re-applies the change into the **one file the
run targets** so its live Figma work tracks the registry. Each run is deliberately single-file:
other mapped files — including cross-file duplicates of the same flows — are updated by a
separate run against each, never automatically swept in (§2, §8).

This is the mirror of how `drift-sync` is the playbook the `sync-screens` command defers to.
The two are **orthogonal**: `sync-screens` / `drift-sync` own **content** drift between an iOS
and Android twin; this skill owns **chrome / design-system** state within a platform section. A
DS change touches status bars, app bars, tokens, layout constants, component instances — never
a screen's content — so nothing here overlaps content sync.

**The halves differ on writes.** Reading the delta, building the work set, and detecting
overrides (§1–§3) make **no Figma edits**. The apply half (§4–§7) writes, and **only ever after
the command's explicit approval gate**; it backs up before it mutates and can restore. The
`apply-ds-update` command runs the read-only phases, stops at the reviewed plan, and executes
the mutation half only on explicit approval.

> Prerequisite: the official **Figma MCP** must be running and authenticated
> (`mcp__plugin_figma_figma__whoami`). This skill drives **read-only** `use_figma` for
> detection and **atomic** `use_figma` writes for the re-apply.
>
> **Source of truth for values:** component keys, design tokens, and layout constants live
> in `${CLAUDE_PLUGIN_ROOT}/registry/components.json` — read them from there. This doc teaches
> the *method* and names the registry keys; any literal shown inline is **illustrative**, not
> authoritative. When prose and registry disagree, the registry wins.

---

## 0. Golden rule — derive from the current DS, don't copy

Rebuild each screen's chrome **only** from (1) the source screen it was derived from and (2) the
**current** design system in the registry — never by copying a finished reference section. This
is the same golden rule the translation skills hold: the platform section is *derived*, so a DS
update re-derives it from the now-updated registry. The one addition here: a file may carry a
**deliberate project override** of a DS default — the re-apply **preserves** those overrides
rather than reverting them to stock (§3).

---

## 1. The delta — read it from the registry diff

The DS change already lives in the registry as a committed edit; reconstruct **what changed**
from that record, do not re-invent it:

1. Read the **current** `${CLAUDE_PLUGIN_ROOT}/registry/components.json` — its `version` and the
   keys/tokens/layout constants as they stand now. This is the state you re-apply to.
2. Obtain the machine-readable change via `git diff` on that file (the operator committed it, so
   the diff is the authoritative before→after). The designers themselves keep no git repo — the
   registry commit is the plugin operator's record on their behalf.
3. Cross-check the diff against the human `[summary of what changed]` the run was given, and
   **surface any divergence** (a summary that names a change the diff doesn't show, or vice
   versa) before proceeding — never apply an unstated change.

The delta drives the re-apply but does **not** narrow it to a field patch: a DS change ripples
through chrome in ways a per-field patch would miss, so the delta informs *which* files are
stale, and the apply is a **full per-screen re-apply** from the current DS (§5).

---

## 2. Work set — the flows of the one specified file

The run is scoped to a **single file**: the command resolves a figma.com URL / file key to one
`fileKey` and passes it in. Read `~/.figma-sync/mappings.json` and build the work set **within
that one `fileKey`** for the `[ios|android]` platform:

1. **Restrict to the resolved `fileKey`.** Only flows in that file are eligible — no other mapped
   file is enumerated or touched, even one that maps the same flow.
2. Within it, select flows that have a section node for the requested platform (`iosSectionNodeId`
   for `ios`, `androidSectionNodeId` for `android`). If the file maps no flow for the platform,
   stop — the user must run a `create-*` command against that file first.
3. **Cross-file duplicates are out of scope.** Two flows with the **same node ids but different
   `fileKey`s** are **independent duplicate files** (Figma duplicates preserve node ids across
   files); the one **not** matching the resolved `fileKey` is deliberately **not** updated by this
   run. A DS change is applied one file at a time — re-run the command with that file's URL to
   bring it current. (Same-`fileKey` flows reference the same live nodes, so they are updated
   together as part of the one file.)
4. **Skip already-current flows.** Skip any flow whose `dsVersion` already equals the current
   registry `version` — it is already at this DS state. This is what makes a partial run
   resumable: a re-run against the same file picks up only the flows not yet stamped. Skip is
   overridable when the run carries a **`force`** hint (re-apply even current flows); state clearly
   in the report that force was in effect.

A flow with **no** `dsVersion` is treated as never-DS-updated and is always in the work set.

---

## 3. Override detection & preservation (read-only)

Before rebuilding a file, surface the **deliberate project overrides** the re-apply must carry
forward — a value the file intentionally sets *away* from the DS default (a custom app-bar
color, a non-stock component variant, an adjusted layout constant):

1. For each screen, diff the screen's **chrome** against what the current DS default would
   produce (reconstruct the default from the registry the same way a `create-*` build would).
2. A deviation from the default is a **candidate override**. Carry it into the rebuild so the
   re-apply does not silently revert it to stock.
3. **Confirm ambiguous overrides.** Detection is **best-effort** — a deviation may be a
   deliberate customization *or* stale drift the update should overwrite, and the two can look
   identical. When it is unclear, **stop and ask the designer** rather than guess; never
   auto-clobber a deliberate override and never auto-preserve obvious drift. The confirm step is
   the safety net for detector noise.

This pass makes **no edits** — it produces the override list that (a) feeds the command's
approval gate and (b) is passed into the re-apply spec (§5).

---

## 4. Backup before mutating

**Nothing is mutated until a backup exists.** A full re-apply rebuilds each screen's chrome
wholesale, so it is a **structure-changing** operation — reuse the backup mechanism the
`drift-sync` skill establishes (its §7, structure-changing branch), specialized to
section-level granularity:

- **Duplicate the affected platform section's screens into a hidden frame** (`visible=false`) on
  the page, named for the run (e.g. `ds-backup — <flow> — <version>`), and record the backup
  node id in `mappings.json`. The hidden frame never occupies a screen slot, so it leaves the
  column-aligned section layout undisturbed.
- **Restore swaps the duplicate back wholesale** — re-link it into the section and re-snap it to
  the section's slot.
- **Per file, keep-last-1:** on a successful run, retain only the most recent backup per flow and
  delete any older backup frame.

Do **not** rely on the per-side content snapshot as the backup — it holds **content only** and a
DS change touches **chrome**, so the snapshot cannot restore it. Do **not** rely on reverting the
registry commit — that undoes the DS record, not the Figma files already mutated. **No backup →
no mutation.**

---

## 5. Apply — full per-screen re-apply from the current DS

For each screen in a file's platform section, re-run the **full build** the `create-*` commands
run — rebuilding chrome from the **current** registry/DS while preserving the detected overrides
(§3):

- Route by platform: a `→ android` re-apply runs each screen through the `ios-to-android` skill;
  a `→ ios` re-apply runs each screen through the `android-to-ios` skill — the same build path
  `create-android` / `create-ios` use. Do **not** restate those per-screen build steps here;
  route to them.
- Rebuild derives from the source screen + the current DS, then **applies the override list** so
  deliberate deviations survive the rebuild.
- The re-apply is **overwrite-to-current-DS**, so it is **idempotent** on the DS dimension:
  running it twice converges on the same chrome state.
- The **golden rule** still holds — derive from source + current DS, **never invent UI the source
  lacks** (a terminal screen has no CTA / toolbar; do not add one).
- Work in **atomic ≤10-op batches** and **screenshot between batches**.

**Offload.** The `apply-ds-update` command offloads the **one resolved file** to the
`screen-builder` (→ Android) / `screen-builder-ios` (→ iOS) agent. These agents already build
screens from a spec, so pass a **"full re-apply preserving overrides"** spec: the section node id,
the ordered screen node ids, and the per-screen **override list** from §3. No new agent mode is
required — the override list rides in the spec; an empty list means a clean rebuild to stock DS.

---

## 6. Verify — screenshot, and restore on mismatch

After a file's screens are rebuilt:

1. **Screenshot each touched screen** and confirm the chrome matches the current DS and that each
   preserved override survived the rebuild.
2. On a mismatch, **restore from the hidden-duplicate backup** (§4) and **report the failure**.
   **Never leave a silent partial apply** — a mid-run failure reports precisely which flows in the
   file applied, which were skipped, and which failed-and-restored.
3. Verify the rebuilt screens' app-bar / nav titles read the **flow name**, not a content H1 —
   the same builder gotcha the translation skills warn about.

---

## 7. Version stamp

On a file's **successful** re-apply, write two per-flow fields in `mappings.json` for every flow
whose file was updated:

- **`dsVersion`** — the registry `version` just applied (verbatim string).
- **`dsAppliedAt`** — today's date (ISO `YYYY-MM-DD`).

These are the skip-already-current + resume-partial-run signal §2 reads on the next run against
this file. A flow that **failed** (restored from backup) is **not** stamped — it stays behind so a
re-run retries it. Both fields are optional and back-compatible (see `mappings.example.json`).

---

## 8. Scope — one file per run, no cross-file sweep

State this explicitly: a DS run **touches only the one file it targets** and **does not propagate
to other files**. Like `drift-sync`, a run is a **within-file** operation — a different-`fileKey`
duplicate of the same flow is left untouched and must be brought current by its **own run**
against that file's URL. This is a deliberate scope choice: the operator applies a DS change one
file at a time and reviews each, rather than sweeping every mapped file at once.

Within the one file there is nothing to "reconcile across files" after the fact — the file's flows
were processed directly. Same-`fileKey` flows share live nodes, so updating the file updates them
all at once; their `dsVersion` stamps are written together. Reaching **every** mapped copy of a
flow is the operator's job across **multiple runs**, one per file, tracked by each file's per-flow
`dsVersion` stamp (§7).

---

## 9. Report

Close with a per-flow summary for the one file (name the file explicitly):

- **Applied** — flows re-applied, with the overrides preserved per flow.
- **Skipped (current)** — flows already at the registry `version` (or note **force** if it
  overrode the skip).
- **Failed** — flows that mismatched on verify and were **restored from backup**, with the
  failure reason.
- **Backup + verify** — the backup node id and its verify result.
- **Stamp** — the `dsVersion` / `dsAppliedAt` written per updated flow.
- **Out of scope** — note that other mapped files, including any cross-file duplicate of these
  flows, were **not** updated and need their own run.

Never claim success for a flow that failed verify; the report is the record that the run was
complete and non-silent.

---

## 10. Gotchas

- **The registry commit is the trigger, read the delta from `git diff`** — never apply an
  unstated change; cross-check the diff against the human summary and surface divergence.
- **Scope the work set to the one resolved `fileKey`** — the run updates only that file; a
  different-`fileKey` duplicate is **out of scope** and needs its own run against its URL. Never
  sweep every mapped file at once.
- **Skip flows already at the registry `version`** unless `force` — this is what makes a partial
  run resumable; a flow with no `dsVersion` is never-DS-updated and always in.
- **Override detection is best-effort; confirm the ambiguous ones** — never auto-clobber a
  deliberate override, never auto-preserve obvious drift; ask when the two are indistinguishable.
- **A full re-apply is structure-changing — back up into a hidden duplicate frame first** — the
  content snapshot holds no chrome and cannot restore a DS change; **no backup → no mutation**.
- **Route the per-screen build to the direction skills** (`ios-to-android` / `android-to-ios`);
  do not restate their steps or hardcode DS values — the registry wins on values.
- **Pass the override list in the offload spec** — the `screen-builder` / `screen-builder-ios`
  agents take a "re-apply preserving overrides" spec; no new agent mode is needed.
- **Verify each screen; restore-and-report on mismatch** — never a silent partial apply.
- **Stamp only successful files** — a failed/restored file stays unstamped so a re-run retries it.
- **A DS run touches only its one target file** — other files, including cross-file duplicates,
  need their own run (§8); reaching every mapped copy is done across multiple runs, one per file.
- **Work atomically** — `use_figma` writes in ≤10-op batches, screenshot between; a failed atomic
  op changes nothing.

---

## 11. Reference

- Forward per-screen build method and screen anatomy: the `ios-to-android` skill.
- Reverse per-screen build method (mirror): the `android-to-ios` skill.
- Backup / hidden-duplicate mechanism this skill reuses: the `drift-sync` skill, §7.
- Mapping schema, incl. the per-flow `dsVersion` / `dsAppliedAt` stamps: `${CLAUDE_PLUGIN_ROOT}/mappings.example.json`.
